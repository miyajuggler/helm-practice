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

```yaml
service:
(-) ype: LoadBalancer
(+) type: ClusterIP
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

```yaml
service:
(+) name: happyhelm
    type: ClusterIP
    port: 80
```

templates/service.yaml

```yaml
kind: Service
apiVersion: v1
metadata:
(-) name: {{ include "happyhelm.fullname" . }}
(+) name: {{ .Values.service.name }}
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
