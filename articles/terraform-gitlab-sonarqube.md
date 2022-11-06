---
title: "SonarQubeを用いたGitLabでの静的構文解析 -AWS EC2とdocker composeを用いて-"
emoji: "🌸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gitlab", "sonarqube", "ci", "terraform", "ec2"]
published: true
---

## 概要

本文書では、SonarQubeを用いてGitLabで静的構文解析を行う方法を記述する。内容としては、以下のような部分である。  

- `docker compose` を用いたGitLab・GitLab Runner・SonarQubeの構築方法
- SonarQubeを用いたGitLab上コードの静的構文解析方法。及びMergeRequest(PullRequest)の反映方法
- GitLab OAuthを用いた、SonarQubeでの認証認可

検証を容易にするため、環境構築方法としては、AWS EC2上にEC2インスタンスを立て、 `docker compose` で以下を構築するものとした。  
以下のような基盤に近い部分はTerraformで自動的に構築できるようにした。

- EC2インスタンスの構築
- GitLab・GitLabRunner・SonarQubeの `docker compose up`
- GitLab RunnerのGitLabインスタンスへの登録

ソースコードは全て[GitHub][toyamagu-2021/terraform-aws-gitlab-sonarqube]上に保管している。
最低限の知識があれば、特に以下の本文を読まなくとも、 `terraform apply` するだけで検証用の環境を立ち上げることができる。
本文書の目的は検証環境の構築手順書をまとめること及び備忘録のため、オリジナルな情報は少ない。  

## イントロダクション

### 背景

CICDパイプラインの中でコードの静的構文解析はCIに分類され、非常に重要な位置を占める。  
このように開発パイプラインの早い段階でテストを行うことの重要性は、[Shift-Left][shift-left]という概念でも強調されている。  
従来のソフトウェア開発では設計・開発・テスト・リリースというフローで構成されていたが、このテストを早い段階から行おうという動きである。  

CI中ではどのような静的構文解析が行われるべきだろうか。
よく行われることして、テストカバレッジや、プロジェクトのコーディングスタイルに合致しているかという観点でのチェックは重要である。  
最近の言語ではフォーマットや簡単な静的構文テストがコンパイラやインタープリタに同梱されている場合も多い。  

上記の一般的な構文チェックに加えて、近年ではCICD中でコードのセキュリティチェックを行う場合が多い。  
[DevSecOps][devsecops]という概念に象徴されるように、セキュリティチェックもCICD中で行われることが当たり前になってきているのである。  
セキュリティチェックの内静的構文解析に分類されるものを [SAST][sast] ( Static application security testing ) と呼ぶ。
様々なコードに用いることができるSASTツールとしては様々なものがある[[参考][best-static-code-analysis-tools]]。

1. [SonarQube][sonarqube]
1. [Checkmax SAST][cxsast]
1. [Synopsys Coverity][synopsys-coverity]
1. [SynkCode][synkcode]

ただ、有名なものでフリー版があるものは少ないようだ。本文書ではフリー版のあるSonarQubeを用いる[^sonar-qube-oss]。  
他にも、特定の言語用であれば無料のOSSがあるのも少なくないと思われるので、それを利用するのもいいだろう。  
例えばTerraformであれば[tfsec][tfsec]が該当する。

本文書では、以下のソフトウェアを用いてAWS上でSonarQubeの検証用の環境を立ち上げる。

| ソフトウェア | 用途         |
| :----------- | :----------- |
| GitLab       | VCS          |
| GitLab CI    | CI           |
| SonarQube    | 静的構文解析 |

全てSelf-hostedで立ち上げるため、必要なのはAWS上でのオンデマンドのインスタンス等の使用料金のみである。

説明は以下の順番で行う。

1. Docker Composeファイルの解説
1. Terraformコードの解説
1. SonarQubeとGitLabの連携
1. Appendix: GitLab OAuthを用いたSonarQubeの認証と、SonarQube側での認可設定方法

## Dockerファイル

本設ではGitLab・GitLab Runner・SonarQubeのDockerfile及び `docker-compose.yaml` を解説する。  
Terraform template 形式で記述し、Terraform実行時に環境に応じた変数を挿入するようにしてある。

### GitLab

GitLabはSelf-Hostedが可能なVCSである。Dockerコンテナは公式から公開されているため、そのまま利用する。  
以下に `docker-compose.yaml` の例を示す。 [公式の例][gitlab-docker-compose]と大差ない。  
いくつか注意しておく。

1. 簡単のため `latest` タグを用いるが、実際の運用の際には陽にバージョンを指定するのが望ましい。  
1. モニタリングのため、オプション設定を追記しておいた。
    - GitLabにはGrafanaが同梱されているため、有効化しておいた。 `http://<gitlab_dns>/-/grafana` でアクセスできる。
    - GitLabにはPrometheusが同梱されているため、有効化しておいた。GitLab RunnerとSonarQubeからスクレイピングしている。

```yaml
service:
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

        # For monitoring (not necessarily)
        grafana['enable'] = true
        grafana['disable_login_form'] = false
        grafana['admin_password'] = 'admin'
        # Promtheus
        prometheus['scrape_configs'] = [
          {
            'job_name': 'gitlab-runner',
            'static_configs' => [
              'targets' => ['gitlab-runner:9252'],
            ],
          },
          {
            'job_name': 'sonarqube',
            'metrics_path': 'api/monitoring/metrics',
            'authorization': {
              'type': 'Bearer',
              'credentials': 'prometheus'
            },
            'static_configs' => [
              'targets' => ['sonarqube:9000'],
            ],
          },
        ]``
```

その他、特に追加の設定はない。

## GitLab Runner

GitLab CI用のRunner。公式からDockerコンテナが配布されている。
以下に `docker-compose.yaml` を示す。注意点は以下の通り。

1. ExecuterとしてDocker executerを指定するため、 `docker.sock` をマウントしてある。
1. ローカルの設定ファイルをマウントしている。
1. Prometheusモニタリングのため、ポートを指定している。

```yaml
service:
  gitlab-runner:
    image: gitlab/gitlab-runner:latest
    volumes:
      - ./data/gitlab-runner:/etc/gitlab-runner
      - /var/run/docker.sock:/var/run/docker.sock
    ports:
      - "9252:9252" # For Prometheus
```

次に設定のための、 `config.toml` を示す。注意点は以下の通り。

1. Prometheusメトリクスの公開のため、 `listen_address` を指定している。
1. `${{RUNNER_TOKEN}}` はGitLab RunnerのAuthentication Tokenを挿入する。後述するユーザーデータでのブートストラップ時に自動的に挿入される。
1. `runners.docker.volumes` ではDooDのため、 `docker.sock` をマウントしている。
1. `runners.cache` ではs3のキャッシュ設定をしているが、オプションなので不要である。

```toml
concurrent = 1
# Prometheus monitoring
listen_address = ":9252"
check_interval = 0

[[runners]]
  name = "gitlab-aws-autoscaler"
  url = "${gitlab.url}"
  # Insert Runner Authentication Token
  token = $${RUNNER_TOKEN}
  executor = "docker"
  limit = 1
  [runners.docker]
    tls_verify = false
    image = "alpine"
    privileged = false
    volumes = [
      "/cache",
      "/var/run/docker.sock:/var/run/docker.sock",
    ]
  [runners.cache]
    Type = "s3"
    Shared = true
    Path = "runner"
    [runners.cache.s3]
      ServerAddress = "s3.amazonaws.com"
      AuthenticationType = "iam"
      BucketName = "${gitlab.runner.s3.bucket_name}"
      BucketLocation = "${gitlab.runner.s3.bucket_location}"
```

上記のRunnerをGitLabに自動登録するためのユーザーデータの概略は以下の通り。

1. `RUNNER_REGISTRATION_TOKEN` をGitLabから取得する
1. `RUNNER_REGISTRATION_TOKEN` を用いて、 `RUNNER_AUTHENTICATION_TOKEN` をGitLabから取得する。この際一部設定を行う。
1. `RUNNER_AUTHENTICATION_TOKEN` を、 `config.toml` に挿入する。

```bash
RUNNER_REGISTRATION_TOKEN=$(docker compose exec gitlab gitlab-rails runner -e production "puts Gitlab::CurrentSettings.current_application_settings.runners_registration_token")

## Fetch Runner Authentication Token
RUNNER_AUTHENTICATION_TOKEN=$(curl --request POST "http://localhost/api/v4/runners" \
  --form ...
  | jq .token)

## Insert RUNNER_TOKEN into config.toml of GitLab Runner
RUNNER_TOKEN=$${RUNNER_AUTHENTICATION_TOKEN} envsubst \
  < data/gitlab-runner/config.toml.before_envsubst > data/gitlab-runner/config.toml
```

### SonarQube

SonarQubeは公式からDockerコンテナが提供されている。このCE版を用いる。  
基本的には [公式サンプル][docker-compose-sonarqube] と同一である。  

1. `dockerfile` で [sonarqube-community-branch-plugin][sonarqube-community-branch-plugin] を同梱している。
    - CE版では1ブランチの解析のみ可能なため、このままだと不便だからである。  
    - pluginとSonarQubeバージョンの互換性には注意すること。
1. `service.sonarqube.environment` でSonarQubeの設定を行っている。設定対象は、[sonar.properties][sonar-properties] 参照。
    - `sonar.jdbc.username` が `SONAR_JDBC_USERNAME` に対応している。
    - 大半の設定をここで行うことができるが、 *SonarQubeのGUI上では反映されずわかりにくいため* 、最低限に抑えた。
1. [Prometheusメトリクス発行][sonar-monitoring]のため、 `SONAR_WEB_SYSTEMPASSCODE` を設定した。

```yaml
service:
  sonarqube:
    build:
      context: .
      dockerfile: sonarqube.dockerfile
      args:
        SONARQUBE_VERSION: 9.6.1-community
        PLUGIN_VERSION: 1.12.0
    hostname: sonarqube
    container_name: sonarqube
    depends_on:
      - sonarqube-db
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://sonarqube-db:5432/sonar
      SONAR_JDBC_USERNAME: sonar
      SONAR_JDBC_PASSWORD: sonar
      SONAR_WEB_SYSTEMPASSCODE: prometheus
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_logs:/opt/sonarqube/logs
    ports:
      - "9000:9000"
  sonarqube-db:
    image: postgres:13.8
    hostname: postgresql
    container_name: postgresql
    environment:
      POSTGRES_USER: sonar
      POSTGRES_PASSWORD: sonar
      POSTGRES_DB: sonar
    volumes:
      - postgresql:/var/lib/postgresql
      - postgresql_data:/var/lib/postgresql/data
```

SonarQubeでは内部でElasticSearchを用いているため、ホストOS上で以下の設定をしてあげること[[参考][sonarqube-issue-282]]。  
今回はユーザーデータに含まれている。

```bash
sysctl -q -w vm.max_map_count=262144
```

コンテナのビルドに用いる `sonarqube.dockerfile` は以下の通り。  
前述の通り、プラグインを拾ってきて含める形にしてある。  
[プラグイン作成者が公開しているDockerコンテナ][docker-sonarqube-with-community-branch-plugin]をそのまま拾ってきたり、Docerfileを応用しても良いと思われる。

```dockerfile
ARG SONARQUBE_VERSION
FROM sonarqube:${SONARQUBE_VERSION}

ARG PLUGIN_VERSION
ADD https://github.com/mc1arke/sonarqube-community-branch-plugin/releases/download/${PLUGIN_VERSION}/sonarqube-community-branch-plugin-${PLUGIN_VERSION}.jar /opt/sonarqube/extensions/plugins/
ENV SONAR_WEB_JAVAADDITIONALOPTS="-javaagent:./extensions/plugins/sonarqube-community-branch-plugin-${PLUGIN_VERSION}.jar=web"
ENV SONAR_CE_JAVAADDITIONALOPTS="-javaagent:./extensions/plugins/sonarqube-community-branch-plugin-${PLUGIN_VERSION}.jar=ce"
```


## Terraform

Terraformを用いた構築方法を記述する。特段特別なことはしていないため。コードの解説は省略する。

### アーキテクチャ図

以下にアーキテクチャ図を示す。

![aws-architecture](/images/terraform-gitlab-sonarqube/aws-architecture.drawio.png)

### 構築

コードを[GitHub][toyamagu-2021/terraform-aws-gitlab-sonarqube]に保管してある。  
`cd terraform`,  `terraform init && terraform apply` で動くはず。  
以下は便利なスクリプトである。 `terraform` ディレクトリ内で実行すること。

1. EC2インスタンスへのアクセス (SSM)。
    - `aws ssm start-session --target $(terraform output --raw gitlab_instance_id)`
1. EC1インスタンスへのアクセス (SSH over SSM)。
    - `ssh -i ./outputs/gitlab/gitlab  -o ProxyCommand="aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters 'portNumber=%p'" ec2-user@$(terraform output gitlab_instance_id)`
1. GitLabのadminシークレットを取得
    - `bash ./scripts/fetch_gitlab_admin_secret.sh $(terraform output --raw gitlab_instance_id)`

### ブラウザアクセス

`terraform output` でアウトプットからホスト名を取得できる。 `terraform apply` を実行したホストからのみアクセス可能にしてある。
*https化していないため注意。見られて困るプライベートリポジトリのアップロードなどはしないこと*

1. GitLab: ポート80
    - user: root
    - password: 上記のfetchスクリプトで取得
1. SonarQube: ポート9000
    - user: admin
    - password: admin
1. GitLab Runner
    - GitLabの (Admin Area) -> (Runners) で online になっているのを確認しておくと安心

## SonarQubeとGitLabの連携

GitLabとの連携を設定する。
[公式ドキュメント][sonarqube-gitlab-integration]に基づく。

### グローバル設定

1. GitLabでSonarQube用マシンユーザー作成
    - (Admin Area) -> (Users) で以下画像のように作成。
      ![gitlab-sonar-user](/images/terraform-gitlab-sonarqube/gitlab-sonar-user.drawio.png)
    - Impersonation Token発行。マシンユーザー用のトークン。メモしておくこと。
      ![gitlab-sonar-user-token](/images/terraform-gitlab-sonarqube/gitlab-sonar-user-token.drawio.png)
1. デモ用GitLabプロジェクト作成
    - TemplateからGoプロジェクトを作成する。
    - (NewProject) -> (Create from template) -> (Go Micro)
    - デフォルトのGitLabインスタンスグループ直下に作成
1. GitLabグループにSonarQubeマシンユーザーアタッチ
    - ![gitlab-sonar-user-add](/images/terraform-gitlab-sonarqube/gitlab-sonar-group-user.drawio.png)
1. SonarQubeでGitLab登録
    - (Administration) -> (DevOps Platform Integrations) -> (GitLab)
    - ![sonar-global](/images/terraform-gitlab-sonarqube/sonarqube-gitlab-global.drawio.png)

### 個人設定


1. GitLabでPersonal Access Token発行
    - (Preferences) -> (Access Tokens)
    - `read_api` にチェックして発行
1. SonarQubeで登録
    - ![sonar-personal](/images/terraform-gitlab-sonarqube/sonarqube-gitlab-personal.drawio.png)
    - 以降は指示に従い、作成したSonarQubeプロジェクトを登録する。問題なければ以下のようにCIが成功する。
    - ![gitlab-sonar-ci](/images/terraform-gitlab-sonarqube/gitlab-sonar-ci.drawio.png)
1. MRでBot動作チェック
    - お試しで指摘されているcode smellを修正する。 `feat-fix-code-smell` ブランチを切って、 `develop` ブランチにMRを行う。
    - ![gitlab-fix-code-smell](/images/terraform-gitlab-sonarqube/gitlab-fix-code-smell.drawio.png)

## Appendix: GitLab OAuthを用いたSonarQubeの認証と、SonarQube側での認可設定方法

GitLabとのOAuth連携方法を記述する。
[公式ドキュメント][sonarqube-gitlab-integration-oauth]に基づく。

1. GitLab OAuth Applications作成：BaseURLの設定も忘れないこと
    - ![gitlab-oauth-app](/images/terraform-gitlab-sonarqube/gitlab-oauth-app.drawio.png)
1. SonarQube側での認証設定
    - ![sonar-oauth](/images/terraform-gitlab-sonarqube/sonarqube-gitlab-oauth.drawio.png)
1. SonarQube側での認可設定
    - ![sonnar-oauth-authorization](/images/terraform-gitlab-sonarqube/sonarqube-gitlab-oauth-authorization.drawio.png)
1. ログイン確認
    - ![sonar-oauth-login](/images/terraform-gitlab-sonarqube/sonarqube-gitlab-oauth-login.drawio.png)

## まとめ

本文書では、GitLab CIとSonarQubeを用いた静的構文解析の方法を述べた。
また、構築手順や設定方法についても、画像をもとに詳細に述べた。
概要に記述した通り、新しい内容はないと思うが、製品採用の検討の際に役だてば幸いである。

<!-- Footnotes -->

[^sonar-qube-oss]: SonarQubeはクラウド版が[Argo Rolllouts][argo-rollouts]などのOSSでも利用されているようだ。

<!-- References -->

[toyamagu-2021/terraform-aws-gitlab-sonarqube]: https://github.com/toyamagu-2021/terraform-aws-gitlab-sonarqube
[shift-left]: https://en.wikipedia.org/wiki/Shift-left_testing
[devsecops]: https://www.devsecops.org/
[sast]: https://www.mend.io/resources/blog/sast-static-application-security-testing/
[tfsec]: https://github.com/aquasecurity/tfsec
[best-static-code-analysis-tools]: https://www.comparitech.com/net-admin/best-static-code-analysis-tools/
[sonarqube]: https://www.sonarqube.org/
[cxsast]: https://checkmarx.com/
[synopsys-coverity]: https://www.synopsys.com/software-integrity/security-testing/static-analysis-sast.html
[synkcode]: https://snyk.io/product/snyk-code/
[argo-rollouts]: https://github.com/argoproj/argo-rollouts
[gitlab-docker-compose]: https://docs.gitlab.com/ee/install/docker.html#install-gitlab-using-docker-compose
[sonarqube-community-branch-plugin]: https://github.com/mc1arke/sonarqube-community-branch-plugin
[sonarqube-docker-compose]: https://github.com/SonarSource/docker-sonarqube/blob/master/example-compose-files/sq-with-postgres/docker-compose.yml
[sonar-properties]: https://github.com/SonarSource/sonarqube/blob/master/sonar-application/src/main/assembly/conf/sonar.properties
[sonar-monitoring]: https://docs.sonarqube.org/latest/instance-administration/monitoring/
[sonarqube-issue-282]: https://github.com/SonarSource/docker-sonarqube/issues/282
[sonarqube-gitlab-integration]: https://docs.sonarqube.org/latest/analysis/gitlab-integration/
[sonarqube-gitlab-integration-oauth]: https://docs.sonarqube.org/latest/instance-administration/authentication/gitlab/
[docker-sonarqube-with-community-branch-plugin]: https://hub.docker.com/r/mc1arke/sonarqube-with-community-branch-plugin/tags