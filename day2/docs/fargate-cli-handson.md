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
AWS_ACCOUNT=<your-account-id>
AWS_REGION=<your-region>  # ap-northeast-1
```

**Note**: Cloud9 の動作環境と ECS のデプロイ先となる環境の AWS アカウントおよびリージョンが一致している場合は、 Cloud9 コンソール上で以下のコマンドを実行することでセットすべき値を取得可能です

```bash
# Cloud9 console
AWS_ACCOUNT=$(curl -s http://169.254.169.254/latest/dynamic/instance-identity/document | python3 -c "import json, sys; print(json.load(sys.stdin)['accountId']);")
AWS_REGION=$(curl -s http://169.254.169.254/latest/dynamic/instance-identity/document | python3 -c "import json, sys; print(json.load(sys.stdin)['region']);")
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

まずは CLI のパラメータ仕様を見ていきましょう。

Docs:

- [AWS CLI - ecs register-task-definition](https://docs.aws.amazon.com/cli/latest/reference/ecs/register-task-definition.html)
- [AWS Developer guide - Amazon ECS Task Definitions](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definitions.html)

### ecs register-task-definition パラメータ仕様

Doc: [AWS CLI - aws.ecs.register-task-definition](https://docs.aws.amazon.com/cli/latest/reference/ecs/register-task-definition.html)

arg | description
---|---
--family \<value\> | タスク定義の名前
[--task-role-arn \<value\>] | Task にアタッチする IAM Role
[--execution-role-arn \<value\>] | ECS Agent がタスクを実行するときに使用する IAM Role
[--network-mode \<value\>] | タスクの中で起動する docker コンテナのネットワークモード。none/brigde/awsvpc/host のいずれか。**Fargate の場合は `awsvpc` を指定しなければならない。**
--container-definitions \<value\> | 'コンテナ定義' のリスト（後述）
[--volumes \<value\>] | 'ボリューム定義' のリスト。Docker volume か、 EFS のストレージを使う場合に指定する ([参考](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/using_data_volumes.html))。
[--placement-constraints \<value\>] | Fargate ではサポートされない。コントロールプレーンが新しい Service/Task をどこで起動すべきか、決定するためのロジックをユーザー側で制約を付けるための項目。
[--requires-compatibilities \<value\>] | Task が満たすべき launch type を指定。 ECS/FARGATE のいずれかもしくは両方が指定可能。デフォルト値は 'ECS'
[--cpu \<value\>] | Task が使用する _CPU ユニット_ を指定。EC2 launch type ではオプショナル引数
[--memory \<value\>] | Task が使用するメモリ(MiB)を指定
[--tags \<value\>] | -
[--pid-mode \<value\>] | `docker run` の引数として使用可能な `--pid` に相当するもの
[--ipc-mode \<value\>] | `docker run` の引数として使用可能な [`--ipc`](https://docs.docker.com/engine/reference/run/#ipc-settings---ipc) に相当するもの（詳細は解説不可能）。 none/task/host が指定可能
[--proxy-configuration \<value\>] | App Mesh proxy の設定情報
[--inference-accelerators \<value\>] | Elastic Inference Accelerators を利用する場合の設定 ([doc](https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/deep-learning-containers.html))
[--cli-input-json \<value\>] | -
[--generate-cli-skeleton \<value\>] | -

タスク定義は、**どのようなコンテナが ECS で稼働するのか**を ECS に伝えます。タスク定義作成時に設定できることを日本語表現に置き換えると次のような内容になるでしょう。

- コンテナイメージ
- コンテナが使用する CPU, メモリリソース（タスク単位あるいはコンテナ単体での指定が可能）
- (未訳) The launch type to use, which determines the infrastructure on which your tasks are hosted
- Task の中で使用する Docker のネットワーキングモード
- タスクのロギング設定
- コンテナが実行終了した／fail した場合に、Task の実行は継続すべきかどうか
- コンテナが実行開始したときに、実行されるべきコマンド
- Any data volumes that should be used with the containers in the task
- タスクに対して割り当てる IAM Role

どのようなコンテナが起動するのか、あるいはコンテナ自身がどのような Spec で起動してほしいのか(what)、といった情報がタスク定義における関心事であり、そのコンテナを「どこで(where)、どのように(how)ローンチしたいのか」についてはタスク定義の概念から外れることがわかると思います。

案件のユースケースに対して、対応する設定は可能なのか／可能であるならどこで指定すれば良いのか？...そのようなケースに遭遇したら、この原則を（後述のサービス作成のパラメータ仕様とともに）覚えておくとよいでしょう。

#### Network mode 'awsvpc' (--network-mode)

EC2 起動タイプで `awsvpc` ネットワークモードを利用する場合は、コンテナインスタンスは ECS-optimized AMI を用いたものであるか、もしくは `ecs-init` パッケージをインストールした Amazon Linux である必要がある

> Currently, only Amazon ECS-optimized AMIs, other Amazon Linux variants with the ecs-init package, or AWS Fargate infrastructure support the awsvpc network mode.

#### Volume configuration (--volume)

[Developer guide (ECS) - Using data volumes in tasks](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/using_data_volumes.html)

Fargate の場合は、ボリュームとして "[Fargate Task Storage](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-task-storage.html)" もしくは "EFS" を利用可能。 Fargate Task Storage は Docker volume のようなもの。

タスク内のコンテナで共有可能なストレージを利用したい場合に検討すると良いでしょう。

##### Fargate Task Storage

[Developer guide (ECS) - Fargate Task Storage](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/using_data_volumes.html)

Fargate プラットフォームのバージョンが 1.4 (2020/10 時点の最新バージョン) の場合は、1つのタスクあたり 20GB が使用可能。上記のドキュメントで設定方法の例も記載されているので詳しくはそちらを参照。

##### Amazon EFS Volumes

[Developer guide (ECS) - Amzon EFS Volumes](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/efs-volumes.html)

#### CPU Unit (--cpu) and Memory (--memory)

Fargate の場合は、以下の通り指定可能なレンジが存在する。
 
> If you are using the Fargate launch type, this field is required and you must use one of the following values, which determines your range of > supported values for the memory parameter:
> 
> 256 (.25 vCPU) - Available memory values: 512 (0.5 GB), 1024 (1 GB), 2048 (2 GB) <br/>
> 512 (.5 vCPU) - Available memory values: 1024 (1 GB), 2048 (2 GB), 3072 (3 GB), 4096 (4 GB) <br/>
> 1024 (1 vCPU) - Available memory values: 2048 (2 GB), 3072 (3 GB), 4096 (4 GB), 5120 (5 GB), 6144 (6 GB), 7168 (7 GB), 8192 (8 GB) <br/>
> 2048 (2 vCPU) - Available memory values: Between 4096 (4 GB) and 16384 (16 GB) in increments of 1024 (1 GB) <br/>
> 4096 (4 vCPU) - Available memory values: Between 8192 (8 GB) and 30720 (30 GB) in increments of 1024 (1 GB) <br/>

#### PID Mode

`docker run` のオプション `--pid` に相当する設定項目([参考 - docker docs](https://docs.docker.com/engine/reference/run/#pid-settings---pid)

host/task のいずれかが指定可能で、 "process namespace" をホスト／タスクのスコープで共有することができます。

() 通常は、コンテナ内の PID 空間は独立しており、また共有する必要もないものです。プロセスの名前空間を共有するということは、そのプロセスに対してコミュニケーションできるということを意味するためです。しかし、それでもコンテナ自身の外の世界と PID 空間を利用したいシーンは存在し、[docker docs]((https://docs.docker.com/engine/reference/run/#pid-settings---pid)) ではデバッガを接続したい場合にコンテナとホスト間での PID 空間の共有が必要になる、と述べています（下記）。

> In certain cases you want your container to share the host’s process namespace, basically allowing processes within the container to see all of the processes on the system. For example, you could build a container with debugging tools like strace or gdb, but want to use these tools when debugging processes within the container.

※ process namespace 自体は Linux の用語であり、 [manpage](https://linuxjm.osdn.jp/html/LDP_man-pages/man7/pid_namespaces.7.html) でも解説を見ることができます。

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

まずは `create-service` の仕様を確認して、その後実際にコマンドを実行してみましょう。

### ecs create-service パラメータ仕様

Doc: [AWS CLI - aws.ecs.create-service](https://docs.aws.amazon.com/cli/latest/reference/ecs/create-service.html)

param | description
---|---
[--cluster \<value\>] |  ECS クラスタの名前
--service-name \<value\> | デプロイするサービスの名前
[--task-definition \<value\>] | 基になるタスク定義の名前
[--load-balancers \<value\>] |  このサービスに利用するロードバランサの設定。Load Balancer object のリスト (後述)
[--service-registries \<value\>] | このサービスに割り当てる Service Discovery (今回は不使用)
[--desired-count \<value\>] | 常時稼働していて欲しいタスクの数
[--client-token \<value\>] | -
[--launch-type \<value\>] | サービスの起動タイプ (EC2/FARGATE). `--capacity-provider-strategy` とは排他の関係
[--capacity-provider-strategy \<value\>] | タスクの実行基盤となる [Capacity Provider](https://docs.aws.amazon.com/cli/latest/reference/ecs/create-capacity-provider.html) をどのように選択するか、その重み付けの配分を指定。 `--launch-type` とは排他的関係
[--platform-version \<value\>] | (Fargate のみ) Fargate プラットフォームのバージョン指定。デフォルトは `LATEST` (参考: [AWS Fargate platform versions](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/platform_versions.html))
[--role \<value\>] | ECS サービスがロードバランサを呼び出すための IAM ロール。サービスにロードバランサを割り当てる場合、かつ `awsvpc` ネットワークモード**不使用**の場合（よって Fargate 起動タイプでは非該当）のみ、指定してよい
[--deployment-configuration \<value\>] | デプロイの間、どれだけのタスクを走らせておきたいかのポリシー設定。 `--desired-count` に対する最小／最大の**割合**を指定
[--placement-constraints \<value\>] | 開始するサービス／タスクをどの基盤に配置するかの制約を追加する。例えば「同種のタスクは必ず異なるコンテナインスタンスに配置する」ように指定することが可能。<br/>参考: [Amazon ECS task placement constraints](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-placement-constraints.html)
[--placement-strategy \<value\>] | サービス／タスクを配置する先をどのようにして決定するか、指定する。例えば、既に起動中のコンテナインスタンスのリソースの未使用を最大限なくすようにしたり、コンテナインスタンスあるいは AZ でできるだけ均等になるような配置戦略を指示できる。<br>[Amazon ECS task placement strategies](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-placement-strategies.html)
[--network-configuration \<value\>] | デプロイする VPC/Subnet や SecurityGroup に関する設定
[--health-check-grace-period-seconds \<value\>] | タスクが開始してから、指定秒数だけ ELB Target Health Check の結果を無視する。タスクの起動に時間がかかることがわかっている場合に意図しない Unhealthy を防止するために指定
[--scheduling-strategy \<value\>] | 起動中のタスク数を維持するためのスケジューラサービスに、どのようにタスクを維持するかのポリシーを指定する。REPLICA/DAEMON がサポートされており、**Fargate では REPLICA のみ対応**。<br/>参考: [Amazon ECS services](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs_services.html)
[--deployment-controller \<value\>] | サービスにどのデプロイ戦略を適用するかを指定する。 ECS/CODE_DEPLOY/EXTERNAL が指定可能。
[--tags \<value\>] | -
[--enable-ecs-managed-tags \| --no-enable-ecs-managed-tags] | Enable にした場合は、サービス／タスクに対して ECS が自動でタグを割り当てる。サービスに対してはクラスタ名を、タスクに対してはクラスタ名とサービス名のタグを付与する。<br/>参考: [Tagging your Amazon ECS resources](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-using-tags.html) ... "Tagging your resources for billing" を参照
[--propagate-tags \<value\>] | 実行されるタスクに対して、タスク定義のタグもしくはサービスのタグを継承したい場合に指定。 TASK_DEFINITION/SERVICE いずれかを指定可能。
[--cli-input-json \<value\>] | -
[--generate-cli-skeleton \<value\>] | -

ローンチするネットワークの設定 `--network-configuration` がサービス作成の引数として初めて出てきたことに着目してください。
**タスク定義では「どのようなコンテナが立ち上がってくるのか」といったコンテナ自身のことに主な関心**がありました。対して、サービスの関心事は **「どこで、どのようにデプロイするのか」**であることが上記のパラメータシートからもわかると思います。

#### --load-balancers

サービスをフェイシングする ELB に関する設定です。

オブジェクトのリスト形式で表現します。1つのオブジェクトは次のフォーマットを持つ json です。

```json
{
    "targetGroupArn": "<string>",
    "loadBalancerName": "<string>",
    "containerName": "<string>",
    "containerPort": "<integer>"
}
```

`--deployment-controller` などの指定値や、起動タイプ、ネットワークモードなどによって必要な設定は異なります。CLI のドキュメントから抜粋して紹介します。

- `--deployment-controller="ECS"` かつ ALB or NLB を使用している場合は複数のターゲットグループを指定可能
- `--deployment-controller="CODE_DEPLOY"` である場合
  - CLB はサポートされず ALB, NLB のどちらかを指定する必要がある
  - 2つのターゲットグループを指定する（Blue系/Green系 の切り替えを CodeDeploy が制御するために、2つのターゲットグループを利用している）
- `--deployment-controller="ECS"` の場合、ターゲットの設定情報 (Load Balancer/Target Group/Container/Port) は変更不可能 (指定を間違えた場合はサービスを作り直す必要がある)
- ALB, NLB を使用する場合は、必ず "TargetGroupArn", "Container", "ContainerPort" を指定している必要がある
  - コンテナの名前は、タスク定義の中で "ContainerDefinition" として指定した名前
  - 新しいサービスが配置されたとき、この設定内容に基づいて ALB/NLB ターゲットグループに対してターゲットを自動登録する
- CLB を利用する場合は "LoadBalancer", "ContainerPort" を指定する必要がある
- `awsvpc` ネットワークモードを使用している場合
  - ALB, NLB のみがサポートされている（よって、 Fargate サービスのフェイシングに ELB は使用できない）
  - "TargetGroup" を指定している場合、ターゲットタイプは "ip" でなければならない（"instance" は使用不可能）

#### --deployment-configuration

デプロイ中のタスク数を制御するためのパラメータです。

以下の構造を持ちます。

```json
{
    "maximumPercent": "<integer>",
    "minimumHealthyPercent": "<integer>"
}
```

`--deployment-contoroller` の設定値によってパラメータの解釈が異なります。ローリングデプロイ (`--deployment-contoroller="ECS"`) の場合を抜粋して紹介します。

- maximumPercent
  - RUNNING or PENDING 状態のタスクの最大値。デフォルトは 200%
  - `--desired-count` に対して maximumPercent を掛けて、少数以下を**切り捨て**した数が実質的なタスク数
  - ECS のサービススケジューラに対して、このパラメータでデプロイのバッチサイズを指示できる
    - 例) desired=4, maximumPercent=200 の場合
      - ECS サービススケジューラはデプロイ時に（旧タスクを停止させる前に）新しいタスクを4つデプロイすることがある（クラスタの余剰リソースが足りている場合）
- minimumHealthyPercent
  - デプロイ中に維持したい、 RUNNING 状態にあるタスク数の最小値。デフォルトは 100%
  - `--desired-count` に対して maximumPercent を掛けて、少数以下を**切り上げ**した数が実質的なタスク数
  - クラスタのキャパシティを追加せずに、デプロイできる可能性がある
    - 例) desired=4, minimumHealthyPercent=50 の場合
      - ECS スケジューラは新しいタスクを起動する前に2つのタスクを終了させ、クラスタのキャパシティを解放することがある

これらの定義の理解には、タスクのライフサイクルへの理解も必要です。公式ドキュメント [Task lifecycle](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-lifecycle.html) も参考にしてみてください。

#### --network-configuration

デプロイ先の VPC やタスクに割り当てる SG に関する設定を行います。

このパラメータは以下のような構造を持つ json オブジェクトです。

```json
{
  "awsvpcConfiguration": {
    "subnets": ["string", ...],
    "securityGroups": ["string", ...],
    "assignPublicIp": "ENABLED"|"DISABLED"
  }
}
```

サブネットは上限16個まで指定可能、SG は上限5個まで指定可能です。また、サブネットと SG は同じ VPC のリソースである必要があります。

#### --deployment-controller

"ECS" はローリングデプロイをサポートします。 `--deployment-configuration` で指定した min/max の割合に基づいて、 Healthy 状態のタスク数がその範囲に維持されるように ECS がタスクのリプレースを制御します。

"CODE_DEPLOY" では blue/green デプロイをサポートし、 "EXTERNAL" はそれ以外のデプロイ戦略を適用したい場合に指定します。これらは今回は解説できないので割愛

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

## サービスの削除 (アンデプロイ)

サービスを消す前に、タスクの起動数を 0 にする必要があります。

```bash
aws ecs update-service \
  --service boyacky-webapp-svc \
  --cluster boyacky-cluster \
  --desired-count 0
```

サービスの停止にはしばらくかかります。状態を確認してみましょう。

````bash
aws ecs describe-services \
  --services boyacky-webapp-svc \
  --cluster boyacky-cluster \
  --query "services[0] | { status: status, desiredCount: desiredCount, runningCount: runningCount }"

# {
#     "status": "ACTIVE",
#     "desiredCount": 0,
#     "runningCount": 0
# }
```

runningCount が 0 になっていれば、サービスを削除できます。 `delete-service` を実行しましょう。

```bash
aws ecs update-service \
  --service boyacky-webapp-svc \
  --cluster boyacky-cluster
```