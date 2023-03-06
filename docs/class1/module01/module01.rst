環境
#######

実施環境
====

-  事前にラボ環境へのInviteを行っておりますので、メールをご確認ください
-  利用するコマンド： git , jq , sudo, curl, make, kubectl(kubenetes環境)
-  NGINX Trialライセンスの取得、ラボ実施ユーザのHome Directoryへ配置

ラボ環境 (UDF(Unified Demonstration Framework)) コンポーネントへの接続
====

| 弊社が提供するLAB環境を使って動作を確認いただきます。
| ラボ環境を起動する等、一部ブラウザを使って操作します。
| Google ChromeがSupportブラウザとなります。その他ブラウザでは正しく動作しない場合があることご了承ください。
| 参照：\ `UDF Supported Browsers and Clients <https://help.udf.f5.com/en/articles/3470266-supported-browsers-and-clients>`__

ラボ環境構成図
----

   .. image:: ./media/udflab_k8s_Lab_jp.png
      :width: 400
      
Windows Jump HostへのRDP接続
----------------------------

Windows Jump HostからCLIの操作を行う場合、以下タブからRDP Clientファイルをダウンロードいただき接続ください

   .. image:: ./media/udf_jumpbox.png
      :width: 200

.. NOTE::
   | RDPのUser名、パスワードはDETAILSをクリックし、GeneralのタブのCredentialsの項目を参照ください
   | ``Administrator`` でログインしてください 

   - .. image:: ./media/udf_jumpbox_loginuser.png
       :width: 200
    
   - .. image:: ./media/udf_jumpbox_loginuser2.png
       :width: 200
   
Windows Jump Hostへログインいただくと、SSH
Clientのショートカットがありますので、そちらをダブルクリックし
``ubuntu-master`` を示すホストへ接続ください

   - .. image:: ./media/putty_icon.jpg
      :width: 50

   - .. image:: ./media/putty_menu_kic.jpg
      :width: 200

HELMについて
====

Helm とは
- Kubernetes用パッケージマネージャ
- Helmは、Kubernetes 用に構築されたソフトウェアを検索、共有、使用するための方法です。
- Kubernetes環境にソフトウェアを簡単にデプロイできます

   .. image:: ./media/helm-structure.jpg
      :width: 400

このラボでは、NGINX Ingress Controller(NIC)、NGINX Service Mesh(NSM)、各種監視コンポーネントをHelmを使ってデプロイします



デプロイする構成について
====

このラボでサンプルアプリケーションをデプロイした結果の構成は以下の様になります。

   .. image:: ./media/nginx-apigw-structure.png
      :width: 600

- Namespace ``nginx-ingress`` にNIC、 ``nginx-mesh`` にNSMのコンポーネント、 ``monitor`` に監視コンポーネントをデプロイします
- NSMのSidecarを挿入する対象のNamespaceとして ``prod`` 、 ``staging`` 、 ``legacy`` をデプロイします
- NSMの管理コンポーネントに接続するために ``nic2`` というNICをデプロイします
- NSMのSidecarを挿入するアプリケーションに接続するために ``nic1`` というNICをデプロイします
- GrafanaのDatasouceとして ``Prometheus`` 、 ``Loki`` 、 ``Jaeger`` を指定し、ステータスを確認できるようにします