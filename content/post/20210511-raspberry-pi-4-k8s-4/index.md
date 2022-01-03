---
title: Raspberry Pi 4でおうちk8s構築（argoCD編）
description: raspberry Pi 4でおうちk8sを構築した記録その４です。
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
## k8s
[シリーズリンク](/categories/raspberry-pi-4-k8s)

k8sでアプリケーションを管理するにあたって、やっぱりargoCD入れたいよねということで、入れます。

```
kubectl create namespace argocd
```
実はargoCd、imageがamd64にしか対応していないので、非公式のarm64対応版を入れます。

```
wget https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
cat install.yaml | sed -e "s/quay.io\/argoproj\/argocd/alinbalutoiu\/argocd/g" > argocd.yaml
kubectl apply -n argocd -f argocd.yaml
```

```
kubectl get pods --namespace=argocd
```

> NAME                                  READY   STATUS              RESTARTS   AGE
>
> argocd-application-controller-0       0/1     ContainerCreating   0          40s
>
> argocd-dex-server-ccd6b88c7-dc7lm     0/1     Init:0/1            0          41s
>
> argocd-redis-9567956cd-4v6c6          0/1     ContainerCreating   0          41s
>
> argocd-repo-server-7b49b5cd77-bnvhr   0/1     ContainerCreating   0          41s
>
> argocd-server-6946b99d-hkk4h          0/1     ContainerCreating   0          41s

作成に時間がかかってそうなので待ちます。

> NAME                                  READY   STATUS    RESTARTS   AGE
>
> argocd-application-controller-0       0/1     Running   0          2m9s
>
> argocd-dex-server-ccd6b88c7-dc7lm     1/1     Running   0          2m10s
>
> argocd-redis-9567956cd-4v6c6          1/1     Running   0          2m10s
>
> argocd-repo-server-7b49b5cd77-bnvhr   1/1     Running   0          2m10s
>
> argocd-server-6946b99d-hkk4h          1/1     Running   0          2m10s


argocd-serverのDeployment、containers.commandsにオプション `--insecure` を追加することでhttpsを無効化できますが、それをするとうまくログインできなかったためにhttpsのまま使います。ブラウザでは証明書エラーが出ますが、無視します。

[前回の記事](/posts/raspberry-pi-4-k8s-3)でMetallbを導入し、LoadBalancerが使えるようになっているので、それを使って外部に公開します。
```
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

argoCDがブラウザで見れるようになりました。

いろいろいじっていたらAdminの初期パスワードが分からなくなったので、[リセットしてログイン](https://argoproj.github.io/argo-cd/faq/#i-forgot-the-admin-password-how-do-i-reset-it)します。

password: argocd-initial-admin-secret
```
kubectl -n argocd patch secret argocd-secret \
   -p '{"stringData": {
     "admin.password": "$2a$10$5dgaWDvOFEbV4fIz6pXOP.bj96K3XS4uAg9lILnW7.CCdnJPpl7Uq",
     "admin.passwordMtime": "'$(date +%FT%T%Z)'"
   }}'
```

いい感じです。
{{< figure src="assets/thumbnail/argocd.png" link="assets/argocd.png" title="argocd" height="340">}}

arm64用のバイナリがないので、windowsにcliをインストールします。[argocdのdocumentationを確認します。](https://argoproj.github.io/argo-cd/cli_installation/)

adminのパスワードを更新します。
```
argocd login 192.168.2.50 --insecure
argocd account update-password
```
初期パスワードのk8s secretを削除します。

```
kubectl delete secrets -n argocd argocd-initial-admin-secret
```

configMapを書いてローカルユーザーを作ります。

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
  # add an additional local user with apiKey and login capabilities
  #   apiKey - allows generating API keys
  #   login - allows to login using UI
  accounts.pinfort: apiKey, login
  # disables user. User is enabled by default
  accounts.pinfort.enabled: "false"
```

パスワードを設定します。初期値はadminのpasswordです。
```
argocd account update-password --account devuser --current-password 'myadminpassword' --new-password  'mysecurepass2'
```

新しく作ったユーザーに権限を設定します。
[argocd documentation](https://argoproj.github.io/argo-cd/operator-manual/rbac/#tying-it-all-together)の例に、さらにいろいろ権限を追加したファイルで適用してみます。

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly
  policy.csv: |
    p, role:org-admin, applications, *, */*, allow
    p, role:org-admin, clusters, get, *, allow
    p, role:org-admin, repositories, get, *, allow
    p, role:org-admin, repositories, create, *, allow
    p, role:org-admin, repositories, update, *, allow
    p, role:org-admin, repositories, delete, *, allow
    p, role:org-admin, projects, *, *, allow

    g, pinfort, role:org-admin
```
```
kubectl apply -f argocd-rbac.yaml
```

今回はここまでです。k8sの初期構築はここまでとしたいと思います。ありがとうございました。
