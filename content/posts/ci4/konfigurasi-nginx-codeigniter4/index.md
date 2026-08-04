---
title: "Konfigurasi Nginx untuk CodeIgniter 4 pada Lingkungan Produksi"
slug: "konfigurasi-nginx-codeigniter4"
date: 2026-08-03T20:00:00+07:00
lastmod: 2026-08-03T20:00:00+07:00
draft: false

description: "Panduan lengkap mengkonfigurasi Nginx sebagai reverse proxy untuk CodeIgniter 4 menggunakan PHP-FPM Docker dan Unix Socket."

author: "NR Technology"

series:
  - Membangun Platform Produksi CodeIgniter 4
weight: 5

tags:
  - Nginx
  - CodeIgniter4
  - PHP-FPM
  - Docker
  - Linux
  - Reverse Proxy

categories:
  - Nginx
  - CodeIgniter
  - Docker

keywords:
  - nginx codeigniter
  - nginx php-fpm
  - codeigniter nginx
  - php-fpm socket
  - nginx production

cover:
  image: "ci4-cover.png"
  alt: "Deploy Aplikasi CodeIgniter 4 Menggunakan PHP-FPM Docker dan Nginx Host"
  caption: "CodeIgniter 4 + PHP-FPM Docker + Nginx Host"

toc: true
showReadingTime: true
showCodeCopyButtons: true
showWordCount: true
---

# Konfigurasi Nginx untuk CodeIgniter 4 pada Lingkungan Produksi

Pada artikel sebelumnya kita telah membangun image PHP 8.3, membuat container PHP-FPM, dan mengkonfigurasi pool PHP-FPM.

Langkah berikutnya adalah menghubungkan aplikasi CodeIgniter 4 dengan **Nginx** yang berjalan langsung pada host Linux.

Pendekatan ini memiliki beberapa keuntungan.

- Performa tinggi.
- Konsumsi memori rendah.
- Tidak perlu menjalankan container Nginx untuk setiap aplikasi.
- Konfigurasi lebih sederhana.
- Mudah dipadukan dengan reverse proxy maupun WAF.

---

# Arsitektur

```text
                 Internet
                     │
                     ▼
            Nginx (Host Linux)
                     │
       Unix Socket (/run/php/*.sock)
                     │
                     ▼
         Docker PHP-FPM Container
                     │
                     ▼
            CodeIgniter 4 Application
```

Pada arsitektur ini:

- Nginx menerima seluruh koneksi HTTP dan HTTPS.
- PHP-FPM berjalan di dalam Docker.
- Komunikasi dilakukan menggunakan Unix Socket.
- Tidak ada port PHP-FPM yang dibuka ke jaringan.

---

# Mengapa Menggunakan Nginx di Host?

Banyak administrator menjalankan Nginx di dalam Docker.

Saya lebih memilih menjalankan Nginx langsung pada host karena beberapa alasan.

- Lebih mudah dikelola.
- Konsumsi resource lebih kecil.
- Integrasi SSL lebih sederhana.
- Reverse Proxy lebih mudah.
- Integrasi Web Application Firewall lebih mudah.
- Tidak perlu membuka port tambahan.

---

# Struktur Direktori

```text
/etc/nginx/
├── nginx.conf
├── sites-available/
│   └── gkb.conf
└── sites-enabled/
    └── gkb.conf
```

Source aplikasi.

```text
/var/apps/myapp/
├── htdocs/
│   └── public/
└── data/
```

---

# Virtual Host

Buat file berikut.

```text
/etc/nginx/sites-available/gkb.conf
```

---

# Konfigurasi Lengkap

```nginx
server {

    listen 8080;

    server_name gkb.brebeskab.go.id;

    root /var/apps/myapp/htdocs/public;

    index index.php;

    charset utf-8;

    server_tokens off;

    access_log /var/log/nginx/myapp.access.log;
    error_log  /var/log/nginx/myapp.error.log warn;

    client_max_body_size 100M;

    client_body_timeout 30s;
    client_header_timeout 30s;
    keepalive_timeout 30s;
    send_timeout 30s;

    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    #############################################################
    # Front Controller
    #############################################################

    location / {

        try_files $uri $uri/ /index.php?$query_string;

    }

    #############################################################
    # PHP
    #############################################################

    location = /index.php {

        include fastcgi_params;

        fastcgi_param SCRIPT_FILENAME /var/www/html/public/index.php;
        fastcgi_param DOCUMENT_ROOT /var/www/html/public;

        fastcgi_index index.php;

        fastcgi_intercept_errors on;

        fastcgi_read_timeout 300;

        fastcgi_pass unix:/run/php/myapp.sock;

    }

    #############################################################
    # Block PHP selain index.php
    #############################################################

    location ~ \.php$ {

        return 404;

    }

    #############################################################
    # Hidden File
    #############################################################

    location ~ /\.(?!well-known).* {

        deny all;

    }

}
```

---

# Document Root

Perhatikan bagian berikut.

```nginx
root /var/apps/myapp/htdocs/public;
```

Document Root **harus** mengarah ke direktori `public`.

Jangan pernah menggunakan:

```text
htdocs/
```

karena dapat mengekspos source code aplikasi.

---

# Front Controller

CodeIgniter menggunakan pola Front Controller.

Semua request akan diteruskan ke:

```text
index.php
```

melalui konfigurasi:

```nginx
try_files $uri $uri/ /index.php?$query_string;
```

---

# Mengapa Hanya index.php?

Pada banyak tutorial masih ditemukan konfigurasi berikut.

```nginx
location ~ \.php$
```

Konfigurasi tersebut mengizinkan seluruh file PHP dieksekusi.

Misalnya.

```text
info.php
shell.php
test.php
upload.php
```

Pada server produksi saya lebih memilih hanya mengizinkan:

```text
index.php
```

melalui:

```nginx
location = /index.php
```

Kemudian seluruh file PHP lainnya diblokir.

```nginx
location ~ \.php$ {

    return 404;

}
```

Pendekatan ini jauh lebih aman.

---

# PHP-FPM Menggunakan Unix Socket

Nginx meneruskan request ke PHP-FPM.

```nginx
fastcgi_pass unix:/run/php/myapp.sock;
```

Socket tersebut dibuat oleh PHP-FPM.

```text
/run/php/myapp.sock
```

Keuntungan Unix Socket.

- Lebih cepat.
- Tidak membuka port.
- Overhead kecil.
- Lebih aman.

---

# FastCGI Parameter

```nginx
fastcgi_param SCRIPT_FILENAME /var/www/html/public/index.php;
```

Perhatikan bahwa path tersebut adalah path **di dalam container**, bukan path host.

Container melihat source code pada:

```text
/var/www/html
```

sedangkan host melihat source code pada:

```text
/var/apps/myapp/htdocs
```

---

# Timeout

```nginx
fastcgi_read_timeout 300;
```

Timeout lima menit cukup aman untuk proses upload maupun laporan yang membutuhkan waktu lebih lama.

---

# Security Header

Beberapa header keamanan yang saya aktifkan.

```nginx
add_header X-Frame-Options "SAMEORIGIN";

add_header X-Content-Type-Options "nosniff";

add_header Referrer-Policy "strict-origin-when-cross-origin";
```

Header tersebut membantu mengurangi beberapa risiko keamanan pada browser.

---

# Upload File

Ukuran upload dibatasi.

```nginx
client_max_body_size 100M;
```

Nilai ini dapat disesuaikan sesuai kebutuhan aplikasi.

---

# Mengaktifkan Site

Buat symbolic link.

```bash
sudo ln -s \
/etc/nginx/sites-available/gkb.conf \
/etc/nginx/sites-enabled/gkb.conf
```

---

# Menguji Konfigurasi

```bash
sudo nginx -t
```

Apabila berhasil.

```text
syntax is ok

test is successful
```

---

# Reload Nginx

```bash
sudo systemctl reload nginx
```

---

# Memastikan Socket

Pastikan socket PHP tersedia.

```bash
ls -l /run/php
```

Contoh.

```text
myapp.sock
```

---

# Pengujian

Akses aplikasi.

```text
http://gkb.brebeskab.go.id:8080
```

Apabila seluruh konfigurasi benar maka halaman CodeIgniter akan tampil.

---

# Best Practice

Pada seluruh server produksi saya menggunakan prinsip berikut.

- Nginx berjalan di host.
- PHP-FPM berjalan di Docker.
- MariaDB berjalan di host.
- Unix Socket.
- Document Root hanya `public`.
- Hanya `index.php` yang boleh dieksekusi.
- Hidden file diblokir.
- Source code dibuat read-only.
- Writable dipisahkan.
- Setiap aplikasi memiliki socket sendiri.

Pendekatan tersebut menghasilkan lingkungan yang lebih aman dan mudah dipelihara.

---

# Kesimpulan

Konfigurasi Nginx merupakan salah satu bagian terpenting dalam membangun server produksi berbasis CodeIgniter 4.

Dengan menggunakan Document Root yang benar, Unix Socket, Front Controller, dan hanya mengizinkan eksekusi `index.php`, kita dapat mengurangi permukaan serangan sekaligus memperoleh performa yang tinggi.

Pada artikel berikutnya kita akan membahas **Hardening Nginx untuk Aplikasi PHP**, termasuk teknik membatasi akses terhadap file sensitif, mencegah eksekusi file berbahaya, dan menerapkan konfigurasi keamanan tambahan untuk lingkungan produksi.

---
{{< saweria >}}