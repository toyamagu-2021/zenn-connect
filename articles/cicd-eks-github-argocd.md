---
title: "GitHubとArgoCDを用いたKubernetes(EKS)へのCICD"
emoji: "🌸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kubernetes", "github", "cicd", "argocd", "eks"]
published: false
---

## 概要

Kubernetes (K8s) や microservice を最大限に活用する上で CICD 基盤は欠かせない。  
本文書では、K8s基盤として[EKS][eks]を、CIとして[GitHub Actions][github-actions]を、CDとして[ArgoCD][argocd]を採用した場合のアーキテクチャの例を解説する。  
特に、プロダクションレベルへの拡張を想定し、以下観点を重視して設計する。

- 組織の拡大に従ってスケールするアーキテクチャとなっているか。すなわち、人が介入する作業を減らし、自動化できているか
- 可能な限りすべてのリソースは宣言的に作成されているか
  - Single Source of Truth を設定し、設定・構成を一箇所に集約するため
  - リソースのドリフトを検出するため
- CICD基盤の認証認可は、組織のガバナンスを反映しているか

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
    - 検証用途のため、ArgoCDのインストールにはパラメータ設定の簡便さを重視して[ArgoCD Helm chart][argocd-helm]を採用する[^argocd-helm]

上記を実現するコード群はOrganizationも含めてGitHubに公開している[toyamagu-cicd]。質問・指摘を歓迎します。

## イントロダクション

### 背景

CICDは以下観点から現在のアーキテクチャにおいて重要なものとなっている

- ビルド・テスト・パブリッシュ・デプロイを自動化しリリーススピードをはやめる
- テストを自動的に都度行うことで、品質の改善を行う
- 徹底的な自動化を行うことによって、ビジネスの拡大に従って、サービスをスケールする
  - 特に大量のコンテナをデプロイするK8s等のコンテナオーケストレータを採用する場合、自動デプロイの仕組みがないと運用負荷がかかる

ただし、CICDは以下に注意が必要である

- ブランチ・リポジトリ戦略を定めないと、運用・セキュリティ管理が困難になる。
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
  - 単純なマシンユーザーと比較して、ユーザーの枠を専有しない・一時トークンの発行が楽など多くのメリットが有る[^zenn-github-app]。
  - 許可しているアクションは以下の通り。
    | リソース     | アクション     | 説明                                        |
    | :----------- | :------------- | :------------------------------------------ |
    | Contents     | Read and Write | K8sマニフェストアップデートのため           |
    | Pull Request | Read and Write | K8sマニフェストリポジトリでPRを発行するため |
- Kustomize
  - K8sマニフェストの環境差分管理に用いる
- ArgoCD
  - K8s用のCDツール
  - JenkinsやSppinakerと比較して、K8sに特化している分簡単に利用できる
  - 一方で、FluxCDよりは機能が豊富（ユーザー管理など）
  - 主要リソースの一覧と簡単な説明は以下表の通り
    | リソース     | アクション     | 説明                                        |
    | :----------- | :------------- | :------------------------------------------ |
    |  |  |  |

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
1. ArgoCDは以下のようなCDを行う。
    - `argocd-cicd-k8s-manifest` リポジトリを監視し、K8sマニフェストの変更に応じてEKSにデプロイを行う

以下に注意点を記述する。

1. アプリケーション用のリポジトリと、K8sマニフェスト用のリポジトリを分ける理由
    - 開発者・運用者の認知負荷を下げるため
    - 複数のアプリケーションのマニフェストをまとめるため
    - 不要なCICDのトリガーを避けるため
      - GitHubイベントに応じてCICDをトリガーする際、同じリポジトリだと（作り込みを行わない限り）関係ないリソースの変更に応じてCICD全体がトリガーされてしまう
      - 最悪の場合無限ループに陥る
    - ガバナンスのため
      - Gitリポジトリの権限分割は、リポジトリ単位・ブランチ単位がある。GitHub・GitLab等の主要なVCSにおいて、認可用のロールはリポジトリ単位で振られる
        - リポジトリ単位で分けると運用者・開発者ごとにロールを振ることができ、認可設計し易い
      - CDツールのアクセス対象を分割するため
    - その他、[ArgoCDのベストプラクティス][argocd-best-practices]に詳述されているため参照

### GitHubブランチ戦略

本小節ではGitHubブランチ戦略、またCIに関連するGitHubイベントを説明する。  

ブランチ戦略を複雑にすればガバナンスやリリース管理は厳密になるが、一方で運用負荷が上昇する。変更の取り込み漏れなどの、ミスが発生するもとにもなる。  
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
      - 12 factor app(若干古いが)にもあるベストプラクティスである[^12factor-app-config]。
    - *アプリケーションリポジトリでブランチ分割しているからとしてそちらで環境差分を吸収しないこと*。
      - 環境毎にアプリケーションコードに差分が存在するのは、悪夢である。
1. Kustomizeを利用しているが、環境毎にHelm Chartをデプロイしたい
    - KustomizeはHelmをサポートしている。`kustomize build` 時にオプションを有効にする必要があるので、ArgoCDの設定で有効化すること。

### ArgoCD

![argocd-applicationset](/images/cicd-eks-github-argocd/argocd-applicationset.drawio.png)

### 権限統制

本小節では権限統制方法を説明する。  
前述の通り、CICD基盤ではインフラストラクチャレベルでの権限統制が重要である[^governance]。  
厳しすぎる権限統制は開発効率を落とす一方で、緩すぎる権限統制は不正デプロイなどのインシデントに直結する。
両極限を加味した上で、開発者・運用者双方が権限統制の意義を理解し、合意した上で適切な権限統制を設定すること。

以下図に本文書での権限統制の概念図を示す。ArgoCDはDexを用いてGitHubとのOAuth連携を行うため[^argocd-user-management]、GitHubでの認証・認可はArgoCDの認証・認可に深い関わりを持つ。

![cicd-authorization](/images/cicd-eks-github-argocd/cicd-authorization.drawio.png)

#### GitHubリポジトリ

GitHubリポジトリに対するロールを以下のように定める。
GitHubのブランチ保護設定 `Restrict who can push to matching branches` で、Merge可能なアクターを制限できる。

- ロールをMaintainのみに制限
- (Team以上) Organizationの特定グループ・ユーザーのみに制限
- (Team以上) 特定のGitHub Appに制限

これを利用して、本番環境ブランチへのMergeを統制する[^github-organization-free]。  

| リポジトリ                 | ブランチ | 役割                                                  | Merge可能なロール |
| :------------------------- | :------- | :---------------------------------------------------- | :---------------- |
| アプリケーションリポジトリ | `main`   | 本番環境にリリースされるコードが格納される。          | Maintain          |
|                            | `dev`    | 開発環境にリリースされるコードが格納される。          | Write             |
| K8sマニフェストリポジトリ  | `main`   | 本番環境にデプロイされるK8sマニフェストが格納される。 | Maintain          |
|                            | `dev`    | 開発環境にデプロイされるK8sマニフェストが格納される。 | Write             |

GitHubリポジトリでのアクター・ロールを以下のように定める。GitHub Appだけは特殊だが、本文書の設定だと大体Writeと同じ。

| アクター         | リポジトリ                 | ロール   | 説明                                                                 |
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

| アクター          | 対象          | アクション | `project/obeject` | 説明                                                                             |
| :---------------- | :------------ | :--------- | :---------------- | :------------------------------------------------------------------------------- |
| 全て              | `*`           | `get`      | `*/*`             | 全てのアクターに対して、全てのリソースに対する読み取り許可を与える[^argocd-read] |
| Operator          | `*`           | `admin`    | `*/*`             | 運用管理のため、全てのリソースに対する全ての許可を与える                         |
| Developer(leader) | `application` | `sync`     | `dev/*`           | 開発者に開発環境へのデプロイ権限を与える                                         |
|                   | `application` | `exec`     | `dev/*`           | 開発者に開発環境Podの `exec` 権限を与える[^argocd-exec]                          |

上記をコード化すると以下のようになる

```csv: policy.csv
g, ${github_org}:operator, role:operator
g, ${github_org}:developer, role:developer
g, ${github_org}:developer-leader, role:developer

g, role:operator, role:admin

p, role:developer, applications, sync, dev/*, allow
p, role:developer, exec, create, dev/*, allow
```

### Appendix: ネットワーク経路

本小節はネットワーク設定を説明する。  
CICD自体とは関係ないため、飛ばして良い。

![network](/images/cicd-eks-github-argocd/network.drawio.png)

[^argocd-helm]: TerraformでKustomizeかHelmをインストールする場合、Helmのほうが楽である。ただし、以下理由からプロダクションでの採用は慎重に検討した方が良いと考える。1) ArgoCD Helm chartはcommunity maintainedであるため、公式のマニフェストと比べると若干信頼性に欠ける面がある。2) 頻繁にリリースが行わているため、追従するのは若干大変。3) 公式のバージョンサポートはN, N-1型だが、helmチャートの方は最新マイナーバージョンのみサポートされているようだ。左記の問題点があるため、コアな部分のみ公式のチャートをkustomizeし、ApplicationSet・Application・Projectの部分のみ部分的にargocd-appsチャートを利用するのでも十分便利に利用できると思う。個人的な検証用途だが、Kustomizeを用いたArgoCD インストール Terraform module例の[リンク](https://github.com/toyamagu-2021/terraform-argocd-kustomize)を張っておく。
[^github-organization-free]: GitHub Organization Freeではユーザーグループ毎の細かい権限制御ができないため、Enterprise版に比べてだいぶ粗いものになる。  
[^gitlab-flow]: 必要に応じてリリースブランチや、ステージング環境用のブランチを追加する。
[^k8s-manifest-push]: PRでもよい。
[^cicd-checkout-k8s-manifest]: checkoutしてからPRするのは、devブランチの断面保管のため。
[^kustomize-branch]: もちろん、必要に応じて環境ごとにブランチ分けしてもよい。
[^kustomize-directory]: [公式](https://github.com/kubernetes-sigs/kustomize/blob/master/examples/helloWorld/README.md#compare-overlays)より引用した。
[^12factor-app-config]: <https://12factor.net/config>
[^governance]: 個人的な感想だが、CICD基盤の権限統制はガードレールと呼ぶのが好きである。開発を車の走行に例えると、ガードレールの役割は車の速度を抑えるためにあるのではない。車の損傷や人的な被害を抑えるためにある。
[^zenn-github-app]: <https://zenn.dev/tatsuo48/articles/72c8939bbc6329>
[^argocd-read]: デフォルトの `read-only` ロールを用いる。ログに対する読み取り権限も与えてしまう。本番環境では許容できないかもしれないので、検討する。
[^argocd-exec]: `kubectl exec` とほぼ同じ
[^argocd-user-management]: <https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/>

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
