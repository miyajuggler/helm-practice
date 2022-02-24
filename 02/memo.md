## インストールする

brew が入っていればこれで一発

```sh
$ brew install helm
# もしくは
$ brew install kubernetes-helm
```

これだと最新版を取ってきてしまうので、学習用としてはひとまずやめておく。

確認

```
$ helm version
version.BuildInfo{Version:"v3.8.0", GitCommit:"d14138609b01886f544b2025f5000351c9eb092e", GitTreeState:"clean", GoVersion:"go1.17.6"}
$ helm -h
The Kubernetes package manager
    :
    :
```

https://helm.sh/ja/

## helm v2 インストール

普通にやるとこうなった

[これ参照](https://qiita.com/ffrr55s/items/df6cb2c418bcfba66f59#:~:text=%E7%90%86%E8%A7%A3%E3%81%8F%E3%81%A0%E3%81%95%E3%81%84%E3%80%82-,1.Helm%E3%81%AE%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%AB,-%24%20cd%20~/downloads%0A%24%20curl)

```
$ curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh

$ chmod +x get_helm.sh

$ ./get_helm.sh
No prebuilt binary for darwin-arm64.
To build from source, go to https://github.com/helm/helm
Failed to install helm
        For support, go to https://github.com/helm/helm.
```

初回からつまずいてしまったが、いろいろ調べて紆余曲折あり、以下の方法でインストール完了できる。

### 1. ロゼッタターミナルを作る

intel 版のやつ  
作り方は[こちら](https://qiita.com/funatsufumiya/items/cec08f1ba3387edc2eed)

確認

```sh
# intel 版はこれ
$ uname -m
x86_64

# ちなみに ARM 版 (M1チップ) はこれ
$ uname -m
arm64
```

### 2. コマンド実行

```sh
$ curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh
$ chmod +x get_helm.sh
$ ./get_helm.sh
```

```sh
$ ./get_helm.sh
Helm v2.17.0 is available. Changing from version .
Downloading https://get.helm.sh/helm-v2.17.0-darwin-amd64.tar.gz
Preparing to install helm and tiller into /usr/local/bin
Password:
helm installed into /usr/local/bin/helm
tiller installed into /usr/local/bin/tiller
Run 'helm init' to configure helm.
```

```sh
$ helm version
Client: &version.Version{SemVer:"v2.17.0", GitCommit:"a690bad98af45b015bd3da1a41f6218b1a451dbe", GitTreeState:"clean"}
Error: could not find tiller

```

### 3. Tiller をインストール

helm init により Kubernetes クラスタ上に tiller をデプロイする。

以下コマンドを打ってデプロイしようとするも失敗。

```sh
$ helm init --docker-desktop tiller
Error: unknown flag: --docker-desktop
```

よくわからんが `--docker-desktop tiller` を消してみたらなんか行けた  
[参考](https://qiita.com/loftkun/items/853bbaabd4bf0fa96e9c#:~:text=scripts/get%20%7C%20bash-,tiller%E3%82%92%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%AB%E3%81%99%E3%82%8B,-helm%20init%E3%82%B3%E3%83%9E%E3%83%B3%E3%83%89)

```sh
$ helm init
Creating /Users/miyazakinaohiro/.helm
Creating /Users/miyazakinaohiro/.helm/repository
Creating /Users/miyazakinaohiro/.helm/repository/cache
Creating /Users/miyazakinaohiro/.helm/repository/local
Creating /Users/miyazakinaohiro/.helm/plugins
Creating /Users/miyazakinaohiro/.helm/starters
Creating /Users/miyazakinaohiro/.helm/cache/archive
Creating /Users/miyazakinaohiro/.helm/repository/repositories.yaml
Adding stable repo with URL: https://charts.helm.sh/stable
Adding local repo with URL: http://127.0.0.1:8879/charts
$HELM_HOME has been configured at /Users/miyazakinaohiro/.helm.

Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.

Please note: by default, Tiller is deployed with an insecure 'allow unauthenticated users' policy.
To prevent this, run `helm init` with the --tiller-tls-verify flag.
For more information on securing your installation see: https://v2.helm.sh/docs/securing_installation/
```

確認

```sh
$ helm version
Client: &version.Version{SemVer:"v2.17.0", GitCommit:"a690bad98af45b015bd3da1a41f6218b1a451dbe", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.17.0", GitCommit:"a690bad98af45b015bd3da1a41f6218b1a451dbe", GitTreeState:"clean"}
```

tiller が k8s クラスタの -n kube-system 配下にデプロイされているらしい

確認

```
$ kubectl get po,deploy,svc  -n kube-system -l name=tiller
NAME                                 READY   STATUS    RESTARTS   AGE
pod/tiller-deploy-7b9cbd46c9-q4hl6   1/1     Running   0          23m

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/tiller-deploy   1/1     1            1           23m

NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
service/tiller-deploy   ClusterIP   10.100.25.128   <none>        44134/TCP   23m
```

## 参考

[【ハンズオン】Docker+Kubernetes で Helm を使ってみよう](https://qiita.com/ffrr55s/items/df6cb2c418bcfba66f59#:~:text=%E7%90%86%E8%A7%A3%E3%81%8F%E3%81%A0%E3%81%95%E3%81%84%E3%80%82-,1.Helm%E3%81%AE%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%AB,-%24%20cd%20~/downloads%0A%24%20curl)

[Helm v2 のすゝめ](https://qiita.com/loftkun/items/853bbaabd4bf0fa96e9c)
