````markdown
---
title: "Studi Kasus: Deploy CodeIgniter 4 ke Lingkungan Produksi Menggunakan Nginx, Docker PHP-FPM, dan MariaDB"
slug: "studi-kasus-deploy-codeigniter4-produksi"
date: 2026-08-04T10:00:00+07:00
lastmod: 2026-08-04T10:00:00+07:00
draft: false

description: "Studi kasus deployment aplikasi CodeIgniter 4 pada lingkungan produksi menggunakan Nginx di host, Docker PHP-FPM, dan MariaDB di host dengan pendekatan yang aman dan mudah dipelihara."

author: "NR Technology"

series:
  - Docker PHP Production
weight: 11

tags:
  - CodeIgniter4
  - Docker
  - PHP-FPM
  - Nginx
  - MariaDB
  - Linux
  - DevOps

categories:
  - Docker
  - CodeIgniter
  - Studi Kasus

keywords:
  - deploy codeigniter production
  - docker php-fpm
  - nginx php
  - codeigniter docker
  - php production

toc: true
showReadingTime: true
showWordCount: true
showCodeCopyButtons: true
---

# Studi Kasus: Deploy CodeIgniter 4 ke Lingkungan Produksi Menggunakan Nginx, Docker PHP-FPM, dan MariaDB

## Pendahuluan

Pada sepuluh artikel sebelumnya kita telah membahas setiap komponen yang dibutuhkan untuk membangun lingkungan produksi berbasis CodeIgniter 4, mulai dari pembuatan image Docker, konfigurasi PHP-FPM, Nginx, MariaDB, hingga hardening dan monitoring.

Pada artikel penutup ini seluruh komponen tersebut akan digabungkan menjadi satu alur deployment yang utuh sehingga dapat digunakan sebagai referensi ketika membangun server baru.

Arsitektur yang digunakan tidak bergantung pada penyedia cloud tertentu maupun aplikasi tertentu sehingga dapat diterapkan pada server fisik, virtual machine, VPS, maupun lingkungan private cloud.

---

# Arsitektur

```text
                      Internet
                          │
                          ▼
                 Nginx (Host Linux)
                          │
                Unix Socket PHP-FPM
                          │
                          ▼
            Docker Container (PHP 8.3)
                          │
                TCP Port 3306 (Host)
                          │
                          ▼
               MariaDB (Host Linux)
```

Karakteristik arsitektur ini:

- Nginx berjalan langsung pada host.
- PHP-FPM berjalan di Docker.
- MariaDB berjalan di host.
- Source code bersifat read-only.
- Direktori writable dipisahkan.
- Setiap aplikasi memiliki socket PHP-FPM sendiri.

---

# Struktur Direktori

```text
/opt/docker/apps/
└── myapp/
    ├── docker-compose.yml
    └── zz-custom.conf

/opt/docker/images/
└── php8.3/
    ├── Dockerfile
    └── php.ini

/var/apps/
└── myapp/
    ├── backup/
    ├── data/
    │   └── writable/
    ├── htdocs/
    └── logs/
```

Dengan struktur tersebut seluruh konfigurasi aplikasi tersusun secara konsisten sehingga mudah dipelihara.

---

# Langkah 1 — Menyiapkan Sistem Operasi

Pastikan sistem operasi telah diperbarui.

```bash
sudo apt update

sudo apt upgrade
```

Pastikan layanan berikut telah tersedia.

- Docker Engine
- Docker Compose
- Nginx
- MariaDB

---

# Langkah 2 — Membangun Image PHP

Masuk ke direktori image.

```bash
cd /opt/docker/images/php8.3
```

Bangun image.

```bash
docker build -t local/php:8.3 .
```

Verifikasi.

```bash
docker images
```

---

# Langkah 3 — Menyiapkan Struktur Direktori

```bash
mkdir -p /var/apps/myapp

mkdir -p /var/apps/myapp/htdocs

mkdir -p /var/apps/myapp/data/writable

mkdir -p /var/apps/myapp/logs

mkdir -p /var/apps/myapp/backup

mkdir -p /opt/docker/apps/myapp
```

---

# Langkah 4 — Menyalin Source Code

Salin seluruh aplikasi.

```bash
cp -a /path/source/. \
      /var/apps/myapp/htdocs/
```

Selanjutnya pindahkan isi direktori writable.

```bash
cp -a \
/var/apps/myapp/htdocs/writable/. \
/var/apps/myapp/data/writable/
```

Kosongkan isi direktori writable pada source.

```bash
find /var/apps/myapp/htdocs/writable \
    -mindepth 1 \
    -exec rm -rf {} +
```

---

# Langkah 5 — Mengatur Permission

```bash
chown -R 33:33 \
/var/apps/myapp/data/writable

chmod -R 775 \
/var/apps/myapp/data/writable
```

Pastikan source code tidak memiliki hak tulis yang tidak diperlukan.

---

# Langkah 6 — Menyiapkan Database

Masuk ke MariaDB.

```bash
mysql -u root -p
```

Buat database.

```sql
CREATE DATABASE db_myapp
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;
```

Buat user aplikasi.

```sql
CREATE USER 'myapp'@'%'
IDENTIFIED BY 'your_secure_password';

GRANT ALL PRIVILEGES
ON db_myapp.*
TO 'myapp'@'%';

FLUSH PRIVILEGES;
```

Pada lingkungan produksi, gunakan pembatasan host yang sesuai daripada memberikan akses dari semua alamat (`%`).

---

# Langkah 7 — Mengatur File `.env`

Contoh konfigurasi.

```ini
database.default.hostname = host.docker.internal
database.default.port = 3306
database.default.database = db_myapp
database.default.username = myapp
database.default.password = your_secure_password
database.default.DBDriver = MySQLi
```

Simpan file `.env` di luar repository Git dan gunakan kredensial yang kuat.

---

# Langkah 8 — Menyiapkan Docker Compose

Buat file.

```text
/opt/docker/apps/myapp/docker-compose.yml
```

Container menggunakan:

- image lokal
- source read-only
- writable terpisah
- Unix Socket
- health check
- log rotation

Kemudian jalankan.

```bash
docker compose up -d
```

Verifikasi.

```bash
docker ps
```

---

# Langkah 9 — Menyiapkan PHP-FPM

Buat file.

```text
/opt/docker/apps/myapp/zz-custom.conf
```

Pastikan socket.

```ini
listen=/run/php/myapp.sock
```

Restart container.

```bash
docker compose restart
```

Periksa socket.

```bash
ls -l /run/php
```

---

# Langkah 10 — Menyiapkan Nginx

Buat Virtual Host.

```text
/etc/nginx/sites-available/myapp.conf
```

Pastikan:

- Document Root menuju `public`
- Menggunakan Unix Socket
- Hanya `index.php` yang dieksekusi
- Hidden file diblokir
- Security Header aktif

Uji konfigurasi.

```bash
sudo nginx -t
```

Reload.

```bash
sudo systemctl reload nginx
```

---

# Langkah 11 — Pengujian

Periksa container.

```bash
docker ps
```

Periksa log.

```bash
docker logs myapp-php
```

Periksa health check.

```bash
docker inspect myapp-php \
--format '{{.State.Health.Status}}'
```

Periksa Nginx.

```bash
sudo systemctl status nginx
```

Periksa MariaDB.

```bash
sudo systemctl status mariadb
```

Akses aplikasi menggunakan browser dan pastikan seluruh fungsi utama berjalan dengan baik.

---

# Langkah 12 — Hardening

Sebelum aplikasi dipublikasikan, pastikan:

- Source code read-only.
- Direktori writable dipisahkan.
- HTTPS aktif.
- Security Header aktif.
- Hidden file diblokir.
- Hanya `index.php` yang dapat dieksekusi.
- Docker menggunakan `no-new-privileges`.
- Health check aktif.
- Log rotation aktif.
- Backup telah dikonfigurasi.

---

# Langkah 13 — Monitoring

Lakukan pemantauan secara berkala.

```bash
docker stats
```

```bash
docker logs -f myapp-php
```

```bash
docker ps
```

Pastikan penggunaan CPU, memori, dan ruang penyimpanan tetap dalam batas yang wajar.

---

# Langkah 14 — Backup

Lakukan backup secara rutin.

Source code.

```bash
tar czf source.tar.gz \
/var/apps/myapp/htdocs
```

Direktori writable.

```bash
tar czf writable.tar.gz \
/var/apps/myapp/data/writable
```

Database.

```bash
mysqldump db_myapp > database.sql
```

Lakukan uji restore secara berkala untuk memastikan hasil backup dapat digunakan.

---

# Diagram Deployment

```text
/opt/docker/images/php8.3/
            │
            ▼
     local/php:8.3
            │
            ▼
docker-compose.yml
            │
            ▼
Docker PHP-FPM
            │
     Unix Socket
            │
            ▼
 Nginx (Host Linux)
            │
            ▼
      Internet Users

MariaDB (Host)
      ▲
      │
CodeIgniter 4
```

---

# Alur Deployment

```text
Install Linux
      │
      ▼
Install Docker
      │
      ▼
Build PHP Image
      │
      ▼
Deploy PHP-FPM
      │
      ▼
Deploy Nginx
      │
      ▼
Deploy MariaDB
      │
      ▼
Konfigurasi Aplikasi
      │
      ▼
Hardening
      │
      ▼
Monitoring
      │
      ▼
Go Live
```

---

# Ringkasan Praktik Terbaik

Selama seri ini kita telah menerapkan beberapa praktik terbaik berikut.

| Komponen | Praktik |
|----------|----------|
| Docker | Satu aplikasi satu container PHP-FPM |
| Nginx | Berjalan di host |
| PHP-FPM | Menggunakan Unix Socket |
| Source Code | Read-only |
| Writable | Dipisahkan dari source code |
| Database | MariaDB berjalan di host |
| Logging | Log rotation aktif |
| Monitoring | Health check aktif |
| Backup | Source, writable, dan database dipisahkan |
| Hardening | Prinsip *least privilege* |

---

# Penutup

Tidak ada satu arsitektur yang cocok untuk semua kebutuhan, namun pendekatan **Nginx di host, PHP-FPM di Docker, dan MariaDB di host** memberikan keseimbangan yang baik antara performa, keamanan, dan kemudahan operasional.

Dengan memisahkan source code dari data runtime, menggunakan image Docker yang terstandarisasi, menerapkan hardening pada setiap lapisan, serta melengkapi server dengan monitoring dan backup, lingkungan produksi akan menjadi lebih mudah dipelihara dan lebih siap menghadapi perubahan di masa depan.

Semoga seri **Docker PHP Production** ini dapat menjadi referensi bagi administrator sistem, pengembang, maupun praktisi DevOps yang ingin membangun platform PHP yang stabil, aman, dan mudah dikembangkan.
````
