APIGW通信制御の実施
####

1. NICのトラフィック分割(Canary Release)
====

以下の構成で動作を確認します

   .. image:: ./media/nic-traffic-split.png
      :width: 400

NICの背後にあるアプリケーションのトラフィック分割が求められる状況を想定した構成となります。


1. 設定のデプロイ (割合9:1)
----

設定の内容を確認します

.. code-block:: cmdin

  ## cd ~/f5j-nginx-k8s-apigw-lab/example
  cat split-nic-vs/nic-vs-split1.yaml

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 8-9,11-12,17-22

  apiVersion: k8s.nginx.org/v1
  kind: VirtualServer
  metadata:
    name: nic
  spec:
    host: nic.example.com
    upstreams:
    - name: target-v1
      service: target-v1-0
      port: 80
    - name: target-v2
      service: target-v2-0
      port: 80
    routes:
    - path: /
      splits:
      - weight: 90
        action:
          pass: target-v1
      - weight: 10
        action:
          pass: target-v2

- Upstreamとして 8-9行目 ``target-v1-0`` を、 11-12行目 ``target-v2-0`` を転送先サービスとして指定しています
- 17-19行目で、 ``target-v1-0`` に対し ``wegiht 90(90%)`` 、20-22行目で、 ``target-v2-0`` に対し ``wegiht 10(10%)`` の割合を指定して転送します

設定をデプロイします

.. code-block:: cmdin

  ## cd ~/f5j-nginx-k8s-apigw-lab/example

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  kubectl apply -f split-nic-vs/nic-vs-split1.yaml -n staging

正しく反映されたことを確認します

.. code-block:: cmdin

  kubectl get vs nic -n staging

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  NAME   STATE   HOST              IP    PORTS   AGE
  nic    Valid   nic.example.com                 49s

STATEが ``Valid`` であることを確認します


2. 動作確認 (割合9:1)
----

正しく疎通があることを確認します

.. code-block:: cmdin

  curl -s -H "Host: nic.example.com" http://localhost/

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  target v1.0

複数回実行いただくと ``target v1.0`` または ``target v2.0`` が応答され、2種類のVersionのアプリケーションから応答されている状況が確認いただけます。

以下のコマンドで ``20回`` リクエストを送付します。結果を確認します

.. code-block:: cmdin

  for i in {1..20}; do curl -s -H "Host: nic.example.com" http://localhost/  ; done ;

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  target v1.0
  target v1.0
  target v1.0
  target v1.0
  target v2.0
  target v1.0
  target v1.0
  target v1.0
  target v2.0
  target v1.0
  target v1.0
  target v1.0
  target v1.0
  target v1.0
  target v1.0
  target v1.0
  target v1.0
  target v1.0
  target v1.0
  target v1.0

``v1`` と ``v2`` が指定の値に近い割合で応答が返答されていることが確認できます。


3. 設定のデプロイ (割合2:8)
----


設定の内容を確認します

.. code-block:: cmdin

  ## cd ~/f5j-nginx-k8s-apigw-lab/example
  cat split-nic-vs/nic-vs-split2.yaml

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 8-9,11-12,17-22

  apiVersion: k8s.nginx.org/v1
  kind: VirtualServer
  metadata:
    name: nic
  spec:
    host: nic.example.com
    upstreams:
    - name: target-v1
      service: target-v1-0
      port: 80
    - name: target-v2
      service: target-v2-0
      port: 80
    routes:
    - path: /
      splits:
      - weight: 20
        action:
          pass: target-v1
      - weight: 80
        action:
          pass: target-v2

- Upstreamは変更ありません
- 17-19行目で、 ``target-v1-0`` に対し ``wegiht 20(20%)`` 、20-22行目で、 ``target-v2-0`` に対し ``wegiht 80(80%)`` の割合を指定して転送します。V2の安定した動作が確認できたため割合を増加する想定のシナリオとなります

設定をデプロイします

.. code-block:: cmdin

  ## cd ~/f5j-nginx-k8s-apigw-lab/example

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  kubectl apply -f split-nic-vs/nic-vs-split2.yaml -n staging

正しく反映されたことを確認します

.. code-block:: cmdin

  kubectl get vs nic -n staging

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  NAME   STATE   HOST              IP    PORTS   AGE
  nic    Valid   nic.example.com                 49s

STATEが ``Valid`` であることを確認します

4. 動作確認 (割合2:8)
----

以下のコマンドで ``20回`` リクエストを送付します。結果を確認します

.. code-block:: cmdin

  for i in {1..20}; do curl -s -H "Host: nic.example.com" http://localhost/  ; done ;

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  target v2.0
  target v2.0
  target v1.0
  target v2.0
  target v2.0
  target v2.0
  target v2.0
  target v2.0
  target v2.0
  target v2.0
  target v2.0
  target v1.0
  target v2.0
  target v2.0
  target v2.0
  target v2.0
  target v2.0
  target v1.0
  target v1.0
  target v2.0


先程の内容から割合を変更したため、 ``v2`` が多くなっています。
``v1`` と ``v2`` が指定の値に近い割合で応答が返答されていることが確認できます。


5. 不要設定の削除
----

不要な設定を削除します

.. code-block:: cmdin

  ## cd ~/f5j-nginx-k8s-apigw-lab/example
  kubectl delete -f split-nic-vs/nic-vs-split2.yaml -n staging


2. NSMのトラフィック分割(Canary Release)
====

以下の構成で動作を確認します

   .. image:: ./media/nsm-traffic-split.png
      :width: 400

NICでの制御と異なり、NICの背後のアプリケーションは単一です。
そのアプリケーションの背後にあるアプリケーションのトラフィック分割が求められる状況を想定した構成となります。

1. NIC設定のデプロイ
----

設定の内容を確認します

.. code-block:: cmdin

  ## cd ~/f5j-nginx-k8s-apigw-lab/example
  cat split-nsm-smi/nic-vs-nsmsplit.yaml

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  apiVersion: k8s.nginx.org/v1
  kind: VirtualServer
  metadata:
    name: webapp
  spec:
    host: webapp.example.com
    upstreams:
    - name: webapp-svc
      service: webapp-svc
      port: 80
    routes:
    - path: /
      action:

NICの設定内容は大変シンプルで、後段の ``webapp-svc`` へ転送する構成となります

設定をデプロイします

.. code-block:: cmdin

  ## cd ~/f5j-nginx-k8s-apigw-lab/example

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  kubectl apply -f split-nsm-smi/nic-vs-nsmsplit.yaml -n staging

正しく反映されたことを確認します

.. code-block:: cmdin

  kubectl get vs webapp -n staging

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  NAME     STATE   HOST                 IP    PORTS   AGE
  webapp   Valid   webapp.example.com                 25s

STATEが ``Valid`` であることを確認します

2. 動作確認
----

正しく疎通があることを確認します

.. code-block:: cmdin

  curl -s -H "Host: webapp.example.com" http://localhost/

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  target v1.0

複数回実行すると ``target v1.0`` 、 ``target v2.0`` が交互に応答されることが確認できます

以下のコマンドで ``6回`` リクエストを送付します。結果を確認します

.. code-block:: cmdin

  for i in {1..6}; do curl -s -H "Host: webapp.example.com" http://localhost/  ; done ;

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  target v1.0
  target v2.0
  target v1.0
  target v2.0
  target v1.0
  target v2.0

``v1`` と ``v2`` が交互に応答されていることがわかります。これは ``webapp-svc`` が、 ``target-svc`` に通信を転送した結果となります。

3. NSM設定のデプロイ (割合2:8)
----

NSMを使い ``target-svc`` から、 ``target-v1-0 `` 、 ``target-v2-0`` に対する通信を対象に割合の指定を行います

.. code-block:: cmdin

  ## cd ~/f5j-nginx-k8s-apigw-lab/example
  cat split-nsm-smi/nsm-split1.yaml

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 8-11

  apiVersion: split.smi-spec.io/v1alpha3
  kind: TrafficSplit
  metadata:
    name: target-ts
  spec:
    service: target-svc
    backends:
    - service: target-v1-0
      weight: 90
    - service: target-v2-0
      weight: 10

- 8-9行目で、 ``target-v1-0`` に対し ``wegiht 90(90%)`` 、10-11行目で、 ``target-v2-0`` に対し ``wegiht 10(10%)`` の割合を指定して転送します。

設定をデプロイします

.. code-block:: cmdin

  ## cd ~/f5j-nginx-k8s-apigw-lab/example
  kubectl apply -f split-nsm-smi/nsm-split1.yaml -n staging

正しく反映されたことを確認します

.. code-block:: cmdin

  kubectl get trafficsplit -n staging

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  NAME        AGE
  target-ts   19s


4. NSM設定の動作確認 (割合2:8)
----

以下のコマンドで ``20回`` リクエストを送付します。結果を確認します

.. code-block:: cmdin

  for i in {1..20}; do curl -s -H "Host: webapp.example.com" http://localhost/  ; done ;

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  target v1.0
  target v1.0
  target v1.0
  target v1.0
  target v1.0
  target v2.0
  target v1.0
  target v1.0
  target v1.0
  target v1.0
  target v1.0
  target v1.0
  target v1.0
  target v1.0
  target v1.0
  target v1.0
  target v1.0
  target v1.0
  target v1.0
  target v2.0

先程の設定では、均等(5:5)に分散されていた通信ですが、
``v1`` と ``v2`` が指定の値に近い割合で応答が返答されていることが確認できます。

5. 不要設定の削除
----

不要な設定を削除します

.. code-block:: cmdin

  ## cd ~/f5j-nginx-k8s-apigw-lab/example
  kubectl delete -f split-nsm-smi/nic-vs-nsmsplit.yaml -n staging
  kubectl delete -f split-nsm-smi/nsm-split1.yaml -n staging


3. NIC/NSMのJWT制御
====

以下の構成で動作を確認します

   .. image:: ./media/nic-nsm-jwt.png
      :width: 400

- NICでクライアントのJWTの制御を行います。
- 適切なJWTである場合、JWTの情報をHTTPヘッダーに情報を付与します
- 付与されたHTTPヘッダーの情報を元にNSMで通信の制御を行います。この例では割合を指定し ``v2`` に通信を転送します

1. 設定のデプロイ
----

利用するJWT Policyは
`Ingress Controller で JWT Validation のデプロイ <https://f5j-nginx-ingress-controller-lab1.readthedocs.io/en/latest/class1/module3/module3.html#ingress-controller-jwt-validation>`__
を利用しています。
``jwk-secret.yaml`` 、 ``jwt.yaml`` の解説はこちらを参照ください

その他の設定の内容を確認します

.. code-block:: cmdin

  ## cd ~/f5j-nginx-k8s-apigw-lab/example
  cat jwt-nic-nsm/nic-vs-jwt-addheader.yaml

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 7-8,16-22

  apiVersion: k8s.nginx.org/v1
  kind: VirtualServer
  metadata:
    name: webapp
  spec:
    host: webapp.example.com
    policies:
    - name: jwt-policy
    upstreams:
    - name: webapp-svc
      service: webapp-svc
      port: 80
    routes:
    - path: /
      action:
        proxy:
          upstream: webapp-svc
          requestHeaders:
            pass: true
            set:
            - name: jwtscope
              value: ${jwt_claim_scope}

- 7-8行目で、 ``webapp.example.com`` 宛の通信に対してJWT Validationを設定しています
- 16-22行目で、有効なJWTに指定された ``Scope`` の情報を、HTTPリクエストの ``jwtscope`` というHTTPヘッダーに付与する設定をします

次にNSMの設定を確認します

.. code-block:: cmdin

  ## cd ~/f5j-nginx-k8s-apigw-lab/example
  cat jwt-nic-nsm/nsm-split-jwt.yaml

基本的な設定は ``TrafficSplit`` です。

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 12-14,19,21-24

  apiVersion: split.smi-spec.io/v1alpha3
  kind: TrafficSplit
  metadata:
    name: target-ts
  spec:
    service: target-svc
    backends:
    - service: target-v1-0
      weight: 0
    - service: target-v2-0
      weight: 100
    matches:
    - kind: HTTPRouteGroup
      name: target-scope
  ---
  apiVersion: specs.smi-spec.io/v1alpha3
  kind: HTTPRouteGroup
  metadata:
    name: target-scope
  spec:
    matches:
    - name: jwt-group2-users
      headers:
        jwtscope: ".*group2.*"

- TrafficSplitは、 ``target-v2-0`` にすべての通信を転送する内容となります。ただし、 12-14行目に指定の通り条件を付与しています
- 条件が16行目から記述されており、19行目の ``target-scope`` が 14行目に指定されていることがわかります
- 条件の内容は21-24行目となり、 ``jwtscope`` というHTTPヘッダーに ``group2`` という文字列が含まれている場合該当する、という内容を指定しています。

設定を反映します

.. code-block:: cmdin

  ## cd ~/f5j-nginx-k8s-apigw-lab/example
  kubectl apply -f jwt-nic-nsm/jwk-secret.yaml -n staging
  kubectl apply -f jwt-nic-nsm/jwt.yaml -n staging
  kubectl apply -f jwt-nic-nsm/nic-vs-jwt-addheader.yaml -n staging
  kubectl apply -f jwt-nic-nsm/nsm-split-jwt.yaml -n staging

2. 動作確認
----

JWT Validationの動作を確認します

.. code-block:: cmdin

  curl -s -H "Host: webapp.example.com" http://localhost/

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  <html>
  <head><title>401 Authorization Required</title></head>
  <body>
  <center><h1>401 Authorization Required</h1></center>
  <hr><center>nginx/1.21.6</center>
  </body>
  </html>

``401 Authorization Required`` が応答されていることが確認できます

次に適切なJWTをJWT Policyで指定した通り ``Cookie`` の ``Token`` に指定して通信を行います

.. code-block:: cmdin

  ## cd ~/f5j-nginx-k8s-apigw-lab/example
  curl -s -H "Host: webapp.example.com" http://localhost/ -H "Token: `cat jwt-nic-nsm/nginx1.jwt`"

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  target v1.0

エラーなく応答が確認できました

複数回のリクエストを実行します。予め用意したJWTの ``nginx1.jwt`` と ``nginx3.jwt`` の動作の違いを確認します

.. code-block:: cmdin

  ## cd ~/f5j-nginx-k8s-apigw-lab/example
  for i in {1..4}; do curl -s -H "Host: webapp.example.com" http://localhost/ -H "Token: `cat jwt-nic-nsm/nginx1.jwt`" ;done;

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  target v1.0
  target v2.0
  target v1.0
  target v2.0

.. code-block:: cmdin

  ## cd ~/f5j-nginx-k8s-apigw-lab/example
  for i in {1..4}; do curl -s -H "Host: webapp.example.com" http://localhost/ -H "Token: `cat jwt-nic-nsm/nginx3.jwt`" ;done;

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  target v2.0
  target v2.0
  target v2.0
  target v2.0

``nginx1.jwt`` を指定した場合には、通信が均等に分散されていることが確認できます。
``nginx3.jwt`` は ``v2.0`` のみに通信が転送されていることが確認できます。これは、NIC / NSMで指定したポリシーが正しく動作していることを示します

この条件の設定を組み合わせることで、詳細な条件をKubernetes内部の通信に適用することが可能となります。

3. 不要設定の削除
----

不要な設定を削除します

.. code-block:: cmdin

  ## cd ~/f5j-nginx-k8s-apigw-lab/example
  kubectl delete -f jwt-nic-nsm/jwk-secret.yaml -n staging
  kubectl delete -f jwt-nic-nsm/jwt.yaml -n staging
  kubectl delete -f jwt-nic-nsm/nic-vs-jwt-addheader.yaml -n staging
  kubectl delete -f jwt-nic-nsm/nsm-split-jwt.yaml -n staging


4. JWT制御の対象をWAFで防御する
====

以下の構成で動作を確認します

   .. image:: ./media/nic-jwt-waf.png
      :width: 400

JWTによる通信制御はAPIを保護する有効な手段ですが、正しい認証情報を持ったクライアントが悪意あるプログラムなどにより想定外の動作を行うなどの場合が考えられます。
このような状況を想定して悪意ある通信を防御する方法を確認します。

1. 設定のデプロイ
----

設定の内容を確認します。
JWTに関する設定は 
`3. NIC/NSMのJWT制御 <>`__ 
と同様に
`Ingress Controller で JWT Validation のデプロイ <https://f5j-nginx-ingress-controller-lab1.readthedocs.io/en/latest/class1/module3/module3.html#ingress-controller-jwt-validation>`__
を利用しています。

WAFの設定は最低限の設定を行い、外部からの攻撃をブロックできる設定としています。

.. code-block:: cmdin

  ## cd ~/f5j-nginx-k8s-apigw-lab/example
  cat waf-nic-vs/simple-ap.yaml

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 8

  apiVersion: appprotect.f5.com/v1beta1
  kind: APPolicy
  metadata:
    name: simple-ap
  spec:
    policy:
      name: simple-ap
      applicationLanguage: utf-8
      enforcementMode: blocking
      template:
        name: POLICY_TEMPLATE_NGINX_BASE

``enforcementMode`` で ``blocking`` と指定することでWAFの通信を防御します
WAFは数多くの設定により悪意ある通信をブロックすることが可能です。詳細を確認する場合、以下のページを参照してください。

- `NIC: Using NAP WAF Configuration <https://docs.nginx.com/nginx-ingress-controller/app-protect-waf/configuration/>`__
- `NAP WAF: Configuration Guide <https://docs.nginx.com/nginx-app-protect/configuration-guide/configuration/>`__

.. code-block:: cmdin

  ## cd ~/f5j-nginx-k8s-apigw-lab/example
  cat waf-nic-vs/nic-vs-waf-jwt.yaml

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 8-9

  apiVersion: k8s.nginx.org/v1
  kind: VirtualServer
  metadata:
    name: nic
  spec:
    host: nic.example.com
    policies:
    - name: waf-policy
    - name: jwt-policy
    upstreams:
    - name: target-v1
      service: target-v1-0
      port: 80
    - name: target-v2
      service: target-v2-0
      port: 80
    routes:
    - path: /
      splits:
      - weight: 50
        action:
          pass: target-v1
      - weight: 50
        action:
          pass: target-v2

- 8行目に ``WAF`` 、 9行目に ``JWT`` のポリシーを割り当てています。この様に設定することで ``nic.example.com`` に対する通信に対しJWT Validation及びWAFの防御が可能になります
- Policyは ``Host`` 、 ``Route`` など柔軟に指定することが可能です。詳細は `NIC: VS/VSR <https://docs.nginx.com/nginx-ingress-controller/configuration/virtualserver-and-virtualserverroute-resources/>`__ の ``policies`` Fieldを参照ください


設定を反映します

.. code-block:: cmdin

  ## cd ~/f5j-nginx-k8s-apigw-lab/example
  kubectl apply -f jwt-nic-nsm/jwk-secret.yaml -n staging
  kubectl apply -f jwt-nic-nsm/jwt.yaml -n staging
  kubectl apply -f waf-nic-vs/ap-logconf.yaml -n staging
  kubectl apply -f waf-nic-vs/simple-ap.yaml -n staging
  kubectl apply -f waf-nic-vs/waf.yaml -n staging
  kubectl apply -f waf-nic-vs/nic-vs-waf-jwt.yaml -n staging

.. code-block:: cmdin

  kubectl get aplogconf,appolicy,policy -n staging

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  NAME                                  AGE
  aplogconf.appprotect.f5.com/logconf   21s
  
  NAME                                   AGE
  appolicy.appprotect.f5.com/simple-ap   21s
  
  NAME                              STATE   AGE
  policy.k8s.nginx.org/jwt-policy   Valid   22s
  policy.k8s.nginx.org/waf-policy   Valid   20s


2. 動作確認
----

対象のFQDNにJWTを指定せず、動作確認します

.. code-block:: cmdin

  curl -s -H "Host: nic.example.com" http://localhost/

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  <html>
  <head><title>401 Authorization Required</title></head>
  <body>
  <center><h1>401 Authorization Required</h1></center>
  <hr><center>nginx/1.21.6</center>
  </body>
  </html>
  
``401 Authorization Required`` が応答されることがわかります。適切に JWT Validation が動作しています

適切なJWTを指定し、動作確認します

.. code-block:: cmdin

  curl -s -H "Host: nic.example.com" http://localhost/ -H "Token: `cat jwt-nic-nsm/nginx1.jwt`"

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  target v1.0

正しい応答が確認できます

この正しいJWTを提示している通信で攻撃トラフィックを送信します

.. code-block:: cmdin

  curl -s -H "Host: nic.example.com" "http://localhost//?<script>" -H "Token: `cat jwt-nic-nsm/nginx1.jwt`"

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  <html><head><title>Request Rejected</title></head><body>The requested URL was rejected.Please consult with your administrator.<br><br>
  Your support ID is:   16465265100495552517<br><br><a href='javascript:history.back();'>[Go Back]</a>
  </body></html>

``Request Rejected`` と表示されエラーが応答されました

この様にVSで複数のポリシーを指定することにより、正しいJWTを持つクライアントが悪意ある通信を行った際にも防御することができることが確認できました

3. 不要設定の削除
----

.. code-block:: cmdin

  ## cd ~/f5j-nginx-k8s-apigw-lab/example
  kubectl delete -f jwt-nic-nsm/jwk-secret.yaml -n staging
  kubectl delete -f jwt-nic-nsm/jwt.yaml -n staging
  kubectl delete -f waf-nic-vs/ap-logconf.yaml -n staging
  kubectl delete -f waf-nic-vs/simple-ap.yaml -n staging
  kubectl delete -f waf-nic-vs/waf.yaml -n staging
  kubectl delete -f waf-nic-vs/nic-vs-waf-jwt.yaml -n staging
