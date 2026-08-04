---
title: "Kesalahan yang Sering Terjadi Saat Deploy CodeIgniter 4 Menggunakan Docker"
slug: "kesalahan-deploy-codeigniter4-docker"
date: 2026-08-04T11:00:00+07:00
lastmod: 2026-08-04T11:00:00+07:00
draft: false

description: "Membahas berbagai kesalahan yang sering terjadi saat melakukan deployment CodeIgniter 4 menggunakan Docker PHP-FPM, Nginx, dan MariaDB beserta solusi yang dapat diterapkan."

author: "NR Technology"

tags:
  - CodeIgniter4
  - Docker
  - PHP-FPM
  - Nginx
  - MariaDB
  - Linux
  - Troubleshooting

categories:
  - Docker
  - CodeIgniter
  - Troubleshooting

keywords:
  - codeigniter docker error
  - docker php-fpm
  - codeigniter troubleshooting
  - docker deployment
  - php production

cover:
  image: "ci4-cover.png"
  alt: "Deploy Aplikasi CodeIgniter 4 Menggunakan PHP-FPM Docker dan Nginx Host"
  caption: "CodeIgniter 4 + PHP-FPM Docker + Nginx Host"

toc: true
showReadingTime: true
showWordCount: true
showCodeCopyButtons: true
---

# Kesalahan yang Sering Terjadi Saat Deploy CodeIgniter 4 Menggunakan Docker

Membangun lingkungan produksi menggunakan Docker memang memberikan banyak keuntungan, namun ada beberapa kesalahan yang hampir selalu muncul ketika pertama kali melakukan deployment.

Artikel ini merangkum berbagai masalah yang paling sering ditemui beserta penyebab dan cara mengatasinya.

---

# 1. Writable Tidak Dipisahkan

Masalah paling umum adalah seluruh source code dibuat dapat ditulis.

Contoh:

```text
htdocs/
├── app/
├── public/
├── system/
├── vendor/
└── writable/
```

Kemudian seluruh direktori diberikan hak tulis.

Akibatnya:

- Framework dapat berubah.
- Web shell dapat ditulis.
- Source code sulit diverifikasi.

## Solusi

Pisahkan direktori runtime.

```text
/var/apps/myapp/
├── htdocs/
└── data/
    └── writable/
```

Mount pada Docker.

```yaml
- /var/apps/myapp/htdocs:/var/www/html:ro

- /var/apps/myapp/data/writable:/var/www/html/writable:rw
```

---

# 2. Source Code Read Only Tetapi Mount Gagal

Error yang sering muncul.

```text
read-only file system
```

Biasanya terjadi ketika Docker mencoba membuat mount point pada source code yang sudah dibuat read-only.

## Penyebab

Direktori tujuan belum ada.

Contoh.

```text
writable/uploads
```

belum dibuat.

## Solusi

Pastikan seluruh struktur direktori telah tersedia sebelum container dijalankan.

---

# 3. Permission Writable Salah

Gejala.

- Session gagal dibuat.
- Upload gagal.
- Cache tidak dapat ditulis.

## Solusi

Pastikan direktori writable dimiliki oleh user PHP.

```bash
chown -R 33:33 \
/var/apps/myapp/data/writable
```

Kemudian.

```bash
chmod -R 775 \
/var/apps/myapp/data/writable
```

Hindari penggunaan permission `777`.

---

# 4. Nginx Mengarah ke Direktori yang Salah

Kesalahan.

```nginx
root /var/apps/myapp/htdocs;
```

Akibatnya source code dapat terekspos.

## Solusi

Gunakan.

```nginx
root /var/apps/myapp/htdocs/public;
```

---

# 5. SCRIPT_FILENAME Salah

Gejala.

```
Primary script unknown
```

atau.

```
No input file specified
```

## Penyebab

Path yang digunakan adalah path host.

## Solusi

Gunakan path di dalam container.

```nginx
fastcgi_param SCRIPT_FILENAME \
/var/www/html/public/index.php;
```

---

# 6. Menggunakan TCP Padahal Menggunakan Unix Socket

Kesalahan.

```nginx
fastcgi_pass 127.0.0.1:9000;
```

Padahal PHP menggunakan Unix Socket.

## Solusi

```nginx
fastcgi_pass unix:/run/php/myapp.sock;
```

---

# 7. Socket Tidak Dibuat

Gejala.

```text
connect() to unix:/run/php/myapp.sock failed
```

## Penyebab

- PHP-FPM gagal berjalan.
- File pool salah.
- Socket tidak dibuat.

## Solusi

Periksa.

```bash
docker logs myapp-php
```

dan.

```bash
ls -l /run/php
```

---

# 8. Database Tidak Dapat Dihubungi

Gejala.

```
Connection refused
```

atau.

```
Access denied
```

## Solusi

Pastikan.

- MariaDB berjalan.
- User database benar.
- Password benar.
- Port benar.
- Firewall mengizinkan koneksi.

---

# 9. Tidak Menggunakan host.docker.internal

Banyak administrator menggunakan alamat IP host secara langsung.

Akibatnya ketika server berubah, konfigurasi harus diubah kembali.

## Solusi

Tambahkan.

```yaml
extra_hosts:

- "host.docker.internal:host-gateway"
```

Kemudian gunakan.

```text
host.docker.internal
```

pada konfigurasi database.

---

# 10. Menggunakan Root Database

Kesalahan.

```text
root
```

digunakan oleh aplikasi.

## Solusi

Gunakan user database khusus.

```
myapp
```

Berikan hak akses hanya pada satu database.

---

# 11. Tidak Mengaktifkan Health Check

Tanpa health check Docker tidak mengetahui apakah PHP-FPM masih berjalan.

## Solusi

```yaml
healthcheck:

  test:
    ["CMD-SHELL","pidof php-fpm || exit 1"]
```

---

# 12. Log Docker Terlalu Besar

Gejala.

Disk penuh.

## Solusi

Aktifkan log rotation.

```yaml
logging:

  driver: json-file

  options:

    max-size: "10m"

    max-file: "5"
```

---

# 13. Tidak Memisahkan Konfigurasi

Banyak administrator mencampur:

- Docker Compose
- PHP
- Nginx

ke dalam satu direktori.

## Solusi

Pisahkan.

```text
/opt/docker/apps/myapp/

/opt/docker/images/php8.3/

/var/apps/myapp/
```

---

# 14. Menyimpan Password di Git

Contoh.

```
.env
```

ikut masuk repository.

## Solusi

Tambahkan ke `.gitignore`.

```text
.env
```

Gunakan kredensial yang berbeda untuk setiap lingkungan.

---

# 15. Tidak Melakukan Backup

Deployment sudah berjalan.

Tetapi backup belum ada.

Padahal kegagalan disk dapat terjadi kapan saja.

## Solusi

Backup secara terpisah.

- Database
- Source Code
- Writable
- Konfigurasi Docker
- Konfigurasi Nginx

---

# 16. Tidak Melakukan Uji Restore

Banyak administrator hanya memiliki file backup.

Namun belum pernah melakukan restore.

Backup yang tidak pernah diuji belum tentu dapat digunakan.

---

# 17. Menggunakan Container Sebagai Tempat Menyimpan Data

Kesalahan.

Upload disimpan di dalam filesystem container.

Akibatnya seluruh file hilang ketika container dihapus.

## Solusi

Gunakan bind mount.

```yaml
- /var/apps/myapp/data/writable:/var/www/html/writable
```

---

# 18. Tidak Menggunakan HTTPS

Aplikasi produksi masih berjalan menggunakan HTTP.

## Solusi

Gunakan HTTPS dan aktifkan redirect dari HTTP ke HTTPS.

---

# 19. Tidak Memantau Resource

Server tampak normal.

Namun ternyata RAM hampir habis.

## Solusi

Pantau secara berkala.

```bash
docker stats
```

---

# 20. Tidak Mendokumentasikan Konfigurasi

Ketika administrator berganti, tidak ada dokumentasi.

Akibatnya proses pemeliharaan menjadi lebih sulit.

Dokumentasikan:

- Struktur direktori
- Docker Compose
- Nginx
- PHP
- Backup
- Monitoring
- Jadwal pemeliharaan

---

# Ringkasan

| Masalah | Dampak | Solusi |
|----------|---------|--------|
| Writable tidak dipisahkan | Risiko modifikasi source | Pisahkan writable |
| Permission salah | Upload gagal | Gunakan `www-data` |
| Document Root salah | Source terekspos | Gunakan `public` |
| Socket salah | 502 Bad Gateway | Gunakan Unix Socket |
| Database salah | Aplikasi gagal | Periksa konfigurasi |
| Tidak ada backup | Sulit pemulihan | Backup berkala |
| Tidak ada monitoring | Gangguan terlambat diketahui | Aktifkan monitoring |

---

# Kesimpulan

Sebagian besar masalah deployment bukan disebabkan oleh Docker atau CodeIgniter 4, melainkan oleh konfigurasi yang kurang tepat.

Dengan menerapkan struktur direktori yang konsisten, memisahkan source code dan data runtime, menggunakan Unix Socket, mengatur permission dengan benar, serta melengkapi server dengan monitoring dan backup, proses deployment akan menjadi lebih stabil dan mudah dipelihara.

Troubleshooting yang sistematis dan dokumentasi yang baik juga akan mempercepat penyelesaian masalah ketika terjadi gangguan pada lingkungan produksi.

---
{{< saweria >}}