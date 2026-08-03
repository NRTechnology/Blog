````markdown
---
title: "Menghubungkan CodeIgniter 4 Docker ke MariaDB yang Berjalan di Host"
slug: "codeigniter4-docker-mariadb-host"
date: 2026-08-03T23:00:00+07:00
lastmod: 2026-08-03T23:00:00+07:00
draft: false

description: "Panduan menghubungkan aplikasi CodeIgniter 4 yang berjalan di Docker PHP-FPM dengan MariaDB yang berjalan pada host menggunakan koneksi TCP."

author: "NR Technology"

series:
  - Docker PHP Production
weight: 8

tags:
  - CodeIgniter4
  - Docker
  - MariaDB
  - MySQL
  - Linux
  - DevOps

categories:
  - Docker
  - Database
  - CodeIgniter

keywords:
  - docker mariadb host
  - codeigniter mysql docker
  - codeigniter mariadb
  - php docker mysql
  - docker php database

toc: true
showReadingTime: true
showWordCount: true
showCodeCopyButtons: true
---

# Menghubungkan CodeIgniter 4 Docker ke MariaDB yang Berjalan di Host

Pada artikel sebelumnya kita telah memisahkan source code dan direktori `writable` sehingga aplikasi menjadi lebih aman dan mudah dikelola.

Langkah berikutnya adalah menghubungkan aplikasi CodeIgniter 4 yang berjalan di dalam container Docker dengan server MariaDB yang berjalan langsung pada host Linux.

Pendekatan ini memungkinkan satu server MariaDB digunakan bersama oleh banyak aplikasi tanpa harus menjalankan container database untuk setiap aplikasi.

---

# Arsitektur

```text
                Internet
                    │
                    ▼
            Nginx (Host Linux)
                    │
             Unix Socket
                    │
                    ▼
        Docker PHP-FPM Container
                    │
               TCP Port 3306
                    │
                    ▼
          MariaDB (Host Linux)
```

Pada arsitektur tersebut:

- Nginx berjalan pada host.
- PHP-FPM berjalan di Docker.
- MariaDB berjalan pada host.
- Komunikasi PHP ke database menggunakan TCP.

---

# Mengapa Tidak Menggunakan Unix Socket?

Ketika MariaDB berjalan di host sedangkan PHP berada di dalam container, penggunaan Unix Socket menjadi kurang praktis karena memerlukan bind mount socket database ke dalam container.

Sebaliknya, koneksi TCP memiliki beberapa keuntungan:

- Konfigurasi lebih sederhana.
- Mudah dipindahkan ke server lain.
- Tidak bergantung pada lokasi file socket.
- Konsisten untuk semua aplikasi.
- Mudah diintegrasikan dengan Docker.

---

# Menggunakan TCP

Pada artikel ini komunikasi dilakukan melalui port standar MariaDB.

```text
3306/TCP
```

Tidak diperlukan bind mount terhadap direktori:

```text
/run/mysqld
```

Container hanya memerlukan koneksi jaringan menuju host.

---

# Konfigurasi Docker

Docker Compose tidak memerlukan volume tambahan untuk database.

Contoh.

```yaml
volumes:

  - /run/php:/run/php

  - /var/apps/myapp/htdocs:/var/www/html:ro

  - /var/apps/myapp/data/writable:/var/www/html/writable:rw
```

Perhatikan bahwa tidak terdapat mount menuju:

```text
/run/mysqld
```

---

# Mengakses Host dari Docker

Docker menyediakan mekanisme agar container dapat mengakses host.

Tambahkan pada Docker Compose.

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

Dengan konfigurasi tersebut, container dapat mengakses host menggunakan nama:

```text
host.docker.internal
```

Pendekatan ini lebih fleksibel dibandingkan menggunakan alamat IP tetap.

---

# Konfigurasi Database CodeIgniter 4

Buka file:

```text
.env
```

Kemudian sesuaikan konfigurasi database.

```ini
database.default.hostname = host.docker.internal

database.default.port = 3306

database.default.database = db_myapp

database.default.username = myapp

database.default.password = your_secure_password

database.default.DBDriver = MySQLi
```

> **Catatan:** Gunakan nama database, nama pengguna, dan kata sandi yang sesuai dengan lingkungan Anda. Hindari menggunakan kredensial contoh ini pada sistem produksi.

---

# Membuat Database

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

---

# Membuat User Database

Buat user baru.

```sql
CREATE USER 'myapp'@'%'
IDENTIFIED BY 'your_secure_password';
```

Berikan hak akses.

```sql
GRANT ALL PRIVILEGES
ON db_myapp.*
TO 'myapp'@'%';

FLUSH PRIVILEGES;
```

Pada lingkungan produksi, pertimbangkan untuk membatasi host yang diizinkan mengakses database, misalnya berdasarkan subnet jaringan Docker atau alamat IP tertentu, daripada menggunakan `%`.

---

# Memastikan MariaDB Mendengarkan TCP

Periksa apakah MariaDB membuka port.

```bash
ss -ltn | grep 3306
```

Contoh.

```text
LISTEN 0 80 *:3306
```

Apabila tidak muncul, periksa konfigurasi MariaDB.

---

# Menguji Koneksi dari Container

Masuk ke container.

```bash
docker exec -it myapp-php bash
```

Kemudian uji koneksi menggunakan utilitas yang tersedia di dalam container, misalnya klien MariaDB/MySQL apabila telah dipasang, atau gunakan fitur koneksi database dari aplikasi.

Sebagai contoh apabila klien tersedia:

```bash
mysql \
-h host.docker.internal \
-P 3306 \
-u myapp \
-p
```

---

# Menguji Melalui CodeIgniter

Jalankan aplikasi.

Apabila konfigurasi benar, CodeIgniter akan dapat membaca database tanpa perlu melakukan perubahan tambahan.

Apabila terjadi kesalahan, aktifkan mode pengembangan sementara untuk melihat pesan error yang lebih informatif, lalu kembalikan ke mode produksi setelah selesai melakukan pengujian.

---

# Troubleshooting

## Connection Refused

Pastikan MariaDB berjalan.

```bash
systemctl status mariadb
```

---

## Access Denied

Periksa kembali:

- Nama database.
- Nama pengguna.
- Kata sandi.
- Hak akses user.

---

## Host Tidak Dapat Ditemukan

Pastikan Docker Compose telah berisi.

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

Kemudian restart container.

```bash
docker compose down

docker compose up -d
```

---

## Firewall

Pastikan firewall mengizinkan komunikasi yang diperlukan antara container dan layanan MariaDB pada host sesuai kebijakan keamanan yang diterapkan.

---

# Praktik Keamanan

Beberapa praktik yang saya rekomendasikan.

- Gunakan user database khusus untuk setiap aplikasi.
- Berikan hak akses hanya pada database yang diperlukan.
- Gunakan kata sandi yang kuat dan unik.
- Hindari menggunakan akun `root` dari aplikasi.
- Jangan menyimpan kredensial di repository Git.
- Simpan file `.env` di luar version control.
- Lakukan backup database secara berkala.

---

# Mengapa Satu MariaDB untuk Banyak Aplikasi?

Pada banyak server produksi, satu instance MariaDB dapat melayani banyak aplikasi sekaligus.

Keuntungannya antara lain:

- Konsumsi memori lebih rendah.
- Administrasi database lebih sederhana.
- Backup terpusat.
- Monitoring lebih mudah.
- Update MariaDB cukup dilakukan satu kali.

Selama setiap aplikasi menggunakan database dan user yang terpisah, pendekatan ini tetap mudah dikelola.

---

# Kesimpulan

Menghubungkan CodeIgniter 4 yang berjalan di dalam Docker dengan MariaDB yang berjalan pada host merupakan pendekatan yang sederhana dan efisien.

Dengan menggunakan koneksi TCP, setiap aplikasi dapat memanfaatkan layanan database yang sama tanpa perlu melakukan bind mount Unix Socket. Pendekatan ini memudahkan proses deployment, pemeliharaan, serta migrasi aplikasi ke server lain.

Pada artikel berikutnya kita akan membahas bagaimana memonitor container PHP-FPM, membaca log, dan melakukan troubleshooting apabila terjadi masalah pada lingkungan produksi.
````
