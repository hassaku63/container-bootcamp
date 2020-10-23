# fargate-cli-handson

Fargate service/task のデプロイを CLI で実施します。

文頭に `(見解)` と書いてあるものは筆者の知識に基づく見解であり、事実誤認等が含まれる可能性があります。鵜呑みにしないでください。

## 概要

Fargate タイプの ECS クラスタと、稼働するネットワーク (VPC, Subnet, RouteTable, Gateway等) の構築、 フェイシングするロードバランサ (ALB) が構成済みであることを前提として、以下の手順を踏むことで初回デプロイが完了する。

1. 開発環境でアプリケーションのイメージをビルド
2. ECR にビルドしたイメージを push
3. タスク定義を登録
4. Service を作成

また、初回以降でアプリケーションの変更を伴うデプロイを行う手順は次の通り。

1. 開発環境でアプリケーションのイメージをビルド
2. ECR にビルドしたイメージを push
3. タスク定義の新しいリビジョンを登録
4. Service を更新

## 事前条件／前提条件

[Day 1](https://github.com/hassaku63/boyacky) ハンズオンの [handson/boyacky.yaml](https://github.com/hassaku63/boyacky/blob/master/handson/boyacky.yaml) が実行済みであること

ここで実行するコマンドは、特に断りのない限り Cloud9 Console の上で実行するものとする

## 事前作業

シェル変数のセット

```bash
export AWS_ACCOUNT=<your-account-id>
export AWS_REGION=<your-region>  # ap-northeast-1
```

**Note**: Cloud9 の動作環境と ECS のデプロイ先となる環境の AWS アカウントおよびリージョンが一致している場合は、 Cloud9 コンソール上で以下のコマンドを実行することでセットすべき値を取得可能です

```bash
# Cloud9 console
export AWS_ACCOUNT=$(curl -s http://169.254.169.254/latest/dynamic/instance-identity/document | python3 -c "import json, sys; print(json.load(sys.stdin)['accountId']);")
export AWS_REGION=$(curl -s http://169.254.169.254/latest/dynamic/instance-identity/document | python3 -c "import json, sys; print(json.load(sys.stdin)['region']);")
```

ECR レポジトリの名前をシェル変数にセット

```bash
DOCKER_IMAGE_NAME=boyacky/web-app
DOCKER_REMOTE_REPOSITORY=${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com/${DOCKER_IMAGE_NAME}
```

## アプリケーションの最新版を ECR に push

Docker レジストリ (ECR) にログインします。

```bash
aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com
```

イメージをビルドします。ここでは、ビルドを実行した時点での日時をタグとして付与することにします。

```bash
IMAGE_TAG=$(date +%Y%m%d-%H%M%S)
docker build -t ${DOCKER_IMAGE_NAME}:${IMAGE_TAG} .
```

ECR にイメージを push します。

```bash
docker tag ${DOCKER_IMAGE_NAME}:${IMAGE_TAG} ${DOCKER_REMOTE_REPOSITORY}:${IMAGE_TAG}
docker push ${DOCKER_REMOTE_REPOSITORY}:${IMAGE_TAG}
```

## タスク定義の登録

AWS CLI `ecs register-task-definition` を使って、タスク定義を登録します。

実際に使用する json とパラメータ仕様を突合しながら、何をしているのか確認してみましょう。

- register-task-definition パラメータ仕様 ... [params-resgister-task-definition.md](params-resgister-task-definition.md)
- 入力パラメータとして使用する json の雛形 ... [cli-input-templates/task-definition-input.json](cli-input-templates/task-definition-input.json)

### タスク定義の登録 (CLI)

引数の数が多く、かつ引数のいくつか（特に必須の `--container-defeinitions` ）ため、 `--cli-input-json` を用いて実行します。

`register-task-definition` の入力とするJSON を作成します。

[cli-input-templates/task-definition-input.json](cli-input-templates/task-definition-input.json) に雛形となるファイルがあります。
この json ファイルにはプレースホルダとなる変数 ( `${ECS_TASK_ROLE_ARN}}` など) が含まれており、 `envsubst` コマンドを利用することでけれらのプレースホルダに対応する環境変数の値を適用できます。各自の AWS 環境に応じて値をセットし、以下のコマンドを実行しましょう。

環境変数のセット

```bash
#
# 確認用。これまでの作業で既にセット済みの変数を確認します
#
echo $AWS_ACCOUNT
echo $AWS_REGION
echo $DOCKER_IMAGE_NAME
echo $DOCKER_REMOTE_REPOSITORY # 次の値と等価 ${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com/${DOCKER_IMAGE_NAME}
echo $IMAGE_TAG

#
# 各自の環境に合わせて、セットしてください
#
export ECS_TASK_ROLE_ARN=  # タスク実行ロールの ARN
export ECS_TASK_EXEXCUTION_ROLE_ARN=  # タスクロールの ARN
```

`--cli-input-json` の入力に使用する json を生成

```bash
# envsubst を使用して、プレースホルダに環境変数を埋め込んで出力
envsubst < cli-input-templates/task-definition-input.json > cli-inputs/boyacky-taskdef.json
```

`boyacky-taskdef.json` を開いて、プレースホルダ ( `${xxx}` 形式) が残っていないことを確認してください。

json に問題がなければ、 `ecs register-task-definition` でタスク定義を登録します。

```bash
aws ecs register-task-definition --cli-input-json file://cli-inputs/boyacky-taskdef.json
```

正常に Task difinition が作成された場合の出力例は [cli-output-samples/register-task-definition.json](cli-output-samples/register-task-definition.json) のようになります。

出力の json から、 `taskDefinition.taskDefinitionArn` の値を確認し、 `TASKDEF_ARN` 環境変数にセットします。

```bash
# CLI の戻り値の、taskDefinition.taskDefinitionArn をコピペ
# {
#     "taskDefinition": {
#         "taskDefinitionArn": "arn:aws:ecs:<your-region>:<your-aws-account>:task-definition/<taskdef-name>:<revision>", // この値
#         "containerDefinitions": [
#             {
#                 // ...
export TASKDEF_ARN=<your-taskdef-arn>
```

ここまでの終了時点で、以下の環境変数がセットされていることを確認してください。

```bash
echo AWS_ACCOUNT = ${AWS_ACCOUNT}
echo AWS_REGION = ${AWS_REGION}
echo DOCKER_IMAGE_NAME = ${DOCKER_IMAGE_NAME}
echo DOCKER_REMOTE_REPOSITORY = ${DOCKER_REMOTE_REPOSITORY}
echo IMAGE_TAG = ${IMAGE_TAG}
echo ECS_TASK_ROLE_ARN = ${ECS_TASK_ROLE_ARN}
echo ECS_TASK_EXEXCUTION_ROLE_ARN = ${ECS_TASK_EXEXCUTION_ROLE_ARN}
echo TASKDEF_ARN = ${TASKDEF_ARN}
```

また、マネジメントコンソールで "boyacky" というタスク定義が作成されていることを確認してみましょう。

#### Column: CLI で JSON 形式の入力を使用する

雛形となる JSON の生成を `--generate-cli-skeleton` で行えます。AWS CLI の共通オプションとしてサポートされています。

利用するサブコマンドの引数が多い場合や、今回のように json 形式の引数が含まれる場合に検討してみましょう。

Note: AWS CLI v2 では [yaml 形式もサポート](https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/cli-usage-skeleton.html) されました。

```bash
# ecs register-task-definition の JSON 入力のスケルトンを生成
aws ecs register-task-definition --generate-cli-skeleton input
```

かなり多くのキーを持った JSON が生成されますが、最低限必要なものに絞ればそれほどのボリュームはありません。
CLI ドキュメントのサンプル JSON を引用します。

```json
// sample input json file of register-task-definition
{
    "containerDefinitions": [
        {
            "name": "sleep",
            "image": "busybox",
            "cpu": 10,
            "command": [
                "sleep",
                "360"
            ],
            "memory": 10,
            "essential": true
        }
    ],
    "family": "sleep360"
}
```

実際に Fargate タイプでサービスを稼働させる場合、上記のパラメータでは不十分ですが、足りない／不正なパラメータは CLI がバリデーションしてくれるので、試しながら覚えていきましょう。公式ドキュメントにも [タスク定義のサンプル](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/example_task_definitions.html)が公開されています。参考にしてください。

## サービスの作成 (デプロイ)

タスク定義 "boyacky" をもとに、サービスを作成します。

サービスの作成は `ecs create-service`, 更新は `ecs update-service` を利用します。

入力として使用する json ファイル各パラメータの意味を確認していきましょう。

- create-service パラメータ仕様 ... [params-create-service.md](params-create-service.md)
- 使用する json の雛形 ... [cli-input-templates/create-service-input.json](cli-input-templates/create-service-input.json)

### サービス作成 (CLI) の実行

まずは、開始条件を満たしていることを確認しましょう。

- "boyacky" という名前でタスク定義が作成されていること
- 以下の環境変数がセットされていること

```bash
# 環境変数のチェック
cat <<EOF | column -t
AWS_ACCOUNT = ${AWS_ACCOUNT}
AWS_REGION = ${AWS_REGION}
DOCKER_IMAGE_NAME = ${DOCKER_IMAGE_NAME}
DOCKER_REMOTE_REPOSITORY = ${DOCKER_REMOTE_REPOSITORY}
IMAGE_TAG = ${IMAGE_TAG}
ECS_TASK_ROLE_ARN = ${ECS_TASK_ROLE_ARN}
ECS_TASK_EXEXCUTION_ROLE_ARN = ${ECS_TASK_EXEXCUTION_ROLE_ARN}
TASKDEF_ARN = ${TASKDEF_ARN}
EOF
```

次に、サービス作成に必要な追加パラメータをセットします。
ターゲットグループ、サブネット、セキュリティグループの情報を変数にセットします。次の変数を使ってください。

```bash
export TARGET_GROUP_ARN=xxx  # ALB ターゲットグループのARN
export SUBNET_1=subnet-xxx  # デプロイ先のプライベートサブネット(1)
export SUBNET_2=subnet-xxx  # デプロイ先のプライベートサブネット(2)
export TASK_SECURITY_GROUP=sg-xxx  # タスクにアタッチするSG
```

json 入力のテンプレートファイルから、実際の入力ファイルを作成します。

```bash
envsubst < cli-input-templates/create-service-input.json > cli-inputs/create-service-input.json
```

生成された _cli-inputs/create-service-input.json_ に、ターゲットグループ／サブネット／セキュリティグループの情報がセットされていることを確認してください。

問題なければ `ecs create-service` を実行しましょう。

```bash
aws ecs create-service --cli-input-json file://cli-inputs/create-service-input.json | tee cli-outputs/create-service-output.json
```

サービス作成のリクエストが正常に受け付けられれば、[cli-output-samples/create-service-output.json](cli-output-samples/create-service-output.json) のような形式の json が戻ります。

<!-- ```bash
# 環境変数のチェック
cat <<EOF | column -t
AWS_ACCOUNT = ${AWS_ACCOUNT}
AWS_REGION = ${AWS_REGION}
DOCKER_IMAGE_NAME = ${DOCKER_IMAGE_NAME}
DOCKER_REMOTE_REPOSITORY = ${DOCKER_REMOTE_REPOSITORY}
IMAGE_TAG = ${IMAGE_TAG}
ECS_TASK_ROLE_ARN = ${ECS_TASK_ROLE_ARN}
ECS_TASK_EXEXCUTION_ROLE_ARN = ${ECS_TASK_EXEXCUTION_ROLE_ARN}
TASKDEF_ARN = ${TASKDEF_ARN}
TARGET_GROUP_ARN = ${TARGET_GROUP_ARN}
SUBNET_1 = ${SUBNET_1}
SUBNET_2 = ${SUBNET_2}
TASK_SECURITY_GROUP = ${TASK_SECURITY_GROUP}
EOF
``` -->

## サービスの更新

TODO: （ecs update-service を使って desired-count の変更を行う）

## ワンショットタスクの実行

一回きりの処理（バッチなど）を実行する場合は、サービスからではなく、タスクを直接開始します。

タスク実行に関わるサブコマンド (API) は `start-task`, `run-task` の2つがありますが、今回利用するのは [`ecs run-task`](https://docs.aws.amazon.com/cli/latest/reference/ecs/run-task.html) です。

subcommand | description
---|---
[ecs run-task](https://docs.aws.amazon.com/cli/latest/reference/ecs/run-task.html) | タスクを作成し、実行する
[ecs start-task](https://docs.aws.amazon.com/cli/latest/reference/ecs/start-task.html) | run-task と基本的には同じもの。タスクをどのコンテナインスタンスの上で実行するか、明示的に指定可能

また、ECS Run Task API は結果整合です。状態の変更がただちに AWS 側のリソースに反映されるとは限らないことに留意してください。Run Task の後続処理として何某かの処理を挟む必要がある場合は、Run Task 実行後に describe-tasks を繰り返し実行し、タスクの状態を確認します (exponential backoff で、間隔の最大値は5分ほど) 。 desctibe-tasks が意図した応答を返した場合でも、後続処理の前に待機時間を挟んでおくとよいでしょう。

run-task のパラメータ仕様は [params-run-task.md](params-run-task.md), 入力となる json の雛形は [cli-input-templates/run-task-input.json](cli-input-templates/run-task-input.json) です。

### タスクの実行 (CLI)

TODO: 書く

## 演習課題

Boyacky の稼働が開始され、本格的に運用を考える必要が出てきました。さしあたってアプリケーションの監視設定が何も入っていないため、まずはログ監視の導入を検討しています。

当面の方針として、アプリケーションログは CloudWatch Logs に集約することにしました。サービスを更新や、必要に応じてその他の AWS 環境の設定を変更し、ログ監視の仕組みを実装してください。

- ゴール
  - アプリケーションのログを CloudWatch Logs から閲覧できること
  - 手順書のドラフトとなる情報が残るように、作業内容のメモを残す
    - 作業内容と前提条件をテキスト起こしできると良い。テーブル(Remo)単位で作成してもOK
- 条件
  - サービス更新に関わる作業は AWS CLI で行うこと
  - ECS (サービス) 以外の部分に環境の変更作業が発生する場合は、手作業による変更を行って良い

目安： 1 〜 1.5h（解説含）

## サービスの削除 (アンデプロイ)

サービスを消す前に、タスクの起動数を 0 にする必要があります。

```bash
aws ecs update-service \
  --service boyacky-webapp-svc \
  --cluster boyacky-cluster \
  --desired-count 0
```

サービスの停止にはしばらくかかります。`describe-services` を実行して、状態を確認してみましょう。

````bash
aws ecs describe-services \
  --services boyacky-webapp-svc \
  --cluster boyacky-cluster \
  --query "services[0] | { status: status, desiredCount: desiredCount, runningCount: runningCount }"
```

runningCount が 0 になっていれば、サービスを削除できます。 `delete-service` を実行しましょう。

```bash
aws ecs update-service \
  --service boyacky-webapp-svc \
  --cluster boyacky-cluster
```
