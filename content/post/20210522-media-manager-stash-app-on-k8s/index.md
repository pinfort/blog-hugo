---
title: メディア管理アプリケーション stashを構築 on k8s
description: ローカル環境に、メディア管理ソフトstashを構築しました。その記録です。
date: 2021-05-22
slug: media-manager-stash-app-on-k8s
tags:
  - anime
  - k8s
---
## stashとは
OSSのメディア管理ソフトウェアです。もともと、えっちなメディアはクラウドストレージに置けないから自分で管理したいよねという思いから始まったプロジェクトのようで、公式サイトにNSFWな画像が含まれるなどしていますが、普通にメディア管理ソフトウェアとして優秀そうなので紹介、構築したいと思います。

- [公式サイト(NSFW, R-18)](https://stashapp.cc)
- [GitHub](https://github.com/stashapp/stash)

## 構築
以前の記事で、k8sクラスタ, argoCD, QNAPによるNASを構築しました。これらを使ってstashを構築していきます。

## Nodeの作業
今回、NFSを使うので、必要なソフトウェアをk8sの各ノードにインストールします。
```
sudo apt-get install nfs-common -y
```

## NAS設定

メディアを保存するための共有フォルダを作成します。今回、stashのmetadataを保存するstashapp共有フォルダと、メディアファイルを保存するmedia共有フォルダ、設定保存用にstashapp-configフォルダを作成します。

サーバーからアクセスできるようにNFS(v4)をNAS全体で有効化します。

先ほど作った3つの共有フォルダにNFSの権限設定をします。今回は、k8sクラスタの3つのipアドレスに対して、読み書きを許可しました。


## Kustomize

k8sにデプロイするKustomizeファイルを作成します。
gitlabにレポジトリを作成し、baseディレクトリを作成しています。将来overlaysディレクトリが作成可能なようにこのようにしていますが、追加することはないかもしれません。

ディレクトリ構造は以下のようです。

    - base
       - configMap.yaml
       - deployment.yaml
       - kustomization.yaml
       - service.yaml

configMap.yaml
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: stashapp-map
data:
  data_directory: "/mnt/data"
  generated_content_directory: "/mnt/meta/generated"
  metadata_directory: "/mnt/meta/metadata"
  cache_directory: "/mnt/meta/cache"
```

deployment.ymal
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stashapp-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      deployment: stashapp-deployment
  template:
    metadata:
      labels:
        deployment: stashapp-deployment
    spec:
      containers:
      - name: stashapp-container
        image: stashapp/stash:latest
        ports:
        - containerPort: 9999
        env:
        - name: STASH_STASH
          valueFrom:
            configMapKeyRef:
              name: stashapp-map
              key: data_directory
        - name: STASH_GENERATED
          valueFrom:
            configMapKeyRef:
              name: stashapp-map
              key: generated_content_directory
        - name: STASH_METADATA
          valueFrom:
            configMapKeyRef:
              name: stashapp-map
              key: metadata_directory
        - name: STASH_CACHE
          valueFrom:
            configMapKeyRef:
              name: stashapp-map
              key: cache_directory
        volumeMounts:
          - name:  stashapp-meta-volume
            mountPath: /mnt/meta
          - name:  stashapp-data-volume
            mountPath: /mnt/data
          - name:  stashapp-config-volume
            mountPath: /root/.stash
      volumes:
        - name: stashapp-meta-volume
          nfs:
            server: $NAS_IP
            path: /stashapp
        - name: stashapp-data-volume
          nfs:
            server: $NAS_IP
            path: /media
        - name: stashapp-config-volume
          nfs:
            server: $NAS_IP
            path: /stashapp-config
```

kustomization.yaml
```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: stashapp
commonLabels:
  app: stashapp

resources:
- deployment.yaml
- service.yaml
- configMap.yaml
```

service.yaml
```
kind: Service
apiVersion: v1
metadata:
  name: stashapp-service
spec:
  selector:
    deployment: stashapp-deployment
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9999

```

## argoCDでデプロイ

先に、k8sでnamespaceを作成します。
```
kubectl create namespace stashapp
```

argoCDのUIから、Kustomizeを置いたレポジトリを登録します。

argoCDのUIから、project、stash-appを作成します。以下内容です。

DESTINATIONS 

    Server: https://kubernetes.default.svc
    Namespace: stashapp

CLUSTER RESOURCE ALLOW LIST

    Kind: *
    Group: *

SOURCE REPOSITORIES

    Kustomizeを置いたレポジトリのurl


argoCDのUIから、applicationを作成します。以下内容です。

    Application name: 任意
    project: さっき作成したprojectがsugegstされるので選択
    sync policy: manual
    Repository URL: さっき登録したレポジトリがsuggestされるので選択
    Revision: master
    Path: base
    Cluster URL: https://kubernetes.default.svc
    namespace: stashapp

baseディレクトリの中身を直接使用します。登録ができたらcreateします。
out of syncの状態になるのでsyncします。

無事healthyになったら、kubectlでipを確認して見に行きます。
無事初期設定画面が出れば成功です。
