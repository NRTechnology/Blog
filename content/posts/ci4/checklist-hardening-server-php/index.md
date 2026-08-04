---
title: "Checklist Hardening Server PHP Sebelum Go Live"
slug: "checklist-hardening-server-php"
date: 2026-08-04T09:00:00+07:00
lastmod: 2026-08-04T09:00:00+07:00
draft: false

description: "Checklist hardening server PHP berbasis Nginx, Docker PHP-FPM, dan MariaDB yang dapat digunakan sebagai acuan sebelum aplikasi dipublikasikan ke lingkungan produksi."

author: "NR Technology"

series:
  - Membangun Platform Produksi CodeIgniter 4
weight: 10

tags:
  - Docker
  - PHP
  - PHP-FPM
  - Nginx
  - Linux
  - Security
  - Hardening

categories:
  - Security
  - Docker
  - Linux

keywords:
  - php hardening
  - nginx hardening
  - docker security
  - production checklist
  - linux security

cover:
  image: "ci4-cover.png"
  alt: "Deploy Aplikasi CodeIgniter 4 Menggunakan PHP-FPM Docker dan Nginx Host"
  caption: "CodeIgniter 4 + PHP-FPM Docker + Nginx Host"

toc: true
showReadingTime: true
showWordCount: true
showCodeCopyButtons: true
---

# Checklist Hardening Server PHP Sebelum Go Live

Setelah seluruh komponen server berhasil dikonfigurasi, pekerjaan administrator belum selesai. Sebelum aplikasi dipublikasikan ke internet, perlu dilakukan proses **hardening** untuk memastikan konfigurasi yang diterapkan telah memenuhi praktik keamanan yang baik.

Artikel ini merangkum berbagai langkah hardening yang telah dibahas pada seri sebelumnya dalam bentuk checklist yang dapat digunakan sebagai panduan sebelum aplikasi masuk ke lingkungan produksi.

---

# Arsitektur yang Digunakan

Checklist ini dibuat berdasarkan arsitektur berikut.

```text
                Internet
                    │
                    ▼
            Nginx (Host Linux)
                    │
           Unix Socket PHP-FPM
                    │
                    ▼
        Docker PHP-FPM Container
                    │
             TCP Port 3306
                    │
                    ▼
          MariaDB (Host Linux)
```

Karakteristik arsitektur:

- Nginx berjalan di host.
- PHP-FPM berjalan di Docker.
- MariaDB berjalan di host.
- Source code bersifat read-only.
- Direktori writable dipisahkan.
- Komunikasi Nginx menggunakan Unix Socket.

---

# 1. Hardening Sistem Operasi

Pastikan sistem operasi selalu diperbarui.

```bash
sudo apt update

sudo apt upgrade
```

Checklist:

- [ ] Sistem operasi menggunakan versi yang masih didukung.
- [ ] Patch keamanan telah diterapkan.
- [ ] Hanya paket yang diperlukan yang terpasang.
- [ ] Waktu server telah disinkronkan menggunakan NTP.

---

# 2. Hardening Docker

Pastikan Docker dikonfigurasi dengan baik.

Checklist:

- [ ] Menggunakan image resmi atau image internal yang tepercaya.
- [ ] Tidak menjalankan container dengan mode `--privileged`.
- [ ] Menggunakan `read_only: true`.
- [ ] Menggunakan `security_opt: no-new-privileges:true`.
- [ ] Menggunakan health check.
- [ ] Mengaktifkan log rotation.
- [ ] Tidak mengekspos port PHP-FPM ke internet.
- [ ] Container memiliki hak akses minimum yang diperlukan.

Contoh:

```yaml
read_only: true

security_opt:
  - no-new-privileges:true
```

---

# 3. Hardening Source Code

Checklist:

- [ ] Source code dipasang sebagai read-only.
- [ ] Direktori `writable` dipisahkan.
- [ ] File `.env` tidak masuk ke repository publik.
- [ ] Direktori `vendor` berasal dari sumber tepercaya.
- [ ] File backup tidak berada di Document Root.

Contoh mount.

```yaml
- /var/apps/myapp/htdocs:/var/www/html:ro

- /var/apps/myapp/data/writable:/var/www/html/writable:rw
```

---

# 4. Hardening Nginx

Checklist:

- [ ] Document Root hanya menuju `public`.
- [ ] `server_tokens off`.
- [ ] Hidden file diblokir.
- [ ] File backup diblokir.
- [ ] Repository Git diblokir.
- [ ] Composer diblokir.
- [ ] Hanya `index.php` yang dapat dieksekusi.
- [ ] Security Header telah ditambahkan.

Contoh.

```nginx
server_tokens off;
```

---

# 5. Hardening PHP-FPM

Checklist:

- [ ] Menggunakan Unix Socket.
- [ ] Menggunakan user `www-data`.
- [ ] `pm.max_children` disesuaikan.
- [ ] `pm.max_requests` diaktifkan.
- [ ] `request_terminate_timeout` telah diatur.
- [ ] Monitoring `pm.status_path` telah disiapkan.

Contoh.

```ini
pm.max_requests = 500
```

---

# 6. Hardening PHP

Periksa konfigurasi `php.ini`.

Checklist:

- [ ] `display_errors = Off`
- [ ] `log_errors = On`
- [ ] `expose_php = Off`
- [ ] `memory_limit` disesuaikan.
- [ ] `post_max_size` sesuai kebutuhan.
- [ ] `upload_max_filesize` sesuai kebutuhan.
- [ ] OPcache aktif pada lingkungan produksi.

Contoh.

```ini
display_errors = Off

log_errors = On

expose_php = Off
```

---

# 7. Hardening Database

Checklist:

- [ ] Menggunakan user database khusus.
- [ ] Tidak menggunakan akun `root`.
- [ ] Hak akses dibatasi pada database yang diperlukan.
- [ ] Kata sandi kuat dan unik.
- [ ] Backup database dilakukan secara berkala.
- [ ] Akses jaringan dibatasi sesuai kebutuhan.

Contoh hak akses.

```sql
GRANT ALL PRIVILEGES
ON db_myapp.*
TO 'myapp'@'%';
```

Pada lingkungan produksi, gunakan pembatasan host yang lebih spesifik dibandingkan `%` apabila memungkinkan.

---

# 8. Hardening Permission

Checklist:

- [ ] Direktori `writable` dimiliki oleh user yang menjalankan PHP-FPM.
- [ ] Tidak menggunakan permission `777`.
- [ ] Source code tidak dapat ditulis.
- [ ] File konfigurasi memiliki permission yang sesuai.

Contoh.

```bash
chown -R 33:33 \
/var/apps/myapp/data/writable

chmod -R 775 \
/var/apps/myapp/data/writable
```

---

# 9. Hardening SSL/TLS

Checklist:

- [ ] Menggunakan HTTPS.
- [ ] Sertifikat masih berlaku.
- [ ] Redirect HTTP ke HTTPS.
- [ ] Menggunakan protokol TLS yang aman sesuai rekomendasi saat ini.
- [ ] Cipher suite mengikuti praktik terbaik yang berlaku.

---

# 10. Hardening Firewall

Checklist:

- [ ] Hanya port yang diperlukan dibuka.
- [ ] Port administrasi dibatasi.
- [ ] Firewall aktif.
- [ ] Aturan firewall didokumentasikan.

Contoh layanan yang umumnya dibuka.

| Port | Layanan |
|------|----------|
| 80 | HTTP |
| 443 | HTTPS |
| 22 | SSH (dibatasi) |

---

# 11. Hardening SSH

Checklist:

- [ ] Login sebagai `root` dinonaktifkan atau dibatasi sesuai kebijakan.
- [ ] Menggunakan autentikasi berbasis kunci publik.
- [ ] Password yang kuat apabila autentikasi sandi masih diizinkan.
- [ ] Port SSH dikelola sesuai kebijakan organisasi.
- [ ] Akses dibatasi menggunakan firewall atau VPN jika memungkinkan.

---

# 12. Logging

Checklist:

- [ ] Log Nginx aktif.
- [ ] Log PHP aktif.
- [ ] Log Docker aktif.
- [ ] Log rotation aktif.
- [ ] Kebijakan retensi log telah ditetapkan.

---

# 13. Monitoring

Checklist:

- [ ] Docker Health Check aktif.
- [ ] Monitoring CPU.
- [ ] Monitoring RAM.
- [ ] Monitoring Disk.
- [ ] Monitoring Service.
- [ ] Alert telah dikonfigurasi.

---

# 14. Backup

Checklist:

- [ ] Backup database.
- [ ] Backup source code.
- [ ] Backup konfigurasi Docker.
- [ ] Backup konfigurasi Nginx.
- [ ] Backup diuji secara berkala melalui proses restore.

---

# 15. Verifikasi Sebelum Go Live

Lakukan beberapa pengujian berikut.

```bash
docker compose ps
```

```bash
docker ps
```

```bash
docker logs myapp-php
```

```bash
sudo nginx -t
```

```bash
sudo systemctl status nginx
```

```bash
sudo systemctl status mariadb
```

Pastikan seluruh layanan berjalan dengan baik.

---

# Checklist Akhir

| Komponen | Status |
|-----------|:------:|
| Sistem Operasi | ☐ |
| Docker | ☐ |
| PHP-FPM | ☐ |
| PHP | ☐ |
| Nginx | ☐ |
| Database | ☐ |
| Permission | ☐ |
| SSL/TLS | ☐ |
| Firewall | ☐ |
| SSH | ☐ |
| Logging | ☐ |
| Monitoring | ☐ |
| Backup | ☐ |
| Dokumentasi | ☐ |
| Uji Restore Backup | ☐ |

Checklist ini dapat dicetak atau dijadikan bagian dari prosedur operasional standar (SOP) deployment sehingga setiap implementasi mengikuti langkah yang konsisten.

---

# Dokumentasi Konfigurasi

Selain memastikan server telah aman, dokumentasikan konfigurasi penting yang digunakan, misalnya:

- Versi sistem operasi.
- Versi Docker.
- Versi PHP.
- Versi Nginx.
- Versi MariaDB.
- Struktur direktori deployment.
- Lokasi file konfigurasi.
- Jadwal backup.
- Jadwal pemeliharaan.

Dokumentasi akan sangat membantu ketika melakukan audit, migrasi server, atau proses pemulihan setelah insiden.

---

# Kesimpulan

Hardening bukanlah satu langkah yang dilakukan sekali saja, melainkan proses berkelanjutan yang mencakup konfigurasi, pemantauan, pembaruan, dan evaluasi keamanan secara berkala.

Checklist pada artikel ini dapat digunakan sebagai acuan sebelum aplikasi dipublikasikan ke lingkungan produksi maupun ketika melakukan audit terhadap server yang sudah berjalan.

Dengan menerapkan prinsip *least privilege*, memisahkan source code dari data runtime, menggunakan Docker secara aman, mengamankan Nginx dan PHP-FPM, serta melengkapi server dengan monitoring dan backup yang memadai, lingkungan produksi akan menjadi lebih stabil, mudah dipelihara, dan memiliki tingkat keamanan yang lebih baik.

Pada artikel terakhir seri ini, kita akan menggabungkan seluruh konsep yang telah dibahas ke dalam sebuah studi kasus deployment lengkap, mulai dari server kosong hingga aplikasi CodeIgniter 4 siap digunakan di lingkungan produksi.

---
{{< saweria >}}