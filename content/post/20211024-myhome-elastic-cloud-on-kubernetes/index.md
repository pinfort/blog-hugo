---
title: おうちk8sにElastic Cloud on Kubernetesを構築する
description: elasticsearchとkibanaを構築しました。
date: 2021-10-24
slug: myhome-elastic-cloud-on-kubernetes
tags:
  - k8s
---
## 用意するもの
- [データ保存先となるQNAPのNAS](https://blog.pinfort.me/posts/qnap-nas-10G)
- [構築済みのk8sクラスタ](https://blog.pinfort.me/categories/raspberry-pi-4-k8s)
- [k8sクラスタに構築済みのargoCD](https://blog.pinfort.me/posts/raspberry-pi-4-k8s-4)

## やったこと

### QNAPにPVとなるストレージ領域の作成とk8sへの登録
[QNAP の iSCSI Storage を自宅 Kubernetes の PersistentVolume にする](http://qiita.com/suzuyui/items/0efa505f3db03390f181)
この記事のとおり。LUNのサイズは10Giにした。記事と違って、nodeのOSがubuntuなので[iSCSI : iSCSI イニシエーターの設定](https://www.server-world.info/query?os=Ubuntu_20.04&p=iscsi&f=3)を参考にする。私の環境では、`open-iscsi` はすでにインストール済みだった。

#### 追記

ここで注意。今回のQNAP NASの場合、原因は不明だが一つのiqnに複数のLUNを用意してk8sの複数ノードから同時にアクセスしようとするとできなかった。そのため、必要なpvの数分だけiqnをわけ、lunは常にiqnに対して一つとなるように作製した。また、CHAP認証はうまく動かなかったのでこのあと無効にした。

```
apiVersion: v1
kind: Secret
metadata:
  name: iscsi-chap-secret
type: "kubernetes.io/iscsi-chap"
data:
  node.session.auth.username: "hoge"
  node.session.auth.password: "hoge"
```

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: iscsi-pv01
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  iscsi:
    targetPortal: hogeip:3260
    iqn: hogeiqn
    lun: 0
    readOnly: false
    chapAuthDiscovery: true
    chapAuthSession: true
    secretRef:
      name: iscsi-chap-secret

```

### ECKのデプロイ

[ECK(Elastic Cloud on Kubernetes) オンプレ動作確認](https://qiita.com/suzuyui/items/365fa09fc7c065fa642d)

argoCDをつかってデプロイする。まず、オペレータ用のネームスペースと、アプリケーション群用のネームスペースを作る。

```
kubectl create namespace elastic-system
kubectl create namespace elastic-stack
```

argoCDのプロジェクト`elastic-system`を作る。`https://kubernetes.default.svc`のelastic-stack, elastic-systemをdestinationに入れる。`CLUSTER RESOURCE ALLOW LIST`と`NAMESPACE RESOURCE ALLOW LIST`に\*,\*を設定するのが必要みたい。

argoCDのappをつくる。Helmをつかう。repository: https://helm.elastic.co chart: eck-operator バージョンはとりあえず1.8.xにしてみた。valuesはネームスペースだけ変更してやってみる。 `managedNamespaces: ["elastic-stack", "elastic-system"]`

権限とかで詰まったけどなんかいけた。argoCD上では、何回syncしてもoutOfSyncになっている。設定をすればこのdiffを無視できるらしいが、後でやることにしていったん放置する。

### elasticserchのデプロイ

iSCSIが正常に動作していることを確認する必要がある。各k8sノードのsshに入り、`open-iscsi`が`systemctl status open-iscsi`して正常に動作していることを確認する。動作していなければ、`sudo iscsiadm -m discovery -l`して、`QNAPのip:3260 via sendtargets`が表示されるか確認する、表示されなければ、`sudo iscsiadm -m discovery -t st -p QNAPのIP -o new`をして、追加して、`sudo systemctl restart open-iscsi`で再起動をする。このあと再度discoveryをして、登録されているか確認する。これを全ノードに行う。

#### 追記

iscsidが自動起動になっていないと、nodeを再起動したときに接続できなくなってしまうので `sudo systemctl enable iscsid` を全nodeに行う。


elasticsearchのkustomizeをつくる。

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: elastic-stack
commonLabels:
  app: elasticsearch-log

resources:
- elasticsearch.yaml

```

```
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch-log
spec:
  version: 7.15.1
  http:
    service:
      spec:
        type: LoadBalancer
  nodeSets:
  - name: default
    count: 2
    config:
      cluster.initial_master_nodes:
        - elasticsearch-log-es-default-0
      node.roles:
        - master
        - data
        - ingest
      node.store.allow_mmap: false
      xpack.security.enabled: true
      xpack.security.http.ssl.enabled: true
      xpack.security.transport.ssl.enabled: true
      xpack.monitoring.elasticsearch.collection.enabled: false
      xpack.monitoring.collection.enabled: true
      network.host: 0.0.0.0
    podTemplate:
      spec:
        containers:
        - name: elasticsearch
          resources:
            requests:
              memory: 100Mi
            limits:
              memory: 1Gi
        securityContext:
          runAsUser: 1234
          fsGroup: 1234
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 10Gi
        storageClassName: standard
```

CRI-O環境下で建てる場合、securityContextの設定がないと、chrootが失敗して起動しない。

SSL `xpack.security.enabled` を無効にすると、readness proveが失敗してpodが永遠にreadyにならない。以下のようなエラーになる。readness proveがhttpsでアクセスしようとしているものと思われる。

```
 "message": "readiness probe failed", "curl_rc": "35"
```

argoCDからappをつくり、つくったkustomizeのレポジトリを指定する。kibanaもあとでつくるので、REPOSITORY_ROOT/elasticsearch/baseにつくっておくと、同じレポジトリでkibanaもおけるのでべんり。argoCDのprojectで新しく作ったrepositoryもsource repositoryとして許可しておく必要がある。しばらく待ったら完了した。

### kibanaのデプロイ

elasticsearchと同じくkustomizeを書く。

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: elastic-stack
commonLabels:
  app: kibana-log

resources:
- kibana.yaml
```

```
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: kibana-log
spec:
  version: 7.15.1
  http:
    service:
      spec:
        type: LoadBalancer
  count: 1
  elasticsearchRef:
    name: elasticsearch-log
```

適当にargoCDで設定して、起動。しばらく待ってgreenになったら、`kubectl get svc -A` で割り当てられたIPを確認。192.168.2.53だった。`curl https://192.168.2.53：5601/login --insecure`してみる。みれた。ブラウザで`https://192.168.2.53：5601/login`を開くとログイン画面が見れた。` kubectl get secret -n elastic-stack elasticsearch-log-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode; echo`でパスワードを入手したら、ユーザー名elasticでログイン。ログインできた。

## おわりに
今回は、elasticsearch, kibanaの構築を行いました。実際にログを追加して活用するところは、別の記事にしたいと思います。お疲れさまでした。
