# Oracle Hangout Cafe Season 9 #6 開発者のためのFinOps

## ディレクトリ構成

```sh
.
├── README.md
├── cloud-integration.json -> OCI Integration OpenCost用設定ファイル
└── local.yaml　-> OpenCostイントール時パラメーター
```

## 前提条件

- 環境はOCI Kubernetes Engineを利用
- OCIリソースの認可はAPIキー利用
  - 事前にAPIキーを取得しておくこと
- helm3系をインストールしておくこと

## 環境構築

OKEのプロビジョニングは[こちら](https://oracle-japan.github.io/ocitutorials/cloud-native/oke-for-commons/)を参考に実施してください。
Worker NodeのスペックはVM.Standard.E5.Flexシェイプで2oCPU、16GB RAM × 3 Nodeで作成してください。

## OpenCostのインストール

# Prometheusのインストール

```sh
$ helm install prometheus --repo https://prometheus-community.github.io/helm-charts prometheus \
  --namespace prometheus-system --create-namespace \
  --set prometheus-pushgateway.enabled=false \
  --set alertmanager.enabled=false \
  -f https://raw.githubusercontent.com/opencost/opencost/develop/kubernetes/prometheus/extraScrapeConfigs.yaml
```

# Grafanaのインストール

```sh
$ helm repo add grafana https://grafana.github.io/helm-charts
$ helm install my-grafana grafana/grafana --namespace monitoring
```

# Namespaceの作成

```sh
$ kubectl create namespace opencost
```

# Cloud Costsを取得するためのクレデンシャル設定

`cloud-integration.json`の中身を環境に合わせ修正

```json
{
    "oci": {
      "usageApi": [
        {
          "tenancyID": "<テナンシOICD>",
          "region": "<ホームリージョン>",
          "authorizer": {
            "authorizerType": "OCIRawConfigProvider",
            "tenancyID": "<テナンシOICD>",
            "userID": "<ユーザOCID>",
            "region": "<ホームリージョン>",
            "fingerprint": "<APIキーのフィンガープリント>",
            "privateKey": "<APIキーの秘密鍵>"
          }
        }
      ]
    }
}
```

サンプル

```json
{
    "oci": {
      "usageApi": [
        {
          "tenancyID": "ocid1.tenancy.oc1..aaaaaaaa3mb7xxxxxxxxxxxxxxxxxxxzitpdaymbuazc5tkguca",
          "region": "us-ashburn-1",
          "authorizer": {
            "authorizerType": "OCIRawConfigProvider",
            "tenancyID": "ocid1.tenancy.oc1..aaaaxxxxxxxxxxxxxxx4mczxxxxxxxxxxxxxxxc5tkguca",
            "userID": "ocid1.user.oc1..aaaaaaaxxxxxxxxxxxxxxxhvpmxxxxxxxxxxxxxxxn47jeeyra",
            "region": "us-ashburn-1",
            "fingerprint": "71:01:81:7e:10:d6:30:4c:d1:db:06:d1:50:70:bb:03",
            "privateKey": "-----BEGIN PRIVATE KEY-----\nMIIEvQIBADANBgkqbledDSq+DGwEggk9xxxxxxxxxxxxxxx3RAEtTkt14\nX1HytGI6pWT/HXdMob9RC+w=\n-----END PRIVATE KEY-----"
          }
        }
      ]
    }
}
```

Kubernetesに登録

```sh
kubectl create secret generic cloud-costs --from-file=./cloud-integration.json --namespace opencost
```

# OpenCostのインストール

```sh
$ helm upgrade opencost --repo https://opencost.github.io/opencost-helm-chart opencost \
  --namespace opencost -f local.yaml
```

## 起動確認

```sh
$ kubectl get deploy,svc -A
NAMESPACE           NAME                                            READY   UP-TO-DATE   AVAILABLE   AGE
kube-system         deployment.apps/coredns                         3/3     3            3           15d
kube-system         deployment.apps/kube-dns-autoscaler             1/1     1            1           15d
monitoring          deployment.apps/my-grafana                      1/1     1            1           2d22h
opencost            deployment.apps/opencost                        1/1     1            1           13d
prometheus-system   deployment.apps/prometheus-kube-state-metrics   1/1     1            1           15d
prometheus-system   deployment.apps/prometheus-server               1/1     1            1           15d

NAMESPACE           NAME                                          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE
default             service/kubernetes                            ClusterIP   10.96.0.1       <none>        443/TCP,12250/TCP        15d
kube-system         service/kube-dns                              ClusterIP   10.96.5.5       <none>        53/UDP,53/TCP,9153/TCP   15d
monitoring          service/my-grafana                            ClusterIP   10.96.4.232     <none>        80/TCP                   2d22h
opencost            service/opencost                              ClusterIP   10.96.118.102   <none>        9003/TCP,9090/TCP        13d
prometheus-system   service/prometheus-kube-state-metrics         ClusterIP   10.96.171.253   <none>        8080/TCP                 15d
prometheus-system   service/prometheus-prometheus-node-exporter   ClusterIP   10.96.127.106   <none>        9100/TCP                 15d
prometheus-system   service/prometheus-server                     ClusterIP   10.96.209.234   <none>        80/TCP                   15d
```

## OpenCost UIでの確認

```sh
$ kubectl port-forward --namespace opencost service/opencost 9003 9090
```

`http://localhost:9090/`へのアクセス

![](images/image1.png)
![](images/image2.png)

Cloud CostsはUIで確認できるまで少し時間がかかる場合があります。

## Grafanaでの確認

# パスワード確認

```sh
$ kubectl get secret --namespace monitoring my-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
458JaxxxxxxxxxxxxxewYSeUTqg
```

# UI表示

```sh
 kubectl port-forward svc/my-grafana 8080:80 -n monitoring
```

`http://localhost:8080/`へのアクセス

# データソース設定 

`Connections` -> `Add new connection` 

![](images/image3.png)

Prometheusをデータソースをクリック

![](images/image4.png)

以下のように入力

- Name: `prometheus`
- Connection: `http://prometheus-server.prometheus-system`

あとはデフォルトのまま、一番下の`Save & test`をクリック

以下が表示されればOK

![](images/image5.png)

# Dashboard作成

`Dashboards` -> `New` -> `Import`をクリック

![](images/image6.png)
![](images/image7.png)

以下のように入力し、`Load`をクリック

![](images/image8.png)

以下のように入力し、`Import`をクリック

![](images/image9.png)

Dashboardが表示される

![](images/image10.png)
