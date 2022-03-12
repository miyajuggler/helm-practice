# Helm Cart を発展させる

## SubCharts とは

ひとつの chart の中で複数のソフトウェアを管理することができる。  
また、ある chart の中に別の chart を含ませて、その chart を利用することもできる。  
こういった chart 間で依存関係をもたせる仕組みを subcharts と呼ぶ。

## Happy Helming Chart を SubChart 化する

Envoy Chart を subcharts として加える。
envoy がリクエストを受け、後ろに控えた happy helming の service リソースに対してリクエストをプロキシする。
逆に happy helming の pod のリクエストを envoy が受け取ってクライアントまでリクエストを返す。

詳しい図は教科書にて。

### chart の依存関係

依存関係を扱うための仕組みは requirements.yaml である。
requirements.yaml には依存する chart の名前やバージョン、chart リポジトリの情報を記載する。

今回使う envoy chart は stable 版の envoy chart を利用する

準備

```sh
$ cp  -r ~/github/helm-practice/04/happyhelm .
$ cp  -r ~/github/helm-practice/04/index.yaml .
$ touch happyhelm/repuirements.yaml
```

version は 1.5.0、リポジトリは公式リポジトリを指定した。

確認

```sh
helm repo list
NAME            URL
stable          https://charts.helm.sh/stable
```

requirements.yaml

```yaml
dependencies:
  - name: envoy
    version: 1.5.0
    repository: https://charts.helm.sh/stable
```

requirement.yaml が存在するディレクトリーに移動して helm dependency update を実行すると、依存対象の subchart が chart ディレクトリー配下にダウンロードされる。

```sh
$ helm dependency update
Hang tight while we grab the latest from your chart repositories...
...Unable to get an update from the "local" chart repository (http://127.0.0.1:8879/charts):
        Get "http://127.0.0.1:8879/charts/index.yaml": dial tcp 127.0.0.1:8879: connect: connection refused
...Successfully got an update from the "miyajuggler" chart repository
...Successfully got an update from the "govargo" chart repository
...Successfully got an update from the "incubator" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete.
Saving 1 charts
Downloading envoy from repo https://charts.helm.sh/stable
Deleting outdated charts
```

charts 配下に envoy-1.5.0.tgz、happyhelm 配下に requirements.lock が追加されている。

```sh
$ tree happyhelm
happyhelm
├── Chart.yaml
├── charts
│   └── envoy-1.5.0.tgz
├── requirements.lock
├── requirements.yaml
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── service.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml

3 directories, 10 files
```

requirements.lock には subcharts の情報が保持されている。

```yaml
dependencies:
  - name: envoy
    repository: https://charts.helm.sh/stable
    version: 1.5.0
digest: sha256:71029714e1ff0adf6b6ae7a08767185e981d0b02d016f0b42735adb13590dfad
generated: "2022-03-10T22:45:27.056801+09:00"
```

### values.yaml で subchart の挙動を定義する

subchart として envoy chart を取り込んだので、happy helming 用に envoy の設定を行う。  
envoy chart は tar ボールとして固まっているので、親 chart である happy helming chart の values.yaml にて定義を行う。

happyhelm/values.yaml を編集していく。
まずは happy helming 自体の変更点を修正する。
LoadBalancer から ClusterIP に変更する。

```diff
  service:
-   type: LoadBalancer
+   type: ClusterIP
    port: 80
```

ClusterIP にしたことで kubernetes クラスター外からはアクセスできなくなり、クラスター内部からの通信だけを受け付ける状態になった。

次に envoy の定義を書く。
以下のようにトップレベルに envoy として、subchart 用の値を定義していく。

envoy chart のデフォルト値は envoy-1.5.0.tgz に定義されている。デフォルト値から変更したい部分だけ values.yaml に書いていく。

全文はこちら => https://github.com/helm/charts/blob/master/stable/envoy/values.yaml

変更部分は教科書を参照。

envoy の転送先である socket_address に happyhelm を指定しているが、現在の templates/service.yaml での name は include "happyhelm.fullname" となっている。  
このままだと必ずしも happyhelm になるとは限らなくなる（--name で指定した名前 + happyhelm となるため）
なので以下のように修正する。

values.yaml

```diff
  service:
+   name: happyhelm
    type: ClusterIP
    port: 80
```

templates/service.yaml

```diff
  kind: Service
  apiVersion: v1
  metadata:
-   name: {{ include "happyhelm.fullname" . }}
+   name: {{ .Values.service.name }}
    labels:
```

### NOTES.txt の編集

subchart として envoy を加えた文の使い方を記載する NOTES.txt を編集する。  
ざっくりいうと envoy プロキシにアクセスするための方法を表示する。
詳細は教科書を参考にされたし。

### helm lint による静的解析

構文チェックと推奨構成を満たしているかを確認する。

```sh
$ helm lint happyhelm
==> Linting happyhelm
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, no failures
```

no failure なので OK

### helm test によるテスト

テストをする前にテスト内容も envoy 用に変更する。

ざっくりいうと curl で HTTP ステータスが正常に返ってくるかどうかの判断を service から envoy に変更するだけである。
詳細は教科書を参考にされたし。

では helm install で release を作成してから helm test してみる。

```
$ helm install --name happyhelm happyhelm
NAME:   happyhelm
LAST DEPLOYED: Thu Mar 10 23:31:33 2022
NAMESPACE: default
STATUS: DEPLOYED

(省略)

$ helm test happyhelm
RUNNING: happyhelm-test-connection
PASSED: happyhelm-test-connection
```

### chart のパッケージ

chart を tar ボールに詰める。公開に際し chart の情報をまとめる chart.yaml を編集する。

chart のバージョンである version とアプリケーションのバージョンである appVersion がある。
今回はアプリケーション自体には変更を加えておらず、chart に関しては subchart を追加したため version を更新する。

```yaml
apiVersion: v1
name: happyhelm
version: 1.1.0
appVersion: 1.0.0
description: Echo Happy happy-helming(via Envoy Proxy)
sources:
  - http://github.com/govargo/go-happyhelming
maintainers:
  - name: go_vargo
engine: gotpl
```

変更内容がわかりやすいように description も更新した。

また、README.md も更新しておく。今回は subchart 用に追加したパラメーターを追記するだけ。

chart 情報を更新し終わったら helm package する

```sh
$ helm package happyhelm
Successfully packaged chart and saved it to: /Users/miyazakinaohiro/github/helm-practice/05/happyhelm-1.1.0.tgz
```

happyhelm-1.1.0.tgz が作成された。

### chart リポジトリの更新

chart の目録である index.yaml を更新する。そのためにまず happyhelm-1.1.0.tgz を公開用のリポジトリに配置し、README.md も変更しておく

```sh
$ tree charts-repository
charts-repository
├── README.md
├── happyhelm-1.0.0.tgz
├── happyhelm-1.1.0.tgz
└── index.yaml

0 directories, 4 files
```

charts-repository にて helm repo index コマンドを打つ

```sh
$ github.io/charts-repository main ❯ helm repo index ./ --url https://miyajuggler.github.io/charts-repository/
```

以下のように更新された。

```yaml
apiVersion: v1
entries:
  happyhelm:
    - apiVersion: v1
      appVersion: 1.0.0
      created: "2022-03-11T10:42:13.827357+09:00"
      description: Echo Happy happy-helming(via Envoy Proxy)
      digest: fb43a87c487a2cb11adaa48be82de97b5aa8e91966005163f222faac58faf384
      engine: gotpl
      maintainers:
        - name: go_vargo
      name: happyhelm
      sources:
        - http://github.com/govargo/go-happyhelming
      urls:
        - https://miyajuggler.github.io/charts-repository/happyhelm-1.1.0.tgz
      version: 1.1.0
    - apiVersion: v1
      appVersion: 1.0.0
      created: "2022-03-11T10:42:13.821315+09:00"
      description: Echo Happy happy-helming
      digest: 6b8ea30e1f7f25d0e02ab078806700e9df53d31b4bb2620ed88359360de9cf07
      engine: gotpl
      maintainers:
        - name: go_vargo
      name: happyhelm
      sources:
        - http://github.com/govargo/go-happyhelming
      urls:
        - https://miyajuggler.github.io/charts-repository/happyhelm-1.0.0.tgz
      version: 1.0.0
generated: "2022-03-11T10:42:13.820041+09:00"
```

version: 1.0.0 と version: 1.1.0 の chart が目録として追加されている。

github にプッシュ完了したら helm client 側のリポジトリの更新を行う
helm では明示的に client 側のリポジトリ情報を更新する必要があることに注意

```sh
$ helm search happyhelm
NAME                    CHART VERSION   APP VERSION     DESCRIPTION
govargo/happyhelm       1.1.0           1.0.0           Echo Happy Helming(via Envoy Proxy).
local/happyhelm         1.1.0           1.0.0           Echo Happy happy-helming(via Envoy Proxy)
miyajuggler/happyhelm   1.0.0           1.0.0           Echo Happy happy-helming
```

```sh
$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "govargo" chart repository
...Successfully got an update from the "miyajuggler" chart repository
...Successfully got an update from the "incubator" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete.
```

```sh
$ helm search happyhelm                                               ○ docker-desktop
NAME                    CHART VERSION   APP VERSION     DESCRIPTION
govargo/happyhelm       1.1.0           1.0.0           Echo Happy Helming(via Envoy Proxy).
local/happyhelm         1.1.0           1.0.0           Echo Happy happy-helming(via Envoy Proxy)
miyajuggler/happyhelm   1.1.0           1.0.0           Echo Happy happy-helming(via Envoy Proxy)
```

CHART VERSION がバージョンアップしていることが確認できた。
これで 1.1.0 の chart に対して helm fetch や helm install コマンドが利用できる。

以下のように古いバージョンを指定したい場合は --version フラグを付けることで可能となる

```sh
$ helm install --name happyhelm --version 1.0.0 miyajuggler/happyhelm
```

それでは以下のコマンドで最新版を取ってきてみる

```sh
$ helm install --name miya miyajuggler/happyhelm
NAME:   miya
LAST DEPLOYED: Fri Mar 11 22:29:58 2022
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/ConfigMap
NAME        DATA  AGE
miya-envoy  1     0s

==> v1/Deployment
NAME            READY  UP-TO-DATE  AVAILABLE  AGE
miya-envoy      0/2    2           0          0s
miya-happyhelm  0/1    1           0          0s

==> v1/Pod(related)
NAME                            READY  STATUS             RESTARTS  AGE
miya-envoy-54cbb594c7-fmnzs     0/1    ContainerCreating  0         0s
miya-envoy-54cbb594c7-qnpcl     0/1    ContainerCreating  0         0s
miya-happyhelm-785c78759-42ztc  0/1    ContainerCreating  0         0s

==> v1/Service
NAME       TYPE          CLUSTER-IP      EXTERNAL-IP  PORT(S)          AGE
envoy      LoadBalancer  10.98.233.26    localhost    10000:31162/TCP  0s
happyhelm  ClusterIP     10.110.231.183  <none>       80/TCP           0s

==> v1beta1/PodDisruptionBudget
NAME        MIN AVAILABLE  MAX UNAVAILABLE  ALLOWED DISRUPTIONS  AGE
miya-envoy  N/A            1                0                    0s


NOTES:
1. Get the application URL by running these commands:
     NOTE: It may take a few minutes for the LoadBalancer IP to be available.
           You can watch the status of by running 'kubectl get --namespace default svc -w envoy'
  export SERVICE_IP=$(kubectl get svc --namespace default envoy -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
  curl http://127.0.0.1:10000
```
