# Locust on Google Kubernetes Engine(GKE)
Locust を活用した負荷分散テスト環境をGKE上で構築していきます。

# 参考サイト
Google Cloud の公式ドキュメントを参考にしつつ、独自にカスタマイズしています。

[Kubernetes を使用した負荷分散のテスト](https://cloud.google.com/solutions/distributed-load-testing-using-kubernetes)  
[Distributed Load Testing Using Kubernetes](https://github.com/GoogleCloudPlatform/distributed-load-testing-using-kubernetes)

# Google Cloud Platform(GCP)
GCPの前準備を行います。

## 利用サービス
- [Kubernetes Engine](https://cloud.google.com/kubernetes-engine/docs/concepts/container-engine-overview?hl=ja)
    - コンテナ化されたアプリケーションのデプロイ、管理、スケーリングを行うマネージド環境
- [Container Registry](https://cloud.google.com/container-registry/docs/overview?hl=ja)
    - GCPで実行される非公開のコンテナイメージレジストリ

## プロジェクトの作成
GCPで負荷テスト用のプロジェクト「locust-gke」を作成します。

## APIを有効化
初回は、APIが無効化になっているので有効化します。
- Kubernetes Engine
- Container Registry

# コンテナイメージの作成
Container Registry にアップロードするコンテナイメージを作成します。

## 作成するコンテナについて
Docker 関連ファイルは、`docker-image` 配下に格納しています。  

```
$ tree
.
├── README.md
├── docker-image ※コンテナイメージ作成用ディレクトリ
│   ├── Dockerfile
│   ├── Pipfile
│   ├── Pipfile.lock
│   └── locust
│       ├── run.sh
│       └── tasks.py
└── kubernetes-config
    ├── locust-master-controller.yaml
    ├── locust-master-controller.yaml.sample
    ├── locust-master-service.yaml
    ├── locust-master-service.yaml.sample
    ├── locust-worker-controller.yaml
    └── locust-worker-controller.yaml.sample
```

### docker-image/Dockerfile
使用する公式イメージは、`python:3.6-alpine` に変更しています。  
alpine を使用している為、パッケージ管理も `apk` に変更しています。

```
FROM python:3.6-alpine

RUN apk --no-cache add g++ linux-headers \
    && pip install locustio

EXPOSE 8089 5557 5558
ADD ./locust /locust
RUN chmod -R 755 /locust
ENTRYPOINT ["/locust/run.sh"]
```

alpine は、組み込み系でよく利用される BusyBox と musl をベースにした Linux ディストリビューションです。  
`python:3.6` の公式イメージが `924MB` に対し、`python:3.6-alpine` は、`79MB` と非常に軽量です。

```
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
python              3.6-alpine          54c461256011        11 days ago         79MB
python              3.6                 d6b15f660ce8        2 weeks ago         924MB
```

### docker-image/Pipfile
ローカル環境で locust を起動する為、Pipenv で環境を整えます。  
最低限のライブラリのみを追加しています。

```
[[source]]
name = "pypi"
url = "https://pypi.org/simple"
verify_ssl = true

[dev-packages]
flake8 = "*"
autopep8 = "*"
flake8-import-order = "*"

[packages]
locust = "*"

[requires]
python_version = "3.6"

[scripts]
lint = "flake8 --show-source ."
format = "autopep8 -ivr ."
```

### docker-image/locust/run.sh
alpine では、bash がサポートされておらず、ash(Almquist Shell) がサポートされています。  
その為、`bash` を `ash` に変更しています。

```
#!/bin/ash

LOCUST="/usr/local/bin/locust"
LOCUS_OPTS="-f /locust/tasks.py --host=$TARGET_HOST"
LOCUST_MODE=${LOCUST_MODE:-standalone}

if [[ "$LOCUST_MODE" = "master" ]]; then
    LOCUS_OPTS="$LOCUS_OPTS --master"
elif [[ "$LOCUST_MODE" = "worker" ]]; then
    LOCUS_OPTS="$LOCUS_OPTS --slave --master-host=$LOCUST_MASTER"
fi

echo "$LOCUST $LOCUS_OPTS"

$LOCUST $LOCUS_OPTS
```

### ローカル環境で locust のシナリオ作成
`locust/tasks.py` に負荷テストのシナリオを作成し、pipenv 経由で locust を起動します。

```
$ cd docker-image
$ pipenv install --dev
$ pipenv run locust -f locust/tasks.py -H http://target.lvh.me
```

下記URLにアクセスし、動作確認を行います。  
http://localhost.lvh.me:8089

### コンテナイメージのビルド
locust の負荷テストシナリオを内包したコンテナイメージをビルドします。  
レジストリ名は、`[HOSTNAME]/[PROJECT-ID]/[IMAGE]:[TAG]` のように指定します。  
詳細は、公式ドキュメントの「イメージの push と pull」、[ローカルイメージをレジストリ名にタグ付けする](https://cloud.google.com/container-registry/docs/pushing-and-pulling) をご参照ください。

```
$ docker build -t gcr.io/locust-gke/locust:latest docker-image/.
```
```
$ docker images
REPOSITORY                 TAG                 IMAGE ID            CREATED             SIZE
gcr.io/locust-gke/locust   latest              8a297c6382ac        34 minutes ago      330MB
```

### コンテナイメージの登録
Container Registry にコンテナイメージを登録する場合、Cloud SDK のインストールや gcloud の認証を行う必要があります。  
詳細は、公式ドキュメントの「イメージの push と pull」の [はじめに](https://cloud.google.com/container-registry/docs/pushing-and-pulling) をご参照ください。

```
$ docker push gcr.io/locust-gke/locust:latest
```

### コンテナイメージの削除
Container Registry からコンテナイメージを削除する場合、下記コマンドで削除出来ます。

```
$ gcloud container images delete gcr.io/locust-gke/locust:latest --force-delete-tags
```



# Kubernetes Engine
Kubernetes Engine の設定や構築を gcloud と kubectl で行います。  
kubectl の導入は、公式ドキュメントの [クイックスタート](https://cloud.google.com/kubernetes-engine/docs/quickstart) をご参照ください。

## デフォルトのプロジェクトの設定
```
$ gcloud config set project locust-gke
```

## デフォルトのコンピューティングゾーンの設定
```
$ gcloud config set compute/zone asia-northeast1-a
```

## 設定の確認
プロジェクトやゾーンが設定されていることを確認します。

```
$ gcloud config list
```

## Kubernetes クラスタの作成
クラスタ名「locust」という Kubernetes クラスタを作成します。  
ノード数を明示的に指定していますが、デフォルトが3なので指定しなくても問題ありません。

```
$ gcloud container clusters create locust --num-nodes=3
```

Kubernetes クラスタのリサイズは、下記コマンドで変更可能です。

```
$ gcloud container cluster resize locust --size 5
```

下記コマンドで Kubernetes クラスタ の削除が可能です。

```
$ gcloud container clusters delete locust
```

## Kubernetes クラスタの確認

```
$ kubectl get nodes
```

## locust-master のデプロイ
`kubernetes-config/locust-master-controller.yaml.sample` 内の括弧部分を書き換え、`locust-master` をデプロイします。

> [CONTAINER-IMAGE] => gcr.io/locust-gke/locust:latest  
> [TARGET_HOST] => http://target.lvh.me

```
$ kubectl create -f kubernetes-config/locust-master-controller.yaml
```

下記コマンドで locust-master の削除が可能です。

```
$ kubectl delete -f kubernetes-config/locust-master-controller.yaml
```

## locust-master の確認

```
$ kubectl get rc
$ kubectl get pods -l name=locust,role=master
```

## サービスの作成
ロードバランサタイプのサービスを作成します。

```
$ kubectl create -f kubernetes-config/locust-master-service.yaml
```

下記コマンドでサービスの削除が可能です。

```
$ kubectl delete -f kubernetes-config/locust-master-service.yaml
```

## サービスの確認

```
$ gcloud compute forwarding-rules list
```

## locust-worker のデプロイ
`kubernetes-config/locust-worker-controller.yaml.sample` 内の括弧部分を書き換え、`locust-worker` をデプロイします。

> [CONTAINER-IMAGE] => gcr.io/locust-gke/locust:latest  
> [TARGET_HOST] => http://target.lvh.me

```
$ kubectl create -f kubernetes-config/locust-c-controller.yaml
```

下記コマンドで locust-worker の削除が可能です。

```
$ kubectl delete -f kubernetes-config/locust-worker-controller.yaml
```

## locust-worker の確認

```
$ kubectl get pods -l name=locust,role=worker -o wide
```

locust-worker のスケールは、下記コマンドで変更可能です。

```
$ kubectl scale --replicas=4 replicationcontrollers locust-worker
```
