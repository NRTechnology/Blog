---
title: "Membangun Open Journal Systems (OJS) 3.4 Menggunakan Nginx Reverse Proxy dan Docker PHP-FPM"
date: 2026-08-02
draft: false
description: "Pengalaman membangun Open Journal Systems (OJS) 3.4 menggunakan Nginx Reverse Proxy, Docker PHP-FPM, dan MariaDB dengan pendekatan yang aman dan mudah dipelihara."

tags:
  - OJS
  - Docker
  - Nginx
  - PHP-FPM
  - MariaDB
  - Reverse Proxy
  - Linux
categories:
  - Linux
  - DevOps
  - Open Journal Systems

series:
  - "Membangun Open Journal Systems (OJS) 3.4"
weight: 1

author: "NR Technology"
cover:
  image: "../ojs-cover.png"
  alt: "Open Journal Systems (OJS) 3.4"
  caption: "Seri Membangun Open Journal Systems (OJS) 3.4"
---

# Pendahuluan

Open Journal Systems (OJS) merupakan aplikasi open source yang banyak digunakan oleh perguruan tinggi maupun lembaga penelitian untuk mengelola publikasi jurnal ilmiah.

Pada artikel ini saya membangun OJS menggunakan arsitektur yang memisahkan setiap komponen sehingga lebih mudah dikelola, diamankan, dan di-backup.

---

# Seri Artikel

Artikel ini merupakan bagian pertama dari seri **Membangun Open Journal Systems (OJS) 3.4 Menggunakan Nginx Reverse Proxy dan Docker PHP-FPM**.

Urutan artikel pada seri ini adalah sebagai berikut.

1. **Membangun Open Journal Systems (OJS) 3.4 Menggunakan Nginx Reverse Proxy dan Docker PHP-FPM** *(Artikel ini)*
2. [Instalasi Docker PHP-FPM untuk OJS 3.4](../instalasi-docker-php-fpm-untuk-ojs-34/)
3. [Konfigurasi PHP-FPM untuk Open Journal Systems (OJS) 3.4](../konfigurasi-php-fpm-untuk-open-journal-systems-ojs-34/)
4. [Konfigurasi Nginx untuk Open Journal Systems (OJS) 3.4](../konfigurasi-nginx-untuk-open-journal-systems-ojs-34/)
5. [Instalasi Open Journal Systems (OJS) 3.4](../instalasi-open-journal-systems-ojs-34/)
6. [Hardening Open Journal Systems (OJS) 3.4](../hardening-open-journal-systems-ojs-34/)
7. [Backup dan Disaster Recovery Open Journal Systems (OJS) 3.4](../backup-dan-disaster-recovery-open-journal-systems-ojs-34/)
8. [Migrasi Open Journal Systems (OJS) 3.4 ke Server Baru](../migrasi-open-journal-systems-ojs-34-ke-server-baru/)
9. [Upgrade Open Journal Systems (OJS) 3.4](../upgrade-open-journal-systems-ojs-34/)

Artikel-artikel tersebut disusun secara berurutan, mulai dari perencanaan arsitektur, instalasi, konfigurasi, hardening, backup, migrasi, hingga proses upgrade pada lingkungan produksi.

---

# Arsitektur

```
                Internet
                    │
                 HTTPS
                    │
        Reverse Proxy (Nginx)
                    │
          HTTP / HTTPS Port 8080
                    │
            Nginx Application
                    │
          Unix Socket PHP-FPM
                    │
            Docker PHP 8.3
                    │
               MariaDB Server
```

Komponen yang digunakan:

- Ubuntu Server
- Nginx
- Docker
- PHP-FPM 8.3
- MariaDB
- Open Journal Systems 3.4

---

# Struktur Direktori

```
/var/apps/ojs
├── htdocs
├── data
│   └── ojsdata
├── backup
└── logs
```

Docker Compose ditempatkan pada:

```
/opt/docker/apps/ojs
```

---

# Konfigurasi Docker

Container hanya menjalankan PHP-FPM.

Keuntungan pendekatan ini:

- lebih ringan
- mudah upgrade PHP
- source code berada di host
- backup lebih mudah
- socket PHP lebih cepat dibanding TCP

Beberapa hardening yang digunakan:

- read_only filesystem
- no-new-privileges
- tmpfs
- Unix Socket PHP-FPM
- source code hanya mount read only

---

# Konfigurasi Nginx

Nginx hanya memiliki satu front controller yaitu:

```
index.php
```

Seluruh request diarahkan menuju index.php.

```
location / {
    try_files $uri $uri/ /index.php?$args;
}
```

Eksekusi PHP dibatasi hanya pada file `index.php`.

```
location ~ ^/index\.php($|/) {

    fastcgi_split_path_info ^(.+?\.php)(/.*)$;

    include fastcgi_params;

    fastcgi_param SCRIPT_FILENAME /var/www/html$fastcgi_script_name;

    fastcgi_pass unix:/run/php/ojs.sock;
}
```

Semua file PHP lainnya ditolak.

```
location ~ \.php$ {
    return 404;
}
```

Pendekatan ini mengurangi kemungkinan web shell dieksekusi apabila penyerang berhasil mengunggah file PHP.

---

# Direktori public dan ojsdata

OJS menggunakan dua direktori penting.

## public

Berisi file yang memang boleh diakses publik.

Contohnya:

- logo
- favicon
- banner
- gambar homepage

## ojsdata

Berisi seluruh file utama jurnal.

Misalnya:

- PDF artikel
- file submission
- revisi reviewer
- supplementary file
- galley

Direktori ini **tidak boleh dapat diakses langsung melalui web server**.

---

# Hardening Nginx

Beberapa konfigurasi keamanan yang diterapkan:

- server_tokens off
- X-Frame-Options
- X-Content-Type-Options
- Referrer-Policy
- menolak file tersembunyi
- menolak config.inc.php
- menolak README
- menolak INSTALL
- menolak UPGRADE

PHP pada direktori upload juga diblokir.

```
location ~* ^/(public|files)/.*\.php$ {
    return 404;
}
```

---

# Konfigurasi Database

MariaDB dijalankan langsung pada host.

Container PHP mengakses MariaDB menggunakan IP host.

Contoh:

```
host = 172.18.0.1
```

Pendekatan ini mempermudah backup dan administrasi database.

---

# Backup

Backup OJS tidak hanya database.

Minimal terdapat tiga komponen.

```
Database
public/
ojsdata/
```

Tanpa direktori `ojsdata`, seluruh artikel dan PDF akan hilang walaupun database masih tersedia.

---

# Migrasi dari Server Lama

Pada implementasi ini dilakukan migrasi dari server lama yang mengalami kompromi keamanan.

Langkah yang dilakukan:

- menggunakan source code OJS baru
- menggunakan database lama
- melakukan sinkronisasi direktori public
- melakukan sinkronisasi direktori ojsdata
- membersihkan cache
- melakukan verifikasi seluruh artikel

Dengan pendekatan ini source code tetap bersih sementara seluruh data jurnal berhasil dipertahankan.

---

# Kesimpulan

Memisahkan Nginx, PHP-FPM, MariaDB, dan data aplikasi memberikan beberapa keuntungan.

- lebih aman
- mudah melakukan backup
- mudah melakukan upgrade PHP
- mudah melakukan migrasi server
- source code tetap bersih
- lebih mudah melakukan hardening

Pendekatan ini juga sangat cocok diterapkan pada server yang mengelola beberapa aplikasi PHP secara bersamaan karena setiap aplikasi dapat memiliki container PHP-FPM sendiri tanpa harus berbagi runtime dengan aplikasi lain.

---
