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

    Aplikasi Perpustakaan

Domain contoh:

    https://perpustakaan.example.go.id

Nama aplikasi:

    perpustakaan

Source code aplikasi:

    /var/apps/perpustakaan/app

Persistent data:

    /var/apps/perpustakaan/data

Konfigurasi Docker:

    /opt/docker/apps/perpustakaan

Arsitektur deployment:

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

Pada arsitektur ini, source code Laravel dipisahkan dari data runtime. Source code dikelola menggunakan Git, sedangkan file upload, cache, session, log, dan data runtime Laravel disimpan pada persistent storage.

Tujuannya adalah agar update aplikasi tidak menghapus data yang dihasilkan selama aplikasi berjalan.


## 2. Struktur Direktori

Source code aplikasi dan data runtime sebaiknya dipisahkan.

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

Konfigurasi Docker disimpan pada:

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

Buat direktori konfigurasi:

```bash
mkdir -p /opt/docker/apps/perpustakaan
```

Buat:

```text
/opt/docker/apps/perpustakaan/docker-compose.yml
```

Contoh konfigurasi:

```yaml
services:

  app:
    container_name: perpustakaan-php
    read_only: true

    image: local/php:8.3

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

      # Laravel source - READ ONLY
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

> **Catatan:** `local/php:8.3` pada contoh di atas merupakan image PHP 8.3 yang telah disiapkan sebelumnya dan harus memiliki PHP-FPM, Composer, serta ekstensi PHP yang dibutuhkan Laravel.


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

Masuk ke direktori:

```bash
cd /opt/docker/apps/perpustakaan
```

Jalankan:

```bash
docker compose up -d
```

Periksa container:

```bash
docker ps -a --filter name=perpustakaan-php
```

Target:

```text
Up ... (healthy)
```

Periksa Unix Socket:

```bash
ls -lah /run/php/
```

Seharusnya tersedia:

```text
perpustakaan.sock
```

Periksa detail socket:

```bash
stat /run/php/perpustakaan.sock
```


## 8. Memeriksa Docker Mount

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

dan:

```text
/var/apps/perpustakaan/data/storage-app
->
/var/www/html/storage/app rw
```

Dengan demikian:

```text
Source Laravel  = READ ONLY
Runtime Laravel = READ WRITE
```


## 9. Install Dependency Composer

Karena source pada container production dibuat read-only, Composer dapat dijalankan menggunakan temporary container yang memperoleh akses write ke source.

Jalankan:

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

Direktori berikut harus tersedia:

```text
vendor/
```


## 10. Membuat File .env

Masuk ke source:

```bash
cd /var/apps/perpustakaan/app
```

Copy template:

```bash
cp .env.example .env
```

Edit:

```bash
nano .env
```

Contoh:
> **Peringatan Keamanan**
>
> Jangan pernah memasukkan file `.env` production ke repository Git atau mempublikasikannya dalam dokumentasi.
> File `.env` dapat berisi informasi sensitif seperti `APP_KEY`, password database, API key, token, dan credential layanan lainnya.
> Seluruh credential dalam tutorial ini hanyalah placeholder dan harus diganti pada server production.
>

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

SESSION_DRIVER=database
CACHE_STORE=database
QUEUE_CONNECTION=database
```

Pada contoh arsitektur ini, MySQL berada pada host dan:

```text
/run/mysqld
```

di-mount ke container:

```yaml
- /run/mysqld:/run/mysqld:ro
```

Karena itu aplikasi dapat menggunakan Unix Socket MySQL.

> Jika MySQL juga dijalankan menggunakan Docker, konfigurasi database dapat berbeda. `DB_HOST` biasanya menggunakan nama service/container MySQL, bukan `localhost`.


## 11. Generate APP_KEY

Untuk aplikasi Laravel baru, generate APP_KEY sebelum permission `.env` dikunci.

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

Contoh:

```text
APP_KEY=base64:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx=
```

> **Penting:** Jangan mengganti `APP_KEY` sembarangan pada aplikasi production yang sudah berjalan. APP_KEY digunakan Laravel untuk fungsi kriptografi dan data terenkripsi.


## 12. Mengamankan Permission .env

PHP-FPM berjalan sebagai:

```text
www-data
```

File `.env` perlu dapat dibaca oleh PHP-FPM, tetapi tidak perlu writable oleh web server.

Atur:

```bash
chown root:www-data /var/apps/perpustakaan/app/.env
chmod 640 /var/apps/perpustakaan/app/.env
```

Periksa:

```bash
ls -lah /var/apps/perpustakaan/app/.env
```

Tes dari container:

```bash
docker exec -u www-data perpustakaan-php \
  grep '^APP_KEY=' /var/www/html/.env
```

Jika APP_KEY tampil, berarti `www-data` dapat membaca `.env`.


## 13. Membuat Database MySQL

Login:

```bash
mysql -u root
```

Buat database:

```sql
CREATE DATABASE db_perpustakaan
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;
```

Buat user:

```sql
CREATE USER 'perpustakaan'@'localhost'
IDENTIFIED BY 'GANTI_DENGAN_PASSWORD_KUAT';
```

Berikan privilege:

```sql
GRANT ALL PRIVILEGES
ON db_perpustakaan.*
TO 'perpustakaan'@'localhost';

FLUSH PRIVILEGES;
```

Periksa:

```sql
SHOW GRANTS FOR 'perpustakaan'@'localhost';
```

Keluar:

```sql
EXIT;
```


## 14. Test Koneksi MySQL

Test dari host:

```bash
mysql -u perpustakaan -p db_perpustakaan
```

Kemudian test melalui Laravel:

```bash
docker exec perpustakaan-php \
  php /var/www/html/artisan migrate:status
```

Jika database benar-benar baru, Laravel dapat menampilkan:

```text
Migration table not found.
```

Ini normal jika migration belum pernah dijalankan.


## 15. Menjalankan Migration

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


## 16. Menjalankan Seeder

Periksa terlebih dahulu:

```bash
cd /var/apps/perpustakaan/app

ls -lah database/seeders/
```

Baca isi seeder:

```bash
cat database/seeders/DatabaseSeeder.php
```

Jika memang diperlukan:

```bash
docker exec perpustakaan-php \
  php /var/www/html/artisan db:seed --force
```

> **Peringatan:** Jangan menjalankan seeder pada production tanpa memeriksa isinya terlebih dahulu. Seeder dapat membuat akun default, mengubah data, atau menambahkan data yang sebenarnya hanya ditujukan untuk development/testing.


## 17. Restore Database Lama

Jika aplikasi memiliki database lama, sebaiknya jangan langsung menimpa database baru hasil migration.

Misalnya tersedia:

```text
/root/perpustakaan-backup.sql
```

Login MySQL:

```bash
mysql -u root
```

Buat database sementara:

```sql
CREATE DATABASE db_perpustakaan_old
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;
```

Keluar:

```sql
EXIT;
```

Restore:

```bash
mysql -u root db_perpustakaan_old \
  < /root/perpustakaan-backup.sql
```

Sekarang terdapat:

```text
db_perpustakaan       = database aplikasi baru
db_perpustakaan_old   = database hasil restore
```

Dengan metode ini, data lama dapat dibandingkan dan dipindahkan secara terkontrol.


## 18. Membandingkan Database Lama dan Baru

Gunakan `COUNT(*)` untuk memperoleh jumlah record aktual.

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

untuk verifikasi jumlah data karena pada beberapa storage engine nilainya dapat berupa estimasi.


## 19. Persistent Storage Laravel

Source:

```text
/var/apps/perpustakaan/app
```

dipasang ke:

```text
/var/www/html:ro
```

Sedangkan direktori runtime menggunakan bind mount terpisah:

```text
/var/apps/perpustakaan/data/storage-app
        |
        +--> /var/www/html/storage/app
```

```text
/var/apps/perpustakaan/data/storage-framework
        |
        +--> /var/www/html/storage/framework
```

```text
/var/apps/perpustakaan/data/storage-logs
        |
        +--> /var/www/html/storage/logs
```

```text
/var/apps/perpustakaan/data/bootstrap-cache
        |
        +--> /var/www/html/bootstrap/cache
```

Dengan demikian container tetap menggunakan:

```yaml
read_only: true
```

tetapi Laravel masih dapat menulis pada direktori runtime yang memang diperlukan.


## 20. Struktur Upload Aplikasi

Sebagai contoh, aplikasi perpustakaan memiliki file public dan private.

### File Public

```text
storage/app/public/
├── anggota/
│   └── foto/
├── template_kartu/
├── institutions/
└── avatars/
```

Contoh penggunaannya:

| Jenis File | Lokasi |
|---|---|
| Foto anggota | `storage/app/public/anggota/foto/` |
| Template kartu anggota | `storage/app/public/template_kartu/` |
| Logo instansi | `storage/app/public/institutions/` |
| Avatar admin/user | `storage/app/public/avatars/` |

File tersebut memang dirancang agar dapat ditampilkan melalui web.

### File Private

Dokumen sensitif tidak disimpan di public storage:

```text
storage/app/
└── anggota/
    ├── identitas/
    ├── kk/
    └── akta/
```

Contoh:

| Jenis Dokumen | Lokasi |
|---|---|
| Scan KTP/Kartu Identitas | `storage/app/anggota/identitas/` |
| Kartu Keluarga | `storage/app/anggota/kk/` |
| Akta Kelahiran | `storage/app/anggota/akta/` |

KTP, KK, Akta, dan dokumen sensitif lainnya sebaiknya tidak dapat diakses langsung menggunakan URL publik.


## 21. Sinkronisasi Initial Asset

Developer terkadang menyimpan initial asset pada repository:

```text
storage/app/public/
```

Misalnya:

```text
storage/app/public/institutions/
storage/app/public/template_kartu/
```

Sedangkan production menggunakan persistent storage:

```text
/var/apps/perpustakaan/data/storage-app/public/
```

Pada initial deployment, copy:

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

Atur permission:

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

Direktori pertama merupakan bagian dari source Git.

Direktori kedua merupakan persistent production storage.

Upload yang dilakukan aplikasi di dalam container akan masuk ke persistent storage melalui Docker bind mount.


## 22. Test Writable Laravel

Jalankan sebagai `www-data`:

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

Hapus file pengujian:

```bash
docker exec -u www-data perpustakaan-php sh -c '
rm -f /var/www/html/storage/logs/test-write
rm -f /var/www/html/storage/framework/cache/test-write
rm -f /var/www/html/storage/framework/sessions/test-write
rm -f /var/www/html/storage/framework/views/test-write
rm -f /var/www/html/bootstrap/cache/test-write
'
```


## 23. Memeriksa Laravel

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

Yang paling penting untuk production:

```text
Environment    production
Debug Mode     OFF
```


## 24. Build Frontend dengan Vite

Periksa:

```bash
cd /var/apps/perpustakaan/app

cat package.json
```

Jika aplikasi menggunakan Vite, production membutuhkan hasil build:

```text
public/build/
```

Build dapat dilakukan pada komputer developer atau build environment:

```bash
npm ci
npm run build
```

Hasilnya biasanya:

```text
public/build/
├── assets/
└── manifest.json
```

Jika strategi deployment memang menyimpan hasil build ke repository, developer dapat melakukan commit dan push terhadap `public/build`.

Server production kemudian cukup melakukan:

```bash
git pull --ff-only origin main
```

Periksa:

```bash
ls -lah public/build/
ls -lah public/build/assets/
```

Dengan metode ini Node.js tidak harus dipasang pada application server production.


## 25. Konfigurasi Nginx

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

        # Mencegah MIME sniffing
        add_header X-Content-Type-Options "nosniff" always;

        # Jangan tampilkan isi directory
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

    # Block hidden files except .well-known
    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```


## 26. Mengaktifkan Nginx Site

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


## 27. Test Backend Nginx

Jalankan:

```bash
curl -i \
  -H "Host: perpustakaan.example.go.id" \
  http://127.0.0.1:8080/
```

Pastikan response berasal dari aplikasi Laravel.

Jika Laravel dikonfigurasi untuk menggunakan URL HTTPS, aplikasi dapat memberikan redirect menuju:

```text
https://perpustakaan.example.go.id
```


## 28. Reverse Proxy HTTPS

Contoh arsitektur:

```text
Internet
    |
    | HTTPS :443
    v
Reverse Proxy
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

Dengan arsitektur ini, SSL/TLS ditangani oleh reverse proxy, sedangkan application server menerima request melalui HTTP pada port `8080`.


## 29. Hardening Direktori Upload

Salah satu risiko penting pada aplikasi web adalah upload file berbahaya.

Misalnya attacker berhasil meng-upload:

```text
shell.php
```

ke:

```text
storage/app/public/anggota/foto/shell.php
```

Jika web server salah dikonfigurasi, file tersebut berpotensi diteruskan ke PHP-FPM dan dieksekusi.

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

Konfigurasi ini merupakan lapisan tambahan. Aplikasi Laravel tetap harus melakukan validasi file upload pada level aplikasi.


## 30. Test Anti Eksekusi PHP

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

Jika response adalah:

```text
403 Forbidden
```

dan kode PHP tidak dijalankan, hardening bekerja.

Hapus file test:

```bash
rm -f \
  /var/apps/perpustakaan/data/storage-app/public/test-security.php
```


## 31. Public Storage Tanpa `php artisan storage:link`

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

Pada arsitektur dalam tutorial ini symbolic link tersebut tidak diperlukan karena Nginx langsung melayani persistent public storage:

```nginx
location ^~ /storage/ {
    alias /var/apps/perpustakaan/data/storage-app/public/;
}
```

Keuntungannya:

1. Source Laravel tetap read-only.
2. Upload terpisah dari repository Git.
3. Persistent storage tidak hilang ketika container diganti.
4. Nginx dapat mengontrol langsung akses `/storage/`.
5. PHP dapat diblokir pada seluruh public upload directory.


## 32. Test File Public Storage

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


## 33. Update Aplikasi dari GitHub

Masuk ke source:

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


## 34. Update Dependency Composer

Jika setelah `git pull` terdapat perubahan pada:

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


## 35. Backup Database Sebelum Migration Production

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

Untuk deployment yang rutin, nama backup sebaiknya menggunakan timestamp:

```bash
mysqldump -u root \
  --single-transaction \
  --routines \
  --triggers \
  db_perpustakaan \
  > "/root/db_perpustakaan_$(date +%Y%m%d_%H%M%S).sql"
```


## 36. Menjalankan Migration Setelah Update

Periksa terlebih dahulu:

```bash
docker exec perpustakaan-php \
  php /var/www/html/artisan migrate:status
```

Setelah backup tersedia, jalankan:

```bash
docker exec perpustakaan-php \
  php /var/www/html/artisan migrate --force
```

Periksa kembali:

```bash
docker exec perpustakaan-php \
  php /var/www/html/artisan migrate:status
```


## 37. Sinkronisasi Initial Asset Setelah Git Pull

Jika developer menambahkan **initial asset baru** ke:

```text
storage/app/public/
```

misalnya:

```text
storage/app/public/institutions/
storage/app/public/template_kartu/
```

sinkronkan ke persistent storage:

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

> **Penting:** Sinkronisasi ini tidak wajib dilakukan pada setiap deployment. Jalankan hanya jika developer memang menambahkan initial asset baru yang harus masuk ke persistent storage.

Jangan menggunakan mekanisme sinkronisasi yang menghapus file destination karena persistent storage dapat berisi upload pengguna yang tidak terdapat pada repository Git.


## 38. Laravel Cache

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

> `route:cache` digunakan jika seluruh route aplikasi kompatibel dengan route caching.


## 39. Restart atau Recreate Container

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


## 40. Checklist Setelah Deployment

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

### Writable Storage

```bash
docker exec -u www-data perpustakaan-php \
  test -w /var/www/html/storage/framework && echo "WRITABLE"
```

Target:

```text
WRITABLE
```

### Public Storage

Test file gambar:

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


## 41. Arsitektur Akhir

Arsitektur akhir deployment:

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

Public upload:

```text
/var/apps/perpustakaan/data/storage-app/public
```


## 42. Prinsip Keamanan Deployment

Deployment ini menerapkan beberapa prinsip defense-in-depth:

1. Source Laravel dipasang read-only.
2. Hanya direktori runtime yang writable.
3. Container menggunakan `read_only: true`.
4. Container menggunakan `no-new-privileges`.
5. `/tmp` menggunakan `tmpfs`.
6. PHP-FPM menggunakan Unix Socket.
7. MySQL dapat menggunakan Unix Socket.
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


## 43. Prosedur Deployment Rutin

Setelah developer melakukan push ke repository, deployment production dapat dilakukan dengan urutan berikut.

### 43.1 Periksa Repository

```bash
cd /var/apps/perpustakaan/app

git status
```

Pastikan working tree bersih.

### 43.2 Update Source

```bash
git fetch origin
git status
git pull --ff-only origin main
```

### 43.3 Update Composer Jika Diperlukan

Jika `composer.json` atau `composer.lock` berubah:

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

### 43.4 Sinkronkan Initial Asset Jika Diperlukan

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

### 43.5 Backup Database

```bash
mysqldump -u root \
  --single-transaction \
  --routines \
  --triggers \
  db_perpustakaan \
  > "/root/db_perpustakaan_$(date +%Y%m%d_%H%M%S).sql"
```

### 43.6 Jalankan Migration

```bash
docker exec perpustakaan-php \
  php /var/www/html/artisan migrate --force
```

### 43.7 Clear Cache

```bash
docker exec perpustakaan-php \
  php /var/www/html/artisan optimize:clear
```

### 43.8 Periksa Nginx

```bash
nginx -t
```

### 43.9 Periksa Container

```bash
docker ps --filter name=perpustakaan-php
```

### 43.10 Periksa Laravel

```bash
docker exec perpustakaan-php \
  php /var/www/html/artisan about
```

### 43.11 Periksa Migration

```bash
docker exec perpustakaan-php \
  php /var/www/html/artisan migrate:status
```

### 43.12 Test Aplikasi

Test backend:

```bash
curl -I \
  -H "Host: perpustakaan.example.go.id" \
  http://127.0.0.1:8080/
```

Kemudian test aplikasi melalui domain production:

```text
https://perpustakaan.example.go.id
```


## 44. Penutup

Deployment Laravel pada server production tidak hanya membutuhkan PHP, database, dan web server. Struktur penyimpanan, permission, isolasi container, mekanisme deployment, serta keamanan direktori upload juga perlu dirancang sejak awal.

Pada arsitektur ini, komponen aplikasi dipisahkan berdasarkan fungsinya:

```text
Git Repository
      |
      v
Laravel Source
  READ ONLY
      |
      +---------------------+
      |                     |
      v                     v
PHP-FPM Docker       Persistent Storage
                         READ WRITE
                              |
                 +------------+------------+
                 |            |            |
                 v            v            v
               Upload       Session       Log
```

Source code tetap dikelola melalui Git, sedangkan data yang dihasilkan aplikasi disimpan pada persistent storage terpisah.

Pendekatan ini memberikan beberapa keuntungan:

- update source tidak menghapus file upload;
- container dapat dibuat lebih ketat dengan filesystem read-only;
- data runtime dapat dibackup secara terpisah;
- file sensitif dapat dipisahkan dari public storage;
- direktori upload dapat di-hardening pada level Nginx;
- PHP-FPM dan MySQL dapat diakses melalui Unix Socket;
- proses deployment menjadi lebih terstruktur dan dapat diuji.

Dengan pola tersebut, deployment Laravel menjadi lebih mudah dikelola sekaligus memberikan fondasi keamanan yang lebih baik untuk lingkungan production.
````
