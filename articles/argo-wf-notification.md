---
title: "Argo WorkflowsのLifecycleHookを用いたSlack notification"
emoji: "🌸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["argo", "github"]
published: true
---

## 概要

Argo WorkflowsのSlack notificationを設定する方法を解説する。
運用負荷を下げるために、通知方法は [LifecycleHook][argo-wf-lifecycle-hook] を用いて、 Running, Succeeded, Failureを統一して扱えるようにした。

検証に用いたコードはすべて [GitHubリポジトリ][toyamagu-2021-argo-workflows-sandbox]上に公開しており、簡単に各々の環境で試すことができる。

## イントロダクション

通知関連はMVPとして上がることは少ないが、運用上地味に有用である。
本記事ではArgo Workflows (ArgoWF) のSlack通知を行う方法を解説する。

通知を行う方法は、大きく分けて [ExitHandler][argo-wf-exit-handler] を使う方法と [LifecycleHook][argo-wf-lifecycle-hook] を用いる方法がある。
前者の方法はWorkflowの終了時のみトリガーされるため、後者の方法を利用し、Running, Succeeded, Failure時に通知を行う。

似た記事は他にもあるが、 ExitHandlerを用いたものだったため、本記事を書いた [参考1][zenn-exit-handler], [参考1][komi-exit-handler], [参考3][aaaanwz-exit-handler] 。

## Slack ApplicationでのWebhook URLの作成

これについてはいくらでも記事があると思うので、省略する。

1. SlackでAppを作成する
1. AppでWebhookURLを作成する。
    - Webhook URLは資格情報と同様のセキュリティレベルで扱うこと。

## ArgoWFのデプロイ

[リポジトリ][toyamagu-2021-argo-workflows-sandbox] を適当にcloneした後、以下実行。

1. `cd workflow-hook/manifests` 
1. `cp sample.env .env` として、WebhookURLを置き換える。
1. `kind create cluster`
1. `set -a; source .env`
1. `helmfile sync -e withought-workflow-defaults .`
1. `ARGO_TOKEN="Bearer $(kubectl get secret -n argo admin.service-account-token -o=jsonpath='{.data.token}' | base64 --decode)"`
1. `k port-forward -n argo deployments/argo-workflows-server 2746:2746`
1. 先のトークンをコピペしてGUIログイン

## Slack通知のテスト

まず、愚直に以下のようにWorkflowTemplateにSlack通知のWorkflowを書いて実行してみる。

- [hook-slack-bot-raw](https://github.com/toyamagu-2021/argo-workflows-sandbox/blob/main/workflow-hook/manifests/workflow/hooks.yaml)

1. GUI上のWorkflowTemplate -> `hook-slack-bot-raw` -> submit
1. 以下のようなWorkflowの実行結果と、Slack通知が確認できる。Hookとして Runnning と Succeeded を指定しているため、2件の通知を確認できる。
   ![slack-bot-hook](/images/argo-wf-notification/slack-bot-hook.drawio.png)
1. submit時にexit codeを変えれば、エラー通知を見ることもできる

コードを説明する。

1. 以下の部分でLifecycleHookを指定しており、Hookに引っかかったときに `notify-slack` Workflowをトリガーするようになっている。

https://github.com/toyamagu-2021/argo-workflows-sandbox/blob/a9617a95157c804add7e39623088820a66b901d5/workflow-hook/manifests/workflow/hooks.yaml#L13-L41

2. `notify-slack` Workflowは以下の部分である。

    - Slack通知用のコンテナとしては、 [nagomiso/slackutils][nagomiso-slackutils] を拝借した。
    - `inputs` として `message` と `color` を受け取り、Slackに通知している。
    - コメントにもあるが、 `workflow.name` 等を利用すると、 実行したWorkflow のデータを参照することができる。

https://github.com/toyamagu-2021/argo-workflows-sandbox/blob/a9617a95157c804add7e39623088820a66b901d5/workflow-hook/manifests/workflow/hooks.yaml#L55-L100

## WorkflowDefaultsを用いた共通化

Slack通知自体はできたが、上記のWorkflowをすべてのWorkflowTemplateに書くのはあまりにも大変である。
そのため、 [Default Workflow spec][argo-wf-workflow-defaults] 機能を用い、すべてのWorkflowに上記をデフォルトで設定することにする。

1. `helmfile diff -e default .` とすると、先程とのdifferenceが表示される。
2. `helmfile sync -e default .` で apply する。
3. GUI上のWorkflowTemplate -> `sh-exit` -> として実行する。今度は exit code 1としてみる。
4. 以下のようなSlack通知と実行結果を確認できる
    ![slack-bot-hook](/images/argo-wf-notification/slack-bot-hook.drawio.png)

先程に追加したのは以下の部分である。

https://github.com/toyamagu-2021/argo-workflows-sandbox/blob/a9617a95157c804add7e39623088820a66b901d5/workflow-hook/manifests/argo-workflows/values.yaml.gotmpl#L2-L90

上記を追加することによって、実行するすべてのWorkflowでデフォルトでSlack通知が設定される。

## まとめ

Argo Workflows でLifecycleHook機能を用いてSlack通知を行う方法を解説した。
また、Default Workflow spec機能を用いるとすべてのWorkflowに共通する設定を抜き出すことができて便利である。

[argo-wf-lifecycle-hook]: https://argoproj.github.io/argo-workflows/lifecyclehook/
[argo-wf-exit-handler]: https://argoproj.github.io/argo-workflows/walk-through/exit-handlers/
[toyamagu-2021-argo-workflows-sandbox]: https://github.com/toyamagu-2021/argo-workflows-sandbox
[nagomiso-slackutils]: https://github.com/nagomiso/slackutils
[argo-wf-workflow-defaults]: https://argoproj.github.io/argo-workflows/default-workflow-specs/
[zenn-exit-handler]: https://zenn.dev/tommy/articles/33bc4fa134493e
[komi-exit-handler]: https://komi.dev/post/2022-01-09-introduction-to-argo-workflows/
[aaaanwz-exit-handler]: https://aaaanwz.github.io/post/2021/argo-workflows-exit-handler/
