````markdown
---
title: "Konfigurasi PHP-FPM untuk Produksi pada Docker"
slug: "konfigurasi-php-fpm-produksi"
date: 2026-08-03T19:00:00+07:00
lastmod: 2026-08-03T19:00:00+07:00
draft: false

description: "Panduan mengoptimalkan PHP-FPM pada Docker untuk lingkungan produksi menggunakan Unix Socket, process manager, dan konfigurasi pool yang aman."

author: "NR Technology"

series:
  - Docker PHP Production
weight: 4

tags:
  - PHP
  - PHP-FPM
  - Docker
  - Linux
  - DevOps
  - Performance

categories:
  - Docker
  - PHP

keywords:
  - php-fpm
  - php-fpm tuning
  - php-fpm docker
  - php production
  - php socket

toc: true
showReadingTime: true
showCodeCopyButtons: true
showWordCount: true
---

# Konfigurasi PHP-FPM untuk Produksi pada Docker

Pada artikel sebelumnya kita telah membuat container PHP-FPM menggunakan Docker Compose. Selanjutnya kita akan mengoptimalkan PHP-FPM agar mampu melayani aplikasi produksi secara stabil, aman, dan mudah dipantau.

Konfigurasi dilakukan melalui file **pool PHP-FPM**, sehingga setiap aplikasi dapat memiliki konfigurasi yang berbeda tanpa memengaruhi aplikasi lain.

---

# Apa Itu PHP-FPM?

PHP-FPM (*PHP FastCGI Process Manager*) adalah implementasi FastCGI yang dirancang untuk menjalankan proses PHP secara efisien.

Keuntungan menggunakan PHP-FPM antara lain:

- Manajemen worker yang lebih baik.
- Konsumsi memori lebih efisien.
- Mendukung Unix Socket.
- Mendukung banyak pool.
- Mudah dikombinasikan dengan Nginx.

---

# Mengapa Menggunakan Pool?

Setiap aplikasi dapat memiliki pool sendiri.

Contoh:

```text
/run/php/myapp.sock
/run/php/literasikita.sock
/run/php/ojs.sock
```

Keuntungannya:

- Tidak saling mengganggu.
- Mudah melakukan troubleshooting.
- Konfigurasi setiap aplikasi dapat berbeda.
- Beban aplikasi lebih mudah dikontrol.

---

# Lokasi File Pool

Pada implementasi ini file pool berada di host.

```text
/opt/docker/apps/myapp/zz-custom.conf
```

Kemudian di-*mount* ke dalam container.

```yaml
- /opt/docker/apps/myapp/zz-custom.conf:/usr/local/etc/php-fpm.d/zz-custom.conf:ro
```

Dengan pendekatan ini, perubahan konfigurasi cukup dilakukan di host tanpa membangun ulang image Docker.

---

# Contoh Konfigurasi Pool

Berikut konfigurasi yang digunakan.

```ini
[www]

user = www-data
group = www-data

listen = /run/php/myapp.sock

listen.owner = www-data
listen.group = www-data
listen.mode = 0660

pm = dynamic

pm.max_children = 20
pm.start_servers = 4
pm.min_spare_servers = 2
pm.max_spare_servers = 6

pm.max_requests = 500

request_terminate_timeout = 300s

catch_workers_output = yes
decorate_workers_output = no

clear_env = no

ping.path = /fpm-ping
ping.response = pong

pm.status_path = /fpm-status
```

---

# User dan Group

```ini
user = www-data
group = www-data
```

Worker PHP dijalankan menggunakan user **www-data**.

Hal ini sesuai dengan standar Debian dan Ubuntu sehingga memudahkan pengelolaan permission.

---

# Menggunakan Unix Socket

```ini
listen = /run/php/myapp.sock
```

Komunikasi antara Nginx dan PHP-FPM dilakukan menggunakan Unix Socket.

Dibandingkan TCP, pendekatan ini memiliki beberapa keuntungan:

- Lebih cepat.
- Tidak membuka port.
- Lebih aman.
- Overhead lebih kecil.

---

# Permission Socket

```ini
listen.owner = www-data
listen.group = www-data
listen.mode = 0660
```

Permission tersebut memastikan hanya proses yang memiliki hak akses yang dapat menggunakan socket.

---

# Process Manager

PHP-FPM memiliki beberapa mode.

| Mode | Keterangan |
|------|------------|
| static | Jumlah worker tetap |
| dynamic | Jumlah worker berubah sesuai beban |
| ondemand | Worker dibuat ketika diperlukan |

Untuk server produksi saya lebih memilih:

```ini
pm = dynamic
```

Mode ini memberikan keseimbangan antara performa dan penggunaan memori.

---

# pm.max_children

```ini
pm.max_children = 20
```

Menentukan jumlah maksimum worker PHP.

Apabila seluruh worker sibuk, request berikutnya akan menunggu hingga worker tersedia.

Nilai ini harus disesuaikan dengan kapasitas RAM server.

---

# pm.start_servers

```ini
pm.start_servers = 4
```

Jumlah worker yang langsung dijalankan ketika PHP-FPM aktif.

---

# pm.min_spare_servers

```ini
pm.min_spare_servers = 2
```

Jumlah minimum worker yang selalu siaga.

---

# pm.max_spare_servers

```ini
pm.max_spare_servers = 6
```

Jumlah maksimum worker idle.

Apabila jumlah worker idle melebihi nilai tersebut, PHP-FPM akan menghentikan worker yang tidak diperlukan.

---

# pm.max_requests

```ini
pm.max_requests = 500
```

Setelah melayani 500 request, worker akan dihentikan dan dibuat ulang.

Keuntungan konfigurasi ini:

- Mengurangi memory leak.
- Menjaga penggunaan memori tetap stabil.
- Cocok untuk aplikasi yang berjalan dalam jangka panjang.

---

# request_terminate_timeout

```ini
request_terminate_timeout = 300s
```

Request yang berjalan lebih dari lima menit akan dihentikan.

Hal ini mencegah worker terkunci oleh proses yang tidak selesai.

---

# catch_workers_output

```ini
catch_workers_output = yes
```

Output dari worker PHP akan diteruskan ke log sehingga memudahkan proses debugging.

---

# clear_env

```ini
clear_env = no
```

Mengizinkan worker PHP membaca variabel lingkungan seperti:

- TZ
- APP_ENV
- Variabel lain yang diberikan Docker.

---

# Monitoring Menggunakan Ping

```ini
ping.path = /fpm-ping
```

Digunakan untuk memastikan PHP-FPM masih berjalan.

Contoh hasil.

```text
pong
```

---

# Monitoring Status

```ini
pm.status_path = /fpm-status
```

Halaman ini dapat digunakan untuk melihat:

- Worker aktif.
- Worker idle.
- Jumlah request.
- Status proses.

Sebaiknya akses ke halaman ini dibatasi hanya dari localhost atau jaringan administrasi.

---

# Menguji Konfigurasi

Setelah mengubah konfigurasi pool, lakukan restart container.

```bash
docker compose restart
```

Periksa log.

```bash
docker logs myapp-php
```

Pastikan tidak ada kesalahan konfigurasi.

---

# Memastikan Socket Terbentuk

```bash
ls -l /run/php
```

Contoh.

```text
myapp.sock
```

---

# Memastikan PHP-FPM Berjalan

```bash
docker exec myapp-php ps -ef
```

Contoh hasil.

```text
php-fpm: master process
php-fpm: pool www
```

---

# Best Practice

Beberapa praktik yang saya gunakan pada server produksi.

- Satu aplikasi satu socket.
- Gunakan Unix Socket.
- Gunakan mode dynamic.
- Batasi jumlah worker sesuai RAM.
- Gunakan pm.max_requests.
- Aktifkan monitoring.
- Pisahkan file pool setiap aplikasi.
- Simpan konfigurasi di host.

Pendekatan ini membuat setiap aplikasi memiliki konfigurasi PHP-FPM yang independen sehingga lebih mudah dipelihara.

---

# Kesimpulan

Konfigurasi PHP-FPM yang baik merupakan salah satu faktor penting dalam menjaga performa dan stabilitas aplikasi PHP.

Dengan memanfaatkan Unix Socket, process manager yang tepat, monitoring, dan konfigurasi pool yang terpisah untuk setiap aplikasi, administrator dapat membangun lingkungan produksi yang lebih aman, lebih mudah dipelihara, dan lebih efisien.

Pada artikel berikutnya kita akan membahas konfigurasi **Nginx** untuk CodeIgniter 4, termasuk optimasi FastCGI dan berbagai teknik hardening agar aplikasi lebih aman ketika diakses melalui internet.
````
