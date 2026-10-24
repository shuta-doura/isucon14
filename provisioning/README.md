# AWS環境の構築とPR用CI

## 公開されていたAMI

競技時に案内されていたAMIは東京リージョンの `ami-0e334c50145a3ee41` です。利用できなくなっている可能性があるため、起動可能か確認したうえで使用してください。利用できない場合は、以下の手順で自分のAWSアカウントにAMIを作成します。

## 通常版AMIの手動ビルド

AMIイメージは

- 各言語の実行環境を含むベースイメージ
- アプリケーションコードを含む競技イメージ

の2つに別れています。

競技イメージはベースイメージをソースにビルドを実行するため、作成するAWS環境にベースイメージが存在しない場合、必ず一度ベースイメージの作成を実行する必要があります。  
実行環境に修正を加えた場合も手動で再度ベースイメージのビルドが必要になります。  
(所要時間20分程度かかる可能性があります)

ベースイメージを作成後、アプリケーションコードの変更のみ反映したい場合は競技イメージのビルドのみを行うことが出来ます。

```
# AMIを作成するAWS環境にログイン
aws --profile ${aws_env} sso login

# ビルド実行(ベースイメージ)
cd provisioning/packer
make base-build

# ビルド実行(アプリケーション)
cd provisioning/packer
make
```

## 前提条件

以下のものがインストールされていること
- packer
- ansible-playbook
- go-task (バックエンドのビルドのため)
- Go (バックエンドのビルドのため)
- Node.js (フロントエンドのビルドのため)
- pnpm (フロントのビルドのため)

macOSでHomebrewを使う場合、Packer、Ansible、Go Taskは以下で導入できます。`task` コマンドはHomebrewの `task` パッケージではなく `go-task` パッケージが提供します。

```sh
brew tap hashicorp/tap
brew install hashicorp/tap/packer ansible go-task
packer version
ansible-playbook --version
task --version
```

Go、Node.js、pnpmも必要に応じて用意し、`go version`、`node --version`、`pnpm --version` で確認してください。macOSでNode.jsとpnpmがない場合は `brew install node pnpm` で導入できます。
pnpm 12でフロントエンドをビルドするため、`frontend/pnpm-workspace.yaml` では `@swc/core` と `esbuild` のビルドスクリプトを許可しています。

## Go版のみのAMIを作る

Goのアプリケーションだけをビルドする場合は、AWSにログインした状態でリポジトリのルートから以下を実行します。東京リージョンにGo用のベースAMIを作り、それを元にGo用の競技者AMIを作ります。
ビルド前に変更をコミットしてください。AMI名にはビルド時の `HEAD` のコミットIDが入ります。

PackerにはAWSの認証情報が必要です。AWS CLIの `aws login` を使う場合はログイン後、Packerが読み取れる認証プロファイルを `AWS_PROFILE` に指定してください。AWS SSOを使う場合は `aws sso login --profile <プロファイル名>` でログインしてから、そのプロファイルを指定します。まず `aws sts get-caller-identity` が成功し、ビルド先のAWSアカウントであることを確認します。

```sh
export AWS_PROFILE=profile-name
aws sts get-caller-identity
make -C provisioning/packer go-ami
```

`profile-name` は自分のPacker用プロファイル名に置き換えます。`No valid credential sources found` と表示された場合は、ログイン状態と `AWS_PROFILE` を確認してください。AWS CLIの `aws login` 方式をPackerが読み取れない場合は、AWS CLIの `aws configure export-credentials --profile default --format process` を使う `credential_process` プロファイルを用意できます。

ビルドを分けて実行する場合は、最初に `make -C provisioning/packer go-base-build`、次に `make -C provisioning/packer go-build` を実行します。ベースAMIが既にある場合は `go-build` だけで構いません。`go-ami` と `go-build` は、AWS接続とローカルの必要コマンドをビルド前に確認します。Go用ベースAMIは `isucon14_gobaseimage-*`、完成した競技者AMIは `isucon14_go_contestant-*` という名前です。通常版のベースAMIとは名前を分けているため、Go版が通常版のビルドに混ざりません。

作成されたAMIのIDは以下で確認し、完成した `isucon14_go_contestant-*` のAMI IDをGitHub Actionsの変数 `EC2_AMI_ID` に設定してください。

```sh
aws ec2 describe-images --region ap-northeast-1 --owners self \
  --filters 'Name=name,Values=isucon14_go_contestant-*' \
  --query 'Images[].[ImageId,Name,CreationDate,State]' --output table
```

Go版ではベースAMIへのRust、Node.js、Perl、Ruby、PHP、Pythonのランタイム導入と、競技者AMIでのそれらのアプリケーションのビルド・起動を省きます。MySQL、nginx、Goアプリ、マッチャー、決済モックは導入します。フロントエンドとベンチマーカーのビルドには、AMIを作るローカル環境でNode.js、pnpm、Go、Taskが引き続き必要です。PackerとAnsibleも必要です。

## AMIからEC2を起動する場合

- 起動に必要なEBSは最低16GBですが、ベンチマーク中のログなどを考慮して余裕を持たせてください（競技環境では20GiB）。
- EC2終了後にルートEBSボリュームを残さない場合は、起動時の「終了時に削除」を有効にしてください。既存のAMIでは無効になっていることがあります。PR用CIは起動時に明示的に有効化します。
- 手動で接続する場合は、用途に応じてセキュリティグループのTCP/443とTCP/22を許可します。
- 適切なインスタンスプロファイルを設定するとセッションマネージャーでも接続できます。
- EC2キーペアを指定して起動した場合は、対応する秘密鍵を使って `ubuntu` ユーザーでSSH接続できます。その後 `sudo su - isucon` で `isucon` ユーザーに切り替えられます。PR用CIではEC2 Instance Connectを使うため、キーペアは不要です。

## PRごとのEC2 SSH確認CI

[`.github/workflows/isucon-ci.yml`](../.github/workflows/isucon-ci.yml) は、同じリポジトリ内のPRを作成・更新した際にEC2を1台起動し、`ubuntu`ユーザーでSSH接続を確認します。チェックアウトするのはPR先頭のコミットです。接続確認後、失敗時も含めてEC2と一時的なセキュリティグループを削除します。forkからのPRはAWS認証情報を扱わないため実行しません。SSH鍵はCI実行中に生成し、EC2 Instance Connectで一時的に公開鍵を登録するため、保存済みの秘密鍵は不要です。

GitHubリポジトリの **Settings → Secrets and variables → Actions** に以下を設定してください。

| 種類 | 名前 | 内容 |
| --- | --- | --- |
| Variable | `AWS_EC2_ROLE_ARN` | GitHub ActionsのOIDCから引き受けるIAMロールのARN |
| Variable | `EC2_AMI_ID` | `ap-northeast-1`にある競技者用AMIのID |
| Variable | `EC2_SUBNET_ID` | インターネットへルーティングできるパブリックサブネットのID |

IAMロールにはGitHubリポジトリのOIDCトークンだけを許可し、EC2のサブネット参照、セキュリティグループの作成・SSH許可・削除、インスタンスの起動・状態参照・終了、タグ作成とEC2 Instance Connectに必要な権限を付けてください。具体的には `ec2:DescribeSubnets`、`ec2:CreateSecurityGroup`、`ec2:AuthorizeSecurityGroupIngress`、`ec2:DeleteSecurityGroup`、`ec2:RunInstances`、`ec2:DescribeInstanceStatus`、`ec2:DescribeInstances`、`ec2:TerminateInstances`、`ec2:CreateTags`、`ec2-instance-connect:SendSSHPublicKey` です。起動するAMIやサブネットなどにも `RunInstances` のリソース権限が必要です。AMIにEC2 Instance Connectがインストールされていることも確認してください（Ubuntu 24.04ベースのAMIでは通常導入済みです）。

この方式でも、ワークフローを編集して実行できる共同編集者はIAMロールの権限を利用できます。IAMロールはこのCIに必要な権限だけに絞り、リポジトリへの書き込み権限を信頼できる人に限定してください。

このCIはまだベンチマークを実行しません。次の段階では、チェックアウトしたPR先頭コミットのコードをEC2へ転送し、アプリケーションを再起動してベンチマークを実行します。ワークフローはPR更新のたびに実行されますが、複数コミットをまとめてpushした場合は先頭コミットに対して1回実行されます。通常終了時はリソースを削除しますが、ランナーの強制停止などで削除ステップが実行されなかった場合は、`isucon14-pr-` で始まるインスタンスタグと `pr-ssh-` で始まるセキュリティグループ名を確認してください。

## Ansibleでの直接構築

Ubuntu 24.04のサーバーに本リポジトリを `git clone` して、Ansibleを実行できます。`make_latest_files.sh` はフロントエンドとベンチマーカーをビルドするため、実行元にNode.js、pnpm、Go、Taskが必要です。

### 競技者環境

```sh
cd provisioning/ansible
./make_latest_files.sh
ansible-playbook -i inventory/localhost application.yml
```

### ベンチマーカー環境

```sh
cd provisioning/ansible
./make_latest_files.sh
ansible-playbook -i inventory/localhost benchmark.yml
```

デフォルトではすべての言語環境をビルドします。Goのみの競技者環境にする場合は `application-base.yml` と `application.yml` の両方へ `-e go_only=true` を渡します。AMIを作る場合は上記の `go-ami` ターゲットがこの設定を渡します。

## 留意事項

- 旧CIワークフローは `disabled-workflows` に置いてあり、動作未確認です
- 作問合宿用の暫定として `trial.isucon14.net` の自己証明書を含んでいます

## フロントエンド
`frontend/Makefile` に依存
`pnpm run build` した結果の `build/client` 以下のファイルをnginxのディレクトリにデプロイしています

## API
現時点で `/api`, `/app`, `/provider`, `/chair` 宛のリクエストが 8080にproxyされるようにnginxが構成されています
エンドポイントの増減がある場合はnginxの設定ファイルの修正が必要です

APIサーバの動作に必要な環境変数は `provisioning/ansible/roles/isucon-user/templates/env.sh` で定義しています

## DB
初期化処理として `webapp/sql/init.sh` を流しています
初期化したい処理が追加になった場合は修正が必要です
