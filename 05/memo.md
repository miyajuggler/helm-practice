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


