---
title: "Deploy PHP-FPM Menggunakan Docker Compose untuk CodeIgniter 4"
slug: "deploy-php-fpm-docker-compose"
date: 2026-08-03T18:00:00+07:00
lastmod: 2026-08-03T18:00:00+07:00
draft: false

description: "Membangun container PHP-FPM menggunakan Docker Compose untuk CodeIgniter 4 dengan pendekatan source code read-only, writable terpisah, dan PHP-FPM melalui Unix Socket."

author: "NR Technology"

series:
  - Membangun Platform Produksi CodeIgniter 4
weight: 3

tags:
  - Docker
  - Docker Compose
  - PHP-FPM
  - CodeIgniter4
  - DevOps
  - Linux

categories:
  - Docker
  - CodeIgniter

keywords:
  - docker compose php
  - php-fpm docker compose
  - codeigniter docker compose
  - php docker production
  - docker php-fpm

cover:
  image: "ci4-cover.png"
  alt: "Deploy Aplikasi CodeIgniter 4 Menggunakan PHP-FPM Docker dan Nginx Host"
  caption: "CodeIgniter 4 + PHP-FPM Docker + Nginx Host"

toc: true
showReadingTime: true
showCodeCopyButtons: true
showWordCount: true
---

# Deploy PHP-FPM Menggunakan Docker Compose untuk CodeIgniter 4

Pada artikel sebelumnya kita telah membangun image **local/php:8.3** yang akan digunakan oleh seluruh aplikasi PHP.

Selanjutnya kita akan membuat **Docker Compose** untuk menjalankan PHP-FPM secara aman pada lingkungan produksi.

Artikel ini menggunakan pendekatan yang sama seperti yang saya gunakan pada server produksi, yaitu:

- Nginx berjalan di host
- MariaDB berjalan di host
- PHP-FPM berjalan di Docker
- Source code dibuat read-only
- Direktori writable dipisahkan dari source code

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
      Docker Container PHP 8.3
                    │
                    ▼
        Source Code (Read Only)
                    │
                    ▼
          MariaDB (Host TCP)
```

Dengan arsitektur ini setiap aplikasi memiliki container PHP sendiri, sedangkan Nginx dan MariaDB tetap dikelola secara terpusat.

---

# Struktur Direktori

```text
/opt/docker/apps/
└── myapp/
    ├── docker-compose.yml
    └── zz-custom.conf

/var/apps/
└── myapp/
    ├── backup/
    ├── data/
    │   └── writable/
    ├── htdocs/
    └── logs/
```

Penjelasan:

| Direktori | Fungsi |
|-----------|--------|
| htdocs | Source Code CodeIgniter 4 |
| writable | Cache, Log, Upload, Session |
| backup | Backup aplikasi |
| logs | Log tambahan |

---

# Membuat Docker Compose

Buat file berikut.

```text
/opt/docker/apps/myapp/docker-compose.yml
```

Isi file tersebut.

```yaml
services:

  app:
    container_name: myapp-php

    image: local/php:8.3

    read_only: true

    restart: unless-stopped

    init: true

    stop_grace_period: 30s

    security_opt:
      - no-new-privileges:true

    working_dir: /var/www/html

    environment:
      TZ: Asia/Jakarta

    volumes:

      # PHP Socket
      - /run/php:/run/php

      # PHP Configuration
      - /opt/docker/images/php8.3/php.ini:/usr/local/etc/php/conf.d/99-custom.ini:ro

      - /opt/docker/apps/myapp/zz-custom.conf:/usr/local/etc/php-fpm.d/zz-custom.conf:ro

      # Source Code
      - /var/apps/myapp/htdocs:/var/www/html:ro

      # Runtime Data
      - /var/apps/myapp/data/writable:/var/www/html/writable:rw

    tmpfs:
      - /tmp

    healthcheck:
      test: ["CMD-SHELL","pidof php-fpm || exit 1"]
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

---

# Penjelasan Konfigurasi

Mari kita bahas setiap bagian penting.

---

## Image

```yaml
image: local/php:8.3
```

Container menggunakan image yang telah dibuat pada artikel sebelumnya sehingga seluruh aplikasi memiliki lingkungan PHP yang sama.

---

## Read Only

```yaml
read_only: true
```

Ini merupakan salah satu hardening paling penting.

Dengan konfigurasi tersebut, container tidak dapat memodifikasi source code.

Apabila aplikasi berhasil dieksploitasi, penyerang tidak dapat dengan mudah menanamkan web shell atau memodifikasi framework.

---

## Security Option

```yaml
security_opt:
  - no-new-privileges:true
```

Opsi ini mencegah proses di dalam container memperoleh hak akses tambahan melalui mekanisme Linux seperti `setuid`.

---

## Working Directory

```yaml
working_dir: /var/www/html
```

Menentukan direktori kerja utama PHP.

---

## Timezone

```yaml
environment:
  TZ: Asia/Jakarta
```

Menyamakan zona waktu container dengan server.

Hal ini penting untuk log, session, dan timestamp database.

---

# Memisahkan Source Code dan Writable

Source code dipasang sebagai read-only.

```yaml
- /var/apps/myapp/htdocs:/var/www/html:ro
```

Sedangkan direktori runtime dipasang sebagai read-write.

```yaml
- /var/apps/myapp/data/writable:/var/www/html/writable:rw
```

Pendekatan ini memberikan beberapa keuntungan.

- Backup lebih mudah.
- Source code tidak berubah.
- Deployment menjadi lebih aman.
- Runtime data tetap tersimpan walaupun container dihapus.

---

# PHP Configuration

Konfigurasi PHP dipasang langsung dari host.

```yaml
- /opt/docker/images/php8.3/php.ini:/usr/local/etc/php/conf.d/99-custom.ini:ro
```

Dengan demikian administrator cukup mengubah satu file php.ini tanpa harus membangun ulang image.

---

# PHP-FPM Pool

Konfigurasi PHP-FPM juga dipisahkan.

```yaml
- /opt/docker/apps/myapp/zz-custom.conf:/usr/local/etc/php-fpm.d/zz-custom.conf:ro
```

Setiap aplikasi dapat memiliki socket PHP-FPM sendiri.

Contoh.

```text
/run/php/myapp.sock
```

---

# tmpfs

```yaml
tmpfs:
  - /tmp
```

Direktori `/tmp` berada di memori sehingga lebih cepat dan otomatis bersih ketika container dihentikan.

---

# Health Check

```yaml
healthcheck:
  test:
```

Docker akan memeriksa apakah PHP-FPM masih berjalan.

Administrator dapat mengetahui apabila container mengalami masalah tanpa harus memeriksa log secara manual.

---

# Log Rotation

```yaml
logging:
```

Log Docker akan diputar secara otomatis.

Hal ini mencegah file log tumbuh tanpa batas dan menghabiskan ruang penyimpanan.

---

# Menjalankan Container

Masuk ke direktori aplikasi.

```bash
cd /opt/docker/apps/myapp
```

Jalankan container.

```bash
docker compose up -d
```

Periksa status.

```bash
docker ps
```

---

# Melihat Log

Untuk melihat log container.

```bash
docker logs -f myapp-php
```

---

# Memastikan Socket PHP-FPM

Pastikan socket berhasil dibuat.

```bash
ls -l /run/php
```

Contoh.

```text
myapp.sock
```

Socket tersebut nantinya akan digunakan oleh Nginx.

---

# Memastikan Health Check

```bash
docker inspect myapp-php --format '{{.State.Health.Status}}'
```

Apabila berhasil akan menghasilkan.

```text
healthy
```

---

# Best Practice

Saya menggunakan beberapa prinsip berikut pada seluruh server produksi.

- Satu aplikasi satu container PHP.
- Nginx tetap berjalan di host.
- MariaDB tetap berjalan di host.
- Source code read-only.
- Writable dipisahkan.
- Unix Socket untuk PHP-FPM.
- Log rotation aktif.
- Health check aktif.
- no-new-privileges aktif.
- tmpfs digunakan untuk direktori sementara.

Dengan pendekatan tersebut, pengelolaan puluhan aplikasi PHP pada satu server menjadi jauh lebih mudah.

---

# Kesimpulan

Docker Compose merupakan fondasi utama dalam menjalankan PHP-FPM pada lingkungan produksi.

Dengan menggabungkan **source code read-only**, **writable terpisah**, **health check**, **Unix Socket**, dan **hardening Docker**, kita dapat membangun lingkungan yang lebih aman, mudah dipelihara, dan konsisten.

Pada artikel berikutnya kita akan membahas bagaimana mengoptimalkan **PHP-FPM** melalui konfigurasi pool menggunakan file **zz-custom.conf** sehingga performa aplikasi menjadi lebih baik pada lingkungan produksi.

---
{{< saweria >}}