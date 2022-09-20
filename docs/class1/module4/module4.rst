インストール時に設定ファイルを作成する
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

Playbookの内容を確認します

.. code-block:: cmdin

  ## cd ~/f5j-nginx-ansible-lab
  cat playbook/deploy-nginx-plus-app-protect-waf-dos-proxyconf.yaml

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 29,31-33,34-35,139,140

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
  
      - name: Install NGINX App Protect WAF/DoS
        ansible.builtin.include_role:
          name: nginx_app_protect
        vars:
          nginx_app_protect_waf_enable: true
          nginx_app_protect_dos_enable: true
          nginx_app_protect_install_signatures: true
          nginx_app_protect_install_threat_campaigns: true
          nginx_app_protect_setup_license: false
          nginx_app_protect_remove_license: false
  
      - name: Configure NGINX
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
                  - name: upstr
                    least_conn: true
                    servers:
                      - address: 0.0.0.0:8081
                      - address: 0.0.0.0:8082
                servers:
                  - core:
                      listen:
                        - port: 80
                      server_name: localhost
                    app_protect_waf:
                      enable: true
                      security_log_enable: true
                    app_protect_dos:
                      enable: true
                    log:
                      access:
                        - path: /var/log/nginx/access.log
                          format: main
                    locations:
                      - location: /
                        proxy:
                          pass: http://upstr/
                          set_header:
                            field: Host
                            value: $host
                  - core:
                      listen:
                        - port: 8081
                      server_name: localhost
                    log:
                      access:
                        - path: /var/log/nginx/access.log
                          format: main
                    locations:
                      - location: /
                        core:
                          root: /usr/share/nginx/html
                          index: server_one.html
                    sub_filter:
                      sub_filters:
                        - string: server_hostname
                          replacement: $hostname
                        - string: server_address
                          replacement: $server_addr:$server_port
                        - string: server_url
                          replacement: $request_uri
                        - string: remote_addr
                          replacement: '$remote_addr:$remote_port'
                        - string: server_date
                          replacement: $time_local
                        - string: client_browser
                          replacement: $http_user_agent
                        - string: request_id
                          replacement: $request_id
                        - string: nginx_version
                          replacement: $nginx_version
                        - string: document_root
                          replacement: $document_root
                        - string: proxied_for_ip
                          replacement: $http_x_forwarded_for
                      once: false
                  - core:
                      listen:
                        - port: 8082
                      server_name: localhost
                    log:
                      access:
                        - path: /var/log/nginx/access.log
                          format: main
                    locations:
                      - location: /
                        core:
                          root: /usr/share/nginx/html
                          index: server_two.html
                    sub_filter:
                      sub_filters:
                        - string: server_hostname
                          replacement: $hostname
                        - string: server_address
                          replacement: $server_addr:$server_port
                        - string: server_url
                          replacement: $request_uri
                        - string: remote_addr
                          replacement: '$remote_addr:$remote_port'
                        - string: server_date
                          replacement: $time_local
                        - string: client_browser
                          replacement: $http_user_agent
                        - string: request_id
                          replacement: $request_id
                        - string: nginx_version
                          replacement: $nginx_version
                        - string: document_root
                          replacement: $document_root
                        - string: proxied_for_ip
                          replacement: $http_x_forwarded_for
                      once: false
  
          nginx_config_html_demo_template_enable: true
          nginx_config_html_demo_template:
            - template_file: www/index.html.j2
              deployment_location: /usr/share/nginx/html/server_one.html
              web_server_name: Ansible NGINX collection - Server one
            - template_file: www/index.html.j2
              deployment_location: /usr/share/nginx/html/server_two.html
              web_server_name: Ansible NGINX collection - Server two


- 29行目で ``nginx_config`` のロールを指定し、以降パラメータを指定し設定ファイルを生成します
- 31-33行目で、 ``nginx_config_modules`` により、利用するモジュールを指定します。この内容は ``nginx.conf`` の先頭に記述されます
- 34行目で、 ``nginx_config_http_template_enable`` により、HTTPを制御するNGINXの設定を記述することを指定していします
- 35行目の ``nginx_config_http_template`` に続き36行目から設定内容を記述します

  - 36行目 ``template_file`` : 利用するHTTP Teamplateを指定します
  - 37行目 ``deployment_location`` : 生成したファイルの保存場所を指定します
  - 38行目 ``config`` : 以降、設定ファイルに記述する内容を指定します

- 139行目で、 ``nginx_config_html_demo_template_enable`` により、HTMLファイルの内容記述することを指定していします
- 140行目の ``nginx_config_html_demo_template`` に続き36行目からHTMLの内容を記述します

  - 141,144行目 ``template_file`` : 利用するHTML Teamplateを指定します
  - 142,145行目 ``deployment_location`` : 生成したファイルの保存場所を指定します
  - 143,146行目 ``web_server_name`` : サーバ名を指定します

- 入力されたパラメータの結果以下のような設定が反映されます

  - Port 80 で待ち受けるServerブロックでローカルホストの8081、8082に通信を転送する
  - Port 8081 、 Port 8082 それぞれで、指定したHTMLを応答するWEBサーバとして動作する

- その他の詳細は `GitHub ansible-role-nginx-config <https://github.com/nginxinc/ansible-role-nginx-config>`__ を参照してください

NGINX Plus、NGINX App Protect WAF/DoS をインストール

.. code-block:: cmdin

  ## cd ~/f5j-nginx-ansible-lab
  ansible-playbook -i inventories/hosts -l nginx1 playbook/deploy-nginx-plus-app-protect-waf-dos-proxyconf.yaml --private-key="~/ssh_key/id_rsa"  --become

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  PLAY [all] ******************************************************************************************************************************************************************************************************************************************************************
  
  TASK [Gathering Facts] ******************************************************************************************************************************************************************************************************************************************************
  ok: [10.1.1.7]
  
  TASK [Install NGINX Plus] ***************************************************************************************************************************************************************************************************************************************************
  
  ** 省略 **

  PLAY RECAP ******************************************************************************************************************************************************************************************************************************************************************
  10.1.1.7                   : ok=56   changed=8    unreachable=0    failed=0    skipped=58   rescued=0    ignored=0

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

生成した ``default.conf`` の内容を確認します

.. code-block:: cmdin

  cat /etc/nginx/conf.d/default.conf

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 1

  #
  # Ansible managed
  #
  
  upstream upstr {
      server 0.0.0.0:8081;
      server 0.0.0.0:8082;
      least_conn;
  }
  
  server {
      listen 80;
      server_name localhost;
  
      app_protect_enable on;
      app_protect_security_log_enable on;
  
      app_protect_dos_enable on;
  
      access_log /var/log/nginx/access.log main;
  
      location / {
          proxy_pass http://upstr/;
          proxy_set_header Host $host;
  
  
      }
  }
  server {
      listen 8081;
      server_name localhost;
  
      access_log /var/log/nginx/access.log main;
  
      sub_filter server_hostname $hostname;
      sub_filter server_address $server_addr:$server_port;
      sub_filter server_url $request_uri;
      sub_filter remote_addr $remote_addr:$remote_port;
      sub_filter server_date $time_local;
      sub_filter client_browser $http_user_agent;
      sub_filter request_id $request_id;
      sub_filter nginx_version $nginx_version;
      sub_filter document_root $document_root;
      sub_filter proxied_for_ip $http_x_forwarded_for;
      sub_filter_once off;
  
      location / {
          root /usr/share/nginx/html;
          index server_one.html;
  
  
      }
  }
  server {
      listen 8082;
      server_name localhost;
  
      access_log /var/log/nginx/access.log main;
  
      sub_filter server_hostname $hostname;
      sub_filter server_address $server_addr:$server_port;
      sub_filter server_url $request_uri;
      sub_filter remote_addr $remote_addr:$remote_port;
      sub_filter server_date $time_local;
      sub_filter client_browser $http_user_agent;
      sub_filter request_id $request_id;
      sub_filter nginx_version $nginx_version;
      sub_filter document_root $document_root;
      sub_filter proxied_for_ip $http_x_forwarded_for;
      sub_filter_once off;
  
      location / {
          root /usr/share/nginx/html;
          index server_two.html;
  
  
      }
  }

- 改めてAnsible Playbookの内容を確認すると、config配下で、 ``upstream`` と ``servers`` というパートに分かれ、更に ``servers`` の配下に3つの ``core`` が記述されていることが確認できます
- 設定ファイルの内容は大きく、 ``upstream`` ブロック、 3つの ``server`` ブロックが記述されていることが確認でき、Playbookとの対比がわかります
- Playbookの ``core`` と設定ファイルの ``server`` は、listenに指定されているポート番号から確認できます

grep コマンドで ``Ansible`` の文字列を指定し、HTMLファイルの出力結果を確認します

.. code-block:: cmdin

  grep Ansible /usr/share/nginx/html/*html

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 1,4

  /usr/share/nginx/html/server_one.html:<!-- Ansible managed -->
  /usr/share/nginx/html/server_one.html:<title>Hello World - Ansible NGINX collection - Server one</title>
  /usr/share/nginx/html/server_one.html:<p><span>Web Server name:</span> <span> Ansible NGINX collection - Server one </span></p>
  /usr/share/nginx/html/server_two.html:<!-- Ansible managed -->
  /usr/share/nginx/html/server_two.html:<title>Hello World - Ansible NGINX collection - Server two</title>
  /usr/share/nginx/html/server_two.html:<p><span>Web Server name:</span> <span> Ansible NGINX collection - Server two </span></p>

- 1-3行目が ``server_one.html`` 、 4-6行目が ``server_two.html`` の内容です
- 3,6行目に ``title`` 、 4,7行目に ``span`` で それぞれ指定した ``web_server_name`` の文字列が挿入されています

2. 動作確認
====

Curlコマンドを実行し、応答結果を確認します

.. code-block:: cmdin

  for i in {1..3}; do echo "==$i==" ; curl -s localhost | grep -e Ansible -e "<span>"  ; sleep 1 ; done

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 1,14,27

  ==1==
  <!-- Ansible managed -->
  <title>Hello World - Ansible NGINX collection - Server one</title>
  <p><span>Web Server name:</span> <span> Ansible NGINX collection - Server one </span></p>
  <p><span>Server name:</span> <span>ip-10-1-1-7</span></p>
  <p><span>Server address:</span> <span>127.0.0.1:8081</span></p>
  <p><span>User Agent:</span> <span><small>curl/7.68.0</small></span></p>
  <p class="smaller"><span>URI:</span> <span>/</span></p>
  <p class="smaller"><span>Doc Root:</span> <span>/usr/share/nginx/html</span></p>
  <p class="smaller"><span>Date:</span> <span>15/Sep/2022:11:28:56 +0900</span></p>
  <p class="smaller"><span>NGINX Front-End Load Balancer IP:</span><span>127.0.0.1:34952</span></p>
  <p class="smaller"><span>Client IP:</span> <span></span></p>
  <p class="smaller"><span>NGINX Version:</span> <span>1.21.6</span></p>
  ==2==
  <!-- Ansible managed -->
  <title>Hello World - Ansible NGINX collection - Server two</title>
  <p><span>Web Server name:</span> <span> Ansible NGINX collection - Server two </span></p>
  <p><span>Server name:</span> <span>ip-10-1-1-7</span></p>
  <p><span>Server address:</span> <span>127.0.0.1:8082</span></p>
  <p><span>User Agent:</span> <span><small>curl/7.68.0</small></span></p>
  <p class="smaller"><span>URI:</span> <span>/</span></p>
  <p class="smaller"><span>Doc Root:</span> <span>/usr/share/nginx/html</span></p>
  <p class="smaller"><span>Date:</span> <span>15/Sep/2022:11:28:57 +0900</span></p>
  <p class="smaller"><span>NGINX Front-End Load Balancer IP:</span><span>127.0.0.1:54620</span></p>
  <p class="smaller"><span>Client IP:</span> <span></span></p>
  <p class="smaller"><span>NGINX Version:</span> <span>1.21.6</span></p>
  ==3==
  <!-- Ansible managed -->
  <title>Hello World - Ansible NGINX collection - Server one</title>
  <p><span>Web Server name:</span> <span> Ansible NGINX collection - Server one </span></p>
  <p><span>Server name:</span> <span>ip-10-1-1-7</span></p>
  <p><span>Server address:</span> <span>127.0.0.1:8081</span></p>
  <p><span>User Agent:</span> <span><small>curl/7.68.0</small></span></p>
  <p class="smaller"><span>URI:</span> <span>/</span></p>
  <p class="smaller"><span>Doc Root:</span> <span>/usr/share/nginx/html</span></p>
  <p class="smaller"><span>Date:</span> <span>15/Sep/2022:11:28:58 +0900</span></p>
  <p class="smaller"><span>NGINX Front-End Load Balancer IP:</span><span>127.0.0.1:34960</span></p>
  <p class="smaller"><span>Client IP:</span> <span></span></p>
  <p class="smaller"><span>NGINX Version:</span> <span>1.21.6</span></p>

- forで3回 Curl コマンドによるリクエストを実行します
- 1,14,27行目に、コマンドを実行した回数が表示され、その番号以降がその結果となります
- 3,17,30行目の内容が、Playbookで指定した ``web_server_name`` となり、 one , two , one と交互に表示されていることがわかります
- ``<span>`` で表示される要素は、 ``sub_filter`` により値が置換されていることが確認できます

3. 環境の削除
====

環境を削除する場合、 ` <https://f5j-nginx-ansible.readthedocs.io/en/latest/class1/module3/module3.html#id3>`__ の内容を参考にコマンドを実行してください