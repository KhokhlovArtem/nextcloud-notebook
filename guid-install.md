# Установка Nextcloud на AlmaLinux и Ubuntu

## AlmaLinux

### 1. После установки системы
Устанавливаем расширенный репозиторий:

```bash
sudo dnf install epel-release -y
sudo dnf config-manager --set-enable crb --enable -y
sudo dnf module enable mariadb:10.11
```

Обновляем пакеты и систему:

```bash
sudo dnf update -y
```

### 2. Установка репозитория для PHP

```bash
sudo dnf -y install http://rpms.remirepo.net/enterprise/remi-release-9.rpm
sudo dnf module reset php -y
sudo dnf module enable php:rmi-8.3
```

### 3. Отключаем SELinux

```bash
grubby --update-kernel ALL --args selinux=0
```

### 4. Настройка часового пояса

```bash
sudo timedatectl set-timezone Europe/Moscow
```

### 5. Установка и настройка Nginx

```bash
sudo dnf install nginx-all-modules
sudo systemctl enable nginx.service
sudo firewall-cmd --add-port=81/tcp --permanent
```

### 6. Установка PHP и дополнительных модулей

```bash
sudo dnf install php php-fpm php-mysqlnd php-gd php-intl php-gmp php-smbclient php-bcmath php-process php-apcu php-zip php-imagick php-redis exif wget mariadb-server ffmpeg-free redis
```

### 7. Настройка MariaDB

```bash
sudo mariadb -u root -p
```

```sql
CREATE DATABASE nxcloud;
CREATE USER nxuser@localhost IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON nxcloud.* TO nxuser@localhost;
FLUSH PRIVILEGES;
SHOW GRANTS FOR nxuser@localhost;
```

### 8. Настройка PHP

Файл `/etc/php.d/10-opcache.ini`:
```ini
opcache.enable = 1
opcache.interned_strings_buffer = 32
opcache.max_accelerated_files = 10000
opcache.memory_consumption = 128
opcache.save_comments = 1
opcache.revalidate_freq = 1
```

Файл `/etc/php.ini`:
```ini
file_uploads = On
allow_url_fopen = On
memory_limit = 512M
upload_max_filesize = 500M
post_max_size = 600M 
max_execution_time = 300
display_errors = Off
date.timezone = Europe/Moscow
```

### 9. Установка Nextcloud

```bash
wget https://download.nextcloud.com/server/releases/latest.zip
unzip latest.zip -d /var/www/
chown -R nginx:nginx /var/www/nextcloud
sudo -u nginx crontab -e
```

Добавить в cron:
```cron
*/5  *  *  *  * /usr/bin/php -f /var/www/nextcloud/cron.php
```

```bash
occ config:system:set maintenance_window_start --type=integer --value=1
```

### 10. Установка OnlyOffice (Docker)

```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install docker-ce docker-ce-cli containerd.io
systemctl enable docker
systemctl start docker
sudo docker run -i -t -d -p 8080:80 --restart=always \
    -v /srv/onoffice/logs:/var/log/onlyoffice  \
    -v /srv/onoffice/data:/var/www/onlyoffice/Data  \
    -v /srv/onoffice/lib:/var/lib/onlyoffice \
    -v /srv/onoffice/db:/var/lib/postgresql -e JWT_SECRET=mpNMjN1sSKM8DQ49 onlyoffice/documentserver
```

## Ubuntu

### 1. Установка пакетов

```bash
sudo apt install php php-fpm php-mysqlnd php-gd php-intl php-gmp php-smbclient php-bcmath php-apcu php-zip php-imagick php-redis php-dom php-mbstring php-curl php-xml-svg php-xml imagemagick-6.q16 exif ffmpeg wget mariadb-server redis nginx-full -y
```

### 2. Настройка PHP

Файл `/etc/php/8.3/fpm/conf.d/10-opcache.ini`:
```ini
opcache.enable = 1
opcache.interned_strings_buffer = 32
opcache.max_accelerated_files = 10000
opcache.memory_consumption = 128
opcache.save_comments = 1
opcache.revalidate_freq = 1
```

Файл `/etc/php/8.3/fpm/php.ini`:
```ini
file_uploads = On
allow_url_fopen = On
memory_limit = 512M
upload_max_filesize = 500M
post_max_size = 600M 
max_execution_time = 300
display_errors = Off
date.timezone = Europe/Moscow
```

### 3. Установка Nextcloud

```bash
wget https://download.nextcloud.com/server/releases/latest.zip
unzip latest.zip -d /srv/nextcloud
```

### 4. Настройка MariaDB

```sql
CREATE DATABASE nxcloud;
CREATE USER nxuser@localhost IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON nxcloud.* TO nxuser@localhost;
FLUSH PRIVILEGES;
SHOW GRANTS FOR nxuser@localhost;
```

### 5. Настройка планировщика

```bash
sudo -u www-data crontab -e
```

Добавить:
```cron
*/5  *  *  *  * /usr/bin/php -f /srv/nxcloud/cron.php
```

### 6. Установка Docker

```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo docker run -d -p 127.0.0.1:8080:80 --restart=always --name documentserver \
    -v /srv/onoffice/logs:/var/log/onlyoffice  \
    -v /srv/onoffice/data:/var/www/onlyoffice/Data  \
    -v /srv/onoffice/lib:/var/lib/onlyoffice \
    -v /srv/onoffice/db:/var/lib/postgresql -e JWT_SECRET=password onlyoffice/documentserver
```

## Конфигурация Nginx

```nginx
upstream docservice {
  server 127.0.0.1:8080;
}

map $http_x_forwarded_proto $the_scheme {
     default $http_x_forwarded_proto;
     "" $scheme;
}

map $http_x_forwarded_host $the_host {
    default $http_x_forwarded_host;
    "" $host;
}

map $http_upgrade $proxy_connection {
  default upgrade;
  "" close;
}

proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection $proxy_connection;
proxy_set_header X-Forwarded-Host $the_host/editors;
proxy_set_header X-Forwarded-Proto $the_scheme;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

upstream php-handler {
    server unix:/run/php/ncloud.sock;
}

[... остальная часть конфигурации ...]
```

## Бэкап и восстановление Nextcloud

Создание бэкапа базы данных:
```bash
mysqldump --single-transaction --default-character-set=utf8mb4 nxcloud > nextcloud-sqlbkp.bak
```

Восстановление базы данных:
```bash
mysql -e "DROP DATABASE nxcloud"
mysql -e "CREATE DATABASE nxcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci"
mysql nxcloud < nextcloud-sqlbkp.bak
```
