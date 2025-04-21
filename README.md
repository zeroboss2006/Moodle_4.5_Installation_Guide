# Moodle 4.5 安裝指南（主機：moodle-a.lab.com）

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
```sql
CREATE DATABASE moodle DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'moodlelabuser'@'localhost' IDENTIFIED BY 'Jq82Vx1tTg!#';
GRANT ALL PRIVILEGES ON moodle.* TO 'moodlelabuser'@'localhost';
FLUSH PRIVILEGES;
```

---

### 3️⃣ 下載並部署 Moodle 原始碼
```bash
cd /opt
wget https://download.moodle.org/download.php/direct/stable405/moodle-4.5.tgz
tar -xvzf moodle-4.5.tgz
sudo mv moodle /var/www/moodle
```

---

### 4️⃣ 建立 Moodle 資料目錄
```bash
sudo mkdir /var/moodledata
sudo chown -R www-data:www-data /var/moodledata
```

---

### 5️⃣ 設定 Apache 處理端口
## 🔹 HTTP 設定檔：/etc/apache2/sites-available/moodle.conf
```apache
<VirtualHost *:80>
    ServerName moodle-a.lab.com
    DocumentRoot /var/www/moodle

    # 自動轉導至 HTTPS
    Redirect permanent / https://moodle-a.lab.com/
</VirtualHost>
```

### 啟用處理端口：

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
    DocumentRoot /var/www/moodle

    <Directory /var/www/moodle>
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

```arduino
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
如將站台搬移或變更 DNS，可使用 CLI 工具進行網址全站替換：
```bash
sudo -u www-data php /var/www/moodle/admin/tool/replace/cli/replace.php \
--search=https://moodle-a.lab.com \
--replace=https://moodle-b.lab.com \
--shortenurls=0
```

### ✅ 請務必先備份資料庫：
```bash
mysqldump -u moodlelabuser -p moodle > ~/moodle_backup_before_url_replace.sql
```
