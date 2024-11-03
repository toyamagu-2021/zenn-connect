---
title: "Kubectl debugを使ってPodやNodeを気軽にdebugしよう"
emoji: "🌸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Kubernetes", "EKS"]
published: false
---

## Abstract

Kubernetes (K8s) を運用する上でNodeは意識しなくて良いものです.
一方で、Podレベルの問題ではなく、Nodeレベルの問題を調査したいときはNodeに入る必要があります.

* iptablesなどのNW問題
* Node横断のログ調査
* ノードレベルでインストーするミドルウェアの調査

本記事ではNodeに入って調査する方法や、ついでにPodに入る方法を共有します. 具体例として以下を紹介します

* NodeのIP tablesを参照する方法
* EKS Logs Collectorを使ってログを取得し、AWSサポートに投稿する方法 [^aws-gomen]

## What is `kubectl debug`

### For Pods

`kubectl debug` はKubernetesのPodやNodeに対してデバッグを行うためのツールである.
これ自体に関する解説記事は優れたものがたくさんある. 例えば意義やPodの場合の使い方は [^kubernetes-ephemeral-containers-intro] が優れている.

簡単にまとめると以下の通りである。

* 通常 `kubectl exec` を使ってコンテナの中に入るが、distressイメージを使っていてshellが入っていない場合や、debugのためのコマンドがない場合など、`kubectl debug` を使うことで任意のcontainerを追加することができる
* `kubectl debug <pod>` は、PodのSideCarとして ephemeral container を追加し、PodのSideCarとしてコンテナを立ち上げることができる
  * 開発環境などサクッと入りたい時に便利
* `kubectl debug <pod> --copy-to debug-pod --share-processes` は、Podをcopyすることができる. copyしたPodは、上記の `kubectl debug` で入ることができる
  * 本番環境など、Podのサイドカーを建てたくない場合に便利
  * CopyしたPodの消し忘れに注意

具体例としては以下の通り

* Podに入りたい時
  * `k debug -it <pod> --image alpine -- sh`
* copyしたPodをたててに入りたい時 (本番環境でおすすめ)
  * `k debug -it <pod> --copy-to debug-$(date +%s) --share-processes --image alpine -- sh`

### For Nodes

Nodeに入る場合もほぼ同様である [^kubectl-node-debug].

* `kubectl debug node/${node}` でNodeに入ることができる
  * ログの収集であればこれで十分
* 更なる特権を付与したい場合は、 profileをつける.
  * `kubectl debug node/${node} --profile=sysadmin` (全権限)
  * `kubectl debug node/${node} --profile=netadmin` (`iptables` を叩ければいい時など)

具体例

#### Collect log by EKS Logs Collector

AWSに問い合わせをすると提出をお願いされる [^eks-log-collector] を使ってログを取得する方法.

1. `kubectl debug -iti nodes/<node_name> --image amazon/aws-cli -- bash`
1. `chroot /host`
2. `curl -O https://raw.githubusercontent.com/awslabs/amazon-eks-ami/main/log-collector-script/linux/eks-log-collector.sh`
3. `bash eks-log-collector.sh`
4. ログファイルをコピーする。
  * `aws s3 cp` でコピーする方法(おすすめ): `aws s3 cp  /host/var/log/<log_file_name> s3://<bucket_name>/`
  * `kubectl cp` でコピーする方法
    * `yum install -y tar`
    * debugしているコンソールに戻って、 `kubectl cp <debug_pod_name>:/host/var/log/<log_file_name> ./<log_file_name>`
5. AWSサポートに解析をお願いする

#### Check iptables

iptablesなどをチェックしたい時は profile をつける

1. `k debug -it nodes/<node_name> --image alpine --profile=sysadmin -- sh`
  * `netadmin` で十分
1. `apk add iptables`
1. `iptables -L -n -v`

```shell
# iptables -nL -v
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
 575K   35M KUBE-PROXY-FIREWALL  0    --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate NEW /* kubernetes load balancer firewall */
  57M 8931M KUBE-NODEPORTS  0    --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes health check service ports */
 575K   35M KUBE-EXTERNAL-SERVICES  0    --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate NEW /* kubernetes externally-visible service portals */
  57M 8934M KUBE-FIREWALL  0    --  *      *       0.0.0.0/0            0.0.0.0/0
<...>
```

[^aws-gomen]: これの投稿によってサポートの人の負荷が増えたらすみません
[^kubernetes-ephemeral-containers-intro]: <https://developer.mamezou-tech.com/blogs/2022/08/25/kubernetes-ephemeral-containers-intro/>
[^kubectl-node-debug]: <https://kubernetes.io/ja/docs/tasks/debug/debug-cluster/kubectl-node-debug/>
[^eks-log-collector]: <https://github.com/awslabs/amazon-eks-ami/blob/main/log-collector-script/linux/README.md>
