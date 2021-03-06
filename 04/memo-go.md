# GO アプリケーションを chart 化する

## chart を作る順番

chart を作る際は、先に動作確認の取れた kubernetes リソースをマニフェスト化しておくのが良い。
このことを踏まえると以下の流れがおすすめ。

1. kubernetes マニフェストの作成、動作確認
2. マニフェストの template 化

3. lint や test で静的解析と動作確認
4. chart の公開

## サンプルアプリとマニフェスト

こちらの kubernetes マニフェストを題材に helm chart を作っていく。

[サンプルアプリ](https://github.com/govargo/go-happyHelming)
[サンプルマニフェスト](https://github.com/govargo/sample-charts/blob/master/kubernetes/happyHelming.yaml)
こちらのアプリは HTTP リクエストパスに渡された値をもとに「Happy Helming, XXX!」と表示するだけのアプリ。

```sh
# LoadBalancer の場合
$ kubectl apply -f sample-charts-master/kubernetes/happyHelming.yaml
$ curl http://localhost:80/miyazaki
Happy Helming, miyazaki!%

# NodePort の場合
$ kubectl get svc
NAME           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
echo-service   NodePort    10.102.245.195   <none>        80:30125/TCP   7m3s

$ curl http://127.0.0.1:30125/miya
Happy Helming, miya!%
```

kubernetes マニフェストは deployment と service を利用している。構成図は P93 を参考にされたし

## template 化

共通的に利用できる部分、頻繁に変更できる部分を考慮して template 化する。  
今回のアプリは対象外だが、たいていの chart では PV、Secret、Ingress の有無なども template 化の対象になることがある。

今回は deployment.yaml の次の部分を template 化する。

- metadata.name
- labels ブロック
- replicas の数
- image 名とタグ
- imagePullPolicy
- livenessProbe と readinessProbe

service.yaml は次の部分を template 化する

- metadata.name
- type
- selector
- port 番号

事前準備として helm create で chart を作成し、不要なファイルを消しておく。
NOTES.txt や \_helpers.tpl と test ディレクトリーは再利用するため、残しておく。

```sh
$ helm create happyhelm

# templates ディレクトリーのyaml ファイル削除
$ rm -rf happyhelm/tempates/*.yaml

# 確認
$ tree happyhelm
happyhelm
├── Chart.yaml
├── charts
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   └── tests
│       └── test-connection.yaml
└── values.yaml

3 directories, 5 files
```

\_helpers.tpl ファイルは初期状態のまま利用する。

### deployment.yaml の template 化

まずファイルを作る。

```
$ touch happyhelm/templates/deployment.yaml
```

そして、`sample-charts-master/kubernetes/happyHelming.yaml` の中身から deployment の部分をコピペしてくる。

<details>
<summary>deployment.yaml</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: happy-helming
  labels:
    app: happy-helming
    version: only-echo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: happy-helming
      version: only-echo
  template:
    metadata:
      labels:
        app: happy-helming
        version: only-echo
    spec:
      containers:
        - name: echo-happy-helming
          image: govargo/happy-helming:only-echo
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
          livenessProbe:
            tcpSocket:
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
          readinessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
```

</details>

上から順に template 化していく。まずは以下のように template 化してみる。

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "happyhelm.fullname" . }}
  labels:
    app.kubernetes.io/name: {{ include "happyhelm.name" . }}
    helm.sh/chart: {{ include "happyhelm.chart" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
```

name と label を template 化した。

name: {{ include "happyhelm.name" . }} は `_helpers.tpl` から定義を読み込んでいる。
labels は[ベストプラクティス](https://v2.helm.sh/docs/chart_best_practices/#labels-and-annotations)で定義されている標準的な label を適用した。

次に spec の情報を以下のように template 化した。

```
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ include "happyhelm.name" . }}
      app.kubernetes.io/instance: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app.kubernetes.io/name: {{ include "happyhelm.name" . }}
        app.kubernetes.io/instance: {{ .Release.Name }}
```

replicaset の数を values.yaml で変更できるように template 化した。  
また、selector や template:metadata も上の labels と一致するように書き換えた。

次に pod の情報を以下のように template 化した。

```
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 8080
```

name は chart の名前で定義した。  
image は {{ .Values.image.repository }}:{{ .Values.image.tag }} でパラメータを可変できるようにした。(values.yaml の image 名も変更すること)  
また、imagePullPolicy も利用者の好みや要件で変更できるように template 化した。

次に livenessProbe と readnessProbe の設定を template 化してみる。

```
          livenessProbe:
            {{- toYaml .Values.livenessProbe | nindent 12 }}
          readinessProbe:
            {{- toYaml .Values.readinessProbe | nindent 12 }}
```

かなり省略できた。
{{- toYaml .Values.livenessProbe | nindent 10 }} のように書くことで、values,yaml に記載している定義を読み込む。
その後 nindent 12 でインデントを整え、テンプレートエンジンにより yaml が定義されているように振る舞う。（toYaml がないとエラーになる。）

values.yaml には livenessProbe と readnessProbe の設定は記載されていないため追記する。

```yaml
livenessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
readinessProbe:
  httpGet:
    path: /
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

これらによる出力結果は以下のようになる。

```yaml
livenessProbe:
  initialDelaySeconds: 5
  periodSeconds: 5
  tcpSocket:
    port: 8080

readinessProbe:
  httpGet:
    path: /
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

不要な改行が入ってしまって入るが実行に問題はない。  
この改行を直そうとして {{- toYaml .Values.livenessProbe | nindent 10 -}} のように liveness と readness ともに修正すると以下のような出力になって、普通に行けそうだが？？？（本と言っていることが違うので要検証）

ひとまず改行されている方ですすめる。

```yaml
livenessProbe:
  initialDelaySeconds: 5
  periodSeconds: 5
  tcpSocket:
    port: 8080
readinessProbe:
  httpGet:
    path: /
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

これで deployment の template 化は完了した。
だが、利用者の好み、要件に応じて resource を設定できるように template を追加する。

```
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

合わせて values.yaml を以下のように定義する。（すでに定義してあったので確認しただけ）

```yaml
resources:
  {}
  # We usually recommend not to specify default resources and to leave this as a conscious
  # choice for the user. This also increases chances charts run on environments with little
  # resources, such as Minikube. If you do want to specify resources, uncomment the following
  # lines, adjust them as necessary, and remove the curly braces after 'resources:'.
  # limits:
  #   cpu: 100m
  #   memory: 128Mi
  # requests:
  #   cpu: 100m
  #   memory: 128Mi
```

具体的な設定値はコメントアウトされている。デフォルト値では resource は殻で設定される。  
任意でコメントアウトを外して、resources の値を設定してマニフェストに反映する。

以上で deployment.yaml の template 化が完了した。完成形は以下である。

<details>
<summary>deployment.yaml 完成形</summary>

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "happyhelm.fullname" . }}
  labels:
    app.kubernetes.io/name: {{ include "happyhelm.name" . }}
    helm.sh/chart: {{ include "happyhelm.chart" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ include "happyhelm.name" . }}
      app.kubernetes.io/instance: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app.kubernetes.io/name: {{ include "happyhelm.name" . }}
        app.kubernetes.io/instance: {{ .Release.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 8080
          livenessProbe:
            {{- toYaml .Values.livenessProbe | nindent 12 }}
          readinessProbe:
            {{- toYaml .Values.readinessProbe | nindent 12 }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

</details>

### service.yaml の template 化

deployment と同じように template 化すれば良い。

```sh
$ touch happyhelm/templates/service.yaml
```

そして、`sample-charts-master/kubernetes/happyHelming.yaml` の中身から service の部分をコピペしてくる。

<details>
<summary>service.yaml</summary>

```yaml
kind: Service
apiVersion: v1
metadata:
  name: echo-service
spec:
  type: LoadBalancer
  selector:
    app: happy-helming
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

</details>

まず metadata の下に label の情報をつける。書く内容は deployment と同じ。  
spec の中の selector は deployment と同じよう template 化。  
そして type と port も template 化すれば汎用的に使えるため良い

<details>
<summary>service.yaml 完成形</summary>

```
kind: Service
apiVersion: v1
metadata:
  name: {{ include "happyhelm.fullname" . }}
  labels:
    app.kubernetes.io/name: {{ include "happyhelm.name" . }}
    helm.sh/chart: {{ include "happyhelm.chart" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
spec:
  type: {{ .Values.service.type }}
  selector:
    app.kubernetes.io/name: {{ include "happyhelm.name" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
  ports:
    - protocol: TCP
      port: {{ .Values.service.port }}
      targetPort: 8080

```

</details>

そして values.yaml の完成形は以下のようになった。

<details>
<summary>values.yaml 完成形</summary>

```
# Default values for happyhelm.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 1

image:
  repository: govargo/happy-helming
  tag: only-echo
  pullPolicy: Always

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

livenessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
readinessProbe:
  httpGet:
    path: /
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10

serviceAccount:
  # Specifies whether a service account should be created
  create: true
  # The name of the service account to use.
  # If not set and create is true, a name is generated using the fullname template
  name: ""

podSecurityContext:
  {}
  # fsGroup: 2000

securityContext:
  {}
  # capabilities:
  #   drop:
  #   - ALL
  # readOnlyRootFilesystem: true
  # runAsNonRoot: true
  # runAsUser: 1000

service:
  type: LoadBalancer
  port: 80

ingress:
  enabled: false
  annotations:
    {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  hosts:
    - host: chart-example.local
      paths: []

  tls: []
  #  - secretName: chart-example-tls
  #    hosts:
  #      - chart-example.local

resources:
  {}
  # We usually recommend not to specify default resources and to leave this as a conscious
  # choice for the user. This also increases chances charts run on environments with little
  # resources, such as Minikube. If you do want to specify resources, uncomment the following
  # lines, adjust them as necessary, and remove the curly braces after 'resources:'.
  # limits:
  #   cpu: 100m
  #   memory: 128Mi
  # requests:
  #   cpu: 100m
  #   memory: 128Mi

nodeSelector: {}

tolerations: []

affinity: {}


```

</details>

### NOTES.txt の編集

次はインストールの手引を記す NOTES.txt を編集する。NOTES.txt にも template 化や pipeline や function が利用できる。

一般的には service の type に if で条件分岐してアプリケーションのアクセスの仕方を表示する。とりあえずデフォルトのままで変更せずに行く。

<details>
<summary>NOTES.txt</summary>

```
1. Get the application URL by running these commands:
{{- if .Values.ingress.enabled }}
{{- range $host := .Values.ingress.hosts }}
  {{- range .paths }}
  http{{ if $.Values.ingress.tls }}s{{ end }}://{{ $host.host }}{{ . }}
  {{- end }}
{{- end }}
{{- else if contains "NodePort" .Values.service.type }}
  export NODE_PORT=$(kubectl get --namespace {{ .Release.Namespace }} -o jsonpath="{.spec.ports[0].nodePort}" services {{ include "happyhelm.fullname" . }})
  export NODE_IP=$(kubectl get nodes --namespace {{ .Release.Namespace }} -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT
{{- else if contains "LoadBalancer" .Values.service.type }}
     NOTE: It may take a few minutes for the LoadBalancer IP to be available.
           You can watch the status of by running 'kubectl get --namespace {{ .Release.Namespace }} svc -w {{ include "happyhelm.fullname" . }}'
  export SERVICE_IP=$(kubectl get svc --namespace {{ .Release.Namespace }} {{ include "happyhelm.fullname" . }} --template "{{"{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}"}}")
  echo http://$SERVICE_IP:{{ .Values.service.port }}
{{- else if contains "ClusterIP" .Values.service.type }}
  export POD_NAME=$(kubectl get pods --namespace {{ .Release.Namespace }} -l "app.kubernetes.io/name={{ include "happyhelm.name" . }},app.kubernetes.io/instance={{ .Release.Name }}" -o jsonpath="{.items[0].metadata.name}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl port-forward $POD_NAME 8080:80
{{- end }}

```

</details>

## helm lint による静的解析。

helm lint で構文チェックや推奨構成を見たいているのか確認する。

```sh
$ helm lint happyhelm
==> Linting happyhelm
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, no failures
```

## helm test によるテスト

構文チェックが終わった後は helm test で release が期待通りに動いているかをチェックする。

helm test は template/test 配下の yaml ファイルを実行する。コンテナが正常に終了すると(exit 0)と成功になる。  
テスト用の yaml ファイルを記述する必要があるが詳しい内容は割愛する。

テストを実行するには事前に helm install で release を作成して helm test で指定する必要がある。

helm install

```sh
$ helm install --name happyhelm happyhelm
NAME:   happyhelm
LAST DEPLOYED: Tue Mar  8 22:01:59 2022
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Deployment
NAME       READY  UP-TO-DATE  AVAILABLE  AGE
happyhelm  0/1    1           0          0s

==> v1/Pod(related)
NAME                        READY  STATUS             RESTARTS  AGE
happyhelm-55cb7f5cf7-mtktk  0/1    ContainerCreating  0         1s

==> v1/Service
NAME       TYPE          CLUSTER-IP      EXTERNAL-IP  PORT(S)       AGE
happyhelm  LoadBalancer  10.108.107.201  localhost    80:31572/TCP  0s


NOTES:
1. Get the application URL by running these commands:
     NOTE: It may take a few minutes for the LoadBalancer IP to be available.
           You can watch the status of by running 'kubectl get --namespace default svc -w happyhelm'
  export SERVICE_IP=$(kubectl get svc --namespace default happyhelm --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
  curl http://$SERVICE_IP:80
```

NodePort の場合、インストール時に出てくる NOTES の内容で取ってこれるだとうまく行かなかったので色々調べたところ、以下のどちらかコマンドで curl したほうがいいとのこと

```sh
$ curl http://localhost:$NODE_PORT
$ curl http://127.0.0.1:$NODE_PORT
```

テスト

```sh
$ helm test happyhelm
RUNNING: happyhelm-test-connection
PASSED: happyhelm-test-connection
```

無事テストを通過し成功した。

ちなみに NOTES にあるコマンドを実行して URL を取得し、curl コマンドでつないでみると

```sh
$ export SERVICE_IP=$(kubectl get svc --namespace default happyhelm --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
  echo http://$SERVICE_IP:80
http://localhost:80

$ curl http://localhost:80/miyazaki
Happy Helming, miyazaki!%
```

無事起動していることがわかる。

### chart のパッケージ

構文チェックとテストが終わったため、chart を公開用に tar ボールに固める。
公開に際し、chart の情報をまとめる chart.yaml を編集する。

happyhelm/Chart.yaml

```yaml
apiVersion: v1
name: happyhelm
version: 1.0.0
appVersion: 1.0.0
description: Echo Happy happy-helming
sources:
  - http://github.com/govargo/go-happyhelming
maintainers:
  - name: go_vargo
engine: gotpl
```

chart のバージョン、アプリのバージョン、説明、ソース情報などを記載する。
これとは別に README.md も用意するべし。README.md には chart のインストールコマンドやパラメータの説明などを記載する。

chart の情報を更新したら、helm package コマンドで tar ボールに詰める。

```sh
$ helm package happyhelm
Successfully packaged chart and saved it to: /Users/miyazakinaohiro/github/helm-practice/04/happyhelm-1.0.0.tgz
```

`happyhelm-1.0.0.tgz` が作成された。

次に chart の目録になる index.yaml を作成する。

index.yaml ファイルは「helm repo index ＜ディレクトリ＞」コマンドで作成できる。
--url で chart リポジトリをつけることで chaart リポジトリを index.yaml に指定できる。

```yaml
apiVersion: v1
entries:
  happyhelm:
    - apiVersion: v1
      appVersion: 1.0.0
      created: "2022-03-08T22:27:20.97378+09:00"
      description: Echo Happy happy-helming
      digest: 26ae993e0a2531774f91be43b7943b46b08804fd37e683ca00a108a879f7c075
      engine: gotpl
      maintainers:
        - name: go_vargo
      name: happyhelm
      sources:
        - http://github.com/govargo/go-happyhelming
      urls:
        - https://miyajuggler.github.io/charts-repository/happyhelm-1.0.0.tgz
      version: 1.0.0
generated: "2022-03-08T22:27:20.97238+09:00"
```

chart の情報やリンクなどが記録されている。
index.yaml とチャートの.tgz ファイルが HTTP(または HTTPS)でサービスされる URL が Helm チャートリポジトリである

### chart リポジトリーへの公開

今回は github pages を利用して chart を公開する。（ぶっちゃけ HTTP、HTTPS でアクセスできる web サーバーなら何でもいい）

```sh
$ tree github.io
github.io
├── charts-repository
│   ├── README.md
│   ├── happyhelm-1.0.0.tgz
│   └── index.yaml
└── index.html
```

```sh
# github 上のchart リポジトリを追加
$ helm repo add miyajuggler https://miyajuggler.github.io/charts-repository
"miyajuggler" has been added to your repositories

# リポジトリ情報の更新
$ helm update
Command "update" is deprecated, Use 'helm repo update'

Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "miyajuggler" chart repository
...Successfully got an update from the "incubator" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete.

# chart リポジトリの一覧表示
$ helm repo list
NAME            URL
stable          https://charts.helm.sh/stable
local           http://127.0.0.1:8879/charts
incubator       https://charts.helm.sh/incubator
miyajuggler     https://miyajuggler.github.io/charts-repository

# 確認
$ helm search miyajuggler
NAME                    CHART VERSION   APP VERSION     DESCRIPTION
miyajuggler/happyhelm   1.0.0           1.0.0           Echo Happy happy-helming
```

miyajuggler/happyhelm が chart リポジトリに登録されていることが確認できた。
これにより helm fetch や helm install コマンドもできるようになった。

最後に、作成した chart のインストールをしてみて終わりとする。

```sh
helm install --name miya miyajuggler/happyhelm                               ○ docker-desktop
NAME:   miya
LAST DEPLOYED: Tue Mar  8 23:45:34 2022
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Deployment
NAME            READY  UP-TO-DATE  AVAILABLE  AGE
miya-happyhelm  0/1    1           0          0s

==> v1/Pod(related)
NAME                            READY  STATUS             RESTARTS  AGE
miya-happyhelm-785c78759-wsbjm  0/1    ContainerCreating  0         0s

==> v1/Service
NAME            TYPE          CLUSTER-IP     EXTERNAL-IP  PORT(S)       AGE
miya-happyhelm  LoadBalancer  10.103.41.163  localhost    80:31662/TCP  0s


NOTES:
1. Get the application URL by running these commands:
     NOTE: It may take a few minutes for the LoadBalancer IP to be available.
           You can watch the status of by running 'kubectl get --namespace default svc -w miya-happyhelm'
  export SERVICE_IP=$(kubectl get svc --namespace default miya-happyhelm --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
  echo http://$SERVICE_IP:80
```

## 参考

service を NodePort で指定した場合、もともと書かれていた NOTES.txt のコードではうまく接続できなかったので、以下を参考にした

[【Kubernetes】Service を作成してローカル PC からアクセスしてみる](https://amateur-engineer-blog.com/kubernetes-service/)

[Kubernetes の NodePort に接続できない時の原因と対策](https://qiita.com/zousan010sanpotyu/items/208dd6b4cb00f88eca67)

ClusterIP の Service はクラスタ内部からしかアクセスできないため、ポートフォワードしてローカル PC から確認する。
