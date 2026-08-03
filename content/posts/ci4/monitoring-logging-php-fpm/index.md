---
title: "Monitoring dan Logging PHP-FPM Docker untuk Lingkungan Produksi"
slug: "monitoring-logging-php-fpm"
date: 2026-08-04T08:00:00+07:00
lastmod: 2026-08-04T08:00:00+07:00
draft: false

description: "Panduan melakukan monitoring dan logging PHP-FPM Docker pada lingkungan produksi agar lebih mudah melakukan troubleshooting, analisis performa, dan pemeliharaan server."

author: "NR Technology"

series:
  - Docker PHP Production
weight: 9

tags:
  - Docker
  - PHP-FPM
  - Monitoring
  - Logging
  - Linux
  - DevOps

categories:
  - Docker
  - Monitoring

keywords:
  - php-fpm monitoring
  - docker logging
  - php-fpm logs
  - docker healthcheck
  - monitoring docker

toc: true
showReadingTime: true
showWordCount: true
showCodeCopyButtons: true
---

# Monitoring dan Logging PHP-FPM Docker untuk Lingkungan Produksi

Sebuah aplikasi yang berjalan dengan baik hari ini belum tentu akan tetap berjalan dengan baik beberapa bulan kemudian.

Oleh karena itu, selain membangun arsitektur deployment yang aman, administrator juga harus memiliki mekanisme untuk memantau kondisi aplikasi, mendeteksi masalah sejak dini, serta mengumpulkan informasi yang dibutuhkan ketika terjadi gangguan.

Pada artikel ini kita akan membahas bagaimana melakukan monitoring dan logging terhadap container PHP-FPM yang berjalan menggunakan Docker.

---

# Mengapa Monitoring Penting?

Banyak administrator baru mulai memeriksa server ketika aplikasi sudah tidak dapat diakses.

Pendekatan tersebut sering kali membuat proses investigasi menjadi lebih sulit.

Monitoring memungkinkan administrator mengetahui kondisi aplikasi sebelum gangguan menjadi lebih besar.

Beberapa manfaat monitoring antara lain:

- Mengetahui apakah container masih berjalan.
- Memantau penggunaan CPU dan memori.
- Mengetahui apabila PHP-FPM berhenti.
- Memudahkan analisis performa.
- Membantu proses troubleshooting.

---

# Arsitektur Monitoring

```text
                Internet
                    │
                    ▼
              Nginx (Host)
                    │
                    ▼
           Docker PHP-FPM
                    │
      ┌─────────────┴─────────────┐
      ▼                           ▼
 Docker Health Check        Docker Logs
      │                           │
      └─────────────┬─────────────┘
                    ▼
            Administrator
```

---

# Health Check

Docker menyediakan mekanisme untuk memeriksa kondisi container secara berkala.

Contoh konfigurasi.

```yaml
healthcheck:
  test: ["CMD-SHELL","pidof php-fpm || exit 1"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 10s
```

Docker akan menjalankan perintah tersebut secara otomatis.

Apabila proses PHP-FPM tidak ditemukan, status container akan berubah menjadi **unhealthy**.

---

# Melihat Status Health Check

Periksa status container.

```bash
docker ps
```

Atau.

```bash
docker inspect myapp-php \
--format '{{.State.Health.Status}}'
```

Contoh.

```text
healthy
```

---

# Melihat Detail Health Check

Informasi lebih lengkap dapat diperoleh melalui.

```bash
docker inspect myapp-php
```

Bagian **Health** akan menampilkan:

- waktu pemeriksaan
- hasil pemeriksaan
- exit code
- log health check

---

# Melihat Log Container

Docker menyimpan log setiap container.

Melihat seluruh log.

```bash
docker logs myapp-php
```

Melihat log secara real time.

```bash
docker logs -f myapp-php
```

Menampilkan 100 baris terakhir.

```bash
docker logs --tail=100 myapp-php
```

---

# Menggunakan Log Rotation

Log yang terus bertambah dapat memenuhi ruang penyimpanan.

Karena itu aktifkan log rotation.

```yaml
logging:
  driver: json-file

  options:
    max-size: "10m"
    max-file: "5"
```

Dengan konfigurasi tersebut.

- Maksimum satu file log berukuran 10 MB.
- Docker menyimpan maksimum lima file log.

Total penggunaan ruang sekitar 50 MB.

---

# Melihat Penggunaan Resource

Docker menyediakan informasi penggunaan resource.

```bash
docker stats
```

Contoh.

```text
CONTAINER      CPU %     MEM USAGE
myapp-php      1.5%      96MiB
```

Informasi tersebut berguna untuk mengetahui apakah container membutuhkan penyesuaian konfigurasi PHP-FPM.

---

# Memeriksa Proses PHP-FPM

Masuk ke container.

```bash
docker exec -it myapp-php bash
```

Lihat proses.

```bash
ps -ef
```

Contoh.

```text
php-fpm: master process

php-fpm: pool www

php-fpm: pool www
```

Pastikan terdapat satu **master process** dan beberapa **worker process**.

---

# Memastikan Socket PHP

Periksa socket PHP.

```bash
ls -l /run/php
```

Contoh.

```text
myapp.sock
```

Apabila socket tidak muncul, kemungkinan PHP-FPM gagal berjalan.

---

# Monitoring Pool PHP-FPM

Apabila `pm.status_path` telah diaktifkan.

```ini
pm.status_path = /fpm-status
```

Administrator dapat memperoleh informasi seperti:

- jumlah worker
- worker aktif
- worker idle
- request yang diproses
- antrian request

Sebaiknya halaman ini hanya dapat diakses dari jaringan administrasi.

---

# Monitoring Ping

Apabila diaktifkan.

```ini
ping.path = /fpm-ping
```

PHP-FPM akan memberikan respon.

```text
pong
```

Fitur ini dapat digunakan oleh monitoring system untuk memastikan PHP-FPM masih merespons dengan benar.

---

# Memeriksa Permission

Masalah permission sering menjadi penyebab aplikasi gagal berjalan.

Periksa.

```bash
ls -lah /var/www/html
```

Pastikan direktori `writable` memiliki hak akses yang sesuai.

---

# Melihat Mount Docker

Pastikan bind mount berjalan dengan benar.

```bash
mount
```

Atau.

```bash
cat /proc/mounts
```

Administrator dapat memastikan bahwa:

- source code dipasang sebagai read-only.
- writable dipasang sebagai read-write.

---

# Restart Container

Apabila diperlukan.

```bash
docker compose restart
```

Atau.

```bash
docker restart myapp-php
```

---

# Mengikuti Log Saat Restart

```bash
docker logs -f myapp-php
```

Administrator dapat segera mengetahui apabila terjadi kesalahan konfigurasi.

---

# Monitoring Penggunaan Disk

Pastikan log Docker tidak memenuhi disk.

```bash
docker system df
```

Untuk melihat penggunaan ruang oleh image, container, volume, dan cache.

---

# Monitoring Image

Daftar image.

```bash
docker images
```

Pastikan image yang digunakan sesuai dengan versi yang diharapkan.

---

# Monitoring Container

Daftar seluruh container.

```bash
docker ps -a
```

Container yang berhenti secara tiba-tiba dapat segera diketahui melalui perintah ini.

---

# Monitoring Docker Service

Pastikan Docker berjalan.

```bash
systemctl status docker
```

Apabila Docker berhenti, seluruh container juga akan berhenti.

---

# Troubleshooting

## Container Tidak Mau Berjalan

Periksa log.

```bash
docker logs myapp-php
```

---

## Health Check Gagal

Periksa apakah proses PHP-FPM berjalan.

```bash
docker exec myapp-php \
pidof php-fpm
```

---

## Socket Tidak Ada

Masuk ke container.

```bash
docker exec -it myapp-php bash
```

Periksa konfigurasi pool PHP-FPM.

---

## Penggunaan Memori Tinggi

Periksa.

```bash
docker stats
```

Kemudian sesuaikan parameter seperti:

- pm.max_children
- pm.start_servers
- pm.max_spare_servers

---

## Penggunaan CPU Tinggi

Periksa apakah:

- terdapat request yang sangat lama,
- aplikasi mengalami loop,
- atau ada proses latar belakang yang berjalan terus-menerus.

---

# Integrasi dengan Monitoring Lain

Docker menyediakan data dasar, tetapi pada lingkungan produksi saya menyarankan mengintegrasikannya dengan sistem monitoring seperti:

- Zabbix
- Wazuh
- Prometheus
- Grafana

Dengan demikian administrator dapat memperoleh:

- grafik penggunaan CPU
- penggunaan memori
- status container
- kapasitas disk
- alert apabila terjadi gangguan

---

# Best Practice

Beberapa praktik yang saya gunakan.

- Aktifkan Docker Health Check.
- Aktifkan log rotation.
- Gunakan Unix Socket.
- Pantau penggunaan memori.
- Pantau penggunaan CPU.
- Pantau ruang disk.
- Periksa log secara berkala.
- Gunakan sistem monitoring terpusat.
- Simpan log sesuai kebijakan retensi organisasi.

Monitoring yang konsisten akan membantu mendeteksi masalah lebih awal dan mempermudah proses investigasi apabila terjadi insiden.

---

# Kesimpulan

Monitoring dan logging merupakan bagian penting dari operasional server produksi.

Docker telah menyediakan berbagai fitur bawaan seperti health check, log container, statistik penggunaan resource, dan informasi status container yang dapat dimanfaatkan untuk memantau kondisi aplikasi.

Dengan mengombinasikan fitur-fitur tersebut dengan sistem monitoring yang lebih lengkap, administrator dapat menjaga stabilitas aplikasi, mempercepat proses troubleshooting, dan meningkatkan keandalan layanan.

Pada artikel berikutnya kita akan menyusun **checklist hardening server PHP** yang dapat digunakan sebagai panduan sebelum aplikasi dipublikasikan ke lingkungan produksi.
