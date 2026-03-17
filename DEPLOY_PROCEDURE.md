# DEPLOY / RESTORE PROCEDURE — Dan Pedia

> Panduan lengkap restore backup atau deploy ulang ke VPS baru  
> Platform: **dan-pedia.com** | Backend: Laravel 12 | Frontend: Next.js 14

---

## ⚠️ PENTING — Keamanan File Backup

File backup zip berisi:
- File `.env` Laravel dan Next.js → mengandung password DB, API key, JWT secret
- Database dump → mengandung data user dan transaksi

**Jangan bagikan file ini secara publik. Simpan di tempat aman.**

---

## Isi File Backup (`danpedia_backup_YYYYMMDD.zip`)

```
danpedia_backup_YYYYMMDD.zip
├── Laravel_Backend/                   ← Source code Laravel (tanpa vendor/)
│   ├── app/
│   ├── config/
│   ├── database/
│   ├── routes/
│   ├── .env                           ← TIDAK disertakan (lihat .env_laravel_backup)
│   └── ...
├── NextJS_FrontEnd/                   ← Source code Next.js (tanpa node_modules/ & .next/)
│   ├── src/
│   ├── public/
│   ├── .env.local                     ← TIDAK disertakan (lihat .env_nextjs_backup)
│   └── ...
├── danpedia_backup_YYYYMMDD/
│   ├── .env_laravel_backup            ← Isi lengkap .env Laravel
│   ├── .env_nextjs_backup             ← Isi lengkap .env Next.js
│   └── database_danpedia_YYYYMMDD.sql.gz ← Dump database MySQL (compressed)
```

---

## Spesifikasi Server

| Item | Detail |
|------|--------|
| OS | Ubuntu 24.04 LTS |
| Minimum RAM | 2 GB |
| Minimum Storage | 20 GB |
| PHP | 8.3 |
| Node.js | v20 LTS |
| MySQL | 8.0 |
| Nginx | 1.24+ |
| PM2 | v6+ |
| Supervisor | 4.2+ |

---

## BAGIAN 1 — Setup VPS Baru (Fresh Install)

### 1.1 — Update & Install Dependencies

```bash
apt update && apt upgrade -y

# Install PHP 8.3
apt install -y software-properties-common
add-apt-repository ppa:ondrej/php -y
apt update
apt install -y php8.3 php8.3-fpm php8.3-cli php8.3-mysql php8.3-mbstring \
  php8.3-xml php8.3-curl php8.3-zip php8.3-bcmath php8.3-tokenizer \
  php8.3-gd php8.3-intl php8.3-redis

# Install MySQL 8.0
apt install -y mysql-server

# Install Node.js v20
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt install -y nodejs

# Install Composer
curl -sS https://getcomposer.org/installer | php
mv composer.phar /usr/local/bin/composer

# Install Nginx
apt install -y nginx

# Install PM2
npm install -g pm2

# Install Supervisor
apt install -y supervisor

# Install zip/unzip
apt install -y zip unzip git
```

---

### 1.2 — Setup Database MySQL

```bash
mysql -u root -p

# Di dalam MySQL:
CREATE DATABASE danpedia CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'danpedia'@'localhost' IDENTIFIED BY 'PASSWORD_BARU';
GRANT ALL PRIVILEGES ON danpedia.* TO 'danpedia'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

---

### 1.3 — Ekstrak File Backup

```bash
# Upload zip ke VPS
# Dari Windows lokal:
# pscp -pw "VPS_PASSWORD" danpedia_backup_YYYYMMDD.zip root@VPS_IP:/var/backup/

mkdir -p /var/www
cd /var/backup
unzip danpedia_backup_YYYYMMDD.zip -d /tmp/danpedia_restore

# Copy Laravel Backend
cp -r /tmp/danpedia_restore/Laravel_Backend /var/www/Laravel_Backend

# Copy NextJS Frontend
cp -r /tmp/danpedia_restore/NextJS_FrontEnd /var/www/NextJS_FrontEnd
```

---

### 1.4 — Restore Database

```bash
# Uncompress dan restore database dump
gunzip -c /var/backup/danpedia_backup_YYYYMMDD/database_danpedia_YYYYMMDD.sql.gz | \
  mysql -u danpedia -p danpedia
```

---

### 1.5 — Setup Laravel Backend

```bash
cd /var/www/Laravel_Backend

# Copy file .env dari backup (lalu sesuaikan jika IP/password berubah)
cp /var/backup/danpedia_backup_YYYYMMDD/danpedia_backup_YYYYMMDD/.env_laravel_backup .env

# Edit .env jika ada perubahan server
nano .env
# Ubah: DB_HOST, DB_PASSWORD, APP_URL, dll sesuai server baru

# Install dependencies PHP
composer install --no-dev --optimize-autoloader

# Set permissions
chown -R www-data:www-data /var/www/Laravel_Backend
chmod -R 755 /var/www/Laravel_Backend/storage
chmod -R 755 /var/www/Laravel_Backend/bootstrap/cache

# Generate key (HANYA jika fresh install, bukan restore)
# php artisan key:generate

# Clear & cache config
php artisan config:clear
php artisan cache:clear
php artisan route:clear
php artisan view:clear
php artisan config:cache
php artisan route:cache

# Run migration (HANYA jika database kosong/fresh)
# php artisan migrate --force

# Buat symlink storage
php artisan storage:link
```

---

### 1.6 — Setup NextJS Frontend

```bash
cd /var/www/NextJS_FrontEnd

# Copy file .env dari backup (lalu sesuaikan jika domain berubah)
cp /var/backup/danpedia_backup_YYYYMMDD/danpedia_backup_YYYYMMDD/.env_nextjs_backup .env.local

# Edit .env.local jika ada perubahan domain/API URL
nano .env.local
# Ubah: NEXT_PUBLIC_API_URL, NEXT_PUBLIC_SITE_URL, dll

# Install dependencies Node.js
npm install

# Build production
NODE_OPTIONS=--max_old_space_size=1024 npm run build

# Set permissions
chown -R www-data:www-data /var/www/NextJS_FrontEnd
```

---

### 1.7 — Setup PM2 (Next.js Process Manager)

```bash
cd /var/www/NextJS_FrontEnd

# Start dengan PM2
pm2 start npm --name "danpedia" -- start

# Simpan konfigurasi PM2
pm2 save

# Auto-start PM2 saat server reboot
pm2 startup
# Jalankan perintah yang ditampilkan PM2, contoh:
# systemctl enable pm2-root
```

---

### 1.8 — Setup Supervisor (Laravel Queue Worker)

```bash
# Buat config Supervisor
cat > /etc/supervisor/conf.d/laravel-worker.conf << 'EOF'
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/Laravel_Backend/artisan queue:work database --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=www-data
numprocs=2
redirect_stderr=true
stdout_logfile=/var/www/Laravel_Backend/storage/logs/worker.log
stopwaitsecs=3600
EOF

# Aktivasi
systemctl enable supervisor
systemctl start supervisor
supervisorctl reread
supervisorctl update
supervisorctl start laravel-worker:*

# Cek status
supervisorctl status
```

---

### 1.9 — Setup Nginx

```bash
# Buat config Nginx
cat > /etc/nginx/sites-available/danpedia << 'EOF'
# Backend API — api.dan-pedia.com
server {
    listen 80;
    server_name api.dan-pedia.com;
    root /var/www/Laravel_Backend/public;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.ht {
        deny all;
    }

    client_max_body_size 10M;
}

# Frontend — www.dan-pedia.com & dan-pedia.com
server {
    listen 80;
    server_name www.dan-pedia.com dan-pedia.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
EOF

# Aktifkan site
ln -sf /etc/nginx/sites-available/danpedia /etc/nginx/sites-enabled/danpedia
rm -f /etc/nginx/sites-enabled/default

# Test dan reload Nginx
nginx -t
systemctl reload nginx
```

---

### 1.10 — Setup SSL (HTTPS) dengan Certbot

```bash
apt install -y certbot python3-certbot-nginx

# Generate SSL untuk semua domain
certbot --nginx -d www.dan-pedia.com -d dan-pedia.com -d api.dan-pedia.com

# Auto-renewal
systemctl enable certbot.timer
```

---

### 1.11 — Setup Laravel Scheduler (Cron)

```bash
# Tambah cron job untuk Laravel scheduler
crontab -e

# Tambahkan baris berikut:
* * * * * cd /var/www/Laravel_Backend && php artisan schedule:run >> /dev/null 2>&1
```

---

## BAGIAN 2 — Re-Deploy ke VPS yang Sudah Ada

Gunakan jika server masih jalan dan ingin update kode saja (tanpa restore DB).

### 2.1 — Upload & Replace File

```bash
# Upload file baru ke VPS (dari Windows):
pscp -pw "VPS_PASSWORD" -r Laravel_Backend/ root@VPS_IP:/var/www/Laravel_Backend/
pscp -pw "VPS_PASSWORD" -r NextJS_FrontEnd/ root@VPS_IP:/var/www/NextJS_FrontEnd/
```

### 2.2 — Deploy Backend (Laravel)

```bash
cd /var/www/Laravel_Backend
composer install --no-dev --optimize-autoloader
php artisan config:clear
php artisan cache:clear
php artisan route:clear
php artisan view:clear
php artisan config:cache
php artisan route:cache
php artisan migrate --force   # hanya jika ada migration baru
chown -R www-data:www-data storage bootstrap/cache
supervisorctl restart laravel-worker:*
```

### 2.3 — Deploy Frontend (Next.js)

```bash
cd /var/www/NextJS_FrontEnd
npm install
NODE_OPTIONS=--max_old_space_size=1024 npm run build
pm2 restart danpedia
```

---

## BAGIAN 3 — Verifikasi Setelah Deploy

```bash
# Cek status PM2
pm2 status

# Cek status Supervisor
supervisorctl status

# Cek status Nginx
systemctl status nginx

# Cek log error Laravel
tail -50 /var/www/Laravel_Backend/storage/logs/laravel.log

# Cek log PM2
pm2 logs danpedia --lines 30

# Test API
curl -s https://api.dan-pedia.com/api/settings | head -c 200

# Test Frontend
curl -s -o /dev/null -w "%{http_code}" https://www.dan-pedia.com

# Cek jadwal scheduler Laravel
cd /var/www/Laravel_Backend && php artisan schedule:list
```

---

## BAGIAN 4 — Environment Variables Penting

### Laravel `.env` — Variabel yang Perlu Disesuaikan

```dotenv
APP_URL=https://api.dan-pedia.com
FRONTEND_URL=https://www.dan-pedia.com

DB_HOST=127.0.0.1
DB_DATABASE=danpedia
DB_USERNAME=danpedia
DB_PASSWORD=SESUAIKAN

QUEUE_CONNECTION=database

# Payment Gateways
DUITKU_MERCHANT_CODE=...
DUITKU_API_KEY=...

# Digiflazz
DIGIFLAZZ_USERNAME=...
DIGIFLAZZ_API_KEY_PRODUCTION=...

# Fonnte/MPWA (WhatsApp)
FONNTE_TOKEN=...

# Koalastore
KOALASTORE_API_KEY=...
```

### NextJS `.env.local` — Variabel yang Perlu Disesuaikan

```dotenv
NEXT_PUBLIC_API_URL=https://api.dan-pedia.com
NEXT_PUBLIC_SITE_URL=https://www.dan-pedia.com
API_ACCESS_KEY=SESUAIKAN
```

---

## BAGIAN 5 — Jadwal Otomatis (Scheduler)

| Perintah | Jadwal | Fungsi |
|----------|--------|--------|
| `product:fetch-product` | 00:00 & 06:00 WIB | Sync produk Digiflazz |
| `koalastore:sync-products --force-update` | Setiap 30 menit | Sync produk digital Koalastore |
| `queue:work` (Supervisor) | Terus-menerus | Proses background job |

---

## BAGIAN 6 — Troubleshooting Umum

### Next.js tidak bisa build (out of memory)
```bash
NODE_OPTIONS=--max_old_space_size=1024 npm run build
```

### Laravel 500 error setelah deploy
```bash
cd /var/www/Laravel_Backend
php artisan config:clear && php artisan cache:clear
chown -R www-data:www-data storage bootstrap/cache
chmod -R 775 storage bootstrap/cache
```

### Queue worker tidak jalan
```bash
supervisorctl status
supervisorctl restart laravel-worker:*
# Jika masih error:
supervisorctl reread && supervisorctl update
```

### PM2 tidak otomatis start setelah reboot
```bash
pm2 startup
# Jalankan perintah yang ditampilkan
pm2 save
```

### Database connection error
```bash
# Test koneksi dari Laravel
cd /var/www/Laravel_Backend && php artisan db:show
```

---

*Terakhir diupdate: 2026-03-17*
