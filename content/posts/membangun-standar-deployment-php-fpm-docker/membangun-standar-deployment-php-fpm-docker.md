---
title: "Membangun Standar Deployment PHP-FPM Docker untuk Web Server Production"
date: 2026-09-08
draft: false
description: "Panduan praktis membangun standar deployment aplikasi PHP menggunakan Docker PHP-FPM, Nginx, dan MariaDB dengan dukungan PHP 7.4, 8.3, 8.4, dan 8.5."

tags:
  - Docker
  - Docker PHP
  - PHP
  - PHP-FPM
  - Docker Compose
  - Nginx
  - MariaDB
  - Linux
  - Web Server
  - DevOps
  - Deployment
  - Container
  - Containerization
  - Security
  - Hardening
  - Cyber Security
  - Production Server
  - Infrastructure
  - Server Management

categories:
  - Docker
  - DevOps
  - Web Server

series:
  - "Docker PHP Production"

weight: 1
author: "NR Technology"

cover:
  image: "docker-php-production-cover.png"
  alt: "Membangun Standar Deployment PHP-FPM Docker untuk Web Server Production"
  caption: "Panduan membangun standar deployment PHP-FPM menggunakan Docker, Nginx, dan MariaDB untuk lingkungan server production."
---

# Membangun Standar Deployment PHP-FPM Docker untuk Web Server Production

Menjalankan banyak aplikasi PHP pada satu server production membutuhkan standar deployment yang konsisten.

Tanpa standar yang jelas, setiap aplikasi dapat memiliki konfigurasi yang berbeda, mulai dari versi PHP, extension, konfigurasi PHP-FPM, permission directory, konfigurasi Nginx, hingga lokasi penyimpanan file writable.

Pada artikel ini kita akan membangun repository bernama **`docker-php`** yang digunakan sebagai standar deployment PHP-FPM berbasis Docker.

Konsep yang digunakan:

- Nginx berjalan langsung pada host.
- MariaDB berjalan langsung pada host.
- Setiap aplikasi PHP memiliki container PHP-FPM sendiri.
- Source code aplikasi di-mount secara read-only.
- Directory yang membutuhkan write dipisahkan.
- Nginx berkomunikasi dengan PHP-FPM menggunakan Unix Socket.
- PHP-FPM berkomunikasi dengan MariaDB menggunakan TCP.
- Container menggunakan beberapa mekanisme hardening.
- Deployment aplikasi baru dapat dibuat menggunakan `create-php-app.sh`.
- Beberapa versi PHP dapat digunakan secara bersamaan.

---

## 1. Arsitektur

Arsitektur deployment:

```text
INTERNET
    |
    v
+-----------+
|   Nginx   |
|   HOST    |
+-----+-----+
      |
  Unix Socket
      |
+-----+------+-------------+
|            |             |
v            v             v
```

  myapp        myapp2        myapp3
  PHP 8.3      PHP 8.4       PHP 8.5

```text
|            |             |
+------------+-------------+
             |
          TCP 3306
             |
             v
       +-----------+
       |  MariaDB  |
       |   HOST    |
       +-----------+
```

Dengan arsitektur tersebut, PHP tidak perlu di-install langsung pada host.

Host menyediakan:

```text
Nginx
MariaDB
Docker
Git
```

Runtime PHP disediakan oleh container.

---

## 2. Mengapa Menggunakan PHP-FPM Docker?

Misalnya server menjalankan beberapa aplikasi:

```text
myapp
myapp2
myapp3
```

Aplikasi tersebut belum tentu menggunakan versi PHP yang sama.

Dengan Docker, masing-masing aplikasi dapat menggunakan runtime sendiri:

```text
myapp
└── PHP 8.3
```


```text
myapp2
└── PHP 8.4
```


```text
myapp3
└── PHP 8.5
```

Aplikasi legacy dapat menggunakan:

```text
legacy-app
└── PHP 7.4
```

Keuntungannya:

1. Versi PHP dapat berbeda antar aplikasi.
2. Dependency PHP lebih terisolasi.
3. Upgrade PHP dapat dilakukan per aplikasi.
4. Konfigurasi dapat disimpan sebagai kode.
5. Deployment menjadi lebih konsisten.
6. Resource container dapat dibatasi.
7. Risiko konflik antar aplikasi dapat dikurangi.
8. Rollback runtime menjadi lebih mudah.

---

## 3. Instalasi Repository

Repository `docker-php` digunakan sebagai standar template, Dockerfile, konfigurasi PHP-FPM, konfigurasi Nginx, dan script deployment.

Repository tersedia di GitHub:

```text
https://github.com/NRTechnology/docker-php
```

Clone repository ke `/opt/docker-php`:

```text
mkdir -p /opt
cd /opt
git clone https://github.com/NRTechnology/docker-php.git docker-php
```

Masuk ke repository:

```text
cd /opt/docker-php
```

Periksa isi:

```text
ls -lah
```

Periksa status Git:

```text
git status
```

Untuk mengambil perubahan terbaru:

```text
git pull --ff-only origin main
```

Penggunaan `--ff-only` direkomendasikan pada server production agar Git hanya melakukan fast-forward dan tidak membuat merge commit otomatis.

---

## 4. Struktur Repository

Struktur repository:

```text
/opt/docker-php/
├── README.md
├── create-php-app.sh
├── .gitignore
│
├── images/
│   ├── 7.4/
│   │   ├── Dockerfile
│   │   └── php.ini
│   ├── 8.3/
│   │   ├── Dockerfile
│   │   └── php.ini
│   ├── 8.4/
│   │   ├── Dockerfile
│   │   └── php.ini
│   └── 8.5/
│       ├── Dockerfile
│       └── php.ini
│
├── templates/
│   ├── docker/
│   │   └── docker-compose.yml
│   ├── nginx/
│   │   └── app.conf
│   └── php-fpm/
│       └── zz-custom.conf
│
└── scripts/
    ├── build-all.sh
    └── test-images.sh
```

Repository digunakan sebagai template dan tooling, bukan sebagai tempat source code aplikasi production.

---

## 5. Pemisahan Directory

### Repository Docker PHP

```text
/opt/docker-php
```

Digunakan untuk Dockerfile, `php.ini`, template, generator, script build, script testing, dan dokumentasi.

### Konfigurasi Docker Aplikasi

```text
/opt/docker-apps
```

Contoh:

```text
/opt/docker-apps/myapp/
├── docker-compose.yml
└── zz-custom.conf
```

### Source Code dan Data Aplikasi

```text
/var/apps
```

Contoh:

```text
/var/apps/myapp/
├── htdocs/
├── data/
│   └── writable/
├── logs/
└── backup/
```

---

## 6. Struktur Directory Aplikasi

Setiap aplikasi menggunakan:

```text
/var/apps/<application-name>/
├── htdocs/
├── data/
│   └── writable/
├── logs/
└── backup/
```

Source code:

```text
/var/apps/myapp/htdocs/
```

Source code di-mount read-only:

```text
/var/apps/myapp/htdocs:/var/www/html:ro
```

Directory writable dipisahkan dari source code.

Prinsip:

```text
Source Code
    |
    +-- Read Only
```


```text
Writable Data
    |
    +-- Read Write
```


```text
Backup
    |
    +-- Tidak diakses PHP-FPM
```

---

## 7. PHP Version

Runtime yang disediakan:

```text
PHP 7.4
PHP 8.3
PHP 8.4
PHP 8.5
```

Image lokal:

```text
local/php:7.4
local/php:8.3
local/php:8.4
local/php:8.5
```

PHP 7.4 dipertahankan untuk kebutuhan aplikasi legacy. Untuk aplikasi baru, gunakan versi PHP 8.x yang kompatibel.

---

## 8. PHP Extension

Baseline extension:

```text
bcmath
curl
exif
gd
intl
mbstring
mysqli
opcache
pcntl
pdo
pdo_mysql
pdo_pgsql
redis
xml
zip
```

Extension tambahan dapat ditambahkan apabila memang diperlukan aplikasi.

---

## 9. Konfigurasi php.ini

Baseline:

```text
[PHP]
expose_php = Off
default_charset = "UTF-8"
default_socket_timeout = 60
max_execution_time = 120
max_input_time = 120
max_input_vars = 5000
memory_limit = 256M
file_uploads = On
upload_max_filesize = 64M
post_max_size = 64M
max_file_uploads = 20
display_errors = Off
display_startup_errors = Off
log_errors = On
error_log = /proc/self/fd/2
error_reporting = E_ALL & ~E_DEPRECATED & ~E_STRICT
cgi.fix_pathinfo = 0
allow_url_fopen = On
allow_url_include = Off
session.use_strict_mode = 1
session.use_cookies = 1
session.use_only_cookies = 1
session.cookie_httponly = 1
session.cookie_secure = 1
session.cookie_samesite = Lax
realpath_cache_size = 4096K
realpath_cache_ttl = 600
date.timezone = Asia/Jakarta
```

Pada production, error tidak ditampilkan langsung kepada pengguna.

```text
display_errors = Off
log_errors = On
error_log = /proc/self/fd/2
```

---

## 10. OPcache

Konfigurasi:

```text
[opcache]
opcache.enable = 1
opcache.enable_cli = 1
opcache.memory_consumption = 128
opcache.interned_strings_buffer = 16
opcache.max_accelerated_files = 20000
opcache.max_wasted_percentage = 5
opcache.use_cwd = 1
opcache.validate_timestamps = 1
opcache.revalidate_freq = 2
opcache.save_comments = 1
```

OPcache mengurangi kebutuhan PHP melakukan parsing dan kompilasi ulang script pada setiap request.

---

## 11. Session Security

Untuk production HTTPS:

```text
session.use_strict_mode = 1
session.use_cookies = 1
session.use_only_cookies = 1
session.cookie_httponly = 1
session.cookie_secure = 1
session.cookie_samesite = Lax
```

`session.cookie_secure = 1` mengharuskan browser mengirim cookie session melalui HTTPS.

---

## 12. Dockerfile

Contoh PHP 8.3:

```text
FROM php:8.3-fpm-bookworm
```


```text
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        git \
        unzip \
        zip \
        libzip-dev \
        libicu-dev \
        libonig-dev \
        libxml2-dev \
        libpng-dev \
        libjpeg62-turbo-dev \
        libfreetype6-dev \
        libwebp-dev \
        libpq-dev \
    && rm -rf /var/lib/apt/lists/*
```


```text
RUN docker-php-ext-configure gd \
        --with-freetype \
        --with-jpeg \
        --with-webp
```


```text
RUN docker-php-ext-install -j"$(nproc)" \
        bcmath \
        curl \
        exif \
        gd \
        intl \
        mbstring \
        mysqli \
        opcache \
        pcntl \
        pdo \
        pdo_mysql \
        pdo_pgsql \
        xml \
        zip
```


```text
RUN pecl install redis \
    && docker-php-ext-enable redis
```


```text
COPY php.ini /usr/local/etc/php/conf.d/99-production.ini
```


```text
CMD ["php-fpm", "-F"]
```

Jangan memaksakan `USER www-data` pada Dockerfile. PHP-FPM master process tetap berjalan dengan model normal dan worker pool menggunakan `www-data`.

---

## 13. PHP-FPM Pool

Contoh:

```text
[myapp]
```


```text
user = www-data
group = www-data
```


```text
listen = /run/php/myapp.sock
```


```text
listen.owner = www-data
listen.group = www-data
listen.mode = 0660
```


```text
pm = dynamic
```


```text
pm.max_children = 20
pm.start_servers = 3
pm.min_spare_servers = 2
pm.max_spare_servers = 5
pm.max_requests = 500
```


```text
request_terminate_timeout = 120s
request_slowlog_timeout = 10s
slowlog = /proc/self/fd/2
```


```text
catch_workers_output = yes
clear_env = no
expose_php = Off
```

`pm.max_children` harus disesuaikan dengan RAM, CPU, memory usage aplikasi, dan concurrency.

---

## 14. PHP-FPM Unix Socket

Nginx berkomunikasi dengan PHP-FPM menggunakan:

```text
/run/php/myapp.sock
```

Alurnya:

```text
Browser
   |
   v
 Nginx
   |
   | Unix Socket
   v
myapp-php
   |
   v
PHP-FPM
```

Tidak diperlukan publish port:

```text
ports:
  - "9000:9000"
```

---

## 15. Docker Compose

Contoh:

```text
services:
```


```text
  php:
    image: local/php:8.3
```


```text
    container_name: myapp-php
```


```text
    restart: unless-stopped
```


```text
    read_only: true
```


```text
    security_opt:
      - no-new-privileges:true
```


```text
    working_dir: /var/www/html
```


```text
    tmpfs:
      - /tmp:rw,noexec,nosuid,size=128m
```


```text
    volumes:
      - /var/apps/myapp/htdocs:/var/www/html:ro
      - /var/apps/myapp/data/writable:/var/www/html/data:rw
      - /var/apps/myapp/logs:/var/log/php-app:rw
      - /run/php:/run/php:rw
      - /run/mysqld:/run/mysqld:ro
      - /opt/docker-apps/myapp/zz-custom.conf:/usr/local/etc/php-fpm.d/zz-custom.conf:ro
```


```text
    environment:
      TZ: Asia/Jakarta
```


```text
    cpus: "2.0"
    mem_limit: 1g
    pids_limit: 100
```


```text
    ulimits:
      nofile:
        soft: 65535
        hard: 65535
```


```text
    stop_grace_period: 30s
```


```text
    networks:
      - myapp-network
```


```text
networks:
  myapp-network:
    driver: bridge
```

---

## 16. Docker Hardening

Container menggunakan:

```text
read_only: true
```


```text
security_opt:
  - no-new-privileges:true
```


```text
tmpfs:
  - /tmp:rw,noexec,nosuid,size=128m
```

Resource:

```text
cpus: "2.0"
mem_limit: 1g
pids_limit: 100
```

Hardening bertujuan mengurangi dampak apabila aplikasi mengalami compromise.

---

## 17. Mengapa Tidak Menggunakan cap_drop: ALL?

Template `docker-php` tidak menggunakan:

```text
cap_drop:
  - ALL
```

Penghilangan seluruh Linux capabilities dapat menimbulkan masalah compatibility atau runtime pada implementasi tertentu.

Sebagai gantinya digunakan:

```text
read_only
no-new-privileges
tmpfs noexec/nosuid
CPU limit
Memory limit
PID limit
Container isolation
Writable directory isolation
```

Hardening harus mempertimbangkan security dan compatibility.

---

## 18. Laravel

Laravel menggunakan document root:

```text
/var/apps/myapp/htdocs/public
```

Writable:

```text
/var/apps/myapp/data/writable/
├── storage/
└── bootstrap-cache/
```

Mount:

```text
- /var/apps/myapp/htdocs:/var/www/html:ro
- /var/apps/myapp/data/writable/storage:/var/www/html/storage:rw
- /var/apps/myapp/data/writable/bootstrap-cache:/var/www/html/bootstrap/cache:rw
```

Source code tetap read-only.

---

## 19. CodeIgniter 4

Document root:

```text
/var/apps/myapp/htdocs/public
```

Writable:

```text
/var/apps/myapp/data/writable/
├── cache/
├── logs/
├── session/
└── uploads/
```

Mount:

```text
- /var/apps/myapp/htdocs:/var/www/html:ro
- /var/apps/myapp/data/writable:/var/www/html/writable:rw
```

---

## 20. Generic PHP

Document root:

```text
/var/apps/myapp/htdocs
```

Writable:

```text
/var/apps/myapp/data/writable/
├── cache/
├── logs/
├── session/
└── uploads/
```

Mount:

```text
- /var/apps/myapp/htdocs:/var/www/html:ro
- /var/apps/myapp/data/writable:/var/www/html/data:rw
```

---

## 21. Nginx

Konfigurasi Nginx:

```text
/etc/nginx/sites-available/myapp.conf
```

Symlink:

```text
/etc/nginx/sites-enabled/myapp.conf
```

Validasi:

```text
nginx -t
```

Reload:

```text
systemctl reload nginx
```

---

## 22. Nginx Document Root

Laravel dan CodeIgniter 4:

```text
root /var/apps/myapp/htdocs/public;
```

Generic PHP:

```text
root /var/apps/myapp/htdocs;
```

Framework modern sebaiknya menggunakan `public` sebagai document root.

---

## 23. Nginx Security Header

Baseline:

```text
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

Header tambahan dapat diterapkan setelah compatibility testing.

---

## 24. PHP Location pada Nginx

Contoh:

```text
location ~ \.php$ {
    try_files $uri =404;
```


```text
    include fastcgi_params;
```


```text
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_param DOCUMENT_ROOT $document_root;
    fastcgi_param HTTP_PROXY "";
```


```text
    fastcgi_pass unix:/run/php/myapp.sock;
```


```text
    fastcgi_connect_timeout 10s;
    fastcgi_send_timeout 120s;
    fastcgi_read_timeout 120s;
}
```

`try_files $uri =404` memastikan file PHP tersedia sebelum diteruskan ke PHP-FPM.

---

## 25. Proteksi File Sensitif

Gunakan:

```text
location ~* \.(env|ini|log|sql|bak|backup|old|orig|save|swp)$ {
    deny all;
}
```

File tersembunyi:

```text
location ~ /\.(?!well-known).* {
    deny all;
}
```

Tujuannya mencegah file seperti `.env`, database dump, log, dan file konfigurasi diakses melalui HTTP.

---

## 26. Mencegah Eksekusi PHP pada Writable Directory

Laravel:

```text
location ~ ^/(storage|bootstrap/cache)/.*\.php$ {
    deny all;
}
```

CodeIgniter 4:

```text
location ~ ^/writable/.*\.php$ {
    deny all;
}
```

Generic PHP:

```text
location ~ ^/data/.*\.php$ {
    deny all;
}
```

Writable directory harus diperlakukan sebagai data, bukan executable code.

---

## 27. Upload Security

Upload directory harus dianggap sebagai data.

Gunakan beberapa lapisan:

```text
Extension Validation
        +
MIME Validation
        +
File Size Limit
        +
Permission Restriction
        +
Nginx Restriction
        +
Application Validation
```

Untuk kebutuhan security lebih tinggi dapat ditambahkan:

```text
Antivirus
Malware Scanning
YARA
Content Inspection
```

Jika memungkinkan, file upload disimpan di luar web root.

---

## 28. Generator create-php-app.sh

Sintaks:

```text
./create-php-app.sh <application-name> <php-version> <framework>
```

Laravel:

```text
./create-php-app.sh myapp 8.3 laravel
```

CodeIgniter 4:

```text
./create-php-app.sh myapp2 8.4 ci
```

Generic:

```text
./create-php-app.sh myapp3 8.5 generic
```

Legacy:

```text
./create-php-app.sh legacy-app 7.4 generic
```

---

## 29. Apa yang Dilakukan Generator?

Generator:

1. Memvalidasi argument.
2. Memvalidasi nama aplikasi.
3. Memvalidasi versi PHP.
4. Memvalidasi framework.
5. Memastikan Docker tersedia.
6. Memastikan Docker Compose tersedia.
7. Memastikan Nginx tersedia.
8. Memastikan image PHP tersedia.
9. Membuat directory aplikasi.
10. Membuat directory writable.
11. Membuat konfigurasi PHP-FPM.
12. Membuat Docker Compose.
13. Membuat konfigurasi Nginx.
14. Membuat symlink Nginx.
15. Menjalankan `nginx -t`.
16. Memvalidasi Docker Compose.
17. Menampilkan langkah deployment berikutnya.

Generator tidak otomatis menjalankan `docker compose up -d`, sehingga administrator dapat melakukan review terlebih dahulu.

---

## 30. Validasi Nama Aplikasi

Contoh valid:

```text
myapp
my-app
my_app
myapp2
```

Contoh tidak valid:

```text
../myapp
../../myapp
/var/www/myapp
```

Validasi diperlukan untuk mencegah path traversal pada parameter generator.

---

## 31. Permission

Source code:

```text
root:root
```

Writable:

```text
www-data:www-data
```

Konsep:

```text
htdocs
   |
   +-- root-owned
   +-- read-only
```


```text
data/writable
   |
   +-- www-data
   +-- read-write
```

PHP-FPM hanya memiliki write access pada directory yang memang diperlukan.

---

## 32. Build PHP Image

PHP 7.4:

```text
cd /opt/docker-php/images/7.4
docker build -t local/php:7.4 .
```

PHP 8.3:

```text
cd /opt/docker-php/images/8.3
docker build -t local/php:8.3 .
```

PHP 8.4:

```text
cd /opt/docker-php/images/8.4
docker build -t local/php:8.4 .
```

PHP 8.5:

```text
cd /opt/docker-php/images/8.5
docker build -t local/php:8.5 .
```

Periksa:

```text
docker images local/php
```

---

## 33. Build Semua Image

Jalankan:

```text
cd /opt/docker-php
./scripts/build-all.sh
```

Script akan melakukan build seluruh versi PHP yang tersedia.

---

## 34. Test PHP Image

Versi PHP:

```text
docker run --rm local/php:8.3 php -v
```

Extension:

```text
docker run --rm local/php:8.3 php -m
```

Konfigurasi:

```text
docker run --rm local/php:8.3 php --ini
```

PHP-FPM:

```text
docker run --rm local/php:8.3 php-fpm -t
```

Semua versi:

```text
cd /opt/docker-php
./scripts/test-images.sh
```

---

## 35. Membuat Aplikasi Baru

Setelah image tersedia:

```text
cd /opt/docker-php
```

Contoh:

```text
./create-php-app.sh myapp 8.3 laravel
```

Generator membuat:

```text
/opt/docker-apps/myapp/
├── docker-compose.yml
└── zz-custom.conf
```

dan:

```text
/var/apps/myapp/
├── htdocs/
├── data/
│   └── writable/
├── logs/
└── backup/
```

serta:

```text
/etc/nginx/sites-available/myapp.conf
/etc/nginx/sites-enabled/myapp.conf
```

---

## 36. Deploy Source Code

Source code ditempatkan:

```text
/var/apps/myapp/htdocs/
```

Contoh Laravel:

```text
/var/apps/myapp/htdocs/
├── app/
├── bootstrap/
├── config/
├── database/
├── public/
├── resources/
├── routes/
├── vendor/
├── artisan
└── composer.json
```

Source code sebaiknya berasal dari Git repository atau release artifact yang terkontrol.

---

## 37. Validasi Docker Compose

Masuk:

```text
cd /opt/docker-apps/myapp
```

Validasi:

```text
docker compose config
```

Jika valid:

```text
docker compose up -d
```

Periksa:

```text
docker compose ps
```

---

## 38. Memeriksa Container

Periksa:

```text
docker compose ps
```

Log:

```text
docker compose logs -f php
```

Resource:

```text
docker stats
```

Detail:

```text
docker inspect myapp-php
```

---

## 39. Memeriksa PHP-FPM Socket

Periksa:

```text
ls -lah /run/php/
```

Target:

```text
myapp.sock
```

Periksa:

```text
ls -lah /run/php/myapp.sock
```

Jika socket tidak tersedia:

```text
cd /opt/docker-apps/myapp
docker compose logs php
```

Periksa pool:

```text
cat /opt/docker-apps/myapp/zz-custom.conf
```

---

## 40. Validasi Nginx

Test:

```text
nginx -t
```

Jika berhasil:

```text
systemctl reload nginx
```

Konfigurasi lengkap:

```text
nginx -T
```

---

## 41. Troubleshooting HTTP 502

Jika Nginx memberikan `502 Bad Gateway`, periksa:

```text
Nginx
   |
   v
Unix Socket
   |
   v
PHP-FPM Container
   |
   v
PHP-FPM Pool
```

Test Nginx:

```text
nginx -t
```

Socket:

```text
ls -lah /run/php/myapp.sock
```

Container:

```text
cd /opt/docker-apps/myapp
docker compose ps
```

Log:

```text
docker compose logs php
```

Pool:

```text
cat /opt/docker-apps/myapp/zz-custom.conf
```

---

## 42. Troubleshooting Permission

Periksa:

```text
ls -ld /var/apps/myapp/data/writable
```

dan:

```text
ls -la /var/apps/myapp/data/writable
```

Pastikan writable directory:

```text
www-data:www-data
```

Source code tetap:

```text
root:root
```

---

## 43. Backup

Backup production minimal mempertimbangkan:

```text
Application Source
Application Writable Data
Database
Nginx Configuration
Docker Compose
PHP-FPM Configuration
Environment/Secrets
```

Secret seperti password database, API key, private key, application secret, dan token jangan dimasukkan ke Git repository.

---

## 44. Isolasi Antar Aplikasi

Setiap aplikasi mempunyai container sendiri:

```text
myapp-php
myapp2-php
myapp3-php
```

Dan network sendiri:

```text
myapp-network
myapp2-network
myapp3-network
```

Struktur:

```text
HOST
  |
  +-- myapp-php
  |      |
  |      +-- myapp-network
  |
  +-- myapp2-php
  |      |
  |      +-- myapp2-network
  |
  +-- myapp3-php
         |
         +-- myapp3-network
```

Hal ini membuat boundary antar aplikasi menjadi lebih jelas.

---

## 45. Security by Design

Deployment menerapkan beberapa prinsip keamanan.

### Least Privilege

PHP-FPM hanya diberikan akses write pada lokasi yang membutuhkan write.

### Defense in Depth

Keamanan tidak bergantung pada satu mekanisme:

```text
Nginx
+
Read-only Source
+
Read-only Container Filesystem
+
no-new-privileges
+
tmpfs
+
Resource Limit
+
PID Limit
+
Container Isolation
+
Writable Directory Isolation
```

### Separation of Concerns

Komponen dipisahkan:

```text
Source Code
Runtime Data
Logs
Backup
Docker Configuration
Nginx Configuration
```

### Immutable Source

Source code di-mount read-only sehingga PHP-FPM tidak memiliki akses write ke seluruh source code aplikasi.

---

## 46. Workflow Deployment

Workflow:

```text
Build PHP Image
       |
       v
Test PHP Image
       |
       v
create-php-app.sh
       |
       v
Deploy Source Code
       |
       v
Set Permission
       |
       v
docker compose config
       |
       v
docker compose up -d
       |
       v
Check PHP-FPM
       |
       v
Check Unix Socket
       |
       v
nginx -t
       |
       v
Reload Nginx
       |
       v
Application Test
```

---

## 47. Upgrade PHP

Misalnya:

```text
local/php:8.3
```

akan diubah menjadi:

```text
local/php:8.4
```

Sebelum upgrade production:

```text
Backup
   |
   v
Build Image
   |
   v
Test PHP
   |
   v
Test Composer Dependencies
   |
   v
Test PHP Extensions
   |
   v
Test Application
   |
   v
Test Database
   |
   v
Deploy
```

Jangan melakukan upgrade major PHP langsung pada production tanpa compatibility testing.

---

## 48. Monitoring

Container production perlu dimonitor.

Pemeriksaan dasar:

```text
docker stats
```

dan:

```text
docker compose ps
```

Metric yang perlu diperhatikan:

```text
CPU
Memory
Container Restart
PHP-FPM Process
Request Latency
HTTP 5xx
Disk Usage
Unix Socket
Database Connection
Application Error
```

Untuk environment yang lebih besar, monitoring dapat diintegrasikan dengan sistem monitoring seperti Zabbix atau platform observability lainnya.

---

## 49. Production Checklist

Sebelum aplikasi dinyatakan siap:

```text
[ ] PHP version sesuai compatibility aplikasi
[ ] Docker image berhasil dibuild
[ ] PHP extension lengkap
[ ] php.ini sudah sesuai
[ ] PHP-FPM configuration valid
[ ] Docker Compose valid
[ ] Container berjalan
[ ] Unix Socket tersedia
[ ] Permission source code benar
[ ] Permission writable benar
[ ] Nginx configuration valid
[ ] HTTPS aktif
[ ] File sensitif tidak dapat diakses
[ ] PHP execution di writable/upload directory diblok
[ ] Database connection berhasil
[ ] Application test berhasil
[ ] Backup tersedia
[ ] Monitoring tersedia
[ ] Log tersedia
[ ] Secret tidak masuk Git
```

---

## 50. Kesimpulan

Repository `docker-php` dibuat sebagai standar deployment PHP-FPM untuk server production.

Pemisahan directory:

```text
/opt/docker-php
```

digunakan untuk template dan tooling.

```text
/opt/docker-apps
```

digunakan untuk konfigurasi Docker setiap aplikasi.

```text
/var/apps
```

digunakan untuk source code dan runtime data aplikasi.

Dengan pendekatan ini, satu server dapat menjalankan beberapa aplikasi dengan versi PHP berbeda:

```text
myapp
  └── PHP 8.3
```


```text
myapp2
  └── PHP 8.4
```


```text
myapp3
  └── PHP 8.5
```


```text
legacy-app
  └── PHP 7.4
```

Source code dipisahkan dari runtime writable, PHP-FPM diisolasi dalam container, Nginx tetap menjadi web server pada host, dan setiap aplikasi mempunyai konfigurasi deployment sendiri.

Dengan adanya `create-php-app.sh`, standar tersebut dapat diterapkan secara konsisten setiap kali aplikasi baru akan ditempatkan pada server.

---

## Referensi

- Docker Official Images — PHP
- PHP Documentation — PHP-FPM
- PHP Documentation — OPcache
- Nginx Documentation
- Docker Compose Documentation
- Laravel Documentation
- CodeIgniter 4 Documentation
