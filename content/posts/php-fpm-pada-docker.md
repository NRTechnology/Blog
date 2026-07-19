+++
date = '2026-07-19T09:38:30+07:00'
draft = false
title = 'Membangun Web Server dengan Nginx, PHP-FPM Docker, dan MariaDB'
+++

Seperti yang sudah diterangkan pada tutorial sebelumnya, Docker adalah platform container yang memungkinkan kita menjalankan aplikasi dalam lingkungan yang terisolasi.

Pada tutorial kali ini, kita akan membangun sebuah web server menggunakan Nginx, MariaDB, dan hanya PHP-FPM yang berjalan di dalam container Docker. Dengan arsitektur ini, satu web server atau satu Virtual Machine dapat digunakan untuk menyediakan beberapa lingkungan PHP dengan versi yang berbeda.

## Persiapan

Hal yang pertama harus dilakukan adalah melakukan intalasi Ubuntu Server versi 24.04.4 LTS atau yang lebih baru dengan hanya menginstal minimum system dan ssh server. Kemudian setelah instalasi Ubuntu Server selesai pastikan sistem Ubuntu sudah diperbarui.

```bash
sudo apt update
sudo apt upgrade -y
```

## Instalasi Software yang Dibutuhkan

Selanjutnya kita harus melakukan instalasi Nginx, Docker, dan MariaDB

```bash
apt update
apt install -y \
    nginx \
    mariadb-server \
    docker.io \
    docker-compose-v2 \
    unzip \
    curl \
    git
```

## Menyiapkan direktori yang dibutuhkan

Pada tutorial ini, kita akan menempatkan semua sumber daya aplikasi web pada direktori /var/apps dan pada tulisan ini kita akan membuat sebuah website sederhana dengan nama project website

```bash
mkdir -p /var/apps/website/{htdocs,data}
mkdir -p /var/apps/website/data/{uploads,cache,sessions,logs}
```

kemudian kita harus membuat direktori tree untuk environment dockernya

```bash
mkdir -p \
  /opt/docker/images/php8.3 \
  /opt/docker/apps/website \
  /opt/docker/scripts \
  /opt/docker/templates
```
kenapa kita memisahkan antara apps dan docker karena pada arsitektur ini, /var/apps digunakan untuk menyimpan source code dan data runtime aplikasi, sedangkan /opt/docker digunakan untuk menyimpan Docker image definition dan konfigurasi container. Pemisahan ini bertujuan agar source code aplikasi tidak tercampur dengan konfigurasi infrastruktur Docker, sehingga nantinya akan didapatkan struktur direktori seperti berikut :

```bash
/var/apps
└── website
    ├── htdocs       ← Source code
    └── data         ← Runtime data

/opt/docker
├── images           ← Docker image definitions
└── apps             ← Konfigurasi container
```

kemudian buat file yang akan kita butuhkan

```bash
touch /opt/docker/images/php8.3/Dockerfile
touch /opt/docker/images/php8.3/php.ini

touch /opt/docker/apps/website/docker-compose.yml
touch /opt/docker/apps/website/zz-custom.conf
```

## Konfigurasi PHP-FPM pada Docker

Setelah semua direktori dan file yang dibutuhkan dibuat, sekarang kita masuk pembuatan file konfigurasi untuk Docker dan PHP-FPM

Pertama kita buat file Dockerfile

```bash
nano /opt/docker/images/php8.3/Dockerfile
```

Masukkan konfigurasi berikut

```bash
FROM php:8.3-fpm

RUN apt-get update && apt-get install -y \
    libzip-dev \
    libicu-dev \
    libpng-dev \
    libjpeg62-turbo-dev \
    libfreetype6-dev \
    libonig-dev \
    libxml2-dev \
    unzip \
    git \
    curl \
 && docker-php-ext-configure gd --with-freetype --with-jpeg \
 && docker-php-ext-install \
    mysqli \
    pdo_mysql \
    intl \
    zip \
    gd \
    bcmath \
    exif \
    opcache \
    sockets \
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/*

COPY --from=composer:2 /usr/bin/composer /usr/bin/composer
```

Kemudian konfigurasi untuk php.ini

```bash
nano /opt/docker/images/php8.3/php.ini
```

Masukan konfigurasi berikut

```bash
expose_php = Off

display_errors = Off
display_startup_errors = Off
log_errors = On

memory_limit = 512M

upload_max_filesize = 100M
post_max_size = 100M

max_execution_time = 120
max_input_time = 120

date.timezone = Asia/Jakarta

cgi.fix_pathinfo = 0

opcache.enable=1
opcache.memory_consumption=256
opcache.max_accelerated_files=30000
opcache.validate_timestamps=1
opcache.revalidate_freq=2

```

Build image PHP 8.3

```bash
cd /opt/docker/images/php8.3
docker build -t local/php:8.3 .
```

Verifikasi bahwa proses pembuatan image berhasil 

```bash
docker image ls
```

harusnya paling tidak akan muncul image berikut

```bash
REPOSITORY   TAG    IMAGE ID
local/php    8.3    xxxxxxxxxxxx
```

Setelah image berhasil dibuat, kita perlu mengetahui UID dan GID user www-data yang digunakan oleh PHP-FPM di dalam container. Informasi ini nantinya digunakan untuk mengatur kepemilikan dan permission direktori runtime agar PHP-FPM dapat menulis ke direktori uploads, cache, sessions, dan logs

```bash
docker run --rm local/php:8.3 id www-data
```

jika hasilnya 

```bash
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
Hasil tersebut menunjukkan bahwa user www-data di dalam container menggunakan UID 33 dan GID 33. Oleh karena itu, permission direktori runtime pada host perlu diatur agar proses PHP-FPM yang berjalan sebagai www-data dapat melakukan operasi baca dan tulis pada direktori yang diperlukan.

Langkah selanjutnya adalah membuat konfigurasi untuk zz-custom.conf

```bash
nano /opt/docker/apps/website/zz-custom.conf
```

masukan konfigurasi berikut

```bash
[www]

listen = /run/php/website.sock

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

kemudian buat konfigurasi untuk docker-compose.yml

```bash
nano /opt/docker/apps/website/docker-compose.yml
```

masukan konfigurasi beritkut

```bash
services:

  app:
    container_name: website-php
    read_only: true

    image: local/php:8.3

    restart: unless-stopped
    init: true
    stop_grace_period: 30s

    security_opt:
      - no-new-privileges:true

    volumes:
      - /run/php:/run/php
      - /run/mysqld:/run/mysqld:ro

      - /opt/docker/images/php8.3/php.ini:/usr/local/etc/php/conf.d/99-custom.ini:ro
      - /opt/docker/apps/website/zz-custom.conf:/usr/local/etc/php-fpm.d/zz-custom.conf:ro

      # Source code (READ ONLY)
      - /var/apps/website/htdocs:/var/www/html:ro

      # Runtime data (READ WRITE)
      - /var/apps/website/data/uploads:/var/www/uploads:rw
      - /var/apps/website/data/cache:/var/www/cache:rw
      - /var/apps/website/data/sessions:/var/lib/php/sessions:rw
      - /var/apps/website/data/logs:/var/log/php:rw

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

pada konfigurasi docker-compose.yml terdapat perintah

```bash
read_only: true
```

hal tersebut penting dilakukan untuk membuat environment docker menjadi read-only dan hanya membolehkan beberapa direktori yang memiliki akses tulis. Tujuan pemberian perintah read-only adalah untuk mencegah eksploitasi environment jika ternyata pada aplikasi yang di hosting memiliki celah keamanan seperti RCE atau arbitrary file write, sehingga proses di dalam container tidak bebas memodifikasi filesystem container. Akses tulis hanya diberikan pada direktori berikut :

```bash
/var/www/uploads
/var/www/cache
/var/lib/php/sessions
/var/log/php
```

pada docker-compose.yml juga terdapat perintah 

```bash
- /var/apps/website/htdocs:/var/www/html:ro
```

perintah tersebut bertujuan agar source code aplikasi di-mount sebagai read-only. Dengan konfigurasi ini, proses PHP-FPM di dalam container tidak dapat memodifikasi file aplikasi seperti index.php. Hal ini membantu mengurangi risiko perubahan source code apabila aplikasi mengalami celah keamanan yang memungkinkan arbitrary file write atau remote code execution.

selain dua perintah penting pada docker-compose.yml yang sudah saya jelaskan diatas, ada perintah lain yang tidak kalah lebih penting yaitu :

```bash
- /run/php:/run/php
- /run/mysqld:/run/mysqld:ro
```

Direktori /run/php digunakan untuk berbagi Unix socket antara PHP-FPM yang berjalan di dalam container dan Nginx yang berjalan pada host. PHP-FPM akan membuat socket /run/php/website.sock, kemudian Nginx menggunakan socket tersebut untuk meneruskan request PHP ke PHP-FPM.

Direktori /run/mysqld di-mount secara read-only agar aplikasi PHP di dalam container dapat mengakses Unix socket MariaDB pada /run/mysqld/mysqld.sock. Dengan pendekatan ini, PHP dapat berkomunikasi dengan MariaDB yang berjalan pada host tanpa harus membuka port database ke jaringan Docker.

## Mengatur Permission Direktori Runtime

Pada konfigurasi sebelumnya, source code aplikasi yang berada pada direktori `/var/apps/website/htdocs` di-mount ke dalam container sebagai read-only. Dengan demikian, PHP-FPM tidak dapat melakukan perubahan terhadap source code aplikasi.

Namun, aplikasi tetap membutuhkan beberapa direktori yang memiliki akses baca dan tulis untuk menyimpan file upload, cache, session, dan log. Direktori tersebut adalah:

```text
/var/apps/website/data/uploads
/var/apps/website/data/cache
/var/apps/website/data/sessions
/var/apps/website/data/logs
```

Direktori tersebut kemudian di-mount ke dalam container sebagai:

```text
/var/www/uploads
/var/www/cache
/var/lib/php/sessions
/var/log/php
```

Sebelumnya kita sudah memeriksa UID dan GID user `www-data` pada image PHP menggunakan perintah:

```bash
docker run --rm local/php:8.3 id www-data
```

Apabila hasilnya adalah:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

maka proses PHP-FPM yang berjalan sebagai user `www-data` di dalam container akan mengakses bind mount pada host menggunakan UID `33` dan GID `33`.

Oleh karena itu, kita perlu memberikan kepemilikan direktori runtime kepada UID dan GID tersebut.

Jalankan perintah berikut:

```bash
chown -R 33:33 /var/apps/website/data/uploads
chown -R 33:33 /var/apps/website/data/cache
chown -R 33:33 /var/apps/website/data/sessions
chown -R 33:33 /var/apps/website/data/logs
```

Kemudian atur permission direktori agar hanya pemilik dan grup yang memiliki akses penuh:

```bash
chmod -R 770 /var/apps/website/data/uploads
chmod -R 770 /var/apps/website/data/cache
chmod -R 770 /var/apps/website/data/sessions
chmod -R 770 /var/apps/website/data/logs
```

Untuk memeriksa hasil konfigurasi permission, jalankan:

```bash
ls -ld /var/apps/website/data/*
```

Hasilnya kurang lebih akan terlihat seperti berikut:

```text
drwxrwx--- 2 www-data www-data ... cache
drwxrwx--- 2 www-data www-data ... logs
drwxrwx--- 2 www-data www-data ... sessions
drwxrwx--- 2 www-data www-data ... uploads
```

Perlu diperhatikan bahwa pada host, nama user yang ditampilkan oleh perintah `ls` bergantung pada apakah UID `33` terdaftar sebagai user `www-data`. Yang terpenting adalah UID dan GID pada host sesuai dengan UID dan GID proses PHP-FPM di dalam container.

Dengan konfigurasi tersebut, proses PHP-FPM dapat melakukan operasi baca dan tulis pada direktori runtime, tetapi tetap tidak dapat melakukan perubahan terhadap source code aplikasi yang berada pada:

```text
/var/www/html
```

karena direktori tersebut di-mount sebagai read-only:

```yaml
- /var/apps/website/htdocs:/var/www/html:ro
```

Arsitektur permission yang dihasilkan menjadi:

```text
/var/www/html
    └── READ ONLY
        └── Source code aplikasi

/var/www/uploads
    └── READ / WRITE
        └── File yang diunggah aplikasi

/var/www/cache
    └── READ / WRITE
        └── Cache aplikasi

/var/lib/php/sessions
    └── READ / WRITE
        └── PHP session

/var/log/php
    └── READ / WRITE
        └── Log PHP
```

Pemisahan antara source code yang bersifat read-only dan direktori runtime yang bersifat read-write merupakan salah satu lapisan keamanan pada arsitektur ini. Apabila aplikasi memiliki celah keamanan yang memungkinkan arbitrary file write, proses PHP tetap tidak dapat mengganti file source code seperti `index.php` karena filesystem source code telah di-mount sebagai read-only.

Namun perlu dipahami bahwa direktori runtime seperti `uploads` tetap dapat ditulisi oleh aplikasi. Oleh karena itu, pada konfigurasi Nginx yang akan dibuat pada tahap berikutnya, kita juga akan memastikan bahwa file script seperti `.php`, `.phtml`, dan `.phar` yang berhasil masuk ke direktori upload tidak dapat dieksekusi sebagai script PHP.

Setelah permission direktori runtime selesai dikonfigurasi, langkah berikutnya adalah menjalankan container PHP-FPM dan melakukan pengujian untuk memastikan bahwa source code benar-benar bersifat read-only sementara direktori runtime tetap dapat ditulisi oleh aplikasi.

## Menjalankan Container PHP-FPM

Setelah Docker image PHP 8.3, konfigurasi PHP-FPM, Docker Compose, dan permission direktori runtime selesai disiapkan, langkah berikutnya adalah menjalankan container PHP-FPM.

Masuk terlebih dahulu ke direktori konfigurasi Docker aplikasi:

```bash
cd /opt/docker/apps/website
```

Sebelum menjalankan container, kita dapat melakukan validasi terhadap file `docker-compose.yml` menggunakan perintah berikut:

```bash
docker compose config
```

Perintah tersebut digunakan untuk memeriksa dan menampilkan konfigurasi Docker Compose yang telah diproses. Jika terdapat kesalahan pada struktur atau sintaks YAML, Docker Compose akan menampilkan pesan kesalahan yang dapat diperbaiki sebelum container dijalankan.

Jika konfigurasi sudah benar, jalankan container menggunakan perintah:

```bash
docker compose up -d
```

Parameter `-d` atau `--detach` digunakan agar container berjalan di background sehingga terminal dapat tetap digunakan untuk menjalankan perintah lainnya.

Setelah proses selesai, periksa container yang sedang berjalan:

```bash
docker ps
```

Hasilnya kurang lebih akan terlihat seperti berikut:

```text
CONTAINER ID   IMAGE           COMMAND                  STATUS
xxxxxxxxxxxx   local/php:8.3   "docker-php-entrypoi…"   Up
```

Kita juga dapat memeriksa status container menggunakan Docker Compose:

```bash
docker compose ps
```

Jika healthcheck yang telah dikonfigurasi berjalan dengan baik, setelah beberapa saat status container seharusnya berubah menjadi:

```text
Up (healthy)
```

atau pada output Docker Compose akan terlihat status:

```text
running (healthy)
```

Untuk melihat informasi status healthcheck secara langsung, jalankan:

```bash
docker inspect website-php \
  --format '{{.State.Status}} - {{.State.Health.Status}}'
```

Jika container berjalan dengan normal, hasilnya seharusnya:

```text
running - healthy
```

Selanjutnya, periksa log PHP-FPM untuk memastikan tidak terdapat kesalahan saat proses startup:

```bash
docker logs website-php
```

Jika PHP-FPM berhasil berjalan, kurang lebih akan muncul informasi:

```text
NOTICE: fpm is running
NOTICE: ready to handle connections
```

Apabila container mengalami restart berulang atau tidak dapat berjalan, periksa status seluruh container:

```bash
docker ps -a
```

Kemudian periksa log container:

```bash
docker logs website-php --tail=100
```

Untuk memantau log secara real-time, gunakan:

```bash
docker logs -f website-php
```

Tekan `Ctrl+C` untuk keluar dari pemantauan log tanpa menghentikan container.

### Memeriksa Versi PHP

Setelah container berhasil berjalan, kita dapat memastikan versi PHP yang digunakan dengan menjalankan:

```bash
docker exec website-php php -v
```

Hasilnya akan menampilkan versi PHP yang tersedia di dalam container, misalnya:

```text
PHP 8.3.x (cli)
```

Kita juga dapat memeriksa modul PHP yang telah terinstal:

```bash
docker exec website-php php -m
```

Untuk memastikan beberapa ekstensi penting telah tersedia, gunakan:

```bash
docker exec website-php php -m | grep -E 'mysqli|pdo_mysql|intl|zip|gd|bcmath|exif|opcache|sockets'
```

Ekstensi yang sebelumnya ditambahkan melalui `Dockerfile` seharusnya muncul pada hasil perintah tersebut.

### Memeriksa Konfigurasi PHP-FPM

Selanjutnya, lakukan pengujian terhadap konfigurasi PHP-FPM:

```bash
docker exec website-php php-fpm -t
```

Apabila konfigurasi PHP-FPM tidak memiliki kesalahan, akan muncul informasi bahwa pengujian konfigurasi berhasil.

Untuk melihat konfigurasi PHP-FPM secara lebih lengkap, gunakan:

```bash
docker exec website-php php-fpm -tt
```

Pastikan konfigurasi pool PHP-FPM menggunakan Unix socket yang telah ditentukan:

```text
listen = /run/php/website.sock
```

Kita juga dapat memeriksa keberadaan socket tersebut dari host:

```bash
ls -lah /run/php/website.sock
```

Socket tersebut seharusnya memiliki permission yang sesuai dengan konfigurasi `zz-custom.conf`, yaitu:

```ini
listen.owner = www-data
listen.group = www-data
listen.mode = 0660
```

Unix socket `/run/php/website.sock` nantinya akan digunakan oleh Nginx yang berjalan pada host untuk meneruskan request PHP ke PHP-FPM yang berjalan di dalam container.

Alur komunikasinya adalah sebagai berikut:

```text
Client
   │
   ▼
Nginx (Host)
   │
   │ /run/php/website.sock
   ▼
PHP-FPM (Docker)
   │
   ▼
Aplikasi PHP
```

### Menguji Source Code Read-Only

Setelah container berjalan, kita perlu memastikan bahwa direktori source code benar-benar tidak dapat ditulisi dari dalam container.

Jalankan:

```bash
docker exec website-php touch /var/www/html/test.php
```

Karena direktori `/var/www/html` di-mount sebagai read-only:

```yaml
- /var/apps/website/htdocs:/var/www/html:ro
```

maka perintah tersebut seharusnya gagal dengan pesan seperti:

```text
touch: cannot touch '/var/www/html/test.php': Read-only file system
```

Hasil tersebut menunjukkan bahwa proses di dalam container tidak dapat membuat atau mengubah file pada direktori source code aplikasi.

### Menguji Direktori Runtime Read-Write

Selanjutnya, lakukan pengujian pada direktori runtime yang memang diperbolehkan untuk ditulisi.

Untuk memastikan pengujian dilakukan menggunakan user yang sama dengan worker PHP-FPM, yaitu `www-data`, jalankan:

```bash
docker exec -u www-data website-php touch /var/www/uploads/test.txt
docker exec -u www-data website-php touch /var/www/cache/test.txt
docker exec -u www-data website-php touch /var/lib/php/sessions/test.txt
docker exec -u www-data website-php touch /var/log/php/test.txt
```

Jika permission direktori runtime telah dikonfigurasi dengan benar, seluruh perintah tersebut seharusnya berhasil tanpa menampilkan pesan kesalahan.

Periksa file yang telah dibuat:

```bash
docker exec website-php ls -lah /var/www/uploads/test.txt
docker exec website-php ls -lah /var/www/cache/test.txt
docker exec website-php ls -lah /var/lib/php/sessions/test.txt
docker exec website-php ls -lah /var/log/php/test.txt
```

Setelah pengujian selesai, hapus file pengujian:

```bash
docker exec -u www-data website-php rm -f /var/www/uploads/test.txt
docker exec -u www-data website-php rm -f /var/www/cache/test.txt
docker exec -u www-data website-php rm -f /var/lib/php/sessions/test.txt
docker exec -u www-data website-php rm -f /var/log/php/test.txt
```

Dengan demikian, kita telah memastikan bahwa container PHP-FPM memiliki karakteristik sebagai berikut:

* Root filesystem container bersifat read-only.
* Source code aplikasi pada `/var/www/html` bersifat read-only.
* Direktori upload dapat ditulisi oleh PHP-FPM.
* Direktori cache dapat ditulisi oleh PHP-FPM.
* Direktori session dapat ditulisi oleh PHP-FPM.
* Direktori log dapat ditulisi oleh PHP-FPM.
* PHP-FPM berjalan dan siap menerima request melalui Unix socket `/run/php/website.sock`.

Setelah seluruh pengujian tersebut berhasil, langkah berikutnya adalah melakukan konfigurasi Nginx pada host agar request dari web dapat diteruskan ke PHP-FPM yang berjalan di dalam container melalui Unix socket.

## Konfigurasi Nginx

Setelah container PHP-FPM berhasil berjalan dan Unix socket `/run/php/website.sock` telah tersedia, langkah berikutnya adalah melakukan konfigurasi Nginx.

Pada arsitektur yang digunakan dalam tutorial ini, Nginx berjalan secara langsung pada host Ubuntu, sedangkan PHP-FPM berjalan di dalam container Docker. Komunikasi antara Nginx dan PHP-FPM dilakukan menggunakan Unix socket.

Alur komunikasi request PHP adalah sebagai berikut:

```text
Client
   │
   ▼
Nginx (Host)
   │
   │ Unix Socket
   │ /run/php/website.sock
   ▼
PHP-FPM (Docker)
   │
   ▼
/var/www/html
```

Perlu diperhatikan bahwa lokasi source code yang dilihat oleh Nginx dan PHP-FPM berbeda.

Pada host, source code berada di:

```text
/var/apps/website/htdocs
```

Sedangkan di dalam container PHP-FPM, direktori tersebut di-mount ke:

```text
/var/www/html
```

Oleh karena itu, konfigurasi Nginx harus menggunakan path `/var/apps/website/htdocs` untuk membaca file statis dan memeriksa keberadaan file. Namun, parameter `SCRIPT_FILENAME` yang dikirimkan kepada PHP-FPM harus menggunakan path `/var/www/html`, karena path tersebut merupakan lokasi source code dari sudut pandang container PHP-FPM.

### Membuat Virtual Host Nginx

Buat file konfigurasi virtual host baru:

```bash
nano /etc/nginx/sites-available/website.conf
```

Masukkan konfigurasi berikut:

```nginx
server {
    listen 8080;
    server_name example.com;

    root /var/apps/website/htdocs;
    index index.php index.html;

    charset utf-8;

    access_log /var/log/nginx/website.access.log;
    error_log  /var/log/nginx/website.error.log warn;

    client_max_body_size 100M;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        try_files $uri =404;

        include fastcgi_params;

        fastcgi_param SCRIPT_FILENAME /var/www/html$fastcgi_script_name;
        fastcgi_param DOCUMENT_ROOT /var/www/html;

        fastcgi_pass unix:/run/php/website.sock;
    }
}
```

Ganti:

```text
example.com
```

dengan domain atau hostname yang akan digunakan oleh aplikasi.

Pada lingkungan LAB, apabila belum menggunakan domain, konfigurasi `server_name` dapat disesuaikan dengan hostname atau alamat yang digunakan untuk mengakses server.

### Penjelasan Document Root

Konfigurasi:

```nginx
root /var/apps/website/htdocs;
```

menentukan lokasi source code aplikasi yang dapat diakses oleh Nginx pada host.

Nginx menggunakan direktori tersebut untuk melayani file statis seperti:

```text
HTML
CSS
JavaScript
Gambar
Dokumen
```

Sementara itu, request terhadap file PHP akan diteruskan kepada PHP-FPM.

### Konfigurasi Front Controller

Konfigurasi:

```nginx
location / {
    try_files $uri $uri/ /index.php?$query_string;
}
```

memerintahkan Nginx untuk terlebih dahulu mencari file atau direktori yang diminta.

Apabila file ditemukan, Nginx akan melayani file tersebut secara langsung.

Apabila file atau direktori tidak ditemukan, request akan diteruskan ke:

```text
/index.php
```

dengan query string tetap dipertahankan.

Konfigurasi ini umum digunakan oleh aplikasi PHP yang menggunakan pola front controller.

### Konfigurasi PHP-FPM

Bagian berikut digunakan untuk menangani request terhadap file PHP:

```nginx
location ~ \.php$ {
    try_files $uri =404;

    include fastcgi_params;

    fastcgi_param SCRIPT_FILENAME /var/www/html$fastcgi_script_name;
    fastcgi_param DOCUMENT_ROOT /var/www/html;

    fastcgi_pass unix:/run/php/website.sock;
}
```

Konfigurasi:

```nginx
try_files $uri =404;
```

memastikan bahwa Nginx hanya meneruskan request PHP apabila file tersebut benar-benar tersedia pada document root di host.

Selanjutnya:

```nginx
fastcgi_param SCRIPT_FILENAME /var/www/html$fastcgi_script_name;
```

memberikan lokasi file PHP kepada PHP-FPM.

Walaupun source code pada host berada di:

```text
/var/apps/website/htdocs
```

PHP-FPM yang berjalan di dalam container melihat source code tersebut sebagai:

```text
/var/www/html
```

Oleh karena itu, apabila client meminta:

```text
/index.php
```

Nginx akan mengirimkan parameter:

```text
SCRIPT_FILENAME=/var/www/html/index.php
```

kepada PHP-FPM.

Terakhir, konfigurasi:

```nginx
fastcgi_pass unix:/run/php/website.sock;
```

memerintahkan Nginx untuk meneruskan request PHP kepada PHP-FPM menggunakan Unix socket yang sebelumnya dibuat oleh PHP-FPM.

### Mengaktifkan Virtual Host

Setelah konfigurasi selesai dibuat, aktifkan virtual host dengan membuat symbolic link:

```bash
ln -s /etc/nginx/sites-available/website.conf \
      /etc/nginx/sites-enabled/website.conf
```

Periksa symbolic link:

```bash
ls -lah /etc/nginx/sites-enabled/
```

Selanjutnya, lakukan pengujian terhadap konfigurasi Nginx:

```bash
nginx -t
```

Apabila konfigurasi tidak memiliki kesalahan, hasilnya kurang lebih:

```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Kemudian reload Nginx:

```bash
systemctl reload nginx
```

Periksa status layanan Nginx:

```bash
systemctl status nginx
```

### Membuat File PHP untuk Pengujian

Buat file PHP sederhana pada document root:

```bash
nano /var/apps/website/htdocs/index.php
```

Masukkan:

```php
<?php
phpinfo();
?>
```

Simpan file tersebut, kemudian pastikan file dapat dibaca:

```bash
ls -lah /var/apps/website/htdocs/index.php
```

Untuk pengujian awal dari server yang sama, gunakan:

```bash
curl -H "Host: example.com" http://127.0.0.1:8080/
```

Ganti `example.com` sesuai dengan nilai `server_name` pada konfigurasi Nginx.

Apabila konfigurasi Nginx dan PHP-FPM berjalan dengan benar, output HTML dari halaman `phpinfo()` akan ditampilkan.

Kita juga dapat memeriksa log PHP-FPM:

```bash
docker logs website-php --tail=50
```

dan log Nginx:

```bash
tail -n 50 /var/log/nginx/website.access.log
tail -n 50 /var/log/nginx/website.error.log
```

### Menguji Komunikasi Nginx dan PHP-FPM

Untuk memastikan Unix socket tersedia, jalankan:

```bash
ls -lah /run/php/website.sock
```

Kemudian pastikan Nginx dapat mengakses socket tersebut.

Pada konfigurasi PHP-FPM sebelumnya, socket dibuat dengan:

```ini
listen.owner = www-data
listen.group = www-data
listen.mode = 0660
```

Karena Nginx pada Ubuntu umumnya menjalankan worker process sebagai user `www-data`, Nginx dapat berkomunikasi dengan PHP-FPM melalui socket tersebut.

Apabila terjadi error:

```text
502 Bad Gateway
```

periksa terlebih dahulu apakah container PHP-FPM berjalan:

```bash
docker ps
```

Kemudian periksa keberadaan socket:

```bash
ls -lah /run/php/website.sock
```

Periksa log container:

```bash
docker logs website-php --tail=100
```

dan log error Nginx:

```bash
tail -n 100 /var/log/nginx/website.error.log
```

### Catatan Mengenai Port 8080

Pada contoh konfigurasi ini, Nginx menggunakan:

```nginx
listen 8080;
```

Port tersebut dapat digunakan pada lingkungan LAB atau ketika terdapat reverse proxy lain di depan Nginx.

Apabila Nginx akan menerima koneksi HTTP secara langsung, port dapat disesuaikan menjadi:

```nginx
listen 80;
```

Untuk implementasi produksi yang menggunakan HTTPS, konfigurasi TLS/SSL dapat ditambahkan pada tahap berikutnya.

### Menghapus File phpinfo()

Setelah pengujian selesai, file `phpinfo()` sebaiknya dihapus atau diganti dengan aplikasi yang sebenarnya karena halaman tersebut menampilkan informasi detail mengenai konfigurasi PHP dan environment server.

Hapus file pengujian dengan:

```bash
rm /var/apps/website/htdocs/index.php
```

atau ganti isinya dengan aplikasi PHP yang akan digunakan.

Sampai tahap ini, komunikasi antara Nginx pada host dan PHP-FPM di dalam container telah berhasil dikonfigurasi menggunakan Unix socket.

Arsitektur yang telah berjalan adalah:

```text
Client
   │
   ▼
Nginx (Host)
   │
   │ /run/php/website.sock
   ▼
PHP-FPM (Docker)
   │
   ▼
Source Code
/var/www/html
```

Langkah berikutnya adalah mengonfigurasi dan menguji koneksi antara PHP-FPM di dalam container dengan MariaDB yang berjalan pada host menggunakan Unix socket `/run/mysqld/mysqld.sock`.

## Konfigurasi MariaDB

Setelah komunikasi antara Nginx pada host dan PHP-FPM di dalam container berhasil dikonfigurasi, langkah berikutnya adalah mengonfigurasi MariaDB.

Pada arsitektur yang digunakan dalam tutorial ini, MariaDB berjalan secara langsung pada host Ubuntu dan tidak dijalankan di dalam container Docker. Aplikasi PHP yang berada di dalam container akan terhubung ke MariaDB menggunakan Unix socket.

Arsitektur komunikasi yang digunakan adalah sebagai berikut:

```text
Client
   │
   ▼
Nginx (Host)
   │
   │ /run/php/website.sock
   ▼
PHP-FPM (Docker)
   │
   │ /run/mysqld/mysqld.sock
   ▼
MariaDB (Host)
```

Dengan pendekatan ini, komunikasi antara PHP dan MariaDB tidak perlu menggunakan jaringan TCP Docker. Direktori `/run/mysqld` pada host sebelumnya telah di-mount ke dalam container secara read-only melalui konfigurasi:

```yaml
- /run/mysqld:/run/mysqld:ro
```

Dengan demikian, PHP yang berjalan di dalam container dapat mengakses Unix socket MariaDB pada:

```text
/run/mysqld/mysqld.sock
```

### Memastikan MariaDB Berjalan

Periksa terlebih dahulu status MariaDB:

```bash
systemctl status mariadb
```

Apabila MariaDB berjalan dengan normal, status service akan menunjukkan:

```text
active (running)
```

Jika MariaDB belum berjalan, aktifkan menggunakan:

```bash
systemctl enable --now mariadb
```

Selanjutnya, periksa apakah Unix socket MariaDB tersedia:

```bash
ls -lah /run/mysqld/mysqld.sock
```

Hasilnya kurang lebih akan terlihat seperti:

```text
srwxrwxrwx 1 mysql mysql ... /run/mysqld/mysqld.sock
```

Lokasi socket yang digunakan MariaDB juga dapat diperiksa dengan:

```bash
mariadb -e "SHOW VARIABLES LIKE 'socket';"
```

Pastikan nilai yang ditampilkan adalah:

```text
/run/mysqld/mysqld.sock
```

atau sesuai dengan lokasi socket MariaDB pada server yang digunakan.

### Menjalankan Hardening Awal MariaDB

Setelah instalasi MariaDB selesai, jalankan konfigurasi keamanan awal menggunakan:

```bash
mariadb-secure-installation
```

Ikuti proses yang ditampilkan pada terminal.

Pada tahap ini, beberapa konfigurasi keamanan dasar yang dapat dilakukan antara lain:

* Menghapus anonymous user.
* Menonaktifkan login root dari remote host.
* Menghapus database test.
* Melakukan reload privilege table.

Konfigurasi yang dipilih dapat disesuaikan dengan kebijakan keamanan server yang digunakan.

### Memastikan MariaDB Tidak Terbuka ke Jaringan Eksternal

Karena aplikasi PHP pada server ini menggunakan Unix socket, MariaDB tidak perlu diekspos ke jaringan publik.

Periksa port MariaDB menggunakan:

```bash
ss -lntp | grep 3306
```

Untuk arsitektur ini, MariaDB sebaiknya hanya mendengarkan koneksi TCP pada localhost, misalnya:

```text
127.0.0.1:3306
```

dan bukan:

```text
0.0.0.0:3306
```

Apabila MariaDB mendengarkan pada:

```text
127.0.0.1:3306
```

maka koneksi TCP ke MariaDB hanya dapat dilakukan dari host lokal.

Meskipun demikian, aplikasi PHP dalam container yang kita buat tidak menggunakan koneksi TCP tersebut. Aplikasi akan menggunakan Unix socket `/run/mysqld/mysqld.sock`.

### Membuat Database Aplikasi

Masuk ke MariaDB menggunakan akun administrator:

```bash
sudo mariadb
```

Kemudian buat database untuk aplikasi:

```sql
CREATE DATABASE website
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;
```

Periksa database yang telah dibuat:

```sql
SHOW DATABASES;
```

Database `website` seharusnya muncul pada daftar database.

### Membuat User Khusus Aplikasi

Aplikasi sebaiknya tidak menggunakan user `root` maupun user administrator MariaDB untuk koneksi database.

Buat user khusus untuk aplikasi, misalnya:

```sql
CREATE USER 'website'@'localhost'
IDENTIFIED BY 'GantiDenganPasswordYangKuat';
```

Kemudian berikan hak akses hanya terhadap database `website`:

```sql
GRANT ALL PRIVILEGES
ON website.*
TO 'website'@'localhost';
```

Setelah itu, jalankan:

```sql
FLUSH PRIVILEGES;
```

Periksa hak akses user:

```sql
SHOW GRANTS FOR 'website'@'localhost';
```

Dengan konfigurasi tersebut, user `website` hanya memiliki hak akses terhadap:

```text
website.*
```

dan tidak memiliki akses administratif terhadap seluruh server MariaDB.

Gunakan password yang kuat dan unik pada implementasi sebenarnya. Jangan menggunakan password contoh yang terdapat pada dokumentasi ini untuk lingkungan produksi.

Keluar dari MariaDB:

```sql
EXIT;
```

### Menguji Login Database dari Host

Sebelum melakukan pengujian dari container, pastikan user database dapat digunakan dari host melalui Unix socket.

Jalankan:

```bash
mariadb \
  --socket=/run/mysqld/mysqld.sock \
  -u website \
  -p \
  website
```

Masukkan password user `website` ketika diminta.

Apabila berhasil, jalankan:

```sql
SELECT DATABASE();
SELECT CURRENT_USER();
```

Hasilnya kurang lebih:

```text
DATABASE()
website
```

dan:

```text
CURRENT_USER()
website@localhost
```

Kemudian keluar:

```sql
EXIT;
```

### Memastikan Unix Socket Tersedia di Dalam Container

Karena direktori `/run/mysqld` telah di-mount melalui Docker Compose:

```yaml
- /run/mysqld:/run/mysqld:ro
```

periksa keberadaan socket dari dalam container:

```bash
docker exec website-php ls -lah /run/mysqld/mysqld.sock
```

Jika socket dapat terlihat dari dalam container, PHP dapat menggunakan socket tersebut untuk melakukan koneksi ke MariaDB.

Mount menggunakan mode `ro` atau read-only karena container PHP hanya membutuhkan akses untuk menggunakan Unix socket dan tidak perlu membuat atau menghapus file pada direktori `/run/mysqld`.

### Menguji Koneksi PHP ke MariaDB

Selanjutnya, lakukan pengujian menggunakan PDO dari dalam container.

Jalankan:

```bash
docker exec -it website-php php -r '
try {
    $pdo = new PDO(
        "mysql:unix_socket=/run/mysqld/mysqld.sock;dbname=website;charset=utf8mb4",
        "website",
        "GantiDenganPasswordYangKuat",
        [
            PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION
        ]
    );

    echo "=== KONEKSI BERHASIL ===\n";
    echo "Server   : " . $pdo->getAttribute(PDO::ATTR_SERVER_VERSION) . "\n";
    echo "Database : " . $pdo->query("SELECT DATABASE()")->fetchColumn() . "\n";
    echo "User     : " . $pdo->query("SELECT CURRENT_USER()")->fetchColumn() . "\n";

} catch (PDOException $e) {
    echo "=== KONEKSI GAGAL ===\n";
    echo $e->getMessage() . "\n";
}
'
```

Ganti:

```text
GantiDenganPasswordYangKuat
```

dengan password user database yang sebelumnya dibuat.

Apabila koneksi berhasil, hasilnya kurang lebih:

```text
=== KONEKSI BERHASIL ===
Server   : 10.x.x-MariaDB
Database : website
User     : website@localhost
```

Hasil tersebut menunjukkan bahwa PHP-FPM di dalam container dapat berkomunikasi dengan MariaDB yang berjalan pada host menggunakan Unix socket.

### Konfigurasi Koneksi pada Aplikasi PHP

Pada aplikasi PHP, koneksi PDO dapat dibuat menggunakan DSN:

```php
$dsn = 'mysql:unix_socket=/run/mysqld/mysqld.sock;dbname=website;charset=utf8mb4';

$pdo = new PDO(
    $dsn,
    'website',
    'PASSWORD_DATABASE',
    [
        PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
        PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC
    ]
);
```

Password database sebaiknya tidak ditulis langsung di dalam source code aplikasi. Pada implementasi sebenarnya, kredensial database sebaiknya disimpan menggunakan mekanisme konfigurasi atau secret management yang sesuai dengan aplikasi.

Perlu diperhatikan bahwa konfigurasi PHP-FPM yang digunakan pada tutorial ini memiliki:

```ini
clear_env = yes
```

Jika nantinya kredensial database akan diberikan menggunakan environment variable Docker, konfigurasi PHP-FPM perlu disesuaikan agar environment variable yang diperlukan dapat diteruskan secara eksplisit kepada worker PHP-FPM.

### Prinsip Least Privilege untuk User Database

Setiap aplikasi sebaiknya memiliki database dan user MariaDB masing-masing.

Sebagai contoh, apabila server nantinya memiliki beberapa aplikasi:

```text
website
simpeg
perizinan
keuangan
```

maka struktur user database sebaiknya dipisahkan:

```text
website@localhost
    └── website.*

simpeg@localhost
    └── simpeg.*

perizinan@localhost
    └── perizinan.*

keuangan@localhost
    └── keuangan.*
```

Dengan pendekatan tersebut, apabila salah satu aplikasi mengalami kebocoran kredensial database, akun aplikasi tersebut tidak secara otomatis memiliki akses ke database milik aplikasi lainnya.

Hindari menggunakan akun MariaDB administrator untuk koneksi rutin aplikasi.

### Pengujian Akhir Arsitektur

Setelah koneksi database berhasil, alur lengkap platform yang telah dibangun adalah:

```text
Client
   │
   │ HTTP/HTTPS
   ▼
Nginx
(Host Ubuntu)
   │
   │ Unix Socket
   │ /run/php/website.sock
   ▼
PHP-FPM
(Docker Container)
   │
   │ Unix Socket
   │ /run/mysqld/mysqld.sock
   ▼
MariaDB
(Host Ubuntu)
```

Source code aplikasi berada pada:

```text
/var/apps/website/htdocs
```

dan di-mount ke container sebagai:

```text
/var/www/html
```

dengan mode read-only.

Sedangkan direktori yang membutuhkan akses tulis dipisahkan menjadi:

```text
/var/www/uploads
/var/www/cache
/var/lib/php/sessions
/var/log/php
```

Dengan demikian, sampai tahap ini kita telah memiliki web server dengan Nginx dan MariaDB yang berjalan pada host serta PHP-FPM yang berjalan di dalam container Docker. Komunikasi Nginx ke PHP-FPM dan PHP ke MariaDB dilakukan menggunakan Unix socket tanpa perlu mengekspos port PHP-FPM maupun port database ke jaringan Docker.


## Hardening Nginx

Setelah Nginx, PHP-FPM, dan MariaDB berhasil terhubung, langkah berikutnya adalah melakukan hardening pada Nginx.

Hardening dilakukan untuk mengurangi permukaan serangan terhadap web server dan aplikasi. Konfigurasi ini tidak menggantikan keamanan pada source code aplikasi, tetapi berfungsi sebagai lapisan keamanan tambahan.

Pada tutorial ini, konfigurasi hardening akan dibuat secara modular agar dapat digunakan kembali oleh beberapa virtual host.

### Menyembunyikan Versi Nginx

Secara default, Nginx dapat menampilkan informasi versi pada beberapa halaman error dan HTTP response header. Untuk mengurangi informasi yang dapat diperoleh oleh pihak yang tidak berkepentingan, kita dapat menonaktifkan informasi versi Nginx.

Edit konfigurasi utama Nginx:

```bash
nano /etc/nginx/nginx.conf
```

Pastikan konfigurasi berikut berada di dalam blok `http`:

```nginx
http {
    server_tokens off;

    # Konfigurasi lainnya
}
```

Kemudian lakukan pengujian:

```bash
nginx -t
```

Jangan melakukan reload Nginx apabila pengujian konfigurasi menghasilkan error.

### Menambahkan Security Headers

Selanjutnya, buat file konfigurasi khusus untuk HTTP security headers:

```bash
nano /etc/nginx/conf.d/security.conf
```

Masukkan konfigurasi berikut:

```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
```

Konfigurasi tersebut akan berlaku pada server Nginx yang berada dalam konteks `http`, kecuali terdapat konfigurasi `add_header` lain pada level yang lebih spesifik.

Header:

```text
X-Frame-Options: SAMEORIGIN
```

digunakan untuk membatasi agar halaman web hanya dapat ditampilkan di dalam `frame` atau `iframe` yang berasal dari origin yang sama. Hal ini membantu mengurangi risiko serangan clickjacking.

Header:

```text
X-Content-Type-Options: nosniff
```

memerintahkan browser agar tidak mencoba menebak tipe konten yang berbeda dari nilai `Content-Type` yang diberikan oleh server.

Header:

```text
Referrer-Policy: strict-origin-when-cross-origin
```

mengatur informasi referrer yang dikirimkan browser ketika pengguna berpindah dari satu halaman ke halaman lainnya.

Sedangkan:

```text
Permissions-Policy: geolocation=(), microphone=(), camera=()
```

menonaktifkan akses terhadap geolocation, microphone, dan camera melalui browser. Jika aplikasi memang membutuhkan salah satu fitur tersebut, konfigurasi harus disesuaikan.

Perlu diperhatikan bahwa security header sebaiknya disesuaikan dengan kebutuhan masing-masing aplikasi. Konfigurasi global dapat menyebabkan masalah apabila suatu aplikasi membutuhkan fitur yang dinonaktifkan.

### Mengatur Timeout

Untuk membatasi koneksi yang terlalu lama atau client yang mengirimkan request secara sangat lambat, buat konfigurasi timeout:

```bash
nano /etc/nginx/conf.d/timeout.conf
```

Masukkan:

```nginx
client_header_timeout 30s;
client_body_timeout 30s;
send_timeout 30s;
keepalive_timeout 30s;
```

`client_header_timeout` menentukan batas waktu Nginx menunggu header request dari client.

`client_body_timeout` menentukan batas waktu antara operasi pembacaan body request dari client.

`send_timeout` menentukan batas waktu antara operasi pengiriman data kepada client.

Sedangkan `keepalive_timeout` menentukan berapa lama koneksi HTTP keep-alive dapat tetap terbuka dalam kondisi idle.

Nilai timeout tersebut dapat disesuaikan dengan karakteristik aplikasi. Aplikasi yang menerima upload file berukuran besar atau koneksi lambat mungkin membutuhkan nilai yang lebih tinggi.

### Memblokir File Sensitif

Aplikasi terkadang memiliki file konfigurasi, backup, atau file tersembunyi yang tidak seharusnya dapat diakses melalui web.

Untuk membuat konfigurasi yang dapat digunakan kembali, buat file snippet:

```bash
nano /etc/nginx/snippets/deny-files.conf
```

Masukkan:

```nginx
# Hidden files seperti .git dan .env
# Direktori .well-known dikecualikan untuk kebutuhan seperti ACME challenge
location ~ /\.(?!well-known).* {
    deny all;
    access_log off;
    log_not_found off;
}

# Backup dan temporary files
location ~* \.(bak|old|orig|save|swp|tmp|dist)$ {
    deny all;
    access_log off;
    log_not_found off;
}

# File konfigurasi dan database
location ~* \.(env|ini|conf|sql|sqlite|db)$ {
    deny all;
    access_log off;
    log_not_found off;
}
```

Kemudian tambahkan snippet tersebut ke virtual host:

```bash
nano /etc/nginx/sites-available/website.conf
```

Tambahkan:

```nginx
include snippets/deny-files.conf;
```

sehingga konfigurasi virtual host menjadi kurang lebih:

```nginx
server {
    listen 8080;
    server_name example.com;

    root /var/apps/website/htdocs;
    index index.php index.html;

    charset utf-8;

    access_log /var/log/nginx/website.access.log;
    error_log  /var/log/nginx/website.error.log warn;

    client_max_body_size 100M;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        try_files $uri =404;

        include fastcgi_params;

        fastcgi_param SCRIPT_FILENAME /var/www/html$fastcgi_script_name;
        fastcgi_param DOCUMENT_ROOT /var/www/html;

        fastcgi_pass unix:/run/php/website.sock;
    }

    include snippets/deny-files.conf;
}
```

Dengan konfigurasi tersebut, request terhadap file sensitif akan ditolak oleh Nginx sebelum file tersebut dapat dikirimkan kepada client.

### Melindungi Direktori Upload

Pada arsitektur yang kita gunakan, direktori upload sengaja dipisahkan dari source code aplikasi.

Pada host, file upload disimpan di:

```text
/var/apps/website/data/uploads
```

Sedangkan di dalam container PHP-FPM direktori tersebut tersedia sebagai:

```text
/var/www/uploads
```

Source code utama:

```text
/var/apps/website/htdocs
```

bersifat read-only bagi container PHP-FPM, tetapi direktori upload tetap harus memiliki akses tulis agar aplikasi dapat menyimpan file.

Apabila aplikasi memiliki celah arbitrary file upload, penyerang mungkin mencoba mengunggah file seperti:

```text
shell.php
shell.phtml
shell.phar
```

Oleh karena itu, file script yang berada di direktori upload tidak boleh diteruskan ke PHP-FPM.

Jika file upload memang perlu diakses langsung melalui URL `/uploads/`, tambahkan konfigurasi berikut **sebelum blok PHP umum**:

```nginx
location ~* ^/uploads/.*\.(php|phar|phtml|pht|cgi|pl|py|sh)$ {
    deny all;
}

location ^~ /uploads/ {
    alias /var/apps/website/data/uploads/;
    autoindex off;
    try_files $uri =404;
}
```

Konfigurasi virtual host menjadi:

```nginx
server {
    listen 8080;
    server_name example.com;

    root /var/apps/website/htdocs;
    index index.php index.html;

    charset utf-8;

    access_log /var/log/nginx/website.access.log;
    error_log  /var/log/nginx/website.error.log warn;

    client_max_body_size 100M;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~* ^/uploads/.*\.(php|phar|phtml|pht|cgi|pl|py|sh)$ {
        deny all;
    }

    location ^~ /uploads/ {
        alias /var/apps/website/data/uploads/;
        autoindex off;
        try_files $uri =404;
    }

    location ~ \.php$ {
        try_files $uri =404;

        include fastcgi_params;

        fastcgi_param SCRIPT_FILENAME /var/www/html$fastcgi_script_name;
        fastcgi_param DOCUMENT_ROOT /var/www/html;

        fastcgi_pass unix:/run/php/website.sock;
    }

    include snippets/deny-files.conf;
}
```

Namun terdapat satu hal penting mengenai penggunaan:

```nginx
location ^~ /uploads/
```

Modifier `^~` menyebabkan Nginx tidak melanjutkan pencarian ke regular expression location setelah prefix tersebut cocok. Oleh karena itu, blok regex terpisah yang berada di atasnya tidak dapat dijadikan satu-satunya perlindungan terhadap ekstensi script apabila request sudah dipilih oleh blok `^~`.

Untuk konfigurasi yang lebih sederhana dan tidak bergantung hanya pada pencocokan ekstensi, direktori upload publik sebaiknya diperlakukan sebagai lokasi file statis dan tidak pernah memiliki jalur ke PHP-FPM. Dengan konfigurasi `^~ /uploads/`, seluruh request di bawah `/uploads/` akan ditangani sebagai file statis dan tidak akan masuk ke blok PHP umum.

Dengan demikian, meskipun terdapat file:

```text
/uploads/shell.php
```

file tersebut tidak akan dieksekusi oleh PHP-FPM.

Untuk keamanan yang lebih ketat, aplikasi tetap harus melakukan validasi file upload berdasarkan extension, MIME type, ukuran file, dan kebutuhan bisnis aplikasi. Nama file juga sebaiknya dibuat ulang oleh aplikasi dan tidak menggunakan nama file asli dari pengguna.

### Menguji Proteksi Source Code

Sebelumnya, source code telah di-mount sebagai read-only:

```yaml
- /var/apps/website/htdocs:/var/www/html:ro
```

Kita dapat melakukan pengujian kembali:

```bash
docker exec website-php touch /var/www/html/webshell.php
```

Hasil yang diharapkan:

```text
touch: cannot touch '/var/www/html/webshell.php': Read-only file system
```

Hal ini menunjukkan bahwa proses di dalam container tidak dapat membuat atau mengganti source code aplikasi.

### Menguji Proteksi Direktori Upload

Buat file PHP pengujian pada direktori upload dari host:

```bash
echo '<?php echo "TEST"; ?>' > /var/apps/website/data/uploads/test.php
```

Kemudian akses menggunakan:

```bash
curl -i -H "Host: example.com" \
    http://127.0.0.1:8080/uploads/test.php
```

Yang paling penting dalam pengujian ini adalah memastikan isi PHP:

```text
TEST
```

tidak pernah muncul sebagai hasil eksekusi PHP-FPM.

Setelah pengujian selesai, hapus file tersebut:

```bash
rm -f /var/apps/website/data/uploads/test.php
```

Untuk memastikan file upload normal tetap dapat diakses, buat file teks:

```bash
echo "UPLOAD TEST" > /var/apps/website/data/uploads/test.txt
```

Kemudian:

```bash
curl -H "Host: example.com" \
    http://127.0.0.1:8080/uploads/test.txt
```

Setelah selesai:

```bash
rm -f /var/apps/website/data/uploads/test.txt
```

### Melakukan Validasi Konfigurasi

Setelah seluruh konfigurasi hardening selesai, selalu lakukan pengujian konfigurasi:

```bash
nginx -t
```

Jika hasilnya:

```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

reload Nginx:

```bash
systemctl reload nginx
```

Kemudian periksa status Nginx:

```bash
systemctl status nginx
```

Dengan konfigurasi ini, platform memiliki beberapa lapisan perlindungan:

```text
Internet / Client
        │
        ▼
Nginx
├── Security Headers
├── Timeout
├── Hidden Files Protection
├── Backup / Configuration File Protection
└── Upload Directory Isolation
        │
        ▼
PHP-FPM Container
├── Read-only Root Filesystem
├── Read-only Source Code
└── Writable Runtime Directories
        │
        ▼
MariaDB
└── Unix Socket
```

Konfigurasi tersebut membantu mengurangi dampak apabila aplikasi memiliki celah keamanan. Source code tidak dapat dimodifikasi dari dalam container, file upload dipisahkan dari source code, dan direktori upload tidak memiliki jalur eksekusi ke PHP-FPM.

Hardening Nginx tetap merupakan salah satu lapisan pertahanan. Keamanan utama untuk mencegah arbitrary file upload, local file inclusion, command injection, dan remote code execution tetap harus diterapkan pada level aplikasi, dependency, PHP, sistem operasi, serta proses deployment dan monitoring.
