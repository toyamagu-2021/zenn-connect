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

## アーキテクチャの説明

### CICD全体像

本節では全体像を解説する。本節は全体像を説明することに注力し、実際にどのように実現するかは、後の節で詳しく説明する。

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
      - Gitリポジトリの権限分割は、リポジトリ単位・ブランチ単位がある。GitHub・GitLab等の主要なVCSにおいて、認可用のロールはリポジトリ単位で振られる。
        - リポジトリ単位で分けると運用者・開発者ごとにロールを振ることができ、認可が非常に楽になる
      - CDツールのアクセス対象を分割するため
    - その他、[ArgoCDのベストプラクティス][argocd-best-practices]に詳述されているため参照

### GitHubブランチ・イベント戦略

![cicd-authorization](/images/cicd-eks-github-argocd/cicd-authorization.drawio.png)

### ArgoCD

![argocd-applicationset](/images/cicd-eks-github-argocd/argocd-applicationset.drawio.png)

### Appendix: ネットワーク経路

![network](/images/cicd-eks-github-argocd/network.drawio.png)

[^argocd-helm]: TerraformでKustomizeかHelmをインストールする場合、Helmのほうが楽である。ただし、以下理由からプロダクションでの採用は慎重に検討した方が良いと考える。1) ArgoCD Helm chartはcommunity maintainedであるため、公式のマニフェストと比べると若干信頼性に欠ける面がある。2) 頻繁にリリースが行わているため、追従するのは若干大変。3) 公式のバージョンサポートはN, N-1型だが、helmチャートの方は最新マイナーバージョンのみサポートされているようだ。左記の問題点があるため、コアな部分のみ公式のチャートをkustomizeし、ApplicationSet・Application・Projectの部分のみ部分的にargocd-appsチャートを利用するのでも十分便利に利用できると思う。個人的な検証用途だが、Kustomizeを用いたArgoCD インストール Terraform module例の[リンク](https://github.com/toyamagu-2021/terraform-argocd-kustomize)を張っておく。

[eks]: https://aws.amazon.com/jp/eks/
[github-actions]: https://github.com/features/actions
[argocd]: https://argo-cd.readthedocs.io/en/stable/
[terraform-github-provider]: https://registry.terraform.io/providers/integrations/github/latest/docs
[terraform-eks-module]: https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest
[terraform-helm-provider]: https://registry.terraform.io/providers/integrations/github/latest/docs
[argocd-helm]: https://github.com/argoproj/argo-helm
[toyamagu-cicd]: https://github.com/toyamagu-cicd
[argocd-best-practices]: https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/
