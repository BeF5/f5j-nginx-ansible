(参考) Ansibleで期待した動作をしない場合の対応
####

Jinja Templateが正しく動作しない場合
====

Ansible実行時で出力されるJinjaのエラー
----

| Ansible Playbook では Jinja Templateを利用する機能があります。
| 以下のようなエラーが出力されJinja Templateが正しく完了しない場合があります。

.. code-block:: bash
  :linenos:
  :caption: エラーサンプル

  TASK [nginxinc.nginx_core.nginx_config : Dynamically generate NGINX HTTP config files] ******************************
  An exception occurred during task execution. To see the full traceback, use -vvv. The error was: jinja2.exceptions.TemplateAssertionError: no test named 'boolean'
  failed: [10.1.1.7] (item=/etc/nginx/conf.d/default.conf) => {"ansible_loop_var": "item", "changed": false, "item": {"config": {"servers": [{"app_protect_dos": {"enable": true}, "app_protect_waf": {"enable": true, "security_log_enable": true}, "core": {"listen": [{"port": 80}], "server_name": "localhost"}, "locations": [{"location": "/", "proxy": {"pass": "http://upstr/", "set_header": {"field": "Host", "value": "$host"}}}], "log": {"access": [{"format": "main", "path": "/var/log/nginx/access.log"}]}}, {"core": {"listen": [{"port": 8081}], "server_name": "localhost"}, "locations": [{"core": {"index": "server_one.html", "root": "/usr/share/nginx/html"}, "location": "/"}], "log": {"access": [{"format": "main", "path": "/var/log/nginx/access.log"}]}, "sub_filter": {"once": false, "sub_filters": [{"replacement": "$hostname", "string": "server_hostname"}, {"replacement": "$server_addr:$server_port", "string": "server_address"}, {"replacement": "$request_uri", "string": "server_url"}, {"replacement": "$remote_addr:$remote_port", "string": "remote_addr"}, {"replacement": "$time_local", "string": "server_date"}, {"replacement": "$http_user_agent", "string": "client_browser"}, {"replacement": "$request_id", "string": "request_id"}, {"replacement": "$nginx_version", "string": "nginx_version"}, {"replacement": "$document_root", "string": "document_root"}, {"replacement": "$http_x_forwarded_for", "string": "proxied_for_ip"}]}}, {"core": {"listen": [{"port": 8082}], "server_name": "localhost"}, "locations": [{"core": {"index": "server_two.html", "root": "/usr/share/nginx/html"}, "location": "/"}], "log": {"access": [{"format": "main", "path": "/var/log/nginx/access.log"}]}, "sub_filter": {"once": false, "sub_filters": [{"replacement": "$hostname", "string": "server_hostname"}, {"replacement": "$server_addr:$server_port", "string": "server_address"}, {"replacement": "$request_uri", "string": "server_url"}, {"replacement": "$remote_addr:$remote_port", "string": "remote_addr"}, {"replacement": "$time_local", "string": "server_date"}, {"replacement": "$http_user_agent", "string": "client_browser"}, {"replacement": "$request_id", "string": "request_id"}, {"replacement": "$nginx_version", "string": "nginx_version"}, {"replacement": "$document_root", "string": "document_root"}, {"replacement": "$http_x_forwarded_for", "string": "proxied_for_ip"}]}}], "upstreams": [{"least_conn": true, "name": "upstr", "servers": [{"address": "0.0.0.0:8081"}, {"address": "0.0.0.0:8082"}]}]}, "deployment_location": "/etc/nginx/conf.d/default.conf", "template_file": "http/default.conf.j2"}, "msg": "TemplateAssertionError: no test named 'boolean'"}


UbuntuのパッケージでインストールされたJinja TemplateのVersionによる動作である場合があるため、Jinja をアップデートします


インストールされているパッケージの確認
----

.. code-block:: bash
  :linenos:
  :caption: バージョンの確認

  $ pip list | grep -i jinja
  Jinja2                 2.10.1
  
  $ ansible --version
  ansible [core 2.12.8]
    config file = /etc/ansible/ansible.cfg
    configured module search path = ['/home/ubuntu/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
    ansible python module location = /usr/lib/python3/dist-packages/ansible
    ansible collection location = /home/ubuntu/.ansible/collections:/usr/share/ansible/collections
    executable location = /usr/bin/ansible
    python version = 3.8.10 (default, Jun 22 2022, 20:18:18) [GCC 9.4.0]
    jinja version = 2.10.1
    libyaml = True
  
  $ dpkg-query -l | grep -e jinja -e ansible
  ii  ansible                            5.10.0-1ppa~focal                     all          batteries-included package providing a curated set of Ansible collections in addition to ansible-core
  ii  ansible-core                       2.12.8-1ppa~focal                     all          Ansible IT Automation
  ii  python3-jinja2                     2.10.1-2                              all          small but fast and easy to use stand-alone template engine

JinjaのUpdate
----

.. NOTE::

  PIPがインストールされていない場合、以下のコマンドを参考にインストールしてください
  $ sudo apt install python3-pip

インストール可能なバージョンの確認 (Versionを指定せず、エラーの内容のVersionを確認します)

.. code-block:: bash
  :linenos:
  :caption: インストール可能バージョンの確認

  $ pip install jinja2==
  ERROR: Could not find a version that satisfies the requirement jinja2== (from versions: 2.0rc1, 2.0, 2.1, 2.1.1, 2.2, 2.2.1, 2.3, 2.3.1, 2.4, 2.4.1, 2.5, 2.5.1, 2.5.2, 2.5.3, 2.5.4, 2.5.5, 2.6, 2.7, 2.7.1, 2.7.2, 2.7.3, 2.8, 2.8.1, 2.9, 2.9.1, 2.9.2, 2.9.3, 2.9.4, 2.9.5, 2.9.6, 2.10, 2.10.1, 2.10.2, 2.10.3, 2.11.0, 2.11.1, 2.11.2, 2.11.3, 3.0.0a1, 3.0.0rc1, 3.0.0rc2, 3.0.0, 3.0.1, 3.0.2, 3.0.3, 3.1.0, 3.1.1, 3.1.2)
  ERROR: No matching distribution found for jinja2==

.. code-block:: bash
  :linenos:
  :caption: jinja2のインストール

  $ pip install jinja2==3.1.2
  Collecting jinja2==3.1.2
    Downloading Jinja2-3.1.2-py3-none-any.whl (133 kB)
       |████████████████████████████████| 133 kB 7.7 MB/s
  Collecting MarkupSafe>=2.0
    Downloading MarkupSafe-2.1.1-cp38-cp38-manylinux_2_17_x86_64.manylinux2014_x86_64.whl (25 kB)
  Installing collected packages: MarkupSafe, jinja2
  Successfully installed MarkupSafe-2.1.1 jinja2-3.1.2

インストールしたパッケージの内容を確認します

.. code-block:: bash
  :linenos:
  :caption: jinja2 バージョン確認

  $ pip list | grep -i jinja
  Jinja2                 3.1.2

.. code-block:: bash
  :linenos:
  :caption: ansible バージョンの確認
  :emphasize-lines: 9

  $ ansible --version
  ansible [core 2.12.8]
    config file = /etc/ansible/ansible.cfg
    configured module search path = ['/home/ubuntu/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
    ansible python module location = /usr/lib/python3/dist-packages/ansible
    ansible collection location = /home/ubuntu/.ansible/collections:/usr/share/ansible/collections
    executable location = /usr/bin/ansible
    python version = 3.8.10 (default, Jun 22 2022, 20:18:18) [GCC 9.4.0]
    jinja version = 3.1.2
    libyaml = True
