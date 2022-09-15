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

- その他の詳細は `GitHub ansible-role-nginx-config <https://github.com/nginxinc/ansible-role-nginx-config>`__ を参照してください

NGINX Plus、NGINX App Protect WAF/DoS をインストール

.. code-block:: cmdin

  ## cd ~/f5j-nginx-ansible-lab
  ansible-playbook -i inventories/hosts -l nginx1 playbook/deploy-nginx-plus-app-protect-waf-dos-proxyconfig.yaml --private-key="~/ssh_key/id_rsa"  --become

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

実際に生成されたファイルの内容を確認します

.. code-block:: cmdin

  head /etc/nginx/nginx.conf

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 1



.. code-block:: cmdin

  cat /etc/nginx/conf.d/default.conf

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 1


.. code-block:: cmdin

  diff -u /usr/share/nginx/html/server_one.html /usr/share/nginx/html/server_two.html

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 1
