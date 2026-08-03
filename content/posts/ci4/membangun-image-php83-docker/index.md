---
title: "Membangun Image PHP 8.3 Docker untuk Produksi"
slug: "membangun-image-php83-docker"
date: 2026-08-03T17:00:00+07:00
lastmod: 2026-08-03T17:00:00+07:00
draft: false

description: "Panduan membangun image Docker PHP 8.3 untuk lingkungan produksi yang ringan, aman, dan siap digunakan bersama Nginx dan PHP-FPM."

author: "NR Technology"

series:
  - Docker PHP Production
weight: 2

tags:
  - Docker
  - PHP
  - PHP83
  - PHP-FPM
  - DevOps
  - Linux

categories:
  - Docker
  - PHP

keywords:
  - php docker
  - php 8.3 docker
  - docker php production
  - php-fpm docker
  - docker image php

toc: true
showReadingTime: true
showCodeCopyButtons: true
showWordCount: true
---

# Membangun Image PHP 8.3 Docker untuk Produksi

Pada artikel sebelumnya kita telah membahas arsitektur deployment CodeIgniter 4 menggunakan Docker PHP-FPM.

Pada artikel ini kita akan membangun image PHP 8.3 yang nantinya akan digunakan oleh seluruh aplikasi sehingga tidak perlu lagi mengunduh image PHP dari internet setiap kali melakukan deployment.

---

# Mengapa Membuat Image Sendiri?

Sebagian administrator langsung menggunakan image resmi dari Docker Hub.

Misalnya:

```yaml
image: php:8.3-fpm
```

Pendekatan tersebut memang mudah, tetapi memiliki beberapa kelemahan.

- Extension PHP harus dipasang setiap deployment.
- Build menjadi lebih lama.
- Konfigurasi antar aplikasi menjadi tidak konsisten.
- Sulit melakukan standarisasi.

Karena itu saya lebih memilih membuat image sendiri.

```
local/php:8.3
```

Seluruh aplikasi nantinya cukup menggunakan image tersebut.

---

# Keuntungan Menggunakan Image Lokal

Beberapa keuntungan menggunakan image sendiri antara lain:

- Build hanya dilakukan satu kali.
- Semua aplikasi menggunakan extension yang sama.
- Konfigurasi PHP seragam.
- Deployment lebih cepat.
- Mudah diperbarui.

---

# Struktur Direktori

Saya menggunakan struktur berikut.

```text
/opt/docker/
└── images/
    └── php8.3/
        ├── Dockerfile
        ├── php.ini
        └── docker-php-ext.ini
```

---

# Membuat Dockerfile

Buat file:

```text
/opt/docker/images/php8.3/Dockerfile
```

Isi Dockerfile sebagai berikut.

```dockerfile
FROM php:8.3-fpm-bookworm

LABEL maintainer="NR Technology"

RUN apt-get update && apt-get install -y \
    git \
    unzip \
    zip \
    curl \
    libzip-dev \
    libicu-dev \
    libpng-dev \
    libjpeg62-turbo-dev \
    libfreetype6-dev \
    libonig-dev \
    libxml2-dev \
    libsqlite3-dev \
    libpq-dev \
    libcurl4-openssl-dev \
    libxslt1-dev \
    libgmp-dev \
    libldap2-dev \
    libkrb5-dev \
    libmagickwand-dev \
    libssl-dev \
    && rm -rf /var/lib/apt/lists/*

RUN docker-php-ext-configure gd \
    --with-freetype \
    --with-jpeg

RUN docker-php-ext-install \
    bcmath \
    exif \
    gd \
    intl \
    mysqli \
    opcache \
    pcntl \
    pdo_mysql \
    sockets \
    zip

COPY --from=composer:2 /usr/bin/composer /usr/local/bin/composer

WORKDIR /var/www/html
```

---

# Extension yang Dipasang

Image di atas telah memiliki extension:

| Extension | Fungsi |
|------------|--------|
| mysqli | MySQL |
| pdo_mysql | PDO MySQL |
| intl | Internationalization |
| gd | Image Processing |
| zip | ZIP Archive |
| exif | Image Metadata |
| sockets | Socket |
| pcntl | Process Control |
| bcmath | Big Number |
| opcache | PHP Accelerator |

Extension tersebut sudah mencukupi untuk sebagian besar aplikasi modern seperti:

- CodeIgniter 4
- Laravel
- WordPress
- OJS
- Moodle

---

# Menambahkan Composer

Daripada menginstal Composer secara manual, kita dapat langsung mengambil binary dari image Composer.

```dockerfile
COPY --from=composer:2 /usr/bin/composer /usr/local/bin/composer
```

Cara ini lebih sederhana dan menghasilkan image yang lebih kecil.

---

# Konfigurasi php.ini

File php.ini disimpan pada host sehingga seluruh container menggunakan konfigurasi yang sama.

Contohnya:

```text
/opt/docker/images/php8.3/php.ini
```

Kemudian di-mount ke dalam container.

```yaml
- /opt/docker/images/php8.3/php.ini:/usr/local/etc/php/conf.d/99-custom.ini:ro
```

Dengan pendekatan ini kita tidak perlu melakukan build ulang image hanya untuk mengubah konfigurasi PHP.

---

# Build Image

Masuk ke direktori image.

```bash
cd /opt/docker/images/php8.3
```

Kemudian build image.

```bash
docker build -t local/php:8.3 .
```

Apabila berhasil akan muncul image baru.

```bash
docker images
```

Contoh hasilnya.

```text
REPOSITORY     TAG     IMAGE ID
local/php      8.3     xxxxxxxxxxxx
```

---

# Menguji Image

Pastikan image dapat berjalan.

```bash
docker run --rm local/php:8.3 php -v
```

Contoh hasil.

```text
PHP 8.3.x (cli)
```

Kemudian pastikan Composer tersedia.

```bash
docker run --rm local/php:8.3 composer --version
```

Lalu periksa user PHP.

```bash
docker run --rm local/php:8.3 id www-data
```

Hasil yang diharapkan.

```text
uid=33(www-data)
gid=33(www-data)
```

---

# Mengapa Menggunakan www-data?

PHP-FPM secara default menggunakan user:

```
www-data
```

Dengan mengetahui UID dan GID, kita dapat memberikan permission yang benar pada direktori writable.

Misalnya:

```bash
chown -R 33:33 /var/apps/myapp/data/writable
```

Pendekatan ini jauh lebih baik dibandingkan memberikan permission 777.

---

# Menggunakan Image pada Docker Compose

Setelah image selesai dibangun, seluruh aplikasi cukup menggunakan:

```yaml
image: local/php:8.3
```

Dengan demikian deployment menjadi jauh lebih cepat karena Docker tidak lagi perlu membangun image setiap kali aplikasi baru dibuat.

---

# Best Practice

Beberapa praktik yang saya gunakan pada server produksi:

- Menggunakan image lokal.
- Menyimpan php.ini di host.
- Menjalankan PHP-FPM menggunakan Unix Socket.
- Source code dibuat read-only.
- Direktori writable dipisahkan.
- Setiap aplikasi memiliki socket PHP sendiri.
- Menggunakan health check pada Docker Compose.
- Mengaktifkan log rotation.

---

# Kesimpulan

Membangun image PHP sendiri memberikan banyak keuntungan untuk lingkungan produksi. Selain membuat deployment lebih cepat, pendekatan ini juga menghasilkan konfigurasi yang konsisten pada seluruh aplikasi.

Pada artikel berikutnya kita akan membahas bagaimana membangun **Docker Compose** yang aman untuk menjalankan CodeIgniter 4 menggunakan image **local/php:8.3** yang telah dibuat pada artikel ini.
````
