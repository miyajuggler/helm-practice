# Prometheus と Grafana をデプロイ

それぞれ別の chart のため、インストールを 2 回に分ける

## 一応 Prometheus と Grafana について

Prometheus は、メトリクスに基づいた監視とアラートができるツールキット。  
ちなみにメトリクスとは、特定の対象を測定した数値のこと。(メモリの使用率とか)

Grafana は、データ可視化ツール(dashboard や グラフ化など)  
Prometheus の可視化ツールとしてよく使われている。

## grafana をインストールするための準備

こちらは values.yaml を用いて release 名や名前空間を指定してインストールしてみる。

```sh
# grafana の chart を取得する
$ helm fetch stable/grafana

# chart tar ボールを回答する
$ tar -zxf grafana-5.5.7.tgz

# grafana ディレクトリ確認
$ tree grafana
grafana
├── Chart.yaml
├── README.md
├── ci
│   ├── default-values.yaml
│   └── with-dashboard-json-values.yaml
├── dashboards
│   └── custom-dashboard.json
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── _pod.tpl
│   ├── clusterrole.yaml
│   ├── clusterrolebinding.yaml
│   ├── configmap-dashboard-provider.yaml
│   ├── configmap.yaml
│   ├── dashboards-json-configmap.yaml
│   ├── deployment.yaml
│   ├── headless-service.yaml
│   ├── ingress.yaml
│   ├── poddisruptionbudget.yaml
│   ├── podsecuritypolicy.yaml
│   ├── pvc.yaml
│   ├── role.yaml
│   ├── rolebinding.yaml
│   ├── secret-env.yaml
│   ├── secret.yaml
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   ├── statefulset.yaml
│   └── tests
│       ├── test-configmap.yaml
│       ├── test-podsecuritypolicy.yaml
│       ├── test-role.yaml
│       ├── test-rolebinding.yaml
│       ├── test-serviceaccount.yaml
│       └── test.yaml
└── values.yaml

4 directories, 33 files
```

### 変更点 1

values.yaml (service)

```yaml
  service:
(-) type: ClusterIP
(+) type: NodePort
    port: 80
    targetPort: 3000
        # targetPort: 4181 To be used with a proxy extraContainer
    annotations: {}
    labels: {}
    portName: service
```

kubernetes 上の Pod にアクセスするために方法を変更した。  
今回は docker desktop でオンプレ利用のため NodePort を選択した。

ちなみに ClusterIP と NodePort の違いについて

> ClusterIP は各ポッド間の通信で利用するものですが、NodeIP は Kubernetes クラスタ外からもアクセスが可能というわけです。  
> よって NodeIP は ClusterIP の機能は持ち合わせた上でさらに外部に公開用ポートを開いています。  
> 参考 [【Kubernetes 入門】ClusterIP と NodePort の違いは？](https://www.mtioutput.com/entry/kube-what-nodeport)

### 変更点 2

values.yaml (persistentvolume)

```yml
  persistence:
    type: pvc
(-) enabled: true
(+) enabled: true
    # storageClassName: default
    accessModes:
        - ReadWriteOnce
    size: 10Gi
```

### 変更点 3

values.yaml (datasource)

```yaml
datasources: {}
#  datasources.yaml:
#    apiVersion: 1
#    datasources:
#    - name: Prometheus
#      type: prometheus
#      url: http://prometheus-prometheus-server
#      access: proxy
#      isDefault: true
```

↓

```yaml
datasources:
  datasources.yaml:
  apiVersion: 1
  datasources:
  - name: Prometheus
      type: prometheus
      url: http://prometheus-prometheus-server
      access: proxy
      isDefault: true
```

datasource と呼ばれる監視ダッシュボードに表示するデータともを定義。  
今回は prometheus を立てるため、コメントアウトを外した。

### インストールする

今回は prometheus と grafana をインストールすつための名前空間「monitoring」を作成して、そこにインストールしてみる。

```sh
# 名前空間「monitoring」を作成
$ kubectl create ns monitoring
namespace/monitoring created

# 指定した名前で、指定した名前空間に prometheus をインストール
$ helm install --name prometheus --namespace monitoring stable/prometheus
WARNING: This chart is deprecated
NAME:   prometheus
LAST DEPLOYED: Sun Feb 27 19:35:28 2022
NAMESPACE: monitoring
STATUS: DEPLOYED

(省略)

NOTES:
DEPRECATED and moved to <https://github.com/prometheus-community/helm-charts>The Prometheus server can be accessed via port 80 on the following DNS name from within your cluster:
prometheus-server.monitoring.svc.cluster.local


Get the Prometheus server URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace monitoring -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace monitoring port-forward $POD_NAME 9090


The Prometheus alertmanager can be accessed via port 80 on the following DNS name from within your cluster:
prometheus-alertmanager.monitoring.svc.cluster.local


Get the Alertmanager URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace monitoring -l "app=prometheus,component=alertmanager" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace monitoring port-forward $POD_NAME 9093
#################################################################################
######   WARNING: Pod Security Policy has been moved to a global property.  #####
######            use .Values.podSecurityPolicy.enabled with pod-based      #####
######            annotations                                               #####
######            (e.g. .Values.nodeExporter.podSecurityPolicy.annotations) #####
#################################################################################


The Prometheus PushGateway can be accessed via port 9091 on the following DNS name from within your cluster:
prometheus-pushgateway.monitoring.svc.cluster.local


Get the PushGateway URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace monitoring -l "app=prometheus,component=pushgateway" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace monitoring port-forward $POD_NAME 9091

For more information on running Prometheus, visit:
https://prometheus.io/

# values.yaml を指定して、指定した名前で、指定した名前空間に grafana をインストール
$ helm install --name grafana --namespace monitoring -f grafana/values.yaml stable/grafana
WARNING: This chart is deprecated
NAME:   grafana
LAST DEPLOYED: Sun Feb 27 19:39:52 2022
NAMESPACE: monitoring
STATUS: DEPLOYED

(省略)

NOTES:
*******************
****DEPRECATED*****
*******************
* The chart is deprecated. Future development has been moved to https://github.com/grafana/helm2-grafana

1. Get your 'admin' user password by running:

   kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

2. The Grafana server can be accessed via port 80 on the following DNS name from within your cluster:

   grafana.monitoring.svc.cluster.local

   Get the Grafana URL to visit by running these commands in the same shell:
export NODE_PORT=$(kubectl get --namespace monitoring -o jsonpath="{.spec.ports[0].nodePort}" services grafana)
     export NODE_IP=$(kubectl get nodes --namespace monitoring -o jsonpath="{.items[0].status.addresses[0].address}")
     echo http://$NODE_IP:$NODE_PORT


3. Login with the password from step 1 and the username: admin
```

### ログインしてみる

NOTES を参考にしてログインしてみる。

1. admin ユーザーのパスワードを取得

   ```sh
   $ kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
   CV7SfpVTNw0hevsHjLr0Nk9W7w3nbv8hPR8xmL1s
   ```

2. grafana にアクセスするためのグローバル IP を取得

   ```sh
   $ export NODE_PORT=$(kubectl get --namespace monitoring -o jsonpath="{.spec.ports[0].nodePort}" services grafana)
   $ export NODE_IP=$(kubectl get nodes --namespace monitoring -o jsonpath="{.items[0].status.addresses[0].address}")
   $ echo http://$NODE_IP:$NODE_PORT
   http://192.168.65.4:32300
   ```

3. ブラウザで http://192.168.65.4:32300 を入力

何故かログインできない。

### 試してみたこと

1. ポートフォワードでつなげる

   色々調べると以下コマンドを使ってつなげていたのでやってみた。

   結果的に `localhost:3000` でつながったがなぜかログインができず止まってしまった。

   ```
   $ export POD_NAME=$(kubectl get pods --namespace monitoring -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" -o jsonpath="{.items[0].metadata.name}")

   $ kubectl --namespace monitoring port-forward $POD_NAME 3000
   ```

2. バージョンを参考書のやつに合わせた。

   特にうまく行かなかった

3. ingress でつなぐ？

   こちら未検証なのでやってみる。
   そもそも ingeress とは？

4. 設定をデフォルトでやってみる

5. 教科書のやり方はひとまず諦めて、ネットの記事のやり方を参考にしてみる (helm v2 であることをちゃんと確認する)

```yml
ingress:
  enabled: false
  # Values can be templated
  annotations:
    {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  labels: {}
  path: /
  hosts:
    - chart-example.local
  ## Extra paths to prepend to every host configuration. This is useful when working with annotation based services.
  extraPaths: []
  # - path: /*
  #   backend:
  #     serviceName: ssl-redirect
  #     servicePort: use-annotation
  tls: []
  #  - secretName: chart-example-tls
  #    hosts:
  #      - chart-example.local
```
