---
- name: Setup LAMP (Apache, PHP, MySQL) and configure Master/Slave MySQL settings
  hosts: all
  become: yes

  vars:
    mysql_root_password: "RootPass123!"
    php_user: "user"
    php_user_password: "YourSecurePassword123!"
    src_db_name: "test_db"
    dest_db_name: "app_db"

  tasks:

    # ---------------------------------------------------------------------
    # 1. 必要なミドルウェアのインストール
    # ---------------------------------------------------------------------
    - name: Install PyMySQL
      apt:
        name: python3-pymysql
        state: present
        update_cache: yes

    - name: Install MySQL server
      apt:
        name: mysql-server
        state: present
        update_cache: yes

    - name: Install Apache
      apt:
        name: apache2
        state: present

    - name: Install PHP and modules
      apt:
        name:
          - php
          - php-mysql
          - libapache2-mod-php
        state: present

    # ---------------------------------------------------------------------
    # 2. MySQL サービス起動と初期データベース自動作成
    # ---------------------------------------------------------------------
    - name: Ensure MySQL is started and enabled
      service:
        name: mysql
        state: started
        enabled: yes
      when: "'gcp_mysql_master' in group_names or 'gcp_mysql_slave' in group_names"

    - name: Create app_db database on GCP Master
      command: >
        mysql -u root -e
        "DROP DATABASE IF EXISTS {{ dest_db_name }}; CREATE DATABASE {{ dest_db_name }};"
      when: "'gcp_mysql_master' in group_names"

    # ---------------------------------------------------------------------
    # 3. VMマスターのローカルMySQLから「test_db」を吸い出してGCPへ転送
    # ---------------------------------------------------------------------
    - name: Dump latest live data on VM Master local
      shell: "mysqldump -u root -p'{{ mysql_root_password }}' --skip-lock-tables --single-transaction --no-create-db {{ src_db_name }} > /tmp/local_live_dump.sql"
      delegate_to: localhost
      run_once: true
      become: no

    - name: Copy live DB dump to GCP Master
      copy:
        src: /tmp/local_live_dump.sql
        dest: /tmp/gcp_master_dump.sql
      when: "'gcp_mysql_master' in group_names"

    - name: Import data into app_db on GCP Master
      shell: mysql -u root {{ dest_db_name }} < /tmp/gcp_master_dump.sql
      when: "'gcp_mysql_master' in group_names"
      changed_when: true

    # ---------------------------------------------------------------------
    # 4. 接続ユーザー作成と権限付与（ERROR 1410 回避のクリーン作成方式）
    # ---------------------------------------------------------------------
    - name: Configure PHP MySQL user safely (localhost & 127.0.0.1)
      command: >
        mysql -u root -e
        "DROP USER IF EXISTS '{{ php_user }}'@'localhost';
         DROP USER IF EXISTS '{{ php_user }}'@'127.0.0.1';
         CREATE USER '{{ php_user }}'@'localhost' IDENTIFIED BY '{{ php_user_password }}';
         CREATE USER '{{ php_user }}'@'127.0.0.1' IDENTIFIED BY '{{ php_user_password }}';
         GRANT ALL PRIVILEGES ON {{ dest_db_name }}.* TO '{{ php_user }}'@'localhost';
         GRANT ALL PRIVILEGES ON {{ dest_db_name }}.* TO '{{ php_user }}'@'127.0.0.1';
         FLUSH PRIVILEGES;"
      when: "'gcp_mysql_master' in group_names"

    # ---------------------------------------------------------------------
    # 5. Web アプリケーション配置
    # ---------------------------------------------------------------------
    - name: Restart Apache after PHP install
      service:
        name: apache2
        state: restarted
      when: "'gcp_mysql_master' in group_names or 'gcp_mysql_slave' in group_names"

    - name: Deploy file_search.php
      copy:
        dest: /var/www/html/file_search.php
        content: |
          <?php
          $dbname = 'app_db';
          $username = 'user';
          $password = 'YourSecurePassword123!';

          mysqli_report(MYSQLI_REPORT_OFF);
          $conn = @new mysqli('127.0.0.1', $username, $password, $dbname);
          if ($conn->connect_error) {
              die("GCPサーバ データベース接続失敗: " . $conn->connect_error);
          }

          $search_word = isset($_GET['search']) ? $_GET['search'] : '';

          if ($search_word !== '') {
              $stmt = $conn->prepare("SELECT id, file_name, category, created_at FROM files WHERE file_name LIKE ? OR category LIKE ?");
              $like_word = "%" . $search_word . "%";
              $stmt->bind_param("ss", $like_word, $like_word);
              $stmt->execute();
              $result = $stmt->get_result();
          } else {
              $sql = "SELECT id, file_name, category, created_at FROM files";
              $result = $conn->query($sql);
          }
          ?>
          <!DOCTYPE html>
          <html lang="ja">
          <head>
              <meta charset="UTF-8">
              <title>GCPサーバ 📁 ファイル検索システム (LAMP検証用)</title>
              <style>
                  body { font-family: sans-serif; margin: 40px; background: #f9f9f9; }
                  table { width: 100%; border-collapse: collapse; margin-top: 20px; background: #fff; }
                  th, td { border: 1px solid #ccc; padding: 10px; text-align: left; }
                  th { background: #e0e0e0; }
                  .search-box { margin-bottom: 20px; padding: 15px; background: #eee; border-radius: 5px; }
                  input[type="text"] { padding: 8px; width: 300px; font-size: 16px; }
                  input[type="submit"] { padding: 8px 15px; font-size: 16px; cursor: pointer; }
              </style>
          </head>
          <body>
              <h2>GCPサーバ 📁 ファイル一覧・検索システム</h2>
              <p>Tailscaleプライベートネットワーク経由でのWeb-DB動的連携検証画面</p>
              <div class="search-box">
                  <form action="" method="GET">
                      <input type="text" name="search" placeholder="ファイル名を入力（例: cisco, config）" value="<?php echo htmlspecialchars($search_word, ENT_QUOTES, 'UTF-8'); ?>">
                      <input type="submit" value="検索">
                      <?php if ($search_word !== ''): ?>
                          <a href="file_search.php">クリア</a>
                      <?php endif; ?>
                  </form>
              </div>
              <table>
                  <thead>
                      <tr>
                          <th>ID</th>
                          <th>ファイル名</th>
                          <th>カテゴリ</th>
                          <th>登録日時</th>
                      </tr>
                  </thead>
                  <tbody>
                      <?php if ($result && $result->num_rows > 0): ?>
                          <?php while($row = $result->fetch_assoc()): ?>
                              <tr>
                                  <td><?php echo $row['id']; ?></td>
                                  <td><?php echo htmlspecialchars($row['file_name'], ENT_QUOTES, 'UTF-8'); ?></td>
                                  <td><?php echo htmlspecialchars($row['category'], ENT_QUOTES, 'UTF-8'); ?></td>
                                  <td><?php echo $row['created_at']; ?></td>
                              </tr>
                          <?php endwhile; ?>
                      <?php else: ?>
                          <tr>
                              <td colspan="4">該当するファイルが見つかりません。</td>
                          </tr>
                      <?php endif; ?>
                  </tbody>
              </table>
          </body>
          </html>
          <?php
          if (isset($stmt)) { $stmt->close(); }
          $conn->close();
          ?>
        owner: www-data
        group: www-data
        mode: '0644'
      when: "'gcp_mysql_master' in group_names or 'gcp_mysql_slave' in group_names"
