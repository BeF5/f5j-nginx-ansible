NGINXのセットアップ
####

1. 事前確認
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

2. NGINX Plus、NAP WAF/DoSのインストール
====

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
  ansible-playbook -i inventories/hosts -l nginx1 playbook/deploy-nginx-plus-app-protect-waf-dos.yaml --private-key="~/ssh_key/id_rsa"  --become

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

3. NGINX Plus、NAP WAF/DoSのアンインストール
====

ライセンス、Playbookなど正しく配置していることを想定し説明を進めます。
確認が必要な場合 `1. 事前確認 <https://f5j-nginx-ansible.readthedocs.io/en/latest/class1/module3/module3.html#id2>`__ を参照してください。


Playbookの内容を確認します

.. code-block:: cmdin

  ## cd ~/f5j-nginx-ansible-lab
  cat playbook/remove-nginx-plus-app-protect-waf-dos.yaml

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 8,21,17,27

  ---
  - hosts: all
    collections:
      - nginxinc.nginx_core
    tasks:
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


- 8行目で ``nginx`` のロールを指定し、20行目で ``nginx_setup`` で ``uninstall`` を指定します
- 21行目で ``nginx_app_protect`` のロールを指定し、23行目で ``nginx_app_protect_waf_setup`` 24行目 ``uninstnginx_app_protect_dos_setupall`` で ``uninstall`` を指定します
- 17行目で ``nginx_start: false`` 、 27行目で ``nginx_app_protect_start: false`` としています。このパラメータによりアンインストール後のプロセス再起動の動作を回避します


NGINX Plus、NGINX App Protect WAF/DoS をアンインストール

.. code-block:: cmdin

  ## cd ~/f5j-nginx-ansible-lab
  ansible-playbook -i inventories/hosts -l nginx1 playbook/remove-nginx-plus-app-protect-waf-dos.yaml --private-key="~/ssh_key/id_rsa"  --become

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
  10.1.1.7                   : ok=33   changed=10   unreachable=0    failed=0    skipped=39   rescued=0    ignored=1

- 13-15行目で、nginx_app_protect ロールで設定ファイルの書式をチェックする ``nginx -t`` が実行されていますが、NGINX Plusがアンインストールされているためコマンドが正常に完了しません。こちらは無視してよい動作です
- 21行目で、実行結果が表示されます。 13-15行目の実行結果が ``ignore`` として表示されています

インストール時に確認したコマンドで状態を確認すると以下のようになります。

.. code-block:: bash
  :linenos:
  :caption: 確認結果サンプル

  $ cat /opt/app_protect/VERSION
  cat: /opt/app_protect/VERSION: No such file or directory

  $ nginx -v
  
  Command 'nginx' not found, but can be installed with:
  
  sudo apt install nginx-core    # version 1.18.0-0ubuntu1.3, or
  sudo apt install nginx-extras  # version 1.18.0-0ubuntu1.3
  sudo apt install nginx-full    # version 1.18.0-0ubuntu1.3
  sudo apt install nginx-light   # version 1.18.0-0ubuntu1.3


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