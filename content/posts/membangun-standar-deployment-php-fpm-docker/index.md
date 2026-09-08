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

Tanpa standar yang jelas, setiap aplikasi dapat memiliki konfigurasi berbeda, mulai dari versi PHP, konfigurasi PHP-FPM, permission directory, konfigurasi Nginx, hingga mekanisme penyimpanan file writable.

Pada artikel ini kita akan membangun sebuah repository bernama **`docker-php`** yang digunakan sebagai standar deployment PHP-FPM berbasis Docker.

Konsep yang digunakan adalah:

- Nginx berjalan langsung pada host.
- MariaDB berjalan langsung pada host.
- Setiap aplikasi PHP memiliki container PHP-FPM sendiri.
- Source code aplikasi di-mount secara read-only.
- Directory yang membutuhkan write dipisahkan.
- Nginx berkomunikasi dengan PHP-FPM melalui Unix Socket.
- PHP-FPM berkomunikasi dengan MariaDB melalui TCP.
- Container menggunakan beberapa mekanisme hardening.
- Deployment aplikasi baru dapat dibuat menggunakan script `create-php-app.sh`.

---

## 1. Arsitektur

Arsitektur yang digunakan:

```text
                         INTERNET
                            │
                            ▼
                    ┌───────────────┐
                    │     Nginx     │
                    │     HOST      │
                    └───────┬───────┘
                            │
                       Unix Socket
                            │
                            ▼
                 ┌─────────────────────┐
                 │   PHP-FPM Docker    │
                 │                     │
                 │ local/php:7.4       │
                 │ local/php:8.3       │
                 │ local/php:8.4       │
                 │ local/php:8.5       │
                 └──────────┬──────────┘
                            │
                         TCP 3306
                            │
                            ▼
                    ┌───────────────┐
                    │    MariaDB    │
                    │     HOST      │
                    └───────────────┘
```

Dengan arsitektur ini, PHP tidak perlu di-install langsung pada host.

Host cukup menyediakan:

```text
Nginx
MariaDB
Docker
```

Sedangkan runtime PHP disediakan oleh container.

---

## 2. Mengapa Menggunakan PHP-FPM Docker?

Misalnya server menjalankan beberapa aplikasi:

```text
myapp
myapp2
myapp3
```

Aplikasi tersebut belum tentu menggunakan versi PHP yang sama.

Dengan pendekatan Docker:

```text
myapp
   └── PHP 8.3

myapp2
   └── PHP 8.4

myapp3
   └── PHP 8.5
```

Setiap aplikasi memiliki runtime PHP sendiri.

Keuntungannya:

1. Versi PHP dapat berbeda antar aplikasi.
2. Dependency PHP lebih terisolasi.
3. Upgrade PHP dapat dilakukan per aplikasi.
4. Konfigurasi dapat disimpan sebagai kode.
5. Deployment menjadi lebih konsisten.
6. Resource container dapat dibatasi.
7. Risiko konflik antar aplikasi dapat dikurangi.

---

## 3. Struktur Repository

Repository utama berada di:

```text
/opt/docker-php
```

Strukturnya:

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
│   │
│   ├── 8.3/
│   │   ├── Dockerfile
│   │   └── php.ini
│   │
│   ├── 8.4/
│   │   ├── Dockerfile
│   │   └── php.ini
│   │
│   └── 8.5/
│       ├── Dockerfile
│       └── php.ini
│
├── templates/
│   ├── docker/
│   ├── nginx/
│   └── php-fpm/
│
└── scripts/
    ├── build-all.sh
    └── test-images.sh
```

Repository tersebut digunakan sebagai **template dan tooling**, bukan sebagai tempat source code aplikasi production.

---

## 4. Pemisahan Directory

Kita menggunakan tiga lokasi utama.

### Repository Docker PHP

```text
/opt/docker-php
```

Digunakan untuk:

- Dockerfile
- `php.ini`
- template
- generator
- script build
- script testing
- dokumentasi

### Konfigurasi Docker Aplikasi

```text
/opt/docker-apps
```

Digunakan untuk konfigurasi masing-masing aplikasi.

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

Dengan pemisahan ini, source code aplikasi tidak bercampur dengan konfigurasi Docker.

---

## 5. Struktur Directory Aplikasi

Setiap aplikasi menggunakan struktur standar:

```text
/var/apps/<application-name>/
├── htdocs/
├── data/
│   └── writable/
├── logs/
└── backup/
```

### `htdocs`

Berisi source code aplikasi.

Contoh:

```text
/var/apps/myapp/htdocs/
├── app/
├── public/
├── resources/
├── routes/
├── vendor/
├── composer.json
└── ...
```

Directory tersebut di-mount ke container secara read-only:

```yaml
- /var/apps/myapp/htdocs:/var/www/html:ro
```

### `data/writable`

Berisi data runtime yang memang membutuhkan akses write.

### `logs`

Digunakan untuk log aplikasi.

### `backup`

Digunakan untuk penyimpanan backup aplikasi.

PHP-FPM tidak perlu diberikan akses write ke directory backup.

Prinsip yang digunakan:

```text
Source Code
    │
    └── Read Only

Writable Data
    │
    └── Read Write

Backup
    │
    └── Tidak diakses PHP-FPM
```

---

## 6. PHP Version

Runtime yang disediakan:

```text
PHP 7.4
PHP 8.3
PHP 8.4
PHP 8.5
```

Image lokal menggunakan:

```text
local/php:7.4
local/php:8.3
local/php:8.4
local/php:8.5
```

PHP 7.4 dipertahankan untuk kebutuhan aplikasi legacy.

Untuk aplikasi baru, gunakan versi PHP 8.x yang kompatibel dengan framework dan dependency aplikasi.

---

## 7. PHP Extension

Baseline extension yang digunakan:

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

| Extension | Fungsi |
|---|---|
| `bcmath` | Perhitungan presisi |
| `curl` | HTTP/API request |
| `exif` | Metadata gambar |
| `gd` | Manipulasi gambar |
| `intl` | Internationalization |
| `mbstring` | String multibyte |
| `mysqli` | MySQL/MariaDB |
| `opcache` | Optimasi PHP |
| `pcntl` | Process control |
| `pdo` | Database abstraction |
| `pdo_mysql` | MySQL/MariaDB |
| `pdo_pgsql` | PostgreSQL |
| `redis` | Redis |
| `xml` | XML/DOM |
| `zip` | Archive dan dependency |

Extension tambahan dapat ditambahkan apabila memang diperlukan oleh aplikasi.

---

## 8. Konfigurasi `php.ini`

Baseline konfigurasi production:

```ini
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

cgi.fix_pathinfo = 0

allow_url_include = Off

date.timezone = Asia/Jakarta
```

Pada production, error tidak ditampilkan langsung kepada pengguna:

```ini
display_errors = Off
```

Error diarahkan ke log:

```ini
log_errors = On
error_log = /proc/self/fd/2
```

Hal ini membantu mencegah informasi internal aplikasi muncul pada response HTTP.

---

## 9. OPcache

OPcache diaktifkan:

```ini
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

OPcache mengurangi kebutuhan PHP untuk melakukan parsing dan kompilasi ulang script pada setiap request.

Nilai tersebut merupakan baseline dan dapat dioptimalkan setelah karakteristik workload aplikasi diketahui.

---

## 10. Session Security

Untuk production HTTPS:

```ini
session.use_strict_mode = 1

session.use_cookies = 1

session.use_only_cookies = 1

session.cookie_httponly = 1

session.cookie_secure = 1

session.cookie_samesite = Lax
```

`HttpOnly` membantu mencegah cookie session diakses langsung oleh JavaScript.

`Secure` memastikan cookie session hanya dikirim melalui HTTPS.

Karena itu production sebaiknya menggunakan:

```text
HTTPS
  │
  ▼
Nginx
  │
  ▼
PHP-FPM
```

---

## 11. Dockerfile

Setiap versi PHP memiliki Dockerfile sendiri.

Contoh PHP 8.3:

```dockerfile
FROM php:8.3-fpm-bookworm

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

Extension PHP kemudian dipasang menggunakan:

```dockerfile
RUN docker-php-ext-install ...
```

Redis dipasang menggunakan PECL:

```dockerfile
RUN pecl install redis \
    && docker-php-ext-enable redis
```

PHP-FPM dijalankan dalam foreground:

```dockerfile
CMD ["php-fpm", "-F"]
```

---

## 12. Mengapa Tidak Menggunakan `USER www-data`?

PHP-FPM mempunyai model master dan worker process.

Master process PHP-FPM melakukan initialization, termasuk pengelolaan pool dan Unix Socket.

Worker kemudian dijalankan menggunakan:

```text
www-data
```

Karena itu kita tidak memaksakan:

```dockerfile
USER www-data
```

pada level Dockerfile.

Sebagai gantinya, PHP-FPM pool menentukan:

```ini
user = www-data
group = www-data
```

Dengan demikian worker aplikasi tetap berjalan sebagai user dengan privilege terbatas.

---

## 13. PHP-FPM Pool

Setiap aplikasi mempunyai konfigurasi pool sendiri.

Contoh:

```ini
[myapp]

user = www-data
group = www-data

listen = /run/php/myapp.sock

listen.owner = www-data
listen.group = www-data
listen.mode = 0660

pm = dynamic

pm.max_children = 20
pm.start_servers = 3
pm.min_spare_servers = 2
pm.max_spare_servers = 5
pm.max_requests = 500

request_terminate_timeout = 120s
request_slowlog_timeout = 10s

catch_workers_output = yes

clear_env = no
```

`pm.max_children` merupakan baseline.

Nilainya harus disesuaikan berdasarkan:

- RAM
- CPU
- jumlah concurrent request
- memory usage aplikasi
- karakteristik workload
- kebutuhan queue atau worker

---

## 14. PHP-FPM Unix Socket

Komunikasi Nginx dengan PHP-FPM menggunakan Unix Socket.

Contoh:

```text
/run/php/myapp.sock
```

Alurnya:

```text
Browser
   │
   ▼
 Nginx
   │
   │ Unix Socket
   ▼
myapp-php
   │
   ▼
PHP-FPM
```

PHP-FPM tidak perlu dipublish menggunakan:

```yaml
ports:
  - "9000:9000"
```

Dengan demikian PHP-FPM tidak perlu diekspos melalui port TCP host.

---

## 15. Docker Compose Hardening

Container aplikasi menggunakan:

```yaml
read_only: true

security_opt:
  - no-new-privileges:true

tmpfs:
  - /tmp:rw,noexec,nosuid,size=128m
```

Resource limit:

```yaml
cpus: "2.0"

mem_limit: 1g

pids_limit: 100
```

---

## 16. Read-only Filesystem

Filesystem root container dibuat read-only:

```yaml
read_only: true
```

Source code juga di-mount read-only:

```yaml
- /var/apps/myapp/htdocs:/var/www/html:ro
```

Dengan demikian:

```text
Container Filesystem
        │
        └── Read Only

Application Source
        │
        └── Read Only

Runtime Writable
        │
        └── Explicit Volume
```

Directory yang memang membutuhkan write diberikan secara terpisah.

---

## 17. `no-new-privileges`

Container menggunakan:

```yaml
security_opt:
  - no-new-privileges:true
```

Tujuannya membatasi proses di dalam container agar tidak memperoleh privilege tambahan melalui mekanisme privilege escalation tertentu.

---

## 18. Tmpfs

Directory `/tmp` menggunakan:

```yaml
tmpfs:
  - /tmp:rw,noexec,nosuid,size=128m
```

Konfigurasi tersebut memberikan:

```text
rw
noexec
nosuid
size=128m
```

Dengan demikian aplikasi tetap memiliki temporary directory yang writable tanpa harus membuat root filesystem container writable.

---

## 19. Resource Limit

Container diberikan batas:

```yaml
cpus: "2.0"

mem_limit: 1g

pids_limit: 100
```

Tujuannya mencegah satu aplikasi menghabiskan seluruh resource server.

Nilai tersebut merupakan baseline dan harus disesuaikan berdasarkan hasil monitoring.

---

## 20. Mengapa Tidak Menggunakan `cap_drop: ALL`?

Template `docker-php` tidak menggunakan:

```yaml
cap_drop:
  - ALL
```

Penghilangan seluruh Linux capabilities dapat menimbulkan masalah compatibility atau runtime pada implementasi tertentu.

Sebagai gantinya digunakan hardening berlapis:

```text
read_only
+
no-new-privileges
+
tmpfs noexec/nosuid
+
CPU limit
+
Memory limit
+
PID limit
+
Container isolation
+
Writable directory isolation
```

Hardening tidak hanya harus ketat, tetapi juga tetap mempertimbangkan compatibility dan stabilitas aplikasi.

---

## 21. Laravel

Laravel menggunakan:

```text
/var/apps/myapp/htdocs/public
```

sebagai document root.

Directory writable dipisahkan:

```text
/var/apps/myapp/data/writable/
├── storage/
└── bootstrap-cache/
```

Docker volume:

```yaml
volumes:
  - /var/apps/myapp/htdocs:/var/www/html:ro

  - /var/apps/myapp/data/writable/storage:/var/www/html/storage:rw

  - /var/apps/myapp/data/writable/bootstrap-cache:/var/www/html/bootstrap/cache:rw
```

Dengan demikian source code Laravel tetap read-only.

---

## 22. CodeIgniter 4

CodeIgniter 4 menggunakan:

```text
/var/apps/myapp/htdocs/public
```

sebagai document root.

Writable directory:

```text
/var/apps/myapp/data/writable/
├── cache/
├── logs/
├── session/
└── uploads/
```

Mount:

```yaml
- /var/apps/myapp/data/writable:/var/www/html/writable:rw
```

---

## 23. Generic PHP

Untuk aplikasi PHP generic:

```text
/var/apps/myapp/htdocs
```

digunakan sebagai document root.

Writable:

```text
/var/apps/myapp/data/writable/
├── cache/
├── logs/
├── session/
└── uploads/
```

Mount:

```yaml
- /var/apps/myapp/data/writable:/var/www/html/data:rw
```

---

## 24. Nginx

Nginx tetap berjalan pada host.

Konfigurasi berada pada:

```text
/etc/nginx/sites-available/
```

dan:

```text
/etc/nginx/sites-enabled/
```

Contoh:

```text
/etc/nginx/sites-available/myapp.conf
```

kemudian dibuat symlink:

```text
/etc/nginx/sites-enabled/myapp.conf
```

Sebelum reload:

```bash
nginx -t
```

Jika konfigurasi valid:

```bash
systemctl reload nginx
```

---

## 25. Nginx Document Root

Untuk Laravel dan CodeIgniter 4:

```nginx
root /var/apps/myapp/htdocs/public;
```

Untuk generic PHP:

```nginx
root /var/apps/myapp/htdocs;
```

Framework modern sebaiknya menggunakan directory `public` sebagai document root agar source internal aplikasi tidak langsung diekspos oleh web server.

---

## 26. PHP Location pada Nginx

Contoh:

```nginx
location ~ \.php$ {
    try_files $uri =404;

    include fastcgi_params;

    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_param DOCUMENT_ROOT $document_root;
    fastcgi_param HTTP_PROXY "";

    fastcgi_pass unix:/run/php/myapp.sock;

    fastcgi_connect_timeout 10s;
    fastcgi_send_timeout 120s;
    fastcgi_read_timeout 120s;
}
```

`try_files $uri =404` membantu memastikan file PHP yang diminta memang tersedia sebelum request diteruskan ke PHP-FPM.

---

## 27. Proteksi File Sensitif

Nginx dapat digunakan untuk menolak akses terhadap file sensitif:

```nginx
location ~* \.(env|ini|log|sql|bak|backup|old|orig|save|swp)$ {
    deny all;
}
```

File tersembunyi juga diblok:

```nginx
location ~ /\.(?!well-known).* {
    deny all;
}
```

Tujuannya mencegah file seperti:

```text
.env
database.sql
application.log
config.ini
backup.sql
```

diakses melalui HTTP.

---

## 28. Mencegah Eksekusi PHP pada Writable Directory

Directory writable harus dianggap sebagai **data**, bukan executable code.

Laravel:

```nginx
location ~ ^/(storage|bootstrap/cache)/.*\.php$ {
    deny all;
}
```

CodeIgniter 4:

```nginx
location ~ ^/writable/.*\.php$ {
    deny all;
}
```

Generic PHP:

```nginx
location ~ ^/data/.*\.php$ {
    deny all;
}
```

Contoh ancaman:

```text
uploads/
└── shell.php
```

Attacker kemudian mencoba mengakses:

```text
https://example.com/uploads/shell.php
```

Directory upload harus dikonfigurasi agar file PHP tidak dapat dieksekusi.

---

## 29. Upload Security

Upload directory harus diperlakukan sebagai **data**, bukan code.

Jangan hanya melakukan validasi berdasarkan extension.

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

Untuk aplikasi dengan kebutuhan keamanan lebih tinggi dapat ditambahkan:

```text
Antivirus Scanning
Malware Scanning
YARA
Content Inspection
```

Jika memungkinkan, file upload dapat disimpan di luar web root.

---

## 30. Generator `create-php-app.sh`

Repository menyediakan generator:

```text
create-php-app.sh
```

Sintaks:

```bash
./create-php-app.sh <application-name> <php-version> <framework>
```

Contoh Laravel:

```bash
./create-php-app.sh myapp 8.3 laravel
```

CodeIgniter 4:

```bash
./create-php-app.sh myapp2 8.4 ci
```

Generic PHP:

```bash
./create-php-app.sh myapp3 8.5 generic
```

Legacy:

```bash
./create-php-app.sh legacy-app 7.4 generic
```

---

## 31. Apa yang Dilakukan Generator?

Script akan melakukan beberapa proses:

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

Generator tidak langsung menjalankan:

```bash
docker compose up -d
```

Administrator tetap dapat melakukan review konfigurasi terlebih dahulu.

---

## 32. Validasi Nama Aplikasi

Nama aplikasi harus sederhana dan aman.

Contoh valid:

```text
myapp
my-app
my_app
myapp2
```

Contoh yang harus ditolak:

```text
../myapp

../../myapp

/var/www/myapp
```

Validasi ini penting agar parameter generator tidak dapat digunakan untuk melakukan path traversal.

---

## 33. Permission

Source code:

```text
root:root
```

Writable:

```text
www-data:www-data
```

Konsepnya:

```text
htdocs
   │
   ├── root-owned
   └── read-only

data/writable
   │
   ├── www-data
   └── read-write
```

Dengan demikian PHP-FPM hanya memiliki write access pada directory yang memang diperlukan.

---

## 34. Build PHP Image

Build PHP 7.4:

```bash
cd /opt/docker-php/images/7.4

docker build -t local/php:7.4 .
```

Build PHP 8.3:

```bash
cd /opt/docker-php/images/8.3

docker build -t local/php:8.3 .
```

Build PHP 8.4:

```bash
cd /opt/docker-php/images/8.4

docker build -t local/php:8.4 .
```

Build PHP 8.5:

```bash
cd /opt/docker-php/images/8.5

docker build -t local/php:8.5 .
```

Periksa image:

```bash
docker images local/php
```

---

## 35. Build Semua Image

Repository menyediakan:

```text
scripts/build-all.sh
```

Jalankan:

```bash
cd /opt/docker-php

./scripts/build-all.sh
```

Script akan melakukan build seluruh versi PHP yang tersedia.

---

## 36. Test PHP Image

Periksa versi PHP:

```bash
docker run --rm local/php:8.3 php -v
```

Periksa extension:

```bash
docker run --rm local/php:8.3 php -m
```

Periksa konfigurasi:

```bash
docker run --rm local/php:8.3 php --ini
```

Periksa PHP-FPM:

```bash
docker run --rm local/php:8.3 php-fpm -t
```

Untuk semua versi:

```bash
cd /opt/docker-php

./scripts/test-images.sh
```

---

## 37. Membuat Aplikasi Baru

Setelah image tersedia:

```bash
cd /opt/docker-php
```

Jalankan:

```bash
./create-php-app.sh myapp 8.3 laravel
```

Generator akan membuat:

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

## 38. Deploy Source Code

Source code aplikasi ditempatkan pada:

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

## 39. Validasi Docker Compose

Masuk ke directory konfigurasi aplikasi:

```bash
cd /opt/docker-apps/myapp
```

Validasi:

```bash
docker compose config
```

Jika valid:

```bash
docker compose up -d
```

Periksa:

```bash
docker compose ps
```

---

## 40. Memeriksa Container

Periksa container:

```bash
docker compose ps
```

Periksa log:

```bash
docker compose logs -f php
```

Periksa resource:

```bash
docker stats
```

Periksa konfigurasi container:

```bash
docker inspect myapp-php
```

---

## 41. Memeriksa PHP-FPM Socket

Periksa:

```bash
ls -lah /run/php/
```

Target:

```text
myapp.sock
```

Kemudian:

```bash
ls -lah /run/php/myapp.sock
```

Jika socket tidak tersedia:

```bash
cd /opt/docker-apps/myapp

docker compose logs php
```

Periksa konfigurasi PHP-FPM:

```bash
cat /opt/docker-apps/myapp/zz-custom.conf
```

---

## 42. Validasi Nginx

Test konfigurasi:

```bash
nginx -t
```

Jika berhasil:

```bash
systemctl reload nginx
```

Untuk melihat konfigurasi lengkap yang sedang dimuat:

```bash
nginx -T
```

---

## 43. Troubleshooting HTTP 502

Jika Nginx memberikan:

```text
502 Bad Gateway
```

periksa secara berurutan:

```text
Nginx
   │
   ▼
Unix Socket
   │
   ▼
PHP-FPM Container
   │
   ▼
PHP-FPM Pool
```

Test Nginx:

```bash
nginx -t
```

Periksa socket:

```bash
ls -lah /run/php/myapp.sock
```

Periksa container:

```bash
cd /opt/docker-apps/myapp

docker compose ps
```

Periksa log:

```bash
docker compose logs php
```

Periksa PHP-FPM pool:

```bash
cat /opt/docker-apps/myapp/zz-custom.conf
```

---

## 44. Troubleshooting Permission

Periksa:

```bash
ls -ld /var/apps/myapp/data/writable
```

dan:

```bash
ls -la /var/apps/myapp/data/writable
```

Pastikan directory runtime dimiliki oleh user/group yang sesuai dengan PHP-FPM.

Baseline:

```text
www-data:www-data
```

Source code tetap:

```text
root:root
```

---

## 45. Backup

Backup production tidak hanya mencakup source code.

Minimal pertimbangkan:

```text
Application Source
Application Writable Data
Database
Nginx Configuration
Docker Compose
PHP-FPM Configuration
Environment/Secrets
```

Database MariaDB harus dibackup menggunakan mekanisme backup database yang sesuai.

Secret seperti:

```text
Database Password
API Key
Private Key
Application Secret
Token
```

jangan dimasukkan ke repository Git.

---

## 46. Isolasi Antar Aplikasi

Setiap aplikasi memiliki container sendiri.

Contoh:

```text
myapp-php
myapp2-php
myapp3-php
```

Masing-masing juga mempunyai network sendiri:

```text
myapp-network
myapp2-network
myapp3-network
```

Struktur:

```text
                    HOST
                      │
        ┌─────────────┼─────────────┐
        │             │             │
        ▼             ▼             ▼
   myapp-php     myapp2-php     myapp3-php
        │             │             │
        ▼             ▼             ▼
 myapp-network  myapp2-network  myapp3-network
```

Hal ini membuat boundary antar aplikasi menjadi lebih jelas.

---

## 47. Security by Design

Deployment ini menerapkan beberapa prinsip keamanan.

### Least Privilege

PHP-FPM hanya diberi akses write pada lokasi yang memang membutuhkan write.

### Defense in Depth

Keamanan tidak bergantung pada satu mekanisme.

Digunakan:

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

Source code di-mount:

```text
read-only
```

sehingga PHP-FPM tidak memiliki akses write ke seluruh source code aplikasi.

---

## 48. Workflow Deployment

Workflow standar:

```text
Build PHP Image
       │
       ▼
Test PHP Image
       │
       ▼
create-php-app.sh
       │
       ▼
Deploy Source Code
       │
       ▼
Set Permission
       │
       ▼
docker compose config
       │
       ▼
docker compose up -d
       │
       ▼
Check PHP-FPM
       │
       ▼
Check Unix Socket
       │
       ▼
nginx -t
       │
       ▼
Reload Nginx
       │
       ▼
Application Test
```

Dengan workflow ini, setiap aplikasi mengikuti proses deployment yang sama.

---

## 49. Upgrade PHP

Salah satu keuntungan menggunakan container adalah upgrade PHP dapat dilakukan per aplikasi.

Misalnya:

```text
myapp
```

sebelumnya menggunakan:

```text
local/php:8.3
```

kemudian akan dipindahkan ke:

```text
local/php:8.4
```

Sebelum upgrade production:

```text
Backup
   │
   ▼
Build Image
   │
   ▼
Test PHP
   │
   ▼
Test Composer Dependencies
   │
   ▼
Test PHP Extensions
   │
   ▼
Test Application
   │
   ▼
Test Database
   │
   ▼
Deploy
```

Jangan melakukan upgrade major PHP langsung pada production tanpa compatibility testing.

---

## 50. Monitoring

Container production perlu dimonitor.

Pemeriksaan dasar:

```bash
docker stats
```

dan:

```bash
docker compose ps
```

Metric yang perlu diperhatikan antara lain:

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
```

Untuk environment yang lebih besar, monitoring dapat diintegrasikan dengan sistem seperti Zabbix atau platform observability lainnya.

---

## 51. Checklist Production

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

## 52. Kesimpulan

`docker-php` dibuat sebagai standar deployment PHP-FPM untuk server production.

Pemisahan directory:

```text
/opt/docker-php
```

digunakan untuk **template dan tooling**.

```text
/opt/docker-apps
```

digunakan untuk **konfigurasi Docker setiap aplikasi**.

```text
/var/apps
```

digunakan untuk **source code dan runtime data aplikasi**.

Dengan pendekatan ini, satu server dapat menjalankan beberapa aplikasi dengan versi PHP berbeda:

```text
myapp
  └── PHP 8.3

myapp2
  └── PHP 8.4

myapp3
  └── PHP 8.5

legacy-app
  └── PHP 7.4
```

Source code dipisahkan dari runtime writable, PHP-FPM diisolasi dalam container, Nginx tetap menjadi web server pada host, dan setiap aplikasi mempunyai konfigurasi deployment sendiri.

Pendekatan ini memberikan fondasi yang lebih konsisten untuk:

- deployment aplikasi;
- pengelolaan multi-version PHP;
- hardening;
- troubleshooting;
- backup;
- monitoring;
- isolasi aplikasi; dan
- upgrade PHP.

Dengan adanya script `create-php-app.sh`, standar tersebut juga dapat diterapkan secara konsisten setiap kali aplikasi baru akan ditempatkan pada server.

---

## Referensi

- Docker Official Images — PHP
- PHP Documentation — PHP-FPM
- PHP Documentation — OPcache
- Nginx Documentation
- Docker Compose Documentation
- Laravel Documentation
- CodeIgniter 4 Documentation
````
