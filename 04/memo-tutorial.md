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

```yaml
{{ if PIPELINE }}
  # Do something
{{ else if OTHER PIPELINE }}
  # Do something else
{{ else }}
  # Default case
```

↑ pipeline に対しての条件判定

false になる判定としては

- boolean 値で false
- 数字の 0
- 空文字
- null または nil
- からのコレクション

これ以外は true 判定となる。

実際に試してみる。

values.yaml

```yaml
favorite:
  drink: coffee
  food: pizza
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{ if eq .Values.favorite.drink "coffee" }}mug: true{{ end }}
```

values.yaml の favolite.drink が coffee の場合、mug:true が出力される。

デバッグしてみる。

```sh
$ helm install --debug --dry-run mychart
(省略)

---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: wistful-antelope-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
  mug: true
```

### ホワイトスペースの制御

helm で意外と悩まされる問題がホワイトスペースの扱い。
例えば上の yaml ファイルを読みやすい形式にすると、ホワイトスペースの問題に直面する。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{ if eq .Values.favorite.drink "coffee" }}
    mug: true
  {{ end }}
```

デバッグしてみるとエラーが発生する。
これはインデントをずらしたことによって余分なホワイトスペースがあるからだ。

```sh
$ helm install --debug --dry-run mychart
[debug] Created tunnel using local port: '50779'

[debug] SERVER: "127.0.0.1:50779"

[debug] Original chart version: ""
[debug] CHART PATH: /Users/miyazakinaohiro/github/helm-practice/04/mychart

Error: YAML parse error on mychart/templates/configmap.yaml: error converting YAML to JSON: yaml: line 9: did not find expected key
```

なので mug: true 部分のインデントを修正してみる。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{ if eq .Values.favorite.drink "coffee" }}
  mug: true
  {{ end }}
```

これを実行するとエラーはなくなるが以下のような余計な改行ができてしまう。

```sh
$ helm install --debug --dry-run mychart
(省略)

---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: morbid-anteater-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"

  mug: true
```

これはテンプレートエンジンが実行したとき、「{{ }}」のコンテンツが取り除かれても残りの空白がそのまま残るからである。  
この問題を解決するためのツールが helm には存在する。

「{{-」をつけることで左側の改行文字を取り除き、「-}}」をつけることで右側の改行文字を取り除くことができる。

以下のように書き換えてデバッグしてみる。

```yaml
(省略)
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{- if eq .Values.favorite.drink "coffee" }}
  mug: true
  {{ end }}
```

```sh
$ helm install --debug --dry-run mychart                                    ○ docker-desktop
(省略)

---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nonexistent-anaconda-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
  mug: true
```

このように左側の改行が取り除かれて上のような出力結果となる

ちなみに以下のように誤ると改行が取り除かれすぎてしまう。

```yaml
(省略)
  food: {{ .Values.favorite.food | upper | quote }}
  {{- if eq .Values.favorite.drink "coffee" -}}
  mug: true
  {{ end }}
```

```sh
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"mug: true
```

### with

with を使うことで、変数のスコープができる。これまで出てきた「.」これはカレントスコープを示しており、「.Values」はカレントスコープの「Values」オブジェクトにアクセスしている。

どういうことかというと以下の例を参考にしたら良い。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{ end }}
```

「.Values.favolite」を with で宣言している。
これにより「.drink」と呼び出す階層が浅くなった。
「{{ with }}」から「{{ end }}」までの変数のスコープが変更になったからである。

### range

プログラミングで言う for 文である。

values.yaml を以下のように書き換える。

```yaml
favorite:
  drink: coffee
  food: pizza
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions
```

そして configmap.yaml を以下のように書き換える。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{- end }}
  toppings: |-
    {{- range .Values.pizzaToppings }}
    - {{ . | title | quote }}
    {{- end }}
```

「range .Values.pizzaToppings」で pizzaToppings リストの個数分、処理が繰り返される。  
range の次の行で「. | title | quote」とあるがこの「.」に 1 回目のループで mushrooms 2 回目のループで cheese 、、、といった感じでセットされる。
「title | quote」で先頭を大文字にして、ダブルクォーテーションがつく。

また、tuple function を使って簡単にリストを反復処理できる。
以下のような内容に書き換えてみる。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{- end }}
  size: |-
  {{- range tuple "small" "medium" "large" }}
  - {{ . }}
  {{- end }}
```

すると、出力は以下のようになる。

```yaml
(省略)

size: |-
  - small
  - medium
  - large

```

### valiables

変数を template 内で代入する仕組みがある。
「$変数名」と「:=」を利用することで、template 内で変数を代入できる。

以下のような configmap.yaml ファイルを用意する。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- $relname := .Release.Name }}
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  release: {{ $relname }}
  {{- end }}
```

「$relname := .Release.Name」で relname という変数に 「.Release.Name」 の値を代入している。  
そして「release: {{ $relname }}」のところで relname に代入された値を使っている。

デバッグ

```sh
$ helm install --debug --dry-run mychart
(省略)

NAME:   juiced-buffalo
(省略)

---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: juiced-buffalo-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
  release: juiced-buffalo
```

この変数代入は range と組み合わせると便利である。
以下にその例を乗せる。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{- end }}
  toppings: |-
    {{- range $index, $topping := .Values.pizzaToppings }}
    - {{ $index }}: {{ $topping }}
    {{- end }}
```

「$index」は予約語で、0 から始まりループを繰り返すごとにひとつずつ数が増えていく

デバッグ

```sh
$ helm install --debug --dry-run mychart
(省略)
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: wandering-wasp-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
  toppings: |-
    - 0: mushrooms
    - 1: cheese
    - 2: peppers
    - 3: onions
```

インデックスと値がリストの個数分、出力される。また、range と変数代入を組み合わせることで、key と value の両方を取得できる。

以下のように configmap.yaml ファイルを用意する。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
```

「range $key, $val := .Values.favorite」で「.Values.favorite」の key と value を取得する。
1 回目のループで「drink: "coffee"」、2 回目のループで「food: "pizza"」が取得できる。

デバッグ

```sh
$ helm install --debug --dry-run mychart
(省略)
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: unhinged-lynx-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
```

## Named Template

これまでのはひとつの template の中で使うものだった。これから見ていくのは複数の template ファイルで宣言・利用ができる変数である。
define、template、block の 3 つのことであり、これらは Named Template と呼ばれている。

重要なことは template name はグルーバルスコープであるということだ。もし同じ名前の 2 つの template を宣言した場合、後勝ちで最後に良いこまれたほうで上書きされる。

また、サブ chart 内の template も親の chart と一緒にコンパイルされるため、template には chart 独自の名前をつけて、重複しないようにするのが良い。

### helper files

まず `_` というプレフィックスの付いたファイルについて説明する。

template ファイルには以下の規約がある。

- templates/配下のファイルは kubernetes マニフェストとして扱われる。
- NOTES.txt は例外である（ただ、NOTES 本文に template や pipeline を利用可能）
- `_` で始まるファイルはマニフェストとして扱われない。

`_` で始まるファイルマニフェストとして扱われないが、chart の template のどこからでも利用できる。（例「\_helpers.tpl」というファイルのようにヘルパーとしてグローバルな値を定義したいとき。）  
chart 全体で利用するグローバルな値を定義していることが多い。

### defile / template アクション

この 2 つはセットで利用し、define で変数定義、template で変数呼び出し、という使い方をする。
他にも変数を定義し、その変数を利用する方法はこれまでで説明してきたが、define と template はグローバルに使えるため、chart 全体で利用できる。

```yaml
{{- define "mychart.labels" }}
labels:
  generator: helm
  date: {{ now | htmlDate }}
{{- end }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.labels" }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
```

上記のような configmap.yaml ファイルを用意する。
define と end の間に kubernetes オブジェクトで利用する labels ブロックを定義し、metadata に template アクションを挿入。

テンプレートエンジンが読み込むと「define "mychart.labels"」を参照し、「template "mychart.labels"」の箇所に上書きされる。

デバッグ

```sh
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: wandering-moose-configmap
labels:
  generator: helm
  date: 2022-03-04
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
```

labels のブロックが出力されていることが確認できた。

define のようなグローバルな値は `_helpers.tpl` のようなヘルパーファイルで宣言をする。  
実際に作ってやってみる。

`mychart/templates/_helpers.tpl`

```yaml
{{/* Generate basic labels */}}
{{- define "mychart.labels" }}
labels:
  generator: helm
  date: {{ now | htmlDate }}
{{- end }}
```

上記のように define の宣言場所は {{/* ... */}} とくくるのが定番

configmap.yaml から define 部分を取り除く

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.labels" }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
```

define が `_helpers.tpl` に移植したが、変わらず変数定義にアクセスでき、出力結果が変わらない

```sh
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: idolized-hydra-configmap
labels:
  generator: helm
  date: 2022-03-04
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
```

### template アクションの範囲

define と template アクションを扱う上で、変数のスコープについて気をつける必要がある。

以下の例のように define の中で .Chart.Name と .Chart.Version が登場している。

```yaml
{{/* Generate basic labels */}}
{{- define "mychart.labels" }}
labels:
  generator: helm
  date: {{ now | htmlDate }}
  chart: {{ .Chart.Name }}
  version: {{ .Chart.Version }}
{{- end }}
```

出力すると以下のようになる。

```sh
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: agile-jackal-configmap
labels:
  generator: helm
  date: 2022-03-04
  chart:
  version:
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
```

chart と version が空白で出力された。これは、{{- template "mychart.labels" }} で展開されたときに、「.」 にスコープが含まれていないからである。

なので{{- template "mychart.labels" . }} のように含めてあげることで「.」 にスコープが通るようになる。

以下のように configmap.yaml を編集する

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.labels" . }}
(省略)
```

デバッグ

```sh
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nobby-dragon-configmap
labels:
  generator: helm
  date: 2022-03-04
  chart: mychart
  version: 0.1.0
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
```

このように chart と version に値が入った。
「.」 にスコープが通っているので .Chart 以外にも .Values も利用可能

試す

- .Values.favorite.drink を追加しても通る
- configmap.yaml の「.」 を 「.Chart」 のように変えて、helpers.tql の方で{{ .Version }}と変えても通る

### include function

include function は template アクション と同じように define で定義した値を読み込む機能である。
template との使い分けを以下から見ていく。

`_helpers.tpl`

```yaml
{{/* Generate basic labels */}}
{{- define "mychart.app" -}}
app_name: {{ .Chart.Name }}
app_version: "{{ .Chart.Version }}+{{ .Release.Time.Seconds }}""
{{- end }}
```

このように app_name と app_version を定義

`configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  labels:
    {{ template "mychart.app" . }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
{{ template "mychart.app" . }}
```

labels ブロックと data ブロックに template アクションを挿入する。2 箇所のインデントが違うことに着目

デバッグ

```sh
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kissing-cricket-configmap
  labels:
    app_name: mychart
app_version: "0.1.0+1646402558"
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
app_name: mychart
app_version: "0.1.0+1646402558"
```

labels の app_name 以外のインデントが左揃えになってずれていることがわかる。代入された template がテキストで左揃えになるからこうなってしまう。  
template はアクションであって関数ではないので、出力結果を他の関数に渡すことができない。単純にインラインで一行が挿入される。

こうした場合を回避するために include function がある

インデントを修正するために include とあわせて nindent function を使う。

以下のように修正する。nindent を使うと指定した数字分インデントが調整できる。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  labels:
    {{- include "mychart.app" . | nindent 4 }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
{{- include "mychart.app" . | nindent 2 }}
```

デバッグ

```sh
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rousing-manatee-configmap
  labels:
    app_name: mychart
    app_version: "0.1.0+1646402921"
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
  app_name: mychart
  app_version: "0.1.0+1646402921"
```

インデントが揃った。
公式ドキュメントによれば、yaml のフォーマットに合わせることができるので、template よりも include を推奨している。
