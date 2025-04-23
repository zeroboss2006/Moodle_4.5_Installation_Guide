# Moodle 4.5 安裝指南

本專案說明如何在 Ubuntu 24.04 環境中部署 Moodle 4.5，搭配 Apache、MariaDB、PHP 8.3，並使用 Let's Encrypt 建立 HTTPS。文末也提供如何將 Moodle 網址從 `moodle-a.lab.com` 轉換為 `moodle-b.lab.com` 的指令。

---

## ✨ 環境資訊

- OS：Ubuntu 24.04.1 LTS
- 網站主機：`https://moodle-a.lab.com`
- 備用轉址：`https://moodle-b.lab.com`
- PHP：8.3
- Apache：2.4+
- 資料庫：MariaDB 支援
- Moodle 版本：4.5

---

## ✅ 安裝流程

### 1️⃣ 安裝必要套件

```bash
sudo apt update && sudo apt install apache2 mariadb-server php php-{cli,fpm,gd,intl,mbstring,xml,xmlrpc,curl,soap,zip,mysql} unzip git -y
```

---

### 2️⃣建立 Moodle 資料庫與使用者
```bash
sudo mysql -u root -p
```
```sql
CREATE DATABASE moodle DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'moodlelabuser'@'localhost' IDENTIFIED BY 'Jq82Vx1tTg!#';
GRANT ALL PRIVILEGES ON moodle.* TO 'moodlelabuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

---

### 3️⃣ 下載並部署 Moodle 原始碼與設定目錄權限
```bash
cd /opt
wget https://download.moodle.org/download.php/direct/stable405/moodle-4.5.tgz
tar -xvzf moodle-4.5.tgz
sudo mv moodle /var/www/html/moodle
sudo chown -R www-data:www-data /var/www/html/moodle
sudo chmod -R 755 /var/www/html/moodle
```

---

### 4️⃣ 建立 Moodle 資料目錄與權限
```bash
sudo mkdir /var/moodledata
sudo chown -R www-data:www-data /var/moodledata
sudo chmod -R 770 /var/moodledata
```

---

### 5️⃣ 設定 Apache 處理 HTTP 端口
#### 🔹 HTTP 設定檔：/etc/apache2/sites-available/moodle.conf
```apache
<VirtualHost *:80>
    ServerName moodle-a.lab.com
    DocumentRoot /var/www/html/moodle

    # 自動轉導至 HTTPS
    Redirect permanent / https://moodle-a.lab.com/
</VirtualHost>
```

#### 啟用處理端口：

```bash
sudo a2ensite moodle
sudo systemctl reload apache2
```

---

### 6️⃣ 安裝與啟用 SSL (Let's Encrypt)
```bash
sudo apt install certbot python3-certbot-apache -y
sudo certbot --apache -d moodle-a.lab.com
```

Certbot 會自動產生 /etc/apache2/sites-available/moodle-le-ssl.conf，例如：

🔹 HTTPS 設定檔：/etc/apache2/sites-available/moodle-le-ssl.conf
```apache
<IfModule mod_ssl.c>
<VirtualHost *:443>
    ServerName moodle-a.lab.com
    DocumentRoot /var/www/html/moodle

    <Directory /var/www/html/moodle>
        Options +FollowSymlinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/moodle_error.log
    CustomLog ${APACHE_LOG_DIR}/moodle_access.log combined

    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/moodle-a.lab.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/moodle-a.lab.com/privkey.pem
    Include /etc/letsencrypt/options-ssl-apache.conf
</VirtualHost>
</IfModule>
```

---

### 7️⃣ 瀏覽器開始安裝流程
進入網站開始設定：

```
https://moodle-a.lab.com
```
照指引完成安裝，資料目錄請指定為 /var/moodledata，並輸入下列資料庫資訊：
```
使用者：moodlelabuser

密碼：Jq82Vx1tTg!#

資料庫：moodle

主機：localhost
```

⚙️ 安裝後設定修正
```bash
sudo nano /etc/php/8.3/apache2/php.ini
```

修改下列內容：
```ini
max_input_vars = 5000
```

套用設定：
```bash
sudo systemctl restart apache2
```

資料庫完整 UTF-8 支援
修改 /etc/mysql/mariadb.conf.d/50-server.cnf：
```ini
[mysqld]
innodb_file_format = Barracuda
innodb_file_per_table = 1
innodb_large_prefix = ON
```

重啟服務：
```bash
sudo systemctl restart mariadb
```

### 🔁 網址轉換：將 moodle-a.lab.com 轉為 moodle-b.lab.com
如將站台搬移或變更 DNS

#### ✅ 步驟一：設定 DNS 與 SSL 憑證
DNS 設定：
到你的 DNS 提供商（如 Cloudflare、gandi、Namecheap 等）新增設定一筆 A 記錄，指向：
```
moodle-b.lab.com → 目前的moodle-a.lab.com IP位置
```

SSL 憑證（Let’s Encrypt）：
使用 Certbot 產生新憑證（或更新）：
```bash
sudo certbot --apache -d test-minmax.byi-e.com
```

#### ✅ 步驟二：修改 Apache 設定
編輯原本的 Apache 設定檔 /etc/apache2/sites-available/moodle.conf：
```bash
sudo nano /etc/apache2/sites-available/moodle.conf
```

將裡面的 ServerName 改為新的網址：
```moodle.conf
<VirtualHost *:80>
    ServerName test-minmax.byi-e.com
    ...
</VirtualHost>
```

如果你也啟用了 SSL，檢查 /etc/apache2/sites-available/moodle-le-ssl.conf，也要改：
```bash
sudo nano /etc/apache2/sites-available/moodle-le-ssl.conf
```

```moodle-le-ssl.conf
<VirtualHost *:443>
    ServerName test-minmax.byi-e.com
    ...
</VirtualHost>
```

重新啟動 Apache：
```bash
sudo systemctl reload apache2
```

#### ✅ 步驟三：修改 Moodle 設定
1. 編輯 config.php
```bash
sudo nano /var/www/html/moodle/config.php
```

找到以下這行：
```php
$CFG->wwwroot   = 'https://minmax.byi-e.com';
```

改為：
```php
$CFG->wwwroot   = 'https://test-minmax.byi-e.com';
```
儲存後離開。

#### ✅ 步驟四：清除快取與重建設定
進入 Moodle 網站根目錄執行以下指令清快取：
```bash
sudo -u www-data /usr/bin/php /var/www/html/moodle/admin/cli/purge_caches.php
```

#### ✅ 步驟五：更新資料庫中的 URL（如果有內容中包含原網址）
如果你之前有上傳課程或頁面內文含舊網址，你需要替換 DB 中的網址。

建議使用 Moodle 官方的搜尋與取代工具：

#### ✅ 請務必先備份資料庫：
```bash
mysqldump -u moodlelabuser -p moodle > ~/moodle_backup_before_url_replace.sql
```

##### ✅ 方法一：使用 --shortenurls 參數（Moodle 4.2 以上支援）
```bash
sudo -u www-data php /var/www/html/moodle/admin/tool/replace/cli/replace.php \
--search=https://moodle-a.lab.com \
--replace=https://moodle-b.lab.com \
--non-interactive
```

##### ✅ 方法二：修改資料庫內網址（進階）

進入 MySQL 資料庫
你已知的資料庫連線資訊如下：
```
使用者：moodlelabuser

密碼：Jq82Vx1tTg!#

資料庫名稱：moodle
```

請使用以下指令登入 MySQL：
```bash
mysql -u moodlelabuser -p moodle
```

系統會提示你輸入密碼，輸入：
```
Jq82Vx1tTg!#
```

登入後你會看到類似這樣的提示符號：
```sql
mysql>
```

✅ 執行 SQL 更新語句
登入後，依序貼上下列每一條指令（可以整段貼入）：
```sql
UPDATE mdl_config SET value = REPLACE(value, 'https://moodle-a.lab.com', 'https://moodle-b.lab.com');
UPDATE mdl_course_sections SET summary = REPLACE(summary, 'https://moodle-a.lab.com', 'https://moodle-b.lab.com');
UPDATE mdl_label SET intro = REPLACE(intro, 'https://moodle-a.lab.com', 'https://moodle-b.lab.com');
UPDATE mdl_page SET content = REPLACE(content, 'https://moodle-a.lab.com', 'https://moodle-b.lab.com');
```

每執行一行都會顯示類似：
```sql
Query OK, 2 rows affected (0.02 sec)
Rows matched: 2  Changed: 2  Warnings: 0
```

✅ 離開 MySQL：
```sql
exit
```

---

### 🆙 cron 排程說明（補充）
可以在最後補上：

```bash
sudo crontab -u www-data -e
```
加入：

```cron
*/1 * * * * /usr/bin/php /var/www/html/moodle/admin/cli/cron.php >/dev/null 2>&1
```

## ⚠️ 安全性提醒
本文使用的資料庫帳號 moodlelabuser 及密碼 Jq82Vx1tTg!# 僅為範例，實際部署時請務必修改為強密碼，以保障伺服器安全。

## ⚠️ 本文件為非官方社群指南，與原版 Moodle 專案無任何關聯，僅作為學習與部署參考用途。
