---
title: Raspberry Pi 4でおうちk8s構築（metalLB編）
description: raspberry Pi 4でおうちk8sを構築した記録その３です。
date: 2021-05-11
slug: raspberry-pi-4-k8s-3
tags:
  - Linux
  - k8s
  - network
  - raspberrypi
categories:
  - "raspberry-pi-4-k8s"
---
## k8sクラスタ
前回まででk8sクラスタを作ることができました。しかし、このままではクラスタ外からなにも見えません。これでは意味がありませんので、外部から見れるようにします。簡単な方法としてnodePortがありますが、いまいちスマートではありません。本当なら、クラウドサービスで利用できるようなロードバランサによるipのアタッチを行いたいところです。そこで、今回は、[MetalLB](https://metallb.universe.tf)というものをつかって、簡易的にローカルipの自動割り当てを行えるようにしたいと思います。
[シリーズリンク](/categories/raspberry-pi-4-k8s)

## 材料
- k8sクラスタ
- 利用するローカルip範囲
- metalLB

## 導入
公式のドキュメントをみながら導入します。

[ipvsを利用する](https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/ipvs/README.md#run-kube-proxy-in-ipvs-mode)ので、事前にカーネルモジュールがあるか確認します。そして、起動時にロードするように設定します。必要なのは次の五つです。これは、クラスタの**すべてのノード**に行います。
- ip_vs
- ip\_vs_rr
- ip\_vs_wrr
- ip\_vs_sh
- nf_conntrack

```
sudo lsmod | grep ip_vs
sudo lsmod | grep nf_conntrack
```
```
cat <<EOF | sudo tee /etc/modules-load.d/ipvs.conf
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
EOF
```

クラスタはkubeadmで作成したので、その設定ファイルを書き換えます。
```
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
sed -e "s/mode: \"\"/mode: ipvs/" | \
kubectl diff -f - -n kube-system
```
差分を確認します。modeがipvsになっている。strictARPが有効になっている。を確認します。

設定ファイルは以下で確認できるので見ながら環境に合わせます。
```
kubectl edit configmap -n kube-system kube-proxy
```

問題がなさそうであればapplyします。
```
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
sed -e "s/mode: \"\"/mode: ipvs/" | \
kubectl apply -f - -n kube-system
```

なんかwarnが出ました。The missing annotation will be patched automatically.とあるのできっと大丈夫でしょう。一応したところ、kubectl.kubernetes.io/last-applied-configurationが追加されていました。きっと問題ないのでしょう。
```
Warning: resource configmaps/kube-proxy is missing the kubectl.kubernetes.io/last-applied-configuration annotation which is required by kubectl apply. kubectl apply should only be used on resources created declaratively by either kubectl create --save-config or kubectl apply. The missing annotation will be patched automatically.
configmap/kube-proxy configured
```

MetalLBを入れます。0.9.6が最新でした。
```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.6/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.6/manifests/metallb.yaml
# On first install only
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```

動作確認。
```
kubectl get pods -n metallb-system
kubectl get daemonsets -n metallb-system
kubectl get deploy -n metallb-system
```
いいかんじにrunningでした。

起動しただけでは何の役にも立たないので、[configMapを入れます](https://metallb.universe.tf/configuration/)。BGPルータなどという高尚なものはないので、L2モードで動かします。今回、ip範囲は2.50から2.99にしました。
```
cat <<EOF | tee metallb-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.2.50-192.168.2.99
EOF
```

applyします。
```
kubectl apply -f metallb-config.yaml
```

動作確認をします。
```
cat <<EOF | tee nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-metallb
  name: nginx-metallb
spec:
  selector:
    matchLabels:
      app: nginx-metallb
  template:
    metadata:
      labels:
        app: nginx-metallb
    spec:
      containers:
      - image: nginx
        name: nginx-metallb
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-metallb
spec:
  type: LoadBalancer
  selector:
    app: nginx-metallb
  ports:
    - name: http
      port: 80
      targetPort: 80
EOF
```

```
kubectl apply -f nginx.yaml
```

get podsでnginxがrunningになったら、serviceを確認します。
```
kubectl get svc nginx-metallb
```

> NAME            TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)        AGE
>
> nginx-metallb   LoadBalancer   10.98.89.91   192.168.2.50   80:30536/TCP   55s

良さそうです。192.168.2.50が割り当てられました。

{{< figure src="assets/thumbnail/nginx-web.png" link="assets/nginx-web.png" title="nginx-web" height="340">}}

無事ブラウザからも見れました。今回はここまでです。ありがとうございました。
