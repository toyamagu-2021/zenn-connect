---
title: "Conftest,Gatekeeperを用いたKubernetesマニフェストバリデーションと、Gatorを用いたテスト"
emoji: "🌸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kubernetes", "konstraint", "conftest", "gatekeeper", "gator"]
published: false
---

- [Abstract](#abstract)
- [Introduction](#introduction)
  - [Background](#background)
  - [Example](#example)
  - [Source code](#source-code)
- [Conftest](#conftest)
  - [Code](#code)
  - [Test](#test)
- [Konstraint](#konstraint)
- [Gator](#gator)
- [CI/CD](#cicd)
- [Test in a Kind cluster](#test-in-a-kind-cluster)
- [Conclusion](#conclusion)
- [References](#references)

## Abstract

Kubernetes (K8s) のマニフェストのバリデーションを行う手法を解説する。  

効率的な開発・リリースを行うため、K8sを採用する場合は通常CI/CDパイプラインの採用も行う。  
すなわち、コードの変更に応じて自動的にビルド・テスト・デプロイまで行われる前提となる。  
CI中でアプリケーションのビルド・テストが行われることはもちろん必要だが、同様にK8sマニフェストのテストやセキュリティチェックが行われることが望ましい。  
また、CD中で組織のポリシーに沿わないK8sマニフェストがデプロイされることも防ぐ必要がある。  

本解説ではOPA, Conftest, Konstraint, Gatekeeper, Gatorという技術スタックを用いて上記を自動的に実現する仕組みを、簡単なコードと共に解説する。

## Introduction

### Background

Abstractで記述したように、CI/CDパイプラインでK8sマニフェストのバリデーションを実施することは重要である。  
現在のところ、CI中で主要なツールは以下である。

- OPA[^opa]
  - Rego言語を用いたポリシーエンジン
  - K8s マニフェストの他に、APIやHCL(terraform等)の静的解析も実施することができる
- Conftest[^conftest]
  - OPAをベースとして動くCLIツール[^conftest-cncf]
    - Conftestのソースコードを確認すると、内部でRegoを呼び出していることを確認できる[^conftest-policy-engine]。
  - `opa` コマンドを直接実行するより簡単にOPAを利用することができる。
  - coverage等、 `opa test` で実装されている機能が利用できない場合がある[^mercari-introduce_conftest]。

CD中で主要なツールは以下である。

- Gatekeeper[^gatekeeper]
  - OPAをベースとして動くK8s admission 管理ソフトフェア
  - ベースとなるCRDを作成し( `ConstraintTemplate` )、そのCRDを利用することでデプロイされるリソースのバリデーションを行うことができる

CI・CDの片方だけで採用しても、もう片方でバリデーションが行えないと不便であったりリスクが考えられるため、CI/CD中では上記ソフトウェア群を併用するのが望ましい。  
ただし、OPAとGatekeeperはベースとなる言語は同一であるものの、書き方が微妙に異なり、単純に統合することは困難である。  
したがって、OPA用とGatekeeper用に2度ポリシーを記述する必要がある。  
この点は詳細な解説があるため省略する[^konstraint-why-exists] [^nikkei-advent20201224]。  

この点を解決するために、Konstraint[^konstraint]というライブラリが存在する。  
これは、Conftestの記法をGatekeeperに変換するライブラリである。  
このライブラリを採用することで以下のメリットが有る。

- CI/CD中で記述するポリシーコードが単一になる。  
- Rego言語を用いてポリシーのモジュール化を行うことができる。
- Rego言語を用いて簡単にコーディングやテストを行いつつ、Gatekeeperに自動的に変換することができる。  

反面、変換時にコードの可読性が下がるなど、デメリットも有る。  
しかし、総合的に判断して現状では有力な選択肢と考えられる。

上記に加えて、Gatekeeper v3.7.x からはGator[^gator]を用いてGatekeeperポリシー自体のテストを行うことができるようになった。
v3.9.x 時点では alpha 版であるが、Gatekeeperコミュニティによって精力的に開発が行われており、将来的に有力な選択肢になる可能性が高い。  

GatorでもCI中のバリデーションが行えるようになったため、ConftestやKonstraintが不要になるかという議論はあっても良いと考える。
この点、個人的な意見だが、コーディング・テストのしやすさ、coverageの導出などの手間を考えると、相補的に運用していくのが好ましいと思われる。

本解説では上記の技術スタックを用いて、K8sマニフェストのバリデーション基盤のCI/CDを行う方法を記述する。
全体感を重視し、各々の詳細な解説はスコープ外とする。

全体の流れとしては以下の通り。

1. [Confest](#conftest)を用いて、Rego言語に基づいてK8sマニフェストのバリデーションを行うコードを記述する。  
   確認として、ユニットテストの記述やopaを用いたカバレッジの測定、実際のK8sマニフェストのバリデーションを行う。  
1. [Konstraint](#konstraint)を用いてRego言語をGatekeeper形式に変更する。これにより、実際のK8sクラスタでadmissionを行うことができる。  
1. 生成されたGatekeeperマニフェストのテストとして、[Gator](#gator)を用いたテストを行う。
1. 上記を自動化するために、GitHub Actionsを用いた[CI](#cicd)を構築する。
1. 最終的な確認として、[kindクラスターを用いて生成されたマニフェストの確認](#test-in-a-kind-cluster)を行う。

### Example

K8sマニフェストのバリデーションの例としては、個人的な興味から有名なCDツールであるArgoCD[^argocd]を取り上げる。

ArgoCDにはProject[^argocd-project]という機能があるが、デフォルトで導入されている `default` Projectは権限の範囲が広く、削除不可であるという難点がある（変更は可能）。  
このため、`default` Projectを利用不可にしてしまいたい[^footnote-argocd]。  
具体的には、以下の `application.yaml` をデプロイ不可とする。  
`.spec.project` の値にのみ注目すればよい。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bad-application
  namespace: argocd
spec:
  destination:
    namespace: default
    server: https://kubernetes.default.svc
  project: default # <- This will be not allowed
  source:
    path: kustomize-guestbook
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: main
```

### Source code

本解説で用いるソースコードは全てGithub 上にある[^toyamagu-konstraint-examples]。

## Conftest

### Code

まず、Rego言語を用いて `default` Projectへのデプロイを禁止にする。  
コードの例を以下に示す[^toyamagu-konstraint-examples-argocd]。

```rego:src.rego
# METADATA
# title: Deny ArgoCD Application deployed in the default project
# description: |-
#   Deny ArgoCD application deployed in the default project.
# custom:
#   matchers:
#     kinds:
#     - kinds:
#       - Application
#       apiGroups:
#       - "argoproj.io"
package argocd.deny_app_in_default_project

import data.lib.core

violation[{"msg": msg}] {
  name := core.resource.metadata.name
  project := core.resource.spec.project
  project == "default"
  msg := sprintf("Error: %s ArgoCD Application is not permitted to use default ArgoCD project.", [name])
}
```

大体は読めば理解できると考えるが、ポイントは以下の通り。

- Konstraintを用いるため、METADATAにGatekeeper用の設定を記述すること。
- `import data.lib.core` を用いて、Konstraintのライブラリをimportすること。
  - `conftest pull github.com/plexsystems/konstraint/examples/lib -p lib` でpullできる[^nikkei-advent20201224]。
- ConftestとGatekeeperの差分を吸収するために、 `input.metadata.name` ではなく、 `core.resource.metadata.name` として上記のライブラリを用いること。
  - この事情はKonstraint公式に詳しく記述されている[^konstraint-why-exists]。

簡単なテストも記述しておく。解説は省略する。

```rego:src_test.rego
package argocd.deny_app_in_default_project

test_violation {
  input := {
    "apiVersion": "argoproj.io/v1alpha1",
    "kind": "Application",
    "metadata": {"name": "test-app"},
    "spec": {"project": "default"},
  }

  violation[{"msg": "Error: test-app ArgoCD Application is not permitted to use default ArgoCD project."}] with input as input
}

test_no_violation {
  input := {
  "apiVersion": "argoproj.io/v1alpha1",
  "kind": "Application",
    "metadata": {"name": "test-app"},
    "spec": {"project": "not-default"},
  }

  not violation[{"msg": "Error: test-app ArgoCD Application is not permitted to use default ArgoCD project."}] with input as input
}
```

### Test

コーディングが完了したので、テストを行う（前述の通り、Konstraintのライブラリをダウンロードするのを忘れないこと）。  
手元の環境では他にもいくつかテストを追加しているため、outputの値は違うことに注意。

- Conftest
  - テスト: `conftest verify -p .`

      ```bash:output
      34 tests, 34 passed, 0 warnings, 0 failures, 0 exceptions, 0 skipped
      ```

    - 失敗する場合は `--trace` オプションをつけてデバッグすると良い。
- OPA
  - テスト: `opa test . --ignore *.yaml`

    ```bash:output
    PASS: 34/34
    ```

    - 後に配置するyamlファイルをテストから除くため、ignoreとしている。

  - coverage: `opa test . --ignore *.yaml  --coverage -v | jq .coverage` [^mercari-introduce_conftest]

    ```bash:output
    89.4
    ```
  
  - マニフェストチェック(`default` Projectのため失敗): `conftest test -p . --namespace  argocd.deny_app_in_default_project ./argocd/argocd-deny-app-in-default-project/tests/disallowed/aplication-in-default-prj.yaml`

    ```bash:output
    FAIL - ./argocd/argocd-deny-app-in-default-project/tests/disallowed/aplication-in-default-prj.yaml - argocd.deny_app_in_default_project - Error: bad-application ArgoCD Application is not permitted to use default ArgoCD project.
    ```

  - マニフェストチェック(`non-default` Projectのため成功): `conftest test -p . --namespace  argocd.deny_app_in_default_project ./argocd/argocd-deny-app-in-default-project/tests/allowed/aplication-in-non-default-prj.yaml`

    ```bash:output
    1 test, 1 passed, 0 warnings, 0 failures, 0 exceptions
    ```

上記により OPA, Conftestを用いたテストの実行、及び実際のマニフェストのチェックが完了した。

## Konstraint

上記コードを元にKonstraintを実行し、Gatekeeperマニフェストの生成を行う。

```bash
konstraint doc .
konstraint create .
```

自動的に `policy.md`, `template.yaml`, `constraint.yaml` が作成されたはずである。

```bash
$ ls ./argocd/argocd-deny-app-in-default-project/
constraint.yaml  src.rego  src_test.rego  suite.yaml  template.yaml  tests
```

## Gator

Gatorを用いて生成された `template.yaml`, `constraint.yaml` のテストを行う。  
`gator test` と `gator verify` があるが、多くのテストをシステマチックに実行する場合は `verify` のほうが便利だと思うので、こちらを利用する[^gator]。  
v3.9.xの時点ではα版であることを再度注意しておく。

`suite.yaml` を作成し、これを元にテストを行う[^gator-suites]。
ファイルは以下の通りである。読めばだいたいわかるように、Suite、tests、casesからなる。

- Suite
  - テスト用のCRD
- tests
  - caseのまとまり。test毎にGatekeeper `ConstraintTemplate` リソースと `Constraints` リソースを指定する。
- case(詳細は公式ドキュメント参照[^gator-case])
  - テストケース。テストする対象のマニフェストを指定する。
  - assertionsでviolationの回数を指定できる。yesはat least once。
  - messageでエラーメッセージの1部を指定できる。

```yaml:suite.yaml
kind: Suite
apiVersion: test.gatekeeper.sh/v1alpha1
tests:
- name: deny-app-default-prj
  template: template.yaml
  constraint: constraint.yaml
  cases:
  - name: allowed-non-default-prj
    object: "./tests/allowed/aplication-in-non-default-prj.yaml"
    assertions:
    - violations: no
  - name: disallowed-default-prj
    object: "./tests/disallowed/aplication-in-default-prj.yaml"
    assertions:
    - violations: yes
    - message: "ArgoCD Application is not permitted to use default ArgoCD project."
      violations: 1
```

Gatorの実行は容易。

```bash
gator verify ./argocd/argocd-deny-app-in-default-project/
```

```bash:output
ok      home/toyamagu/github.com/toyamagu-2021/konstraint-examples/argocd/argocd-deny-app-in-default-project/suite.yaml 0.017s
PASS
```

上記のように成功していることがわかる。  

ConftestからGatorまで、全てをいちいち手で実行するのは面倒なため、スクリプトを作成した[^test-and-run]。

## CI/CD

ここでのCI/CDは上記のKonstraint, Gatorなどの実行を自動化することを指す。  
素直にコード化すれば良く、解説は省略するが、GitHub Actionsを用いて自動化を行った[^konstraint-github-actions]。  
PRに coverage をコメントさせる等の処置を行ってある。  
GatekeeperのテストをGatorのみでなくよりきちんとやるのであれば、 Kind などを用いたローカルクラスタでのE2Eテストをしておくとより安全と考えられる。

## Test in a Kind cluster

KindクラスターにArgoCDとGatekeeperをデプロイし、動作テストを行う。

- クラスター作成とArgoCD・Gatekeeperデプロイ。スクリプトを作成した[^kind-with-argocd-and-gatekeeper]

- マニフェスト apply

  ```bash
  $ k apply -f ./argocd/argocd-deny-app-in-default-project/template.yaml 
  constrainttemplate.templates.gatekeeper.sh/argocddenyappindefaultproject created
  $ k apply -f ./argocd/argocd-deny-app-in-default-project/constraint.yaml 
  argocddenyappindefaultproject.constraints.gatekeeper.sh/argocddenyappindefaultproject created
  ```

- 失敗例

  ```bash
  $ k apply -f ./argocd/argocd-deny-app-in-default-project/tests/disallowed/aplication-in-default-prj.yaml
  Error from server (Forbidden): error when creating "./argocd/argocd-deny-app-in-default-project/tests/disallowed/aplication-in-default-prj.yaml": admission webhook "validation.gatekeeper.sh" denied the request: [argocddenyappindefaultproject] Error: bad-application ArgoCD Application is not permitted to use default ArgoCD project
  ```

- 成功例

  ```bash
  $ k apply -f ./argocd/argocd-deny-app-in-default-project/tests/allowed/aplication-in-non-default-prj.yaml 
  application.argoproj.io/good-application created
  ```

## Conclusion

本解説では、K8sマニフェストのバリデーションを行う方法、またそのCI/CDの方法を記述した。  
ConftestやKonstraintを用いた、優れた解説は以前からあった [^nikkei-advent20201224] [^mercari-introduce_conftest]が、本解説では異なる例や、Github Actionsの例を含めて記述した。
加えて、GatorというCLIツールが加わることにより、容易にCI中でGatekeeperマニフェストのテストが行えるようになった。  
しかし、コーディング・テストの容易性という観点から、依然としてConftestやKonstraintの利点は多くあると思われる。

今回用いたのはArgoCDでデプロイできるProjectを制限するという、トリビアルな例だった。  
他にも、CDツールであるArgoCDにはAutoSync等場合によっては(本番環境など)セキュリティ上disableとしたい機能がある。  
このように、環境に応じて適切な機能制限を加えることは、Gatekeeperの重要な役割であると考えられる。本番運用する際の導入を今後も検討していきたい。

## References

[^opa]: <https://www.openpolicyagent.org/>
[^conftest]: <https://www.conftest.dev/>
[^conftest-cncf]: <https://www.cncf.io/blog/2020/07/23/conftest-joins-the-open-policy-agent-project/#:~:text=Conftest%20fits%20nicely%20into%20the,lots%20of%20different%20use%20cases.>
[^gatekeeper]: <https://open-policy-agent.github.io/gatekeeper/website/docs/>
[^konstraint-why-exists]: <https://github.com/plexsystems/konstraint#why-this-tool-exists>
[^nikkei-advent20201224]: <https://hack.nikkei.com/blog/advent20201224/>
[^konstraint]: <https://github.com/plexsystems/konstraint>
[^gator]: <https://open-policy-agent.github.io/gatekeeper/website/docs/gator/>
[^argocd]: <https://argo-cd.readthedocs.io/en/stable/>
[^argocd-project]: <https://argo-cd.readthedocs.io/en/stable/user-guide/projects/>
[^footnote-argocd]: RBACを用いて制限するという手もあるが、ArgoCD v2.4.xでapp of apps パターンを用いる場合などは、開発者がhackできる可能性がある。
[^toyamagu-konstraint-examples]: <https://github.com/toyamagu-2021/konstraint-examples>
[^toyamagu-konstraint-examples-argocd]: <https://github.com/toyamagu-2021/konstraint-examples/blob/main/argocd/argocd-deny-app-in-default-project/>
[^mercari-introduce_conftest]: <https://engineering.mercari.com/blog/entry/introduce_conftest/>
[^gator-suites]: <https://open-policy-agent.github.io/gatekeeper/website/docs/gator#suites>
[^test-and-run]: <https://github.com/toyamagu-2021/konstraint-examples/blob/main/scripts/test-and-run.sh>
[^konstraint-github-actions]: <https://github.com/toyamagu-2021/konstraint-examples/blob/main/.github/workflows/konstraint.yaml>
[^kind-with-argocd-and-gatekeeper]: <https://github.com/toyamagu-2021/konstraint-examples/blob/main/scripts/kind-with-argocd-and-gatekeeper.sh>
[^conftest-policy-engine]: <https://github.com/open-policy-agent/conftest/blob/108edfe44f247c2048ed7247f6ea28cea72bcb26/policy/engine.go#L401>
[^gator-case]: <https://open-policy-agent.github.io/gatekeeper/website/docs/gator/#cases>
