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
特に、プロダクションレベルの採用も加味して以下観点を重視して設計する。

- 組織の拡大に従ってスケールするアーキテクチャとなっているか。すなわち、人が介入する作業を減らし、自動化できているか
- CICD基盤の認証認可は、組織のガバナンスを保っているか

本文書で記述する内容は以下の通り

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


[^argocd-helm]: ArgoCD Helm chartはcommunity maintainedであるため、公式のinstallマニフェストと比べると若干信頼性に欠ける面もある。また、公式のバージョンサポートはN, N-1型だが、helmチャートの方は最新マイナーバージョンのみサポートされているようだ。このためプロダクションでの採用は慎重に検討すること。例えば、コアな部分のみ公式のチャートをkustomizeし、ApplicationSet・Application・Projectの部分のみ部分的にargocd-appsチャートを利用するのでも十分便利に利用できると思う。個人的な検証用途だが、Kustomizeを用いたmodule例の[リンク](https://github.com/toyamagu-2021/terraform-argocd-kustomize)を張っておく。

[eks]: https://aws.amazon.com/jp/eks/
[github-actions]: https://github.com/features/actions
[argocd]: https://argo-cd.readthedocs.io/en/stable/
[terraform-github-provider]: https://registry.terraform.io/providers/integrations/github/latest/docs
[terraform-eks-module]: https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest
[terraform-helm-provider]: https://registry.terraform.io/providers/integrations/github/latest/docs
[argocd-helm]: https://github.com/argoproj/argo-helm
[toyamagu-cicd]: https://github.com/toyamagu-cicd
