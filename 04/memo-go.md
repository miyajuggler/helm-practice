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
image は {{ .Values.image.repository }}:{{ .Values.image.tag }} でパラメータを可変できるようにした。  
また、imagePullPolicy も利用者の好みや要件で変更できるように template 化した。

次に livenessProbe と readnessProbe の設定を template 化してみる。

```
          livenessProbe:
            {{- toYaml .Values.livenessProbe | nindent 10 }}
          readinessProbe:
            {{- toYaml .Values.readinessProbe | nindent 10 }}
```

かなり省略できた。
{{- toYaml .Values.livenessProbe | nindent 10 }} のように書くことで、values,yaml に記載している定義を読み込む。
その後 nindent 10 でインデントを整え、テンプレートエンジンにより yaml が定義されているように振る舞う。（toYaml がないとエラーになる。）

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
            {{- toYaml .Values.resources | nindent 10 }}
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
