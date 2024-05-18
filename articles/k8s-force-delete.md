---
title: "Pod force deleteは `force --grace-period=0` でなく、 `--now` を使おうという話"
emoji: "🌸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Kubernetes"]
published: true
---

## Abstract

* KubernetesでPodを即座に削除したいとき、 `--force --grace-period=0` がよく紹介される
* が、このコマンドを実行しても、現行のv1.30時点ではコンテナやプロセスとしては `pod.spec.terminationGracePeriodSeconds` 分生き残ってしまう（Kubernetesからは消えるので、Podとしては消える）
* 特に意図がなければ、`--now` オプションまたは同等の `--grace-period=1` のほうがよい
* v1.30時点ではevict時もプロセスが生き残ってしまうが、これはv1.31で修正されそう

## BackGround

K8sでPodを削除したいとき、 `kubectl delete pod <pod-name> --force --grace-period=0` というコマンドがよく検索結果に出てくる
これはPodを即座に削除するためのコマンドで、 `--force` でPodを即座に削除し、 `--grace-period=0` でGrace Periodを0秒に設定している
実際Podはすぐに消えているように見えるが、実はK8sのversionによっては、コンテナや実行しているプロセスは生き残ってしまっている
特殊な事情がなければ、`delete --now` オプションまたは同等の `delete --grace-period=1` のほうがよさそう

### Related Issue and PRs

Issueとしては、1. evictされたときにもprocessが生き残ってしまう[^115819]、ものや、 2. `--force --grace-period=0` を設定してもprocessが生き残ってしまう[^120449] あたりに記載されている
1.に対してはMergeされたPRがあり [^119570] [^124063] 、v1.31あたりで解消されていそう。
2.に対してはまだMergeされていない [^120449] が、上記PR [^124063] で一緒に治っているようにも見える。。。？

### Root cause

[ここ](https://github.com/kubernetes/kubernetes/blob/227c2e7c2b2c05a9c8b2885460e28e4da25cf558/pkg/kubelet/pod_workers.go#L998-L1000) で、 `grace-period` が0のときに `pod.Spec.TerminationGracePeriodSeconds` を拾ってきてしまっているからぽい。
導入されたPR [^102344] は K8s v1.22に含まれていることから、 v1.21以前では `--force --grace-period=0` でPodを削除すると、Podが消えてもコンテナやプロセスもkillされていた。
K8s v1.22以降では上記のissue群が発生しているようだ。

実は `--force` をつけずに `--grace-period=0` のみ設定すれば上記の問題はおこらない。
というのも、client(kubectl)側では `--grace-period=0` は実質的には `--grace-period=1` だからである [kubectl参考](https://github.com/kubernetes/kubectl/blob/5b7c8b24b4361a97ab19de1d1e301a6c1bbaed1a/pkg/cmd/delete/delete.go#L189-L193)。
CRIの都合上 `gracePeriod=0` にはできず、実はServer側でも`gracePeriod=1` に強制的に設定している [Ref](https://github.com/kubernetes/kubernetes/blob/a31030543c47aac36cf323b885cfb6d8b0a2435f/pkg/kubelet/pod_workers.go#L1004-L1007)。
なので、 `--grace-period=0` も `--grace-period=1` も本質的な違いはない

一方で、 `--force` はAPIサーバから強制的にPodを消去してしまう。これが根本的な問題なのだと思われる。

## Verification

以下のyamlとMakefileで試せる。要kind。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  terminationGracePeriodSeconds: 10
  containers:
  - name: nginx
    image: nginx:1.14.2
    command: [ 'sleep', '12345']
    ports:
    - containerPort: 80
```

```Makefile
k8s_30=v1.30.0
k8s_29=v1.29.4
k8s_28=v1.28.9
k8s_27=v1.27.13
k8s_26=v1.26.15
k8s_25=v1.25.16
k8s_24=v1.24.17
k8s_23=v1.23.17
k8s_22=v1.22.17
k8s_21=v1.21.14

VERSION ?= 30
K8S_VESRION := $(k8s_$(VERSION))

cluster.create:
	kind create cluster --name ${VERSION} --image kindest/node:${K8S_VESRION}

cluster.delete:
	kind delete cluster --name ${VERSION}

pod.create:
	kubectl apply -f pod.yaml

pod.delete.force:
	date; kubectl delete po nginx --force	--grace-period=0

pod.delete.not-force:
	date; kubectl delete po nginx --grace-period=0

pod.delete.now:
	date; kubectl delete po nginx --now

docker.bash:
	docker exec -it ${VERSION}-contro-plane bash

docker.ps:
	docker exec -it ${VERSION}-control-plane bash -c "while true; do date; ps -ef | grep 'sleep 12345'; sleep 1; done"
```

### K8s v1.21

`--force grace-period=0` でもプロセスが即座にkillされる。

1. `make cluster.create VERSION=21`
2. `make pod.create`
3. `make docker.ps VERSION=21`
4. `make pod.delete.force`

```console
$ make pod.delete.force
date; kubectl delete po nginx --force --grace-period=0
Sun May 19 01:51:31 JST 2024
Warning: Immediate deletion does not wait for confirmation that the running resource has been terminated. The resource may continue to run on the cluster indefinitely.
pod "nginx" force deleted
```

```console
$ make docker.ps
Sat May 18 16:51:33 UTC 2024
root        4173    4122  0 16:28 ?        00:00:00 sleep 12345
root        4466       0  0 16:51 pts/1    00:00:00 bash -c while true; do date; ps -ef | grep 'sleep 12345'; sleep 1; done
root        4586    4466  0 16:51 pts/1    00:00:00 grep sleep 12345
Sat May 18 16:51:34 UTC 2024
root        4466       0  0 16:51 pts/1    00:00:00 bash -c while true; do date; ps -ef | grep 'sleep 12345'; sleep 1; done
root        4714    4466  0 16:51 pts/1    00:00:00 grep sleep 12345
Sat May 18 16:51:35 UTC 2024
root        4466       0  0 16:51 pts/1    00:00:00 bash -c while true; do date; ps -ef | grep 'sleep 12345'; sleep 1; done
root        4718    4466  0 16:51 pts/1    00:00:00 grep sleep 12345
```

上記のように、Podをforce deleteしてもプロセスは即座にkillされている。

### K8s >= v1.22

`--force grace-period=0` でもプロセスが即座にkillされる。

1. `make cluster.create VERSION=22`
2. `make pod.create`
3. `make docker.ps VERSION=22`
4. `make pod.delete.force`

```console
$ make pod.delete.force
date; kubectl delete po nginx --force   --grace-period=0
Sun May 19 01:54:29 JST 2024
Warning: Immediate deletion does not wait for confirmation that the running resource has been terminated. The resource may continue to run on the cluster indefinitely.
pod "nginx" force deleted
```

```console
$ make docker.ps
Sat May 18 16:54:37 UTC 2024
root        5446    5396  0 16:54 ?        00:00:00 sleep 12345
root        5467       0  0 16:54 pts/1    00:00:00 bash -c while true; do date; ps -ef | grep 'sleep 12345'; sleep 1; done
root        5569    5467  0 16:54 pts/1    00:00:00 grep sleep 12345
Sat May 18 16:54:38 UTC 2024
root        5446    5396  0 16:54 ?        00:00:00 sleep 12345
root        5467       0  0 16:54 pts/1    00:00:00 bash -c while true; do date; ps -ef | grep 'sleep 12345'; sleep 1; done
root        5575    5467  0 16:54 pts/1    00:00:00 grep sleep 12345
Sat May 18 16:54:39 UTC 2024
root        5467       0  0 16:54 pts/1    00:00:00 bash -c while true; do date; ps -ef | grep 'sleep 12345'; sleep 1; done
root        5666    5467  0 16:54 pts/1    00:00:00 grep sleep 12345
```

上記のような感じで、29秒時点に消しても 38秒時点までプロセスが生き残っている。 `--now` だとどうだろうか？

5. `make pod.create`
6. `make docker.ps VERSION=22`

```console
$ make pod.delete.now
date; kubectl delete po nginx --now
Sun May 19 01:56:52 JST 2024
pod "nginx" deleted
```

```console
Sat May 18 16:56:53 UTC 2024
root        5821    5771  0 16:56 ?        00:00:00 sleep 12345
root        5836       0  0 16:56 pts/1    00:00:00 bash -c while true; do date; ps -ef | grep 'sleep 12345'; sleep 1; done
root        5920    5836  0 16:56 pts/1    00:00:00 grep sleep 12345
Sat May 18 16:56:54 UTC 2024
root        5836       0  0 16:56 pts/1    00:00:00 bash -c while true; do date; ps -ef | grep 'sleep 12345'; sleep 1; done
root        6062    5836  0 16:56 pts/1    00:00:00 grep sleep 12345
```

上記のように、52秒時点で消せば54秒時点で消している。したがって、 `--now` だと（ほぼ）即座にプロセスもkillされる
このふるまいはv1.22~v1.30まで同じである
ちなみに、root causeで議論したように、 `--force` をつけずに `--grace-period=0` のみであれば、プロセスも即座に終了する。

## Summery

特殊な事情がなければ `delete --now` をつかいましょう

## References

[^115819]: <https://github.com/kubernetes/kubernetes/issues/115819>
[^120449]: <https://github.com/kubernetes/kubernetes/issues/120449>
[^124063]: <https://github.com/kubernetes/kubernetes/pull/124063>
[^119570]: <https://github.com/kubernetes/kubernetes/pull/119570>
[^120451]: <https://github.com/kubernetes/kubernetes/pull/120451>
[^108741]: <https://github.com/kubernetes/kubernetes/issues/108741>
[^102344]: <https://github.com/kubernetes/kubernetes/pull/102344>
