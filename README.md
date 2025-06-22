# **Pterodactyl Migration - Manual**

✅ **SYARAT VPS BARU**
> Ubuntu 20.04 / 22.04

> Node.js (sama versi)

> PHP 8.1

> MySQL/MariaDB

> Wings pakai Docker

> RAM minimal 4GB

> Domain diarahkan ke IP baru setelah semua siap

---
🔁 **LANGKAH DETAIL MIGRASI PTERODACTYL + WINGS**
---
🧩 1. VPS LAMA — BACKUP DATA
---

✅ Backup database:
```bash
mysqldump -u root -p pterodactyl > /root/pterodactyl.sql
```

✅ Backup file panel:
```bash
tar -czvf /root/panel.tar.gz /var/www/pterodactyl
```

✅ Backup eggs:
```bash
tar -czvf /root/eggs.tar.gz /var/lib/pterodactyl/volumes
```

✅ Backup konfigurasi wings:
```bash
cp /etc/pterodactyl/config.yml /root/wings-config.yml
```

✅ Backup file daemon (optional):
```bash
tar -czvf /root/wings.tar.gz /etc/pterodactyl
```
---
📤 2. TRANSFER KE VPS BARU
---

✅ Command di VPS baru:
```bash
scp root@IP_VPS_LAMA:/root/pterodactyl.sql /root/
scp root@IP_VPS_LAMA:/root/panel.tar.gz /root/
scp root@IP_VPS_LAMA:/root/eggs.tar.gz /root/
scp root@IP_VPS_LAMA:/root/wings-config.yml /root/
```
---
⚙️ 3. VPS BARU — INSTALL ULANG DEPENDENSI
---

✅ Install Pterodactyl Panel:

Ikuti panduan resmi Pterodactyl panel tanpa setup database baru.

1. Setup VPS
```bash
sudo apt update && sudo apt upgrade -y

# Install Redis
sudo apt install redis-server -y

# Enable Redis
sudo systemctl enable redis
sudo systemctl start redis

# Cek Status Redis (optional)
sudo systemctl status redis
```

2. Install Dependensi:
```bash
# Add "add-apt-repository" command
apt -y install software-properties-common curl apt-transport-https ca-certificates gnupg

# Add additional repositories for PHP (Ubuntu 22.04)
LC_ALL=C.UTF-8 add-apt-repository -y ppa:ondrej/php

# Add Redis official APT repository
curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list

# Update repositories list
apt update

# Install Dependencies
apt -y install php8.3 php8.3-{common,cli,gd,mysql,mbstring,bcmath,xml,fpm,curl,zip} mariadb-server nginx tar unzip git redis-server
```

3. Install Composer
```bash
curl -sS https://getcomposer.org/installer | sudo php -- --install-dir=/usr/local/bin --filename=composer
```

4. Buat file Pterodactyl
```bash
mkdir -p /var/www/pterodactyl
tar -xzvf /root/panel.tar.gz -C /var/www/pterodactyl --strip-components=1
chmod -R 755 storage/* bootstrap/cache/
chown -R www-data:www-data /var/www/pterodactyl
```
---
🧮 4. RESTORE DATABASE
---

```bash
# Buat database
mysql -u root -p
CREATE DATABASE pterodactyl;
exit;

# Restore database
mysql -u root -p pterodactyl < /root/pterodactyl.sql
chown -R www-data:www-data /var/www/pterodactyl/*
```
---
📃 5. Crontab Configuration
---

```* * * * * php /var/www/pterodactyl/artisan schedule:run >> /dev/null 2>&1``` 
```bash
sudo crontab -e

# setelah masuk tempel kodenya lalu tekan "shift + ;" ketik wq dan enter
```

```
# Pterodactyl Queue Worker File
# ----------------------------------

[Unit]
Description=Pterodactyl Queue Worker
After=redis-server.service

[Service]
# On some systems the user and group might be different.
# Some systems use `apache` or `nginx` as the user and group.
User=www-data
Group=www-data
Restart=always
ExecStart=/usr/bin/php /var/www/pterodactyl/artisan queue:work --queue=high,standard,low --sleep=3 --tries=3
StartLimitInterval=180
StartLimitBurst=30
RestartSec=5s

[Install]
WantedBy=multi-user.target
```
```bash
# copy dan paste di sini
nano /etc/systemd/system/pteroq.service
```

```bash
sudo systemctl enable --now redis-server
sudo systemctl enable --now pteroq.service
```
---
🔒 6. Creating SSL Certificates
---

```bash
sudo apt update
sudo apt install -y certbot
# Run this if you use Nginx
sudo apt install -y python3-certbot-nginx
```

```bash
# Nginx
certbot certonly --nginx -d <domain>
```

```bash
curl https://get.acme.sh | sh
```
Memperoleh Kunci API CloudFlare

Setelah menginstal acme.sh, kita perlu mengambil kunci API CloudFlare. Di situs web Cloudfare, pilih domain Anda, lalu di sisi kanan, salin "ID Zona" dan "ID Akun" Anda, lalu klik "Dapatkan token API Anda", klik "Buat Token" > pilih templat "Edit DNS zona" > pilih cakupan "Sumber Daya Zona" lalu klik "Lanjutkan ke ringkasan", salin token Anda.

```bash
sudo mkdir -p /etc/letsencrypt/live/<domain>
```

```bash
export CF_Token="Your_CloudFlare_API_Key"
export CF_Account_ID="Your_CloudFlare_Account_ID"
export CF_Zone_ID="Your_CloudFlare_Zone_ID"
```

```bash
# Ganti <domain> menjadi domain anda

./.acme.sh/acme.sh --issue --dns dns_cf -d "<domain>" --server letsencrypt \
--key-file /etc/letsencrypt/live/<comain>/privkey.pem \
--fullchain-file /etc/letsencrypt/live/<domain>/fullchain.pem
```

```bash
rm /etc/nginx/sites-enabled/default
```

```
server {
    # Replace the example <domain> with your domain name or IP address
    listen 80;
    server_name <domain>;
    return 301 https://$server_name$request_uri;
}

server {
    # Replace the example <domain> with your domain name or IP address
    listen 443 ssl http2;
    server_name <domain>;

    root /var/www/pterodactyl/public;
    index index.php;

    access_log /var/log/nginx/pterodactyl.app-access.log;
    error_log  /var/log/nginx/pterodactyl.app-error.log error;

    # allow larger file uploads and longer script runtimes
    client_max_body_size 100m;
    client_body_timeout 120s;

    sendfile off;

    # SSL Configuration - Replace the example <domain> with your domain
    ssl_certificate /etc/letsencrypt/live/<domain>/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/<domain>/privkey.pem;
    ssl_session_cache shared:SSL:10m;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384";
    ssl_prefer_server_ciphers on;

    # See https://hstspreload.org/ before uncommenting the line below.
    # add_header Strict-Transport-Security "max-age=15768000; preload;";
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Robots-Tag none;
    add_header Content-Security-Policy "frame-ancestors 'self'";
    add_header X-Frame-Options DENY;
    add_header Referrer-Policy same-origin;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param PHP_VALUE "upload_max_filesize = 100M \n post_max_size=100M";
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param HTTP_PROXY "";
        fastcgi_intercept_errors off;
        fastcgi_buffer_size 16k;
        fastcgi_buffers 4 16k;
        fastcgi_connect_timeout 300;
        fastcgi_send_timeout 300;
        fastcgi_read_timeout 300;
        include /etc/nginx/fastcgi_params;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

```bash
# ubah <domain> menjadi domain panelmu lalu tempel disini
nano /etc/nginx/sites-available/pterodactyl.conf
```

```bash
sudo ln -s /etc/nginx/sites-available/pterodactyl.conf /etc/nginx/sites-enabled/pterodactyl.conf
sudo systemctl restart nginx
```
---
🔧 7. SESUAIKAN .env PANEL
---

```bash
# Edit file .env 
nano /var/www/pterodactyl/.env

# Ubah APP_URL=<domain>

# Ubah koneksi DB kalau ada perubahan

# Cek APP_KEY masih sama
```
---
🛠️ 8. BUILD UI PANEL
---

```bash
cd /var/www/pterodactyl
yarn install
yarn build:production
php artisan view:clear
php artisan config:clear
php artisan queue:restart
```
---
🐳 9. INSTALL WINGS DI VPS BARU
---

1. Installing Docker
```bash
cd
curl -sSL https://get.docker.com/ | CHANNEL=stable bash
```

2. Start Docker on Boot
```bash
sudo systemctl enable --now docker
```

3. Enabling Swap
```bash
nano /etc/default/grub
```
Pada Baris ```GRUB_CMDLINE_LINUX_DEFAULT=""``` tambahkan ```swapaccount=1```
Example:
```GRUB_CMDLINE_LINUX_DEFAULT="swapaccount=1"```

```bash
# Lakukan update dan reboot
sudo update-grub
sudo reboot #Optional
```

4. Install Wings
```bash
sudo mkdir -p /etc/pterodactyl
curl -L -o /usr/local/bin/wings "https://github.com/pterodactyl/wings/releases/latest/download/wings_linux_$([[ "$(uname -m)" == "x86_64" ]] && echo "amd64" || echo "arm64")"
sudo chmod u+x /usr/local/bin/wings
```

5. Pindahkan config.yml
```bash
mv /root/wings-config.yml /etc/pterodactyl/config.yml
```

6. Daemonizing (using systemd)

```
[Unit]
Description=Pterodactyl Wings Daemon
After=docker.service
Requires=docker.service
PartOf=docker.service

[Service]
User=root
WorkingDirectory=/etc/pterodactyl
LimitNOFILE=4096
PIDFile=/var/run/wings/daemon.pid
ExecStart=/usr/local/bin/wings
Restart=on-failure
StartLimitInterval=180
StartLimitBurst=30
RestartSec=5s

[Install]
WantedBy=multi-user.target
```
```bash
# Copy dan paste ke wings.service
nano /etc/systemd/system/wings.service
```

```bash
sudo systemctl enable --now wings
```
---
📦 10. RESTORE EGG VOLUMES (DATA SERVER)
---

```bash
tar -xzvf /root/eggs.tar.gz -C /
```
