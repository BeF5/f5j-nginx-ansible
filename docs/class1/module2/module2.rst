Ansible実行環境のセットアップ
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
  sudo apt install software-properties-common -y
  sudo apt-add-repository --yes --update ppa:ansible/ansible
  sudo apt install ansible -y

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
  | 詳細は `Jinja Templateが正しく動作しない場合 <https://f5j-nginx-ansible.readthedocs.io/en/latest/class1/module9/module9.html#jinja-template>`__ を参照してください。

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
