APIGW 通信制御の実施
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

3. NSM設定のデプロイ (割合9:1)
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


4. NSM設定の動作確認 (割合9:1)
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


4. JWT制御とWAFによる防御
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
`3. NIC/NSMのJWT制御 <https://f5j-nginx-k8s-apigw.readthedocs.io/en/latest/class1/module03/module03.html#nic-nsmjwt>`__ 
と同様に
`Ingress Controller で JWT Validation のデプロイ <https://f5j-nginx-ingress-controller-lab1.readthedocs.io/en/latest/class1/module3/module3.html#ingress-controller-jwt-validation>`__
の内容を利用しています。

WAFの設定は最低限の設定を行い、外部からの攻撃をブロックできる設定としています。

.. code-block:: cmdin

  ## cd ~/f5j-nginx-k8s-apigw-lab/example
  cat waf-nic-vs/simple-ap.yaml

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 9

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

反映の結果を確認します


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

不要な設定を削除します

.. code-block:: cmdin

  ## cd ~/f5j-nginx-k8s-apigw-lab/example
  kubectl delete -f jwt-nic-nsm/jwk-secret.yaml -n staging
  kubectl delete -f jwt-nic-nsm/jwt.yaml -n staging
  kubectl delete -f waf-nic-vs/ap-logconf.yaml -n staging
  kubectl delete -f waf-nic-vs/simple-ap.yaml -n staging
  kubectl delete -f waf-nic-vs/waf.yaml -n staging
  kubectl delete -f waf-nic-vs/nic-vs-waf-jwt.yaml -n staging


5. NICによる条件に応じた制御
====

以下の構成で動作を確認します

   .. image:: ./media/nic-vs-acl.png
      :width: 400

``request_path`` , ``methods`` , ``headers`` による通信制御を確認します

1. 設定のデプロイ
----

設定の内容を確認します

.. code-block:: cmdin

  ## cd ~/f5j-nginx-k8s-apigw-lab/example
  cat jwt-nic-nsm/nic-vs-acl.yaml

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 13-15,19-22,24-27,28-32

  apiVersion: k8s.nginx.org/v1
  kind: VirtualServer
  metadata:
    name: nic
  spec:
    host: nic.example.com
    policies:
    upstreams:
    - name: target-svc
      service: target-svc
      port: 80
    routes:
    - path: ~ /.*valid.*
      action:
        pass: target-svc
    - path: /
      matches:
      - conditions:
        - header: X-Type
          value: valid
        action:
          pass: target-svc
      - conditions:
        - variable: $request_method
          value: POST
        action:
          pass: target-svc
      action:
        return:
          code: 403
          type: text/plain
          body: "Error\n"

- 13-15行目で、 PATHの条件を正規表現で指定し、 ``valid`` の文字列が含まれる場合、 ``target-svc`` に転送します
- 19-22行目で、 ``X-Type`` というHTTP Headerの値をチェックし ``valid`` である場合、 ``target-svc`` に転送します
- 24-27行目で、 ``$request_method`` という変数を指定しHTTP Methodが ``POST`` である場合、 ``target-svc`` に転送します
- その他の通信は、 ``path: /`` の 28-32行目の条件に該当します 

設定を反映します

.. code-block:: cmdin

  ## cd ~/f5j-nginx-k8s-apigw-lab/example
  kubectl apply -f acl-nic-vs/nic-vs-acl.yaml -n staging

2. 動作確認
----

動作を確認します

まずシンプルなリクエストを送付し、結果を確認します

.. code-block:: cmdin

  curl -v -H "Host: nic.example.com" http://localhost/

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  *   Trying 127.0.0.1:80...
  * TCP_NODELAY set
  * Connected to localhost (127.0.0.1) port 80 (#0)
  > GET / HTTP/1.1
  > Host: nic.example.com
  > User-Agent: curl/7.68.0
  > Accept: */*
  >
  * Mark bundle as not supporting multiuse
  < HTTP/1.1 403 Forbidden
  < Server: nginx/1.21.6
  < Date: Wed, 30 Nov 2022 12:32:55 GMT
  < Content-Type: text/plain
  < Content-Length: 6
  < Connection: keep-alive
  <
  Error

通信を送付したところ ``403`` が応答されていることがわかります。curlコマンドではデフォルトのMethodがGETであり、指定したポリシーの条件に該当しないためエラーとなっています


ポリシーに記述したMethodでリクエストを送付します。

.. code-block:: cmdin

  curl -v -H "Host: nic.example.com" http://localhost/ -X POST

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 4,10,18

  *   Trying 127.0.0.1:80...
  * TCP_NODELAY set
  * Connected to localhost (127.0.0.1) port 80 (#0)
  > POST / HTTP/1.1
  > Host: nic.example.com
  > User-Agent: curl/7.68.0
  > Accept: */*
  >
  * Mark bundle as not supporting multiuse
  < HTTP/1.1 200 OK
  < Server: nginx/1.21.6
  < Date: Wed, 30 Nov 2022 12:37:24 GMT
  < Content-Type: text/plain
  < Content-Length: 12
  < Connection: keep-alive
  < X-Mesh-Request-ID: 3d4569c9fb09e210121aa3efca06ca85
  <
  target v1.0

4行目で ``POST`` で通信が送付され、 10行目で ``200 OK`` 18行目で正しく応答が返されていることが確認できます

制御対象のURL ポリシーに記述したPathの条件を満たすリクエストを送付します。

.. code-block:: cmdin

  curl -v -H "Host: nic.example.com" http://localhost/dummy/this-is-valid-path/a.jpg

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 4,10,18

  *   Trying 127.0.0.1:80...
  * TCP_NODELAY set
  * Connected to localhost (127.0.0.1) port 80 (#0)
  > GET /dummy/this-is-valid-path/a.jpg HTTP/1.1
  > Host: nic.example.com
  > User-Agent: curl/7.68.0
  > Accept: */*
  >
  * Mark bundle as not supporting multiuse
  < HTTP/1.1 200 OK
  < Server: nginx/1.21.6
  < Date: Wed, 30 Nov 2022 12:38:02 GMT
  < Content-Type: image/jpeg
  < Content-Length: 12
  < Connection: keep-alive
  < X-Mesh-Request-ID: ef2205a2b89c70b653c642df14dc2f4d
  <
  target v2.0
  * Connection #0 to host localhost left intact

4行目で 指定のPATHに通信が送付され、 10行目で ``200 OK`` 18行目で正しく応答が返されていることが確認できます

ポリシーに記述したCookieの条件を満たすリクエストを送付します。

.. code-block:: cmdin

  curl -v -H "Host: nic.example.com" http://localhost/ -H "X-Type: valid"

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 4,10,19

  *   Trying 127.0.0.1:80...
  * TCP_NODELAY set
  * Connected to localhost (127.0.0.1) port 80 (#0)
  > GET / HTTP/1.1
  > Host: nic.example.com
  > User-Agent: curl/7.68.0
  > Accept: */*
  > X-Type: valid
  >
  * Mark bundle as not supporting multiuse
  < HTTP/1.1 200 OK
  < Server: nginx/1.21.6
  < Date: Wed, 30 Nov 2022 12:38:41 GMT
  < Content-Type: text/plain
  < Content-Length: 12
  < Connection: keep-alive
  < X-Mesh-Request-ID: cc3e1e8df58f0a1200456d76a551f6c7
  <
  target v1.0

8行目で指定したHTTP Headerが付与された通信が送付され、 11行目で ``200 OK`` 19行目で正しく応答が返されていることが確認できます

このサンプルでは、条件に該当する場合サービスに転送し、それ以外をエラーとする設定です。
condition は様々な条件を記述することが可能です。該当する処理をエラーだけでなくリダイレクト、その他通信と違うServiceに転送するなどが可能となります

3. 不要設定の削除
----

不要な設定を削除します

.. code-block:: cmdin

  ## cd ~/f5j-nginx-k8s-apigw-lab/example
  kubectl delete -f acl-nic-vs/nic-vs-acl.yaml -n staging


6. NSMによる条件に応じた制御
====

以下の構成で動作を確認します

   .. image:: ./media/nsm-smi-acl.png
      :width: 400

``request_path`` , ``methods`` , ``headers`` による通信制御を確認します。
詳細は以降の設定で確認しますが、SMIの記述では ``許可する条件`` を指定することが可能となります。
``5. NICによる条件に応じた制御`` では条件に対して自由なActionを指定できましたが、その点が異なることを注意ください


1. 設定のデプロイ
----

ここで実施するNSMのSMIによる通信制御では、Deploymentに指定されたServiceAccountを確認し、その送信元・送信先ServiceAccountを指定し通信を制御します。

サービスアカウントを確認します

.. code-block:: cmdin

  kubectl get sa -n staging

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 3-5

  NAME          SECRETS   AGE
  default       1         6d9h
  target-v1-0   1         11m
  target-v2-0   1         14s
  webapp        1         25s

DeploymentのPod Templateで指定されている ``Service Account`` を確認します

.. code-block:: cmdin

  kubectl describe deployment -n staging | egrep 'Pod Template:|Service Account:|^Name:'

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 3,6,9

  Name:                   target-v1-0
  Pod Template:
    Service Account:  target-v1-0
  Name:                   target-v2-0
  Pod Template:
    Service Account:  target-v2-0
  Name:                   webapp
  Pod Template:
    Service Account:  webapp

各Deploymentに対し、それぞれService Accountが指定されていることが確認できます。


設定の内容を確認します。VSの内容は `2. NSMのトラフィック分割(Canary Release) のNIC設定 <https://f5j-nginx-k8s-apigw.readthedocs.io/en/latest/class1/module03/module03.html#nsm-canary-release>`__ と同じであり大変シンプルな内容のため割愛します。

NSMに指定するポリシーの内容を確認します。

.. code-block:: cmdin

  ## cd ~/f5j-nginx-k8s-apigw-lab/example
  cat acl-nsm-smi/nsm-acl.yaml

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  apiVersion: specs.smi-spec.io/v1alpha3
  kind: HTTPRouteGroup
  metadata:
    name: route-group
  spec:
    matches:
    - name: method
      methods:
      - POST
    - name: path
      pathRegex: "/.*valid.*"
    - name: header
      headers:
        X-Type: "valid"

  ---
  apiVersion: access.smi-spec.io/v1alpha2
  kind: TrafficTarget
  metadata:
    name: traffic-target-v1
  spec:
    destination:
      kind: ServiceAccount
      name: target-v1-0
    rules:
    - kind: HTTPRouteGroup
      name: route-group
      matches:
      - method
      - path
      - header
    sources:
    - kind: ServiceAccount
      name: webapp
  
  ---
  apiVersion: access.smi-spec.io/v1alpha2
  kind: TrafficTarget
  metadata:
    name: traffic-target-v2
  spec:
    destination:
      kind: ServiceAccount
      name: target-v2-0
    rules:
    - kind: HTTPRouteGroup
      name: route-group
      matches:
      - method
      - path
      - header
    sources:
    - kind: ServiceAccount
      name: webapp
  
  :emphasize-lines: 7-14,24,28-30,44,48-50

- 1-14行目で、対象とする条件を指定します。kind は ``HTTPRouteGroup`` となり、オブジェクト名は ``route-group`` です
- 条件は以下の三種類となります
  
  - 7-9行目: HTTP Method で ``POST`` を指定
  - 10-11行目: path で ``"/.*valid.*"`` を指定し、 ``valid`` が含まれる pathを対象とする
  - 12-14行目: HTTP Header を対象とし、 ``X-Type`` の値が ``valid`` となっているものを対象とする

- 17-34行目が、 ``webapp`` から ``target-v1-0`` に対する設定、27-54行目が、 ``webapp`` から ``target-v2-0`` に対する設定となります
- これらの違いは destination のみで、24行目で ``target-v1-0`` 、 44行目で ``target-v2-0`` を指定しています。その他の内容は同様です

設定を反映します

.. code-block:: cmdin

  ## cd ~/f5j-nginx-k8s-apigw-lab/example
  kubectl apply -f acl-nsm-smi/nic-vs-acl.yaml -n staging
  kubectl apply -f acl-nsm-smi/nsm-acl.yaml -n staging


2. 動作確認
----

動作を確認します

まずシンプルなリクエストを送付し、結果を確認します

.. code-block:: cmdin

  curl -s -H "Host: webapp.example.com" http://localhost/

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  <html>
  <head><title>403 Forbidden</title></head>
  <body>
  <center><h1>403 Forbidden</h1></center>
  <hr><center>nginx/1.21.6</center>
  </body>
  </html>

通信を送付したところ ``403`` が応答されていることがわかります。curlコマンドではデフォルトのMethodがGETであり、指定したポリシーの条件に該当しないためエラーとなっています

ポリシーに記述したMethodでリクエストを送付します。

.. code-block:: cmdin

  curl -v -H "Host: webapp.example.com" http://localhost/ -X POST

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 4,10,19

  *   Trying 127.0.0.1:80...
  * TCP_NODELAY set
  * Connected to localhost (127.0.0.1) port 80 (#0)
  > POST / HTTP/1.1
  > Host: webapp.example.com
  > User-Agent: curl/7.68.0
  > Accept: */*
  >
  * Mark bundle as not supporting multiuse
  < HTTP/1.1 200 OK
  < Server: nginx/1.21.6
  < Date: Wed, 30 Nov 2022 11:54:06 GMT
  < Content-Type: text/plain
  < Content-Length: 12
  < Connection: keep-alive
  < X-Mesh-Request-ID: 4a64156f62a3a613671af6e6650b9ac5
  < X-Mesh-Request-ID: 1df02eadd498d94aaaa0db3d76b901a3
  <
  target v1.0
  * Connection #0 to host localhost left intact

4行目で ``POST`` で通信が送付され、 10行目で ``200 OK`` 19行目で正しく応答が返されていることが確認できます

ポリシーに記述したPathの条件を満たすリクエストを送付します。

.. code-block:: cmdin

  curl -v -H "Host: webapp.example.com" http://localhost/dummy/this-is-valid-path/a.jpg

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 4,10,19

  *   Trying 127.0.0.1:80...
  * TCP_NODELAY set
  * Connected to localhost (127.0.0.1) port 80 (#0)
  > GET /dummy/this-is-valid-path/a.jpg HTTP/1.1
  > Host: webapp.example.com
  > User-Agent: curl/7.68.0
  > Accept: */*
  >
  * Mark bundle as not supporting multiuse
  < HTTP/1.1 200 OK
  < Server: nginx/1.21.6
  < Date: Wed, 30 Nov 2022 11:58:30 GMT
  < Content-Type: image/jpeg
  < Content-Length: 12
  < Connection: keep-alive
  < X-Mesh-Request-ID: 0fa1636fee4c962e79fc7091bdb47e01
  < X-Mesh-Request-ID: 53d69013d169237c948b2a1ca0962428
  <
  target v2.0

4行目で 指定のPATHに通信が送付され、 10行目で ``200 OK`` 19行目で正しく応答が返されていることが確認できます

ポリシーに記述したCookieの条件を満たすリクエストを送付します。

.. code-block:: cmdin

  curl -v -H "Host: webapp.example.com" http://localhost/ -H "X-Type: valid"

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 8,11,20

  *   Trying 127.0.0.1:80...
  * TCP_NODELAY set
  * Connected to localhost (127.0.0.1) port 80 (#0)
  > GET / HTTP/1.1
  > Host: webapp.example.com
  > User-Agent: curl/7.68.0
  > Accept: */*
  > X-Type: valid
  >
  * Mark bundle as not supporting multiuse
  < HTTP/1.1 200 OK
  < Server: nginx/1.21.6
  < Date: Wed, 30 Nov 2022 11:59:23 GMT
  < Content-Type: text/plain
  < Content-Length: 12
  < Connection: keep-alive
  < X-Mesh-Request-ID: aa232231e3b082bd8487a907ec3d8e32
  < X-Mesh-Request-ID: 103858cec23cdc936331e4009aa20759
  <
  target v1.0
  * Connection #0 to host localhost left intact

8行目で指定したHTTP Headerが付与された通信が送付され、 11行目で ``200 OK`` 20行目で正しく応答が返されていることが確認できます

この様にコンテナ内部の通信に対して、Deploymentに指定したService Accountを使って通信の制御を行うことが可能です

3. 不要設定の削除
----

不要な設定を削除します

.. code-block:: cmdin

  ## cd ~/f5j-nginx-k8s-apigw-lab/example
  kubectl delete -f acl-nsm-smi/nic-vs-acl.yaml -n staging
  kubectl delete -f acl-nsm-smi/nsm-acl.yaml -n staging
