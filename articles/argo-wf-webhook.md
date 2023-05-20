---
title: "ArgoWorkflowsをGitHub Webhookでトリガーする"
emoji: "🌸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["argo", "github"]
published: true
---


## 概要

本記事ではArgo Workflowsの Workflow を GitHub Webhookでトリガーする方法を紹介する。  
Webhookでのトリガー自体は先行する記事があるが、 `curl` を用いた実行までの紹介であった [参考1][zenn-argo-wf-webhook] [参考2][qiita-argo-wf-webhook]。  
GitHub Webhookなどの外部Webhookでトリガーする情報は少なかったため、本記事の執筆にいたった。

本記事ではArgo WorkflowsをローカルK8sクラスタにデプロイし、GitHub Webhookを用いてWorkflowをトリガーする方法を紹介する。
GitHub Webhookを受け取るためにはローカルK8sクラスタをインターネット公開する必要があるが、そのために [ngrok][ngrok] を用いた。
ngorkはローカルホストに立てた環境を簡単にインターネットに公開することができるソフトウェアである。

検証に用いたコードはすべて [GitHubリポジトリ][toyamagu-2021-argo-workflows-sandbox]上に公開しており、簡単に各々の環境で試すことができる。

以下に概要図を示す。

![architecture-diagram](/images/argo-wf-webhook/architecture-diagram.drawio.png)

## イントロダクション

### Argo Workflows

[Argo Workflows][argo-wf] (ArgoWF) は Kubernetes (K8s) ベースのワークフロー管理OSSである。
シンプルなジョブを簡単に実行できることはもちろん、複雑な依存関係を持ったジョブをDAGなどの手法を用いて比較的に取り扱うことができる。

Argo FamilyにはArgoCDがあるが、ArgoCDはCDを目的とするツールである。一方、ArgoWFはCIに主眼を置いている。
ワークフロー管理OSSは様々あるが、ArgoWFは優れたGUIやSSOによる権限管理機能を持っており、大きな組織でも運用がスケールするのが優れている点であると感じる。
正直特段ArgoCDと相性が良いと感じることは少ないが、ArgoCDはGitOpsを元にしたCDツールであるため、One-ShotあるいはadhocなJobの実行には向かない。
その点、ArgoCDを補完するツールとして、同じArgo Familyであり、表面上のUIが似通っているArgoWFを採用するのは割りとありな選択肢だろう。

ただ、筆者は大規模なデータ分析基盤を扱った経験もなければ本番環境でジョブを扱った経験も少ないため、その辺の比較は他の記事を参照してほしい。
日本でいうと[ZoZo][zozo-argo-wf]さん、[Line][line-argo-wf]さんあたりの事例記事が参考になると思われる。

### Argo WorkflowとAPI

ArgoWFはREST APIを公開しており、各エンドポイントにリクエストを送ることで色々と操作できる。
ArgoWFをインストールすると、API docsが確認できる。

APIエンドポイントのトリガーには `curl` 等を用いることもできるが、実用的には様々な外部のWebhookを用いてトリガーすることになるだろう。
ArgoWF単体でもGitHub等とは連携できるが、より豊富な機能を求めるなら、ArgoEventsを用いて行うのが良いのだと思われる。

ArgoWF単体でやる場合、[WorkflowEventBindings][argo-wf-doc-events] を用いる。
ArgoEventsと連携する場合、 [EventSources][argo-events-event-sources] と [Sensor][argo-events-sensor] を使うことになる。

### GitHub Webhook

GitHub上のイベントを外部のエンドポイントに通知するWebhook。
ArgoWFでの連携だと、PRをソースとしたCIトリガー（イメージビルド、E2Eテスト、マイグレーションテスト等）が考えられるだろうか。
公式ドキュメントでは、以下の部分の記事を参考にした [参考1][argo-wf-doc-events], [参考2][argo-wf-doc-webhooks]。 

## ローカルクラスタへのArgoWFのインストール

:::message alert
以下すぐ作って試して壊す前提で、Secretなどを適当に公開しているが、作業が終わったら必ず削除すること。
:::

K3dでもkindでも好きなものを使えば良いと思うが、今回はkindを用いた。

1. [GitHubのリポジトリ][toyamagu-2021-argo-workflows-sandbox]をclone
1. `cd github-webhook/manifests`
1. `kind create cluster`
1. `helmfile sync .`
1. ArgoWFログイン用トークン取得。
    - `ARGO_TOKEN="Bearer $(kubectl get secret -n argo admin.service-account-token -o=jsonpath='{.data.token}' | base64 --decode)"`
    - `echo $ARGO_TOKEN`
1. `k port-forward -n argo deployments/argo-workflows-server 2746:2746`
1. `http://localhost:2746` でArgoWFにアクセス。真ん中の `client authentication` の部分に上記トークンを貼り付けてログイン。

以上で終了だが、いくつか注意点がある。

1. `argo-workflows/values.yaml` で ClusterRole と ClusterRoleBinding を用意している。
    - これは、 `github.com` というK8s ServiceAccount (SA) をArgoWFが見つけられるようにしている。
    - これを設定しないと、 Webhookが届いたときにWorkflowをトリガーするために利用する `github.com` というSAをArgoWFが見つけられず、以下のようなエラーが出る。
      `Failed to process webhook request" error="failed to get service account \"github.com\": serviceaccounts \"github.com\" is forbidden: User \"system:serviceaccount:argo:argo-workflows-server\" cannot get resource \"serviceaccounts\" in API group \"\" in the namespace \"argo\`
    - 余談だが、Helm chartを利用してSSOを設定していれば `server.sso.rbac.enabled` 、この権限は自動的に付与されるため設定する必要はない。
1. `github` 以下のリソースは[公式リポジトリ][argo-wf-webhooks]からいただいたものである。BitBucket等使いたい人は公式doc及びリポジトリを参照のこと。

## ローカルK8sクラスタのインターネット公開

[ngrok][ngrok] を用いてローカルK8sクラスタをインターネットに公開する。

1. ngrokにアクセスし、アカウントを作成・トークンを取得する
1. `NGROK_AUTHTOKEN=<INSERT-YOUR-TOKEN>`
1. `docker run -it -e NGROK_AUTHTOKEN=${NGROK_AUTHTOKEN} ngrok/ngrok:latest http http://host.docker.internal:2746`
    - WSL2ではこれで動いたのだが、Macでは無理かもしれない。 [公式ドキュメント参照][ngrok-docker]。
1. `https://<YOUR_DOMAIN>.ngrok-free.app` が表示されるのでアクセスする。

以上でローカルK8sクラスタがインターネットに公開された。

## `curl` を用いてWorkflowの送信テストを行う

GitHubとの連携を始める前に、まずは `curl` でエンドポイントを叩いてみる。

1. `ARGO_SERVER=localhost:2746`
1. `ARGO_GITHUB_TOKEN="Bearer $(kubectl get secret -n argo github.com.service-account-token -o=jsonpath='{.data.token}' | base64 --decode)"`
1. `curl http://$ARGO_SERVER/api/v1/events/argo/ -H "Authorization: $ARGO_GITHUB_TOKEN" -H "X-Argo-E2E: true" -d '{"zen": "hello events"}'``
1. Emptyレスポンスが帰ってきて、Workflowがトリガーされているはず。
    - WorkflowEventBindings
      ![WorkflowEventBindings](/images/argo-wf-webhook/workflow-event-bindings.drawio.png)
    - Workflow
      ![SubmittedWorkflow](/images/argo-wf-webhook/workflow-from-http-req.drawio.png)
1. 同様にインターネット公開したAPIエンドポイントも叩けるはずなので、試みると良いだろう。

注意点は以下の通り。

1. `ARGO_GITHUB_TOKEN` は先程とは別のSAのトークンを取得している。 `github.com` というSAが `github.com` のWebhookと紐づくため、これを用いた。
1. WorkflowEventBinding `workflow-templates.workflow-event-binding.yaml` では `event.selector: "true"` としてすべてのリクエストに対して発火するようにしているが、ヘッダーなどで条件を制御することができる。

## GitHub WebhookでWorkflowをトリガーする

後はGitHub上にWebhookを作ってあげて、上記のエンドポイントを叩いて上げれば良い。
以下ではGitHub OrganizationでWebhookを作成したが、Repository上で作成してもよい。

1. Webhook作成
    - Payload URLには ngrok を用いて作成したエンドポイントを作成する。
    - Secretには、 `github/argo-workflows-webhook-clients-secret.yaml` に記載した値を代入する
      ![GitHubWebhook](/images/argo-wf-webhook/github-org-webhook.drawio.png)
1. テストを送ってみる
    ![GitHubWebhookTest](/images/argo-wf-webhook/github-webhook-payload.drawio.png)
1. Workflowをチェック
    ![GitHubWebhookWorkflow](/images/argo-wf-webhook/github-webhook-workflow.drawio.png)

以上でGitHub Webhookを用いてArgoWFのWorkflowをトリガーできた。

## まとめ

:::message alert
作成したリソースを必ず掃除すること

1. GitHub Webhook削除
1. ngrok docker containerの停止

:::

本記事ではGitHub WebhookからArgoWF Workflowをトリガーする方法を紹介した。
色々使いではあると思うので、参考になれば幸いである。

<!-- References -->

[argo-wf]: https://argoproj.github.io/argo-workflows/
[ngrok]: https://ngrok.com/
[zenn-argo-wf-webhook]: https://zenn.dev/hiroms/articles/f108e30b945312
[qiita-argo-wf-webhook]: https://qiita.com/MahoTakara/items/2a1473fe211b5fbc393d
[toyamagu-2021-argo-workflows-sandbox]: https://github.com/toyamagu-2021/argo-workflows-sandbox
[zozo-argo-wf]: https://techblog.zozo.com/entry/faans-argo-workflows
[line-argo-wf]: https://engineering.linecorp.com/ja/blog/automating-baremetal-setup
[argo-wf-webhooks]: https://github.com/argoproj/argo-workflows/tree/master/manifests/quick-start/base/webhooks
[argo-wf-doc-events]: https://argoproj.github.io/argo-workflows/events/
[argo-wf-doc-webhooks]: https://argoproj.github.io/argo-workflows/webhooks/
[ngrok-docker]: https://ngrok.com/docs/using-ngrok-with/docker/
[argo-events-event-sources]: https://argoproj.github.io/argo-events/eventsources/services/
[argo-events-sensor]: https://argoproj.github.io/argo-events/sensors/triggers/argo-workflow/
