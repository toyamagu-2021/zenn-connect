---
title: "ArgoCD v2.6 でApplicationマルチソースがRelease candidateになりました"
emoji: "🌸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["argocd"]
published: true
---


## 概要

ArgoCD v2.6 で Application の `spec.sources` セクションがRCとなったため、どんな点が嬉しいかと、簡単な機能紹介をしたい。

## イントロダクション

ArgoCD v2.6 では Application の `spec.sources` セクションがRCとなり、複数のリポジトリを `source` として利用できるようになった。

* [PR][github-pr-multi-sources]
* [ドキュメント][user-guide-multi-sources]
* [Argo CD v2.6 Release Candidate][medium-argocd-v2-6-rc]

:::message alert
20221220時点で v2.6 は RC かつ Muitl-source は Beta feature のため、実環境への導入は絶対にしないこと。  
また、GUI上での動作はほとんど保証されていないため、怪しい動作をすることがある。
:::

これは何が嬉しいのだろうか？
複数のソースを持った Application をデプロイできることは勿論よさそうだが、コミュニティーにとって特に嬉しいのは、Helm chartと `values.yaml` を分離できることであろう。
通常運用する上でも、Helm chartと `values.yaml` が分離されているのはよくあることだと思う。
これは、今までだとdependencyに含めたり、カスタムプラグインを記述するなど、面倒な処方箋が必要だった。

* [Mixiさんのブログ][medium-mixi-argocd-helm]
* [StackOverflow][stackoverflow-argocd-helm]

今回のアップデートにより、マルチソースがRCとなったため、GA後には上記運用が不要になる（と思われる。）。  
簡単に実験してみることにする。

## インストール

今回は [Kind][kind] を用いることにする。

1. `kind create cluster`
1. ArgoCD インストール

    ```bash
    kubectl create ns argocd
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.6.0-rc1/manifests/install.yaml 
    ```

1. ArgoCD コンソールにアクセス

    * パスワード取得と、ポートフォワード

      ```bash
      kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
      kubectl port-forward svc/argocd-server 8080:80
      ```

    * ユーザー名は `admin`

1. サンプルApplicationデプロイ

    * 今回は以下の用に `helm-guestbook` を `https://github.com/toyamagu-2021/argocd-multirepo-test` にあるマニフェストを利用してデプロイできるようにする。
        1. `spec.sources.ref` で `values.yaml` が保管されているリポジトリの reference (今回は `valuerepo` ) を定義する。
        1. 他の `spec.sources.helm.valuefiles` で `${valuerepo}` などとして参照する。

    * 具体的には以下のようになる。

      ```yaml:application.yaml
      apiVersion: argoproj.io/v1alpha1
      kind: Application
      metadata:
        name: test
        namespace: argocd
      spec:
        destination:
          namespace: default
          server: https://kubernetes.default.svc
        project: default
        sources:
          - ref: valuerepo
            repoURL: https://github.com/toyamagu-2021/argocd-multirepo-test
          - helm:
              valueFiles:
                - $valuerepo/env/dev/values.yaml
              releaseName: dev
            path: helm-guestbook
            repoURL: https://github.com/argoproj/argocd-example-apps
            targetRevision: HEAD
          - helm:
              valueFiles:
                - $valuerepo/env/prd/values.yaml
              releaseName: prd
            path: helm-guestbook
            repoURL: https://github.com/argoproj/argocd-example-apps
            targetRevision: HEAD
      ```

    * `kubectl apply -f application.yaml`

1. Sync

    * GUI上でもよいし、CUI上でもよい。
    * Syncに成功し、以下のように複数のアプリケーションがデプロイされる。
    * ![multi-repo-console](/images/argocd-multi-repo/multi-repo-console.drawio.png)

## まとめ

Applicationのマルチソースがサポートされた事により、 helm でのデプロイが非常に楽になった。  
現在Beta featureであり、特にUI上では不安定な部分がある (現時点での [Issue][github-issue-multi-source-apps] のまとめ)。
試して見てバグがあったりしたら、GitHubで報告するとGAが早くなるかもしれない。  
年末年始のお供にいかがでしょうか。

<!-- References -->

[medium-mixi-argocd-helm]: https://mixi-developers.mixi.co.jp/argocd-with-helm-7ec01a325acb
[stackoverflow-argocd-helm]: https://stackoverflow.com/questions/73023423/templates-and-values-in-different-repos-via-argocd
[kind]: https://kind.sigs.k8s.io/
[github-issue-multi-source-apps]: https://github.com/argoproj/argo-cd/issues?q=is%3Aopen+label%3Amulti-source-apps+sort%3Aupdated-desc
[github-pr-multi-sources]: https://github.com/argoproj/argo-cd/pull/10432
[user-guide-multi-sources]: https://argo-cd.readthedocs.io/en/latest/user-guide/multiple_sources/
[medium-argocd-v2-6-rc]: https://blog.argoproj.io/draft-argo-cd-v2-6-release-candidate-ced1853bbfdb
