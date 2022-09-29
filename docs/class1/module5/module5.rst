インストール・WAFポリシーを設定する
####

.. NOTE::

  | 同環境でNGINXをアンインストール後に本作業を行う場合、設定ファイルの情報が残っている場合があります。
  | 完全に削除する場合には以下コマンドを参考に削除してください。

  .. code-block::

    sudo apt remove nginx-plus --purge

.. NOTE::

  | Ansibleでjinja templateを利用し、NGINXの設定を行う場合、エラーとなる場合があります。
  | エラーを回避するためには、Pythonのパッケージマネージャである ``pip`` をインストールし、 ``jinja2`` のパッケージをアップデートする必要があります。
  | 詳細は `Jinja Templateが正しく動作しない場合 <https://f5j-nginx-ansible.readthedocs.io/en/latest/class1/module9/module9.html#jinja-template>`__ を参照してください。


1. インストール、設定ファイル作成
====

NGINX Plus, NAP WAF/DoSインストールと同時に設定ファイルを作成することが可能です。
プロジェクトで共通の設定や初期設定がある場合にはこの方法が便利です。

ライセンス、Playbookなど正しく配置していることを想定し説明を進めます。
確認が必要な場合 `1. 事前確認 <https://f5j-nginx-ansible.readthedocs.io/en/latest/class1/module3/module3.html#id2>`__ を参照してください。

| Playbookの内容を確認します。
| (設定の内容は `NGINX Plus Lab Security の 2. 通信のブロック <https://f5j-nginx-plus-lab2-security.readthedocs.io/en/latest/class1/module03/module03.html>`__  の内容となります)

.. code-block:: cmdin

  ## cd ~/f5j-nginx-ansible-lab
  cat playbook/deploy-nginx-plus-app-protect-waf-dos-proxywafconf.yaml

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 26-34,46,60-66
  ---
  - hosts: all
    collections:
      - nginxinc.nginx_core
    tasks:
      - name: Install NGINX Plus
        ansible.builtin.include_role:
          name: nginx
        vars:
          nginx_type: plus
          nginx_license:
            certificate: ~/nginx-repo.crt
            key: ~/nginx-repo.key
          nginx_remove_license: false
  
      - name: Install NGINX App Protect WAF/DoS Copy Policy
        ansible.builtin.include_role:
          name: nginx_app_protect
        vars:
          nginx_app_protect_waf_enable: true
          nginx_app_protect_dos_enable: true
          nginx_app_protect_install_signatures: true
          nginx_app_protect_install_threat_campaigns: true
          nginx_app_protect_setup_license: false
          nginx_app_protect_remove_license: false
          # Copy local NGINX App Protect log policy to host
          nginx_app_protect_security_policy_file_enable: true
          nginx_app_protect_security_policy_file:
            - src: ../files/nap-waf/custom_policy.json
              dest: /etc/nginx/conf.d/custom_policy.json
          nginx_app_protect_log_policy_file_enable: true
          nginx_app_protect_log_policy_file:
            - src: ../files/nap-waf/custom_log_format.json
              dest: /etc/nginx/conf.d/custom_log_format.json
  
      - name: Configure NGINX for NAP WAF/DoS
        ansible.builtin.include_role:
          name: nginx_config
        vars:
          nginx_config_modules:
            - modules/ngx_http_app_protect_module.so
            - modules/ngx_http_app_protect_dos_module.so
          nginx_config_http_template_enable: true
          nginx_config_http_template:
            - template_file: http/default.conf.j2
              deployment_location: /etc/nginx/conf.d/default.conf
              config:
                upstreams:
                  - name: server_group
                    zone:
                      name: backend
                      size: 64k
                    servers:
                      - address: security-backend1:80
                servers:
                  - core:
                      listen:
                        - port: 80
                      server_name: localhost
                    app_protect_waf:
                      enable: true
                      policy_file: /etc/nginx/conf.d/custom_policy.json
                      security_log_enable: true
                      security_log:  # Dictionary or a list of dictionaries
                        - path: /etc/nginx/conf.d/custom_log_format.json
                          dest: syslog:server=elasticsearch:5144
                    app_protect_dos:
                      enable: true
                    log:
                      access:
                        - path: /var/log/nginx/access.log
                          format: main
                    locations:
                      - location: /
                        proxy:
                          pass: http://server_group/
                          set_header:
                            field: Host
                            value: $host


- 26-34行目で、Ansibleで実行するPlaybookから相対パスで指定する ``../files/nap-waf/`` に配置されているSecurity Policy、Log Policyのファイルを実行先のホストへコピーします
- 60-66行目で、 ``App Protect WAF`` の設定を記述します。この内容は、46行目のファイルに反映されます
- その他のパラメータの詳細は `GitHub ansible-role-nginx-config app_protect_waf <https://github.com/nginxinc/ansible-role-nginx-config/blob/main/defaults/main/template.yml#L312-L330>`__ の項目を参照してください

NGINX Plus、NGINX App Protect WAF/DoS をインストール

.. code-block:: cmdin

  ## cd ~/f5j-nginx-ansible-lab
  ansible-playbook -i inventories/hosts -l nginx1 playbook/deploy-nginx-plus-app-protect-waf-dos-proxywafconf.yaml --private-key="~/.ssh/id_rsa"  --become

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  PLAY [all] ******************************************************************************************************************************************************************************************************************************************************************
  
  TASK [Gathering Facts] ******************************************************************************************************************************************************************************************************************************************************
  ok: [10.1.1.7]
  
  TASK [Install NGINX Plus] ***************************************************************************************************************************************************************************************************************************************************
  
  ** 省略 **
  
  PLAY RECAP ******************************************************************************************************************************************************************************************************************************************************************
  10.1.1.7                   : ok=61   changed=10   unreachable=0    failed=0    skipped=57   rescued=0    ignored=0


実際に生成されたファイルの内容を確認します

``nginx.conf`` の先頭行を確認し、モジュールのロードを行うコマンドについて確認します

.. code-block:: cmdin

  head /etc/nginx/nginx.conf

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 1,2

  load_module modules/ngx_http_app_protect_dos_module.so;
  load_module modules/ngx_http_app_protect_module.so;
  
  user  nginx;
  worker_processes  auto;
  
  error_log  /var/log/nginx/error.log notice;
  pid        /var/run/nginx.pid;

- 1,2行目に、指定したモジュールを読み込む設定が記述されています

WAFの設定で利用するセキュリティポリシーファイルがコピーされていることを確認します

.. code-block:: cmdin

  ls -1 /etc/nginx/conf.d/custom*

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  /etc/nginx/conf.d/custom_log_format.json
  /etc/nginx/conf.d/custom_policy.json

生成した ``default.conf`` の内容を確認します

.. code-block:: cmdin

  cat /etc/nginx/conf.d/default.conf

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 14-17

  #
  # Ansible managed
  #
  
  upstream server_group {
      server security-backend1:80;
      zone backend 64k;
  }
  
  server {
      listen 80;
      server_name localhost;
  
      app_protect_enable on;
      app_protect_policy_file /etc/nginx/conf.d/custom_policy.json;
      app_protect_security_log_enable on;
      app_protect_security_log /etc/nginx/conf.d/custom_log_format.json syslog:server=elasticsearch:5144;
  
      app_protect_dos_enable on;
  
      access_log /var/log/nginx/access.log main;
  
      location / {
          proxy_pass http://server_group/;
          proxy_set_header Host $host;
  
  
      }
  }

- 14-15行目で、App Protect WAFを有効にし、WAFポリシーファイルを指定します
- 16-17行目で、App Protect WAFのログを有効にし、ログポリシーを指定したファイルと転送先を指定します

2. 動作確認
====

Curlコマンドを実行し、応答結果を確認します

.. code-block:: cmdin

  curl -s "localhost/?a=<script>"

以下が表示結果サンプルです。HTMLを見やすいように開業し整形しています。

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 3,7

  <html>
    <head>
      <title>Request Rejected</title>
    </head>
    <body>
      The requested URL was rejected. Please consult with your administrator.<br><br>
      Your support ID is: 3589473112366060476<br><br>
      <a href='javascript:history.back();'>[Go Back]</a>
    </body>
  </html>


- NGINX App Protect WAFによって通信がブロックされており、 3行目に ``Request Rejected`` と表示されています
- 7行目に ``Support ID`` が表示されています。このIDを利用することにより該当するログメッセージを特定することができます

F5 UDFのラボ環境で実行する場合、同環境にELKをデプロイしています。以下の手順を参考にELKを開き、WAFのLogに ``support_id **画面に表示されたsupport ID**`` を入力し結果を確認してください
`NGINX Plus Lab Security の各動作確認 <https://f5j-nginx-plus-lab2-security.readthedocs.io/en/latest/class1/module03/module03.html>`__ の内容を確認してください

3. 環境の削除
====

環境を削除する場合、 `3. NGINX Plus、NAP WAF/DoSのアンインストール <https://f5j-nginx-ansible.readthedocs.io/en/latest/class1/module3/module3.html#id2>`__ の内容を参考にコマンドを実行してください