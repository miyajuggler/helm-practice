# Helm コマンドを試す

## helm repo

chart リポジトリの追加、削除、更新、一覧を表示するコマンド。

```sh
# chart リポジトリーの一覧表示
$ helm repo list
NAME    URL
stable  https://charts.helm.sh/stable
local   http://127.0.0.1:8879/charts
```

stable という名前のリポジトリが公式リポジトリ。  
local がローカルリポジトリ。  
※local リポジトリを使用するには helm serve で明示的に HTTP サーバーを起動させる必要あり。

Incubator リポジトリを利用するには以下を対処。

```sh
# incubator リポジトリ追加
$ helm repo add incubator https://charts.helm.sh/incubator
"incubator" has been added to your repositories

# 確認
$ helm repo list
NAME            URL
stable          https://charts.helm.sh/stable
local           http://127.0.0.1:8879/charts
incubator       https://charts.helm.sh/incubator
```

※ここで参考書にあった `https://kubernetes-charts-incubator.storage.googleapis.com` やサイトにあった `http://storage.googleapis.com/kubernetes-charts-incubator` はリポジトリが廃止されてて、もう使えないらしい。

以下参考

- [Helm の official chart repository が クローズとなった](https://note.com/loftkun/n/n6a62995fe2ee)
- [Helm のサブコマンドを全部使ってみた](https://dev.classmethod.jp/articles/helm-sub-commands/#:~:text=%E5%A4%B1%E6%95%97%E3%81%97%E3%81%BE%E3%81%97%E3%81%9F%E3%81%AD%E2%80%A6%E8%AA%BF%E3%81%B9%E3%81%9F%E3%81%A8%E3%81%93%E3%82%8D%E3%81%AB%E3%82%88%E3%82%8B%E3%81%A8%E3%80%81mariadb%E3%83%81%E3%83%A3%E3%83%BC%E3%83%88%E9%85%8D%E5%B8%83%E5%85%83%E3%81%AEhttps%3A//kubernetes%2Dcharts.storage.googleapis.com/%E3%83%AA%E3%83%9D%E3%82%B8%E3%83%88%E3%83%AA%E3%81%8C%E5%BB%83%E6%AD%A2%E3%81%95%E3%82%8C%E3%80%81%E4%BB%A3%E3%82%8F%E3%82%8A%E3%81%ABhttps%3A//charts.helm.sh/stable%E3%82%92%E4%BD%BF%E3%81%86%E5%BF%85%E8%A6%81%E3%81%8C%E3%81%82%E3%82%8B%E3%82%88%E3%81%86%E3%81%A7%E3%81%99%E3%80%82%E3%81%A8%E3%81%84%E3%81%86%E3%82%8F%E3%81%91%E3%81%A7%E5%85%88%E7%A8%8B%E3%81%AEYAML%E3%82%92%E6%9B%B4%E6%96%B0%E3%81%97%E3%81%BE%E3%81%99%E3%80%82)
- https://github.com/helm/charts

## helm search

指定したキーワードで chart を検索するコマンド。

```sh
# mysql キーワードで関連する chart を検索
$ helm search mysql
NAME                                    CHART VERSION   APP VERSION     DESCRIPTION
incubator/mysqlha                       2.0.2           5.7.13          DEPRECATED MySQL cluster with a single master and zero or...
stable/mysql                            1.6.9           5.7.30          DEPRECATED - Fast, reliable, scalable, and easy to use op...
stable/mysqldump                        2.6.2           2.4.1           DEPRECATED! - A Helm chart to help backup MySQL databases...
stable/prometheus-mysql-exporter        0.7.1           v0.11.0         DEPRECATED A Helm chart for prometheus mysql exporter wit...
stable/percona                          1.2.3           5.7.26          DEPRECATED - free, fully compatible, enhanced, open sourc...
stable/percona-xtradb-cluster           1.0.8           5.7.19          DEPRECATED - free, fully compatible, enhanced, open sourc...
stable/phpmyadmin                       4.3.5           5.0.1           DEPRECATED phpMyAdmin is an mysql administration frontend
stable/gcloud-sqlproxy                  0.6.1           1.11            DEPRECATED Google Cloud SQL Proxy
stable/mariadb                          7.3.14          10.3.22         DEPRECATED Fast, reliable, scalable, and easy to use open...
```

すでに chart が公開されていないか確認するときに使う。
またコマンド上じゃなくても、HelmHub で UI 上で検索することも可能

## helm fetch

chart をリポジトリからダウンロードするときに使う。

```sh
# mysql chart (アーカイブファイル) をローカルにダウンロード
$ helm fetch stable/mysql

# mysql のアーカイブファイルがダウンロードされている。
$ ls | grep mysql
mysql-1.6.9.tgz
```

後に説明する `helm template` と組み合わせて、外部公開されている chart の yaml を簡単に取得することができる。

## helm create

指定した名前の chart テンプレートを生成するコマンド。

```sh
# sample という名前の chart 作成
$ helm create sample
Creating sample

# sample ディレクトリの中身確認
$ tree sample
sample
├── Chart.yaml
├── charts
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── ingress.yaml
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml

3 directories, 9 files
```

sample ディレクトリそのものが chart に相当

## helm package

chart をアーカイブするコマンド (「<chart 名>-<バージョン>.tgz」の形)

```sh
#
$ helm package sample
Successfully packaged chart and saved it to: /Users/miyazakinaohiro/github/helm-practice/03/sample-0.1.0.tgz
```

chart リポジトリを稼働させるにはアーカイブファイルを HTTP サーバーに配置する必要がある。
そのため chart を自作して公開する場合には必須のコマンド。

## helm lint

chart を構文チェックするコマンド。推奨構成についても指摘してくれる。

```sh
# sample chart を helm lint してみる
# helm lint sample-0.1.0.tgz でも良い
$ helm lint sample
==> Linting sample
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, no failures
```

lint の結果に問題はなく、chart 似アイコンを付けることが推奨されていることがわかる。

## helm template

template 配下の yaml ファイルを表示するコマンド。  
template 配下の yaml ファイルは変数定義されている。`helm template` コマンドを実行することで、`values.yaml` に記載された値を templates 配下の yaml に代入し、yaml ファイル形式の標準出力が得られる。

```sh
# helm template sample-0.1.0.tgz でも良い
$ helm template sample
```

<details><summary>結果</summary>

```sh
helm template sample
---
# Source: sample/templates/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: release-name-sample
  labels:
    app.kubernetes.io/name: sample
    helm.sh/chart: sample-0.1.0
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/version: "1.0"
    app.kubernetes.io/managed-by: Tiller
---
# Source: sample/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: release-name-sample
  labels:
    app.kubernetes.io/name: sample
    helm.sh/chart: sample-0.1.0
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/version: "1.0"
    app.kubernetes.io/managed-by: Tiller
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app.kubernetes.io/name: sample
    app.kubernetes.io/instance: release-name

---
# Source: sample/templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "release-name-sample-test-connection"
  labels:
    app.kubernetes.io/name: sample
    helm.sh/chart: sample-0.1.0
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/version: "1.0"
    app.kubernetes.io/managed-by: Tiller
  annotations:
    "helm.sh/hook": test-success
spec:
  containers:
    - name: wget
      image: busybox
      command: ['wget']
      args:  ['release-name-sample:80']
  restartPolicy: Never

---
# Source: sample/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: release-name-sample
  labels:
    app.kubernetes.io/name: sample
    helm.sh/chart: sample-0.1.0
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/version: "1.0"
    app.kubernetes.io/managed-by: Tiller
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: sample
      app.kubernetes.io/instance: release-name
  template:
    metadata:
      labels:
        app.kubernetes.io/name: sample
        app.kubernetes.io/instance: release-name
    spec:
      serviceAccountName: release-name-sample
      securityContext:
        {}

      containers:
        - name: sample
          securityContext:
            {}

          image: "nginx:stable"
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /
              port: http
          readinessProbe:
            httpGet:
              path: /
              port: http
          resources:
            {}

---
# Source: sample/templates/ingress.yaml
```

</details>

`valluel.yaml` の値を変更することで出力される yaml の値も変わる。
`helm fetch` と組み合わせることで、外部公開されている chart の yaml を簡単に取得可能。

```sh
# chart のダウンロード
$ helm fetch stable/mysql

# 確認
$ ls | grep mysql
mysql-1.6.9.tgz

# yaml 出力
$ helm template mysql-1.6.9.tgz
```

## helm install

chart をクラスターにインストールするコマンド。
以下コマンドのように sample chart を kubernetes クラスターにインストールする。

```sh
$ helm install sample
NAME:   soft-meerkat
LAST DEPLOYED: Sat Feb 26 12:22:07 2022
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Deployment
NAME                 READY  UP-TO-DATE  AVAILABLE  AGE
soft-meerkat-sample  0/1    1           0          0s

==> v1/Pod(related)
NAME                                  READY  STATUS             RESTARTS  AGE
soft-meerkat-sample-5b7856978c-9zk4f  0/1    ContainerCreating  0         0s

==> v1/Service
NAME                 TYPE       CLUSTER-IP      EXTERNAL-IP  PORT(S)  AGE
soft-meerkat-sample  ClusterIP  10.102.239.202  <none>       80/TCP   0s

==> v1/ServiceAccount
NAME                 SECRETS  AGE
soft-meerkat-sample  1        0s


NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=sample,app.kubernetes.io/instance=soft-meerkat" -o jsonpath="{.items[0].metadata.name}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl port-forward $POD_NAME 8080:80
```

確認

```sh
$ kubectl get po,deploy,svc,sa -l app.kubernetes.io/instance=soft-meerkat
NAME                                       READY   STATUS    RESTARTS   AGE
pod/soft-meerkat-sample-5b7856978c-9zk4f   1/1     Running   0          4m55s

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/soft-meerkat-sample   1/1     1            1           4m55s

NAME                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/soft-meerkat-sample   ClusterIP   10.102.239.202   <none>        80/TCP    4m55s

NAME                                 SECRETS   AGE
serviceaccount/soft-meerkat-sample   1         4m55s
```

コマンドを実行すると release のステータスやインストールされた kubernetes のリソース、NOTES が表示。  
NOTES にはソフトウェアにアクセスするための方法やパスワード情報の取得方法などが記載。

NAME は --name または -n で指定しない場合はランダムな名前が付与される (今回は soft-meerkat)  
--namespace を指定することで、kubernetes クラスター上のどの名前空間にインストールするかを選択できる。省略すると現在のコンテキストの名前空間 (普通は default) になる。

指定した名前で、指定した名前空間に sample chart をインストールしてみる。

```sh
$ kubectl create ns application
namespace/application created
```

</details>

<details><summary>結果</summary>

```sh
$ helm install -n sample --namespace application sample/                                                       ○ docker-desktop
NAME:   sample
LAST DEPLOYED: Sat Feb 26 12:41:53 2022
NAMESPACE: application
STATUS: DEPLOYED

RESOURCES:
==> v1/Deployment
NAME READY UP-TO-DATE AVAILABLE AGE
sample 0/1 1 0 0s

==> v1/Pod(related)
NAME READY STATUS RESTARTS AGE
sample-85469f95-h49qq 0/1 ContainerCreating 0 0s

==> v1/Service
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
sample ClusterIP 10.104.255.44 <none> 80/TCP 0s

==> v1/ServiceAccount
NAME SECRETS AGE
sample 1 0s

NOTES:

1. Get the application URL by running these commands:
   export POD_NAME=$(kubectl get pods --namespace application -l "app.kubernetes.io/name=sample,app.kubernetes.io/instance=sample" -o jsonpath="{.items[0].metadata.name}")
   echo "Visit http://127.0.0.1:8080 to use your application"
   kubectl port-forward $POD_NAME 8080:80

# 確認
$ kubectl get po,deploy,svc,sa -l app.kubernetes.io/instance=sample -n application
NAME                        READY   STATUS    RESTARTS   AGE
pod/sample-85469f95-h49qq   1/1     Running   0          7m27s

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/sample   1/1     1            1           7m27s

NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/sample   ClusterIP   10.104.255.44   <none>        80/TCP    7m27s

NAME                    SECRETS   AGE
serviceaccount/sample   1         7m27s
```

</details>

<details><summary>結果</summary>

```sh
$ helm install -n test --namespace application sample/                                                         ○ docker-desktop
NAME:   test
LAST DEPLOYED: Sat Feb 26 12:40:41 2022
NAMESPACE: application
STATUS: DEPLOYED

RESOURCES:
==> v1/Deployment
NAME READY UP-TO-DATE AVAILABLE AGE
test-sample 0/1 1 0 0s

==> v1/Pod(related)
NAME READY STATUS RESTARTS AGE
test-sample-6d7d6bf6b9-jdrxt 0/1 ContainerCreating 0 0s

==> v1/Service
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
test-sample ClusterIP 10.109.90.151 <none> 80/TCP 0s

==> v1/ServiceAccount
NAME SECRETS AGE
test-sample 1 0s

NOTES:

1. Get the application URL by running these commands:
   export POD_NAME=$(kubectl get pods --namespace application -l "app.kubernetes.io/name=sample,app.kubernetes.io/instance=test" -o jsonpath="{.items[0].metadata.name}")
   echo "Visit http://127.0.0.1:8080 to use your application"
   kubectl port-forward $POD_NAME 8080:80

# 確認
$ kubectl get po,deploy,svc,sa -l app.kubernetes.io/instance=test -n application
NAME                               READY   STATUS    RESTARTS   AGE
pod/test-sample-6d7d6bf6b9-jdrxt   1/1     Running   0          8m45s

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/test-sample   1/1     1            1           8m45s

NAME                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/test-sample   ClusterIP   10.109.90.151   <none>        80/TCP    8m45s

NAME                         SECRETS   AGE
serviceaccount/test-sample   1         8m45s
```

</details>

ちなみに helm 上にある release と同じ名前で release を作ろうとするとエラーになる。

## helm list

release のリストを表示するコマンド。`helm install` でインストールした release が存在する。

```sh
# helm ls でも可
$ helm list
NAME            REVISION        UPDATED                         STATUS          CHART           APP VERSION     NAMESPACE
sample          1               Sat Feb 26 12:41:53 2022        DEPLOYED        sample-0.1.0    1.0             application
soft-meerkat    1               Sat Feb 26 12:22:07 2022        DEPLOYED        sample-0.1.0    1.0             default
test            1               Sat Feb 26 12:40:41 2022        DEPLOYED        sample-0.1.0    1.0             application
```

STATUS が DEPLOYED の release の一覧が見える。  
STATUS が DELETED になっている release を見たい場合は `-a` をつけると良い

## helm status

指定した release のステータスを確認するコマンド。

```
$ helm status sample
LAST DEPLOYED: Sat Feb 26 12:41:53 2022
NAMESPACE: application
STATUS: DEPLOYED

RESOURCES:
==> v1/Deployment
NAME    READY  UP-TO-DATE  AVAILABLE  AGE
sample  1/1    1           1          12m

==> v1/Pod(related)
NAME                   READY  STATUS   RESTARTS  AGE
sample-85469f95-h49qq  1/1    Running  0         12m

==> v1/Service
NAME    TYPE       CLUSTER-IP     EXTERNAL-IP  PORT(S)  AGE
sample  ClusterIP  10.104.255.44  <none>       80/TCP   12m

==> v1/ServiceAccount
NAME    SECRETS  AGE
sample  1        12m


NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace application -l "app.kubernetes.io/name=sample,app.kubernetes.io/instance=sample" -o jsonpath="{.items[0].metadata.name}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl port-forward $POD_NAME 8080:80
```

`helm install` した際に出力された情報を見ることができる。

## helm delete

クラスターにデプロイされていた kubernetes リソースが削除される。

```sh
$ helm delete soft-meerkat
release "soft-meerkat" deleted

# 確認
$ helm ls -a
NAME            REVISION        UPDATED                         STATUS          CHART           APP VERSION     NAMESPACE
sample          1               Sat Feb 26 12:41:53 2022        DEPLOYED        sample-0.1.0    1.0             application
soft-meerkat    1               Sat Feb 26 12:22:07 2022        DELETED         sample-0.1.0    1.0             default
test            1               Sat Feb 26 12:40:41 2022        DEPLOYED        sample-0.1.0    1.0             application

# 確認
$ kubectl get po,deploy,svc,sa -l app.kubernetes.io/instance=soft-meerkat
No resources found in default namespace.
```

kubernetes クラスタ上ではリソースがすべて削除されているが、helm 上では STATUS が DELETED という状態で保持されている。

完全に削除するには `--purge` というフラグを付ける必要がある。

```sh
helm delete --purge soft-meerkat
release "soft-meerkat" deleted

# 確認
$ helm ls -a
NAME    REVISION        UPDATED                         STATUS          CHART           APP VERSION     NAMESPACE
sample  1               Sat Feb 26 12:41:53 2022        DEPLOYED        sample-0.1.0    1.0             application
test    1               Sat Feb 26 12:40:41 2022        DEPLOYED        sample-0.1.0    1.0             application
```

## helm rollback

release を指定したバージョンにロールバックするコマンド。

soft-meerkat を DELETED にした状態まで準備

```sh
$ helm install --name soft-meerkat sample

$ helm list
NAME            REVISION        UPDATED                         STATUS          CHART           APP VERSION     NAMESPACE
sample          1               Sat Feb 26 12:41:53 2022        DEPLOYED        sample-0.1.0    1.0             application
soft-meerkat    1               Sat Feb 26 14:12:33 2022        DEPLOYED        sample-0.1.0    1.0             default
test            1               Sat Feb 26 12:40:41 2022        DEPLOYED        sample-0.1.0    1.0             application

$ helm delete soft-meerkat
release "soft-meerkat" deleted

$ helm list -a
NAME            REVISION        UPDATED                         STATUS          CHART           APP VERSION     NAMESPACE
sample          1               Sat Feb 26 12:41:53 2022        DEPLOYED        sample-0.1.0    1.0             application
soft-meerkat    1               Sat Feb 26 14:12:33 2022        DELETED         sample-0.1.0    1.0             default
test            1               Sat Feb 26 12:40:41 2022        DEPLOYED        sample-0.1.0    1.0             application
```

ロールバックしてみる

```sh
# ロールバックする REVISION を1に指定
helm rollback soft-meerkat 1
Rollback was a success.

# 確認
$ helm list -a
NAME            REVISION        UPDATED                         STATUS          CHART           APP VERSION     NAMESPACE
sample          1               Sat Feb 26 12:41:53 2022        DEPLOYED        sample-0.1.0    1.0             application
soft-meerkat    2               Sat Feb 26 14:16:05 2022        DEPLOYED        sample-0.1.0    1.0             default
test            1               Sat Feb 26 12:40:41 2022        DEPLOYED        sample-0.1.0    1.0             application
```

release 名に soft meerkat、REVISION に 1 を指定することで、STATUS が DELETED から DEPLOYED 状態にロールバックした。  
また、REVISION が 1 から 2 に繰り上がったことも確認できる。

次に説明する `helm upgrade` で release を更新した後に、指定した REVISION にロールバックさせることもできる。

## helm upgrade

指定した release を更新するコマンド。  
何も更新していないくても REVISION が繰り上がる。

```sh
# 何も更新せず helm upgrade
$ helm upgrade soft-meerkat sample/
Release "soft-meerkat" has been upgraded.
LAST DEPLOYED: Sat Feb 26 14:49:30 2022
NAMESPACE: default
STATUS: DEPLOYED
(省略)

# 確認
$ helm list -a
NAME            REVISION        UPDATED                         STATUS          CHART           APP VERSION     NAMESPACE
sample          1               Sat Feb 26 12:41:53 2022        DEPLOYED        sample-0.1.0    1.0             application
soft-meerkat    3               Sat Feb 26 14:49:30 2022        DEPLOYED        sample-0.1.0    1.0             default
test            1               Sat Feb 26 12:40:41 2022        DEPLOYED        sample-0.1.0    1.0             application
```

## helm plugin

Helm には Helm plugin というアドオンツール群が存在。この helm plugin コマンドはその管理を担う。  
以下 4 つのサブコマンドが存在

```
Available Commands:
    install Install one or more Helm plugins
    list List installed Helm plugins
    remove Remove one or more Helm plugins
    update Update one or more Helm plugins
```

```sh
# plugin 確認
$ helm plugin list
NAME    VERSION DESCRIPTION
```

## helm diff

helm は標準のコマンドだけでは差分を見ることができない。helm-diff というツールを開発しているため、`helm plugin` コマンドで helm-diff をインストールする。

```sh
$ helm plugin install https://github.com/databus23/helm-diff --version master
Downloading https://github.com/databus23/helm-diff/releases/download/v3.4.2/helm-diff-macos-amd64.tgz
Preparing to install into /Users/miyazakinaohiro/.helm/plugins/helm-diff
helm-diff installed into /Users/miyazakinaohiro/.helm/plugins/helm-diff/helm-diff

The Helm Diff Plugin
(省略)

# 確認
$ helm plugin list
NAME    VERSION DESCRIPTION
diff    3.4.2   Preview helm upgrade changes as a diff
```

以下のようなコマンドを使えば差分を見れる (今回は差分が無い為何も表示されない)

```sh
$ helm diff revision soft-meerkat 2 3
```

### 差分を作ってやってみた

1. `values.yaml` の iamge.pullPolicy を IfNotPresent => Always に変更
2. `helm upgrade` 実行

   ```sh
   $ helm upgrade soft-meerkat sample/
   Release "soft-meerkat" has been upgraded.
   (省略)

   $ helm list -a | cut -f 1,2,3
   NAME            REVISION        UPDATED
   sample          1               Sat Feb 26 12:41:53 2022
   soft-meerkat    4               Sat Feb 26 15:11:46 2022
   test            1               Sat Feb 26 12:40:41 2022

   # 変更があると kubernetes リソースも生まれ変わるみたい
   $ kubectl get po
   NAME                                   READY   STATUS        RESTARTS   AGE
   soft-meerkat-sample-5b7856978c-wjrn5   1/1     Running       0          2s
   soft-meerkat-sample-d8865bf6f-ktfrs    1/1     Terminating   0          19m
   ```

3. `helm diff` 実行
   ```
   $ helm diff revision soft-meerkat 3 4
   default, soft-meerkat-sample, Deployment (apps) has changed:
   # Source: sample/templates/deployment.yaml
   (省略)
   - imagePullPolicy: IfNotPresent
   + imagePullPolicy: Always
   (省略)
   ```

## helm history

release の履歴を表示するコマンド。

```sh
$ helm history soft-meerkat
REVISION        UPDATED                         STATUS          CHART           APP VERSION     DESCRIPTION
1               Sat Feb 26 14:12:33 2022        SUPERSEDED      sample-0.1.0    1.0             Deletion complete
2               Sat Feb 26 14:16:05 2022        SUPERSEDED      sample-0.1.0    1.0             Rollback to 1
3               Sat Feb 26 14:49:30 2022        SUPERSEDED      sample-0.1.0    1.0             Upgrade complete
4               Sat Feb 26 15:11:46 2022        DEPLOYED        sample-0.1.0    1.0             Upgrade complete
```

`helm history` で確認できた REVISION を指定すれば `helm rollback` でロールバックすることも可能。

## 参考

- [Helm の official chart repository が クローズとなった](https://note.com/loftkun/n/n6a62995fe2ee)
- [Helm のサブコマンドを全部使ってみた](https://dev.classmethod.jp/articles/helm-sub-commands/#:~:text=%E5%A4%B1%E6%95%97%E3%81%97%E3%81%BE%E3%81%97%E3%81%9F%E3%81%AD%E2%80%A6%E8%AA%BF%E3%81%B9%E3%81%9F%E3%81%A8%E3%81%93%E3%82%8D%E3%81%AB%E3%82%88%E3%82%8B%E3%81%A8%E3%80%81mariadb%E3%83%81%E3%83%A3%E3%83%BC%E3%83%88%E9%85%8D%E5%B8%83%E5%85%83%E3%81%AEhttps%3A//kubernetes%2Dcharts.storage.googleapis.com/%E3%83%AA%E3%83%9D%E3%82%B8%E3%83%88%E3%83%AA%E3%81%8C%E5%BB%83%E6%AD%A2%E3%81%95%E3%82%8C%E3%80%81%E4%BB%A3%E3%82%8F%E3%82%8A%E3%81%ABhttps%3A//charts.helm.sh/stable%E3%82%92%E4%BD%BF%E3%81%86%E5%BF%85%E8%A6%81%E3%81%8C%E3%81%82%E3%82%8B%E3%82%88%E3%81%86%E3%81%A7%E3%81%99%E3%80%82%E3%81%A8%E3%81%84%E3%81%86%E3%82%8F%E3%81%91%E3%81%A7%E5%85%88%E7%A8%8B%E3%81%AEYAML%E3%82%92%E6%9B%B4%E6%96%B0%E3%81%97%E3%81%BE%E3%81%99%E3%80%82)
- https://github.com/helm/charts
