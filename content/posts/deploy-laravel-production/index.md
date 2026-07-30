+++
date = '2026-07-29T07:00:00+07:00'
draft = false
title = 'Deployment Laravel Production dengan Docker, PHP 8.3, MySQL, dan Nginx'
categories = ['Web Server']
tags = [
    'Laravel',
    'Laravel 12',
    'Docker',
    'PHP',
    'PHP 8.3',
    'PHP-FPM',
    'MySQL',
    'Nginx',
    'Ubuntu Server',
    'Web Server',
    'Server Hardening',
    'DevOps',
    'Git',
    'GitHub',
    'Deployment'
]
+++

## 1. Pendahuluan

Panduan ini menjelaskan langkah demi langkah instalasi dan deployment aplikasi Laravel pada server production menggunakan:

- Ubuntu Server 24.04 LTS
- Laravel 12
- Docker
- PHP-FPM 8.3
- MySQL
- Nginx
- Composer
- Git/GitHub
- Persistent Storage
- Unix Socket PHP-FPM
- Unix Socket MySQL
- Hardening direktori upload

Sebagai contoh, aplikasi yang digunakan dalam panduan ini adalah:

```text
Aplikasi Perpustakaan
```

Domain contoh:

```text
https://perpustakaan.example.go.id
```

Nama aplikasi:

```text
perpustakaan
```

Source code aplikasi:

```text
/var/apps/perpustakaan/app
```

Persistent data:

```text
/var/apps/perpustakaan/data
```

Konfigurasi Docker:

```text
/opt/docker/apps/perpustakaan
```

Arsitektur deployment:

```text
Internet
   |
   | HTTPS
   v
Reverse Proxy Nginx
   |
   | HTTP :8080
   v
Nginx Application Server
   |
   | Unix Socket
   | /run/php/perpustakaan.sock
   v
Docker PHP-FPM 8.3
   |
   +---- Laravel Source (Read Only)
   |
   +---- Persistent Storage (Read Write)
   |
   +---- MySQL Unix Socket
              |
              v
           MySQL Server
```

Pada arsitektur ini, source code Laravel dipisahkan dari data runtime.

Source code dikelola menggunakan Git, sedangkan file upload, cache, session, log, dan data runtime Laravel disimpan pada persistent storage.

Source Laravel dipasang **read-only pada container PHP-FPM runtime**.

Pada proses deployment tertentu, misalnya ketika menjalankan Composer, temporary container dapat diberikan akses write ke source agar dependency pada direktori `vendor/` dapat diperbarui.

Tujuannya adalah agar update aplikasi tidak menghapus data yang dihasilkan selama aplikasi berjalan.


## 2. Struktur Direktori

Source code aplikasi dan data runtime dipisahkan.

Struktur yang digunakan:

```text
/var/apps/perpustakaan/
├── app/                         # Repository Git Laravel
└── data/                        # Persistent runtime data
    ├── storage-app/
    ├── storage-framework/
    ├── storage-logs/
    └── bootstrap-cache/
```

Konfigurasi Docker:

```text
/opt/docker/apps/perpustakaan/
├── docker-compose.yml
└── zz-custom.conf
```

Pemisahan ini penting karena source code dapat diperbarui menggunakan Git tanpa mengganggu file upload pengguna dan data runtime Laravel.


## 3. Clone Repository Laravel

Buat direktori aplikasi:

```bash
mkdir -p /var/apps/perpustakaan
```

Masuk:

```bash
cd /var/apps/perpustakaan
```

Clone repository:

```bash
git clone https://github.com/USERNAME/aplikasi-perpustakaan.git app
```

Masuk ke repository:

```bash
cd /var/apps/perpustakaan/app
```

Periksa repository:

```bash
git status
git branch --show-current
git remote -v
```

Pastikan branch production sesuai, misalnya:

```text
main
```


## 4. Membuat Persistent Storage

Buat direktori runtime:

```bash
mkdir -p /var/apps/perpustakaan/data/storage-app
mkdir -p /var/apps/perpustakaan/data/storage-framework
mkdir -p /var/apps/perpustakaan/data/storage-logs
mkdir -p /var/apps/perpustakaan/data/bootstrap-cache
```

Buat struktur framework Laravel:

```bash
mkdir -p /var/apps/perpustakaan/data/storage-framework/cache
mkdir -p /var/apps/perpustakaan/data/storage-framework/sessions
mkdir -p /var/apps/perpustakaan/data/storage-framework/views
```

Buat public storage:

```bash
mkdir -p /var/apps/perpustakaan/data/storage-app/public
```

Atur ownership:

```bash
chown -R www-data:www-data /var/apps/perpustakaan/data
```

Atur permission:

```bash
chmod -R u=rwX,g=rwX,o= /var/apps/perpustakaan/data
```

Dengan konfigurasi tersebut, hanya owner dan group yang memiliki akses terhadap persistent storage.


## 5. Konfigurasi Docker PHP 8.3

Buat direktori:

```bash
mkdir -p /opt/docker/apps/perpustakaan
```

Buat file:

```text
/opt/docker/apps/perpustakaan/docker-compose.yml
```

Isi:

```yaml
services:

  app:
    container_name: perpustakaan-php
    image: local/php:8.3

    read_only: true
    restart: unless-stopped
    init: true
    stop_grace_period: 30s

    security_opt:
      - no-new-privileges:true

    volumes:

      # PHP-FPM Unix Socket
      - /run/php:/run/php

      # MySQL Unix Socket
      - /run/mysqld:/run/mysqld:ro

      # PHP configuration
      - /opt/docker/images/php8.3/php.ini:/usr/local/etc/php/conf.d/99-custom.ini:ro

      # PHP-FPM configuration
      - /opt/docker/apps/perpustakaan/zz-custom.conf:/usr/local/etc/php-fpm.d/zz-custom.conf:ro

      # Laravel source - READ ONLY pada runtime
      - /var/apps/perpustakaan/app:/var/www/html:ro

      # Laravel writable runtime
      - /var/apps/perpustakaan/data/storage-app:/var/www/html/storage/app:rw
      - /var/apps/perpustakaan/data/storage-framework:/var/www/html/storage/framework:rw
      - /var/apps/perpustakaan/data/storage-logs:/var/www/html/storage/logs:rw
      - /var/apps/perpustakaan/data/bootstrap-cache:/var/www/html/bootstrap/cache:rw

    tmpfs:
      - /tmp

    healthcheck:
      test: ["CMD-SHELL", "pidof php-fpm || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s

    ulimits:
      nofile:
        soft: 4096
        hard: 8192

    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "5"
```

> **Catatan:** `local/php:8.3` merupakan image PHP 8.3 yang telah disiapkan sebelumnya dan harus memiliki PHP-FPM, Composer, serta ekstensi PHP yang dibutuhkan aplikasi Laravel.


## 6. Konfigurasi PHP-FPM

Buat:

```text
/opt/docker/apps/perpustakaan/zz-custom.conf
```

Isi:

```ini
[www]

listen = /run/php/perpustakaan.sock

listen.owner = www-data
listen.group = www-data
listen.mode = 0660

pm = dynamic
pm.max_children = 30
pm.start_servers = 4
pm.min_spare_servers = 2
pm.max_spare_servers = 8
pm.max_requests = 500

clear_env = yes

catch_workers_output = yes
decorate_workers_output = no
```

PHP-FPM akan membuat Unix Socket:

```text
/run/php/perpustakaan.sock
```

Socket tersebut digunakan Nginx untuk meneruskan request PHP ke PHP-FPM di dalam container.


## 7. Menjalankan Container

Masuk:

```bash
cd /opt/docker/apps/perpustakaan
```

Jalankan:

```bash
docker compose up -d
```

Periksa:

```bash
docker ps -a --filter name=perpustakaan-php
```

Target:

```text
Up ... (healthy)
```

Periksa PHP-FPM Unix Socket:

```bash
ls -lah /run/php/perpustakaan.sock
```

Periksa detail:

```bash
stat /run/php/perpustakaan.sock
```


## 8. Memeriksa MySQL Unix Socket

Karena MySQL berjalan pada host dan aplikasi akan mengaksesnya menggunakan Unix Socket, periksa:

```bash
ls -lah /run/mysqld/mysqld.sock
```

Pastikan socket tersedia.

Periksa dari dalam container:

```bash
docker exec perpustakaan-php \
  ls -lah /run/mysqld/mysqld.sock
```

Jika socket terlihat dari dalam container, bind mount:

```yaml
- /run/mysqld:/run/mysqld:ro
```

telah bekerja.


## 9. Memeriksa Docker Mount

Jalankan:

```bash
docker inspect perpustakaan-php --format \
'{{range .Mounts}}{{println .Source "->" .Destination .Mode}}{{end}}'
```

Pastikan terdapat mapping:

```text
/var/apps/perpustakaan/app
->
/var/www/html ro
```

Persistent storage:

```text
/var/apps/perpustakaan/data/storage-app
->
/var/www/html/storage/app rw
```

Framework:

```text
/var/apps/perpustakaan/data/storage-framework
->
/var/www/html/storage/framework rw
```

Log:

```text
/var/apps/perpustakaan/data/storage-logs
->
/var/www/html/storage/logs rw
```

Bootstrap cache:

```text
/var/apps/perpustakaan/data/bootstrap-cache
->
/var/www/html/bootstrap/cache rw
```

Dengan demikian:

```text
Laravel Source  = READ ONLY pada runtime
Laravel Runtime = READ WRITE
```


## 10. Install Dependency Composer

Source Laravel pada container PHP-FPM bersifat read-only.

Karena Composer perlu membuat atau memperbarui:

```text
vendor/
```

gunakan temporary container dengan source mount yang writable:

```bash
docker run --rm \
  -v /var/apps/perpustakaan/app:/var/www/html \
  -w /var/www/html \
  local/php:8.3 \
  composer install \
    --no-dev \
    --optimize-autoloader \
    --no-interaction
```

Periksa:

```bash
ls -lah /var/apps/perpustakaan/app/vendor
```

Direktori:

```text
vendor/
```

harus tersedia.

Temporary container Composer hanya digunakan pada proses deployment.

Container PHP-FPM utama tetap menggunakan:

```text
/var/www/html:ro
```


## 11. Membuat File `.env`

Masuk:

```bash
cd /var/apps/perpustakaan/app
```

Copy:

```bash
cp .env.example .env
```

Edit:

```bash
nano .env
```

> **Peringatan Keamanan**
>
> Jangan pernah memasukkan file `.env` production ke repository Git atau mempublikasikan credential asli dalam dokumentasi.
>
> File `.env` dapat berisi `APP_KEY`, password database, API key, token, dan credential layanan lainnya.

Contoh:

```env
APP_NAME="Aplikasi Perpustakaan"
APP_ENV=production
APP_KEY=
APP_DEBUG=false
APP_URL=https://perpustakaan.example.go.id

APP_LOCALE=id
APP_FALLBACK_LOCALE=id

LOG_CHANNEL=stack
LOG_LEVEL=warning

DB_CONNECTION=mysql
DB_HOST=localhost
DB_PORT=3306
DB_DATABASE=db_perpustakaan
DB_USERNAME=perpustakaan
DB_PASSWORD=GANTI_PASSWORD_DATABASE
DB_SOCKET=/run/mysqld/mysqld.sock

SESSION_DRIVER=database
CACHE_STORE=database
QUEUE_CONNECTION=database
```

Pada arsitektur ini koneksi database menggunakan Unix Socket:

```text
/run/mysqld/mysqld.sock
```

Variabel:

```env
DB_SOCKET=/run/mysqld/mysqld.sock
```

digunakan untuk menentukan lokasi Unix Socket MySQL.


## 12. Memeriksa Konfigurasi Unix Socket Laravel

Periksa:

```bash
grep -n "unix_socket" \
  /var/apps/perpustakaan/app/config/database.php
```

Pada konfigurasi koneksi MySQL Laravel pastikan terdapat:

```php
'unix_socket' => env('DB_SOCKET', ''),
```

Contoh bagian konfigurasi MySQL:

```php
'mysql' => [
    'driver' => 'mysql',
    'url' => env('DB_URL'),
    'host' => env('DB_HOST', '127.0.0.1'),
    'port' => env('DB_PORT', '3306'),
    'database' => env('DB_DATABASE', 'laravel'),
    'username' => env('DB_USERNAME', 'root'),
    'password' => env('DB_PASSWORD', ''),
    'unix_socket' => env('DB_SOCKET', ''),
    'charset' => env('DB_CHARSET', 'utf8mb4'),
    'collation' => env('DB_COLLATION', 'utf8mb4_unicode_ci'),
],
```

Dengan demikian Laravel/PDO menggunakan:

```text
/run/mysqld/mysqld.sock
```

untuk mengakses MySQL pada host.


## 13. Generate APP_KEY

Untuk aplikasi Laravel baru, generate `APP_KEY` sebelum permission `.env` dikunci.

Jalankan:

```bash
docker run --rm \
  -v /var/apps/perpustakaan/app:/var/www/html \
  -w /var/www/html \
  local/php:8.3 \
  php artisan key:generate
```

Periksa:

```bash
grep '^APP_KEY=' /var/apps/perpustakaan/app/.env
```

Target:

```text
APP_KEY=base64:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx=
```

> **Penting:** Jangan mengganti `APP_KEY` sembarangan pada aplikasi production yang sudah berjalan. `APP_KEY` digunakan Laravel untuk fungsi kriptografi dan data terenkripsi.


## 14. Mengamankan Permission `.env`

PHP-FPM berjalan sebagai:

```text
www-data
```

File `.env` harus dapat dibaca PHP-FPM, tetapi tidak perlu writable oleh web server.

Atur:

```bash
chown root:www-data /var/apps/perpustakaan/app/.env
chmod 640 /var/apps/perpustakaan/app/.env
```

Periksa:

```bash
ls -lah /var/apps/perpustakaan/app/.env
```

Test sebagai `www-data`:

```bash
docker exec -u www-data perpustakaan-php \
  grep '^APP_ENV=' /var/www/html/.env
```

Target:

```text
APP_ENV=production
```

Pastikan `.env` tidak writable:

```bash
docker exec -u www-data perpustakaan-php \
  test ! -w /var/www/html/.env && echo "ENV READ ONLY"
```

Target:

```text
ENV READ ONLY
```


## 15. Memeriksa Koneksi Database

Periksa konfigurasi Laravel:

```bash
docker exec perpustakaan-php \
  php /var/www/html/artisan about
```

Kemudian:

```bash
docker exec perpustakaan-php \
  php /var/www/html/artisan migrate:status
```

Jika database masih baru dan tabel migration belum tersedia, pesan mengenai tabel migration yang belum ada dapat terjadi sebelum migration pertama dijalankan.


## 16. Menjalankan Migration

Jalankan:

```bash
docker exec perpustakaan-php \
  php /var/www/html/artisan migrate --force
```

Periksa:

```bash
docker exec perpustakaan-php \
  php /var/www/html/artisan migrate:status
```

Pastikan migration yang diperlukan berstatus:

```text
Ran
```


## 17. Seeder

Periksa terlebih dahulu:

```bash
ls -lah /var/apps/perpustakaan/app/database/seeders/
```

Baca:

```bash
cat /var/apps/perpustakaan/app/database/seeders/DatabaseSeeder.php
```

Jika memang diperlukan:

```bash
docker exec perpustakaan-php \
  php /var/www/html/artisan db:seed --force
```

> Jangan menjalankan seeder pada production tanpa memeriksa isinya terlebih dahulu.


## 18. Restore Database Lama

Jika aplikasi memiliki database lama, jangan langsung menimpa database hasil migration.

Misalnya dump:

```text
/root/perpustakaan-backup.sql
```

Buat database sementara:

```sql
CREATE DATABASE db_perpustakaan_old
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;
```

Restore:

```bash
mysql -u root db_perpustakaan_old \
  < /root/perpustakaan-backup.sql
```

Sekarang tersedia:

```text
db_perpustakaan       = database aplikasi baru
db_perpustakaan_old   = database hasil restore
```

Data dapat dibandingkan dan dipindahkan secara terkontrol.


## 19. Membandingkan Database Lama dan Baru

Gunakan `COUNT(*)` untuk memperoleh jumlah aktual.

Contoh:

```sql
SELECT 'OLD' AS sumber, COUNT(*) AS jumlah
FROM db_perpustakaan_old.anggota

UNION ALL

SELECT 'NEW', COUNT(*)
FROM db_perpustakaan.anggota;
```

Periksa record lama yang belum terdapat pada database baru:

```sql
SELECT COUNT(*) AS old_tidak_ada_di_new
FROM db_perpustakaan_old.anggota o
LEFT JOIN db_perpustakaan.anggota n
    ON n.id = o.id
WHERE n.id IS NULL;
```

Sebaliknya:

```sql
SELECT COUNT(*) AS new_tidak_ada_di_old
FROM db_perpustakaan.anggota n
LEFT JOIN db_perpustakaan_old.anggota o
    ON o.id = n.id
WHERE o.id IS NULL;
```

Hindari hanya mengandalkan:

```text
information_schema.tables.table_rows
```

karena pada beberapa storage engine nilainya dapat berupa estimasi.


## 20. Persistent Storage Laravel

Source:

```text
/var/apps/perpustakaan/app
```

dipasang:

```text
/var/www/html:ro
```

Runtime menggunakan bind mount terpisah:

```text
/var/apps/perpustakaan/data/storage-app
->
/var/www/html/storage/app
```

```text
/var/apps/perpustakaan/data/storage-framework
->
/var/www/html/storage/framework
```

```text
/var/apps/perpustakaan/data/storage-logs
->
/var/www/html/storage/logs
```

```text
/var/apps/perpustakaan/data/bootstrap-cache
->
/var/www/html/bootstrap/cache
```

Dengan demikian container tetap:

```text
read_only
```

tetapi Laravel dapat menulis pada direktori runtime yang memang diperlukan.


## 21. Struktur Upload Aplikasi

Contoh aplikasi perpustakaan menggunakan public dan private storage.


### File Public

```text
storage/app/public/
├── anggota/
│   └── foto/
├── template_kartu/
├── institutions/
└── avatars/
```

Contoh:

```text
Foto anggota
storage/app/public/anggota/foto/
```

```text
Template kartu anggota
storage/app/public/template_kartu/
```

```text
Logo instansi
storage/app/public/institutions/
```

```text
Avatar admin/user
storage/app/public/avatars/
```


### File Private

Dokumen sensitif tidak disimpan pada public storage:

```text
storage/app/
└── anggota/
    ├── identitas/
    ├── kk/
    └── akta/
```

Contoh:

```text
Scan KTP
storage/app/anggota/identitas/
```

```text
Kartu Keluarga
storage/app/anggota/kk/
```

```text
Akta Kelahiran
storage/app/anggota/akta/
```

KTP, KK, Akta, dan dokumen sensitif lainnya tidak boleh dapat diakses langsung menggunakan URL publik.


## 22. Sinkronisasi Initial Asset

Developer terkadang menyimpan initial asset pada repository:

```text
storage/app/public/
```

misalnya:

```text
storage/app/public/institutions/
storage/app/public/template_kartu/
```

Production menggunakan:

```text
/var/apps/perpustakaan/data/storage-app/public/
```

Pada initial deployment:

```bash
cp -a \
  /var/apps/perpustakaan/app/storage/app/public/. \
  /var/apps/perpustakaan/data/storage-app/public/
```

Atur ownership:

```bash
chown -R www-data:www-data \
  /var/apps/perpustakaan/data/storage-app/public
```

Permission:

```bash
chmod -R u=rwX,g=rwX,o= \
  /var/apps/perpustakaan/data/storage-app/public
```

Perlu dipahami bahwa:

```text
/var/apps/perpustakaan/app/storage/app/public/
```

dan:

```text
/var/apps/perpustakaan/data/storage-app/public/
```

merupakan dua direktori berbeda.

Direktori pertama merupakan bagian source Git.

Direktori kedua merupakan persistent production storage.

Jangan menggunakan proses sinkronisasi yang menghapus file pada destination karena persistent storage dapat berisi upload pengguna yang tidak terdapat pada repository Git.


## 23. Test Writable Laravel

Jalankan:

```bash
docker exec -u www-data perpustakaan-php sh -c '
touch /var/www/html/storage/logs/test-write &&
touch /var/www/html/storage/framework/cache/test-write &&
touch /var/www/html/storage/framework/sessions/test-write &&
touch /var/www/html/storage/framework/views/test-write &&
touch /var/www/html/bootstrap/cache/test-write &&
echo "SEMUA WRITABLE"
'
```

Target:

```text
SEMUA WRITABLE
```

Hapus file test:

```bash
docker exec -u www-data perpustakaan-php sh -c '
rm -f /var/www/html/storage/logs/test-write
rm -f /var/www/html/storage/framework/cache/test-write
rm -f /var/www/html/storage/framework/sessions/test-write
rm -f /var/www/html/storage/framework/views/test-write
rm -f /var/www/html/bootstrap/cache/test-write
'
```


## 24. Memeriksa Laravel

Jalankan:

```bash
docker exec perpustakaan-php \
  php /var/www/html/artisan about
```

Pastikan antara lain:

```text
Application Name    Aplikasi Perpustakaan
Laravel Version     12.x
PHP Version         8.3.x
Environment         production
Debug Mode          OFF
Database            mysql
```


## 25. Build Frontend dengan Vite

Periksa:

```bash
cd /var/apps/perpustakaan/app
cat package.json
```

Jika aplikasi menggunakan Vite, production membutuhkan:

```text
public/build/
```

Build dapat dilakukan pada komputer developer atau build environment:

```bash
npm ci
npm run build
```

Hasil:

```text
public/build/
├── assets/
└── manifest.json
```

Jika strategi deployment menggunakan hasil build yang disimpan pada repository, developer harus memastikan `public/build/` ikut tersedia pada branch production.

Server production kemudian cukup memperoleh hasil build melalui:

```bash
git pull --ff-only origin main
```

Periksa:

```bash
ls -lah public/build/
ls -lah public/build/assets/
```

Pastikan manifest tersedia:

```bash
test -f public/build/manifest.json \
  && echo "VITE BUILD OK" \
  || echo "VITE BUILD TIDAK DITEMUKAN"
```

Target:

```text
VITE BUILD OK
```

Dengan metode ini Node.js tidak harus dipasang pada application server production.


## 26. Konfigurasi Nginx Application Server

Buat:

```text
/etc/nginx/sites-available/perpustakaan.conf
```

Isi:

```nginx
server {
    listen 8080;
    server_name perpustakaan.example.go.id;

    root /var/apps/perpustakaan/app/public;
    index index.php index.html;

    charset utf-8;

    access_log /var/log/nginx/perpustakaan.access.log;
    error_log  /var/log/nginx/perpustakaan.error.log warn;

    client_max_body_size 100M;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    # =========================================================
    # Laravel PUBLIC storage
    # PHP/script tidak boleh dieksekusi dari sini
    # =========================================================
    location ^~ /storage/ {
        alias /var/apps/perpustakaan/data/storage-app/public/;

        try_files $uri =404;

        if ($uri ~* "\.(php|php[0-9]*|phtml|pht|phar|phps|cgi|pl|py|sh)$") {
            return 403;
        }

        add_header X-Content-Type-Options "nosniff" always;

        autoindex off;
    }

    # =========================================================
    # PHP hanya dijalankan dari Laravel public/
    # =========================================================
    location ~ \.php$ {
        try_files $uri =404;

        include fastcgi_params;

        fastcgi_param SCRIPT_FILENAME /var/www/html/public$fastcgi_script_name;
        fastcgi_param DOCUMENT_ROOT /var/www/html/public;

        fastcgi_pass unix:/run/php/perpustakaan.sock;
    }

    # =========================================================
    # Block hidden files except .well-known
    # =========================================================
    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```


## 27. Mengaktifkan Nginx Site

Buat symbolic link:

```bash
ln -s \
  /etc/nginx/sites-available/perpustakaan.conf \
  /etc/nginx/sites-enabled/perpustakaan.conf
```

Tes:

```bash
nginx -t
```

Target:

```text
syntax is ok
test is successful
```

Reload:

```bash
systemctl reload nginx
```


## 28. Test Backend Nginx

Jalankan:

```bash
curl -i \
  -H "Host: perpustakaan.example.go.id" \
  http://127.0.0.1:8080/
```

Pastikan response berasal dari aplikasi Laravel.

Jika Laravel dikonfigurasi menggunakan URL HTTPS, aplikasi dapat melakukan redirect ke:

```text
https://perpustakaan.example.go.id
```


## 29. Reverse Proxy HTTPS

Arsitektur:

```text
Internet
    |
    | HTTPS :443
    v
Reverse Proxy Nginx
    |
    | HTTP :8080
    v
Application Server
```

Contoh konfigurasi reverse proxy:

```nginx
server {
    listen 80;

    server_name perpustakaan.example.go.id;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;

    server_name perpustakaan.example.go.id;

    ssl_certificate /path/to/certificate.crt;
    ssl_certificate_key /path/to/private.key;

    client_body_buffer_size 10M;
    client_max_body_size 128M;

    location / {
        proxy_pass http://IP_APPLICATION_SERVER:8080;

        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Port $server_port;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;

        proxy_set_header Accept-Encoding "";
    }
}
```

Ganti:

```text
IP_APPLICATION_SERVER
```

dengan alamat IP application server sebenarnya.


## 30. Hardening Direktori Upload

Salah satu risiko aplikasi web adalah file upload berbahaya.

Misalnya attacker berhasil meng-upload:

```text
shell.php
```

ke:

```text
storage/app/public/anggota/foto/shell.php
```

Jika web server salah dikonfigurasi, file tersebut berpotensi dieksekusi.

Karena itu digunakan:

```nginx
location ^~ /storage/
```

Penggunaan `^~` memastikan request di bawah `/storage/` ditangani oleh location tersebut dan tidak diproses oleh regex PHP:

```nginx
location ~ \.php$
```

Selain itu extension script berikut ditolak:

```text
.php
.php5
.php7
.php8
.phtml
.pht
.phar
.phps
.cgi
.pl
.py
.sh
```

Konfigurasi Nginx merupakan salah satu lapisan pertahanan.

Aplikasi Laravel tetap harus melakukan validasi file upload pada level aplikasi.


## 31. Test Anti Eksekusi PHP

Buat file pengujian:

```bash
cat > /var/apps/perpustakaan/data/storage-app/public/test-security.php <<'EOF'
<?php echo "PHP_EXECUTED_DANGER"; ?>
EOF
```

Atur ownership:

```bash
chown www-data:www-data \
  /var/apps/perpustakaan/data/storage-app/public/test-security.php
```

Test:

```bash
curl -i \
  -H "Host: perpustakaan.example.go.id" \
  http://127.0.0.1:8080/storage/test-security.php
```

Target:

```text
HTTP/1.1 403 Forbidden
```

Tidak boleh muncul:

```text
PHP_EXECUTED_DANGER
```

Jika response:

```text
403 Forbidden
```

dan kode PHP tidak dijalankan, hardening bekerja.

Hapus:

```bash
rm -f \
  /var/apps/perpustakaan/data/storage-app/public/test-security.php
```


## 32. Public Storage Tanpa `php artisan storage:link`

Laravel biasanya menggunakan:

```bash
php artisan storage:link
```

untuk membuat:

```text
public/storage
    ->
storage/app/public
```

Pada arsitektur ini symbolic link tersebut tidak diperlukan karena Nginx langsung melayani persistent public storage:

```nginx
location ^~ /storage/ {
    alias /var/apps/perpustakaan/data/storage-app/public/;
}
```

Keuntungannya:

1. Source Laravel tetap read-only pada runtime.
2. Upload terpisah dari repository Git.
3. Persistent storage tidak hilang ketika container diganti.
4. Nginx dapat mengontrol langsung akses `/storage/`.
5. PHP dapat diblokir pada seluruh public upload directory.


## 33. Test File Public Storage

Misalnya tersedia:

```text
/var/apps/perpustakaan/data/storage-app/public/institutions/logo.png
```

File tersebut seharusnya dapat diakses melalui:

```text
https://perpustakaan.example.go.id/storage/institutions/logo.png
```

Test backend:

```bash
curl -I \
  -H "Host: perpustakaan.example.go.id" \
  http://127.0.0.1:8080/storage/institutions/logo.png
```

Target:

```text
HTTP/1.1 200 OK
Content-Type: image/png
```


## 34. Update Aplikasi dari GitHub

Masuk:

```bash
cd /var/apps/perpustakaan/app
```

Periksa:

```bash
git status
```

Pastikan:

```text
nothing to commit, working tree clean
```

Fetch:

```bash
git fetch origin
```

Periksa:

```bash
git status
```

Update:

```bash
git pull --ff-only origin main
```

Penggunaan:

```text
--ff-only
```

membantu mencegah server production membuat merge commit secara tidak sengaja.


## 35. Update Dependency Composer

Jika setelah `git pull` terdapat perubahan:

```text
composer.json
composer.lock
```

jalankan:

```bash
docker run --rm \
  -v /var/apps/perpustakaan/app:/var/www/html \
  -w /var/www/html \
  local/php:8.3 \
  composer install \
    --no-dev \
    --optimize-autoloader \
    --no-interaction
```

Temporary container tersebut memperoleh akses write ke source untuk memperbarui `vendor/`.

Container PHP-FPM production tetap menggunakan source read-only.


## 36. Backup Database Sebelum Migration Production

Sebelum menjalankan migration pada database production, lakukan backup.

Contoh:

```bash
mysqldump -u root \
  --single-transaction \
  --routines \
  --triggers \
  db_perpustakaan \
  > /root/db_perpustakaan_before_deploy.sql
```

Periksa:

```bash
ls -lh /root/db_perpustakaan_before_deploy.sql
```

Untuk deployment rutin gunakan timestamp:

```bash
mysqldump -u root \
  --single-transaction \
  --routines \
  --triggers \
  db_perpustakaan \
  > "/root/db_perpustakaan_$(date +%Y%m%d_%H%M%S).sql"
```


## 37. Menjalankan Migration Setelah Update

Periksa:

```bash
docker exec perpustakaan-php \
  php /var/www/html/artisan migrate:status
```

Setelah backup tersedia:

```bash
docker exec perpustakaan-php \
  php /var/www/html/artisan migrate --force
```

Periksa kembali:

```bash
docker exec perpustakaan-php \
  php /var/www/html/artisan migrate:status
```


## 38. Sinkronisasi Initial Asset Setelah Git Pull

Jika developer menambahkan initial asset baru ke:

```text
storage/app/public/
```

misalnya:

```text
storage/app/public/institutions/
storage/app/public/template_kartu/
```

sinkronkan:

```bash
cp -a \
  /var/apps/perpustakaan/app/storage/app/public/. \
  /var/apps/perpustakaan/data/storage-app/public/
```

Kemudian:

```bash
chown -R www-data:www-data \
  /var/apps/perpustakaan/data/storage-app/public
```

Atur permission:

```bash
chmod -R u=rwX,g=rwX,o= \
  /var/apps/perpustakaan/data/storage-app/public
```

> **Penting:** Sinkronisasi tidak wajib dilakukan pada setiap deployment. Jalankan hanya jika developer menambahkan initial asset baru.

Jangan menggunakan mekanisme sinkronisasi yang menghapus file destination karena persistent storage dapat berisi upload pengguna yang tidak terdapat pada repository Git.


## 39. Laravel Cache

Setelah deployment:

```bash
docker exec perpustakaan-php \
  php /var/www/html/artisan optimize:clear
```

Untuk production, jika aplikasi mendukungnya:

```bash
docker exec perpustakaan-php \
  php /var/www/html/artisan config:cache
```

Cache route:

```bash
docker exec perpustakaan-php \
  php /var/www/html/artisan route:cache
```

Cache view:

```bash
docker exec perpustakaan-php \
  php /var/www/html/artisan view:cache
```

Periksa:

```bash
docker exec perpustakaan-php \
  php /var/www/html/artisan about
```

> `route:cache` digunakan hanya jika seluruh route aplikasi kompatibel dengan route caching.


## 40. Restart atau Recreate Container

Jika konfigurasi Docker, PHP, atau PHP-FPM berubah:

```bash
cd /opt/docker/apps/perpustakaan
```

Jalankan:

```bash
docker compose up -d --force-recreate
```

Periksa:

```bash
docker ps --filter name=perpustakaan-php
```

Pastikan:

```text
healthy
```


## 41. Checklist Setelah Deployment

### Git

```bash
cd /var/apps/perpustakaan/app
git status
```

Target:

```text
nothing to commit, working tree clean
```


### Docker

```bash
docker ps --filter name=perpustakaan-php
```

Target:

```text
healthy
```


### PHP

```bash
docker exec perpustakaan-php php -v
```

Target:

```text
PHP 8.3.x
```


### Laravel

```bash
docker exec perpustakaan-php \
  php /var/www/html/artisan about
```

Pastikan:

```text
Environment     production
Debug Mode      OFF
Database        mysql
```


### MySQL Unix Socket pada Host

```bash
ls -lah /run/mysqld/mysqld.sock
```

Socket harus tersedia.


### MySQL Unix Socket dalam Container

```bash
docker exec perpustakaan-php \
  ls -lah /run/mysqld/mysqld.sock
```

Socket harus tersedia.


### Database

```bash
docker exec perpustakaan-php \
  php /var/www/html/artisan migrate:status
```

Pastikan migration yang diperlukan:

```text
Ran
```


### Nginx

```bash
nginx -t
```

Target:

```text
syntax is ok
test is successful
```


### PHP-FPM Socket

```bash
ls -lah /run/php/perpustakaan.sock
```


### `.env`

```bash
docker exec -u www-data perpustakaan-php \
  grep '^APP_ENV=' /var/www/html/.env
```

Target:

```text
APP_ENV=production
```

Pastikan tidak writable:

```bash
docker exec -u www-data perpustakaan-php \
  test ! -w /var/www/html/.env && echo "ENV READ ONLY"
```

Target:

```text
ENV READ ONLY
```


### Writable Storage

```bash
docker exec -u www-data perpustakaan-php \
  test -w /var/www/html/storage/framework && echo "WRITABLE"
```

Target:

```text
WRITABLE
```


### Vite

```bash
test -f /var/apps/perpustakaan/app/public/build/manifest.json \
  && echo "VITE BUILD OK" \
  || echo "VITE BUILD TIDAK DITEMUKAN"
```

Target:

```text
VITE BUILD OK
```


### Public Storage

```bash
curl -I \
  -H "Host: perpustakaan.example.go.id" \
  http://127.0.0.1:8080/storage/institutions/logo.png
```

Target:

```text
200 OK
```


### Anti Eksekusi PHP

Request terhadap file PHP di:

```text
/storage/
```

harus menghasilkan:

```text
403 Forbidden
```


## 42. Arsitektur Akhir

```text
                         INTERNET
                            |
                            | HTTPS :443
                            v
                 +-----------------------+
                 | Reverse Proxy Nginx   |
                 | SSL/TLS Termination   |
                 +-----------------------+
                            |
                            | HTTP :8080
                            v
                 +-----------------------+
                 | Application Nginx     |
                 +-----------------------+
                     |              |
                     |              |
          Static     |              | PHP Request
          Files      |              |
                     |              v
                     |    /run/php/perpustakaan.sock
                     |              |
                     |              v
                     |    +----------------------+
                     |    | Docker PHP-FPM 8.3   |
                     |    +----------------------+
                     |         |            |
                     |         |            |
                     |         |            +---- MySQL Socket
                     |         |                      |
                     |         v                      v
                     |    Laravel Source         MySQL Server
                     |       READ ONLY
                     |
                     v
            Persistent Public Storage
                  READ WRITE
```

Source Laravel:

```text
/var/apps/perpustakaan/app
```

Persistent runtime:

```text
/var/apps/perpustakaan/data
```

PHP-FPM:

```text
Docker PHP 8.3
```

PHP-FPM Socket:

```text
/run/php/perpustakaan.sock
```

MySQL:

```text
MySQL pada host
```

MySQL Socket:

```text
/run/mysqld/mysqld.sock
```

Public upload:

```text
/var/apps/perpustakaan/data/storage-app/public
```


## 43. Prinsip Keamanan Deployment

Deployment ini menerapkan beberapa prinsip defense-in-depth:

1. Source Laravel dipasang read-only pada container PHP-FPM runtime.
2. Hanya direktori runtime yang writable oleh aplikasi.
3. Container menggunakan `read_only: true`.
4. Container menggunakan `no-new-privileges`.
5. `/tmp` menggunakan `tmpfs`.
6. PHP-FPM menggunakan Unix Socket.
7. MySQL menggunakan Unix Socket pada arsitektur contoh ini.
8. Document root Nginx hanya menunjuk ke direktori `public/`.
9. Upload production dipisahkan dari source Git.
10. PHP tidak dapat dieksekusi dari `/storage/`.
11. Extension script pada `/storage/` ditolak.
12. `X-Content-Type-Options: nosniff` digunakan.
13. Directory listing dimatikan.
14. Hidden files diblokir.
15. Dokumen sensitif disimpan di private storage.
16. `APP_DEBUG=false` digunakan pada production.
17. `.env` dapat dibaca PHP-FPM tetapi tidak writable oleh web server.
18. Git deployment menggunakan `--ff-only`.
19. Database dibackup sebelum migration production.
20. Persistent storage harus dibackup terpisah dari source Git.
21. Initial asset dari Git tidak boleh dianggap sebagai pengganti persistent upload production.
22. File upload tetap harus divalidasi pada level aplikasi Laravel.
23. Vite production build harus tersedia sebelum aplikasi dilayani.
24. Composer memperoleh akses write hanya melalui temporary deployment container.


## 44. Prosedur Deployment Rutin

Setelah developer melakukan push ke repository, deployment production dapat dilakukan dengan urutan berikut.


### 44.1 Periksa Repository

```bash
cd /var/apps/perpustakaan/app
git status
```

Pastikan:

```text
nothing to commit, working tree clean
```


### 44.2 Update Source

```bash
git fetch origin
git status
git pull --ff-only origin main
```


### 44.3 Periksa Vite Build

```bash
test -f public/build/manifest.json \
  && echo "VITE BUILD OK" \
  || echo "VITE BUILD TIDAK DITEMUKAN"
```

Jika aplikasi menggunakan Vite, jangan lanjutkan deployment sebelum `manifest.json` tersedia.


### 44.4 Update Composer Jika Diperlukan

Jika:

```text
composer.json
composer.lock
```

berubah:

```bash
docker run --rm \
  -v /var/apps/perpustakaan/app:/var/www/html \
  -w /var/www/html \
  local/php:8.3 \
  composer install \
    --no-dev \
    --optimize-autoloader \
    --no-interaction
```


### 44.5 Sinkronkan Initial Asset Jika Diperlukan

Hanya jika developer menambahkan initial asset baru:

```bash
cp -a \
  /var/apps/perpustakaan/app/storage/app/public/. \
  /var/apps/perpustakaan/data/storage-app/public/
```

Rapikan ownership:

```bash
chown -R www-data:www-data \
  /var/apps/perpustakaan/data/storage-app/public
```

Permission:

```bash
chmod -R u=rwX,g=rwX,o= \
  /var/apps/perpustakaan/data/storage-app/public
```


### 44.6 Backup Database

```bash
mysqldump -u root \
  --single-transaction \
  --routines \
  --triggers \
  db_perpustakaan \
  > "/root/db_perpustakaan_$(date +%Y%m%d_%H%M%S).sql"
```


### 44.7 Periksa MySQL Socket

```bash
ls -lah /run/mysqld/mysqld.sock
```

Periksa dari container:

```bash
docker exec perpustakaan-php \
  ls -lah /run/mysqld/mysqld.sock
```


### 44.8 Jalankan Migration

```bash
docker exec perpustakaan-php \
  php /var/www/html/artisan migrate --force
```


### 44.9 Clear Cache

```bash
docker exec perpustakaan-php \
  php /var/www/html/artisan optimize:clear
```


### 44.10 Build Cache Production Jika Digunakan

```bash
docker exec perpustakaan-php \
  php /var/www/html/artisan config:cache
```

Jika kompatibel:

```bash
docker exec perpustakaan-php \
  php /var/www/html/artisan route:cache
```

```bash
docker exec perpustakaan-php \
  php /var/www/html/artisan view:cache
```


### 44.11 Periksa Nginx

```bash
nginx -t
```


### 44.12 Periksa Container

```bash
docker ps --filter name=perpustakaan-php
```


### 44.13 Periksa Laravel

```bash
docker exec perpustakaan-php \
  php /var/www/html/artisan about
```


### 44.14 Periksa Migration

```bash
docker exec perpustakaan-php \
  php /var/www/html/artisan migrate:status
```


### 44.15 Periksa Writable Storage

```bash
docker exec -u www-data perpustakaan-php \
  test -w /var/www/html/storage/framework \
  && echo "STORAGE WRITABLE"
```


### 44.16 Periksa `.env`

```bash
docker exec -u www-data perpustakaan-php \
  test -r /var/www/html/.env \
  && echo "ENV READABLE"
```

Pastikan tidak writable:

```bash
docker exec -u www-data perpustakaan-php \
  test ! -w /var/www/html/.env \
  && echo "ENV READ ONLY"
```


### 44.17 Test Backend

```bash
curl -I \
  -H "Host: perpustakaan.example.go.id" \
  http://127.0.0.1:8080/
```


### 44.18 Test Public Storage

```bash
curl -I \
  -H "Host: perpustakaan.example.go.id" \
  http://127.0.0.1:8080/storage/institutions/logo.png
```


### 44.19 Test Domain Production

Akses:

```text
https://perpustakaan.example.go.id
```

Pastikan halaman aplikasi, CSS, JavaScript, gambar, login, database, dan file upload berjalan normal.


## 45. Backup Persistent Storage

Database bukan satu-satunya data yang perlu dibackup.

Persistent storage:

```text
/var/apps/perpustakaan/data/
```

juga harus dibackup.

Direktori penting antara lain:

```text
storage-app/
storage-framework/
storage-logs/
bootstrap-cache/
```

Data yang paling penting adalah:

```text
storage-app/
```

karena dapat berisi:

```text
upload pengguna
foto anggota
logo instansi
dokumen private
dokumen administrasi
file aplikasi lainnya
```

Source Git bukan pengganti backup persistent storage.

Repository Git dan persistent production storage memiliki fungsi yang berbeda.


## 46. Penutup

Deployment Laravel pada server production tidak hanya membutuhkan PHP, database, dan web server.

Struktur penyimpanan, permission, isolasi container, mekanisme deployment, database, frontend asset, serta keamanan direktori upload perlu dirancang sejak awal.

Pada arsitektur ini komponen aplikasi dipisahkan berdasarkan fungsinya:

```text
Git Repository
      |
      v
Laravel Source
  READ ONLY
  saat runtime
      |
      +-----------------------+
      |                       |
      v                       v
PHP-FPM Docker         Persistent Storage
                           READ WRITE
                                |
                   +------------+------------+
                   |            |            |
                   v            v            v
                 Upload       Session       Log
```

Koneksi PHP:

```text
Nginx
   |
   | Unix Socket
   v
/run/php/perpustakaan.sock
   |
   v
PHP-FPM Docker
```

Koneksi database:

```text
Laravel
   |
   | DB_SOCKET
   v
/run/mysqld/mysqld.sock
   |
   v
MySQL pada Host
```

Source code tetap dikelola melalui Git, sedangkan data yang dihasilkan aplikasi disimpan pada persistent storage terpisah.

Pendekatan ini memberikan beberapa keuntungan:

- update source tidak menghapus file upload;
- container PHP-FPM dapat menggunakan filesystem read-only;
- hanya direktori runtime tertentu yang writable;
- data runtime dapat dibackup secara terpisah;
- file sensitif dapat dipisahkan dari public storage;
- direktori upload dapat di-hardening pada level Nginx;
- PHP-FPM diakses melalui Unix Socket;
- MySQL pada host dapat diakses melalui Unix Socket;
- `.env` dapat dibuat readable tetapi tidak writable oleh PHP-FPM;
- Vite build dapat diverifikasi sebelum deployment;
- Composer dijalankan melalui temporary container;
- deployment Git menggunakan `--ff-only`;
- proses migration diawali dengan backup database;
- proses deployment dapat diuji menggunakan checklist yang konsisten.

Dengan pola tersebut, deployment Laravel menjadi lebih terstruktur, mudah dikelola, dan memiliki fondasi keamanan yang lebih baik untuk lingkungan production.
