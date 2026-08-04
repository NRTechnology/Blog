---
title: "Migrasi Open Journal Systems (OJS) dari Server Lama ke Server Baru"
date: 2026-08-02
draft: false
description: "Panduan lengkap migrasi Open Journal Systems (OJS) dari server lama ke server baru menggunakan arsitektur Nginx, Docker PHP-FPM, dan MariaDB."
tags:
  - OJS
  - Open Journal Systems
  - Migration
  - Linux
  - Docker
  - MariaDB
  - Nginx
categories:
  - OJS
  - Linux

series:
  - "Membangun Open Journal Systems (OJS) 3.4"
weight: 8

author: "NR Technology"
cover:
  image: "ojs-cover.png"
  alt: "Open Journal Systems (OJS) 3.4"
  caption: "Seri Membangun Open Journal Systems (OJS) 3.4"
---

# Migrasi Open Journal Systems (OJS) dari Server Lama ke Server Baru

## Pendahuluan

Migrasi merupakan proses memindahkan layanan Open Journal Systems dari satu server ke server lainnya tanpa kehilangan data maupun konfigurasi aplikasi.

Migrasi dapat dilakukan karena berbagai alasan, seperti peningkatan kapasitas server, penggantian sistem operasi, perubahan arsitektur aplikasi, maupun konsolidasi infrastruktur.

Apabila dilakukan dengan perencanaan yang baik, migrasi dapat berlangsung dengan downtime yang singkat serta risiko kehilangan data yang sangat kecil.

Artikel ini membahas proses migrasi Open Journal Systems menggunakan pendekatan yang sistematis sehingga dapat diterapkan pada lingkungan produksi.

---

# Kapan Migrasi Diperlukan?

Migrasi tidak selalu dilakukan karena server mengalami kerusakan.

Beberapa kondisi yang umum menyebabkan migrasi antara lain.

- Penggantian server lama.
- Upgrade spesifikasi perangkat keras.
- Migrasi ke Virtual Machine baru.
- Migrasi ke lingkungan Docker.
- Perubahan sistem operasi.
- Konsolidasi beberapa server menjadi satu.
- Perpindahan Data Center.
- Modernisasi infrastruktur.

Migrasi juga sering dilakukan sebelum masa dukungan sistem operasi berakhir agar aplikasi tetap memperoleh pembaruan keamanan.

---

# Tujuan Migrasi

Migrasi bertujuan memastikan layanan tetap dapat digunakan dengan gangguan seminimal mungkin.

Beberapa tujuan utama migrasi meliputi.

- Memindahkan seluruh data.
- Memindahkan konfigurasi aplikasi.
- Mempertahankan struktur database.
- Mempertahankan dokumen jurnal.
- Meminimalkan downtime.
- Menjamin integritas data.
- Menyiapkan lingkungan yang lebih mudah dipelihara.

Migrasi yang berhasil tidak hanya memindahkan aplikasi, tetapi juga memastikan seluruh fungsi tetap berjalan seperti sebelum proses migrasi.

---

# Jenis Migrasi

Secara umum terdapat beberapa pendekatan migrasi.

## Server Replacement

Server lama diganti sepenuhnya oleh server baru.

Seluruh data dipindahkan kemudian layanan dialihkan ke server baru.

---

## In-Place Migration

Migrasi dilakukan pada server yang sama dengan memperbarui komponen tertentu.

Sebagai contoh.

- Upgrade sistem operasi.
- Upgrade PHP.
- Upgrade MariaDB.

Pendekatan ini memiliki downtime yang relatif lebih singkat, namun risiko perubahan terhadap sistem yang sedang berjalan lebih besar.

---

## Parallel Migration

Server baru dibangun secara penuh tanpa mengganggu server lama.

Seluruh data dipindahkan ke server baru.

Setelah proses validasi selesai, lalu lintas dialihkan menuju server baru.

Pendekatan ini merupakan metode yang paling direkomendasikan karena memungkinkan proses pengujian dilakukan sebelum layanan dipindahkan.

---

# Arsitektur Migrasi

Artikel ini menggunakan pendekatan **Parallel Migration**.

```text
                 Server Lama
                      │
      ┌───────────────┼────────────────┐
      │               │                │
      ▼               ▼                ▼
 Database        Source Code       ojsdata
      │               │                │
      └───────────────┼────────────────┘
                      │
                 Backup Data
                      │
                      ▼
                 Server Baru
      ┌───────────────┼────────────────┐
      │               │                │
      ▼               ▼                ▼
 Database        Source Code       ojsdata
                      │
                      ▼
                 Validasi Sistem
                      │
                      ▼
                   Cut Over
```

Pendekatan ini memungkinkan administrator melakukan seluruh pengujian sebelum pengguna diarahkan ke server baru.

---

# Komponen yang Dimigrasikan

Migrasi Open Journal Systems tidak hanya memindahkan database.

Seluruh komponen berikut harus ikut dipindahkan.

## Database

Database menyimpan.

- Pengguna.
- Artikel.
- Metadata.
- Workflow.
- Konfigurasi.

---

## Source Code

Source code harus sesuai dengan versi database yang digunakan.

Perbedaan versi source code dapat menyebabkan aplikasi gagal dijalankan.

---

## Direktori ojsdata

Direktori `ojsdata` berisi.

- Artikel PDF.
- Lampiran.
- Dataset.
- Cover Journal.
- File Submission.

Direktori ini merupakan bagian yang tidak boleh terlewat dalam proses migrasi.

---

## File Konfigurasi

File.

```text
config.inc.php
```

berisi konfigurasi penting.

- Database.
- Base URL.
- Lokasi Upload.
- Email.
- Session.

Setelah dipindahkan, beberapa parameter mungkin perlu disesuaikan dengan lingkungan server baru.

---

## Plugin

Apabila menggunakan plugin tambahan, plugin tersebut juga harus dipindahkan ke server baru.

Pastikan plugin kompatibel dengan versi Open Journal Systems yang digunakan.

---

## Theme

Apabila menggunakan theme khusus, theme tersebut harus ikut dimigrasikan.

Tanpa theme yang sesuai, tampilan website dapat berubah setelah migrasi selesai.

---

# Prasyarat Migrasi

Sebelum memulai migrasi, pastikan beberapa kondisi berikut telah terpenuhi.

- Backup telah dibuat.
- Backup telah diverifikasi.
- Server baru telah disiapkan.
- Ruang penyimpanan mencukupi.
- Hak akses administrator tersedia.
- Jadwal migrasi telah ditentukan.

Migrasi sebaiknya dilakukan pada periode dengan aktivitas pengguna yang rendah untuk mengurangi dampak terhadap layanan.

---

# Checklist Persiapan

Sebelum memulai migrasi, lakukan pemeriksaan berikut.

- Inventarisasi server lama.
- Catat versi Open Journal Systems.
- Catat versi PHP.
- Catat versi MariaDB.
- Catat plugin yang digunakan.
- Catat theme yang digunakan.
- Pastikan backup terbaru tersedia.
- Pastikan prosedur rollback telah disiapkan.

Checklist tersebut membantu mengurangi kemungkinan adanya komponen yang terlewat selama proses migrasi.

---

# Menentukan Downtime

Migrasi hampir selalu memerlukan periode downtime, meskipun hanya dalam waktu singkat.

Administrator perlu menentukan.

- Waktu mulai migrasi.
- Estimasi selesai.
- Waktu Cut Over.
- Waktu validasi.
- Waktu layanan kembali dibuka.

Informasi tersebut sebaiknya disampaikan kepada pengelola jurnal sebelum proses migrasi dimulai.

---

# Strategi Cut Over

Cut Over merupakan proses mengalihkan layanan dari server lama ke server baru.

Pada artikel ini digunakan pendekatan berikut.

```text
Server Lama

↓

Freeze Perubahan

↓

Backup Terakhir

↓

Sinkronisasi Data

↓

Validasi

↓

Cut Over

↓

Server Baru
```

Dengan pendekatan tersebut, data yang berpindah merupakan data terbaru sehingga risiko kehilangan transaksi menjadi sangat kecil.

---

# Ruang Lingkup Artikel

Artikel ini akan membahas seluruh tahapan migrasi Open Journal Systems.

- Inventarisasi server lama.
- Persiapan server baru.
- Backup sebelum migrasi.
- Sinkronisasi data.
- Restore pada server baru.
- Validasi aplikasi.
- Cut Over.
- Rollback Plan.
- Monitoring pasca migrasi.
- Best Practices.

Seluruh contoh menggunakan placeholder sehingga dapat diterapkan pada berbagai lingkungan tanpa membawa informasi sensitif dari server produksi.

Pada bagian berikutnya kita akan melakukan inventarisasi server lama, mengidentifikasi versi Open Journal Systems beserta seluruh komponen pendukungnya, serta menyiapkan backup terakhir sebelum proses migrasi dimulai.

---

# Inventarisasi Server Lama

Sebelum melakukan migrasi, administrator harus mengetahui kondisi server lama secara menyeluruh.

Inventarisasi bertujuan untuk memastikan tidak ada komponen penting yang terlewat selama proses migrasi.

Seluruh informasi yang diperoleh pada tahap ini akan menjadi acuan ketika membangun server baru.

Lakukan inventarisasi sebelum melakukan backup terakhir.

---

# Mengidentifikasi Versi Open Journal Systems

Langkah pertama adalah memastikan versi Open Journal Systems yang sedang digunakan.

Versi aplikasi dapat dilihat melalui Dashboard Administrator.

Sebagai alternatif, periksa file berikut.

```bash
cat \
/var/apps/ojs/htdocs/lib/pkp/includes/version.inc.php
```

Catat informasi berikut.

- Versi OJS.
- Build Number.
- Release.

Versi tersebut akan menentukan kompatibilitas plugin, database, maupun proses upgrade di masa mendatang.

---

# Mengidentifikasi Versi PHP

Periksa versi PHP yang digunakan.

Apabila menggunakan Docker PHP-FPM.

```bash
docker exec ojs-php php -v
```

Contoh.

```text
PHP 8.x.x
```

Selanjutnya periksa extension PHP.

```bash
docker exec ojs-php php -m
```

Pastikan seluruh extension yang dibutuhkan OJS tersedia.

Versi PHP pada server baru sebaiknya tetap kompatibel dengan versi Open Journal Systems yang akan dimigrasikan.

---

# Mengidentifikasi Versi MariaDB

Periksa versi MariaDB.

```bash
mysql -V
```

atau.

```bash
mariadb --version
```

Selanjutnya masuk ke database.

```bash
mysql -u root -p
```

Kemudian.

```sql
SELECT VERSION();
```

Catat versi database karena dapat memengaruhi kompatibilitas ketika dilakukan restore pada server baru.

---

# Mengidentifikasi Sistem Operasi

Periksa sistem operasi.

```bash
cat /etc/os-release
```

Catat informasi berikut.

- Distribusi Linux.
- Versi.
- Nama Rilis.

Informasi tersebut berguna apabila administrator ingin membangun server baru dengan spesifikasi yang sama.

---

# Mengidentifikasi Web Server

Periksa versi Nginx.

```bash
nginx -v
```

Apabila menggunakan Reverse Proxy tambahan, dokumentasikan juga konfigurasi tersebut.

Informasi ini diperlukan agar perilaku aplikasi pada server baru tetap konsisten.

---

# Mengidentifikasi Struktur Direktori

Dokumentasikan struktur direktori yang digunakan.

Contoh.

```text
/var/apps/ojs

├── backup
├── data
│   └── ojsdata
├── htdocs
└── logs
```

Selanjutnya catat lokasi.

- Source Code.
- Upload Directory.
- Backup.
- Log.
- SSL Certificate (apabila dikelola pada server yang sama).

Inventarisasi struktur direktori akan mempercepat proses migrasi.

---

# Mengidentifikasi Database

Masuk ke MariaDB.

```bash
mysql -u root -p
```

Lihat daftar database.

```sql
SHOW DATABASES;
```

Pastikan nama database yang digunakan oleh Open Journal Systems.

Kemudian.

```sql
USE db_ojs;
```

Periksa jumlah tabel.

```sql
SHOW TABLES;
```

Jumlah tabel yang tidak sesuai dapat mengindikasikan database belum lengkap atau mengalami kerusakan.

---

# Mengidentifikasi Pengguna Database

Periksa akun yang digunakan oleh Open Journal Systems.

```sql
SELECT
User,
Host
FROM mysql.user;
```

Selanjutnya periksa hak akses pengguna.

```sql
SHOW GRANTS FOR 'ojs_user'@'%';
```

Pastikan akun tersebut memiliki hak akses terhadap database OJS.

Hak akses tersebut akan dibuat kembali pada server baru.

---

# Mengidentifikasi Direktori Upload

Periksa lokasi upload pada file konfigurasi.

```bash
grep "^files_dir" \
/var/apps/ojs/htdocs/config.inc.php
```

Contoh.

```text
files_dir = /var/apps/ojs/data/ojsdata
```

Pastikan direktori tersebut benar-benar ada.

```bash
ls -lah \
/var/apps/ojs/data/ojsdata
```

Direktori upload merupakan salah satu komponen terpenting yang harus ikut dimigrasikan.

---

# Mengidentifikasi Base URL

Periksa Base URL.

```bash
grep "^base_url" \
/var/apps/ojs/htdocs/config.inc.php
```

Contoh.

```text
base_url = "https://jurnal.example.go.id"
```

Informasi ini diperlukan ketika melakukan validasi setelah migrasi selesai.

---

# Mengidentifikasi Plugin

Masuk ke direktori plugin.

```bash
ls \
/var/apps/ojs/htdocs/plugins
```

Dokumentasikan plugin tambahan yang digunakan.

Pastikan.

- Plugin masih aktif.
- Plugin kompatibel.
- Plugin berasal dari sumber terpercaya.

Plugin pihak ketiga sebaiknya ikut dibackup sebelum migrasi dilakukan.

---

# Mengidentifikasi Theme

Periksa theme.

```bash
ls \
/var/apps/ojs/htdocs/plugins/themes
```

Apabila menggunakan theme khusus, pastikan seluruh file theme ikut dimigrasikan.

Theme yang tidak dipindahkan dapat menyebabkan perubahan tampilan website.

---

# Mengidentifikasi Scheduled Task

Periksa Cron Job.

```bash
crontab -l
```

Catat Scheduled Task yang berkaitan dengan Open Journal Systems.

Contohnya.

- Scheduled Tasks OJS.
- Backup Otomatis.
- Sinkronisasi.
- Monitoring.

Konfigurasi tersebut perlu diterapkan kembali pada server baru.

---

# Mengidentifikasi Sertifikat SSL

Apabila SSL dikelola pada server yang sama.

Catat.

- Lokasi sertifikat.
- Lokasi private key.
- Tanggal kedaluwarsa.

Apabila sertifikat dikelola oleh Reverse Proxy atau Load Balancer, dokumentasikan mekanisme tersebut.

---

# Mengidentifikasi Konfigurasi Nginx

Periksa Virtual Host.

```bash
ls \
/etc/nginx/sites-enabled
```

Kemudian buka konfigurasi.

```bash
cat \
/etc/nginx/sites-available/ojs.conf
```

Dokumentasikan.

- Server Name.
- Root Directory.
- FastCGI.
- SSL.
- Redirect.
- Header Security.

Konfigurasi ini akan diterapkan kembali pada server baru.

---

# Melakukan Backup Sebelum Migrasi

Setelah seluruh inventarisasi selesai, lakukan backup terakhir.

Minimal terdiri atas.

- Database.
- config.inc.php.
- ojsdata.
- Plugin.
- Theme.
- Source Code.

Backup terakhir menjadi titik pemulihan apabila migrasi harus dibatalkan.

---

# Freeze Perubahan

Sebelum backup terakhir dibuat, hentikan sementara aktivitas pada Open Journal Systems.

Tujuannya adalah memastikan tidak ada perubahan data selama proses backup.

Administrator sebaiknya menginformasikan jadwal migrasi kepada pengguna.

Selama periode tersebut.

- Jangan membuat submission baru.
- Jangan melakukan review.
- Jangan mengubah konfigurasi.
- Jangan memasang plugin baru.

Dengan demikian backup terakhir akan berada pada kondisi yang konsisten.

---

# Menyiapkan Dokumentasi Migrasi

Sebelum melanjutkan ke server baru, siapkan dokumentasi yang berisi.

- Versi OJS.
- Versi PHP.
- Versi MariaDB.
- Versi Nginx.
- Nama Database.
- Nama User Database.
- Lokasi Source Code.
- Lokasi ojsdata.
- Lokasi Backup.
- Daftar Plugin.
- Daftar Theme.
- Konfigurasi Reverse Proxy.
- Konfigurasi Cron Job.

Dokumentasi ini akan menjadi referensi utama selama proses migrasi berlangsung.

---

# Ringkasan

Pada bagian ini telah dilakukan inventarisasi terhadap seluruh komponen pada server lama.

- Versi Open Journal Systems.
- PHP.
- MariaDB.
- Sistem Operasi.
- Nginx.
- Struktur Direktori.
- Database.
- User Database.
- Direktori Upload.
- Plugin.
- Theme.
- Scheduled Task.
- SSL.
- Backup Terakhir.

Inventarisasi merupakan langkah penting untuk memastikan seluruh komponen yang dibutuhkan telah terdokumentasi sebelum proses migrasi dimulai.

Pada bagian berikutnya kita akan membangun server baru, menyiapkan lingkungan aplikasi, melakukan restore seluruh komponen Open Journal Systems, serta menyiapkan server baru agar siap menerima proses Cut Over.

---

# Menyiapkan Server Baru

Setelah inventarisasi server lama selesai dan seluruh backup telah diverifikasi, langkah berikutnya adalah menyiapkan server baru.

Server baru sebaiknya dibangun menggunakan arsitektur yang sama atau lebih baik dibandingkan server lama.

Pada seri artikel ini digunakan arsitektur.

```text
Internet
    │
    ▼
Nginx
    │
    ▼
Docker PHP-FPM
    │
    ▼
MariaDB
```

Dengan pendekatan ini, proses migrasi menjadi lebih mudah dan pemeliharaan sistem menjadi lebih sederhana.

---

# Menyiapkan Struktur Direktori

Sebelum melakukan restore, buat struktur direktori yang akan digunakan.

Contoh.

```text
/var/apps/ojs

├── backup
├── data
│   └── ojsdata
├── htdocs
└── logs
```

Apabila menggunakan Docker, struktur direktori yang konsisten akan mempermudah proses mounting volume.

---

# Memastikan Server Siap

Pastikan seluruh layanan telah berjalan.

Periksa Nginx.

```bash
systemctl status nginx
```

Periksa MariaDB.

```bash
systemctl status mariadb
```

Periksa Docker.

```bash
docker ps
```

Seluruh layanan harus berada pada kondisi aktif sebelum proses restore dimulai.

---

# Menyiapkan Database

Masuk ke MariaDB.

```bash
mysql -u root -p
```

Buat database.

```sql
CREATE DATABASE db_ojs
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;
```

Buat pengguna database.

```sql
CREATE USER 'ojs_user'@'%'
IDENTIFIED BY 'StrongPassword';
```

Berikan hak akses.

```sql
GRANT ALL PRIVILEGES
ON db_ojs.*
TO 'ojs_user'@'%';
```

Kemudian.

```sql
FLUSH PRIVILEGES;
```

Keluar.

```sql
EXIT;
```

Pastikan nama database dan akun sesuai dengan konfigurasi yang akan digunakan.

---

# Menempatkan Source Code

Salin source code Open Journal Systems ke server baru.

Apabila menggunakan arsip.

```bash
tar -xzf \
backup-source.tar.gz \
-C /
```

Apabila menggunakan Git atau paket resmi, ekstrak source code pada lokasi yang sama dengan struktur direktori sebelumnya.

Contoh.

```text
/var/apps/ojs/htdocs
```

Selanjutnya periksa.

```bash
ls \
/var/apps/ojs/htdocs
```

Pastikan file seperti.

```text
index.php

lib/

plugins/

classes/
```

telah tersedia.

---

# Restore Database

Restore database menggunakan backup terakhir.

Apabila menggunakan file SQL.

```bash
mysql \
-u root \
-p \
db_ojs \
< backup-db_ojs.sql
```

Apabila menggunakan file terkompresi.

```bash
gunzip -c \
backup-db_ojs.sql.gz \
| mysql \
-u root \
-p \
db_ojs
```

Tunggu hingga proses selesai.

Jangan melakukan perubahan terhadap database selama proses restore berlangsung.

---

# Memverifikasi Database

Masuk kembali ke MariaDB.

```bash
mysql -u root -p
```

Pilih database.

```sql
USE db_ojs;
```

Periksa tabel.

```sql
SHOW TABLES;
```

Selanjutnya lakukan pemeriksaan sederhana.

```sql
SELECT COUNT(*)
FROM users;
```

dan.

```sql
SELECT COUNT(*)
FROM submissions;
```

Pastikan data telah berhasil dipulihkan.

---

# Restore Direktori Upload

Ekstrak direktori `ojsdata`.

```bash
tar -xzf \
backup-ojsdata.tar.gz \
-C /
```

Periksa hasilnya.

```bash
ls -lah \
/var/apps/ojs/data/ojsdata
```

Pastikan struktur direktori sesuai dengan server lama.

---

# Restore File Konfigurasi

Salin file konfigurasi.

```bash
cp \
backup-config.inc.php \
/var/apps/ojs/htdocs/config.inc.php
```

Selanjutnya buka file tersebut.

```bash
nano \
/var/apps/ojs/htdocs/config.inc.php
```

Periksa kembali.

- Database Host.
- Database Name.
- Database Username.
- Base URL.
- Files Directory.

Sesuaikan apabila terdapat perubahan pada server baru.

---

# Restore Plugin

Apabila menggunakan plugin tambahan.

Ekstrak kembali.

```bash
tar -xzf \
backup-plugin.tar.gz \
-C /
```

Pastikan plugin berada pada direktori.

```text
plugins/
```

Lakukan pemeriksaan.

```bash
ls \
/var/apps/ojs/htdocs/plugins
```

---

# Restore Theme

Apabila menggunakan theme tambahan.

Ekstrak kembali.

```bash
tar -xzf \
backup-theme.tar.gz \
-C /
```

Periksa.

```bash
ls \
/var/apps/ojs/htdocs/plugins/themes
```

Pastikan seluruh theme telah berhasil dipindahkan.

---

# Mengatur Ownership

Seluruh file aplikasi sebaiknya dimiliki oleh user yang menjalankan PHP-FPM.

Contoh.

```bash
chown -R \
www-data:www-data \
/var/apps/ojs
```

Periksa hasilnya.

```bash
ls -lah \
/var/apps/ojs
```

Ownership yang benar akan menghindari berbagai masalah permission setelah migrasi.

---

# Mengatur Permission

Atur permission direktori.

```bash
find \
/var/apps/ojs/htdocs \
-type d \
-exec chmod 755 {} \;
```

Kemudian seluruh file.

```bash
find \
/var/apps/ojs/htdocs \
-type f \
-exec chmod 644 {} \;
```

Selanjutnya file konfigurasi.

```bash
chmod 640 \
/var/apps/ojs/htdocs/config.inc.php
```

Terakhir periksa direktori upload.

```bash
chown -R \
www-data:www-data \
/var/apps/ojs/data/ojsdata
```

Permission yang konsisten akan mempermudah proses validasi setelah migrasi.

---

# Memeriksa Konfigurasi Nginx

Pastikan Virtual Host telah dibuat.

Lakukan pengujian konfigurasi.

```bash
nginx -t
```

Apabila tidak terdapat kesalahan.

Reload konfigurasi.

```bash
systemctl reload nginx
```

Pastikan seluruh konfigurasi SSL, FastCGI, dan Reverse Proxy telah sesuai dengan server baru.

---

# Memeriksa PHP-FPM

Apabila menggunakan Docker.

Pastikan container berjalan.

```bash
docker ps
```

Masuk ke container.

```bash
docker exec -it ojs-php bash
```

Periksa versi PHP.

```bash
php -v
```

Selanjutnya periksa extension.

```bash
php -m
```

Pastikan seluruh extension yang dibutuhkan OJS tersedia.

---

# Validasi Awal

Sebelum dilakukan Cut Over, lakukan pemeriksaan awal.

Pastikan.

- Database berhasil direstore.
- Source code tersedia.
- `config.inc.php` telah sesuai.
- Direktori `ojsdata` lengkap.
- Plugin tersedia.
- Theme tersedia.
- Permission telah benar.
- Ownership telah benar.
- Nginx berjalan.
- PHP-FPM berjalan.
- MariaDB berjalan.

Server baru seharusnya sudah siap digunakan untuk proses pengujian sebelum layanan dialihkan dari server lama.

---

# Ringkasan

Pada bagian ini telah dibangun lingkungan Open Journal Systems pada server baru.

- Menyiapkan struktur direktori.
- Menyiapkan database.
- Restore source code.
- Restore database.
- Restore `ojsdata`.
- Restore `config.inc.php`.
- Restore plugin.
- Restore theme.
- Mengatur ownership.
- Mengatur permission.
- Memverifikasi Nginx.
- Memverifikasi PHP-FPM.
- Melakukan validasi awal.

Setelah seluruh tahapan tersebut selesai, server baru telah memiliki seluruh komponen Open Journal Systems dan siap memasuki tahap berikutnya, yaitu sinkronisasi data terakhir, proses **Cut Over**, validasi pasca migrasi, serta penyusunan **Rollback Plan** apabila terjadi kendala selama perpindahan layanan.

---

# Sinkronisasi Data Terakhir

Apabila server lama masih digunakan selama proses pembangunan server baru, kemungkinan terdapat perubahan data setelah backup pertama dibuat.

Contohnya.

- Submission baru.
- Perubahan metadata artikel.
- Proses review.
- Publikasi edisi baru.
- Penambahan pengguna.

Sebelum Cut Over dilakukan, administrator perlu melakukan sinkronisasi terakhir agar server baru memiliki data yang sama dengan server lama.

---

# Freeze Layanan

Sebelum sinkronisasi terakhir dilakukan, hentikan sementara aktivitas pada Open Journal Systems.

Administrator dapat mengumumkan jadwal pemeliharaan kepada pengguna.

Selama periode tersebut.

- Jangan membuat Submission baru.
- Jangan melakukan Review.
- Jangan mempublikasikan Issue baru.
- Jangan mengubah konfigurasi.
- Jangan menambah Plugin.
- Jangan mengubah Theme.

Freeze layanan bertujuan menjaga konsistensi data selama proses sinkronisasi.

---

# Membuat Backup Terakhir

Setelah seluruh aktivitas dihentikan, lakukan backup terakhir terhadap server lama.

Minimal terdiri atas.

- Database.
- config.inc.php.
- ojsdata.

Backup terakhir inilah yang akan digunakan pada proses sinkronisasi.

---

# Sinkronisasi Database

Restore kembali backup database terakhir ke server baru.

```bash
mysql \
-u root \
-p \
db_ojs \
< backup-db_ojs-final.sql
```

Apabila menggunakan file terkompresi.

```bash
gunzip -c \
backup-db_ojs-final.sql.gz \
| mysql \
-u root \
-p \
db_ojs
```

Dengan demikian seluruh perubahan terakhir ikut berpindah ke server baru.

---

# Sinkronisasi Direktori Upload

Selanjutnya sinkronkan kembali direktori.

```text
ojsdata
```

Apabila menggunakan arsip.

```bash
tar \
-xzf \
backup-ojsdata-final.tar.gz \
-C /
```

Pastikan seluruh file terbaru telah tersedia.

---

# Memastikan Konfigurasi Terbaru

Bandingkan file konfigurasi pada server lama dengan server baru.

Periksa terutama.

- Database.
- Base URL.
- Files Directory.
- Session.
- Email.

Apabila tidak terdapat perubahan konfigurasi sejak backup pertama, langkah ini hanya berfungsi sebagai verifikasi.

---

# Validasi Sebelum Cut Over

Sebelum layanan dialihkan ke server baru, lakukan pengujian secara menyeluruh.

Pastikan.

- Login Administrator berhasil.
- Dashboard dapat diakses.
- Submission dapat dibuka.
- PDF dapat diunduh.
- Upload berhasil.
- Plugin aktif.
- Theme tampil dengan benar.

Seluruh fungsi utama harus berjalan tanpa menghasilkan error.

---

# Melakukan Cut Over

Cut Over merupakan proses mengalihkan layanan dari server lama ke server baru.

Tahapan yang direkomendasikan.

```text
Freeze Layanan

↓

Backup Terakhir

↓

Sinkronisasi

↓

Validasi

↓

Cut Over

↓

Monitoring
```

Pendekatan tersebut meminimalkan risiko kehilangan data.

---

# Memperbarui DNS

Apabila menggunakan nama domain.

```text
jurnal.example.go.id
```

ubah DNS agar mengarah ke alamat IP server baru.

Contoh.

```text
Server Lama

203.0.113.10
```

menjadi.

```text
Server Baru

203.0.113.20
```

Perubahan DNS dapat memerlukan waktu propagasi tergantung nilai TTL yang digunakan.

---

# Alternatif Cut Over

Selain menggunakan perubahan DNS, Cut Over juga dapat dilakukan melalui.

- Load Balancer.
- Reverse Proxy.
- NAT.
- Virtual IP.

Metode yang digunakan bergantung pada arsitektur infrastruktur masing-masing.

---

# Memverifikasi DNS

Setelah perubahan dilakukan.

Periksa.

```bash
dig jurnal.example.go.id
```

atau.

```bash
nslookup jurnal.example.go.id
```

Pastikan alamat IP yang ditampilkan merupakan alamat server baru.

---

# Validasi Aplikasi

Setelah DNS mengarah ke server baru.

Lakukan pengujian ulang.

- Halaman utama.
- Login.
- Dashboard.
- Submission.
- Review.
- Issue.
- Download PDF.
- Upload Dokumen.

Pastikan seluruh fungsi bekerja sebagaimana mestinya.

---

# Memantau Log

Setelah Cut Over selesai.

Pantau log aplikasi.

Log Nginx.

```bash
tail -f \
/var/log/nginx/error.log
```

Log PHP.

```bash
docker logs -f ojs-php
```

Log MariaDB.

```bash
journalctl \
-u mariadb \
-f
```

Monitoring sebaiknya dilakukan selama beberapa jam pertama setelah migrasi.

---

# Memantau Performa

Selain memantau error, administrator juga perlu memperhatikan performa.

Periksa.

- Penggunaan CPU.
- Penggunaan Memori.
- Penggunaan Disk.
- Penggunaan Database.
- Waktu Respons Aplikasi.

Perubahan performa yang signifikan dapat mengindikasikan adanya konfigurasi yang perlu disesuaikan.

---

# Menyusun Rollback Plan

Setiap proses migrasi harus memiliki prosedur rollback.

Rollback dilakukan apabila server baru mengalami masalah yang tidak dapat diselesaikan dalam waktu yang dapat diterima.

Langkah rollback dapat berupa.

- Mengarahkan kembali DNS ke server lama.
- Mengaktifkan kembali layanan pada server lama.
- Membatalkan Cut Over.
- Menunda migrasi hingga masalah diperbaiki.

Rollback yang terdokumentasi akan mengurangi waktu pemulihan layanan.

---

# Kapan Rollback Dilakukan?

Rollback dapat dipertimbangkan apabila.

- Aplikasi tidak dapat diakses.
- Login gagal.
- Database mengalami inkonsistensi.
- Artikel tidak dapat diunduh.
- Upload gagal.
- Plugin penting tidak berfungsi.
- Terjadi kehilangan data.

Keputusan rollback sebaiknya diambil secepat mungkin agar dampak terhadap pengguna dapat diminimalkan.

---

# Dokumentasi Migrasi

Setelah migrasi selesai, dokumentasikan seluruh proses.

Minimal meliputi.

- Waktu mulai migrasi.
- Waktu selesai migrasi.
- Durasi downtime.
- Backup yang digunakan.
- Versi OJS.
- Versi PHP.
- Versi MariaDB.
- Kendala yang ditemukan.
- Solusi yang dilakukan.

Dokumentasi tersebut menjadi referensi apabila migrasi serupa dilakukan pada masa mendatang.

---

# Ringkasan

Pada bagian ini telah dibahas tahapan akhir proses migrasi.

- Sinkronisasi data terakhir.
- Freeze layanan.
- Backup terakhir.
- Validasi sebelum Cut Over.
- Cut Over.
- Perubahan DNS.
- Validasi aplikasi.
- Monitoring.
- Rollback Plan.
- Dokumentasi migrasi.

Dengan mengikuti tahapan tersebut, proses perpindahan layanan dari server lama ke server baru dapat dilakukan secara terstruktur, meminimalkan downtime, serta menjaga konsistensi data selama proses migrasi.

Pada bagian terakhir kita akan membahas optimasi pasca migrasi, monitoring jangka panjang, audit konfigurasi, checklist akhir, best practices, serta evaluasi hasil migrasi sebelum server lama dinonaktifkan.

---

# Optimasi Setelah Migrasi

Setelah proses migrasi selesai dan layanan telah dialihkan ke server baru, pekerjaan administrator belum selesai.

Tahap berikutnya adalah memastikan seluruh komponen berjalan secara optimal dan stabil.

Optimasi setelah migrasi bertujuan untuk.

- Memastikan seluruh layanan berjalan normal.
- Mengidentifikasi masalah yang mungkin tidak muncul saat pengujian awal.
- Menyesuaikan konfigurasi apabila diperlukan.
- Meningkatkan stabilitas sistem.

Tahap ini sangat penting terutama pada beberapa hari pertama setelah migrasi.

---

# Monitoring Layanan

Lakukan pemantauan terhadap seluruh layanan utama.

Pastikan.

- Nginx berjalan.
- PHP-FPM berjalan.
- MariaDB berjalan.
- Docker berjalan.
- Scheduled Task berjalan.

Periksa status layanan.

```bash
systemctl status nginx
```

```bash
systemctl status mariadb
```

```bash
docker ps
```

Seluruh layanan harus berada pada kondisi aktif.

---

# Monitoring Log

Selama beberapa hari pertama setelah migrasi, lakukan pemantauan log secara berkala.

Log Nginx.

```bash
tail -f \
/var/log/nginx/error.log
```

Log PHP.

```bash
docker logs -f ojs-php
```

Log MariaDB.

```bash
journalctl \
-u mariadb \
-f
```

Perhatikan apabila muncul.

- PHP Warning.
- PHP Fatal Error.
- Permission Denied.
- Database Error.
- Upload Error.

Semakin cepat masalah ditemukan, semakin mudah proses penanganannya.

---

# Monitoring Penggunaan Resource

Periksa penggunaan sumber daya server.

Beberapa komponen yang perlu dipantau.

- CPU.
- Memory.
- Disk.
- Network.
- Database.

Gunakan.

```bash
top
```

atau.

```bash
htop
```

Untuk kapasitas penyimpanan.

```bash
df -h
```

Monitoring resource membantu memastikan server baru memiliki kapasitas yang memadai.

---

# Memeriksa Scheduled Task

Pastikan Scheduled Task Open Journal Systems tetap berjalan.

Apabila menggunakan Cron.

```bash
crontab -l
```

Pastikan Scheduled Task yang diperlukan telah diterapkan pada server baru.

Selanjutnya lakukan pengujian manual apabila diperlukan.

---

# Menguji Fungsi Aplikasi

Lakukan pengujian terhadap seluruh fungsi utama.

Minimal meliputi.

- Login.
- Logout.
- Submission.
- Upload Artikel.
- Download Artikel.
- Workflow Editorial.
- Reviewer.
- Editor.
- Publikasi Issue.
- Pencarian Artikel.

Pengujian dilakukan menggunakan akun dengan role yang berbeda agar seluruh alur kerja dapat diverifikasi.

---

# Memastikan Email Berfungsi

Apabila Open Journal Systems menggunakan notifikasi email.

Lakukan pengujian.

- Reset Password.
- Notifikasi Submission.
- Notifikasi Reviewer.
- Notifikasi Editor.

Pastikan email berhasil dikirim tanpa menghasilkan error.

---

# Memastikan Plugin Berfungsi

Periksa seluruh plugin yang digunakan.

Pastikan.

- Plugin aktif.
- Tidak menghasilkan error.
- Konfigurasi tetap tersimpan.
- Berfungsi sebagaimana mestinya.

Plugin pihak ketiga memerlukan perhatian khusus karena kemungkinan memiliki ketergantungan terhadap versi PHP maupun OJS.

---

# Memastikan Theme Berfungsi

Periksa tampilan website.

Pastikan.

- Halaman utama tampil normal.
- Logo muncul.
- Menu berfungsi.
- CSS termuat.
- JavaScript berjalan.

Perubahan tampilan sering kali mengindikasikan adanya file theme yang belum ikut dimigrasikan.

---

# Audit Permission

Lakukan audit ulang terhadap permission.

Periksa.

```bash
ls -lah \
/var/apps/ojs
```

Kemudian.

```bash
find \
/var/apps/ojs \
-type f
```

Pastikan permission masih sesuai dengan kebijakan keamanan yang telah diterapkan.

---

# Audit Konfigurasi

Lakukan pemeriksaan ulang terhadap.

- config.inc.php.
- Nginx.
- PHP-FPM.
- MariaDB.
- Docker.
- Cron Job.

Pastikan tidak terdapat konfigurasi yang masih mengarah ke server lama.

---

# Menonaktifkan Server Lama

Jangan langsung menghapus server lama setelah migrasi selesai.

Biarkan server lama tetap tersedia selama masa observasi.

Setelah administrator yakin seluruh layanan berjalan dengan baik pada server baru, server lama dapat dinonaktifkan sesuai prosedur organisasi.

Apabila server lama akan dipensiunkan, pastikan seluruh data penting telah dipindahkan dan backup terakhir telah disimpan dengan aman.

---

# Evaluasi Migrasi

Setelah proses migrasi selesai, lakukan evaluasi.

Beberapa hal yang dapat dievaluasi.

- Durasi migrasi.
- Durasi downtime.
- Kendala yang ditemukan.
- Kendala yang berhasil diatasi.
- Prosedur yang dapat diperbaiki.

Evaluasi tersebut akan menjadi referensi apabila migrasi dilakukan kembali pada masa mendatang.

---

# Checklist Akhir

Sebelum migrasi dinyatakan selesai, pastikan seluruh poin berikut telah terpenuhi.

- Server baru berjalan normal.
- Database berhasil dipindahkan.
- Source code sesuai versi.
- `config.inc.php` sesuai.
- Direktori `ojsdata` lengkap.
- Plugin berfungsi.
- Theme berfungsi.
- HTTPS berjalan.
- Login berhasil.
- Upload berhasil.
- Download berhasil.
- Scheduled Task berjalan.
- Email berfungsi.
- Monitoring aktif.
- Backup terbaru tersedia.
- Dokumentasi migrasi telah diperbarui.

Checklist ini dapat digunakan sebagai bagian dari proses serah terima sistem.

---

# Best Practices

Berikut beberapa praktik terbaik yang direkomendasikan ketika melakukan migrasi Open Journal Systems.

- Bangun server baru sebelum melakukan Cut Over.
- Lakukan backup penuh sebelum migrasi.
- Verifikasi seluruh file backup.
- Gunakan versi OJS yang sama selama proses migrasi.
- Dokumentasikan seluruh konfigurasi.
- Lakukan pengujian menyeluruh sebelum mengubah DNS.
- Siapkan Rollback Plan.
- Pantau log setelah migrasi.
- Jangan langsung menghapus server lama.
- Simpan backup terakhir sebelum migrasi pada lokasi yang aman.

Penerapan praktik tersebut membantu mengurangi risiko kegagalan migrasi dan mempercepat proses pemulihan apabila terjadi kendala.

---

# Kesimpulan

Migrasi Open Journal Systems bukan sekadar memindahkan database atau source code, tetapi memindahkan seluruh ekosistem aplikasi secara utuh.

Keberhasilan migrasi bergantung pada perencanaan yang baik, inventarisasi yang lengkap, backup yang tervalidasi, proses restore yang benar, validasi aplikasi, serta monitoring setelah Cut Over.

Dengan pendekatan **Parallel Migration**, administrator dapat membangun server baru tanpa mengganggu layanan yang sedang berjalan, melakukan pengujian secara menyeluruh, kemudian mengalihkan layanan pada waktu yang telah direncanakan.

Pendekatan tersebut membantu meminimalkan downtime sekaligus menjaga integritas data selama proses perpindahan layanan.

---

# Ringkasan

Pada artikel ini telah dibahas seluruh tahapan migrasi Open Journal Systems dari server lama ke server baru.

- Perencanaan migrasi.
- Inventarisasi server lama.
- Persiapan server baru.
- Backup sebelum migrasi.
- Restore seluruh komponen.
- Sinkronisasi data terakhir.
- Cut Over.
- Validasi.
- Rollback Plan.
- Monitoring pasca migrasi.
- Audit konfigurasi.
- Checklist migrasi.
- Best Practices.

Migrasi yang direncanakan dengan baik akan menghasilkan proses perpindahan layanan yang lebih aman, terdokumentasi, mudah diaudit, dan meminimalkan gangguan terhadap pengguna.

---

# Artikel Selanjutnya

Pada seri berikutnya akan dibahas **Upgrade Open Journal Systems (OJS) 3.4**, meliputi persiapan sebelum upgrade, pemeriksaan kompatibilitas, backup sebelum upgrade, proses upgrade aplikasi, migrasi database, validasi pasca-upgrade, troubleshooting, serta praktik terbaik untuk melakukan upgrade pada lingkungan produksi.

---
{{< saweria >}}