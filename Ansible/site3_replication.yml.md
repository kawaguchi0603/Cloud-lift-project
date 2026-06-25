---
# ============================================================
#  MySQL 8 Replication Setup (GTID Auto-Position Method)
# ============================================================

# ------------------------------------------------------------
#  1. MASTER側：GTID有効化、特権操作、ダンプ回収
# ------------------------------------------------------------
- name: MySQL Replication Setup (Master)
  hosts: gcp_mysql_master
  become: yes

  vars:
    repl_user: "repl"
    repl_password: "Repl1234!"
    mysql_root_password: "RootPass123!"
    app_user: "user"
    app_user_password: "YourSecurePassword123!"
    dest_db_name: "app_db"

  tasks:
    # Master側でGTIDを有効化します
    - name: Enable GTID on Master
      lineinfile:
        path: /etc/mysql/mysql.conf.d/mysqld.cnf
        regexp: "{{ item.regexp }}"
        line: "{{ item.line }}"
      loop:
        - { regexp: '^gtid_mode', line: 'gtid_mode = ON' }
        - { regexp: '^enforce_gtid_consistency', line: 'enforce_gtid_consistency = ON' }
      notify: Restart MySQL Master

    - name: Flush handlers to restart MySQL Master if needed
      meta: flush_handlers

    - name: Create replication user on master
      command: >
        mysql -u root -p'{{ mysql_root_password }}' -e
        "CREATE USER IF NOT EXISTS '{{ repl_user }}'@'%' IDENTIFIED BY '{{ repl_password }}';
         GRANT REPLICATION SLAVE ON *.* TO '{{ repl_user }}'@'%';
         FLUSH PRIVILEGES;"

    - name: Allow remote connection for application user (Tailscale Network)
      command: >
        mysql -u root -p'{{ mysql_root_password }}' -e
        "CREATE USER IF NOT EXISTS '{{ app_user }}'@'100.%' IDENTIFIED BY '{{ app_user_password }}';
         GRANT ALL PRIVILEGES ON {{ dest_db_name }}.* TO '{{ app_user }}'@'100.%';
         FLUSH PRIVILEGES;"

    - name: Dump MySQL databases with master-data
      shell: >
        mysqldump --all-databases --single-transaction --set-gtid-purged=ON
        --flush-logs -u root -p'{{ mysql_root_password }}' > /tmp/master_dump.sql

    - name: Set loose permission on Master dump
      file:
        path: /tmp/master_dump.sql
        mode: '0777'

    - name: Fetch dump file to local control machine safely
      fetch:
        src: /tmp/master_dump.sql
        dest: /tmp/ansible_transfer_dump.sql
        flat: yes

  handlers:
    - name: Restart MySQL Master
      service:
        name: mysql
        state: restarted

# ------------------------------------------------------------
#  2. SLAVE側：GTID有効化、データの流し込み、同期開始
# ------------------------------------------------------------
- name: MySQL Replication Setup (Slave)
  hosts: gcp_mysql_slave
  become: yes

  vars:
    repl_user: "repl"
    repl_password: "Repl1234!"
    mysql_root_password: "RootPass123!"
    master_ip: "100.90.208.68"

  tasks:
    - name: Allow MySQL Slave to listen on all interfaces
      lineinfile:
        path: /etc/mysql/mysql.conf.d/mysqld.cnf
        regexp: '^bind-address'
        line: 'bind-address            = 0.0.0.0'
      notify: Restart MySQL Slave

    # Slave側でもGTIDを有効化します
    - name: Enable GTID on Slave
      lineinfile:
        path: /etc/mysql/mysql.conf.d/mysqld.cnf
        regexp: "{{ item.regexp }}"
        line: "{{ item.line }}"
      loop:
        - { regexp: '^gtid_mode', line: 'gtid_mode = ON' }
        - { regexp: '^enforce_gtid_consistency', line: 'enforce_gtid_consistency = ON' }
        - { regexp: '^server-id', line: 'server-id = 2' } # ⭐これを追加
      notify: Restart MySQL Slave

    - name: Flush handlers to restart MySQL Slave if needed
      meta: flush_handlers

    - name: Copy dump file to Slave
      copy:
        src: /tmp/ansible_transfer_dump.sql
        dest: /tmp/master_dump_slave.sql
        mode: '0644'

    - name: Stop replica if running
      command: mysql -u root -p'{{ mysql_root_password }}' -e "STOP REPLICA;"
      ignore_errors: yes

    - name: Reset replica
      command: mysql -u root -p'{{ mysql_root_password }}' -e "RESET REPLICA ALL;"
      ignore_errors: yes

    # スレーブ側の古いGTID実行履歴を完全にクリーンアップします
    - name: Reset binary logs and GTID history on slave
      command: mysql -u root -p'{{ mysql_root_password }}' -e "RESET MASTER;"
      ignore_errors: yes

    - name: Import dump into slave
      shell: "mysql -u root -p'{{ mysql_root_password }}' < /tmp/master_dump_slave.sql"

    - name: Configure replication source on slave (MySQL 8 GTID Auto Syntax)
      command: >
        mysql -u root -p'{{ mysql_root_password }}' -e
        "CHANGE REPLICATION SOURCE TO
         SOURCE_HOST='{{ master_ip }}',
         SOURCE_USER='{{ repl_user }}',
         SOURCE_PASSWORD='{{ repl_password }}',
         SOURCE_AUTO_POSITION=1,
         GET_SOURCE_PUBLIC_KEY=1;"

    - name: Start replica
      command: mysql -u root -p'{{ mysql_root_password }}' -e "START REPLICA;"

  handlers:
    - name: Restart MySQL Slave
      service:
        name: mysql
        state: restarted

# ------------------------------------------------------------
#  3. ロードバランサー（Nginx）の設定
# ------------------------------------------------------------
- name: Configure Nginx Load Balancer
  hosts: gcp_nginx_lb
  become: yes

  tasks:
    - name: Disable default Apache on LB node (to free port 80)
      service:
        name: apache2
        state: stopped
        enabled: no
      ignore_errors: yes

    - name: Install Nginx
      apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Configure Nginx Reverse Proxy with Failover
      copy:
        dest: /etc/nginx/sites-available/default
        content: |
          upstream backend {
              # マスターが応答しなければ1秒で諦めてスレーブに回す設定
              server gcp-mysql-master:80 max_fails=1 fail_timeout=5s;
              server gcp-mysql-slave:80 max_fails=1 fail_timeout=5s;
          }

          server {
              listen 80 default_server;
              listen [::]:80 default_server;

              location / {
                  proxy_pass http://backend;
                  proxy_connect_timeout 1s;
                  proxy_read_timeout 5s;
                  proxy_send_timeout 5s;

                  proxy_set_header Host $host;
                  proxy_set_header X-Real-IP $remote_addr;
                  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
              }
          }
        owner: root
        group: root
        mode: '0644'
      notify: Restart Nginx

  handlers:
    - name: Restart Nginx
      service:
        name: nginx
        state: restarted
