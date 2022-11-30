NGINX NIC / NSM のセットアップ
####

1. 環境のセットアップ
====
NGINX Kubernetes Observability Labの以下マニュアルを参考に環境のセットアップを行ってください

- `NGINX NIC / NSM のセットアップ <https://f5j-nginx-k8s-observability.readthedocs.io/en/latest/class1/module02/module02.html>`__
- `監視コンポーネントのデプロイ <https://f5j-nginx-k8s-observability.readthedocs.io/en/latest/class1/module03/module03.html>`__
- `アプリケーションの外部公開・必要な設定のデプロイ <https://f5j-nginx-k8s-observability.readthedocs.io/en/latest/class1/module04/module04.html>`__ の内、以下の内容を実施してください

  - `1. NGINX Ingress Controllerの設定 <https://f5j-nginx-k8s-observability.readthedocs.io/en/latest/class1/module04/module04.html#nginx-ingress-controller>`__
  - `2. Grafana Datasouce の追加 <https://f5j-nginx-k8s-observability.readthedocs.io/en/latest/class1/module04/module04.html#grafana-datasouce>`__

2. サンプルアプリケーションのデプロイ
====

必要なファイルを取得します。

.. code-block:: cmdin
  
  cd ~/
  git clone https://github.com/BeF5/f5j-nginx-k8s-apigw-lab.git
  cd ~/f5j-nginx-k8s-apigw-lab/example

サンプルアプリケーションをデプロイします。

.. code-block:: cmdin

  ## cd ~/f5j-nginx-k8s-apigw-lab/example
  kubectl apply -f sample-app/target-v1.0.yaml -n staging
  kubectl apply -f sample-app/target-v2.0-successful.yaml -n staging
  kubectl apply -f sample-app/webapp-gw.yaml -n staging
  kubectl apply -f sample-app/target-svc.yaml -n staging

Podの状態を確認します

.. code-block:: cmdin

  kubectl get pod -n staging
  NAME                           READY   STATUS    RESTARTS   AGE
  target-v1-0-76d7d5c9bf-bpz7f   2/2     Running   0          24s
  target-v2-0-67f558f9b4-kbxzj   2/2     Running   0          21s
  webapp-5db45f6c57-ml5bn        2/2     Running   0          17s

対象のPodがデプロイされていることが確認できます。
また、NGINX Service Meshをデプロイしているため、READY欄の表示が ``2/2`` となっていることが確認できます

SVCの状態を確認します

.. code-block:: cmdin

  kubectl get svc -n staging
  NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
  target-svc    ClusterIP   10.105.127.166   <none>        80/TCP    2m
  target-v1-0   ClusterIP   10.99.222.248    <none>        80/TCP    116s
  target-v2-0   ClusterIP   10.104.185.195   <none>        80/TCP    113s
  webapp-svc    ClusterIP   10.108.203.113   <none>        80/TCP    109s


サンプルアプリケーションをデプロイすると以下のような構成となります。

   .. image:: ./media/nic-nsm-sample-app.png
      :width: 400

主に以下２つの経路に関する各種設定を確認いたします
- 赤色: NICから後段のアプリケーションに対する通信制御を確認する経路です
- 紫色: NICからNSMを経由し、アプリケーションへ転送するトラフィックの通信制御を確認する経路です
