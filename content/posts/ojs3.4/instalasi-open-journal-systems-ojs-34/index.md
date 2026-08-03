---
title: "Instalasi Open Journal Systems (OJS) 3.4"
date: 2026-08-02
draft: false
description: "Panduan lengkap instalasi Open Journal Systems (OJS) 3.4 pada Ubuntu Server menggunakan Nginx, Docker PHP-FPM, dan MariaDB."
tags:
  - OJS
  - Open Journal Systems
  - Docker
  - PHP-FPM
  - Nginx
  - MariaDB
  - Ubuntu Server
categories:
  - OJS
  - Docker
  - Linux

series:
  - "Membangun Open Journal Systems (OJS) 3.4"
weight: 5

author: "NR Technology"
---

# Instalasi Open Journal Systems (OJS) 3.4

## Pendahuluan

Open Journal Systems (OJS) merupakan perangkat lunak **Open Source** yang dikembangkan oleh **Public Knowledge Project (PKP)** untuk mengelola proses penerbitan jurnal ilmiah secara elektronik.

OJS mendukung seluruh alur pengelolaan jurnal, mulai dari proses pengiriman artikel (submission), penugasan editor, proses review, revisi, penyuntingan, publikasi, hingga pengindeksan metadata.

Karena digunakan oleh banyak perguruan tinggi, lembaga penelitian, maupun instansi pemerintah, proses instalasi sebaiknya dilakukan menggunakan arsitektur yang mudah dipelihara, aman, dan mudah ditingkatkan pada masa mendatang.

Pada seri artikel ini, OJS dijalankan menggunakan arsitektur sebagai berikut.

- Nginx berjalan langsung pada host.
- PHP-FPM berjalan di dalam Docker.
- MariaDB berjalan langsung pada host.
- Reverse Proxy menangani terminasi SSL/TLS.
- Direktori data aplikasi dipisahkan dari source code.

Pendekatan tersebut mempermudah proses backup, upgrade, troubleshooting, maupun migrasi ke server lain.

---

# Arsitektur Implementasi

Implementasi yang digunakan pada artikel ini ditunjukkan pada diagram berikut.

```text
                 Internet
                     │
                HTTPS 443
                     │
             Reverse Proxy
                  (Nginx)
                     │
              HTTP Port 8080
                     │
        Nginx Application Server
                     │
          Unix Socket PHP-FPM
                     │
        Docker PHP 8.3 FPM
                     │
                 MariaDB
```

Alur komunikasi berlangsung sebagai berikut.

1. Browser mengakses website menggunakan HTTPS.
2. Reverse Proxy menerima koneksi HTTPS.
3. Request diteruskan menuju Application Server.
4. Nginx menentukan apakah request berupa file statis atau request PHP.
5. Request PHP diteruskan menuju PHP-FPM menggunakan Unix Socket.
6. PHP menjalankan Open Journal Systems.
7. OJS mengambil maupun menyimpan data ke MariaDB.
8. Response dikirim kembali kepada browser.

Dengan pemisahan tersebut setiap komponen memiliki tanggung jawab yang jelas sehingga lebih mudah dikelola.

---

# Persyaratan Sistem

Sebelum melakukan instalasi, pastikan seluruh komponen berikut telah tersedia.

| Komponen | Status |
|----------|--------|
| Ubuntu Server | ✔ |
| Nginx | ✔ |
| Docker Engine | ✔ |
| Docker Compose | ✔ |
| PHP-FPM Container | ✔ |
| MariaDB | ✔ |

Artikel ini mengasumsikan seluruh komponen tersebut telah dikonfigurasi sesuai dengan seri artikel sebelumnya.

---

# Struktur Direktori

Implementasi pada artikel ini menggunakan struktur direktori berikut.

```text
/var/apps/ojs
├── htdocs
├── data
│   └── ojsdata
├── backup
└── logs
```

Sedangkan konfigurasi Docker berada pada direktori.

```text
/opt/docker/apps/ojs
├── docker-compose.yml
├── php
│   ├── php.ini
│   └── www.conf
└── logs
```

Pemisahan direktori tersebut mempermudah proses backup maupun migrasi aplikasi.

---

# Komponen yang Diinstal

Instalasi Open Journal Systems terdiri atas beberapa komponen.

- Source Code OJS
- Direktori Upload
- Database
- Administrator
- Konfigurasi Aplikasi

Seluruh komponen tersebut saling berkaitan sehingga proses instalasi sebaiknya dilakukan secara berurutan.

---

# Menyiapkan Source Code

Masuk ke direktori aplikasi.

```bash
cd /var/apps/ojs
```

Apabila direktori belum tersedia, buat terlebih dahulu.

```bash
mkdir -p /var/apps/ojs
```

Selanjutnya buat direktori untuk source code.

```bash
mkdir -p /var/apps/ojs/htdocs
```

Direktori tersebut nantinya akan menjadi **Document Root** yang digunakan oleh Nginx.

---

# Mengunduh Open Journal Systems

Masuk ke direktori sementara.

```bash
cd /tmp
```

Unduh paket instalasi Open Journal Systems 3.4 dari situs resmi PKP.

```bash
wget https://pkp.sfu.ca/ojs/download/ojs-3.4.0-x.tar.gz
```

> Ganti `3.4.0-x` dengan nomor rilis terbaru pada seri OJS 3.4 yang akan digunakan.

Ekstrak paket tersebut.

```bash
tar -xzf ojs-3.4.0-x.tar.gz
```

Salin seluruh source code ke direktori aplikasi.

```bash
cp -a ojs-3.4.0-x/. /var/apps/ojs/htdocs/
```

Periksa hasilnya.

```bash
ls -lah /var/apps/ojs/htdocs
```

Pastikan direktori seperti berikut telah tersedia.

```text
cache/
classes/
config.TEMPLATE.inc.php
dbscripts/
index.php
lib/
plugins/
public/
registry/
```

---

# Menyiapkan Direktori Upload

Open Journal Systems memisahkan source code dengan file yang diunggah pengguna.

Buat direktori upload.

```bash
mkdir -p /var/apps/ojs/data/ojsdata
```

Direktori tersebut akan digunakan untuk menyimpan.

- Artikel PDF
- Supplementary Files
- Cover Jurnal
- Gambar
- Dataset
- Lampiran

Direktori upload **tidak boleh** berada di dalam Document Root Nginx.

Dengan demikian seluruh dokumen hanya dapat diakses melalui mekanisme yang disediakan oleh OJS.

---

# Menyiapkan Direktori Cache

Pastikan direktori cache tersedia.

```bash
mkdir -p /var/apps/ojs/htdocs/cache
```

Periksa isi direktori.

```bash
ls -lah /var/apps/ojs/htdocs/cache
```

Secara umum akan terdapat beberapa subdirektori seperti.

```text
_db/
t_cache/
t_compile/
```

Direktori tersebut digunakan OJS untuk menyimpan cache aplikasi.

---

# Menyiapkan Database

Masuk ke MariaDB.

```bash
mysql -u root -p
```

Buat database baru.

```sql
CREATE DATABASE db_ojs
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;
```

Selanjutnya buat pengguna database.

```sql
CREATE USER 'ojs_user'@'%'
IDENTIFIED BY 'StrongPassword123!';
```

Berikan hak akses.

```sql
GRANT ALL PRIVILEGES
ON db_ojs.*
TO 'ojs_user'@'%';
```

Simpan perubahan.

```sql
FLUSH PRIVILEGES;
```

Periksa hasilnya.

```sql
SHOW DATABASES;
```

Pastikan database telah muncul.

```text
db_ojs
```

Kemudian keluar dari MariaDB.

```sql
EXIT;
```

---

# Memastikan PHP Berjalan

Pastikan container PHP-FPM telah aktif.

```bash
docker ps
```

Contoh hasil.

```text
STATUS

Up (healthy)
```

Selanjutnya pastikan socket PHP tersedia.

```bash
ls -lah /run/php
```

Contoh.

```text
ojs.sock
```

Apabila socket belum tersedia, selesaikan terlebih dahulu konfigurasi PHP-FPM sebelum melanjutkan proses instalasi.

---

# Memastikan Nginx Berjalan

Periksa status Nginx.

```bash
systemctl status nginx
```

Kemudian lakukan validasi konfigurasi.

```bash
nginx -t
```

Pastikan hasilnya.

```text
syntax is ok

test is successful
```

Seluruh komponen kini telah siap digunakan untuk melakukan instalasi Open Journal Systems.

Pada bagian berikutnya kita akan membahas penyesuaian hak akses direktori, konfigurasi `config.inc.php`, persiapan direktori `public` dan `ojsdata`, serta menjalankan **Web Installer** untuk pertama kalinya.

---

# Menyiapkan Hak Akses Direktori

Sebelum menjalankan Web Installer, seluruh direktori yang digunakan oleh Open Journal Systems harus memiliki hak akses yang sesuai.

Hak akses yang benar sangat penting karena OJS akan melakukan beberapa aktivitas berikut.

- Menulis file konfigurasi.
- Membuat cache.
- Mengunggah artikel.
- Mengunggah gambar.
- Menyimpan file publik.
- Menyimpan session dan cache template.

Apabila permission tidak sesuai, proses instalasi biasanya akan gagal.

---

# Menentukan User Web Server

Pada implementasi ini seluruh proses PHP dijalankan menggunakan user.

```text
www-data
```

Pastikan user tersebut memiliki hak akses terhadap direktori aplikasi.

Periksa user PHP.

```bash
docker exec -it ojs-php id
```

Contoh.

```text
uid=33(www-data)
gid=33(www-data)
```

Karena PHP berjalan sebagai **www-data**, maka direktori yang memerlukan akses tulis harus dimiliki oleh user tersebut.

---

# Mengubah Ownership

Masuk ke direktori aplikasi.

```bash
cd /var/apps/ojs
```

Ubah ownership.

```bash
chown -R www-data:www-data \
/var/apps/ojs
```

Periksa hasilnya.

```bash
ls -lah
```

Contoh.

```text
drwxr-xr-x www-data www-data htdocs
drwxr-xr-x www-data www-data data
```

---

# Mengatur Permission Source Code

Source code tidak perlu dapat ditulis setiap saat.

Atur permission direktori.

```bash
find /var/apps/ojs/htdocs \
-type d \
-exec chmod 755 {} \;
```

Kemudian atur permission file.

```bash
find /var/apps/ojs/htdocs \
-type f \
-exec chmod 644 {} \;
```

Konfigurasi tersebut mengikuti praktik umum pada sistem Linux.

---

# Menyiapkan config.inc.php

Open Journal Systems menyediakan file template.

Pastikan file berikut tersedia.

```text
config.TEMPLATE.inc.php
```

Salin menjadi.

```bash
cp \
/var/apps/ojs/htdocs/config.TEMPLATE.inc.php \
/var/apps/ojs/htdocs/config.inc.php
```

Periksa hasilnya.

```bash
ls -lah \
/var/apps/ojs/htdocs/config*
```

Contoh.

```text
config.TEMPLATE.inc.php

config.inc.php
```

---

# Permission config.inc.php

Selama proses instalasi, Web Installer perlu menulis konfigurasi ke dalam file tersebut.

Karena itu sementara waktu ubah permission menjadi.

```bash
chmod 666 \
/var/apps/ojs/htdocs/config.inc.php
```

Permission tersebut hanya digunakan selama instalasi berlangsung.

Setelah instalasi selesai, permission akan dikembalikan agar lebih aman.

---

# Menyiapkan Direktori public

Open Journal Systems menggunakan direktori **public** untuk menyimpan file yang memang boleh diakses oleh pengguna.

Periksa keberadaannya.

```bash
ls -lah \
/var/apps/ojs/htdocs/public
```

Apabila direktori belum ada.

```bash
mkdir -p \
/var/apps/ojs/htdocs/public
```

Ubah ownership.

```bash
chown -R www-data:www-data \
/var/apps/ojs/htdocs/public
```

Atur permission.

```bash
chmod 755 \
/var/apps/ojs/htdocs/public
```

Direktori **public** berbeda dengan **ojsdata**.

Direktori ini memang berada di dalam Document Root dan digunakan oleh OJS untuk menyimpan aset yang boleh diakses secara langsung.

---

# Menyiapkan Direktori ojsdata

Pastikan direktori upload telah dibuat.

```bash
ls -lah \
/var/apps/ojs/data/ojsdata
```

Apabila belum ada.

```bash
mkdir -p \
/var/apps/ojs/data/ojsdata
```

Berikan ownership.

```bash
chown -R www-data:www-data \
/var/apps/ojs/data/ojsdata
```

Atur permission.

```bash
chmod 755 \
/var/apps/ojs/data/ojsdata
```

Direktori ini merupakan lokasi penyimpanan seluruh file yang diunggah melalui Open Journal Systems.

Contohnya.

- Artikel PDF
- Supplementary Files
- Cover Jurnal
- Gambar
- Dataset Penelitian

Direktori tersebut **tidak boleh berada di dalam Document Root** sehingga file hanya dapat diakses melalui mekanisme yang disediakan oleh aplikasi.

---

# Memeriksa Direktori Cache

Pastikan seluruh direktori cache tersedia.

```bash
ls -lah \
/var/apps/ojs/htdocs/cache
```

Biasanya akan terdapat beberapa subdirektori.

```text
_db
t_cache
t_compile
```

Apabila salah satunya belum tersedia.

```bash
mkdir -p \
/var/apps/ojs/htdocs/cache/_db

mkdir -p \
/var/apps/ojs/htdocs/cache/t_cache

mkdir -p \
/var/apps/ojs/htdocs/cache/t_compile
```

---

# Permission Cache

Seluruh direktori cache harus dapat ditulis oleh PHP.

```bash
chown -R www-data:www-data \
/var/apps/ojs/htdocs/cache
```

Kemudian.

```bash
chmod -R 755 \
/var/apps/ojs/htdocs/cache
```

Cache digunakan OJS untuk.

- Template Smarty
- Cache Database
- Cache Bahasa
- Cache Metadata

Tanpa cache yang dapat ditulis, OJS tidak akan berjalan dengan baik.

---

# Memastikan Socket PHP

Pastikan PHP-FPM telah berjalan.

```bash
ls -lah /run/php
```

Contoh.

```text
ojs.sock
```

Kemudian pastikan container masih aktif.

```bash
docker ps
```

Contoh.

```text
STATUS

Up (healthy)
```

---

# Memastikan Nginx Siap

Lakukan validasi konfigurasi.

```bash
nginx -t
```

Kemudian lakukan reload.

```bash
systemctl reload nginx
```

Pastikan tidak muncul kesalahan.

---

# Membuka Web Installer

Buka browser kemudian akses.

```text
https://jurnal.example.go.id
```

Apabila seluruh konfigurasi sebelumnya benar, Open Journal Systems akan menampilkan halaman instalasi.

Halaman tersebut berisi beberapa kelompok konfigurasi.

- Language Settings
- Administrator Account
- Database Settings
- File Settings
- OAI Settings

Seluruh parameter tersebut akan dijelaskan secara rinci pada bagian berikutnya sehingga administrator memahami fungsi setiap pilihan yang tersedia selama proses instalasi.

---

# Menjalankan Web Installer

Setelah seluruh persiapan selesai dilakukan, langkah berikutnya adalah menjalankan **Web Installer** Open Journal Systems.

Buka browser kemudian akses alamat website.

```text
https://jurnal.example.go.id
```

Apabila seluruh konfigurasi sebelumnya telah benar, halaman instalasi Open Journal Systems akan ditampilkan.

Halaman tersebut terdiri atas beberapa bagian utama.

- Language Settings
- Administrator Account
- Database Settings
- File Settings
- OAI Settings

Setiap bagian memiliki fungsi yang berbeda dan akan dijelaskan secara rinci pada bagian berikut.

---

# Language Settings

Bagian pertama yang muncul adalah **Language Settings**.

Pengaturan ini menentukan bahasa yang digunakan oleh aplikasi.

Beberapa pilihan yang biasanya tersedia antara lain.

- English
- Bahasa Indonesia
- Bahasa lainnya yang disediakan oleh OJS

---

## Primary Locale

Primary Locale merupakan bahasa utama yang digunakan oleh website.

Contohnya.

```text
English (en_US)
```

atau.

```text
Bahasa Indonesia (id_ID)
```

Bahasa ini akan digunakan sebagai bahasa bawaan ketika pengguna pertama kali membuka website.

Apabila jurnal hanya menggunakan satu bahasa, cukup pilih bahasa yang paling sesuai.

---

## Additional Locales

Open Journal Systems mendukung penggunaan lebih dari satu bahasa.

Contohnya.

```
English

Bahasa Indonesia
```

Dengan mengaktifkan beberapa locale, administrator dapat menyediakan antarmuka maupun metadata artikel dalam beberapa bahasa.

Apabila jurnal hanya diterbitkan menggunakan satu bahasa, opsi ini dapat dikonfigurasi kemudian melalui menu administrasi.

---

# Administrator Account

Bagian berikutnya digunakan untuk membuat akun administrator pertama.

Administrator memiliki hak penuh terhadap seluruh konfigurasi Open Journal Systems.

Karena itu gunakan informasi yang mudah diingat namun tetap aman.

---

## Username

Masukkan nama pengguna administrator.

Contoh.

```text
admin
```

atau.

```text
journaladmin
```

Gunakan nama yang sederhana dan mudah dikenali.

---

## Password

Masukkan password administrator.

Gunakan password yang kuat.

Contoh.

```text
StrongPassword123!
```

Password sebaiknya terdiri dari kombinasi.

- Huruf besar
- Huruf kecil
- Angka
- Karakter khusus

Jangan menggunakan password yang sama dengan password database maupun password sistem operasi.

---

## Repeat Password

Masukkan kembali password yang sama.

Langkah ini digunakan untuk memastikan tidak terjadi kesalahan pengetikan.

---

## Email

Masukkan alamat email administrator.

Contoh.

```text
admin@example.go.id
```

Email tersebut akan digunakan untuk berbagai keperluan administrasi, seperti notifikasi dan proses pemulihan password.

---

# Database Settings

Open Journal Systems menyimpan seluruh data pada database.

Bagian ini digunakan untuk menghubungkan aplikasi dengan MariaDB.

---

## Database Driver

Pilih jenis database.

Pada implementasi ini digunakan.

```text
MySQL / MariaDB
```

Pilihan tersebut kompatibel dengan MariaDB.

---

## Host

Masukkan alamat server database.

Apabila MariaDB berjalan pada server yang sama.

```text
localhost
```

Apabila database berada pada server lain.

```text
db.example.local
```

atau.

```text
192.168.x.x
```

Gunakan hostname atau alamat IP sesuai dengan implementasi masing-masing.

---

## Database Name

Masukkan nama database.

Contoh.

```text
db_ojs
```

Nama tersebut harus sama dengan database yang telah dibuat sebelumnya.

---

## Username

Masukkan nama pengguna database.

Contoh.

```text
ojs_user
```

Pastikan pengguna tersebut memiliki hak akses terhadap database OJS.

---

## Password

Masukkan password database.

Gunakan password yang telah diberikan ketika membuat user MariaDB.

---

## Persistent Connection

Biasanya tersedia pilihan untuk menggunakan **Persistent Connection**.

Pada sebagian besar implementasi, opsi ini dapat dibiarkan tidak aktif.

Pengelolaan koneksi database telah ditangani dengan baik oleh PHP-FPM sehingga penggunaan persistent connection umumnya tidak memberikan keuntungan yang signifikan.

---

## Character Set

Biarkan menggunakan konfigurasi bawaan.

```text
utf8mb4
```

Character set ini mendukung penyimpanan karakter Unicode secara lengkap, termasuk berbagai simbol dan emoji.

---

# File Settings

Bagian ini menentukan lokasi penyimpanan file yang diunggah melalui Open Journal Systems.

---

## Directory for Uploads

Masukkan direktori upload.

Contoh.

```text
/var/apps/ojs/data/ojsdata
```

Direktori tersebut harus memenuhi beberapa syarat.

- Sudah dibuat sebelumnya.
- Dapat ditulis oleh PHP.
- Tidak berada di dalam Document Root.
- Tidak dapat diakses langsung melalui browser.

Seluruh artikel, gambar, lampiran, dan dokumen jurnal akan disimpan pada lokasi tersebut.

---

# OAI Settings

Open Journal Systems mendukung **Open Archives Initiative Protocol for Metadata Harvesting (OAI-PMH)**.

Fitur ini memungkinkan metadata jurnal diambil oleh berbagai layanan pengindeks.

---

## Repository Identifier

Masukkan identifier yang unik.

Contoh.

```text
ojs2.jurnal.example.go.id
```

Identifier tersebut digunakan sebagai identitas repository OAI.

Gunakan nama yang stabil dan tidak mudah berubah.

Apabila di kemudian hari domain berubah, identifier tetap dapat dipertahankan untuk menjaga konsistensi metadata.

---

## OAI Beacon

Pada beberapa versi OJS tersedia opsi untuk mengirimkan informasi repository kepada PKP.

Fitur ini bersifat opsional.

Apabila organisasi memiliki kebijakan yang membatasi komunikasi keluar, opsi tersebut dapat dibiarkan tidak aktif.

---

# Mengisi Seluruh Form

Setelah seluruh bagian selesai diisi, lakukan pemeriksaan kembali.

Pastikan informasi berikut telah benar.

- Bahasa utama.
- Username administrator.
- Password administrator.
- Email administrator.
- Host database.
- Nama database.
- Username database.
- Password database.
- Direktori upload.
- Repository Identifier.

Kesalahan pada salah satu parameter tersebut dapat menyebabkan proses instalasi gagal ataupun menghasilkan konfigurasi yang tidak sesuai.

---

# Memulai Instalasi

Setelah seluruh data diverifikasi, klik tombol instalasi pada bagian bawah halaman.

Open Journal Systems akan melakukan beberapa proses secara otomatis.

- Membuat struktur database.
- Mengisi data awal.
- Membuat akun administrator.
- Menulis file konfigurasi.
- Mengaktifkan plugin bawaan.
- Menyiapkan cache aplikasi.

Lamanya proses bergantung pada spesifikasi server, namun umumnya hanya memerlukan beberapa menit.

Pada bagian berikutnya kita akan membahas proses yang terjadi setelah instalasi selesai, melakukan validasi aplikasi, login pertama sebagai administrator, serta konfigurasi awal yang direkomendasikan sebelum Open Journal Systems digunakan pada lingkungan produksi.

---

# Proses Instalasi

Setelah tombol **Install Open Journal Systems** ditekan, installer akan menjalankan beberapa proses secara otomatis.

Tahapan yang dilakukan meliputi.

- Membuat struktur database.
- Membuat seluruh tabel.
- Mengisi data awal (seed data).
- Membuat akun administrator.
- Menulis konfigurasi ke `config.inc.php`.
- Mengaktifkan plugin bawaan.
- Menyiapkan cache aplikasi.
- Menyelesaikan proses instalasi.

Lamanya proses bergantung pada spesifikasi server.

Pada umumnya proses hanya memerlukan waktu beberapa menit.

Apabila seluruh konfigurasi benar, browser akan diarahkan menuju halaman utama Open Journal Systems.

---

# Memastikan Instalasi Berhasil

Hal pertama yang perlu dilakukan adalah memastikan tidak terdapat pesan kesalahan selama proses instalasi.

Apabila proses berhasil, beberapa kondisi berikut biasanya dapat diamati.

- Halaman utama OJS dapat diakses.
- Halaman login tersedia.
- Administrator dapat melakukan login.
- Tidak terdapat pesan error PHP.
- Tidak terdapat error database.

Selanjutnya lakukan beberapa pemeriksaan berikut.

---

# Memeriksa File Konfigurasi

Pastikan file berikut telah dibuat.

```text
/var/apps/ojs/htdocs/config.inc.php
```

Periksa keberadaannya.

```bash
ls -lah \
/var/apps/ojs/htdocs/config.inc.php
```

Selanjutnya buka file tersebut.

```bash
nano \
/var/apps/ojs/htdocs/config.inc.php
```

Pastikan beberapa parameter utama telah terisi.

Contohnya.

- Database Host
- Database Name
- Database Username
- File Directory
- Installed = On

Apabila file masih berisi konfigurasi template, kemungkinan proses instalasi belum selesai dengan benar.

---

# Mengembalikan Permission config.inc.php

Selama proses instalasi, file konfigurasi dibuat dapat ditulis oleh installer.

Setelah instalasi selesai, permission harus dikembalikan.

```bash
chmod 640 \
/var/apps/ojs/htdocs/config.inc.php
```

Periksa hasilnya.

```bash
ls -lah \
/var/apps/ojs/htdocs/config.inc.php
```

Contoh.

```text
-rw-r----- www-data www-data
```

Dengan demikian file konfigurasi tidak lagi dapat diubah secara bebas.

---

# Memeriksa Direktori Upload

Pastikan direktori upload tetap tersedia.

```bash
ls -lah \
/var/apps/ojs/data/ojsdata
```

Direktori tersebut akan berisi berbagai file yang diunggah melalui aplikasi.

Pada tahap awal biasanya direktori masih kosong.

---

# Memeriksa Direktori public

Selanjutnya periksa direktori public.

```bash
ls -lah \
/var/apps/ojs/htdocs/public
```

Direktori ini akan digunakan oleh OJS untuk menyimpan file yang memang boleh diakses melalui web.

Contohnya.

- Cover Journal
- Logo
- Gambar tertentu

Direktori ini berbeda dengan `ojsdata`.

---

# Login Pertama

Masuk ke halaman login.

```text
https://jurnal.example.go.id/login
```

Masukkan.

- Username Administrator
- Password Administrator

Apabila berhasil, Dashboard Administrator akan ditampilkan.

Dashboard merupakan pusat administrasi Open Journal Systems.

Dari halaman ini administrator dapat mengelola seluruh konfigurasi aplikasi.

---

# Memeriksa Informasi Sistem

Setelah berhasil login, lakukan pemeriksaan terhadap informasi sistem.

Beberapa hal yang sebaiknya diperiksa antara lain.

- Nama Website
- Bahasa
- Timezone
- PHP Version
- Database
- Plugin

Pemeriksaan awal ini membantu memastikan seluruh komponen berjalan sesuai dengan yang diharapkan.

---

# Mengubah Nama Website

Secara bawaan nama website biasanya masih menggunakan nilai standar.

Masuk ke menu pengaturan website kemudian ubah informasi berikut.

- Nama Website
- Deskripsi
- Kontak
- Institusi
- Email

Informasi tersebut akan ditampilkan pada halaman utama jurnal.

---

# Memeriksa Base URL

Pastikan alamat website menggunakan URL yang benar.

Contoh.

```text
https://jurnal.example.go.id
```

Apabila website berada di belakang Reverse Proxy, seluruh URL juga harus menggunakan HTTPS.

Gejala konfigurasi yang tidak tepat biasanya berupa.

- CSS tidak termuat.
- JavaScript tidak termuat.
- Redirect berulang.
- Mixed Content.
- URL masih menggunakan HTTP.

Apabila gejala tersebut muncul, periksa kembali konfigurasi Reverse Proxy dan parameter `base_url` pada `config.inc.php`.

---

# Memeriksa Plugin

Open Journal Systems menyertakan berbagai plugin bawaan.

Masuk ke menu plugin dan pastikan seluruh plugin inti telah aktif.

Plugin tersebut menangani berbagai fungsi penting seperti.

- Import
- Export
- Theme
- Metadata
- Authentication
- Reports

Plugin yang tidak digunakan dapat dinonaktifkan sesuai kebutuhan.

---

# Mengaktifkan Cron Job

Beberapa proses pada Open Journal Systems dijalankan menggunakan Scheduled Tasks.

Contohnya.

- Pengiriman Email
- Pembersihan Cache
- Indexing
- Workflow tertentu

Buat Cron Job.

```bash
crontab -e
```

Tambahkan.

```cron
*/5 * * * * php /var/apps/ojs/htdocs/tools/runScheduledTasks.php
```

Contoh di atas akan menjalankan Scheduled Tasks setiap lima menit.

Apabila menggunakan Docker PHP-FPM, sesuaikan perintah dengan metode eksekusi yang digunakan pada lingkungan masing-masing.

---

# Memeriksa Scheduled Tasks

Scheduled Tasks juga dapat dijalankan secara manual.

Masuk ke container.

```bash
docker exec -it ojs-php bash
```

Kemudian jalankan.

```bash
php tools/runScheduledTasks.php
```

Apabila tidak muncul pesan kesalahan, konfigurasi Scheduled Tasks telah berjalan dengan baik.

---

# Menguji Upload Artikel

Langkah berikutnya adalah memastikan proses upload berjalan dengan baik.

Sebagai pengujian awal.

- Login sebagai Administrator.
- Masuk ke Dashboard.
- Buat Submission baru.
- Unggah sebuah file PDF kecil.

Pastikan.

- Upload berhasil.
- File tersimpan.
- Tidak muncul error PHP.
- Tidak muncul error permission.

Apabila upload gagal, periksa kembali.

- Permission direktori `ojsdata`.
- `client_max_body_size` pada Nginx.
- `upload_max_filesize` pada PHP.
- `post_max_size` pada PHP.

---

# Menguji Download Artikel

Setelah upload berhasil, lakukan pengujian download.

Pastikan.

- File PDF dapat dibuka.
- Browser tidak menampilkan error.
- Tidak terdapat status HTTP 403.
- Tidak terdapat status HTTP 404.

Hal ini memastikan direktori upload telah dikonfigurasi dengan benar.

---

# Memeriksa Log

Selama proses pengujian, lakukan pemantauan log.

Log Nginx.

```bash
tail -f /var/log/nginx/error.log
```

Log PHP.

```bash
docker logs -f ojs-php
```

Log MariaDB.

```bash
journalctl -u mariadb -f
```

Monitoring log sangat membantu untuk mendeteksi kesalahan sejak awal.

---

# Validasi Instalasi

Sebelum Open Journal Systems digunakan oleh pengguna, lakukan pemeriksaan akhir.

Pastikan.

- Website dapat diakses.
- Login berhasil.
- Dashboard muncul.
- Database terhubung.
- Upload berhasil.
- Download berhasil.
- Scheduled Tasks berjalan.
- Tidak terdapat error pada log.
- Tidak terdapat pesan PHP Warning maupun Fatal Error.

Apabila seluruh pengujian tersebut berhasil, instalasi dapat dianggap selesai dengan baik.

Pada bagian berikutnya kita akan membahas langkah-langkah hardening setelah instalasi, termasuk pengamanan permission, perlindungan file konfigurasi, pengaturan direktori upload, strategi backup, serta praktik terbaik agar Open Journal Systems siap digunakan pada lingkungan produksi.

---

# Hardening Setelah Instalasi

Setelah Open Journal Systems berhasil diinstal, masih terdapat beberapa langkah penting yang sebaiknya dilakukan sebelum aplikasi digunakan pada lingkungan produksi.

Tujuan hardening adalah mengurangi risiko perubahan konfigurasi yang tidak disengaja, mencegah akses terhadap file sensitif, serta memastikan hanya komponen yang memang memerlukan hak tulis yang dapat melakukan perubahan.

---

# Mengembalikan Permission config.inc.php

Selama proses instalasi, file konfigurasi dibuat dapat ditulis oleh installer.

Setelah instalasi selesai, ubah kembali permission.

```bash
chmod 640 /var/apps/ojs/htdocs/config.inc.php
```

Pastikan owner tetap menggunakan user web server.

```bash
chown www-data:www-data \
/var/apps/ojs/htdocs/config.inc.php
```

Periksa hasilnya.

```bash
ls -lah /var/apps/ojs/htdocs/config.inc.php
```

Contoh.

```text
-rw-r----- 1 www-data www-data config.inc.php
```

Dengan konfigurasi tersebut file hanya dapat diubah oleh administrator sistem.

---

# Mengamankan Source Code

Source code Open Journal Systems tidak memerlukan hak tulis selama aplikasi berjalan.

Atur permission direktori.

```bash
find /var/apps/ojs/htdocs \
-type d \
-exec chmod 755 {} \;
```

Kemudian atur seluruh file.

```bash
find /var/apps/ojs/htdocs \
-type f \
-exec chmod 644 {} \;
```

Langkah ini mencegah perubahan file secara tidak sengaja maupun akibat proses yang tidak diinginkan.

---

# Memastikan Direktori Upload Tetap Terpisah

Pastikan direktori upload berada di luar Document Root.

Contoh.

```text
/var/apps/ojs/data/ojsdata
```

Jangan menggunakan.

```text
/var/apps/ojs/htdocs/files
```

atau.

```text
/var/www/html/files
```

Memisahkan direktori upload dari Document Root membantu mencegah akses langsung terhadap file yang diunggah oleh pengguna.

---

# Memeriksa Direktori public

Direktori `public` memang berada di dalam Document Root.

Namun hanya file yang memang boleh diakses publik yang sebaiknya ditempatkan pada direktori tersebut.

Contohnya.

- Logo Website
- Cover Journal
- Banner
- Gambar Publik

Jangan menyimpan.

- Artikel PDF
- Dokumen Review
- Dataset
- Lampiran

pada direktori `public`.

---

# Memastikan Direktori Cache

Periksa permission direktori cache.

```bash
ls -lah \
/var/apps/ojs/htdocs/cache
```

Pastikan owner menggunakan.

```text
www-data
```

Direktori cache tetap memerlukan hak tulis karena digunakan selama aplikasi berjalan.

---

# Menonaktifkan Installer

Setelah instalasi selesai, halaman installer seharusnya tidak dapat dijalankan kembali.

Periksa dengan membuka.

```text
https://jurnal.example.go.id/index.php/index/install
```

Apabila OJS telah terinstal dengan benar, installer akan menolak proses instalasi ulang.

Hal ini ditentukan oleh parameter `installed` pada file `config.inc.php`.

---

# Memastikan Status Installed

Buka file konfigurasi.

```bash
nano \
/var/apps/ojs/htdocs/config.inc.php
```

Pastikan parameter berikut bernilai.

```ini
installed = On
```

Parameter tersebut menandakan bahwa proses instalasi telah selesai.

---

# Backup File Konfigurasi

Buat salinan file konfigurasi.

```bash
mkdir -p \
/var/apps/ojs/backup
```

Salin file konfigurasi.

```bash
cp \
/var/apps/ojs/htdocs/config.inc.php \
/var/apps/ojs/backup/
```

Backup ini sangat berguna ketika melakukan migrasi maupun pemulihan server.

---

# Backup Database

Lakukan backup database.

```bash
mysqldump \
db_ojs \
> /var/apps/ojs/backup/db_ojs.sql
```

Periksa hasilnya.

```bash
ls -lah \
/var/apps/ojs/backup
```

Contoh.

```text
config.inc.php

db_ojs.sql
```

Backup database sebaiknya dilakukan secara berkala.

---

# Backup Direktori Upload

Selanjutnya backup direktori upload.

```bash
tar -czf \
/var/apps/ojs/backup/ojsdata.tar.gz \
/var/apps/ojs/data/ojsdata
```

Direktori upload berisi seluruh artikel dan dokumen jurnal.

Kehilangan direktori ini berarti kehilangan seluruh dokumen yang telah dipublikasikan.

---

# Memeriksa Log

Periksa log Nginx.

```bash
tail -f \
/var/log/nginx/error.log
```

Periksa log container PHP.

```bash
docker logs -f ojs-php
```

Periksa log MariaDB.

```bash
journalctl -u mariadb -f
```

Lakukan monitoring selama beberapa hari pertama setelah implementasi.

Hal ini membantu menemukan konfigurasi yang mungkin masih perlu disesuaikan.

---

# Pengujian Fungsional

Sebelum website digunakan oleh pengguna, lakukan pengujian terhadap fungsi-fungsi utama.

- Login Administrator.
- Membuat Journal.
- Membuat User.
- Membuat Submission.
- Upload Artikel.
- Download Artikel.
- Mengubah Pengaturan.
- Mengaktifkan Plugin.
- Logout.
- Login kembali.

Seluruh proses tersebut sebaiknya berhasil tanpa menghasilkan error.

---

# Validasi HTTPS

Pastikan seluruh halaman menggunakan HTTPS.

Periksa.

- Halaman utama.
- Dashboard.
- Login.
- Upload.
- Download Artikel.

Browser tidak boleh menampilkan.

- Mixed Content.
- Redirect Loop.
- SSL Warning.

Apabila ditemukan masalah tersebut, periksa kembali konfigurasi Reverse Proxy dan `base_url`.

---

# Validasi Header HTTP

Lakukan pemeriksaan.

```bash
curl -I https://jurnal.example.go.id
```

Pastikan header keamanan muncul.

```text
X-Frame-Options

X-Content-Type-Options

Referrer-Policy
```

Header tersebut menunjukkan bahwa hardening Nginx telah diterapkan.

---

# Checklist Instalasi

Sebelum Open Journal Systems digunakan pada lingkungan produksi, pastikan seluruh checklist berikut telah terpenuhi.

- Source code berhasil dipasang.
- Database berhasil dibuat.
- Installer berhasil dijalankan.
- Administrator berhasil dibuat.
- Login berhasil.
- Upload berhasil.
- Download berhasil.
- Cache berfungsi.
- Direktori upload berada di luar Document Root.
- Permission telah dikembalikan.
- File konfigurasi telah diamankan.
- Backup awal telah dibuat.
- HTTPS berjalan dengan baik.
- Tidak terdapat error pada log.

Apabila seluruh checklist telah terpenuhi, instalasi Open Journal Systems dapat dinyatakan selesai dan siap digunakan pada lingkungan produksi.

---

# Ringkasan

Pada artikel ini telah dibahas seluruh proses instalasi Open Journal Systems 3.4 menggunakan arsitektur **Nginx + Docker PHP-FPM + MariaDB**, mulai dari persiapan direktori, konfigurasi database, proses instalasi melalui Web Installer, validasi hasil instalasi, hingga langkah-langkah hardening setelah instalasi selesai.

Dengan pendekatan tersebut, lingkungan OJS menjadi lebih mudah dipelihara, lebih aman, serta siap dikembangkan lebih lanjut melalui proses backup, upgrade, monitoring, maupun migrasi ke server lain.

---

# Artikel Selanjutnya

Pada seri berikutnya akan dibahas **Backup dan Restore Open Journal Systems (OJS) 3.4**, meliputi strategi backup database, backup direktori `ojsdata`, backup source code, prosedur restore, migrasi server, serta praktik terbaik untuk pemulihan layanan setelah terjadi kegagalan sistem.