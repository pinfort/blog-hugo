---
title: おうちk3sにGitHubのARC(Actions Runner Controller)をデプロイする
description: k3sにargocdを使用してARCをデプロイします。
date: 2023-12-28
slug: k3s-arc-deploy
---
## デプロイ手順
[公式の手順](https://docs.github.com/ja/actions/hosting-your-own-runners/managing-self-hosted-runners-with-actions-runner-controller/quickstart-for-actions-runner-controller)を参考にデプロイします。
argoCDを構築済みなので今回はそれを利用します。

### ネームスペース作成
ARCには、controller用とrunner用の二つのnamespaceが必要です。あらかじめ用意します。
```
kubectl create ns gh-actions-system
kubectl create ns gh-actions-runner
```

### argoCD project作成
argoCD上で、projectを作成します。

```yml
General.name: github-actions-runner
Destinations:
  - Server: https://kubernetes.default.svc
    Name: in-cluster
    Namespace: gh-actions-runner
  - Server: https://kubernetes.default.svc
    Name: in-cluster
    Namespace: gh-actions-system
CLUSTER RESOURCE ALLOW LIST:
  Kind: *
  Group: *
NAMESPACE RESOURCE ALLOW LIST:
  Kind: *
  Group: *
```

### argoCD repository connect
argoCDにarcのレポジトリを登録します。EnableOCIをtrueにしないと登録できないようです。

```yml
Choose your connection method: Via HTTPS
CONNECT REPO USING HTTPS:
  Type: Helm
  Name: github-actions-runner
  Project: github-actions-runner
  Repository URL: oci://ghcr.io/actions/actions-runner-controller-charts
  Enable OCI: true
```

### create controller application
controllerのアプリケーションを作成します。バージョンは現時点で最新の0.8.1を使用します。

```yml
General:
  Application Name: arc
  Project Name: github-actions-runner
  SYNC POLICY: Manual
Source:
  Repository URL: ghcr.io/actions/actions-runner-controller-charts
  Chart: gha-runner-scale-set-controller
  Version: 0.8.1
Destination:
  Cluster URL: https://kubernetes.default.svc
  Namespace: gh-actions-system
```

上記の設定でアプリケーション作成後、手動でsyncします。すると、`autoscalingRunnersets.actions.github.com`だけが作られない。ApplicationのSync Statusを見てみると、`metadata.annotations: Too long: must have at most 262144 bytes`というようなエラーが出ている。argoCDは内部でkubectl applyをするのだが、このCRDは長すぎてapplyでは作成できない。どうすればいいかというと、applyの代わりにcreateをすればいい。argoCDでこれを実施するには、手動syncで、REPLACEをチェックしてからsyncすればいい。

{{< figure src="assets/argocd.png" link="assets/argocd.png" title="argocd" height="340">}}

### create runner application
runnerのアプリケーションを作成します。バージョンはcontrollerと同じ0.8.1を使用します。Application NameはGithubからこのランナーを指定するときに使用する名前になるので、それに気を付けながら決めます。今回は`arc-local-k3s`とします。

次に、GitHubでPersonal Access Token(classic)を作成します。取得出来たら、以下のように設定します。

```yml
General:
  Application Name: arc-local-k3s
  Project Name: github-actions-runner
  SYNC POLICY: Manual
Source:
  Repository URL: ghcr.io/actions/actions-runner-controller-charts
  Chart: gha-runner-scale-set
  Version: 0.8.1
Destination:
  Cluster URL: https://kubernetes.default.svc
  Namespace: gh-actions-runner
Helm:
  Values: |-
    controllerServiceAccount:
      namespace: gh-actions-system
      name: arc-gha-rs-controller
  Parameters:
    githubConfigSecret.github_token: <YOUR PAT>
    githubConfigUrl: <YOUR organization OR repo>
```

## 確認
### runner登録確認
runnerは、正常に起動していればcontrollerによって自動的にorgやrepoに登録されます。

{{< figure src="assets/runner.png" link="assets/runner.png" title="runner" height="340">}}

もし登録されていなければ、controllerのpodのlogを見るとエラー内容がわかるかもしれません。

### runner動作確認
適当なレポジトリでciの実行設定をし、適切に動作しているか検証します。このとき、自分で決めた名前をruns-onに指定します。podが自動的に立ち上がり、指定内容が実施されれば問題ありません。

```yml
name: CI

on: [push]

jobs:
  build:
    runs-on: arc-local-k3s
    steps:
      - uses: actions/checkout@v4
      - name: Use Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20.x'
```

## 最終的に
argoCDのアプリケーション画面は以下のようになりました。

{{< figure src="assets/argocd-applications.png" link="assets/argocd-applications.png" title="argocd-applications" height="340">}}

## はまった個所
1. controllerのpodでgithubの認証エラーが出続けていて、runnerが登録されませんでした。原因は、tokenのコピペミスで適切に入力できていなかったことでした。
1. `autoscalingRunnersets.actions.github.com`の作成失敗。先に書いた通り、これだけcreateしないといけない。
