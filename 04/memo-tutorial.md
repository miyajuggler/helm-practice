# helm chart 作成チュートリアル

## helm create

```sh
# mychart 作成
$ helm create mychart
Creating mychart

# ディレクトリーの中身を確認
$ tree mychart
mychart
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

## template ファイルの作成

template 配下のすべてのファイルを削除する。これはフルスクラッチで作るためである。  
合わせて `values.yaml` の中身を初期化します。

```sh
# templates 配下のファイルを削除
$ rm -rf mychart/templates

# values.yaml を初期化
$ echo "" > mychart/values.yaml

# 確認
$ tree mychart
mychart
├── Chart.yaml
├── charts
└── values.yaml

1 directory, 2 files
```

すべての template ファイルを削除できたので、template ファイルを作成していく。

まずは configmap を作る。
configmap は環境変数や設定情報を定義するのに利用するリソースである。

key に「myvalue」、value に「"Hello World"」を定義する

```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-configmap
data:
  myvalue: "Hello World"
```

templates ディレクトリに配置したファイルは、Helm コマンド実行時にテンプレートエンジンに送られる。  
今回は何も変数化していないため、Tiller がこの template を読んでも、そのままの値が kubernetes に送られる。

この単純な template だけでもインストール可能なので試しにやってみる。

```sh
$ helm install mychart
NAME:   dinky-vulture
LAST DEPLOYED: Tue Mar  1 20:52:15 2022
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/ConfigMap
NAME               DATA  AGE
mychart-configmap  1     0s

# 確認
$ kubectl get cm
NAME                DATA   AGE
kube-root-ca.crt    1      121d
mychart-configmap   1      27s
```

## template ファイルを変数化

「name」は release ごとにユニークであるべき為、変数化してみる。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
```

`{{ .Release.Name }}-configmap` が変数した箇所で、組み込み変数と呼ばれる予約語。

`helm install --debug --dry-run` でどのように展開されるかをインストールすることなく確認することができる。

```sh
$ helm install --debug --dry-run
(省略)

NAME:   soft-anteater
REVISION: 1
RELEASED: Tue Mar  1 21:13:51 2022
CHART: mychart-0.1.0

(省略)

---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: soft-anteater-configmap
data:
  myvalue: "Hello World"
```

release 名である「soft-anteater」が代入されているのがわかる。

## 組み込み変数

利用頻度が高いのは、Release、Chart、Values の 3 種類。

Release は release に関するメタ情報

Chart は chart に関するメタ情報

Values 最も利用頻度の高い変数
ベストプラクティスとしては以下のようになるべく階層を持たずにフラットに変数定義をするのがいいらしい。

```yaml
# 階層パターン
server:
  name: nginx
  port: 80

# フラット
serverName: nginx
serverPort: 80
```

## Values File で変数化

`values.yaml` を使った変数化をやってみる。
以下のようなパラメーターを一つ定義する。

```sh
$ echo "favoriteDrink: coffee" > mychart/values.yaml
favoriteDrink: coffee mychart/values.yaml
```

templates/configmap.yaml に key「drink」を追加

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favoriteDrink}}
```

これで `values.yaml` で定義されている値が取得できる。

デバッグ

```sh
$ helm install --debug --dry-run mychart
(省略)

NAME:   intended-sabertooth
REVISION: 1
RELEASED: Tue Mar  1 21:36:47 2022
CHART: mychart-0.1.0
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
favoriteDrink: coffee

HOOKS:
MANIFEST:

---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: intended-sabertooth-configmap
data:
  myvalue: "Hello World"
  drink: coffee
```

ちゃんと coffee が出力されているのがわかる。  
ちなみに前に説明した --set フラグを使うこともでき、この場合は --set のほうが `values.yaml` よりも優先して変数を代入できる。

さらに `values.yaml` を修正し、階層構造にしてみた。
helm 公式の推奨はフラットな構成だが、このように変数を階層化させることもできる。

```yaml
favorite:
  drink: coffee
  food: pizza
```

configmap.yaml も以下のように変更。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink }}
  food: {{ .Values.favorite.food }}
```

階層構造になったのに従い、取得方法も{{ .Values.favorite.キー名 }}に変わった

デバッグ

```sh
$ helm install --debug --dry-run mychart
(省略)

NAME:   worn-skunk
REVISION: 1
RELEASED: Tue Mar  1 21:50:42 2022
CHART: mychart-0.1.0
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
favorite:
  drink: coffee
  food: pizza

HOOKS:
MANIFEST:

---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: worn-skunk-configmap
data:
  myvalue: "Hello World"
  drink: coffee
  food: pizza
```

## template Function と Pipeline

helm chart では template のファイルに function を使うことで、変数の値を加工することができる。
以下のように quote function を使ってみる。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ quote .Values.favorite.drink }}
  food: {{ quote .Values.favorite.food }}
```

{{ quote .Values.favorite.drink }} とすることで、.Values.favorite.drink に対して quote function が機能する。  
デバッグして出力を見てみる。

```sh
$ helm install --debug --dry-run mychart
(省略)
NAME:   juiced-zebra
REVISION: 1
RELEASED: Tue Mar  1 22:25:08 2022
CHART: mychart-0.1.0
(省略)

---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: juiced-zebra-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
```

ダブルクォーテーションが付いた。

helm には 60 以上の function がある。これは helm 特有のものではなく、Go template language や Spring temlate library の機能である。「Helm 実践ガイド」の巻末に「Chart 用 Spring Functions チートシート」があるため参考にするとよし。

### Pipelines

unix の pipeline と同じように、chart のも pipeline を使うことができる。  
pipeline をつなげることで、複数の機能を連鎖させることができる。

以下のように configmap.yaml を書き換えて、試してみる。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
```

前にやった quote function を pipeline で書き換えた例である。
さらに{{ .Values.favorite.food | upper | quote }}とすることで、function を 2 つ使っている。

これをデバッグして見ると、

```sh
$ helm install --debug --dry-run mychart
(省略)

---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: knotted-alpaca-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
```

{{ .Values.favorite.food | upper | quote }}としたことで、pizza が PIZZA となった

### default function

chart で default function を使うことでデフォルト値を設定できる。

以下のように configmap.yaml を書き換えて、試してみる。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
```

default function により drink に tea をデフォルト値で定義している。  
`values.yaml` から drink 部分をコメントアウトしてからデバッグしてみる。

```yaml
favorite:
  # drink: coffee
  food: pizza
```

デバッグ

```sh
$ helm install --debug --dry-run mychart
(省略)

---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: torrid-bison-configmap
data:
  myvalue: "Hello World"
  drink: "tea"
  food: "PIZZA"
```

このように default の値が drink の部分に入ってきた。
実際の chart では変数定義が `values.yaml` に集約されており、default function は繰り返し使用すべきない。
ただし、`values.yaml` で宣言できない変数に使うには最適である。

## フロー制御

プログラミング言語で条件分岐や繰り返し制御ができるようになんと chart でもフロー制御ができる。
helm では以下のフロー制御が使える。

- if / else による条件分岐
- with による範囲指定
- range によるループ

加えて、template 内で使用できる次のような機能もある

- define による変数定義
- template で定義した変数をインポート
- include による変数読み込み

### if / else

if / else により chart で条件分岐ができる。
たとえば PersistentVolume を使う場合は PV を宣言し、使わない場合は emptyDir を使う、などの条件分岐が実現できる。
