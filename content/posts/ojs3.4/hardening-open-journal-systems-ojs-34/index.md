---
title: "Hardening Open Journal Systems (OJS) 3.4"
date: 2026-08-02
draft: false
description: "Panduan hardening Open Journal Systems (OJS) 3.4 untuk meningkatkan keamanan aplikasi pada lingkungan produksi."
tags:
  - OJS
  - Open Journal Systems
  - Security
  - Hardening
  - Nginx
  - Docker
  - PHP
categories:
  - OJS
  - Security

series:
  - "Membangun Open Journal Systems (OJS) 3.4"
weight: 6

author: "NR Technology"
cover:
  image: "ojs-cover.png"
  alt: "Open Journal Systems (OJS) 3.4"
  caption: "Seri Membangun Open Journal Systems (OJS) 3.4"
---

# Hardening Open Journal Systems (OJS) 3.4

## Pendahuluan

Menginstal Open Journal Systems hingga dapat diakses melalui browser bukanlah akhir dari proses implementasi. Sebelum digunakan pada lingkungan produksi, aplikasi harus diamankan agar mampu menghadapi berbagai risiko keamanan yang dapat mengganggu kerahasiaan, integritas, maupun ketersediaan layanan.

Hardening merupakan proses mengurangi permukaan serangan (attack surface) dengan menghapus, membatasi, atau mengamankan komponen yang tidak diperlukan oleh aplikasi.

Pada artikel ini pembahasan difokuskan pada **hardening aplikasi Open Journal Systems**, bukan hardening sistem operasi, Nginx, PHP-FPM, Docker, maupun MariaDB karena seluruh komponen tersebut telah dibahas pada artikel sebelumnya.

---

# Tujuan Hardening

Hardening dilakukan untuk mencapai beberapa tujuan berikut.

- Melindungi data jurnal dan metadata.
- Melindungi akun administrator.
- Mengurangi risiko perubahan konfigurasi.
- Mengurangi risiko eksekusi file yang tidak sah.
- Melindungi dokumen yang diunggah pengguna.
- Mempermudah proses audit keamanan.
- Meningkatkan keandalan sistem pada lingkungan produksi.

Hardening bukan berarti membuat sistem menjadi kebal terhadap serangan, tetapi mengurangi peluang keberhasilan serangan dan memperkecil dampak apabila terjadi insiden.

---

# Arsitektur Keamanan

Pada seri artikel ini digunakan arsitektur berikut.

```text
                 Internet
                      │
                 HTTPS 443
                      │
              Reverse Proxy
                      │
             Nginx Application
                      │
              PHP-FPM Docker
                      │
                  MariaDB
```

Sedangkan Open Journal Systems terdiri atas beberapa komponen.

```text
Open Journal Systems

├── Source Code
├── config.inc.php
├── public
├── cache
├── plugins
├── themes
└── ojsdata
```

Setiap komponen memiliki tingkat sensitivitas yang berbeda sehingga membutuhkan perlakuan keamanan yang berbeda pula.

---

# Memahami Attack Surface

Attack Surface merupakan seluruh titik yang berpotensi menjadi sasaran serangan.

Pada Open Journal Systems beberapa attack surface utama meliputi.

- Halaman Login
- Dashboard Administrator
- Upload File
- Plugin
- Theme
- File Konfigurasi
- Direktori Upload
- Session
- Cookie
- Database

Semakin sedikit attack surface yang tersedia, semakin kecil peluang sistem berhasil disusupi.

---

# Ancaman yang Umum Terjadi

Open Journal Systems merupakan aplikasi web yang dapat diakses melalui Internet sehingga berpotensi menghadapi berbagai ancaman keamanan.

Beberapa ancaman yang sering dijumpai antara lain.

- Percobaan login berulang (Brute Force).
- Pencurian akun administrator.
- Upload file berbahaya.
- Penyalahgunaan plugin.
- Penyalahgunaan permission direktori.
- Kebocoran file konfigurasi.
- Defacement website.
- Eksploitasi kerentanan aplikasi.
- Pengungkapan informasi sensitif.

Pemahaman terhadap ancaman tersebut menjadi dasar dalam menentukan langkah hardening yang tepat.

---

# Prinsip Hardening

Seluruh langkah hardening pada artikel ini mengikuti beberapa prinsip dasar.

## Least Privilege

Setiap proses hanya diberikan hak akses yang benar-benar diperlukan.

Sebagai contoh.

- Source code tidak perlu dapat ditulis selama aplikasi berjalan.
- Direktori upload hanya dapat ditulis oleh PHP.
- File konfigurasi tidak boleh dapat diubah oleh pengguna biasa.

---

## Defense in Depth

Keamanan tidak bergantung pada satu lapisan perlindungan.

Sebagai contoh.

```text
Firewall

↓

Reverse Proxy

↓

Nginx

↓

PHP

↓

Open Journal Systems

↓

MariaDB
```

Apabila satu lapisan gagal, masih terdapat lapisan lain yang memberikan perlindungan.

---

## Secure by Default

Konfigurasi bawaan harus dibuat seaman mungkin.

Contohnya.

- Installer tidak dapat dijalankan kembali.
- File konfigurasi tidak dapat diubah.
- Direktori upload tidak dapat diakses secara langsung.
- Hanya file PHP yang diperlukan yang dapat dieksekusi.

Pendekatan ini membantu mengurangi kesalahan konfigurasi setelah implementasi.

---

## Separation of Components

Setiap komponen memiliki fungsi yang berbeda.

Sebagai contoh.

```text
Nginx
↓
Melayani HTTP
```

```text
PHP-FPM
↓
Menjalankan aplikasi
```

```text
MariaDB
↓
Menyimpan data
```

```text
ojsdata
↓
Menyimpan dokumen
```

Pemisahan tersebut membuat pengelolaan keamanan menjadi lebih sederhana.

---

# Komponen yang Akan Diamankan

Artikel ini akan membahas hardening terhadap beberapa komponen berikut.

```text
config.inc.php
```

```text
Source Code
```

```text
public
```

```text
ojsdata
```

```text
cache
```

```text
Plugin
```

```text
Theme
```

```text
Administrator
```

```text
Session
```

```text
Cookie
```

```text
Backup
```

```text
Log
```

Masing-masing komponen akan dibahas secara terpisah agar administrator memahami tujuan dari setiap langkah hardening.

---

# Hardening Setelah Instalasi

Proses hardening sebaiknya dilakukan segera setelah instalasi Open Journal Systems selesai dan sebelum sistem digunakan oleh pengguna.

Urutan yang digunakan pada artikel ini adalah sebagai berikut.

1. Mengamankan file konfigurasi.
2. Mengamankan source code.
3. Mengamankan direktori upload.
4. Mengamankan direktori public.
5. Mengamankan cache.
6. Mengamankan akun administrator.
7. Mengamankan plugin dan theme.
8. Melakukan audit konfigurasi.
9. Menyiapkan backup.
10. Menyiapkan monitoring.

Urutan tersebut membantu administrator melakukan hardening secara sistematis tanpa mengganggu operasional aplikasi.

---

# Ruang Lingkup Artikel

Artikel ini hanya membahas hardening pada tingkat aplikasi Open Journal Systems.

Topik berikut **tidak dibahas kembali** karena telah dijelaskan pada artikel sebelumnya.

- Hardening Ubuntu Server.
- Hardening Nginx.
- Hardening PHP-FPM.
- Hardening Docker.
- Hardening MariaDB.
- Konfigurasi Reverse Proxy.
- Konfigurasi SSL/TLS.

Dengan demikian pembahasan dapat difokuskan pada pengamanan Open Journal Systems itu sendiri.

---

# Prasyarat

Sebelum mengikuti artikel ini, pastikan.

- Open Journal Systems telah berhasil diinstal.
- Administrator dapat login ke Dashboard.
- Nginx telah dikonfigurasi.
- PHP-FPM telah berjalan dengan baik.
- MariaDB dapat diakses.
- Seluruh fungsi dasar OJS telah diuji.

Hardening tidak boleh dilakukan pada sistem yang masih mengalami masalah instalasi maupun konfigurasi.

Pada bagian berikutnya kita akan mulai mengamankan komponen yang paling penting dalam Open Journal Systems, yaitu file konfigurasi `config.inc.php`, struktur source code, direktori `public`, direktori `ojsdata`, serta direktori `cache`.

---

# Mengamankan File Konfigurasi

File yang paling sensitif pada Open Journal Systems adalah.

```text
config.inc.php
```

File tersebut berisi hampir seluruh konfigurasi utama aplikasi.

Di dalamnya terdapat informasi seperti.

- Konfigurasi database
- Lokasi direktori upload
- Base URL
- Pengaturan session
- Pengaturan keamanan
- Pengaturan email
- Konfigurasi aplikasi

Apabila file ini dapat dibaca atau dimodifikasi oleh pihak yang tidak berwenang, seluruh instalasi Open Journal Systems dapat dikompromikan.

Karena itu file ini harus menjadi prioritas utama dalam proses hardening.

---

# Memeriksa Ownership

Periksa ownership file.

```bash
ls -lah /var/apps/ojs/htdocs/config.inc.php
```

Contoh.

```text
-rw-r----- 1 www-data www-data config.inc.php
```

Pastikan owner menggunakan user yang menjalankan PHP-FPM.

Pada implementasi ini.

```text
www-data
```

---

# Mengatur Permission

Setelah proses instalasi selesai, ubah permission menjadi.

```bash
chmod 640 \
/var/apps/ojs/htdocs/config.inc.php
```

Permission tersebut memberikan hak.

```text
Owner

Read
Write
```

```text
Group

Read
```

```text
Others

No Access
```

Konfigurasi tersebut cukup untuk kebutuhan operasional aplikasi.

---

# Mencegah Perubahan Tidak Sengaja

Administrator sering melakukan perubahan konfigurasi menggunakan editor teks.

Sebelum melakukan perubahan, selalu buat salinan file.

```bash
cp \
config.inc.php \
config.inc.php.bak
```

Dengan demikian konfigurasi sebelumnya dapat dipulihkan apabila terjadi kesalahan.

---

# Mengamankan Source Code

Source code Open Journal Systems tidak memerlukan hak tulis selama aplikasi berjalan.

Periksa owner.

```bash
ls -lah /var/apps/ojs
```

Selanjutnya atur permission direktori.

```bash
find /var/apps/ojs/htdocs \
-type d \
-exec chmod 755 {} \;
```

Kemudian seluruh file.

```bash
find /var/apps/ojs/htdocs \
-type f \
-exec chmod 644 {} \;
```

Konfigurasi tersebut mengikuti praktik umum pada sistem Linux.

---

# Mengapa Source Code Tidak Perlu Writable?

Selama aplikasi berjalan.

Open Journal Systems.

- membaca source code
- menjalankan PHP
- menghasilkan HTML

Aplikasi tidak pernah mengubah source code miliknya sendiri.

Apabila source code tetap writable, maka malware maupun pihak yang memperoleh akses tidak sah memiliki peluang lebih besar untuk memodifikasi file aplikasi.

Menghilangkan hak tulis pada source code merupakan salah satu langkah hardening yang paling efektif.

---

# Mengamankan Direktori public

Direktori.

```text
public
```

berada di dalam Document Root.

Artinya seluruh file di dalamnya dapat diakses secara langsung melalui browser.

Direktori ini hanya digunakan untuk menyimpan aset yang memang ditujukan untuk publik.

Contohnya.

- Logo
- Cover Journal
- Banner
- Gambar

Jangan pernah menyimpan.

- Artikel PDF
- Dokumen Review
- Backup Database
- File Konfigurasi

pada direktori tersebut.

---

# Memeriksa Permission public

Periksa permission.

```bash
ls -lah \
/var/apps/ojs/htdocs/public
```

Pastikan owner.

```text
www-data
```

Kemudian gunakan permission.

```bash
chmod 755 \
/var/apps/ojs/htdocs/public
```

---

# Mengamankan Direktori Upload

Direktori upload merupakan salah satu komponen yang paling sering menjadi sasaran serangan.

Pada implementasi ini digunakan.

```text
/var/apps/ojs/data/ojsdata
```

Direktori tersebut berada di luar Document Root.

Dengan demikian browser tidak dapat mengakses file secara langsung.

Seluruh akses terhadap file dilakukan melalui mekanisme yang disediakan oleh Open Journal Systems.

---

# Mengapa ojsdata Berada di Luar Document Root?

Misalkan artikel PDF disimpan pada.

```text
/var/www/html/files
```

Pengguna cukup mengetahui URL file tersebut untuk mengaksesnya secara langsung.

Sebaliknya apabila file berada pada.

```text
/var/apps/ojs/data/ojsdata
```

Browser tidak memiliki jalur langsung menuju file tersebut.

Open Journal Systems akan memeriksa hak akses pengguna terlebih dahulu sebelum mengirimkan file.

Pendekatan ini memberikan lapisan perlindungan tambahan terhadap dokumen jurnal.

---

# Permission Direktori Upload

Periksa ownership.

```bash
ls -lah \
/var/apps/ojs/data
```

Kemudian.

```bash
chown -R www-data:www-data \
/var/apps/ojs/data/ojsdata
```

Gunakan permission.

```bash
chmod 755 \
/var/apps/ojs/data/ojsdata
```

Direktori upload harus dapat ditulis oleh PHP namun tidak memerlukan hak akses yang lebih longgar.

---

# Jangan Menyimpan Backup di ojsdata

Direktori upload hanya digunakan untuk file yang dikelola oleh aplikasi.

Jangan menyimpan.

- Backup Database
- Backup Source Code
- Arsip Server

di dalam direktori tersebut.

Gunakan direktori khusus.

```text
/var/apps/ojs/backup
```

untuk menyimpan file backup.

Pemisahan ini mempermudah pengelolaan data sekaligus mengurangi risiko kehilangan file penting.

---

# Mengamankan Direktori Cache

Open Journal Systems menggunakan beberapa direktori cache.

```text
cache/_db
```

```text
cache/t_cache
```

```text
cache/t_compile
```

Direktori tersebut memang harus dapat ditulis oleh PHP karena digunakan selama aplikasi berjalan.

Namun administrator tetap perlu memastikan bahwa hanya proses PHP yang memiliki hak untuk menulis ke dalamnya.

---

# Memeriksa Permission Cache

Periksa seluruh direktori cache.

```bash
ls -lah \
/var/apps/ojs/htdocs/cache
```

Pastikan owner.

```text
www-data
```

Kemudian gunakan permission.

```bash
find \
/var/apps/ojs/htdocs/cache \
-type d \
-exec chmod 755 {} \;
```

dan.

```bash
find \
/var/apps/ojs/htdocs/cache \
-type f \
-exec chmod 644 {} \;
```

Dengan konfigurasi tersebut cache tetap dapat digunakan tanpa memberikan hak akses yang berlebihan.

---

# Memastikan Tidak Ada File Sensitif di Document Root

Lakukan pemeriksaan sederhana.

```bash
find \
/var/apps/ojs/htdocs \
-type f
```

Periksa apakah terdapat file yang seharusnya tidak berada di dalam Document Root.

Contohnya.

- Backup Database
- Backup ZIP
- File SQL
- File TAR
- Arsip Konfigurasi

Apabila ditemukan, segera pindahkan ke direktori backup.

---

# Audit Struktur Direktori

Struktur direktori yang direkomendasikan adalah sebagai berikut.

```text
/var/apps/ojs
├── backup
├── data
│   └── ojsdata
├── htdocs
│   ├── cache
│   ├── classes
│   ├── plugins
│   ├── public
│   ├── registry
│   └── config.inc.php
└── logs
```

Dengan struktur tersebut.

- Source code terpisah dari data.
- Backup terpisah dari aplikasi.
- File upload berada di luar Document Root.
- Konfigurasi berada pada lokasi yang mudah dikelola.

Struktur yang konsisten mempermudah proses audit keamanan, backup, upgrade, maupun migrasi server.

---

# Ringkasan

Pada bagian ini telah dilakukan hardening terhadap komponen yang paling penting pada Open Journal Systems.

- File konfigurasi.
- Source code.
- Direktori `public`.
- Direktori `ojsdata`.
- Direktori `cache`.
- Struktur direktori aplikasi.

Langkah-langkah tersebut menjadi fondasi utama sebelum melanjutkan hardening terhadap aspek lain seperti session, cookie, upload file, akun administrator, plugin, dan konfigurasi aplikasi yang akan dibahas pada bagian berikutnya.

---

# Mengamankan Akses Installer

Installer Open Journal Systems hanya digunakan satu kali, yaitu pada saat proses instalasi awal.

Setelah instalasi selesai, halaman installer tidak boleh lagi dapat digunakan untuk melakukan instalasi ulang.

Hal ini dikendalikan oleh parameter berikut pada file `config.inc.php`.

```ini
installed = On
```

Periksa nilainya.

```bash
grep "^installed" \
/var/apps/ojs/htdocs/config.inc.php
```

Contoh.

```text
installed = On
```

Apabila parameter tersebut berubah menjadi `Off`, installer dapat dijalankan kembali sehingga berpotensi menimbulkan risiko keamanan.

---

# Memastikan Installer Tidak Dapat Diakses

Lakukan pengujian menggunakan browser.

```text
https://jurnal.example.go.id/index.php/index/install
```

Installer seharusnya tidak lagi menampilkan halaman instalasi.

Sebaliknya, aplikasi akan mengarahkan pengguna menuju halaman utama atau menampilkan pesan bahwa Open Journal Systems telah terinstal.

Langkah sederhana ini memastikan proses instalasi tidak dapat diulang secara tidak sengaja.

---

# Menonaktifkan Directory Listing

Directory Listing memungkinkan pengguna melihat isi suatu direktori melalui browser apabila tidak terdapat file index.

Sebagai contoh.

```text
https://jurnal.example.go.id/plugins/
```

atau.

```text
https://jurnal.example.go.id/public/
```

Apabila Directory Listing aktif, pengguna dapat melihat struktur direktori aplikasi.

Walaupun pengaturannya berada pada Nginx, administrator tetap perlu memastikan bahwa fitur tersebut benar-benar telah dinonaktifkan karena struktur direktori dapat memberikan informasi yang berguna bagi penyerang.

---

# Memastikan File Upload Tidak Dapat Dieksekusi

Open Journal Systems mengizinkan pengguna mengunggah berbagai jenis dokumen.

Namun file yang diunggah tidak boleh dapat dieksekusi sebagai program.

Sebagai contoh.

```text
shell.php
```

atau.

```text
backdoor.php
```

Apabila file tersebut dapat dijalankan oleh web server, maka seluruh server dapat dikompromikan.

Karena itu direktori upload harus diperlakukan sebagai lokasi penyimpanan data, bukan lokasi eksekusi aplikasi.

---

# Membatasi Jenis File Upload

Administrator sebaiknya hanya mengizinkan jenis file yang benar-benar diperlukan.

Contohnya.

- PDF
- DOCX
- ODT
- XLSX
- ZIP
- CSV
- Gambar

Hindari mengizinkan file yang dapat dieksekusi.

Contohnya.

- PHP
- PHTML
- PHAR
- CGI
- SH
- EXE

Semakin sedikit jenis file yang diperbolehkan, semakin kecil risiko penyalahgunaan fitur upload.

---

# Melakukan Validasi Upload

Selain membatasi ekstensi file, administrator juga perlu memastikan bahwa proses upload berjalan sesuai mekanisme Open Journal Systems.

Lakukan pengujian.

- Upload artikel PDF.
- Upload gambar.
- Upload lampiran.
- Upload file dengan ekstensi yang tidak diizinkan.

Pastikan aplikasi menolak file yang tidak sesuai dengan kebijakan organisasi.

---

# Mengamankan Session

Session digunakan untuk mempertahankan status login pengguna.

Setiap pengguna yang berhasil login akan memperoleh Session ID yang digunakan selama berinteraksi dengan aplikasi.

Administrator perlu memastikan bahwa session.

- Tidak mudah ditebak.
- Memiliki masa berlaku yang sesuai.
- Dihapus ketika pengguna logout.
- Tidak digunakan kembali setelah login ulang.

Pengelolaan session yang baik membantu mengurangi risiko pembajakan sesi (Session Hijacking).

---

# Mengamankan Cookie

Cookie menyimpan informasi yang digunakan selama proses autentikasi.

Apabila website diakses menggunakan HTTPS, cookie sebaiknya hanya dikirim melalui koneksi yang terenkripsi.

Selain itu, cookie sebaiknya tidak dapat diakses oleh JavaScript apabila tidak diperlukan.

Konfigurasi tersebut umumnya ditangani oleh PHP maupun Reverse Proxy, namun administrator tetap perlu memverifikasi bahwa browser menerima cookie dengan atribut keamanan yang sesuai.

---

# Memastikan Base URL Benar

Periksa parameter berikut pada `config.inc.php`.

```ini
base_url = "https://jurnal.example.go.id"
```

Pastikan menggunakan URL produksi yang benar.

Kesalahan konfigurasi Base URL dapat menyebabkan.

- Redirect berulang.
- Mixed Content.
- CSS tidak termuat.
- JavaScript tidak termuat.
- URL HTTP masih muncul.

---

# Memastikan HTTPS Digunakan

Seluruh akses menuju Open Journal Systems sebaiknya menggunakan HTTPS.

Lakukan pengujian.

```text
http://jurnal.example.go.id
```

Pastikan browser diarahkan menuju.

```text
https://jurnal.example.go.id
```

Selanjutnya buka beberapa halaman.

- Halaman Utama.
- Login.
- Dashboard.
- Submission.
- Download Artikel.

Seluruh halaman harus tetap menggunakan HTTPS.

---

# Memverifikasi Reverse Proxy

Apabila Open Journal Systems berada di belakang Reverse Proxy, pastikan informasi berikut diteruskan dengan benar.

- Host
- Scheme
- Port
- Client IP

Kesalahan konfigurasi Reverse Proxy sering menyebabkan aplikasi menghasilkan URL HTTP walaupun pengguna mengakses melalui HTTPS.

Administrator sebaiknya melakukan pengujian setelah setiap perubahan konfigurasi Reverse Proxy.

---

# Mengurangi Informasi yang Ditampilkan

Hindari menampilkan informasi yang tidak diperlukan kepada pengguna.

Sebagai contoh.

- Versi PHP.
- Versi Web Server.
- Pesan Error Lengkap.
- Path Sistem.

Semakin sedikit informasi teknis yang tersedia bagi pengguna, semakin kecil peluang penyerang melakukan fingerprinting terhadap sistem.

---

# Mengamankan Plugin

Plugin merupakan salah satu komponen yang paling sering ditambahkan setelah instalasi selesai.

Gunakan hanya plugin yang benar-benar diperlukan.

Lakukan pemeriksaan secara berkala.

- Plugin masih digunakan.
- Plugin berasal dari sumber terpercaya.
- Plugin kompatibel dengan versi OJS.
- Plugin memperoleh pembaruan keamanan.

Plugin yang tidak lagi digunakan sebaiknya dinonaktifkan atau dihapus.

---

# Mengamankan Theme

Theme hanya bertugas mengubah tampilan website.

Hindari menggunakan theme dari sumber yang tidak jelas.

Sebelum memasang theme baru.

- Pastikan kompatibel dengan versi OJS.
- Pastikan berasal dari sumber terpercaya.
- Uji terlebih dahulu pada lingkungan pengujian.
- Lakukan backup sebelum pemasangan.

Pendekatan tersebut mengurangi risiko kerusakan aplikasi akibat theme yang tidak kompatibel.

---

# Audit Konfigurasi Berkala

Hardening bukan pekerjaan yang dilakukan satu kali.

Administrator sebaiknya melakukan audit konfigurasi secara berkala.

Periksa.

- Permission Direktori.
- Ownership.
- Plugin.
- Theme.
- File Konfigurasi.
- File Upload.
- Log Error.

Audit berkala membantu mendeteksi perubahan konfigurasi yang tidak diinginkan sebelum berkembang menjadi insiden keamanan.

---

# Ringkasan

Pada bagian ini telah dilakukan hardening terhadap beberapa aspek penting pada tingkat aplikasi.

- Installer.
- Directory Listing.
- Upload File.
- Session.
- Cookie.
- Base URL.
- HTTPS.
- Reverse Proxy.
- Plugin.
- Theme.
- Audit Konfigurasi.

Langkah-langkah tersebut memperkecil permukaan serangan serta membantu memastikan bahwa Open Journal Systems beroperasi dengan konfigurasi yang lebih aman pada lingkungan produksi.

Pada bagian berikutnya kita akan membahas pengamanan akun administrator, pengelolaan pengguna dan peran (Roles), strategi backup, monitoring keamanan, audit log, serta penyusunan checklist hardening sebelum Open Journal Systems digunakan secara penuh.

---

# Mengamankan Akses Installer

Installer Open Journal Systems hanya digunakan satu kali, yaitu pada saat proses instalasi awal.

Setelah instalasi selesai, halaman installer tidak boleh lagi dapat digunakan untuk melakukan instalasi ulang.

Hal ini dikendalikan oleh parameter berikut pada file `config.inc.php`.

```ini
installed = On
```

Periksa nilainya.

```bash
grep "^installed" \
/var/apps/ojs/htdocs/config.inc.php
```

Contoh.

```text
installed = On
```

Apabila parameter tersebut berubah menjadi `Off`, installer dapat dijalankan kembali sehingga berpotensi menimbulkan risiko keamanan.

---

# Memastikan Installer Tidak Dapat Diakses

Lakukan pengujian menggunakan browser.

```text
https://jurnal.example.go.id/index.php/index/install
```

Installer seharusnya tidak lagi menampilkan halaman instalasi.

Sebaliknya, aplikasi akan mengarahkan pengguna menuju halaman utama atau menampilkan pesan bahwa Open Journal Systems telah terinstal.

Langkah sederhana ini memastikan proses instalasi tidak dapat diulang secara tidak sengaja.

---

# Menonaktifkan Directory Listing

Directory Listing memungkinkan pengguna melihat isi suatu direktori melalui browser apabila tidak terdapat file index.

Sebagai contoh.

```text
https://jurnal.example.go.id/plugins/
```

atau.

```text
https://jurnal.example.go.id/public/
```

Apabila Directory Listing aktif, pengguna dapat melihat struktur direktori aplikasi.

Walaupun pengaturannya berada pada Nginx, administrator tetap perlu memastikan bahwa fitur tersebut benar-benar telah dinonaktifkan karena struktur direktori dapat memberikan informasi yang berguna bagi penyerang.

---

# Memastikan File Upload Tidak Dapat Dieksekusi

Open Journal Systems mengizinkan pengguna mengunggah berbagai jenis dokumen.

Namun file yang diunggah tidak boleh dapat dieksekusi sebagai program.

Sebagai contoh.

```text
shell.php
```

atau.

```text
backdoor.php
```

Apabila file tersebut dapat dijalankan oleh web server, maka seluruh server dapat dikompromikan.

Karena itu direktori upload harus diperlakukan sebagai lokasi penyimpanan data, bukan lokasi eksekusi aplikasi.

---

# Membatasi Jenis File Upload

Administrator sebaiknya hanya mengizinkan jenis file yang benar-benar diperlukan.

Contohnya.

- PDF
- DOCX
- ODT
- XLSX
- ZIP
- CSV
- Gambar

Hindari mengizinkan file yang dapat dieksekusi.

Contohnya.

- PHP
- PHTML
- PHAR
- CGI
- SH
- EXE

Semakin sedikit jenis file yang diperbolehkan, semakin kecil risiko penyalahgunaan fitur upload.

---

# Melakukan Validasi Upload

Selain membatasi ekstensi file, administrator juga perlu memastikan bahwa proses upload berjalan sesuai mekanisme Open Journal Systems.

Lakukan pengujian.

- Upload artikel PDF.
- Upload gambar.
- Upload lampiran.
- Upload file dengan ekstensi yang tidak diizinkan.

Pastikan aplikasi menolak file yang tidak sesuai dengan kebijakan organisasi.

---

# Mengamankan Session

Session digunakan untuk mempertahankan status login pengguna.

Setiap pengguna yang berhasil login akan memperoleh Session ID yang digunakan selama berinteraksi dengan aplikasi.

Administrator perlu memastikan bahwa session.

- Tidak mudah ditebak.
- Memiliki masa berlaku yang sesuai.
- Dihapus ketika pengguna logout.
- Tidak digunakan kembali setelah login ulang.

Pengelolaan session yang baik membantu mengurangi risiko pembajakan sesi (Session Hijacking).

---

# Mengamankan Cookie

Cookie menyimpan informasi yang digunakan selama proses autentikasi.

Apabila website diakses menggunakan HTTPS, cookie sebaiknya hanya dikirim melalui koneksi yang terenkripsi.

Selain itu, cookie sebaiknya tidak dapat diakses oleh JavaScript apabila tidak diperlukan.

Konfigurasi tersebut umumnya ditangani oleh PHP maupun Reverse Proxy, namun administrator tetap perlu memverifikasi bahwa browser menerima cookie dengan atribut keamanan yang sesuai.

---

# Memastikan Base URL Benar

Periksa parameter berikut pada `config.inc.php`.

```ini
base_url = "https://jurnal.example.go.id"
```

Pastikan menggunakan URL produksi yang benar.

Kesalahan konfigurasi Base URL dapat menyebabkan.

- Redirect berulang.
- Mixed Content.
- CSS tidak termuat.
- JavaScript tidak termuat.
- URL HTTP masih muncul.

---

# Memastikan HTTPS Digunakan

Seluruh akses menuju Open Journal Systems sebaiknya menggunakan HTTPS.

Lakukan pengujian.

```text
http://jurnal.example.go.id
```

Pastikan browser diarahkan menuju.

```text
https://jurnal.example.go.id
```

Selanjutnya buka beberapa halaman.

- Halaman Utama.
- Login.
- Dashboard.
- Submission.
- Download Artikel.

Seluruh halaman harus tetap menggunakan HTTPS.

---

# Memverifikasi Reverse Proxy

Apabila Open Journal Systems berada di belakang Reverse Proxy, pastikan informasi berikut diteruskan dengan benar.

- Host
- Scheme
- Port
- Client IP

Kesalahan konfigurasi Reverse Proxy sering menyebabkan aplikasi menghasilkan URL HTTP walaupun pengguna mengakses melalui HTTPS.

Administrator sebaiknya melakukan pengujian setelah setiap perubahan konfigurasi Reverse Proxy.

---

# Mengurangi Informasi yang Ditampilkan

Hindari menampilkan informasi yang tidak diperlukan kepada pengguna.

Sebagai contoh.

- Versi PHP.
- Versi Web Server.
- Pesan Error Lengkap.
- Path Sistem.

Semakin sedikit informasi teknis yang tersedia bagi pengguna, semakin kecil peluang penyerang melakukan fingerprinting terhadap sistem.

---

# Mengamankan Plugin

Plugin merupakan salah satu komponen yang paling sering ditambahkan setelah instalasi selesai.

Gunakan hanya plugin yang benar-benar diperlukan.

Lakukan pemeriksaan secara berkala.

- Plugin masih digunakan.
- Plugin berasal dari sumber terpercaya.
- Plugin kompatibel dengan versi OJS.
- Plugin memperoleh pembaruan keamanan.

Plugin yang tidak lagi digunakan sebaiknya dinonaktifkan atau dihapus.

---

# Mengamankan Theme

Theme hanya bertugas mengubah tampilan website.

Hindari menggunakan theme dari sumber yang tidak jelas.

Sebelum memasang theme baru.

- Pastikan kompatibel dengan versi OJS.
- Pastikan berasal dari sumber terpercaya.
- Uji terlebih dahulu pada lingkungan pengujian.
- Lakukan backup sebelum pemasangan.

Pendekatan tersebut mengurangi risiko kerusakan aplikasi akibat theme yang tidak kompatibel.

---

# Audit Konfigurasi Berkala

Hardening bukan pekerjaan yang dilakukan satu kali.

Administrator sebaiknya melakukan audit konfigurasi secara berkala.

Periksa.

- Permission Direktori.
- Ownership.
- Plugin.
- Theme.
- File Konfigurasi.
- File Upload.
- Log Error.

Audit berkala membantu mendeteksi perubahan konfigurasi yang tidak diinginkan sebelum berkembang menjadi insiden keamanan.

---

# Ringkasan

Pada bagian ini telah dilakukan hardening terhadap beberapa aspek penting pada tingkat aplikasi.

- Installer.
- Directory Listing.
- Upload File.
- Session.
- Cookie.
- Base URL.
- HTTPS.
- Reverse Proxy.
- Plugin.
- Theme.
- Audit Konfigurasi.

Langkah-langkah tersebut memperkecil permukaan serangan serta membantu memastikan bahwa Open Journal Systems beroperasi dengan konfigurasi yang lebih aman pada lingkungan produksi.

Pada bagian berikutnya kita akan membahas pengamanan akun administrator, pengelolaan pengguna dan peran (Roles), strategi backup, monitoring keamanan, audit log, serta penyusunan checklist hardening sebelum Open Journal Systems digunakan secara penuh.

---

# Menyusun Strategi Backup

Hardening tidak hanya bertujuan mencegah serangan, tetapi juga memastikan sistem dapat dipulihkan apabila terjadi kegagalan.

Administrator sebaiknya memiliki strategi backup yang mencakup seluruh komponen penting Open Journal Systems.

Komponen yang harus dicadangkan meliputi.

- Database.
- Direktori `ojsdata`.
- File `config.inc.php`.
- Source code (apabila terdapat modifikasi).
- Plugin tambahan.
- Theme tambahan.

Backup sebaiknya dilakukan secara terjadwal dan diverifikasi secara berkala.

---

# Backup Database

Database merupakan komponen utama yang menyimpan seluruh informasi jurnal.

Lakukan backup menggunakan utilitas database.

Contoh.

```bash
mysqldump \
db_ojs \
> backup-db_ojs.sql
```

Backup database sebaiknya disimpan pada media yang berbeda dengan server produksi.

---

# Backup Direktori Upload

Seluruh artikel, lampiran, dan dokumen yang diunggah pengguna disimpan pada direktori `ojsdata`.

Backup direktori tersebut secara berkala.

Contoh.

```bash
tar -czf \
ojsdata.tar.gz \
/var/apps/ojs/data/ojsdata
```

Tanpa backup direktori ini, artikel yang telah dipublikasikan tidak dapat dipulihkan meskipun database masih tersedia.

---

# Backup File Konfigurasi

File `config.inc.php` berisi konfigurasi penting aplikasi.

Lakukan backup setiap kali terdapat perubahan konfigurasi.

Contoh.

```bash
cp \
config.inc.php \
config.inc.php.backup
```

Backup konfigurasi akan mempercepat proses pemulihan apabila file mengalami kerusakan atau perubahan yang tidak diinginkan.

---

# Memantau Log Aplikasi

Monitoring log merupakan bagian penting dari hardening.

Administrator sebaiknya memantau.

- Log Nginx.
- Log PHP.
- Log Database.
- Log Sistem Operasi.

Perhatikan kejadian seperti.

- Login gagal berulang.
- Error PHP.
- Error Database.
- Upload yang tidak biasa.
- Perubahan konfigurasi.

Deteksi dini membantu mengurangi dampak insiden keamanan.

---

# Audit Konfigurasi Secara Berkala

Hardening bukan pekerjaan yang dilakukan satu kali.

Lakukan audit secara berkala terhadap.

- Permission direktori.
- Ownership.
- File konfigurasi.
- Plugin.
- Theme.
- Pengguna.
- Role.
- Direktori upload.

Audit membantu memastikan tidak terdapat perubahan konfigurasi yang tidak sesuai dengan kebijakan organisasi.

---

# Melakukan Update Secara Terencana

Perangkat lunak yang tidak diperbarui berpotensi mengandung kerentanan yang telah diketahui.

Lakukan pembaruan terhadap.

- Open Journal Systems.
- Plugin.
- Theme.

Sebelum melakukan pembaruan.

- Backup database.
- Backup `ojsdata`.
- Backup file konfigurasi.

Setelah pembaruan selesai, lakukan pengujian terhadap fungsi utama aplikasi.

---

# Menyiapkan Prosedur Penanganan Insiden

Setiap organisasi sebaiknya memiliki prosedur apabila terjadi insiden keamanan.

Contohnya.

- Mengisolasi server.
- Mengamankan bukti digital.
- Memulihkan layanan dari backup.
- Mengganti password administrator.
- Melakukan audit konfigurasi.
- Mendokumentasikan kronologi kejadian.

Dokumentasi tersebut membantu mempercepat proses pemulihan serta menjadi bahan evaluasi untuk meningkatkan keamanan di masa mendatang.

---

# Checklist Hardening

Sebelum Open Journal Systems digunakan pada lingkungan produksi, lakukan pemeriksaan terhadap seluruh konfigurasi.

Pastikan.

- File `config.inc.php` telah diamankan.
- Source code tidak memiliki permission yang berlebihan.
- Direktori `public` hanya berisi file publik.
- Direktori `ojsdata` berada di luar Document Root.
- Direktori cache memiliki permission yang sesuai.
- Installer tidak dapat dijalankan kembali.
- HTTPS digunakan pada seluruh halaman.
- Base URL telah benar.
- Plugin yang tidak digunakan telah dinonaktifkan.
- Theme yang tidak digunakan telah dihapus.
- Jumlah administrator telah dibatasi.
- Password administrator memenuhi kebijakan keamanan.
- Backup telah dibuat.
- Monitoring log telah disiapkan.
- Audit konfigurasi telah dilakukan.

Checklist ini dapat digunakan sebagai bagian dari proses serah terima sistem maupun audit keamanan berkala.

---

# Best Practices

Berikut beberapa praktik terbaik yang direkomendasikan.

- Gunakan prinsip **Least Privilege**.
- Lakukan backup secara berkala.
- Dokumentasikan setiap perubahan konfigurasi.
- Uji pembaruan pada lingkungan pengujian sebelum diterapkan di produksi.
- Hapus plugin dan theme yang tidak lagi digunakan.
- Lakukan audit pengguna dan role secara berkala.
- Pantau log aplikasi setiap hari.
- Simpan backup pada lokasi yang terpisah dari server produksi.

Penerapan praktik-praktik tersebut akan membantu menjaga keamanan dan stabilitas Open Journal Systems dalam jangka panjang.

---

# Kesimpulan

Hardening merupakan tahapan penting setelah proses instalasi Open Journal Systems selesai.

Melalui pengamanan file konfigurasi, source code, direktori upload, plugin, theme, akun administrator, serta penerapan strategi backup dan monitoring, risiko penyalahgunaan maupun kerusakan sistem dapat dikurangi secara signifikan.

Keamanan bukan hanya bergantung pada satu konfigurasi atau satu perangkat lunak, tetapi merupakan hasil dari kombinasi konfigurasi yang benar, pengelolaan hak akses yang baik, pemeliharaan rutin, serta disiplin dalam melakukan audit dan pembaruan.

Dengan menerapkan langkah-langkah yang dijelaskan pada artikel ini, Open Journal Systems akan memiliki fondasi keamanan yang lebih kuat sehingga lebih siap digunakan pada lingkungan produksi.

---

# Ringkasan

Pada artikel ini telah dibahas.

- Prinsip hardening Open Journal Systems.
- Pengamanan file konfigurasi.
- Pengamanan source code.
- Pengamanan direktori `public`.
- Pengamanan direktori `ojsdata`.
- Pengamanan cache.
- Pengamanan installer.
- Pengamanan upload file.
- Pengamanan session dan cookie.
- Pengamanan plugin dan theme.
- Pengelolaan administrator dan role.
- Strategi backup.
- Monitoring log.
- Audit konfigurasi.
- Checklist hardening.
- Best practices.

Hardening bukanlah proses yang selesai setelah implementasi awal. Administrator perlu melakukan pemantauan, audit, pembaruan, dan evaluasi secara berkala agar Open Journal Systems tetap aman dan andal sepanjang siklus operasionalnya.

---

# Artikel Selanjutnya

Pada seri berikutnya akan dibahas **Backup dan Restore Open Journal Systems (OJS) 3.4**, meliputi strategi backup database, backup direktori `ojsdata`, pemulihan layanan, migrasi server, serta praktik terbaik untuk menjaga ketersediaan data dan mempercepat proses disaster recovery.

---
{{< saweria >}}