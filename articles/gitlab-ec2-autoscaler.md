---
title: "GitLab EC2 AutoScalerを用いたGitLab Self-Hosted Runnerスケーリング"
emoji: "🌸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gitlab", "gitlabrunner", "aws"]
published: false
---

## 概要

本文書では、GitLab Self-Hosted Runnerとして、EC2 Cluster Autoscalerを用いる方法を記述する。
内容としては、以下が含まれる。

- Terraformを用いたEC2インスタンスの立ち上げと、Dockerコンテナ起動
- docker composeを用いた必要なコンテナの設定
  - GitLab
  - GitLab Runner

この記事の他にも、GitLab Runner EC2 Autoscaler自体の有用性や導入事例などには日本語の解説が多くある。  
一方で、古い記事が多かったり、クレデンシャルを設定ファイルに直書きしているなど、AWSのベストプラクティスではないようにも感じる。  
本文書ではその点も踏まえて設定方法などを説明する。  
Terraformコードは全て[GitHub][github-terraform-aws-gitlab-runner-autoscaler]上に配置してあるため、applyすれば各自のAWS環境に立ち上げることができる。

尚、GitLab Runner EC2 Autoscalerのプロビジョニング自体は[Terraform module][tf-module-gitlab-runner]が公開されている。
Docker Machine自体のスケーリングも自動でしてくれるため、実際の運用ではそれをそのまま利用してもよいだろう。

## イントロダクション

### 背景

CICD基盤は特にマイクロサービスの開発・運用において重要な位置を占めており、開発効率、運用効率、ひいてはサービス自体の品質に大きく関わってくる。  
CICD ジョブ実行がボトルネックになると、ビルド・テストは実行されず、PullRequest時のレビューやMerge承認なども滞ることになる。デプロイ自体が遅延するということも想定される。  
したがって、CICD基盤はジョブの実行数に応じて柔軟にスケールできるのが望ましい。

GitLab Self-Hosted Runnerの具体的なスケール方法はどうすればよいだろうか。
一番簡単なのは、SaaSを利用することである。GitHub・GitLabではこれが利用できる。  
一方で、組織のコンプライアンス要件によりプライベートネットワーク内で実行したい場合や、スペックなどを柔軟に決めたいという場合もあるだろう。  
その場合、Runnerを自分で用意することになる。Runnerを自分で用意し、かつ柔軟にスケールしたい場合、AWSではいくつか選択肢がある。

1. [EC2 AutoScaler][gitlab-runner-ec2-autoscaler]
    - EC2インスタンスを起動し、[DockerMachine Executer][gitlab-runner-docker-machine]を用いてジョブ実行する。
    - スポットインスタンスも利用できる
    - EC2ベースなため、とっつきやすい
1. [ECS Fargate][gitlab-runner-ecs]
    - ECSクラスタを用意し、Fargateを用いてジョブ実行する
    - Fargateなため、OSの管理が不要で、ジョブ実行時間中のみ課金となる
    - 反面、特権コンテナが実行できないため、DockerコンテナのビルドにはKanikoを用いるなどの工夫が必要
1. [Kubernetes (EKS)][gitlab-runner-k8s]
    - Kubernetesクラスタを用意し、ジョブ実行する。
    - 試していないが、Fargate・Managed NodeGroup・SelfManaged NodeGroupから選択できると思われる
    - Kubernetesコントロールプレーンの運用が必要になる
      - AWSだとコントロールプレーンの費用が追加でかかる
      - ただ、導入するプロダクトも最小限で良いため、Kubernetesの運用入門・実績づくりとしては良いかもしれない

本文書では1.のEC2 AutoScalerを利用するパターンを紹介する。  
GitLabをSelf-Hostedで運用している場合は、Dockerコンテナを追加するだけで良いなど、運用も楽である。  
反面、[DockerMachine][docker-machine]は公式では開発が終了しており、[GitLabがFork][gitlab-docker-machine]しているという状況なので、将来性は若干厳しそうである。  
ただ、今でもGitLabによるメンテナスは行われているのと、 `14.x` という最近のバージョンでも `docker+machine` エグゼキュータには機能追加が行われているので、それほど採用を躊躇する理由はないかもしれない。

GitLab Runner EC2 Autoscalerのプロビジョニング自体は、[Terraform module][tf-module-gitlab-runner]が公開されている。  
DockerMachine自体もスポットインスタンスで用意してくれ、かつAutoScalingしてくれるので、そちらをそのまま利用してもよいだろう。  
本文書ではDockerMachineをコンテナとして立ち上げる方針で進めるため、若干上記モジュールとは異なっている。  
上記モジュールと比べ、Self-HostedなGitLabと同一のインスタンス上に立ち上げられるなどの特徴がある。  

### DockerMachine

詳細はよくわかっていない。  
[公式ドキュメント][docker-machine]か、[Qiita][qiita-docker-machine]の記事を参照のこと。
簡単には、Dockerホスト（DockerEngineが動くホスト環境）のプロビジョニングと管理用のツールである。
ローカルまたはリモートマシン上にDockerホストをプロビジョニングできる。

## 構成図

以下に構成図の概要を記述する。

![architecture](/images/terraform-gitlab-ec2-autoscaler/architecture.drawio.png)

## 各種設定ファイル

### Docker compose

`docker-compose.yaml` を説明する。とは言っても、たいした設定はしていない。
せいぜいがGrafanaを有効化し、RunnerからPrometheusメトリクスをスクレイピングしている程度である。

```yaml:docker-compose.yaml
services:
  gitlab:
    image: gitlab/gitlab-ee:latest
    restart: always
    ports:
      - "443:443"
      - "80:80"
    volumes:
      - ./data/gitlab/config:/etc/gitlab
      - ./data/gitlab/logs:/var/log/gitlab
      - ./data/gitlab/data:/var/opt/gitlab
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url '${gitlab.url}'

        # For monitoring
        grafana['enable'] = true
        grafana['disable_login_form'] = false
        grafana['admin_password'] = 'admin'
        prometheus['scrape_configs'] = [
          {
            'job_name': 'gitlab-runner',
            'static_configs' => [
              'targets' => ['gitlab-runner:9252'],
            ],
          },
        ]
  gitlab-runner:
    build:
      dockerfile: ./gitlab-runner-docker-machine.dockerfile
    volumes:
      - ./data/gitlab-runner:/etc/gitlab-runner
```

### `config.toml`: GitLab Runner

GitLab Runnerの設定ファイルを説明する。大体は読めばわかると思うので、[公式ドキュメント][gitlab-runner-config]に任せることとし、ポイントのみ解説する。

1. `concurrent`
    - 最大ジョブ同時実行数。
    - `runners.limit` に対して、全てのRunnerでのGlobalな実行上限である。ややこしいので[公式ドキュメント参照][gitlab-runner-vms]
1. `runners.limit`
    - 各RunnerでのDockerMachineの総数
1. `runners.cache.s3`
    - キャッシュ保存先S3バケット
1. `runners.cache.s3.AuthenticationType`
    - S3バケットへの認証・認可方法
    - 永続的なアクセストークンを記述するか、IAMロールに任せることができる
    - IAMに任せる場合、インスタンスにIAMロールを付与しておく
1. `runners.machine.IdleCount`
    - アイドル状態のEC2インスタンス数。立ち上がり時間軽減のために利用できる
    - "アイドル状態"のため、ジョブが投入されると、このセクションのインスタンス数をキープするためにインスタンスが立ち上がる
    - ただし、ジョブ実行中インスタンスとアイドル状態のインスタンスの和は、 `runners.limit` を超えない[[公式ドキュメント][gitlab-runner-idle-count]]。
1. `runners.machine.IdleScaleFactor`
    - 14.6 で導入された[実験的な機能][gitlab-idle-count-factor]。
    - 現在の実行中のマシン数に応じてIdle状態のマシンを作成できる。つまり、実行中のマシンが多ければ、Idle状態のマシンを増やせる。
    - 20個のマシンが実行中で、 `IdleCountFactor` が1.1なら、 `20x1.1=22` のマシンをアイドル中にしようとする。
    - `IdleCount` がIdle状態のマシンの上限数になる。
    - `runners.IdleCountMin` で、最小アイドルマシン数を指定できる。
1. `runners.IdleCountMin`
    - 上記の `IdleCountFactor` を補助するためのもの。Idleマシンの最低数を指定できる。
    - *1未満にはできないので注意*。1未満に設定すると1とみなされる。
1. `runners.machine.MachineOptions["amazonec2-access-key"]`, `runners.machine.MachineOptions["amazonec2-secret-key"]`
    - EC2インスタンスをプロビジョニングするためのIAMアクセスキー
    - インスタンスのIAMロールに権限が付与されていれば不要
1. `runners.machine.MachineOptions["amazonec2-iam-instance-profile"]`
    - Spotインスタンスに付与するインスタンスプロファイル名を記述する
1. `runners.machine.autoscaling`
    - cron によるRunnerプロビジョニングが可能になる
    - 実際の運用の上では、メトリクスを観察しつつつ、この部分を調整することになるだろう
    - `IdleCount`, `IdleCountMin`, `IdleScaleFactor` が設定できる。

```toml:config.toml
concurrent = 20
check_interval = 0
listen_address = ":9252"

[[runners]]
  name = "gitlab-aws-autoscaler"
  url = "${gitlab.url}"
  token = $${RUNNER_TOKEN}
  executor = "docker+machine"
  limit = 20
  [runners.docker]
    tls_verify = false
    image = "alpine"
    privileged = false
  [runners.cache]
    Type = "s3"
    Shared = true
    Path = "runner"
    volumes = [
      "/var/run/docker.sock:/var/run/docker.sock",
    ]
    [runners.cache.s3]
      ServerAddress = "s3.amazonaws.com"
      AuthenticationType = "iam"
      BucketName = "${gitlab.runner.s3.bucket_name}"
      BucketLocation = "${gitlab.runner.s3.bucket_location}"
  [runners.machine]
    IdleCount = 0
    IdleTime = 1800
    MaxBuilds = 100
    MachineDriver = "amazonec2"
    MachineName = "gitlab-docker-machine-%s"
    MachineOptions = [
      "amazonec2-region=${gitlab.runner.machine.aws_region}",
      "amazonec2-vpc-id=${gitlab.runner.machine.vpc_id}",
      "amazonec2-subnet-id=${gitlab.runner.machine.subnet_id}",
      "amazonec2-use-private-address=true",
      "amazonec2-private-address-only=true",
      "amazonec2-tags=${gitlab.runner.machine.ec2_tags}",
      "amazonec2-security-group=${gitlab.runner.machine.security_group_name}",
      "amazonec2-instance-type=${gitlab.runner.machine.instance_type}",
      "amazonec2-ami=${gitlab.runner.machine.ami}",
      "amazonec2-request-spot-instance=true",
      "amazonec2-spot-price=",
      "amazonec2-iam-instance-profile=${gitlab.runner.machine.instance_profile}",
    ]
    [[runners.machine.autoscaling]]
      Periods = ["* * 9-17 * * mon-fri *"]
      IdleCount = 10
      IdleCountMin = 2
      IdleScaleFactor = 0.5
      IdleTime = 3600
      Timezone = "Asia/Tokyo"
    [[runners.machine.autoscaling]]
      Periods = ["* * * * * sat,sun *"]
      IdleCount = 0
      IdleTime = 1800
      Timezone = "Asia/Tokyo"
```

### Dockerfile: GitLab Runner

以下にGitLab Runner用のDockerfileを記述する。  
基本的には、GitLab Runner用のDockerコンテナから出発し、DockerMachineをインストールするのみである。  

追加で怪しげなことをしているので、これを説明する。
これはコメントに書いたように、ダミーの証明書を発行しておくためである。  
DockerMachineの初回起動時にインスタンスが複数同時に立ち上がると、証明書の発行が同時に起こるため、証明書の情報がおかしな状況になるようだ。  
実際、エラーが生じて色々と大変だった。  
そのため、事前に証明書を発行し、それを同梱しておくことにする。  

```dockerfile
FROM gitlab/gitlab-runner:latest

# Install DockerMachine
RUN curl -O "https://gitlab-docker-machine-downloads.s3.amazonaws.com/v0.16.2-gitlab.11/docker-machine-Linux-x86_64"
RUN mv docker-machine-Linux-x86_64 /usr/local/bin/docker-machine
RUN chmod +x /usr/local/bin/docker-machine

# To avoid creating certificates by multiple parallel processes
## See: https://github.com/docker/machine/issues/3634#issuecomment-575082182
# As far as I tested, no need to set environment variable, USER 
### See: https://gitlab.com/gitlab-org/gitlab-runner/issues/3676
### See: https://github.com/docker/machine/issues/3845#issuecomment-280389178
RUN docker-machine create --driver none --url localhost dummy-machine
RUN docker-machine rm -y dummy-machine
```

### ユーザーデータ: GitLab Runner登録

ユーザーデータ（EC2ブートストラップスクリプト）を主要部分のみ説明する。
ユーザーデータ内でGitLab Runnerの登録まで実施している。以下のステップから成る。

1. GitLabからRegistrationトークンを取得する
    - 特にAPIなどは無いようなので、`docker exec` を発行して無理やり取得している。
1. Registrationトークンをもとに、GitLabにRunnerを登録し、Authenticationトークンを発行する。
    - もともと発行されていたRegistrationトークンがあれば、1.を経由せずにこのコマンドを実行できる。
1. Autheniticationトークンを GitLab Runner の `config.toml` に挿入する。

```bash:user-data.sh
## Fetch Runner registration token
RUNNER_REGISTRATION_TOKEN=$(docker compose exec gitlab gitlab-rails runner -e production "puts Gitlab::CurrentSettings.current_application_settings.runners_registration_token")

## Fetch Runner Authentication Token
RUNNER_AUTHENTICATION_TOKEN=$(curl --request POST "http://localhost/api/v4/runners" \
  --form "token=$${RUNNER_REGISTRATION_TOKEN}" \
  ... options
  | jq .token)

## Insert RUNNER_TOKEN into config.toml of GitLab Runner
RUNNER_TOKEN=$${RUNNER_AUTHENTICATION_TOKEN} envsubst \
  < data/gitlab-runner/config.toml.before_envsubst > data/gitlab-runner/config.toml
docker compose up -d --wait gitlab-runner
```

### Terraform

主要部分のTerraformコードの解説を行う。

#### GitLab Runnerインスタンス（DockerMachine）用IAMロール

以下の権限が必要。

- EC2
  - EC2インスタンスプロビジョニングのため
- S3バケットアクセス
  - キャッシュ保存のため
- PassRole
  - 作成したインスタンスにIAMロールを付与するため

```terraform
// DockerMachine用
data "aws_iam_policy_document" "instance_machine" {
  statement {
    sid       = ""
    effect    = "Allow"
    resources = ["*"]

    actions = [
      "ec2:DescribeKeyPairs",
      "ec2:TerminateInstances",
      "ec2:StopInstances",
      "ec2:StartInstances",
      "ec2:RunInstances",
      "ec2:RebootInstances",
      "ec2:CreateKeyPair",
      "ec2:DeleteKeyPair",
      "ec2:ImportKeyPair",
      "ec2:Describe*",
      "ec2:CreateTags",
      "ec2:RequestSpotInstances",
      "ec2:CancelSpotInstanceRequests",
      "ec2:DescribeSubnets",
      "ec2:AssociateIamInstanceProfile",
    ]
  }

  statement {
    sid       = ""
    effect    = "Allow"
    resources = [aws_iam_role.machine.arn]
    actions   = ["iam:PassRole"]
  }
}

// S3キャッシュ保存用
data "aws_iam_policy_document" "runner" {
  statement {
    # Ref: https://docs.gitlab.com/runner/configuration/advanced-configuration.html#the-runnerscaches3-section
    sid       = "allowGitLabRunnersAccessCache"
    effect    = "Allow"
    resources = ["${aws_s3_bucket.runner_cache.arn}/*"]

    actions = [
      "s3:PutObject",
      "s3:GetObject",
      "s3:GetObjectVersion",
      "s3:DeleteObject"
    ]
  }
}
```

#### Docker Runner用IAMロール

特に要件は無いため、ここには記載しない。  
SSM SessionManager用の権限を付与したのみである。

## プロビジョニング・実行テスト

実際にジョブを実行してみる。

1. Terraform実行
    - `terraform apply`
1. GitLabにアクセス
    - `terraform output` でホスト名がわかる
    - IPアドレスをTerraformが実行されたインスタンスのみに制限している。
    - `root` パスワードは、 `bash ./scripts/fetch_gitlab_admin_secret.sh $(terraform output --raw gitlab_instance_id)` で取得できる。
1. Runnerインスタンス確認
    - Admin -> Overview -> Runners でグリーンになっていることを確認する。
1. GitLabでCICD実行
    - GitLabプロジェクトテンプレートには `Pages/GitBook` があるため、これを利用するとよい。
    - EC2インスタンスが自動的に立ち上がり、ジョブの実行が正常に行われ、キャッシュがS3に保存されることを確認する。
1. Grafanaでのメトリクス取得
    - `http://<gitlab_dns>/-/grafana` でアクセスできる。メトリクスを見てみると、面白いかもしれない。
1. リソースの削除
    - `terraform destroy`
    - *残念ながらRunnerスポットインスタンスは自動的に削除されないため、手で削除すること。*

## まとめ

Terraformを用いてGitLab・GitLab Runnerを立ち上げ、DockerMachineを用いてRunnerをスケールする方法を解説した。  
また、検証のためのTerraformコードを添付した。
Kubernetes Runnerの登場により採用優先度は下がるかもしれないが、小さい組織の場合、費用対効果はかなり高いと思われる。

<!-- References -->

[tf-module-gitlab-runner]: https://github.com/npalm/terraform-aws-gitlab-runner
[gitlab-runner-ec2-autoscaler]: https://docs.gitlab.com/runner/configuration/runner_autoscale_aws/
[gitlab-runner-ecs]: https://docs.gitlab.com/runner/configuration/runner_autoscale_aws_fargate/
[gitlab-runner-k8s]: https://docs.gitlab.com/runner/executors/kubernetes.html
[gitlab-runner-docker-machine]: https://docs.gitlab.com/runner/executors/docker_machine.html
[docker-machine]: https://docs.docker.jp/machine/overview.html#what-is-docker-machine
[gitlab-runner-config]: https://docs.gitlab.com/runner/configuration/advanced-configuration.html
[gitlab-runner-vms]: https://docs.gitlab.com/runner/configuration/autoscale.html#how-concurrent-limit-and-idlecount-generate-the-upper-limit-of-running-machines
[gitlab-runner-idle-count]: https://docs.gitlab.com/runner/configuration/autoscale.html#how-concurrent-limit-and-idlecount-generate-the-upper-limit-of-running-machines
[qiita-docker-machine]: https://qiita.com/etaroid/items/40106f13d47bfcbc2572
[github-terraform-aws-gitlab-runner-autoscaler]: https://github.com/toyamagu-2021/terraform-aws-gitlab-runner-autoscaler/
[gitlab-idle-count-factor]: https://docs.gitlab.com/runner/configuration/autoscale.html#the-idlescalefactor-strategy
[gitlab-docker-machine]: https://gitlab.com/gitlab-org/ci-cd/docker-machine/

<!-- Footnotes -->
