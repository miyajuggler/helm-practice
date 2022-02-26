# helm でアプリケーションをデプロイ

## Mysql をデプロイ

### mysql をインストール

コマンド練習の段階で `helm fetch stable/mysql` で chart はリポジトリからダウンロード済み

```sh
# 確認
$ ls | grep mysql
mysql-1.6.9.tgz
```

なので `helm install` コマンドで release をインストールする

```sh
$ helm install stable/mysql
WARNING: This chart is deprecated
NAME:   tailored-whippet
LAST DEPLOYED: Sat Feb 26 16:19:31 2022
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/ConfigMap
NAME                         DATA  AGE
tailored-whippet-mysql-test  1     0s

==> v1/Deployment
NAME                    READY  UP-TO-DATE  AVAILABLE  AGE
tailored-whippet-mysql  0/1    1           0          0s

==> v1/PersistentVolumeClaim
NAME                    STATUS  VOLUME                                    CAPACITY  ACCESS MODES  STORAGECLASS  AGE
tailored-whippet-mysql  Bound   pvc-83c82a97-11f0-43ae-a27c-31a2229d5a24  8Gi       RWO           hostpath      0s

==> v1/Pod(related)
NAME                                     READY  STATUS   RESTARTS  AGE
tailored-whippet-mysql-6fcc64fbb9-746p6  0/1    Pending  0         0s

==> v1/Secret
NAME                    TYPE    DATA  AGE
tailored-whippet-mysql  Opaque  2     0s

==> v1/Service
NAME                    TYPE       CLUSTER-IP    EXTERNAL-IP  PORT(S)   AGE
tailored-whippet-mysql  ClusterIP  10.97.228.93  <none>       3306/TCP  0s


NOTES:
MySQL can be accessed via port 3306 on the following DNS name from within your cluster:
tailored-whippet-mysql.default.svc.cluster.local

To get your root password run:

    MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace default tailored-whippet-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo)

To connect to your database:

1. Run an Ubuntu pod that you can use as a client:

    kubectl run -i --tty ubuntu --image=ubuntu:16.04 --restart=Never -- bash -il

2. Install the mysql client:

    $ apt-get update && apt-get install mysql-client -y

3. Connect using the mysql cli, then provide your password:
    $ mysql -h tailored-whippet-mysql -p

To connect to your database directly from outside the K8s cluster:
    MYSQL_HOST=127.0.0.1
    MYSQL_PORT=3306

    # Execute the following command to route the connection:
    kubectl port-forward svc/tailored-whippet-mysql 3306

    mysql -h ${MYSQL_HOST} -P${MYSQL_PORT} -u root -p${MYSQL_ROOT_PASSWORD}

```

### 注意

ここで気づいたが、pod が ImagePullBackOff となっていた。

describe して確認

```
$ kubectl describe po tailored-whippet-mysql-6fcc64fbb9-746p6

(省略)

Events:
  Type     Reason            Age                From               Message
  ----     ------            ----               ----               -------
  Warning  FailedScheduling  33s                default-scheduler  0/1 nodes are available: 1 pod has unbound immediate PersistentVolumeClaims.
  Normal   Scheduled         32s                default-scheduler  Successfully assigned default/existing-zebu-mysql-6bc4fbb69c-zss7z to docker-desktop
  Normal   Pulled            31s                kubelet            Container image "busybox:1.32" already present on machine
  Normal   Created           31s                kubelet            Created container remove-lost-found
  Normal   Started           31s                kubelet            Started container remove-lost-found
  Normal   BackOff           27s                kubelet            Back-off pulling image "mysql:5.7.30"
  Warning  Failed            27s                kubelet            Error: ImagePullBackOff
  Normal   Pulling           14s (x2 over 30s)  kubelet            Pulling image "mysql:5.7.30"
  Warning  Failed            11s (x2 over 27s)  kubelet            Failed to pull image "mysql:5.7.30": rpc error: code = Unknown desc = no matching manifest for linux/arm64/v8 in the manifest list entries
  Warning  Failed            11s (x2 over 27s)  kubelet            Error: ErrImagePull
```

どうやら image を性格にとってこれてない模様。  
いつぞやもあったが、MySQL の指定の仕方を amb64 側の Image で id 指定することで解決させた。

`sample-0.1.0.tgz` を解凍して、`deployment.yaml` と `values.yaml` を修正した。

解凍

```sh
$ tar -xzvf mysql-1.6.9.tgz
x mysql/Chart.yaml
x mysql/values.yaml
(省略)

# 確認
$ tree mysql
mysql
├── Chart.yaml
├── README.md
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── configurationFiles-configmap.yaml
│   ├── deployment.yaml
│   ├── initializationFiles-configmap.yaml
│   ├── pvc.yaml
│   ├── secrets.yaml
│   ├── serviceaccount.yaml
│   ├── servicemonitor.yaml
│   ├── svc.yaml
│   └── tests
│       ├── test-configmap.yaml
│       └── test.yaml
└── values.yaml

2 directories, 15 files
```

`deployment.yaml`

```yml
containers:
    - name: {{ template "mysql.fullname" . }}
       (-) image: "{{ .Values.image }}-{{ .Values.imageTag }}"
       (+) image: "{{ .Values.image }}@{{ .Values.imageTag }}"
```

`values.yaml`

```yml
    image: "mysql"
(-) imageTag: "5.7.30"
(+) imageTag: "sha256:5adbbb05d43e67a7ed5f4856d3831b22ece5178d23c565b31cef61f92e3467ea"

```

|![](image/mysql.png)
|:-:|

インストールして確認

```sh
$ helm install mysql/

# 確認
$ kubectl get po
NAME                                     READY   STATUS      RESTARTS   AGE
opinionated-bee-mysql-7897bc9576-55px5   1/1     Running     0          14m
```

### 参考

[M1 の kubernetes で詰まったところ調査](https://zenn.dev/ohkisuguru/scraps/5b9d76333ff13d)

### mysql にアクセス

NOTES に書かれている手順に従って、kubernetes クラスタ内から mysql にアクセスする。

1. パスワード取得

```sh
$ MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace default tailored-whippet-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo)

# 確認
$ echo $MYSQL_ROOT_PASSWORD
10WeUYo18Z
```

2. mysql にアクセスするために、Ubuntu イメージに mysql クライアントをインストールした pod を作成

```sh
# mysql にアクセスするための clientpod を作成
$ kubectl run -i --tty ubuntu --image=ubuntu:16.04 --restart=Never -- bash -il

# mysql クライアントをインストール
If you don't see a command prompt, try pressing enter.
root@ubuntu:/# apt-get update && apt-get install mysql-client -y
```

## 参考

[M1 の kubernetes で詰まったところ調査](https://zenn.dev/ohkisuguru/scraps/5b9d76333ff13d)
