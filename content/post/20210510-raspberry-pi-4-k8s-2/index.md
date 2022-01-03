---
title: Raspberry Pi 4でおうちk8s構築（ソフトウェア編）
description: raspberry Pi 4でおうちk8sを構築した記録その２です。
date: 2021-05-10
slug: raspberry-pi-4-k8s-2
tags:
  - Linux
  - k8s
  - network
  - raspberrypi
categories:
  - "raspberry-pi-4-k8s"
---
## おうちk8s
前回物理的な組み立てをしました。今回はソフトウェアの構築をします。
多くのページを参考にしましたが、[Raspberry Pi 4BにKubernetesをインストール（2021年版）](https://blog.uyorum.net/post/k8s-on-rpi4/)を主に参考にしました。
[シリーズリンク](/categories/raspberry-pi-4-k8s)

## OSインストール
今回は、ラズパイ用ubuntuを使うことにしました。使用例が多そうなので。
[ダウンロードページ](https://ubuntu.com/download/raspberry-pi)から、最新のUbuntu Server LTSを選びます。20.04.2 LTSでした。
64bitを使います。

raspberry pi imagerでダウンロードしたイメージを焼きます。
イメージが焼けたら、SDカードの中身を確認します。

cgroup設定をします。これをしないとk8sが使えない？みたいですがよくわかりません。以前のバージョンのubuntuではnobtcmd.txtに追記するようでしたが、今回20.04.2ではそのファイルがなく、代わりにcmdline.txtに追加するようになったようです。以下の内容を改行せずに末尾に追記します。

```
 cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1
```

ファイルの内容は以下のようになりました。

cmdline.txt
```
net.ifnames=0 dwc_otg.lpm_enable=0 console=serial0,115200 console=tty1 root=LABEL=writable rootfstype=ext4 elevator=deadline rootwait fixrtc cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1
```

これをラズパイの台数分繰り返し、出来たらSDカードをラズパイにセットします。

## トラベルルータの設定
トラベルルータを設定しないとラズパイがネットワークにつながりませんので、これをやります。説明書きの通りにやります。

このトラベルルータでは様々なモードが使えますが、今回の用途ではクライアントモードを使用します。
設定ができたら、とりあえず一番上のラズパイから順に起動してみましょう。

## 一号機起動
仮に、三段のラズパイを上から一号機、二号機のように呼ぶことにします。最初ipを固定するまでは分かりにくくなるので一台づつ起動して設定してみます。

ubuntu serverでは、デフォルトでsshできるようになっています。ルーターの接続中クライアント一覧からそれっぽいのを見つけてsshしましょう。

初回ログイン時にパスワード変更を求められるので変更しておきましょう。変更するとログアウトされますのでログインしなおします。

なにやらシステム再起動が必要らしいので再起動しておきましょう。ubuntuユーザーはpasswordなしでsudoできるんですね。

さて。パスワードでsshするのはよくないので鍵を設定します。また、新しくユーザーを作成し、公開鍵を設定するのもそのユーザーにします。いつも参考にしている過去のメモを参考にしながらやります。このメモはCentOS用なので、適宜読み替えます。例えば、ubuntuではuseraddではなくadduserを使わないとはまりやすいです。公開鍵ログインできることを確認してからパスワードログインをオフにしましょう。確認するときはsudo systemctl reload sshdをして設定を反映してからしましょう。sudoグループにも追加出来たらユーザー追加は終了です。
{{< gist pinfort 00eec11f1478c3a9ba42b6aff6d656a2 >}}

任意でホスト名を分かりやすいものに変更しておくのもいいですね。
```
hostnamectl set-hostname k8sraspino1
```

## ip固定

ip固定をします。

netplanを変更します。/etc/netplan/50-cloud-init.yamlというのが存在しましたが、ファイル内の指示に従って無効化します。

[https://linuxfan.info/ubuntu-2004-server-static-ip-address](https://linuxfan.info/ubuntu-2004-server-static-ip-address)を参考にymlを書きます。意図通りに適用出来たらいったんここまでにして、ほかの二台もここまで同じように構築します。

## cri-oインストール
今回は、コンテナランタイムとしてcri-oを使います。
[kubernetesのドキュメント](https://kubernetes.io/ja/docs/setup/production-environment/container-runtimes/)、[kubeadmに関するドキュメント](https://kubernetes.io/ja/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)や[cri-oの公式ドキュメント](https://cri-o.io)を参考にしながら実行します

たぶんところどころ間違っているので、一行ずつ確かめながら実行します。

```
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/crio.conf
overlay
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sudo sysctl --system

# レガシーバイナリがインストールされていることを確認してください
sudo apt-get install -y iptables arptables ebtables

# レガシーバージョンに切り替えてください。
sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
sudo update-alternatives --set arptables /usr/sbin/arptables-legacy
sudo update-alternatives --set ebtables /usr/sbin/ebtables-legacy

sudo iptables -S

cat <<EOF | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_20.04/ /
EOF
cat <<EOF | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:1.21.list
deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/1.21/xUbuntu_20.04/ /
EOF

curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:1.21/xUbuntu_20.04/Release.key | sudo apt-key add -
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_20.04/Release.key | sudo apt-key add -
sudo apt update
sudo apt install cri-o cri-o-runc -y
```

## kubeletなどインストール
```
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

sudo systemctl daemon-reload
sudo systemctl enable crio
sudo systemctl restart crio

sudo swapoff -a
sudo systemctl restart kubelet
kubelet --version
# Kubernetes v1.21.1
kubeadm version
# kubeadm version: &version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.1", GitCommit:"5e58841cce77d4bc13713ad2b91fa0d961e69192", GitTreeState:"clean", BuildDate:"2021-05-12T14:17:27Z", GoVersion:"go1.16.4", Compiler:"gc", Platform:"linux/arm64"}
kubectl version
# Client Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.1", GitCommit:"5e58841cce77d4bc13713ad2b91fa0d961e69192", GitTreeState:"clean", BuildDate:"2021-05-12T14:18:45Z", GoVersion:"go1.16.4", Compiler:"gc", Platform:"linux/arm64"}
# The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

kubectlでサーバー接続エラーが出ているのはkubeadm initしてないからだろう。よさそう。

```
echo 'source <(kubectl completion bash)' >>~/.bashrc
source ~/.bashrc

cat <<EOF | sudo tee /etc/default/kubelet
KUBELET_EXTRA_ARGS=--cgroup-driver=systemd --container-runtime=remote --container-runtime-endpoint="unix:///var/run/crio/crio.sock" --resolv-conf=/run/systemd/resolve/resolv.conf
EOF
```
**はまりどころ**。今回利用するubuntuでは、systemd-resolvedという仕組みが動いていて、/etc/resolv.confに生のnameserverのipがない。かわりに/run/systemd/resolve/resolv/confに生ipが書いてあるので、k8sにはこっちのファイルを渡す必要がある。

## マスタノード設定
ここからマスタノードとなる一号機で実行します。
```
# 動作を確認します
sudo kubeadm config images pull
# 正常に実行できそうなので次に進みます

sudo systemctl daemon-reload
sudo systemctl restart kubelet
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
initがうまくいかない場合、sudo kubeadm resetでリセットしてやり直す。

kubectl設定
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

```
kubectl get pods -A
# NAMESPACE     NAME                                  READY   STATUS    RESTARTS   AGE
# kube-system   coredns-558bd4d5db-99w2j              1/1     Running   0          61s
# kube-system   coredns-558bd4d5db-m7zs8              1/1     Running   0          61s
# kube-system   etcd-k8sraspino1                      1/1     Running   0          59s
# kube-system   kube-apiserver-k8sraspino1            1/1     Running   0          59s
# kube-system   kube-controller-manager-k8sraspino1   1/1     Running   0          59s
# kube-system   kube-proxy-t62cz                      1/1     Running   0          62s
# kube-system   kube-scheduler-k8sraspino1            1/1     Running   0          59s
```
よさそう。

変なイベントが出ていないか一通り確認する。
```
kubectl describe all -A
```
よさそう。

## マスタノードにflannelをいれる
```
# sudo不要
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/v0.13.0/Documentation/kube-flannel.yml
kubectl describe all -A
```
```
wget https://raw.githubusercontent.com/flannel-io/flannel/v0.13.0/Documentation/kube-flannel.yml
sed -i -e 's/vxlan/host-gw/g' kube-flannel.yml
kubectl apply -f kube-flannel.yml
```
特にエラーは出ていない。

**はまりどころ**なぜかよくわからないが、vxlanモードはうまく動かなかった。オンプレだし、L2で各ノードが直接つながっているので、home-gwモードで動かすことで解決した。

## クラスタにワーカを参加させる
いい感じにできたっぽいのでworkerの二号機と三号機で以下を実行
```
sudo kubeadm join ip:6443 --token hoge \
        --discovery-token-ca-cert-hash sha256:foo
```

{{< figure src="assets/thumbnail/k8snodes.png" link="assets/k8snodes.png" title="k8snodes" height="340">}}

良さそうです。

workerにラベルを付けておきます。
```
kubectl label node k8sraspino2 node-role.kubernetes.io/worker=worker
kubectl label node k8sraspino3 node-role.kubernetes.io/worker=worker
```

## DNSの動作をかくにん
```
kubectl apply -f https://k8s.io/examples/admin/dns/dnsutils.yaml
kubectl get pods dnsutils
# dnsutils   1/1     Running   0          15s

kubectl exec -i -t dnsutils -- nslookup kubernetes.default
# ;; connection timed out; no servers could be reached
# 
# command terminated with exit code 1
```
なんかだめ

```
kubectl describe services -n kube-system kube-dns
# IP:                10.96.0.10
# IPs:               10.96.0.10
# Port:              dns  53/UDP
# TargetPort:        53/UDP
# Endpoints:         10.85.0.2:53,10.85.0.3:53
# Port:              dns-tcp  53/TCP
# TargetPort:        53/TCP
# Endpoints:         10.85.0.2:53,10.85.0.3:53
kubectl exec -ti dnsutils -- cat /etc/resolv.conf
# search default.svc.cluster.local svc.cluster.local cluster.local
# nameserver 10.96.0.10
# options ndots:5
```
よさそうだけどね。

```
nslookup kubernetes.default 10.85.0.2
# Server:         10.85.0.2
# Address:        10.85.0.2#53

# ** server can't find kubernetes.default: NXDOMAIN
nslookup kubernetes.default 10.96.0.10
# Server:         10.96.0.10
# Address:        10.96.0.10#53

# ** server can't find kubernetes.default: NXDOMAIN
```

なんかだめ

ワーカーノードにルートがなかったので手作業で追加
```
sudo ip route add 10.85.0.0/16 via $CONTROL_PLANE_IP dev eth0
sudo ip route add 10.96.0.0/16 via $CONTROL_PLANE_IP dev eth0
```
**はまりどころ**ワーカーノードからどうもマスターノードのpodが見えない。ip routeを見ると、ルートがなかったので、コマンドで追加した。

10.85.0.0/16は、cni0インターフェースのip。podにここからipがわりあてられた。
10.96.0.0/16は、kubeAdmのconfigmapに設定がある、serviceSubnet。clusterIPはここからわりあてられる。

{{< figure src="assets/thumbnail/dnstest.png" link="assets/dnstest.png" title="dnstest" height="340">}}

よさそう。

## 動作確認
試しにnginxをデプロイしてみます。
```
kubectl create -f https://k8s.io/examples/application/deployment.yaml
```

良さそうです。

{{< figure src="assets/thumbnail/nginx.png" link="assets/nginx.png" title="nginx" height="340">}}

想定通りnginxが二つデプロイされています。

{{< figure src="assets/thumbnail/nginx-curl-1.png" link="assets/nginx-curl-1.png" title="nginx-curl-1" height="340">}}
{{< figure src="assets/thumbnail/nginx-curl-2.png" link="assets/nginx-curl-2.png" title="nginx-curl-2" height="340">}}

とりあえず基本的な構成はできたようです。ネットワーク内の他のPCからデプロイしたnginxが見れるようにしなければなりませんが、いったんこの記事はここまでにしたいと思います。
