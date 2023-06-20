---
title: ラズパイのUbuntu 22.04にk3sをインストールする
description: ラズパイでk3sクラスタを作成します
date: 2023-06-20
slug: raspi-k3s
---
## 環境
raspberry pi4三台。すべてubuntu 22.04 64bit。

```
$ cat /etc/os-release
PRETTY_NAME="Ubuntu 22.04.2 LTS"
NAME="Ubuntu"
VERSION_ID="22.04"
VERSION="22.04.2 LTS (Jammy Jellyfish)"
VERSION_CODENAME=jammy
ID=ubuntu
ID_LIKE=debian
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
UBUNTU_CODENAME=jammy
```

## OSインストール
適当にする。 `cgroup_memory=1 cgroup_enable=memory` の記載は実施

## パッケージインストール
最近のubuntuではvxlanパッケージがデフォルトでインストールされておらず、このままでは起動に失敗するので外套のパッケージをインストール

```
sudo apt install linux-modules-extra-raspi
```

## serverにk3sインストール

指定のディレクトリに設定ファイルを置けば反映してくれるのでこれをおいて実施。 [設定ファイルについて](https://docs.k3s.io/installation/configuration#multiple-config-files)

```
sudo mkdir -p /etc/rancher/k3s/config.yaml.d
```

LBには後でmetallbを入れるのでデフォルトのLBを無効に。ルートユーザー以外でもkubectlできるようにkubeconfigのread権限を付与。これはサーバーだけに設置し、agentノードには設置しない。設置したらそのようなオプションはないと言われてagentが起動しない。
no-deployオプションはdisableオプションに名前が変わったのでそちらを使う。このあとインストールするMetalLBに必要なのでkube-proxyをstrict-arpにする。
```
$ cat /etc/rancher/k3s/config.yaml.d/custom.yaml
disable:
  - servicelb
write-kubeconfig-mode: 644
kube-proxy-arg:
  - proxy-mode=ipvs
  - ipvs-strict-arp=true
```

インストール実施
```
curl -sfL https://get.k3s.io | sh -
```

しばらく時間がかかるが、`kubectl get nodes`で正常にnodeが見れるようになったら完了。`journalctl -e`でログが見れる。

トークンを確認
```
sudo cat /var/lib/rancher/k3s/server/node-token
```

## agentにk3sインストール
今回HA構成にはしないので、単純にインストール

https?と思ったけど通ったのでこれでいいっぽい
```
$ curl -sfL https://get.k3s.io | K3S_URL=https://serverIP:6443 K3S_TOKEN=hogehoge sh -s - --resolv-conf=/run/systemd/resolve/resolv.conf
```

agentの二台ともに実施してnodeが見えたらOK.
```
$ kubectl get nodes
NAME      STATUS   ROLES                  AGE     VERSION
k8srpi2   Ready    <none>                 4m21s   v1.26.5+k3s1
k8srpi1   Ready    control-plane,master   2d22h   v1.26.5+k3s1
k8srpi3   Ready    <none>                 50s     v1.26.5+k3s1
```

## ローカルからkubectlできるようにする
今回はローカルからこれを操作できるようになりたいので、サーバーノードの `/etc/rancher/k3s/k3s.yaml` を持ってきて、127.0.0.1になっているサーバーアドレスを書き換えてローカルの.kube/configに入れておく。

## MetalLBを入れる
[k3s+MetalLBの環境を構築してKubernetes-Dashboardをデプロイする](https://qiita.com/ussvgr/items/b98ada65563edf77f12b) を参考にする。
[metallb.universe.tf](https://metallb.universe.tf/installation/) も参考にする

最新を入れる
```
$ kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.10/config/manifests/metallb-native.yaml
```

設定ファイルの書き方がqiitaから変わっているようなので公式を見て書く。特に特別な設定はせずL2で設定。

```yml:metallb-ippool.yml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.2.50-192.168.2.99

```

```yml:metallb-l2config.yml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
```

たぶんこれでよさそう。

webhookがポート443を占領してしまうのでloadBalancerを使用するように変更

```
kubectl patch svc -n metallb-system webhook-service -p '{\"spec\": {\"type\": \"LoadBalancer\"}}'
```

## k8s dashboardを入れる

[k3s docs](https://docs.k3s.io/installation/kube-dashboard)を参考にする。
最新がv2.7.0なので `https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml` を持ってくる。
今入れたロードバランサを使用するのでtypeだけ追記して適用する。

```diff
 kind: Service
 apiVersion: v1
 metadata:
   labels:
     k8s-app: kubernetes-dashboard
   name: kubernetes-dashboard
   namespace: kubernetes-dashboard
 spec:
   ports:
     - port: 443
       targetPort: 8443
+  type: LoadBalancer
   selector:
     k8s-app: kubernetes-dashboard
```

アカウント設定

```yml:dashboard.yml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

applyしたらトークン取得

```
kubectl -n kubernetes-dashboard create token admin-user
```

ip確認

```
kubectl get services -A
```

今回dashboardは192.168.2.51が割り当てられたので、 https://192.168.2.51 にアクセスし、ブラウザの自己証明書の警告を無視すればトークン入力画面がでる。さっき取得したトークンを入力したら見れるようになった。

確認が完了したのでdashboard関連はいったん削除。

## argocdを入れる
公式を見てその通りにやる
```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.7.5/manifests/install.yaml
```

ロードバランサーを使用

```
kubectl patch svc argocd-server -n argocd -p '{\"spec\": {\"type\": \"LoadBalancer\"}}'
```

IPが無事割り当てられたので自己証明書の警告を無視して見に行けばログイン画面が見れた。

初期パスワードを取得
```
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data}'
```

adminアカウントでログイン出来たらパスワードを変えておこう。変更出来たらパスワードのsecretを削除
```
kubectl delete secret argocd-initial-admin-secret -n argocd
```

yamahaルーターがDNS問い合わせでTCPをしゃべってくれない問題でしばらくはまったが、8.8.8.8を追加して問題解決。
今回は以上。
