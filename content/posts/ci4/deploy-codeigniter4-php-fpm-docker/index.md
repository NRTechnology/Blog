---
title: "Deploy Aplikasi CodeIgniter 4 Menggunakan PHP-FPM Docker dan Nginx Host"
date: 2026-08-03T16:30:00+07:00
draft: false
author: "NR Technology"
description: "Membangun lingkungan produksi CodeIgniter 4 menggunakan Docker PHP-FPM, Nginx di host, dan MariaDB di host dengan pendekatan source code read-only."
series:
  - Membangun Platform Produksi CodeIgniter 4
weight: 1

tags:
  - CodeIgniter4
  - Docker
  - PHP-FPM
  - Nginx
  - Linux
  - DevOps
  - Self Hosting
categories:
  - Docker
  - CodeIgniter
  - Web Server
keywords:
  - codeigniter docker
  - php-fpm docker
  - nginx php-fpm
  - codeigniter production
  - docker php production
---

# Deploy Aplikasi CodeIgniter 4 Menggunakan PHP-FPM Docker dan Nginx Host

## Pendahuluan

Mengelola banyak aplikasi PHP pada satu server membutuhkan pendekatan yang mampu memberikan keamanan, kemudahan pemeliharaan, dan performa yang baik. Salah satu arsitektur yang saya gunakan adalah menjalankan **Nginx langsung pada host**, **MariaDB langsung pada host**, sedangkan setiap aplikasi PHP dijalankan di dalam **container Docker PHP-FPM**.

Pendekatan ini memungkinkan setiap aplikasi memiliki lingkungan PHP yang terisolasi, sementara web server dan database tetap dikelola secara terpusat. Selain itu, proses deployment menjadi lebih mudah karena source code dan data aplikasi dipisahkan dengan jelas.

---

> 📚 **Seri Docker PHP Production**
>
> Artikel ini merupakan bagian dari seri **Docker PHP Production**. Untuk mendapatkan pemahaman yang utuh, disarankan membaca artikel secara berurutan.
>
> 1. **Deploy Aplikasi CodeIgniter 4 Menggunakan PHP-FPM Docker dan Nginx Host** *(artikel ini)*
> 2. [Membangun Image PHP 8.3 Docker untuk Produksi](../membangun-image-php83-docker/)
> 3. [Deploy PHP-FPM Menggunakan Docker Compose untuk CodeIgniter 4](../deploy-php-fpm-docker-compose/)
> 4. [Konfigurasi PHP-FPM untuk Produksi pada Docker](../konfigurasi-php-fpm-produksi/)
> 5. [Konfigurasi Nginx untuk CodeIgniter 4 pada Lingkungan Produksi](../konfigurasi-nginx-codeigniter4/)
> 6. [Hardening Nginx untuk Aplikasi PHP pada Lingkungan Produksi](../hardening-nginx-aplikasi-php/)
> 7. [Memisahkan Source Code dan Direktori Writable pada CodeIgniter 4](../memisahkan-source-dan-writable-codeigniter4/)
> 8. [Menghubungkan CodeIgniter 4 Docker ke MariaDB yang Berjalan di Host](../codeigniter4-docker-mariadb-host/)
> 9. [Monitoring dan Logging PHP-FPM Docker untuk Lingkungan Produksi](../monitoring-logging-php-fpm/)
> 10. [Checklist Hardening Server PHP Sebelum Go Live](../checklist-hardening-server-php/)
> 11. [Studi Kasus: Deploy CodeIgniter 4 ke Lingkungan Produksi](../studi-kasus-deploy-codeigniter4-produksi/)
>
> **Artikel Tambahan**
>
> - [Kesalahan yang Sering Terjadi Saat Deploy CodeIgniter 4 Menggunakan Docker](../kesalahan-deploy-codeigniter4-docker/)

---

## Arsitektur

Arsitektur yang digunakan dapat digambarkan sebagai berikut.

```text
                Internet
                    │
                    ▼
              Nginx (Host)
                    │
          Unix Socket PHP-FPM
                    │
                    ▼
        Docker Container PHP-FPM
                    │
                    ▼
      Source Code CodeIgniter 4
                    │
                    ▼
          MariaDB (Host TCP)
```

Pada arsitektur ini:

- Nginx menangani seluruh koneksi HTTP maupun HTTPS.
- PHP dijalankan di dalam container Docker.
- MariaDB tetap berjalan langsung pada host.
- Komunikasi Nginx ke PHP-FPM menggunakan Unix Socket.
- Komunikasi PHP ke MariaDB menggunakan TCP.

---

## Struktur Direktori

Agar source code dan data aplikasi terpisah, saya menggunakan struktur direktori sebagai berikut.

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

Penjelasannya:

| Direktori | Fungsi |
|-----------|--------|
| htdocs | Source Code CodeIgniter 4 |
| data | Data runtime aplikasi |
| writable | Cache, Session, Upload, Log |
| backup | Backup aplikasi dan database |
| logs | Log tambahan aplikasi |

---

## Mengapa Source Code Dibuat Read-Only?

Secara default, container PHP tidak memiliki kebutuhan untuk mengubah source code aplikasi.

Dengan memberikan mount:

```yaml
- /var/apps/myapp/htdocs:/var/www/html:ro
```

source code menjadi **Read Only**.

Keuntungan pendekatan ini:

- Mencegah web shell memodifikasi aplikasi.
- Mengurangi risiko backdoor.
- Menghindari perubahan file framework.
- Mempermudah verifikasi integritas source code.
- Deployment lebih aman.

Direktori yang tetap dapat ditulis hanyalah:

```text
writable/
```

---

## Docker Compose

Container yang dijalankan hanya berisi PHP-FPM.

Beberapa hardening yang diterapkan antara lain:

- read_only
- no-new-privileges
- healthcheck
- log rotation
- tmpfs
- source code read-only
- writable terpisah

Pendekatan ini jauh lebih aman dibandingkan memberikan akses tulis ke seluruh project.

---

## PHP-FPM Menggunakan Unix Socket

Komunikasi antara Nginx dan PHP-FPM dilakukan menggunakan Unix Socket.

Contoh konfigurasi PHP-FPM:

```ini
listen=/run/php/myapp.sock
```

Sedangkan pada Nginx:

```nginx
fastcgi_pass unix:/run/php/myapp.sock;
```

Keuntungan Unix Socket:

- Latensi lebih rendah.
- Tidak membuka port TCP tambahan.
- Lebih aman.
- Konsumsi resource lebih kecil.

---

## Koneksi Database Menggunakan TCP

Berbeda dengan PHP-FPM, komunikasi menuju MariaDB dilakukan menggunakan TCP.

Contoh konfigurasi CodeIgniter 4:

```ini
database.default.hostname = host.docker.internal
database.default.port = 3306
```

Dengan cara ini container tetap dapat mengakses database tanpa perlu melakukan bind mount Unix Socket MariaDB.

---

## Memisahkan Direktori Writable

CodeIgniter 4 menggunakan direktori:

```text
writable/
```

untuk menyimpan seluruh data runtime.

Direktori tersebut dipindahkan ke:

```text
/var/apps/myapp/data/writable
```

yang kemudian di-mount kembali ke dalam container.

Keuntungan pendekatan ini:

- Backup lebih mudah.
- Source code tetap bersih.
- Deployment lebih aman.
- Data runtime tetap tersimpan meskipun container dihapus.

---

## Hardening Nginx

Beberapa hardening yang saya gunakan antara lain:

- DocumentRoot hanya menuju direktori `public`.
- Memblokir akses ke `.env`.
- Memblokir file Composer.
- Memblokir file Git.
- Memblokir file backup.
- Memblokir akses ke `spark`.
- Menambahkan Security Header.
- Membatasi eksekusi PHP hanya pada Front Controller.

Karena direktori `writable` berada di luar `public`, file upload tidak pernah dapat diakses secara langsung melalui browser.

---

## Keuntungan Pendekatan Ini

Beberapa keuntungan yang saya rasakan selama menggunakan arsitektur ini:

- Update aplikasi menjadi lebih mudah.
- Framework tetap bersih.
- Source code tidak dapat dimodifikasi.
- Backup menjadi sederhana.
- Performa tinggi karena Nginx berjalan langsung di host.
- PHP dapat diperbarui tanpa mengubah aplikasi.
- Isolasi antar aplikasi lebih baik.
- Konsumsi resource Docker lebih rendah dibanding menjalankan seluruh stack di dalam container.

---

## Kesimpulan

Menjalankan CodeIgniter 4 menggunakan kombinasi **Nginx pada host**, **PHP-FPM di Docker**, dan **MariaDB pada host** merupakan salah satu pendekatan yang sangat cocok untuk server produksi.

Dengan memisahkan source code dan direktori writable, menggunakan Unix Socket untuk komunikasi antara Nginx dan PHP-FPM, serta menerapkan prinsip **read-only source**, keamanan dan kemudahan pengelolaan aplikasi dapat meningkat secara signifikan.

Pendekatan ini juga sangat cocok digunakan pada server yang mengelola banyak aplikasi karena setiap aplikasi cukup memiliki satu container PHP-FPM, sementara Nginx dan MariaDB tetap berjalan secara terpusat.

---
{{< saweria >}}