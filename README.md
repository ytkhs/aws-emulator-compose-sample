# AWS Emulator (Docker Compose)

[AWS Emulator](https://docs.aws.amazon.com/emulator/latest/userguide/getting-started-docker.html) をローカルの Docker Compose で動かすための設定です。公式の `docker run` 手順を compose に置き換えています。

## ファイル

| ファイル | 内容 |
| --- | --- |
| `Dockerfile` | Amazon Linux 2023 に AWS Emulator の rpm を入れたイメージ。CPU アーキテクチャは `EMU_ARCH` で切り替える |
| `compose.yml` | ポート公開・ボリューム・メモリ制限などの起動設定 |
| `compose.windows.yml` | Windows 用の差分（マウント先・`HOME`・CPU アーキテクチャ）。`compose.yml` に重ねて使う |

## 起動

初回のみ、エミュレーターが証明書や認証情報を書き出すディレクトリを作ります。

```bash
mkdir -p ~/.aws/emulator
```

```bash
docker compose up -d --build
```

ヘルスチェック：

```bash
curl \
  --cacert ~/.aws/emulator/9297/ca.pem \
  https://127.0.0.1:9297/__admin/health
```

`ok` が返れば起動完了です。

## リクエストを送る

エミュレーターが `~/.aws/emulator/9297/` に AWS CLI / SDK 用の config と認証情報を生成します。これを環境変数で指定すれば、`--endpoint-url` なしで普段どおり `aws` コマンドが使えます。

```bash
export AWS_CONFIG_FILE=~/.aws/emulator/9297/config
export AWS_SHARED_CREDENTIALS_FILE=~/.aws/emulator/9297/emulator_credentials
unset AWS_PROFILE AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
```

`unset` は、本番のプロファイルやアクセスキーが環境変数に残っているとそちらが優先されるのを防ぐためです。設定はそのターミナル内だけに効きます。

接続先の確認（アカウントが `000000000000` ならエミュレーター）：

```bash
aws sts get-caller-identity
```

リージョンは `us-east-1` です。SDK を使うアプリも、同じ環境変数を設定して起動すればエミュレーターに接続します。

### S3

```bash
aws s3 mb s3://my-bucket
echo hello > /tmp/h.txt
aws s3 cp /tmp/h.txt s3://my-bucket/
aws s3 ls s3://my-bucket
```

### DynamoDB

```bash
aws dynamodb create-table \
  --table-name users \
  --attribute-definitions AttributeName=id,AttributeType=S \
  --key-schema AttributeName=id,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
aws dynamodb wait table-exists --table-name users
aws dynamodb put-item \
  --table-name users \
  --item '{"id":{"S":"u1"},"name":{"S":"taro"}}'
aws dynamodb get-item \
  --table-name users \
  --key '{"id":{"S":"u1"}}'
```

作成直後のテーブルは本物と同じく `CREATING` 状態なので、`wait table-exists` を挟まないと `ResourceNotFoundException` になります。

### SQS

```bash
aws sqs create-queue --queue-name jobs
QURL=$(aws sqs get-queue-url \
  --queue-name jobs \
  --query QueueUrl \
  --output text)
aws sqs send-message --queue-url "$QURL" --message-body hi
aws sqs receive-message --queue-url "$QURL"
```

返るキュー URL は `sqs.us-east-1.amazonaws.com` の形式ですが、リクエストはエミュレーターに届きます。

### Lambda

関数のコードを zip にして登録し、実行します。compose 側の追加設定（Docker ソケットのマウントなど）は不要です。

```bash
cat > index.py <<'EOF'
def handler(event, context):
    return {"hello": event.get("name", "world")}
EOF
zip f.zip index.py
```

```bash
aws lambda create-function \
  --function-name hello \
  --runtime python3.12 \
  --handler index.handler \
  --role arn:aws:iam::000000000000:role/x \
  --zip-file fileb://f.zip
aws lambda wait function-active-v2 --function-name hello
```

```bash
aws lambda invoke \
  --function-name hello \
  --cli-binary-format raw-in-base64-out \
  --payload '{"name":"taro"}' \
  out.json
cat out.json
```

`{"hello": "taro"}` が返れば成功です。

- `--role` は形式の正しい ARN であれば、事前にロールを作らなくても通ります
- 作成直後は `Pending` 状態なので `wait function-active-v2` を挟みます。コード更新後は `wait function-updated-v2` です
- `--cli-binary-format raw-in-base64-out` は AWS CLI v2 で JSON を素のまま渡すためのオプションです
- `print` の出力は CloudWatch Logs の `/aws/lambda/<関数名>` に出ます

SQS をトリガーにする場合：

```bash
QARN=$(aws sqs get-queue-attributes \
  --queue-url "$QURL" \
  --attribute-names QueueArn \
  --query Attributes.QueueArn \
  --output text)
aws lambda create-event-source-mapping \
  --function-name hello \
  --event-source-arn "$QARN"
```

この状態で SQS にメッセージを送ると、`event["Records"]` に入って関数が呼ばれます。

#### 実行環境の注意

ランタイムは本物の CPython / Node.js ではありません。

- `python3.12` は RustPython 上で動きます（`sys.version` で確認できます）。純 Python のコードと `boto3` は動きますが、numpy や pydantic v2 など C 拡張を含むライブラリは動かない可能性が高いです
- `nodejs22.x` も組み込みの JS エンジンで、`process.versions` には `node` しか出ません。一部の `node:` モジュールは使えない可能性があります

サービス間の連携（S3 → Lambda → DynamoDB など）や軽い関数の確認向きです。重い依存ライブラリや実行時間・メモリの検証には、本物のランタイムをコンテナで動かす `sam local invoke` か実際の AWS を使ってください。

## 停止

```bash
docker compose down
```

リソースはメモリ上にしかないため、停止すると作ったバケット・テーブル・キュー・関数はすべて消えます。`~/.aws/emulator/9297/` 内の証明書と認証情報も削除され、次回起動時に作り直されます。

## 設定の変更

### ポート

ホスト側・コンテナ側・エミュレーターの `-p` の 3 か所を `EMU_PORT` でまとめて変えます。エミュレーターが生成ファイルに自分のポートを書き込むため、3 か所が一致している必要があります。

```bash
EMU_PORT=9300 docker compose up -d
```

この場合、生成ファイルは `~/.aws/emulator/9300/` になるので、上の `export` のパスも合わせて変えてください。

### CPU アーキテクチャ

デフォルトは Apple silicon 向けの `aarch64` です。Intel Mac や x86_64 の Linux ではビルド時に指定します。

```bash
EMU_ARCH=x86_64 docker compose up -d --build
```

### Linux で使う場合

生成ファイルの所有者を自分にするため、`compose.yml` のサービスに `user: "<UID>:<GID>"`（`id -u` / `id -g` の値）を追加してください。

### テレメトリ

デフォルトで有効です。止める場合は `compose.yml` の `environment` に `EMULATOR_TELEMETRY: "off"` を追加します。

## Windows で使う場合（未検証）

> この節は公式ドキュメントの Windows 手順をもとに書いたもので、Windows 実機では動作確認していません。`compose.windows.yml` を重ねたときの設定のマージ結果（`docker compose config`）のみ確認済みです。

`compose.yml` は macOS / Linux 向けに、コンテナ内でもホストと同じパスになるよう `HOME` を合わせています。Windows ではこれができないため、`compose.windows.yml` で次の 3 点を差し替えます。

- マウント元を `%USERPROFILE%\.aws\emulator`、マウント先を `/home/emulator/.aws/emulator` にする（Windows には環境変数 `HOME` が通常ない）
- コンテナ内の `HOME` を `/home/emulator` にする
- `EMU_ARCH` の既定値を `x86_64` にする（Arm 版 Windows では `EMU_ARCH=aarch64` を指定）

WSL2 の Ubuntu などから使う場合は Linux と同じなので、`compose.windows.yml` は使わず、上の「Linux で使う場合」に従ってください。

### 起動（PowerShell）

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.aws\emulator"
```

```powershell
docker compose `
  -f compose.yml `
  -f compose.windows.yml `
  up -d --build
```

```powershell
docker compose `
  -f compose.yml `
  -f compose.windows.yml `
  down
```

### 接続先の設定（PowerShell）

生成される `config` の `ca_bundle` はコンテナ内のパス（`/home/emulator/...`）になるため、`AWS_CA_BUNDLE` で Windows 側のパスを指定し直します。

```powershell
$emu = "$env:USERPROFILE\.aws\emulator\9297"
$env:AWS_CONFIG_FILE = "$emu\config"
$env:AWS_SHARED_CREDENTIALS_FILE = "$emu\emulator_credentials"
$env:AWS_CA_BUNDLE = "$emu\ca.pem"
Remove-Item `
  Env:AWS_PROFILE, Env:AWS_ACCESS_KEY_ID, `
  Env:AWS_SECRET_ACCESS_KEY, Env:AWS_SESSION_TOKEN `
  -ErrorAction Ignore
```

```powershell
aws sts get-caller-identity
```

### コマンド例を使うときの注意

上の S3 / DynamoDB / SQS / Lambda の例は bash 向けです。PowerShell では次のように読み替えてください。

- 行継続の `\` は `` ` `` に変える
- `QURL=$(...)` は `$QURL = aws sqs get-queue-url ...` のように書く
- `cat > index.py <<'EOF'` のヒアドキュメントは使えないので、エディタでファイルを作る。zip は `Compress-Archive index.py f.zip` で作れる
- `--item` や `--payload` に JSON を直接書くと PowerShell の引用符処理で崩れやすいので、JSON をファイルに保存して `--item file://item.json` / `--payload fileb://payload.json` のように渡す

## 補足

- ポートは `127.0.0.1` にのみ公開しています。`9297:9297` のように書くとネットワーク上の他マシンからも接続できてしまうので変えないでください。
- 起動ログに「0.0.0.0 で待ち受けている」旨の WARN が出ますが、これはコンテナ内の話で、ホスト側は上記のとおり loopback のみです。
- メモリ上限は 2 GiB（swap なし）です。超えるとコンテナが終了コード 137 で止まり、リソースは失われます。
- ビルド時に rpm 署名の `NOKEY` 警告が出ますが、インストールは完了します。
