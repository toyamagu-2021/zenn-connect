---
title: "Terraformで作る、GitHubとArgoCDを用いたKubernetes(EKS)へのCICD"
emoji: "🌸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kubernetes", "github", "cicd", "argocd", "eks"]
published: true
---

## 概要

Kubernetes (K8s) や microservice を最大限に活用する上で CICD 基盤は欠かせない。  
本文書では、K8s基盤として[EKS][eks]を、CIとして[GitHub Actions][github-actions]を、CDとして[ArgoCD][argo-cd]を、IaCとして[Terraform][terraform]採用した場合のアーキテクチャの例を解説する。  
特に、プロダクションレベルへの拡張を想定し、以下観点を重視して設計する。

- 組織の拡大に従ってスケールするアーキテクチャとなっているか。すなわち、人が介入する作業を減らし、自動化できているか
- 可能な限りすべてのリソースは宣言的に作成されているか
  - Single Source of Truth を設定し、設定・構成を一箇所に集約するため
  - リソースのドリフトを検出するため
- CICD基盤の認証認可は、組織の権限統制を反映しているか

本文書で記述する内容は以下の通り。

- GitHub Actionsを用いたCIアーキテクチャの例
  - ブランチ戦略
  - GitHub Organizationを用いた認証・認可
  - GitHub Appsを用いたトークン取得・他リポジトリへの書き込み
- ArgoCDを用いたCDアーキテクチャの例
  - GitHub OAuth Appsとの認証連携
  - ArgoCD RBACを用いた認可
  - ApplicationSetを用いたApplicationの一括デプロイ
- Terraformを用いたリソースプロビジョニング
  - [GitHub provider][terraform-github-provider]を用いたGitHubリソースのプロビジョニング
  - [AWS EKS module][terraform-eks-module]を用いたEKSクラスタのプロビジョニング
  - [Helm provider][terraform-helm-provider]を用いたArgoCDのインストール
    - ArgoCDのインストールにはパラメータ設定の簡便さ、Terraform providerの利用しやすさを重視して[ArgoCD Helm chart][argocd-helm]を採用する[^argocd-helm]

本文書では主に設計思想や設計内容を解説する。構築手順の詳細な解説も時間があれば書きたい。  
コード群や簡略手な構築手順は、GitHub Organizationも含めてGitHubに公開している[toyamagu-cicd]。  
質問・指摘を歓迎します。

## イントロダクション

### 背景

CICDは以下観点から現在のアーキテクチャにおいて重要なものとなっている

- ビルド・テスト・パブリッシュ・デプロイを自動化しリリーススピードをはやめる
- テストを自動的に都度行うことで、品質の改善を行う
- 自動化を行うことによって、ビジネスの拡大に従って、サービスをスケールする
  - 特に大量のコンテナをデプロイするK8s等のコンテナオーケストレータを採用する場合、自動デプロイの仕組みがないと人的コストがかかる

ただし、CICDは以下に注意が必要である

- ブランチ・リポジトリ戦略を前もって定めないと、維持年数の経過や基盤拡大に従って基盤を管理しきれなくなり、運用が困難になる。
- CICDをトリガーするソースとなるGitHubイベントに応じて、実環境へのデプロイが自動的に行われる
  - 権限統制の仕組みづくりをインフラストラクチャレベルで考慮しておかないと、デプロイ事故・不正に繋がる

本文書では上記を加味した上で、設計・構築を行う。登場リソースは以下の通り。

| リソース            | 説明                 |
| :------------------ | :------------------- |
| GitHub              | ソースコード管理     |
| GitHub Organization | チーム管理           |
| GitHub Actions      | CI                   |
| ArgoCD              | CD                   |
| AWS                 | クラウド             |
| EKS                 | Kubernetesクラスター |

### 予備知識

本文書を読む上での予備知識を簡単にまとめる。

- GitHub Apps
  - 本文書の範囲では、CICDでリポジトリ間の書き込みを行うための、マシンユーザーだと思って良い。
  - 単純なマシンユーザーと比較して、ユーザーの枠を専有しない・一時トークンの発行が楽など多くのメリットが有る[[解説][zenn-github-app]]。
  - 許可しているアクションは以下の通り。
    | リソース     | アクション     | 説明                                        |
    | :----------- | :------------- | :------------------------------------------ |
    | Contents     | Read and Write | K8sマニフェストアップデートのため           |
    | Pull Request | Read and Write | K8sマニフェストリポジトリでPRを発行するため |
- [Kustomize][kustomize]
  - K8sマニフェストの環境差分管理に用いる
- ArgoCD
  - 本文中で概要レベルで記述する
- NginxIngress
  - インターネットアクセスに用いる

## アーキテクチャの説明

本節ではアーキテクチャを様々な観点から説明する。

### CICD全体像

本小節では全体像を解説する。本節は全体像を説明することに注力し、実際にどのように実現するかは、後の節で詳しく説明する。

以下図にCICDアーキテクチャの全体像を示す。

![cicd-overall-architecture](/images/cicd-eks-github-argocd/cicd-pipelinie.drawio.png)

ポイントは以下の通り。

1. GitHub Organizationを作成し、Organization内部に以下リポジトリを作成する。
    - `argocd-cicd-application`: K8sにデプロイするアプリケーションソースコード用のリポジトリ
    - `argocd-k8s-manifest`: K8sにデプロイするリソースのK8sマニフェスト用のリポジトリ
    - `terraform-argocd-cicd`: リソースプロビジョニング用のTerraformコード用のリポジトリ
1. `argocd-cicd-application` はブランチへのpushをトリガーとして以下のようなCICDを行う。
    - アプリケーションのビルドを行う
    - コンテナリポジトリ (ECR) にビルドされたコンテナをpublishする
    - publishされたコンテナをデプロイするために、 `argocd-cicd-k8s-manifest` リポジトリのK8sマニフェストを書き換え、CDツールにデプロイを委任する
      - 書き換え対象は主にコンテナタグである。CDツールに変更を伝えるために、コミットハッシュを用いて上書きするのが一般的。
1. ArgoCDは以下のようなCDを行う。
    - `argocd-cicd-k8s-manifest` リポジトリを監視し、K8sマニフェストの変更に応じてEKSにデプロイを行う

実際の運用の際は `argocd-cicd-k8s-manifest` リポジトリでも[Open Policy Agent][opa]や[Conftest][conftest]等を用いたK8sマニフェストのコンプライアンスチェックなどを行うのが良いだろう。  
今回はスコープ外とする。

以下に注意点を記述する。

1. アプリケーション用のリポジトリと、K8sマニフェスト用のリポジトリを分ける理由
    - 開発者・運用者の認知負荷を下げるため
    - 複数のアプリケーションのマニフェストをまとめるため
    - 不要なCICDのトリガーを避けるため
      - GitHubイベントに応じてCICDをトリガーする際、同じリポジトリだと（作り込みを行わない限り）関係ないリソースの変更に応じてCICD全体がトリガーされてしまう
      - 最悪の場合無限ループに陥る
    - 権限統制のため
      - Gitリポジトリの権限分割は、リポジトリ単位・ブランチ単位がある。GitHub・GitLab等の主要なVCSにおいて、認可用のロールはリポジトリ単位で振られる
        - リポジトリ単位で分けると運用者・開発者ごとにロールを振ることができ、認可設計し易い
      - CDツールのアクセス対象を分割するため
    - その他、[ArgoCDのベストプラクティス][argocd-best-practices]に詳述されているため参照

### GitHub

本小説ではGitHub関連の情報を記述する。

#### ブランチ戦略

本小節ではGitHubブランチ戦略、またCIに関連するGitHubイベントを説明する。  

ブランチ戦略を複雑にすれば権限統制やリリース管理は厳密になるが、一方で運用負荷が上昇する。変更の取り込み漏れなどの、ミスが発生するもとにもなる。  
このあたりの事情は[Git Flow][git-flow]・[GitHub Flow][github-flow]・[GitLab Flow][gitlab-flow]・[Trunc base flow][trunc-base-flow]などの有名な戦略を学び、自身のプロダクトに合ったものを採用するのがよいだろう。  
それぞれの戦略の比較としては、GitLab Flowの序文がわかりすい。個人的にではあるが、大まかな指針としては、例えば以下のような方針があると考えている。

- リリースごとにパッチ等の長期メンテナンスが必要になるアプリケーションなら、Git Flowベースの重厚なブランチ戦略をベースにする。
- 特に古いバージョンのメンテナンスが必要ないWebアプリケーションなら、GitHub Flowベースの軽量なブランチ戦略をベースにする。  

CICDという観点ではGitLab Flowをベースにするのがわかりやすいと思うので、本文書ではGitLab Flowをベースに進める。
GitLab Flowを簡素化したものに、CIプロセスを加えた図を以下に示す[^gitlab-flow]。

![git-branch](/images/cicd-eks-github-argocd/git-branch.drawio.png)

ポイントは以下の通り。

1. アプリケーション用のリポジトリ (`argocd-cicd-application`)
    - 各デプロイ先環境にリリースされるアプリケーションコードを格納するブランチを用意する。
      - 図では `main` (本番環境), `dev` (開発環境)とした。
      - デプロイ先環境に対応するブランチは `main` ブランチへのマージなどによっても消去せず、永続化する。
        - 誤って消去しないよう、ブランチ保護などを用いて適切に保護する。
    - 機能実装用の一時的なブランチを `feature/functionX` と仮定する。機能実装・テスト後 `argocd-cicd-application`/`dev` ブランチにマージする。
      - Merge後 `feature/functionX` ブランチは削除する。
      - `argocd-cicd-application`/`dev` ブランチへのマージによってCIが動作する。
        - `argocd-cicd-k8s-manifest`/`dev` ブランチへのpushを行う(図の緑線)[^k8s-manifest-push]。
      - `argocd-cicd-k8s-manifest`/`dev` ブランチはCDツールによって監視されているため、 開発環境に新規アプリケーションのデプロイが行われる。
    - 開発環境でのテスト後本番環境にリリースするため、 `argocd-cicd-application`/`dev` ブランチから `argocd-cicd-application`/`main` ブランチへのPullRequestを行う。
      - レビュー後、PullRequestをMergeする。
      - `argocd-cicd-application`/`main` ブランチへのマージによってCIが動作する。
        - `argocd-cicd-k8s-manifest`/`dev` ブランチをcheckoutする。`argocd-cicd-k8s-manifest`/`main` ブランチへのPRを行う (図の赤線) [^cicd-checkout-k8s-manifest]。
        - レビュー後PRをMergeする。
        - Merge後一時的なブランチは削除する。
      - `argocd-cicd-k8s-manifest`/`main` ブランチはCDツールによって監視されているため、 本番環境に新規アプリケーションのデプロイが行われる。
2. K8sマニフェストリポジトリ
    - 各デプロイ先環境にデプロイするK8sマニフェストを格納するブランチを用意する。これらは永続化する。
      - 図では `main` (本番環境), `dev` (本番環境以外)とした。
    - K8sマニフェストの環境分けは Kustomize を用いて行うため、 `dev` ブランチには本番環境以外のすべてのマニフェストを格納する[^kustomize-branch]。
    - 各ブランチはCDツールによって監視されているため、各ブランチへのPush(Merge)は実環境へのデプロイにつながる。
    - Kustomizeのディレクトリ構造は、例えば以下のようにする[^kustomize-directory]。

      ```console
      ├── base
      │   ├── configMap.yaml
      │   ├── deployment.yaml
      │   ├── kustomization.yaml
      │   └── service.yaml
      └── overlays
          ├── prd
          │   ├── .env
          │   └── kustomization.yaml
          └── dev
              ├── kustomization.yaml
              └── .env
      ```

以下に注意点を記述する。

1. 環境差分の吸収方針
    - 開発環境・本番環境の環境差分は、ConfigMapやSecretなどのK8s機能を用いて吸収する。
    - 換言すれば、K8sマニフェストリポジトリの `overlays/{prd|dev}/.env` ファイルなどで吸収する。
      - [12 factor app][12factor-app-config] (若干古いが)にもあるベストプラクティスである 。
    - *アプリケーションリポジトリでブランチ分割しているからとしてそちらで環境差分を吸収しないこと*。
      - 環境毎にアプリケーションコードに差分が存在するのは、悪夢である。
1. Kustomizeを利用しているが、環境毎にHelm Chartをデプロイしたい
    - KustomizeはHelmをサポートしている。`kustomize build` 時にオプションを有効にする必要があるので、ArgoCDの設定で有効化すること。
1. コンテナタグ
    - CDツールはマニフェストの差分を検知するため、コンテナタグには各コミットで一意なものを用いること
    - `latest` などは変更なしとみなされデプロイされない
    - 開発環境ではコミットハッシュを利用するのが最も簡単であるため、コミットハッシュをそのままタグに利用すればよいだろう
    - 本番環境でもコミットハッシュを用いるのも一つの手である本文書中では簡単のためコミットハッシュにしてある
      - ただし、CICDのトリガーをmainブランチへのMergeからReleaseタグの発行に変更し、 `vx.y.z` などをタグ付けに利用するのが一番認知不可が低いのではないかと言うのが個人的な見解である

#### CICD Secret管理

本小説ではCICD Secret管理について記述する

GitHub Action実行時にも資格情報が必要になる。例えば以下の用途があるだろう。

- CI中でのアーティファクトプッシュのための、アーティファクトリポジトリの認証情報
- CI中での統合テストのために、クラウドの認証情報
- DBテストのための、DBの認証情報

Secretの保存先は以下の2通りがある。

1. Repository Secret
    - GitHubリポジトリに直接Secretを保管する
    - 特定リポジトリのみ使用するSecret情報はこちらで良い。リポジトリの管理者にSecret利用権限を委任したい場合なども利用できる。
1. Organization Secret
    - GitHub OrganizationにSecretを保管し、リポジトリにアタッチする
    - Organization中で共有する場合に有用

本文書のアーキテクチャでは以下の認証情報が必要である。実際のコードは [コード例](#CICDコード例)参照。

1. ECRリポジトリの認証情報
    - AWSの認証情報
    - 一時的な認証情報を利用するため、[GitHub OIDC][configuring-openid-connect-in-amazon-web-services]を用いる
    - IAMロールのARNをRepository Secretとして登録する
      - 特に秘匿する必要はないが、外部から変数を与える方法がSecretしかないため
1. K8sマニフェストリポジトリのための認証情報
    - GitHubの認証情報
    - 一時的な認証情報を利用するため、GitHub Appsを用いる
    - Organization内で共有するため、GitHub App秘密鍵をOrganization Secretとして登録する
### CICDコード例

[toyamagu-cicd/argocd-cicd-application][toyamagu-cicd-argocd-cicd-application] に設置した。
コアな部分は以下である。  

- AWS 資格情報取得
  - AWS IAM OIDCプロバイダー機能を用いて、[GitHub OpenID Connect][configuring-openid-connect-in-amazon-web-services] と連携しているため、資格情報の発行不要
  - [参考記事][zenn-github-actions-support-openid-connect]


  ```yaml
  - name: Configure AWS credentials
    uses: aws-actions/configure-aws-credentials@v1-node16
    with:
      role-to-assume: ${{ secrets.IAM_ROLE_ARN }}
      aws-region: ${{ env.AWS_REGION }}
  ```

- GitHub App トークン取得
  - K8sマニフェストリポジトリへの書き込みのために、トークンを取得する。

  ```yaml
  - name: Generate token
    id: generate-token
    uses: tibdex/github-app-token@v1
    with:
      app_id: ${{ secrets.APP_ID }}
      private_key: ${{ secrets[format('PEM_{0}', secrets.APP_ID)] }}
  ```

- Login と Build Image
  - タグにコミットハッシュを指定
  - プレフィックスにブランチ名を指定
    - 勿論、実際の運用ではECRリポジトリを分けるべきだし、そもそも、本番と開発のECRリポジトリでAWSアカウント自体を分けるべきだろう。

  ```yaml
  - name: Login to Amazon ECR
    id: login-ecr
    uses: aws-actions/amazon-ecr-login@v1

  - name: Build, tag, and push image to Amazon ECR
    id: build-image
    env:
      ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
      working-dir: "."
    run: |
      TAG_PREFIX=$(echo ${{github.ref_name}} | sed 's/[\/#]/-/g')
      CONTAINER_REPO="${ECR_REGISTRY}/${{ env.ECR_REPOSITORY }}"
      CONTAINER_TAG="${TAG_PREFIX}-${{ github.sha }}"
      CONTAINER_NAME=${CONTAINER_REPO}:${CONTAINER_TAG}
      docker build -t ${CONTAINER_NAME} .
      docker push ${CONTAINER_NAME}
      echo "::set-output name=container-repo::${CONTAINER_REPO}"
      echo "::set-output name=container-tag::${CONTAINER_TAG}"
  ```

- Update K8sマニフェスト
  - `env.APPLICATION_DIR_PREFIX` でパスのプレフィックスを指定
  - `target-dir` でアプリケーションリポジトリのブランチに応じて書き換え先を指定
    - `main` ブランチ -> `overlays/prd`
    - `dev` ブランチ -> `overlays/dev`

  ```yaml
  - name: Update K8s manifest
    env:
      CONTAINER_TAG: ${{needs.build-and-publish.outputs.container-tag}}
    run: |
      kustomize edit set image sample-app="*:${CONTAINER_TAG}"
    working-directory: "${{ env.APPLICATION_DIR_PREFIX }}/overlays/${{ steps.set-target-branch.outputs.target-dir }}"
  ```

- ブランチに応じてK8sマニフェストリポジトリにpushかPR
  - `dev` -> K8sマニフェストリポジトリ/`dev` ブランチにpush
  - `main` -> K8sマニフェストリポジトリ/`dev` ブランチをチェックアウト、 K8sマニフェストリポジトリ/`main` ブランチにPR

  ```yaml
  - name: Push
    if: ${{ steps.set-target-branch.outputs.target-branch == 'dev' }}
    run: |
      git push origin ${{ steps.set-target-branch.outputs.target-branch }}
  
  - name: Create Pull Request
    if: ${{ steps.set-target-branch.outputs.target-branch == 'main' }}
    id: cpr
    uses: peter-evans/create-pull-request@v4
    ...
  ```

### ArgoCD

本小節ではArgoCDによるCD方法を記述する。

#### ArgoCD概要

ArgoCDはGitOpsに基づきK8sクラスタにデプロイを行う[CNCF Incubating Project][cncf-argocd]である。  
[GitHub][github-argocd]にコードが公開されている。  
他のメジャーなCDツールの比較の概要を以下に示す。

- K8s以外のデプロイにも利用できる[Jenkins][jenkins] や [Sppinaker][spinnaker] と比較して、K8sに特化している分簡単に利用できる。
  - 主観だが、GitOpsの思想にもArgoCDのほうがより忠実であると感じる。
- K8sに特化したツールであるFluxCDよりは機能が豊富
  - ユーザー管理など
- [ArgoWorkflows][argo-workflows]・[ArgoRollout][argo-rollout] など、周辺プロダクトも優秀。

ArgoCDの基本的なリソースと概要を以下表に示す。詳細は[公式ページ][argocd]参照のこと。

| リソース       | 概要                                                                                                                         |
| :------------- | :--------------------------------------------------------------------------------------------------------------------------- |
| Application    | GitOps対象のリポジトリを指定するArgoCD CRD                                                                                   |
| Project        | マルチテナント時の権限分割に用いる[^argocd-project]。Applicationのまとまりの権限境界となる                                   |
| ApplicationSet | ApplicationをTemplate化し、一括で生成する。例えば、GitHubリポジトリパスのパターンに合致するパスのAppliactionを一括作成できる |

典型的な Application + Project のみを利用した構成例を以下図に示す。

![argocd-application](/images/cicd-eks-github-argocd/argocd-application.drawio.png)

ポイントは以下の通り

- Project `prd` は `prd` NSへのデプロイを許可されたProjectである。
- Application `sample-app` は以下に設置されたマニフェスト (`kustomizaton.yaml) を継続的に監視し、変更を反映してデプロイする。
  - リポジトリ: `https://github.com/<user_name>/k8s-manifest-repo.git`
  - パス: `application/sample-app/overlays/prd`
- ArgoCDは、 `kustomization.yaml` の内容に応じて、EKSクラスタに `Service` や `Deployment` がデプロイする

yamlの例としては以下のようになる。

```yaml: application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: prd

  source:
    repoURL: https://github.com/toyamagu-cicd/argocd-cicd-k8s-manifest.git  # Can point to either a Helm chart repo or a git repo.
    targetRevision: main  # For Helm, this refers to the chart version.
    path: application/sample-app/overlays/prd

  # Destination cluster and namespace to deploy the application
  destination:
    server: https://kubernetes.default.svc
    # The namespace will only be set for namespace-scoped resources that have not set a value for .metadata.namespace
    namespace: prd
```

その他GitHub WebHookとの連携やSlack通知など、そこそこ簡単にでき実際の運用では重要な機能もあるが、本筋と関係ないため省略する。

#### ApplicationSetを用いたリソースの一括デプロイ

ArgocdApplicationSet CRDは、Applicationのtemplateを一括でプロイする仕組みである[^argocd-applicationset-go-template]。  
従来、Applicationの一括管理はApp of Appsパターンなどが用いられてきたが、今回はApplicationSetを利用する[^argocd-app-of-apps]。  
ArgoCD v2.3からArgoCD AppliactionSetは標準インストールに同梱されており、気軽に利用することができる。  
ArgoCD ApplicationSet機能には以下のメリットがある。ArgoCD公式ドキュメントの [monorepos][monorepos]ユースケース例も参照すると良い。

- 複数のApplicationを一括でプロイできる
  - マルチクラスタ・マルチNSを指定可能
  - クラスタにbootstrapで複数のリソースをデプロイしたいとき、手間が省ける[^argocd-applicationset-bootstrap]
  - Gitリポジトリパスパターンマッチを利用して、パターンにマッチする複数のパスのApplicationを一括デプロイできる
- 宣言的なApplicationの作成を強制できる
  - Applicationの直接の作成を禁止し、Gitリポジトリに設置されたマニフェストのみデプロイ可能とする
  - GitリポジトリのSingle source of truthとしての信頼性が上昇する
- Templateの一部をハードコーディングすることで、要件に応じたApplicationの作成が保証できる
  - 例えば、Proectを `dev` とすることで、作成されるApplicationのProjectを `dev` に制限できる

上記により、CDツールを採用することでの利便性を享受しつつ、権限統制要件に応じたリソースデプロイが可能になる[^argocd-applicationset-git-generator]。  

前置きが長くなったが、以下にArgoCDのアーキテクチャ図を示す。  

![argocd-applicationset](/images/cicd-eks-github-argocd/argocd-applicationset.drawio.png)

ApplicationSet `boostrap` のみ説明すれば残りの意味も掴めると思うので、 `bootstrap` のみに絞って解説する。

- 用途
  - クラスターのboostrapに必要なリソースを一括でプロイする
  - 今回の例ではインターネットアクセス用の[NginxIngress][ingress-nginx]を対象としている。
- Gitリポジトリ監視対象
  - リポジトリ: [toyamagu-cicd/argocd-cicd-k8s-manifest][toyamagu-cicd-argocd-cicd-k8s-manifest]
  - `main` ブランチ
    - Bootrapに用いる重要なアプリケーションのため、運用管理者が作成する前提とする。後の権限統制も参照。
  - `application/bootstrap/*` パス
    - `application/bootstrap/nginx-ingress`
      - [URL][github-k8s-manifest-nginx-ingress]
      - NginxIngress Helm chartをデプロイする
    - `application/bootstrap/virtual-server`
      - [URL][github-k8s-manifest-virtual-server]
      - NginxIngress CRDの VirtualServer をデプロイする
- 作成されるApplication
  - Template機能 `{{path.basename}}` を利用して、ディレクトリ名をアプリケーション名とする
    - `nginx-ingress`
    - `virtual-server`

上記のApplicationがデプロイされることにより、boostrapに必要なK8sリソースが一括でプロイされることになる。  
今回の場合は、ApplicationSetをTerraform中でデプロイすることにより、EKSのデプロイと同時にArgoCDのデプロイと、インターネットアクセスが可能になっている。  
詳細はTerraformセクション参照。

yamlとしては以下のようになる。

```yaml: applicationset.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: bootstrap
  namespace: argocd
spec:
  generators:
  - git:
      repoURL: https://github.com/toyamagu-cicd/argocd-cicd-k8s-manifest.git
      revision: main
      directories:
      - path: "application/*/overlays/prd"
  template:
    metadata:
      name: '{{path.basename}}'
    spec:
      project: bootstrap
      source:
        repoURL: https://github.com/toyamagu-cicd/argocd-cicd-k8s-manifest.git
        targetRevision: main
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
```


#### K8s Secret管理

CICDを行う場合、K8s Secret管理は切り離せない関係にあるため、この節で簡単に言及しておく。  
実際、ArgoCDを運用するためにもプライベートGitリポジトリの資格情報や、OAuth用クライアントシークレットなどが必要である。

GitOpsは全ての情報がGitリポジトリに格納されていることを理想とするが、勿論資格情報などをGitリポジトリに格納してはいけない。  
[sealed-secrets][sealed-secrets]などを用いて暗号化した上でGitリポジトリに格納することも可能だが、鍵を用意して都度暗号化しないといけないため手間が増え、オペレーション事故などが怖い。  
そのため、各クラウドのシークレットストレージや、[HashiCorp Vault][vault]の外部ストレージを用いるのがとっつきやすいと思われる。  
本文書では AWS SystemsManager ParameterStoreに資格情報を格納することにする。  
AWS SecretsManagerを利用しないのは、料金がかかるのと、消去するのが若干面倒だからである。  
本番運用ではAWS SecretsManagerの方が機能面で優れているので、そちらを優先するとよいだろう。  

外部ストレージに保管した資格情報は、どのようにK8s上にデプロイすれば良いのだろうか。これもいくつか選択肢がある。

- [External Secrets Operator][external-secrets]
  - 外部ストレージからK8s Secretsを作成できる
- [Secrets Store CSI Driver][secrets-store-csi-driver]
  - 外部ストレージからK8s Secretを作成できる。こちらはPodにマウントする必要がある。
  - したがって、Secretが作成されるためにはPodを作成する必要がある。
- [Terraform kubernetes provider][terraform-kubernetes-provider]
  - Terraformのdataで外部ストレージからデータを拾ってきてK8s providerを用いてSecretをデプロイする。
  - K8sクラスタに追加のリソースをデプロイしなくて良いが、`.tfstate` にデータが残ってしまうのが難点。

個人的には、Podに資格情報をマウントする前提なら、SecretsStoreを用いると楽であると考える。  
[AWSの公式ドキュメント][secrets-store-csi-driver-aws]も充実している。  
本文書では追加のリソースが必要ないという簡便さを重視してTerraformでSecretsをデプロイする。*推奨はしない*。  
SecretsStoreに拡張するのは容易である。
さらなる詳細はTerraformセクションに譲る。

#### Helm Chart利用方法

ArgoCDも含むArgoファミリーには公式のHelm chartがないが、[Community maintainedなHelm chartが存在し][argo-helm]、現在も活発にメンテナンスされている。  
この `argo-cd` Helm chartを利用することになる。  

上記に加えて、 `argocd-apps` Helm chartがある[^argocd-apps]。  
このHelm chartでは、Application・Project・ApplicationSetがインストールできる。  

クラスター作成と同時にArgoCDのインストールや、ApplicationSet、Projectの作成まで終わっていると手作業でセットアップする手間が省ける。
そこで、本文書では[Terraform helm provider][terraform-helm-provider]を用いてTerraform中で上記Helm chartのインストールを行っている。

### 権限統制

本小節では権限統制方法を説明する。  

前述の通り、CICD基盤ではインフラストラクチャレベルでの権限統制が重要である[^governance]。  
厳しすぎる権限統制は開発効率を落とす一方で、緩すぎる権限統制は不正デプロイなどのインシデントに直結する。
両極限を加味した上で、開発者・運用者双方が権限統制の意義を理解し、合意した上で適切な権限統制を設定すること。

以下図に本文書での権限統制の概念図を示す。  
[ArgoCD Dex][argocd-user-management]を用いてGitHubとのOAuth連携を行うため、GitHubでの認証・認可はArgoCDの認証・認可と関連する。

![cicd-authorization](/images/cicd-eks-github-argocd/cicd-authorization.drawio.png)

以下GitHubリポジトリ・ArgoCDそれぞれに対して詳しい説明をするが、設定方針の概略は以下の通りである。

- 本番環境・開発環境ともに開発者・運用者に全てのリソースに対する読み取り権限を付与すること。
- 本番環境 `prd` NSは運用者のみがデプロイ権限を持つこと
- 開発環境 `dev` NSは開発者・運用者がデプロイ権限を持つこと

#### GitHubリポジトリ

GitHubリポジトリに対するロールを以下のように定める。
GitHubのブランチ保護設定 `Restrict who can push to matching branches` で、Merge可能なアクターを制限できる。

- ロールをMaintainのみに制限
- (Team以上) Organizationの特定グループ・ユーザーのみに制限
- (Team以上) 特定のGitHub Appに制限

これを利用して、本番環境ブランチへのMergeを統制する [^github-organization-free] 。

| リポジトリ                 | ブランチ | 役割                                                  | Merge可能なロール |
| :------------------------- | :------- | :---------------------------------------------------- | :---------------- |
| アプリケーションリポジトリ | `main`   | 本番環境にリリースされるコードが格納される。          | Maintain          |
|                            | `dev`    | 開発環境にリリースされるコードが格納される。          | Write             |
| K8sマニフェストリポジトリ  | `main`   | 本番環境にデプロイされるK8sマニフェストが格納される。 | Maintain          |
|                            | `dev`    | 開発環境にデプロイされるK8sマニフェストが格納される。 | Write             |

GitHubリポジトリでのアクター・ロールを以下のように定める。GitHub Appだけは特殊だが、本文書の設定だと大体Writeと同じ。

| アクター         | リポジトリ                 | ロール   | 説明・理由                                                                 |
| :--------------- | :------------------------- | :------- | :------------------------------------------------------------------- |
| Developer        | アプリケーションリポジトリ | Write    | 開発環境ブランチへの `push` や、本番環境ブランチへの PR を行うため。 |
| Developer leader | アプリケーションリポジトリ | Maintain | 本番環境ブランチのPRを承認するため。                                 |
| Operator         | アプリケーションリポジトリ | Maintain | 本番環境ブランチのPRを承認するため。                                 |
| Developer        | K8sマニフェストリポジトリ  | Write    | 開発環境ブランチへの `push` や、本番環境ブランチへの PR を行うため。 |
| Developer leader | K8sマニフェストリポジトリ  | Write    | 開発環境ブランチへの `push` や、本番環境ブランチへの PR を行うため。 |
| Operator         | K8sマニフェストリポジトリ  | Maintain | 本番環境ブランチのPRを承認し、CDツールを利用したデプロイを行うため。 |
| GitHub App       | K8sマニフェストリポジトリ  | N/A      | 開発環境ブランチへの `push` や、本番環境ブランチへの PR を行うため。 |

#### ArgoCDの認証・認可

以下表のようにRBAC設計を定める。

| アクター          | 対象          | アクション | `project/object` | 説明・理由                                                                         |
| :---------------- | :------------ | :--------- | :--------------- | :--------------------------------------------------------------------------------- |
| 全て              | `*`           | `get`      | `*/*`            | 全てのアクターに対し、全てのリソースに対する読み取り許可を与えるため[^argocd-read] |
| Operator          | `*`           | `admin`    | `*/*`            | 運用管理のため、全てのリソースに対する全ての許可を与えるため                       |
| Developer(leader) | `application` | `sync`     | `dev/*`          | 開発者に開発環境へのデプロイ権限を与えるため                                       |
|                   | `application` | `exec`     | `dev/*`          | 開発者に開発環境Podの `exec` 権限を与えるため[^argocd-exec]                        |

上記をコード化すると以下のようになる

```csv: policy.csv
g, ${github_org}:operator, role:operator
g, ${github_org}:developer, role:developer
g, ${github_org}:developer-leader, role:developer

g, role:operator, role:admin

p, role:developer, applications, sync, dev/*, allow
p, role:developer, exec, create, dev/*, allow
```

### ネットワーク経路 (おまけ)

本小節はネットワーク設定を説明する。CICD自体とは関係ないため、飛ばして良い。  
NginxIngress VitrualServer CRD自体は[非常に良い解説][thinkit-nginx-ingress]がある。

以下に概要図を示す。特段特殊なことはしていないので、説明は省略する。
設定意図などの思想は[GitHub上][github-toyamagu-cicd-documents-design-internet-access]にまとめている。興味があれば参照されたい。

![network](/images/cicd-eks-github-argocd/network.drawio.png)

### Terraform

本小節はTerraform利用方針を説明する。

#### Terraform概要

本格的にクラウドを利用し、大量のリソースをデプロイする場合、Infrastructure as Code (IaC) は事実上必須であると言ってよいだろう。  
IaCを活用することのメリットには枚挙に暇がないが、例えば以下がある。

- （クラウド）リソースをコード化し、宣言的に作成できる
- 環境をすぐに立ち上げ、不要になったら廃棄できる
- リソースのドリフトを検出し、GitOpsに基づいたあるべき姿からはずれた場合に検出できる
- CICDと組み合わせることで、ブランチ作成やPRに紐づけたリソース作成などの応用ができる

AWSでIaCを行う場合AWS CDK・CloudFormation・Terraform・Pulumi等の選択肢がある。  
本文書では汎用性が高く、安定的で、幅広いprovider（プロビジョニングできるリソース）をサポートしているTerraformを用いる。
Terraform自体のCICDは検証目的のため行わないが、本番環境で運用する場合、TerraformのCICDもできていると望ましいだろう[^atlantis]。  
特に、CICDを定期的にスケジューリングしておけば、(さすがに実際にApplyはしないが)ドリフトの検出ができてよい。

#### Terraform利用方針

本文書ではTerraformで管理できる部分は全て管理するというアプローチを取る。  
したがって、 `terraform apply` コマンドでArgoCDのリソースも含めて全てのリソースのセットアップが完了することを目指す。
Terraformで管理するリソースは例えば以下である。

- EKS及びVPC
- 資格情報
  - OAuth clientID, clientSecret
  - GitHub App秘密鍵
- K8sリソース
  - NS
  - ArgoCD
    - ApplicationSet
    - Project
    - Dex用Secret
- GitHub
  - Organization/Team
  - リポジトリ
    - ロール付与
  - ブランチ (protection)

何を管理しない（できない）かを以下に列挙する。管理しない理由としては、単純に[GitHub provider][terraform-provider-github]が対応していないためである。

- GitHub
  - User
  - Organization
  - OAuth Apps
  - GitHub Apps

#### Secrets管理

Terraformを用いたSecrets管理は[こちらのブログ][handling-secrets-with-terraform]の3.を参考にした。  
すなわち、ParameterStoreに保管する場所だけ作っておいて、Terraformの `local_exec` 機能で書き換えるというやり方である。
若干工夫した点として、資格情報が与えられていればParameterStoreの書き換えを行い、与えられていなければ値をdataを用いて持ってくるという形にした。

#### Terraformコード

全て [`toyamagu-cicd/terraform-argocd-cicd`][terraform-argocd-cicd] 以下に格納している。

- GitHub関連
  - `resources/github`
- ArgoCD関連
  - `resources/argocd`

上記のTerraformコードにより本文書で記述されたほとんどのリソースがデプロイされる。
手順の概略は `README.md` にある。

## まとめ

本文書ではGitHub Actions・ArgoCDをCICDツールとして用い、EKSにデプロイを行う方法を記載した。  
設定は検証用途のため粗い部分も多いが、本番環境での運用も見据えて、抑えるべきポイントは抑えたつもりである。  
また、インフラストラクチャに近い部分の管理は全てTerraformで宣言的に行うという思想のもと、極力Terraformコマンドを実行するだけで必要なリソースが立ち上がるようにした。

途中力付きた部分もあるので、説明が少ない・あるいは誤っている部分があれば、喜んで修正したい。指摘を歓迎します。

本文書がK8s運用の自動化に向けて、なにかの参考になれば幸いである。

## References

[^argocd-helm]: TerraformでKustomizeかHelmをインストールする場合、Helmのほうが楽である。ただし、以下理由からプロダクションでの採用は慎重に検討した方が良いと考える。1) ArgoCD Helm chartはcommunity maintainedであるため、公式のマニフェストと比べると若干信頼性に欠ける面がある。2) 頻繁にリリースが行わているため、追従するのは若干大変。3) 公式のバージョンサポートはN, N-1型だが、helmチャートの方は最新マイナーバージョンのみサポートされているようだ。左記の問題点があるため、コアな部分のみ公式のチャートをkustomizeし、ApplicationSet・Application・Projectの部分のみ部分的にargocd-appsチャートを利用するのでも十分便利に利用できると思う。個人的な検証用途だが、Kustomizeを用いたArgoCD インストール Terraform module例の[リンク](https://github.com/toyamagu-2021/terraform-argocd-kustomize)を張っておく。
[^gitlab-flow]: 必要に応じてリリースブランチや、ステージング環境用のブランチを追加する。
[^github-organization-free]: GitHubOrganizationFreeではユーザーグループ毎の細かい権限制御ができないため、Enterprise版に比べてだいぶ粗いものになる。
[^k8s-manifest-push]: PRでもよい。
[^cicd-checkout-k8s-manifest]: checkoutしてからPRするのは、devブランチの断面保管のため。
[^kustomize-branch]: もちろん、必要に応じて環境ごとにブランチ分けしてもよい。
[^kustomize-directory]: [公式](https://github.com/kubernetes-sigs/kustomize/blob/master/examples/helloWorld/README.md#compare-overlays)より引用した。
[^governance]: 個人的な感想だが、CICD基盤の権限統制はガードレールと呼ぶのが好きである。開発を車の走行に例えると、ガードレールの役割はむやみに車の速度を抑えるためにあるのではない。万が一のことが起こりそうになっても、車の損傷や人的な被害を抑えるためにある（と解釈している）。
[^argocd-read]: デフォルトの `read-only` ロールを用いる。Podのログに対する読み取り権限も与えてしまう。本番環境では許容できないかもしれないので、検討すること。ArgoCD v2.4以降ではRBACを用いてログの閲覧のみを制限することができる。
[^argocd-exec]: `kubectl exec` とほぼ同じ
[^argocd-project]: 権限統制の他にも、定期的なデプロイやOrphaned resourceの検出等、様々な機能がある。本文書では権限統制以外の目的には用いない。
[^argocd-applicationset-go-template]: [goTemplate機能](https://argo-cd.readthedocs.io/en/latest/operator-manual/applicationset/GoTemplate/)が v2.5から実装される。現在v2.5はstableでないので注意。
[^argocd-app-of-apps]: App of Apps パターンに関する諸注意
    - App of Appsパターンには[実質的に全てのProjectにデプロイ可能になってしまうなどの問題があった][argocd-issue-2785]。  
      これはv2.5からの[Application in any namespace機能][argocd-pr-application-in-any-namespace]で部分的に解消される見込みだが、以下の理由から採用は慎重に検討する必要がある。
      - [Beta機能である][argocd-application-in-any-namespace]
      - 現時点ではクラスタ内の任意のNSでのApplication作成を想定しており、クラスタ間がサポートされるかは未定である
[^argocd-applicationset-git-generator]: 今回はGit Generatorの中でも基本的な[Directories][argocd-applicationset-directories]を用いる。Gitリポジトリに設置した config ファイルを参照してデプロイ先クラスターを指定するなどもできる。[Files][argocd-applicationset-files]を参照。
[^argocd-applicationset-bootstrap]: Prometheusや、ServiceMesh、GateKeeperなどインフラに近いリソースはbootstrapに含めてしまいたいというお気持ちになる。
[^atlantis]: TerraformCloudをよく使っているが、[atlantis][atlantis]も面白そうで使ってみたいというお気持ちはある。PRをMergeした後にエラーが出るのはしんどい。
[^argocd-apps]: v5.0.0から `argo-cd` Helm chartから分離した。ArgoCD CRDをHelm chart内に含める際に、CRDに依存している部分があると良くないということで外だしされたようである。興味がある人は[PR][argo-helm-pr-manage-crd-by-helm]参照のこと。

[atlantis]: https://www.runatlantis.io/
[argocd-issue-2785]: https://github.com/argoproj/argo-cd/issues/2785
[eks]: https://aws.amazon.com/jp/eks/
[github-actions]: https://github.com/features/actions
[argocd]: https://argo-cd.readthedocs.io/en/stable/
[terraform-github-provider]: https://registry.terraform.io/providers/integrations/github/latest/docs
[terraform-eks-module]: https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest
[terraform-helm-provider]: https://registry.terraform.io/providers/integrations/github/latest/docs
[argocd-helm]: https://github.com/argoproj/argo-helm
[toyamagu-cicd]: https://github.com/toyamagu-cicd
[argocd-best-practices]: https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/
[git-flow]: https://nvie.com/posts/a-successful-git-branching-model/
[github-flow]: https://docs.github.com/en/get-started/quickstart/github-flow
[gitlab-flow]: https://docs.gitlab.com/ee/topics/gitlab_flow.html
[trunc-base-flow]: https://www.atlassian.com/continuous-delivery/continuous-integration/trunk-based-development
[argo-cd]: https://argo-cd.readthedocs.io/en/stable/
[cncf-argocd]: https://www.cncf.io/projects/argo/
[github-argocd]: https://github.com/argoproj/argo-cd
[jenkins]: https://www.jenkins.io/
[spinnaker]: https://spinnaker.io/
[argo-rollout]: https://argoproj.github.io/argo-rollouts/
[argo-workflows]: https://argoproj.github.io/argo-workflows/
[kustomize]: https://kustomize.io/
[argocd-application-in-any-namespace]: https://argo-cd--10678.org.readthedocs.build/en/10678/operator-manual/app-any-namespace/
[monorepos]: https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Use-Cases/#use-case-monorepos
[argocd-applicationset-directories]: https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-Git/#git-generator-directories
[argocd-applicationset-files]: https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-Git/#git-generator-files
[argocd-user-management]: https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management
[12factor-app-config]: https://12factor.net/config
[github-k8s-manifest-nginx-ingress]: https://github.com/toyamagu-cicd/argocd-cicd-k8s-manifest/tree/main/application/bootstrap/nginx-ingress
[github-k8s-manifest-virtual-server]: https://github.com/toyamagu-cicd/argocd-cicd-k8s-manifest/tree/main/application/bootstrap/virtual-server
[ingress-nginx]: https://github.com/kubernetes/ingress-nginx
[thinkit-nginx-ingress]: https://thinkit.co.jp/article/18771
[github-toyamagu-cicd-documents-design-internet-access]: https://github.com/toyamagu-cicd/document/blob/argocd-cicd/argocd-cicd/design.md#internet-access
[terraform]: https://www.terraform.io/
[zenn-github-app]: https://zenn.dev/tatsuo48/articles/72c8939bbc6329
[toyamagu-cicd-argocd-cicd-k8s-manifest]: https://github.com/toyamagu-cicd/argocd-cicd-k8s-manifest
[terraform-provider-github]: https://registry.terraform.io/providers/integrations/github/latest/docs
[sealed-secrets]: https://github.com/bitnami-labs/sealed-secrets
[vault]: https://www.vaultproject.io/
[external-secrets]: https://external-secrets.io/v0.6.0/
[secrets-store-csi-driver]: https://secrets-store-csi-driver.sigs.k8s.io/
[terraform-kubernetes-provider]: https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs
[secrets-store-csi-driver-aws]: https://docs.aws.amazon.com/secretsmanager/latest/userguide/integrating_csi_driver.html
[handling-secrets-with-terraform]: https://engineering.mobalab.net/2021/03/25/handling-secrets-with-terraform/
[toyamagu-cicd-argocd-cicd-application]: https://github.com/toyamagu-cicd/argocd-cicd-application/blob/main/.github/workflows/push.yaml
[configuring-openid-connect-in-amazon-web-services]: https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services
[zenn-github-actions-support-openid-connect]: https://zenn.dev/miyajan/articles/github-actions-support-openid-connect
[conftest]: https://www.conftest.dev/
[opa]: https://www.openpolicyagent.org/
[terraform-argocd-cicd]: https://github.com/toyamagu-cicd/terraform-argocd-cicd
[argo-helm]: https://github.com/argoproj/argo-helm
[argo-helm-pr-manage-crd-by-helm]: https://github.com/argoproj/argo-helm/pull/1342
[argocd-pr-application-in-any-namespace]: https://github.com/argoproj/argo-cd/pull/9755
