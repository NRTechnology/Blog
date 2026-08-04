---
title: "Hardening Nginx untuk Aplikasi PHP pada Lingkungan Produksi"
slug: "hardening-nginx-aplikasi-php"
date: 2026-08-03T21:00:00+07:00
lastmod: 2026-08-03T21:00:00+07:00
draft: false

description: "Panduan lengkap melakukan hardening Nginx untuk aplikasi PHP agar lebih aman dari web shell, file disclosure, directory traversal, dan berbagai serangan umum."

author: "NR Technology"

series:
  - Membangun Platform Produksi CodeIgniter 4
weight: 6

tags:
  - Nginx
  - PHP
  - Docker
  - Security
  - Hardening
  - DevOps

categories:
  - Nginx
  - Security
  - Docker

keywords:
  - nginx hardening
  - nginx security
  - php security
  - codeigniter security
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

# Hardening Nginx untuk Aplikasi PHP pada Lingkungan Produksi

Pada artikel sebelumnya kita telah mengkonfigurasi Nginx sebagai web server untuk CodeIgniter 4 menggunakan Docker PHP-FPM.

Konfigurasi tersebut sudah cukup untuk menjalankan aplikasi, tetapi belum cukup aman apabila server diakses langsung melalui internet.

Pada artikel ini kita akan membahas berbagai teknik hardening yang saya gunakan pada server produksi agar risiko kompromi aplikasi dapat diminimalkan.

---

# Mengapa Hardening Diperlukan?

Sebagian besar serangan terhadap aplikasi web bukan berasal dari kelemahan Nginx, melainkan dari:

- Upload Web Shell
- File Disclosure
- Directory Traversal
- Local File Inclusion
- Remote File Inclusion
- Source Code Disclosure
- Salah konfigurasi aplikasi

Hardening bertujuan memperkecil **attack surface** sehingga walaupun aplikasi memiliki kelemahan, peluang keberhasilan serangan dapat dikurangi.

---

# Prinsip Hardening

Prinsip yang saya gunakan cukup sederhana.

> Jangan pernah memberikan akses yang sebenarnya tidak dibutuhkan aplikasi.

Artinya:

- Jangan membuka direktori yang tidak digunakan.
- Jangan mengizinkan eksekusi file yang tidak diperlukan.
- Jangan menampilkan informasi server.
- Jangan mengekspos file konfigurasi.

---

# 1. Gunakan Document Root yang Benar

Selalu arahkan Document Root ke direktori:

```text
public/
```

Contoh:

```nginx
root /var/apps/myapp/htdocs/public;
```

Jangan pernah menggunakan:

```text
/var/apps/myapp/htdocs
```

Karena seluruh source code akan dapat diakses apabila terjadi kesalahan konfigurasi.

---

# 2. Nonaktifkan Informasi Versi

Secara default Nginx dapat mengirimkan informasi versinya.

Matikan dengan:

```nginx
server_tokens off;
```

Sebelum:

```
Server: nginx/1.26.1
```

Sesudah:

```
Server: nginx
```

Informasi yang lebih sedikit akan menyulitkan proses fingerprinting oleh penyerang.

---

# 3. Hanya Izinkan index.php

Kesalahan yang paling sering ditemukan adalah:

```nginx
location ~ \.php$
```

Konfigurasi tersebut akan mengeksekusi seluruh file PHP.

Misalnya:

```
info.php
shell.php
upload.php
tes.php
```

Saya lebih menyarankan:

```nginx
location = /index.php {

    include fastcgi_params;

    fastcgi_pass unix:/run/php/myapp.sock;

}
```

Kemudian blok seluruh file PHP lainnya.

```nginx
location ~ \.php$ {

    return 404;

}
```

Pendekatan ini jauh lebih aman.

---

# 4. Blok Hidden File

Banyak aplikasi menyimpan file tersembunyi.

Misalnya:

```
.env
.git
.htaccess
```

Blok seluruh hidden file.

```nginx
location ~ /\.(?!well-known).* {

    deny all;

}
```

---

# 5. Blok File Composer

File berikut tidak perlu diakses publik.

```
composer.json

composer.lock
```

Tambahkan.

```nginx
location ~* ^/(composer\.(json|lock))$ {

    deny all;

}
```

---

# 6. Blok File Development

File berikut sebaiknya tidak pernah dapat diakses.

```
spark

phpunit.xml

README.md

LICENSE

preload.php
```

Contoh.

```nginx
location ~* ^/(spark|README\.md|LICENSE|phpunit\.xml(\.dist)?|preload\.php)$ {

    deny all;

}
```

---

# 7. Blok File Backup

Administrator sering lupa menghapus file backup.

Misalnya.

```
config.php.bak

db.sql

backup.zip

old.tar.gz
```

Tambahkan.

```nginx
location ~* \.(bak|old|orig|save|zip|7z|rar|tar|gz|sql)$ {

    deny all;

}
```

---

# 8. Blok Repository Git

Kesalahan yang cukup sering terjadi.

```
.git/

.svn/

.hg/
```

Blok seluruhnya.

```nginx
location ~ /\.(git|svn|hg) {

    deny all;

}
```

---

# 9. Blok File Environment

Pastikan file berikut tidak pernah dapat diakses.

```
.env

env
```

Tambahkan.

```nginx
location ~* ^/(env|\.env)$ {

    deny all;

}
```

---

# 10. Batasi Ukuran Upload

Batasi ukuran upload sesuai kebutuhan aplikasi.

```nginx
client_max_body_size 100M;
```

Jangan menggunakan nilai terlalu besar apabila tidak diperlukan.

---

# 11. Atur Timeout

```nginx
client_body_timeout 30s;

client_header_timeout 30s;

send_timeout 30s;

keepalive_timeout 30s;
```

Timeout membantu mengurangi risiko koneksi yang menggantung terlalu lama.

---

# 12. Tambahkan Security Header

Header berikut cukup direkomendasikan.

```nginx
add_header X-Frame-Options "SAMEORIGIN" always;

add_header X-Content-Type-Options "nosniff" always;

add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

Header tersebut membantu meningkatkan perlindungan pada sisi browser.

---

# 13. Pisahkan Writable

Direktori:

```
writable/
```

tidak boleh berada di dalam Document Root.

Struktur yang saya gunakan.

```text
htdocs/
└── public/

data/
└── writable/
```

Dengan demikian file upload tidak pernah dapat diakses secara langsung.

---

# 14. Source Code Read Only

Container PHP dijalankan menggunakan.

```yaml
read_only: true
```

Keuntungannya.

- Tidak dapat memodifikasi framework.
- Sulit menanamkan web shell.
- Source code tetap bersih.

---

# 15. Gunakan Unix Socket

Komunikasi antara Nginx dan PHP-FPM menggunakan.

```text
/run/php/myapp.sock
```

Keuntungan.

- Tidak membuka port.
- Lebih cepat.
- Lebih aman.

---

# 16. Gunakan no-new-privileges

Docker menyediakan fitur.

```yaml
security_opt:

- no-new-privileges:true
```

Konfigurasi ini mencegah proses memperoleh privilege tambahan.

---

# 17. Gunakan Health Check

Container sebaiknya memiliki health check.

```yaml
healthcheck:

test:

["CMD-SHELL","pidof php-fpm || exit 1"]
```

Administrator dapat mengetahui apabila PHP-FPM berhenti bekerja.

---

# 18. Aktifkan Log Rotation

Jangan biarkan log Docker tumbuh tanpa batas.

```yaml
logging:

driver: json-file

options:

max-size: "10m"

max-file: "5"
```

---

# 19. Gunakan HTTPS

Seluruh aplikasi produksi sebaiknya menggunakan HTTPS.

Contoh.

```
443/TCP
```

Gunakan sertifikat dari Let's Encrypt atau CA resmi.

---

# 20. Tambahkan Web Application Firewall

Hardening Nginx bukanlah pengganti WAF.

Saya sangat menyarankan menambahkan Web Application Firewall seperti ModSecurity untuk membantu mendeteksi dan memblokir berbagai serangan umum, misalnya SQL Injection, Cross-Site Scripting (XSS), Remote Code Execution (RCE), dan pola serangan lainnya.

WAF bekerja sebagai lapisan perlindungan tambahan yang melengkapi konfigurasi keamanan pada Nginx dan aplikasi.

---

# Checklist Hardening

Sebelum aplikasi dipublikasikan ke internet, pastikan seluruh poin berikut sudah diterapkan.

| Hardening | Status |
|-----------|--------|
| Document Root ke public | ✅ |
| Source Read Only | ✅ |
| Writable Dipisahkan | ✅ |
| Unix Socket | ✅ |
| Hidden File Diblokir | ✅ |
| Composer Diblokir | ✅ |
| Git Diblokir | ✅ |
| Backup Diblokir | ✅ |
| Hanya index.php | ✅ |
| Security Header | ✅ |
| Log Rotation | ✅ |
| HTTPS | ✅ |
| Health Check | ✅ |
| no-new-privileges | ✅ |

---

# Kesimpulan

Hardening merupakan bagian penting dari proses deployment aplikasi web.

Tidak ada satu konfigurasi yang mampu menghentikan seluruh serangan, tetapi kombinasi berbagai teknik sederhana seperti membatasi eksekusi PHP, memisahkan source code dan direktori writable, menggunakan Unix Socket, memblokir file sensitif, serta menerapkan prinsip *least privilege* akan secara signifikan mengurangi permukaan serangan.

Pada artikel berikutnya kita akan membahas bagaimana memisahkan **source code** dan **direktori writable** pada CodeIgniter 4 sehingga proses deployment menjadi lebih aman, lebih mudah dipelihara, dan lebih sesuai untuk lingkungan produksi.

---
{{< saweria >}}