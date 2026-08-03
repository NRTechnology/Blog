---
title: "Upgrade Open Journal Systems (OJS) 3.4"
date: 2026-08-02
draft: false
description: "Panduan lengkap melakukan upgrade Open Journal Systems (OJS) 3.4 pada lingkungan produksi menggunakan arsitektur Nginx, Docker PHP-FPM, dan MariaDB."
tags:
  - OJS
  - Open Journal Systems
  - Upgrade
  - Docker
  - PHP
  - MariaDB
  - Linux
categories:
  - OJS
  - Linux

series:
  - "Membangun Open Journal Systems (OJS) 3.4"
weight: 9

author: "NR Technology"
---

# Upgrade Open Journal Systems (OJS) 3.4

## Pendahuluan

Setelah Open Journal Systems berjalan pada lingkungan produksi, administrator perlu melakukan pemeliharaan secara berkala. Salah satu bentuk pemeliharaan tersebut adalah melakukan upgrade aplikasi ke versi yang lebih baru.

Upgrade dilakukan untuk memperoleh.

- Perbaikan bug.
- Peningkatan keamanan.
- Fitur baru.
- Peningkatan stabilitas.
- Dukungan terhadap komponen perangkat lunak terbaru.

Walaupun demikian, proses upgrade juga memiliki risiko apabila dilakukan tanpa persiapan yang memadai.

Artikel ini membahas tahapan upgrade Open Journal Systems secara sistematis agar proses pembaruan dapat dilakukan dengan aman dan memiliki risiko seminimal mungkin.

---

# Tujuan Upgrade

Upgrade bukan sekadar memperbarui source code.

Tujuan utama upgrade adalah.

- Meningkatkan keamanan aplikasi.
- Memperbaiki bug yang telah diketahui.
- Menambah fitur baru.
- Meningkatkan kompatibilitas.
- Menjaga dukungan dari pengembang.

Administrator sebaiknya tidak melakukan upgrade hanya karena tersedia versi terbaru.

Setiap proses upgrade harus direncanakan dan diuji terlebih dahulu.

---

# Kapan Upgrade Perlu Dilakukan?

Beberapa kondisi yang menjadi alasan dilakukannya upgrade.

- Tersedia pembaruan keamanan.
- Terdapat bug yang memengaruhi operasional.
- Dibutuhkan fitur baru.
- Dukungan terhadap versi lama akan berakhir.
- Infrastruktur server telah diperbarui.
- Organisasi memiliki kebijakan pembaruan aplikasi.

Tidak seluruh pembaruan harus segera diterapkan pada server produksi.

Administrator sebaiknya mempelajari terlebih dahulu perubahan yang diperkenalkan pada versi baru.

---

# Memahami Jenis Upgrade

Secara umum terdapat beberapa jenis upgrade.

## Patch Upgrade

Patch Upgrade merupakan pembaruan dengan perubahan yang relatif kecil.

Contohnya.

```text
3.4.0-4

↓

3.4.0-5
```

Patch biasanya berisi.

- Perbaikan bug.
- Perbaikan keamanan.
- Penyempurnaan kecil.

Patch Upgrade umumnya memiliki risiko yang lebih rendah.

---

## Minor Upgrade

Minor Upgrade memperkenalkan fitur baru namun masih berada pada cabang versi yang sama.

Contohnya.

```text
3.4.x

↓

3.5.x
```

Minor Upgrade dapat membawa perubahan pada struktur database maupun plugin.

Administrator perlu memastikan kompatibilitas seluruh komponen sebelum melakukan upgrade.

---

## Major Upgrade

Major Upgrade merupakan perubahan versi yang signifikan.

Contohnya.

```text
3.x

↓

4.x
```

Upgrade jenis ini biasanya membawa perubahan besar pada.

- Arsitektur aplikasi.
- Struktur database.
- Plugin.
- Theme.
- Persyaratan sistem.

Major Upgrade memerlukan proses pengujian yang lebih menyeluruh.

---

# Memahami Risiko Upgrade

Setiap proses upgrade memiliki risiko.

Beberapa risiko yang umum dijumpai.

- Plugin tidak kompatibel.
- Theme tidak kompatibel.
- Struktur database berubah.
- Konfigurasi berubah.
- Aplikasi gagal dijalankan.
- Downtime lebih lama dari yang direncanakan.

Risiko tersebut dapat dikurangi melalui perencanaan dan pengujian yang baik.

---

# Strategi Upgrade

Pada lingkungan produksi, strategi yang direkomendasikan adalah.

```text
Backup

↓

Bangun Lingkungan Pengujian

↓

Upgrade

↓

Pengujian

↓

Validasi

↓

Produksi
```

Dengan pendekatan tersebut, administrator dapat menemukan masalah sebelum perubahan diterapkan pada server produksi.

---

# Upgrade pada Lingkungan Pengujian

Sebelum melakukan upgrade pada server produksi, lakukan terlebih dahulu pada lingkungan pengujian.

Gunakan.

- Backup database produksi.
- Backup ojsdata.
- Backup source code.

Lingkungan pengujian sebaiknya memiliki konfigurasi yang mendekati server produksi.

Melalui pendekatan ini, kompatibilitas aplikasi dapat diuji tanpa memengaruhi layanan yang sedang digunakan pengguna.

---

# Memastikan Kompatibilitas

Sebelum melakukan upgrade, pastikan seluruh komponen tetap kompatibel.

Periksa.

- Open Journal Systems.
- PHP.
- MariaDB.
- Plugin.
- Theme.
- Sistem Operasi.

Kompatibilitas merupakan salah satu faktor yang paling menentukan keberhasilan proses upgrade.

---

# Mengidentifikasi Plugin

Periksa seluruh plugin yang digunakan.

Dokumentasikan.

- Plugin bawaan.
- Plugin tambahan.
- Plugin pihak ketiga.

Pastikan setiap plugin mendukung versi Open Journal Systems yang akan digunakan setelah upgrade.

Plugin yang belum kompatibel sebaiknya tidak langsung digunakan pada server produksi.

---

# Mengidentifikasi Theme

Lakukan pemeriksaan terhadap seluruh theme.

Pastikan.

- Theme masih dipelihara.
- Theme kompatibel.
- Tidak terdapat modifikasi yang belum didokumentasikan.

Apabila menggunakan theme khusus, lakukan pengujian secara menyeluruh setelah upgrade selesai.

---

# Menentukan Jadwal Upgrade

Upgrade sebaiknya dilakukan pada waktu dengan aktivitas pengguna yang rendah.

Beberapa hal yang perlu ditentukan.

- Waktu mulai.
- Estimasi selesai.
- Estimasi downtime.
- Waktu validasi.
- Waktu layanan dibuka kembali.

Informasikan jadwal tersebut kepada pengelola jurnal sebelum proses upgrade dimulai.

---

# Menyiapkan Rollback Plan

Setiap proses upgrade harus memiliki prosedur rollback.

Rollback dilakukan apabila.

- Upgrade gagal.
- Plugin tidak berjalan.
- Database mengalami masalah.
- Aplikasi tidak dapat diakses.
- Terjadi kehilangan data.

Dengan adanya rollback plan, administrator dapat mengembalikan layanan ke kondisi sebelumnya dengan lebih cepat.

---

# Checklist Sebelum Upgrade

Sebelum memulai upgrade, lakukan pemeriksaan berikut.

- Versi OJS saat ini telah didokumentasikan.
- Backup terbaru tersedia.
- Backup telah diverifikasi.
- Plugin telah diperiksa.
- Theme telah diperiksa.
- Kompatibilitas PHP telah diperiksa.
- Kompatibilitas MariaDB telah diperiksa.
- Lingkungan pengujian tersedia.
- Jadwal upgrade telah ditentukan.
- Rollback plan telah disiapkan.

Checklist tersebut membantu mengurangi risiko kegagalan selama proses upgrade.

---

# Ruang Lingkup Artikel

Artikel ini akan membahas.

- Persiapan upgrade.
- Backup sebelum upgrade.
- Upgrade source code.
- Upgrade database.
- Penyesuaian konfigurasi.
- Validasi pasca-upgrade.
- Troubleshooting.
- Rollback.
- Best Practices.

Seluruh contoh menggunakan placeholder sehingga dapat diterapkan pada berbagai lingkungan tanpa membawa informasi sensitif dari server produksi.

Pada bagian berikutnya kita akan membahas tahapan persiapan teknis sebelum upgrade, mulai dari pembuatan backup, mengaktifkan maintenance mode, mempersiapkan source code versi baru, hingga pemeriksaan plugin dan theme sebelum proses upgrade dijalankan.

---

# Backup Sebelum Upgrade

Sebelum melakukan perubahan apa pun pada Open Journal Systems, lakukan backup terhadap seluruh komponen penting.

Minimal lakukan backup terhadap.

- Database.
- `config.inc.php`.
- Direktori `ojsdata`.
- Source Code.
- Plugin tambahan.
- Theme tambahan.

Backup merupakan langkah yang paling penting karena akan digunakan apabila proses upgrade harus dibatalkan.

Apabila belum memiliki strategi backup yang terdokumentasi, pelajari terlebih dahulu artikel **Backup dan Restore Open Journal Systems (OJS) 3.4** sebelum melanjutkan proses upgrade.

---

# Memastikan Backup Valid

Backup yang berhasil dibuat belum tentu dapat digunakan.

Lakukan verifikasi.

- File SQL dapat dibuka.
- Arsip TAR dapat diekstrak.
- Ukuran file sesuai.
- Tidak terdapat pesan error selama proses backup.

Administrator juga disarankan melakukan pengujian restore pada lingkungan pengujian sebelum upgrade dilakukan pada server produksi.

---

# Mengaktifkan Maintenance Mode

Upgrade sebaiknya dilakukan ketika tidak terdapat aktivitas pengguna.

Administrator dapat mengumumkan jadwal pemeliharaan melalui halaman utama website maupun media komunikasi organisasi.

Selama proses upgrade.

- Jangan menerima Submission baru.
- Jangan melakukan Review.
- Jangan melakukan Publikasi Issue.
- Jangan mengubah konfigurasi.
- Jangan memasang Plugin baru.

Tujuannya adalah menjaga konsistensi data selama proses upgrade.

---

# Menghentikan Aktivitas Aplikasi

Setelah seluruh pengguna diinformasikan.

Hentikan layanan aplikasi.

Apabila menggunakan Docker PHP-FPM.

```bash
docker stop ojs-php
```

Selanjutnya hentikan Nginx apabila diperlukan.

```bash
systemctl stop nginx
```

Menghentikan layanan akan mencegah perubahan data selama proses upgrade berlangsung.

---

# Mengidentifikasi Versi Saat Ini

Sebelum melakukan upgrade, dokumentasikan versi yang sedang digunakan.

Periksa file versi.

```bash
cat \
/var/apps/ojs/htdocs/lib/pkp/includes/version.inc.php
```

Catat informasi berikut.

- Versi OJS.
- Build.
- Release.

Informasi tersebut diperlukan apabila proses rollback harus dilakukan.

---

# Menyiapkan Source Code Baru

Unduh paket Open Journal Systems yang akan digunakan.

Ekstrak pada direktori sementara.

Contoh.

```text
/tmp/ojs-upgrade
```

Jangan langsung menimpa source code produksi.

Gunakan direktori terpisah agar administrator dapat membandingkan struktur file sebelum proses upgrade dilakukan.

---

# Membandingkan Struktur Source Code

Periksa struktur direktori.

```bash
ls \
/tmp/ojs-upgrade
```

Bandingkan dengan.

```bash
ls \
/var/apps/ojs/htdocs
```

Pastikan paket yang digunakan lengkap.

Periksa keberadaan.

- classes
- lib
- plugins
- registry
- tools

Pemeriksaan sederhana ini membantu menghindari penggunaan paket yang tidak lengkap.

---

# Memeriksa Plugin

Sebelum upgrade.

Dokumentasikan plugin yang sedang digunakan.

```bash
ls \
/var/apps/ojs/htdocs/plugins
```

Kelompokkan plugin menjadi.

- Plugin bawaan.
- Plugin tambahan.
- Plugin pihak ketiga.

Selanjutnya periksa kompatibilitas masing-masing plugin terhadap versi OJS yang akan digunakan.

Plugin yang belum kompatibel sebaiknya dinonaktifkan sebelum upgrade.

---

# Memeriksa Theme

Periksa seluruh theme.

```bash
ls \
/var/apps/ojs/htdocs/plugins/themes
```

Pastikan.

- Theme berasal dari sumber terpercaya.
- Theme mendukung versi OJS tujuan.
- Modifikasi lokal telah didokumentasikan.

Apabila menggunakan theme hasil pengembangan sendiri, lakukan pengujian tambahan setelah upgrade selesai.

---

# Memeriksa File Konfigurasi

Buka file konfigurasi.

```bash
nano \
/var/apps/ojs/htdocs/config.inc.php
```

Periksa parameter berikut.

- installed
- base_url
- database
- username
- files_dir

Apabila terdapat parameter khusus yang ditambahkan selama implementasi, dokumentasikan sebelum upgrade dimulai.

---

# Memeriksa Direktori Upload

Pastikan lokasi upload masih sesuai.

```bash
grep "^files_dir" \
/var/apps/ojs/htdocs/config.inc.php
```

Selanjutnya periksa direktori tersebut.

```bash
ls -lah \
/var/apps/ojs/data/ojsdata
```

Direktori upload tidak boleh dihapus maupun ditimpa selama proses upgrade.

---

# Memeriksa Permission

Periksa ownership.

```bash
ls -lah \
/var/apps/ojs
```

Pastikan user yang menjalankan PHP-FPM masih memiliki akses terhadap.

- config.inc.php
- cache
- public
- ojsdata

Permission yang tidak sesuai dapat menyebabkan proses upgrade gagal.

---

# Menyiapkan Lingkungan Pengujian

Sebelum melakukan upgrade pada server produksi, lakukan simulasi pada lingkungan pengujian.

Gunakan.

- Backup Database.
- Backup ojsdata.
- Backup Source Code.

Lingkungan pengujian memungkinkan administrator mengidentifikasi masalah kompatibilitas tanpa memengaruhi layanan produksi.

---

# Menyusun Checklist Upgrade

Sebelum menjalankan proses upgrade, pastikan seluruh poin berikut telah terpenuhi.

- Backup Database selesai.
- Backup `config.inc.php` selesai.
- Backup `ojsdata` selesai.
- Backup Source Code selesai.
- Backup Plugin selesai.
- Backup Theme selesai.
- Backup telah diverifikasi.
- Plugin telah diperiksa.
- Theme telah diperiksa.
- Source Code versi baru telah disiapkan.
- Lingkungan pengujian telah tersedia.
- Maintenance telah dijadwalkan.
- Rollback Plan telah disiapkan.

Checklist tersebut membantu memastikan seluruh persiapan teknis telah selesai sebelum perubahan dilakukan pada server produksi.

---

# Ringkasan

Pada bagian ini telah dibahas seluruh persiapan teknis sebelum proses upgrade dijalankan.

- Backup seluruh komponen.
- Verifikasi backup.
- Maintenance mode.
- Menghentikan layanan.
- Dokumentasi versi OJS.
- Menyiapkan source code baru.
- Pemeriksaan plugin.
- Pemeriksaan theme.
- Pemeriksaan konfigurasi.
- Pemeriksaan direktori upload.
- Pemeriksaan permission.
- Menyiapkan lingkungan pengujian.
- Checklist sebelum upgrade.

Persiapan yang baik merupakan faktor utama keberhasilan proses upgrade. Sebagian besar kegagalan upgrade bukan disebabkan oleh proses upgrade itu sendiri, melainkan karena kurangnya persiapan sebelum perubahan dilakukan.

Pada bagian berikutnya kita akan membahas proses upgrade secara langsung, mulai dari mengganti source code, menjalankan proses upgrade database, membersihkan cache aplikasi, mengatur kembali permission, hingga melakukan verifikasi awal setelah upgrade selesai.

---

# Memulai Proses Upgrade

Setelah seluruh persiapan selesai, administrator dapat memulai proses upgrade.

Pastikan.

- Backup telah diverifikasi.
- Maintenance telah diumumkan.
- Pengguna tidak lagi melakukan aktivitas.
- Rollback Plan telah disiapkan.

Seluruh proses upgrade sebaiknya dilakukan melalui terminal agar setiap langkah dapat dipantau dengan mudah.

---

# Menghentikan Layanan

Sebelum mengganti source code.

Hentikan layanan aplikasi.

Apabila menggunakan Docker PHP-FPM.

```bash
docker stop ojs-php
```

Selanjutnya hentikan Nginx.

```bash
systemctl stop nginx
```

Dengan demikian tidak terdapat proses PHP yang masih menggunakan file lama.

---

# Mengganti Source Code

Ekstrak source code versi baru pada direktori sementara.

Contoh.

```text
/tmp/ojs-upgrade
```

Selanjutnya salin source code ke direktori aplikasi.

Contoh.

```bash
cp -a \
/tmp/ojs-upgrade/. \
/var/apps/ojs/htdocs/
```

Pastikan file konfigurasi.

```text
config.inc.php
```

tidak tertimpa apabila telah disesuaikan dengan lingkungan produksi.

---

# Memeriksa File Konfigurasi

Buka kembali.

```bash
nano \
/var/apps/ojs/htdocs/config.inc.php
```

Pastikan parameter berikut tetap sesuai.

- installed
- base_url
- database
- username
- files_dir

Apabila terdapat parameter baru pada versi OJS yang digunakan, sesuaikan berdasarkan dokumentasi resmi sebelum melanjutkan proses upgrade.

---

# Menjalankan Upgrade Database

Sebagian proses upgrade memerlukan pembaruan struktur database.

Proses tersebut dijalankan menggunakan utilitas yang disediakan oleh Open Journal Systems.

Masuk ke direktori aplikasi.

```bash
cd /var/apps/ojs/htdocs
```

Kemudian jalankan utilitas upgrade menggunakan PHP.

```bash
php tools/upgrade.php upgrade
```

Apabila menggunakan Docker PHP-FPM.

```bash
docker exec \
ojs-php \
php \
/var/www/html/tools/upgrade.php \
upgrade
```

Tunggu hingga proses selesai.

Jangan menghentikan proses sebelum selesai karena dapat menyebabkan struktur database menjadi tidak konsisten.

---

# Memantau Proses Upgrade

Selama proses berlangsung, perhatikan pesan yang ditampilkan.

Administrator perlu memastikan.

- Tidak terdapat Fatal Error.
- Tidak terdapat Database Error.
- Tidak terdapat Permission Denied.

Apabila muncul kesalahan, hentikan proses dan lakukan analisis sebelum melanjutkan.

---

# Memverifikasi Database

Setelah utilitas upgrade selesai dijalankan.

Masuk ke MariaDB.

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

Pastikan seluruh tabel masih tersedia.

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

Jumlah data seharusnya tetap sesuai dengan kondisi sebelum upgrade.

---

# Membersihkan Cache

Setelah source code diperbarui, cache aplikasi sebaiknya dibersihkan.

Masuk ke direktori aplikasi.

```bash
cd /var/apps/ojs/htdocs
```

Hapus isi direktori cache.

```bash
rm -rf cache/t_cache/*
```

```bash
rm -rf cache/t_compile/*
```

```bash
rm -rf cache/_db/*
```

Jangan menghapus direktori cache, cukup hapus isinya.

Pada saat aplikasi dijalankan kembali, cache akan dibuat ulang secara otomatis.

---

# Memeriksa Permission

Setelah proses upgrade selesai.

Periksa ownership.

```bash
chown -R \
www-data:www-data \
/var/apps/ojs
```

Selanjutnya atur permission direktori.

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

Terakhir.

```bash
chmod 640 \
/var/apps/ojs/htdocs/config.inc.php
```

Permission yang benar akan mengurangi kemungkinan munculnya error setelah upgrade.

---

# Menjalankan Kembali Layanan

Aktifkan kembali PHP-FPM.

```bash
docker start ojs-php
```

Kemudian jalankan Nginx.

```bash
systemctl start nginx
```

Pastikan kedua layanan berjalan normal.

---

# Memverifikasi Versi OJS

Periksa kembali file versi.

```bash
cat \
/var/apps/ojs/htdocs/lib/pkp/includes/version.inc.php
```

Pastikan versi yang ditampilkan sesuai dengan versi yang baru dipasang.

Administrator juga dapat memverifikasi versi melalui Dashboard Administrator setelah berhasil login.

---

# Melakukan Pengujian Awal

Sebelum layanan dibuka kembali kepada pengguna.

Lakukan pengujian.

- Login Administrator.
- Dashboard.
- Submission.
- Reviewer.
- Editor.
- Issue.
- Download PDF.
- Upload Dokumen.

Seluruh fungsi utama harus berjalan tanpa menghasilkan error.

---

# Memantau Log

Periksa log selama proses pengujian.

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
- SQL Error.
- Permission Denied.
- File Not Found.

Apabila ditemukan masalah, lakukan perbaikan sebelum layanan dibuka kembali.

---

# Validasi Awal

Sebelum upgrade dinyatakan berhasil.

Pastikan.

- Source code berhasil diperbarui.
- Upgrade database selesai.
- Cache telah dibersihkan.
- Permission sesuai.
- Plugin aktif.
- Theme aktif.
- Login berhasil.
- Artikel dapat dibuka.
- PDF dapat diunduh.
- Upload berhasil.
- Tidak terdapat error pada log.

Validasi awal merupakan tahapan penting sebelum sistem kembali digunakan oleh pengguna.

---

# Ringkasan

Pada bagian ini telah dibahas proses utama upgrade Open Journal Systems.

- Menghentikan layanan.
- Mengganti source code.
- Memeriksa konfigurasi.
- Menjalankan utilitas upgrade.
- Memperbarui struktur database.
- Membersihkan cache.
- Mengatur permission.
- Menjalankan kembali layanan.
- Memverifikasi versi.
- Melakukan pengujian awal.
- Memantau log.
- Validasi awal.

Setelah seluruh tahapan tersebut selesai, Open Journal Systems telah berhasil diperbarui ke versi yang baru. Pada bagian terakhir kita akan membahas validasi pasca-upgrade, penanganan masalah yang umum terjadi, prosedur rollback, checklist upgrade, best practices, serta evaluasi akhir sebelum upgrade dinyatakan selesai.

---

# Validasi Pasca Upgrade

Setelah proses upgrade selesai, administrator tidak boleh langsung membuka kembali layanan kepada pengguna.

Seluruh fungsi utama Open Journal Systems harus diverifikasi terlebih dahulu untuk memastikan aplikasi berjalan dengan normal.

Validasi yang dilakukan setelah upgrade bertujuan untuk.

- Memastikan aplikasi dapat digunakan.
- Memastikan data tetap utuh.
- Memastikan konfigurasi tidak berubah.
- Memastikan plugin tetap berjalan.
- Memastikan theme tetap berfungsi.

Tahapan ini merupakan bagian penting dari proses upgrade pada lingkungan produksi.

---

# Memverifikasi Dashboard Administrator

Login menggunakan akun administrator.

Pastikan Dashboard dapat dibuka tanpa menghasilkan error.

Periksa beberapa menu utama.

- Dashboard.
- Website.
- Users & Roles.
- Settings.
- Statistics.
- Tools.

Seluruh menu harus dapat diakses sebagaimana sebelum proses upgrade.

---

# Memverifikasi Data Jurnal

Periksa beberapa data penting.

- Journal.
- Issue.
- Submission.
- Published Article.
- User.

Pastikan seluruh data masih tersedia.

Upgrade aplikasi tidak boleh menyebabkan kehilangan data.

---

# Menguji Proses Login

Lakukan pengujian login menggunakan beberapa jenis akun.

- Administrator.
- Journal Manager.
- Editor.
- Reviewer.
- Author.

Pastikan masing-masing Role dapat masuk ke sistem sesuai hak aksesnya.

---

# Menguji Workflow Editorial

Lakukan simulasi sederhana terhadap alur kerja jurnal.

Contohnya.

- Membuat Submission baru.
- Menugaskan Reviewer.
- Mengunggah Revisi.
- Mengirim Keputusan.
- Mempublikasikan Artikel.

Pengujian ini membantu memastikan bahwa seluruh proses editorial tetap berjalan setelah upgrade.

---

# Menguji Upload Dokumen

Lakukan pengujian upload.

Contohnya.

- Artikel PDF.
- Supplementary File.
- Cover Journal.

Pastikan seluruh file berhasil diunggah.

Apabila upload gagal, periksa kembali.

- Permission.
- Ownership.
- Konfigurasi `files_dir`.

---

# Menguji Download Artikel

Buka beberapa artikel yang telah dipublikasikan.

Pastikan.

- PDF dapat diunduh.
- Cover tampil dengan benar.
- Lampiran dapat diakses.

Apabila dokumen tidak dapat diakses, lakukan pemeriksaan terhadap direktori `ojsdata`.

---

# Menguji Plugin

Periksa seluruh plugin yang aktif.

Pastikan.

- Plugin dapat dimuat.
- Tidak menghasilkan error.
- Tetap berfungsi.

Plugin pihak ketiga memerlukan perhatian khusus karena mungkin belum mendukung versi terbaru.

Apabila terdapat plugin yang tidak kompatibel, administrator sebaiknya menonaktifkannya sampai tersedia versi yang sesuai.

---

# Menguji Theme

Periksa tampilan website.

Pastikan.

- Logo tampil.
- CSS termuat.
- JavaScript berjalan.
- Navigasi normal.
- Halaman artikel dapat dibuka.

Apabila tampilan berubah secara signifikan, periksa kembali kompatibilitas theme yang digunakan.

---

# Memeriksa Scheduled Task

Pastikan Scheduled Task tetap berjalan.

Periksa Cron.

```bash
crontab -l
```

Selanjutnya lakukan pengujian terhadap Scheduled Task apabila diperlukan.

Scheduled Task yang tidak berjalan dapat menyebabkan beberapa fungsi OJS tidak bekerja sebagaimana mestinya.

---

# Memantau Log

Monitoring log harus dilakukan setelah upgrade.

Periksa.

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

- PHP Fatal Error.
- SQL Error.
- Permission Denied.
- File Not Found.
- Plugin Error.

Monitoring log membantu menemukan masalah yang mungkin tidak terlihat dari antarmuka aplikasi.

---

# Troubleshooting Umum

Beberapa masalah yang sering ditemui setelah upgrade antara lain.

## Halaman Putih (White Screen)

Kemungkinan penyebab.

- PHP Fatal Error.
- Plugin tidak kompatibel.
- Theme tidak kompatibel.

Lakukan pemeriksaan terhadap log PHP.

---

## HTTP 500 Internal Server Error

Periksa.

- Permission.
- Ownership.
- Konfigurasi Nginx.
- Konfigurasi PHP-FPM.

Kemudian lihat log Nginx maupun PHP.

---

## Artikel Tidak Dapat Diunduh

Periksa.

- Direktori `ojsdata`.
- Parameter `files_dir`.
- Permission direktori upload.

Pastikan lokasi upload masih sesuai dengan konfigurasi.

---

## Plugin Tidak Berjalan

Periksa.

- Kompatibilitas plugin.
- Konfigurasi plugin.
- Dokumentasi plugin.

Apabila belum kompatibel, nonaktifkan plugin tersebut sementara waktu.

---

## Theme Tidak Tampil

Pastikan.

- Theme masih tersedia.
- CSS termuat.
- JavaScript termuat.

Apabila diperlukan, gunakan sementara theme bawaan OJS untuk memastikan masalah berasal dari theme, bukan dari proses upgrade.

---

# Melakukan Rollback

Apabila terjadi masalah yang tidak dapat diselesaikan dalam waktu yang dapat diterima, lakukan rollback.

Urutan rollback.

```text
Hentikan Layanan

↓

Restore Database

↓

Restore Source Code

↓

Restore config.inc.php

↓

Restore ojsdata

↓

Menjalankan Layanan

↓

Validasi
```

Rollback harus menggunakan backup terakhir yang telah diverifikasi sebelum proses upgrade dimulai.

---

# Kapan Rollback Dilakukan?

Rollback dapat dipertimbangkan apabila.

- Upgrade gagal.
- Database mengalami inkonsistensi.
- Login tidak berfungsi.
- Submission gagal.
- Plugin penting tidak dapat digunakan.
- Theme tidak kompatibel.
- Terjadi kehilangan data.

Rollback merupakan bagian dari strategi mitigasi risiko dan bukan merupakan kegagalan proses administrasi.

---

# Evaluasi Upgrade

Setelah upgrade selesai.

Lakukan evaluasi terhadap.

- Durasi upgrade.
- Durasi downtime.
- Kendala yang ditemukan.
- Solusi yang diterapkan.
- Plugin yang diperbarui.
- Theme yang diperbarui.

Dokumentasikan seluruh hasil evaluasi sebagai referensi untuk proses upgrade berikutnya.

---

# Checklist Upgrade

Sebelum upgrade dinyatakan selesai.

Pastikan.

- Versi OJS telah berubah sesuai target.
- Dashboard dapat diakses.
- Login berhasil.
- Submission berhasil.
- Upload berhasil.
- Download berhasil.
- Plugin berfungsi.
- Theme berfungsi.
- Scheduled Task berjalan.
- Email berfungsi.
- Tidak terdapat error pada log.
- Backup sebelum upgrade masih tersedia.
- Dokumentasi upgrade telah diperbarui.

Checklist tersebut dapat digunakan sebagai bagian dari proses serah terima setelah upgrade selesai.

---

# Best Practices

Beberapa praktik terbaik yang direkomendasikan.

- Lakukan backup penuh sebelum upgrade.
- Uji upgrade pada lingkungan pengujian.
- Dokumentasikan seluruh perubahan.
- Gunakan plugin yang kompatibel.
- Gunakan theme yang kompatibel.
- Pantau log setelah upgrade.
- Simpan backup sebelum upgrade.
- Siapkan prosedur rollback.
- Lakukan upgrade di luar jam operasional.
- Evaluasi hasil upgrade setelah selesai.

Penerapan praktik tersebut akan mengurangi risiko kegagalan serta meningkatkan stabilitas Open Journal Systems setelah diperbarui.

---

# Kesimpulan

Upgrade Open Journal Systems merupakan bagian dari siklus pemeliharaan aplikasi yang bertujuan menjaga keamanan, stabilitas, dan kompatibilitas sistem.

Keberhasilan upgrade tidak hanya ditentukan oleh proses pembaruan source code, tetapi juga oleh persiapan yang matang, backup yang tervalidasi, pengujian pada lingkungan yang sesuai, serta validasi menyeluruh setelah proses upgrade selesai.

Dengan menerapkan prosedur yang sistematis, administrator dapat memperbarui Open Journal Systems dengan risiko yang lebih rendah serta tetap menjaga ketersediaan layanan bagi seluruh pengguna.

---

# Ringkasan

Pada artikel ini telah dibahas.

- Konsep Upgrade Open Journal Systems.
- Persiapan Upgrade.
- Kompatibilitas Sistem.
- Backup Sebelum Upgrade.
- Pemeriksaan Plugin.
- Pemeriksaan Theme.
- Upgrade Source Code.
- Upgrade Database.
- Pembersihan Cache.
- Validasi Pasca Upgrade.
- Troubleshooting.
- Rollback.
- Checklist Upgrade.
- Best Practices.

Dengan selesainya artikel ini, rangkaian **Membangun Open Journal Systems (OJS) 3.4 dengan Docker** telah mencakup seluruh siklus implementasi, mulai dari perencanaan infrastruktur, instalasi, hardening, backup, restore, migrasi, hingga pemeliharaan melalui proses upgrade pada lingkungan produksi.