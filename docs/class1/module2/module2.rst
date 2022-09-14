NGINX Plus のセットアップ
####

Ansibleを使ってNGINXの環境をセットアップします

1. Ansibleとは？
====

- .. image:: ./media/ansible_explanation.jpg
    :width: 400

NGINXのセットアップはAnsible Galaxyというサイトを通じて、NGINXが提供するモジュールを利用します

NGINXは ``nginx_core`` というコレクションを提供しています。詳細は以下URLを参照してください

- `Ansible Galaxy nginx_core <https://galaxy.ansible.com/nginxinc/nginx_core>`__


2. 環境セットアップ
====

| Ansibleをインストールします。ラボ環境ではUbuntuを利用しておりますので手順に従ってインストールします。
| (その他のLinuxディストリビューションをご利用の場合は、各ディストリビューションの項目を参照してください)

- `Ubuntu への Ansible のインストール <https://docs.ansible.com/ansible/2.9_ja/installation_guide/intro_installation.html#ubuntu-ansible>`__

.. code-block:: cmdin

  sudo apt update
  sudo apt install software-properties-common
  sudo apt-add-repository --yes --update ppa:ansible/ansible
  sudo apt install ansible

Ansibleが正しくインストールされたことを確認します

.. code-block:: cmdin

  ansible --version

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  ansible [core 2.12.8]
    config file = /etc/ansible/ansible.cfg
    configured module search path = ['/home/ubuntu/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
    ansible python module location = /usr/lib/python3/dist-packages/ansible
    ansible collection location = /home/ubuntu/.ansible/collections:/usr/share/ansible/collections
    executable location = /usr/bin/ansible
    python version = 3.8.10 (default, Jun 22 2022, 20:18:18) [GCC 9.4.0]
    jinja version = 2.10.1
    libyaml = True

.. NOTE::

  | Ansibleでjinja templateを利用し、NGINXの設定を行う場合、エラーとなる場合があります。
  | エラーを回避するためには、Pythonのパッケージマネージャである ``pip`` をインストールし、 ``jinja2`` のパッケージをアップデートする必要があります。
  | 詳細は `Jinja Templateが正しく動作しない場合 <https://f5j-nginx-ansible.readthedocs.io/en/latest/class1/module3/module3.html>`__ を参照してください。

AnsibleのNGINX Coreコレクションを取得します

.. code-block:: cmdin

  ansible-galaxy collection install nginxinc.nginx_core

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  Starting galaxy collection install process
  Process install dependency map
  Starting collection install process
  Downloading https://galaxy.ansible.com/download/nginxinc-nginx_core-0.6.0.tar.gz to /home/ubuntu/.ansible/tmp/ansible-local-3312k9xqsukx/tmp5639w5je/nginxinc-nginx_core-0.6.0-ctsyccb0
  Installing 'nginxinc.nginx_core:0.6.0' to '/home/ubuntu/.ansible/collections/ansible_collections/nginxinc/nginx_core'
  nginxinc.nginx_core:0.6.0 was installed successfully


コマンドを実行することにより以下のロールがインストールされます。

+--------------------------+----------------------------------------+--------+
|Name                      |Description                             |Version |
+==========================+========================================+========+
|nginxinc.nginx            |Install NGINX                           |0.23.1  |
+--------------------------+----------------------------------------+--------+
|nginxinc.nginx_config     |Configure NGINX                         |0.5.1   |
+--------------------------+----------------------------------------+--------+
|nginxinc.nginx_app_protect|Install and configure NGINX App Protect |0.8.0   |
+--------------------------+----------------------------------------+--------+

.. NOTE::

  記載のVersionは資料作成時点 ``0.6.0`` の内容となります。最新情報はAnsible Galaxyのページを確認ください


以下のパスにファイルが取得されていることを確認します

.. code-block:: cmdin

  ls ~/.ansible/collections/ansible_collections/nginxinc/nginx_core/roles/*

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  .ansible/collections/ansible_collections/nginxinc/nginx_core/roles/nginx:
  CHANGELOG.md        LICENSE    files     molecule   vars
  CODE_OF_CONDUCT.md  README.md  handlers  tasks
  CONTRIBUTING.md     defaults   meta      templates
  
  .ansible/collections/ansible_collections/nginxinc/nginx_core/roles/nginx_app_protect:
  CHANGELOG.md        LICENSE    files     meta      templates
  CODE_OF_CONDUCT.md  README.md  handlers  molecule  vars
  CONTRIBUTING.md     defaults   images    tasks
  
  .ansible/collections/ansible_collections/nginxinc/nginx_core/roles/nginx_config:
  CHANGELOG.md        LICENSE    files     molecule   vars
  CODE_OF_CONDUCT.md  README.md  handlers  tasks
  CONTRIBUTING.md     defaults   meta      templates


3. NGINX のセットアップ
====

ライセンスファイルが配置されていることを確認してください。
ファイルが配置されていない場合、トライアルを申請し証明書と鍵を取得してください

.. code-block:: cmdin
   
  ## cd ~/
  ls ~/nginx-repo.*

NGINXは Ansible Playbookの設定サンプルをGitHubで公開しています。ファイルを取得します。

.. code-block:: cmdin

  ## cd ~/
  git clone https://github.com/BeF5/f5j-nginx-ansible-lab.git
  cd ~/f5j-nginx-ansible-lab

Inventoryの情報を以下コマンドで確認します

.. code-block:: cmdin

  ## cd ~/f5j-nginx-ansible-lab
  ansible-inventory -i inventories/hosts --graph

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  @all:
    |--@nginx1:
    |  |--ansible_host:10.1.1.7
    |--@nginx2:
    |  |--ansible_host:10.1.1.6
    |--@ungrouped:

``nginx1`` と ``nginx2`` という2つのホストが登録されていることが確認できます

Playbookの内容を確認します

.. code-block:: cmdin

  ## cd ~/f5j-nginx-ansible-lab
  cat playbook/deploy-nginx-plus-app-protect-waf-dos.yaml

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 4,8,18

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

- 4行目で ``nginxinc.nginx_core`` のコレクションを指定します

- 8行目で ``nginx`` のロールを指定し、10-14行目でパラメータを指定します

  - 10行目 ``nginx_type`` : InstallするNGINXをOpenSourceかPlusか指定します
  - 11行目 ``nginx_license`` : NGINX Plusに必要となる証明書・鍵を指定します
  - 14行目 ``nginx_remove_license`` : インストール後ライセンスファイルの削除を指定します
  - その他パラメータは `NGINX installation variables <https://github.com/nginxinc/ansible-role-nginx/blob/main/defaults/main/main.yml>`__ を参照してください

- 18行目で ``nginx_app_protect`` のロールを指定し、20-25行目でパラメータを指定します

  - 20行目 ``nginx_app_protect_waf_enable`` : NGINX App Protect WAF をインストールします
  - 21行目 ``nginx_app_protect_dos_enable`` : NGINX App Protect DoS をインストールします
  - 22-23行目 ``nginx_app_protect_*`` : WAFのSignature、Threat Campaign Signatureをインストールします
  - 24-25 ``nginx_app_protect_*_license`` : ライセンスの利用、インストール後のライセンスファイルの削除を指定します
  - その他パラメータは `NGINX App Protect installation and configuration variables <https://github.com/nginxinc/ansible-role-nginx-app-protect/blob/main/defaults/main.yml>`__ を参照してください

NGINX Plus、NGINX App Protect WAF/DoS をインストール

.. code-block:: cmdin

  ## cd ~/f5j-nginx-ansible-lab
  ansible-playbook -i inventories/hosts -l nginx1 playbook/deploy-nginx-plus-app-protect-waf-dos.yaml --private-key="/home/ubuntu/id_rsa"  --become

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  PLAY [all] *************************************************************************************************************************************************************
  
  TASK [Gathering Facts] *************************************************************************************************************************************************
  ok: [10.1.1.7]
  
  TASK [Install NGINX Plus] **********************************************************************************************************************************************
  
  ** 省略 **

  PLAY RECAP *************************************************************************************************************************************************************
  10.1.1.7                   : ok=49   changed=22   unreachable=0    failed=0    skipped=45   rescued=0    ignored=0

実行したコマンドのオプションの指定パラメータは以下です

.. code-block:: bash
  :linenos:
  :caption: ansible-playbook コマンドのサンプル

  ansible-playbook <option> <playbook file path>

+--------------+-------------------------------------------------------------------+
|option        |用途・役割                                                         |
+==============+===================================================================+
|-i            |実行対象となるインベントリファイルを指定します                     |
+--------------+-------------------------------------------------------------------+
|-l (--limit)  |インベントリの対象となるホストをフィルタで指定します               |
+--------------+-------------------------------------------------------------------+
|--private-key |ホストに接続する際に利用する鍵ファイルのパスを指定します           |
+--------------+-------------------------------------------------------------------+
|--become (-b) |becomeで操作を実行します。権限昇格方法のデフォルトは ``sudo`` です |
+--------------+-------------------------------------------------------------------+

インストールしたパッケージの情報の確認します

| 参考となる記事はこちらです。
| `K72015934: Display the NGINX software version <https://support.f5.com/csp/article/K72015934>`__

.. code-block:: cmdin

  nginx -v

NGINX App Protect のVersion

.. code-block:: cmdin

  cat /opt/app_protect/VERSION

NGINX App Protect DoS のVersion

.. code-block:: cmdin

  admd -v

その他インストールしたパッケージの情報を確認いただけます。ラボ環境のホストはUbuntuとなります。

.. code-block:: cmdin

  dpkg-query -l | grep nginx-plus

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  ii  nginx-plus                         25-1~focal                            amd64        NGINX Plus, provided by Nginx, Inc.
  ii  nginx-plus-module-appprotect       25+3.671.0-1~focal                    amd64        NGINX Plus app protect dynamic module version 3.671.0
  ii  nginx-plus-module-appprotectdos    25+2.0.1-1~focal                      amd64        NGINX Plus appprotectdos dynamic module

.. code-block:: cmdin

  dpkg-query -l | grep app-protect

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  ii  app-protect                        25+3.671.0-1~focal                    amd64        App-Protect package for Nginx Plus, Includes all of the default files and examples. Nginx App Protect provides web application firewall (WAF) security protection for your web applications, including OWASP Top 10 attacks.
  ii  app-protect-attack-signatures      2021.11.16-1~focal                    amd64        Attack Signature Updates for App-Protect
  ii  app-protect-common                 8.12.1-1~focal                        amd64        NGINX App Protect
  ii  app-protect-compiler               8.12.1-1~focal                        amd64        Control-plane(aka CP) for waf-general debian
  ii  app-protect-dos                    25+2.0.1-1~focal                      amd64        Nginx DoS protection
  ii  app-protect-engine                 8.12.1-1~focal                        amd64        NGINX App Protect
  ii  app-protect-plugin                 3.671.0-1~focal                       amd64        NGINX App Protect plugin

4. NGINX の削除
====

同様の手順でNGINXを削除することが可能です

ライセンスファイルが配置されていることを確認してください。
ファイルが配置されていない場合、トライアルを申請し証明書と鍵を取得してください

Playbookの内容を確認します

.. code-block:: cmdin

  ## cd ~/f5j-nginx-ansible-lab
  cat playbook/remove-nginx-plus-app-protect-waf-dos.yaml

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 8,18,31,27,37

  ---
  - hosts: all
    collections:
      - nginxinc.nginx_core
    tasks:
      - name: CleanUp nginx.conf for Uninstall
        ansible.builtin.include_role:
          name: nginx_config
        vars:
          nginx_config_main_template_enable: true
          nginx_config_main_template:
            template_file: nginx.conf.j2
            deployment_location: /etc/nginx/nginx.conf
            backup: true
  
      - name: Uniinstall NGINX Plus
        ansible.builtin.include_role:
          name: nginx
        vars:
          nginx_type: plus
          nginx_setup: uninstall
          nginx_license:
            certificate: ~/nginx-repo.crt
            key: ~/nginx-repo.key
          nginx_remove_license: false
          nginx_start: false
  
      - name: Uninstall NGINX App Protect WAF/DoS
        ansible.builtin.include_role:
          name: nginx_app_protect
        vars:
          nginx_app_protect_waf_setup: uninstall
          nginx_app_protect_dos_setup: uninstall
          nginx_app_protect_setup_license: false
          nginx_app_protect_remove_license: false
          nginx_app_protect_start: false



- 8行目で ``nginx_config`` のロールを指定し、10-14行目でパラメータを指定します

  - 10-14行目 : ``/etc/nginx/`` に ``nginx.conf`` を作成します。アンインストールのエラーを回避するため空の設定ファイルとします

- 18行目で ``nginx`` のロールを指定し、20行目で ``nginx_setup`` で ``uninstall`` を指定します
- 31行目で ``nginx_app_protect`` のロールを指定し、33行目で ``nginx_app_protect_waf_setup`` 34行目 ``uninstnginx_app_protect_dos_setupall`` で ``uninstall`` を指定します
- 27行目で ``nginx_start: false`` 、 37行目で ``nginx_app_protect_start: false`` としています。このパラメータによりアンインストール後のプロセス再起動の動作を回避します


NGINX Plus、NGINX App Protect WAF/DoS をアンインストール

.. code-block:: cmdin

  ## cd ~/f5j-nginx-ansible-lab
  ansible-playbook -i inventories/hosts -l nginx1 playbook/remove-nginx-plus-app-protect-waf-dos.yaml --private-key="/home/ubuntu/id_rsa"  --become

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 13-15,21

  PLAY [all] *************************************************************************************************************************************************************
  
  TASK [Gathering Facts] *************************************************************************************************************************************************
  ok: [10.1.1.7]
  
  TASK [CleanUp nginx.conf for Uninstall] ********************************************************************************************************************************

  ** 省略 **
  
  RUNNING HANDLER [nginxinc.nginx_core.nginx_app_protect : (Handler - NGINX App Protect) Restart NGINX] ******************************************************************************************************************************************
  skipping: [10.1.1.7]
  
  RUNNING HANDLER [nginxinc.nginx_core.nginx_app_protect : (Handler - NGINX App Protect) Check NGINX] ********************************************************************************************************************************************
  fatal: [10.1.1.7]: FAILED! => {"changed": false, "cmd": "nginx -t", "msg": "[Errno 2] No such file or directory: b'nginx'", "rc": 2, "stderr": "", "stderr_lines": [], "stdout": "", "stdout_lines": []}
  ...ignoring
  
  RUNNING HANDLER [nginxinc.nginx_core.nginx_app_protect : (Handler - NGINX App Protect) Print NGINX error if syntax check fails] ****************************************************************************************************************
  skipping: [10.1.1.7]
  
  PLAY RECAP *************************************************************************************************************************************************************************************************************************************
  10.1.1.7                   : ok=38   changed=12   unreachable=0    failed=0    skipped=55   rescued=0    ignored=1


- 13-15行目で、nginx_app_protect ロールで設定ファイルの書式をチェックする ``nginx -t`` が実行されていますが、NGINX Plusがアンインストールされているためコマンドが正常に完了しません。こちらは無視してよい動作です
- 21行目で、実行結果が表示されます。 13-15行目の実行結果が ``ignore`` として表示されています

インストール時に確認したコマンドで状態を確認すると以下のようになります。

.. code-block:: bash
  :linenos:
  :caption: 確認結果サンプル

  $ cat /opt/app_protect/VERSION
  cat: /opt/app_protect/VERSION: No such file or directory

  $ nginx -v
  -bash: /usr/sbin/nginx: No such file or directory

  $ dpkg-query -l | grep nginx-plus
  rc  nginx-plus                         27-1~focal                            amd64        NGINX Plus, provided by Nginx, Inc.

  $ dpkg-query -l | grep app-protect
  rc  app-protect                        27+3.954.0-1~focal                    amd64        App-Protect package for Nginx Plus, Includes all of the default files and examples. Nginx App Protect provides web application firewall (WAF) security protection for your web applications, including OWASP Top 10 attacks.
  ii  app-protect-common                 10.87.0-1~focal                       amd64        NGINX App Protect
  rc  app-protect-compiler               10.87.0-1~focal                       amd64        Control-plane(aka CP) for waf-general debian
  rc  app-protect-dos                    27+2.4.1-1~focal                      amd64        Nginx DoS protection
  rc  app-protect-engine                 10.87.0-1~focal                       amd64        NGINX App Protect
  rc  app-protect-plugin                 3.954.0-1~focal                       amd64        NGINX App Protect plugin
  

プログラムとして ``app-protect-common`` は残った状態になりますので、完全に削除されたい場合には以下のコマンドを参考に削除してください。

.. code-block:: cmdin

  sudo apt remove app-protect-common

Ansibleでアンインストールを行った場合、NGINX Plusの設定ファイルが残ります。完全に削除する場合には以下コマンドを参考に削除してください。

.. code-block:: cmdin

  sudo apt remove nginx-plus --purge

NGINX Plusを完全に削除した後、 ``/etc/nginx`` フォルダが削除されます

.. code-block:: bash
  :linenos:
  :caption: 確認結果サンプル

  $ ls /etc/nginx
  ls: cannot access '/etc/nginx': No such file or directory


Tips1. Ansible Galaxy 各種コマンド
====

コレクションの一覧表示

.. code-block:: cmdin

  ansible-galaxy collection list nginxinc.nginx_core

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  # /home/ubuntu/.ansible/collections/ansible_collections
  Collection          Version
  ------------------- -------
  nginxinc.nginx_core 0.6.0


Authorを指定したロールの検索

.. code-block:: cmdin

  ansible-galaxy search --author nginx

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  Found 20 roles matching your search:
  
   Name                                            Description
   ----                                            -----------
   nginxinc.nginx                                  Official Ansible role for NGINX
   nginxinc.nginx_app_protect                      Official Ansible role for NGINX App Protect WAF and DoS
   nginxinc.nginx_config                           Official Ansible role for configuring NGINX
   nginxinc.nginx_controller_agent                 A role to install, configure, and upgrade the NGINX Controller agen>
   nginxinc.nginx_controller_api_definition_import A role to import Open API definitions to NGINX Controller
   nginxinc.nginx_controller_application           A role to define applications (apps) with NGINX Controller.
   nginxinc.nginx_controller_certificate           A role to upsert (create and update) certificates to NGINX Controll>
   nginxinc.nginx_controller_component             A role to define application components with NGINX Controller.
   nginxinc.nginx_controller_environment           A role to define environments within NGINX Controller.
   nginxinc.nginx_controller_forwarder             A role to define / update data forwarders within NGINX Controller.
   nginxinc.nginx_controller_gateway               A role to upsert (create and update) gateways in NGINX Controller t>
   nginxinc.nginx_controller_generate_token        A role to generate an NGINX Controller authentication token.
   nginxinc.nginx_controller_install               Official Ansible role for installing NGINX Controller
   nginxinc.nginx_controller_integration           A role to define / update integrations within NGINX Controller.
   nginxinc.nginx_controller_license               A role to push an NGINX Controller license to your NGINX Controller>
   nginxinc.nginx_controller_location              A role to define locations within NGINX Controller.
   nginxinc.nginx_controller_publish_api           A role to upsert (create and update) the configurations to publish >
   nginxinc.nginx_controller_user                  A role to define users within NGINX Controller.
   nginxinc.nginx_controller_user_role             A role to define user roles within NGINX Controller.
   nginxinc.nginx_unit                             Official Ansible role for NGINX Unit

キーワード、Authorを指定したロールの検索

.. code-block:: cmdin

  ansible-galaxy search development --author nginx

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  Found 5 roles matching your search:
  
   Name                                  Description
   ----                                  -----------
   nginxinc.nginx                        Official Ansible role for NGINX
   nginxinc.nginx_app_protect            Official Ansible role for NGINX App Protect WAF and DoS
   nginxinc.nginx_config                 Official Ansible role for configuring NGINX
   nginxinc.nginx_controller_environment A role to define environments within NGINX Controller.
   nginxinc.nginx_unit                   Official Ansible role for NGINX Unit


Tips2. Inventory情報確認コマンド
====

Graph形式でのInventory情報の表示

.. code-block:: cmdin

  ansible-inventory -i inventories/hosts --graph

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  @all:
    |--@nginx1:
    |  |--10.1.1.7
    |--@nginx2:
    |  |--10.1.1.6
    |--@ungrouped:


List形式でのInventory情報の表示

.. code-block:: cmdin

  ansible-inventory -i inventories/hosts --list

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  {
      "_meta": {
          "hostvars": {}
      },
      "all": {
          "children": [
              "nginx1",
              "nginx2",
              "ungrouped"
          ]
      },
      "nginx1": {
          "hosts": [
              "10.1.1.7"
          ]
      },
      "nginx2": {
          "hosts": [
              "10.1.1.6"
          ]
      }
  }
