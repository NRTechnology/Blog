---
title: "Deploy Laravel Menggunakan Apache, PHP-FPM dan Hardening Apache di Ubuntu Server"
description: "Panduan lengkap menginstal Apache, PHP-FPM, melakukan deploy aplikasi Laravel, serta menerapkan hardening Apache untuk meningkatkan keamanan server produksi."
date: 2026-08-07T11:00:00+07:00
draft: false

categories:
  - Web Server

tags:
  - Apache
  - PHP-FPM
  - Laravel
  - Ubuntu Server
  - Hardening
  - Apache Security
  - PHP 8.3
  - Linux
  - Cyber Security
  - Web Server

cover:
  image: "apachelaravel-cover.png"
  alt: "Deploy Laravel Menggunakan Apache PHP-FPM dan Hardening Apache"
  caption: "Deploy Laravel Menggunakan Apache PHP-FPM dan Hardening Apache"
---

# Deploy Laravel Menggunakan Apache, PHP-FPM dan Hardening Apache di Ubuntu Server

## Pendahuluan

Laravel merupakan salah satu framework PHP yang paling populer untuk membangun aplikasi web modern. Framework ini menawarkan berbagai fitur seperti routing, ORM Eloquent, sistem autentikasi, queue, caching, hingga dukungan REST API yang memudahkan pengembangan aplikasi berskala kecil maupun enterprise.

Pada lingkungan produksi, Laravel dapat dijalankan menggunakan beberapa kombinasi web server, seperti Apache atau Nginx. Salah satu arsitektur yang banyak digunakan adalah **Apache dengan PHP-FPM (FastCGI Process Manager)**. Dibandingkan menggunakan **mod_php**, PHP-FPM memberikan performa yang lebih baik karena proses PHP berjalan secara terpisah dari web server sehingga lebih efisien dalam mengelola resource server.

Selain performa, keamanan juga menjadi faktor penting dalam proses deployment. Banyak insiden keamanan terjadi bukan karena kelemahan Laravel, melainkan akibat konfigurasi web server yang kurang tepat. Beberapa contoh kesalahan konfigurasi yang sering ditemukan antara lain:

- DocumentRoot mengarah ke direktori root aplikasi, bukan ke folder `public`.
- Seluruh direktori aplikasi dapat diakses langsung melalui web.
- Direktori upload masih mengizinkan eksekusi file PHP.
- File sensitif seperti `.env` atau `composer.json` masih dapat diakses dari browser.
- Informasi versi Apache maupun PHP masih ditampilkan kepada pengguna.

Apabila penyerang berhasil mengunggah file PHP ke direktori upload dan web server masih mengizinkan eksekusi PHP pada lokasi tersebut, maka file tersebut dapat digunakan sebagai **web shell** untuk mengambil alih server.

Melalui artikel ini, kita akan membangun lingkungan Laravel menggunakan Apache dan PHP-FPM dengan konfigurasi yang lebih aman. Seluruh contoh konfigurasi menggunakan data generik sehingga dapat diterapkan pada berbagai lingkungan server tanpa mengungkapkan informasi sensitif.

---

# Arsitektur

Implementasi pada artikel ini menggunakan arsitektur berikut.

```text
                 Internet
                     │
                     ▼
            +-----------------+
            |     Apache      |
            +-----------------+
                     │
             FastCGI (Proxy)
                     │
                     ▼
            +-----------------+
            |   PHP 8.3 FPM   |
            +-----------------+
                     │
                     ▼
           +---------------------+
           | Laravel Application |
           +---------------------+
```

Pada arsitektur tersebut, Apache hanya bertugas menerima permintaan HTTP dari pengguna, sedangkan seluruh proses eksekusi PHP ditangani oleh PHP-FPM melalui socket Unix. Pemisahan tugas ini memberikan performa yang lebih baik sekaligus mempermudah proses tuning dan monitoring.

---

# Persiapan Server

Sebelum memulai instalasi, lakukan pembaruan paket sistem agar seluruh repository menggunakan versi terbaru.

```bash
sudo apt update
sudo apt upgrade -y
```

Apabila server baru selesai diinstal, disarankan untuk melakukan reboot.

```bash
sudo reboot
```

Setelah server kembali aktif, lakukan update kembali apabila diperlukan.

---

# Instalasi Apache

Install Apache Web Server.

```bash
sudo apt install apache2 -y
```

Aktifkan service Apache agar berjalan secara otomatis saat server dinyalakan.

```bash
sudo systemctl enable apache2
sudo systemctl start apache2
```

Pastikan Apache telah berjalan dengan baik.

```bash
systemctl status apache2
```

Contoh hasil.

```text
● apache2.service - The Apache HTTP Server
     Active: active (running)
```

Periksa versi Apache.

```bash
apache2 -v
```

Contoh hasil.

```text
Server version: Apache/2.4.x
```

---

# Instalasi PHP-FPM

Laravel membutuhkan beberapa ekstensi PHP agar seluruh fitur dapat berjalan dengan baik.

Install PHP-FPM beserta ekstensi yang umum digunakan.

```bash
sudo apt install \
php8.3-fpm \
php8.3-cli \
php8.3-common \
php8.3-mysql \
php8.3-mbstring \
php8.3-xml \
php8.3-curl \
php8.3-gd \
php8.3-bcmath \
php8.3-intl \
php8.3-zip \
php8.3-opcache \
unzip -y
```

Aktifkan service PHP-FPM.

```bash
sudo systemctl enable php8.3-fpm
sudo systemctl start php8.3-fpm
```

Pastikan service berjalan.

```bash
systemctl status php8.3-fpm
```

Contoh hasil.

```text
● php8.3-fpm.service
     Active: active (running)
```

Selanjutnya pastikan socket PHP-FPM telah dibuat.

```bash
ls -l /run/php/
```

Contoh hasil.

```text
php8.3-fpm.sock
```

Socket tersebut nantinya digunakan Apache untuk meneruskan permintaan PHP ke PHP-FPM menggunakan FastCGI.

---

# Mengaktifkan Modul Apache

Beberapa modul Apache diperlukan agar Laravel dapat berjalan dengan baik.

Aktifkan modul berikut.

```bash
sudo a2enmod rewrite
sudo a2enmod headers
sudo a2enmod proxy
sudo a2enmod proxy_fcgi
sudo a2enmod expires
sudo a2enmod deflate
```

Penjelasan masing-masing modul.

| Modul | Fungsi |
|--------|--------|
| rewrite | Mendukung URL Rewrite melalui `.htaccess`. |
| headers | Menambahkan HTTP Security Header. |
| proxy | Mengaktifkan fitur reverse proxy Apache. |
| proxy_fcgi | Menghubungkan Apache dengan PHP-FPM. |
| expires | Mengatur cache browser untuk file statis. |
| deflate | Mengaktifkan kompresi HTTP. |

Restart Apache.

```bash
sudo systemctl restart apache2
```

Pastikan modul telah aktif.

```bash
apachectl -M
```

Minimal terdapat modul berikut.

```text
rewrite_module
headers_module
proxy_module
proxy_fcgi_module
expires_module
deflate_module
```

---

# Struktur Direktori Laravel

Misalkan aplikasi Laravel ditempatkan pada direktori berikut.

```text
/var/www/laravel-app
```

Struktur direktori.

```text
laravel-app/
├── app/
├── bootstrap/
├── config/
├── database/
├── public/
├── resources/
├── routes/
├── storage/
├── vendor/
└── artisan
```

Pada deployment Laravel, **DocumentRoot Apache harus mengarah ke direktori `public`**, bukan ke direktori utama aplikasi.

Dengan demikian, direktori seperti `app`, `config`, `database`, `vendor`, dan `storage` tidak dapat diakses secara langsung dari browser.

---

## Bagian Selanjutnya

Pada **Bagian 2**, kita akan membahas:

- Membuat VirtualHost Apache untuk Laravel.
- Menghubungkan Apache dengan PHP-FPM.
- Mengatur permission direktori Laravel.
- Menyiapkan aplikasi agar siap dijalankan pada server produksi.

---

# Konfigurasi VirtualHost Apache

Setelah Apache dan PHP-FPM berhasil diinstal, langkah berikutnya adalah membuat VirtualHost untuk aplikasi Laravel.

Buat file konfigurasi VirtualHost.

```bash
sudo nano /etc/apache2/sites-available/laravel.conf
```

Tambahkan konfigurasi berikut.

```apache
<VirtualHost *:80>

    ServerAdmin admin@example.com
    ServerName example.com
    ServerAlias www.example.com

    DocumentRoot /var/www/laravel-app/public
    DirectoryIndex index.php

    #################################################################
    # Public Directory
    #################################################################

    <Directory "/var/www/laravel-app/public">

        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted

        <FilesMatch "\.php$">
            SetHandler "proxy:unix:/run/php/php8.3-fpm.sock|fcgi://localhost"
        </FilesMatch>

    </Directory>

    #################################################################
    # Logging
    #################################################################

    ErrorLog ${APACHE_LOG_DIR}/laravel_error.log
    CustomLog ${APACHE_LOG_DIR}/laravel_access.log combined

</VirtualHost>
```

Simpan file kemudian aktifkan VirtualHost.

```bash
sudo a2ensite laravel.conf
```

Apabila sebelumnya masih menggunakan VirtualHost bawaan Apache, nonaktifkan terlebih dahulu.

```bash
sudo a2dissite 000-default.conf
```

Reload Apache.

```bash
sudo systemctl reload apache2
```

Periksa konfigurasi Apache.

```bash
apachectl configtest
```

Apabila tidak terdapat kesalahan, hasilnya akan menampilkan.

```text
Syntax OK
```

---

# Mengapa DocumentRoot Harus Mengarah ke Folder Public?

Banyak administrator yang masih mengarahkan **DocumentRoot** ke direktori utama aplikasi Laravel.

Contoh yang **tidak disarankan**.

```text
/var/www/laravel-app
```

Konfigurasi tersebut berpotensi membuka akses terhadap berbagai direktori penting seperti:

```
app/
bootstrap/
config/
database/
resources/
routes/
storage/
vendor/
```

Padahal seluruh direktori tersebut tidak dirancang untuk diakses melalui web.

Laravel telah menyediakan direktori **public** sebagai satu-satunya pintu masuk aplikasi.

Struktur yang benar.

```text
laravel-app/
│
├── app/
├── bootstrap/
├── config/
├── database/
├── public/
│   ├── index.php
│   ├── css/
│   ├── js/
│   └── images/
├── resources/
├── routes/
├── storage/
└── vendor/
```

Dengan menggunakan folder **public** sebagai DocumentRoot, Apache hanya akan melayani file yang memang ditujukan untuk diakses oleh pengguna.

---

# Menyiapkan Permission Laravel

Laravel membutuhkan hak tulis pada beberapa direktori untuk menyimpan cache, session, log, dan file sementara.

Direktori yang wajib dapat ditulis.

```
storage/
bootstrap/cache/
```

Ubah ownership.

```bash
sudo chown -R www-data:www-data storage
sudo chown -R www-data:www-data bootstrap/cache
```

Atur permission direktori.

```bash
sudo find storage -type d -exec chmod 775 {} \;
sudo find storage -type f -exec chmod 664 {} \;

sudo find bootstrap/cache -type d -exec chmod 775 {} \;
sudo find bootstrap/cache -type f -exec chmod 664 {} \;
```

---

# Permission Direktori Upload

Beberapa aplikasi Laravel menyimpan file upload pada direktori berikut.

```
public/storage
```

atau

```
public/uploads
```

Agar PHP dapat membuat file baru, direktori tersebut harus dapat ditulis oleh PHP-FPM.

Contoh.

```bash
sudo chown -R www-data:www-data public/storage
```

Kemudian.

```bash
sudo find public/storage -type d -exec chmod 775 {} \;
sudo find public/storage -type f -exec chmod 664 {} \;
```

Apabila aplikasi menggunakan direktori lain, sesuaikan lokasi folder upload.

---

# Apakah Memberikan Permission pada Folder Upload Berbahaya?

Pertanyaan ini sering muncul ketika melakukan hardening server.

Jawabannya adalah **tidak**, selama Apache dikonfigurasi dengan benar.

Permission Linux dan konfigurasi Apache memiliki fungsi yang berbeda.

Permission Linux menentukan siapa yang dapat membaca maupun menulis file.

Sedangkan Apache menentukan apakah file tersebut boleh dieksekusi sebagai PHP.

Sebagai contoh, PHP-FPM memerlukan hak tulis agar dapat menyimpan file upload.

```
public/storage/files
```

Namun Apache harus tetap menolak apabila terdapat file seperti.

```
shell.php
```

atau

```
backdoor.phtml
```

Dengan demikian aplikasi tetap dapat melakukan upload file tanpa memberikan kesempatan kepada penyerang untuk menjalankan web shell.

---

# Membersihkan Cache Laravel

Apabila aplikasi dipindahkan dari server lain, bersihkan cache Laravel sebelum dijalankan.

```bash
php artisan optimize:clear
```

atau.

```bash
php artisan cache:clear
php artisan config:clear
php artisan route:clear
php artisan view:clear
```

---

# Menguji Koneksi Database

Pastikan konfigurasi database pada file `.env` telah disesuaikan.

Selanjutnya lakukan pengujian.

```bash
php artisan migrate:status
```

Apabila koneksi database berhasil, Laravel akan menampilkan daftar migration yang tersedia.

---

# Menguji Aplikasi

Restart Apache dan PHP-FPM.

```bash
sudo systemctl restart php8.3-fpm
sudo systemctl restart apache2
```

Buka aplikasi melalui browser.

```
http://example.com
```

Apabila halaman Laravel berhasil ditampilkan tanpa error, berarti proses instalasi Apache dan PHP-FPM telah berhasil.

---

## Bagian Selanjutnya

Pada **Bagian 3**, kita akan membahas proses **hardening Apache** untuk Laravel, meliputi:

- Membatasi akses hanya ke direktori `public`.
- Mencegah eksekusi PHP pada direktori upload.
- Melindungi file sensitif seperti `.env`.
- Menambahkan HTTP Security Header.
- Mengurangi informasi yang ditampilkan oleh Apache.
- Menerapkan praktik keamanan yang direkomendasikan untuk server produksi.

---

# Hardening Apache untuk Laravel

Setelah aplikasi berhasil berjalan menggunakan Apache dan PHP-FPM, langkah berikutnya adalah melakukan **hardening Apache**. Tujuannya adalah mengurangi risiko eksploitasi akibat kesalahan konfigurasi web server.

Hardening yang akan dilakukan pada artikel ini meliputi:

- Membatasi akses hanya ke direktori `public`.
- Mencegah eksekusi PHP pada direktori writable.
- Melindungi file sensitif.
- Menonaktifkan informasi versi Apache.
- Menambahkan HTTP Security Header.
- Mencegah directory listing.
- Menonaktifkan metode HTTP TRACE.

Seluruh konfigurasi berikut dapat diterapkan pada server produksi.

---

# Mengapa Hardening Apache Penting?

Secara default Apache tidak mengetahui struktur aplikasi Laravel.

Tanpa konfigurasi tambahan, beberapa kondisi berikut dapat terjadi.

- Direktori upload masih dapat mengeksekusi PHP.
- File `.env` dapat diakses apabila terjadi kesalahan konfigurasi.
- Versi Apache dan PHP ditampilkan kepada pengguna.
- Directory listing masih aktif.
- HTTP TRACE masih diaktifkan.
- Informasi sistem bocor melalui response header.

Semua kondisi tersebut dapat dimanfaatkan oleh penyerang untuk memperoleh informasi maupun menjalankan kode berbahaya.

---

# Prinsip Hardening Laravel

Laravel dirancang agar hanya direktori **public** yang dapat diakses melalui web.

```
laravel-app
│
├── app
├── bootstrap
├── config
├── database
├── public  ← boleh diakses
├── resources
├── routes
├── storage
└── vendor
```

Direktori lainnya tidak boleh dapat diakses secara langsung.

---

# Membatasi Akses ke Direktori Public

Tambahkan konfigurasi berikut pada VirtualHost.

```apache
<Directory "/var/www/laravel-app">

    Options -Indexes +FollowSymLinks
    AllowOverride None
    Require all denied

</Directory>

<Directory "/var/www/laravel-app/public">

    Options -Indexes +FollowSymLinks
    AllowOverride All
    Require all granted

    <FilesMatch "\.php$">
        SetHandler "proxy:unix:/run/php/php8.3-fpm.sock|fcgi://localhost"
    </FilesMatch>

</Directory>
```

Dengan konfigurasi tersebut Apache akan menolak seluruh akses ke direktori aplikasi, kemudian hanya mengizinkan akses ke folder `public`.

Pendekatan ini dikenal sebagai **deny by default**, yaitu seluruh akses ditolak terlebih dahulu, kemudian hanya direktori yang memang diperlukan yang diizinkan.

---

# Mencegah Eksekusi PHP pada Folder Storage

Direktori `storage` digunakan Laravel untuk menyimpan log, cache, session, dan file sementara.

Direktori ini tidak boleh mengeksekusi file PHP.

Tambahkan konfigurasi berikut.

```apache
<Directory "/var/www/laravel-app/storage">

    Require all denied

    <FilesMatch "\.(php|phtml|phar|php[0-9]?)$">
        Require all denied
    </FilesMatch>

    RemoveHandler .php
    RemoveHandler .phtml
    RemoveHandler .phar

    SetHandler None

</Directory>
```

---

# Mencegah Eksekusi PHP pada Bootstrap Cache

Direktori cache juga tidak memerlukan eksekusi PHP melalui web.

```apache
<Directory "/var/www/laravel-app/bootstrap/cache">

    Require all denied

    <FilesMatch "\.(php|phtml|phar|php[0-9]?)$">
        Require all denied
    </FilesMatch>

    RemoveHandler .php
    RemoveHandler .phtml
    RemoveHandler .phar

    SetHandler None

</Directory>
```

---

# Hardening Direktori Upload

Beberapa aplikasi menyimpan file upload pada direktori berikut.

```
public/storage
```

atau

```
public/uploads
```

Direktori tersebut harus tetap dapat digunakan untuk menyimpan file, tetapi tidak boleh menjalankan file PHP.

```apache
<Directory "/var/www/laravel-app/public/storage">

    Require all granted

    <FilesMatch "\.(php|phtml|phar|php[0-9]?)$">
        Require all denied
    </FilesMatch>

    RemoveHandler .php
    RemoveHandler .phtml
    RemoveHandler .phar

    SetHandler None

</Directory>
```

Konfigurasi tersebut memungkinkan aplikasi melakukan upload file, tetapi Apache akan menolak apabila terdapat file PHP.

Sebagai contoh.

```
public/storage/shell.php
```

Ketika diakses melalui browser, Apache akan mengembalikan status **403 Forbidden** atau **404 Not Found**, bukan mengeksekusi file tersebut.

---

# Melindungi File Sensitif

Laravel memiliki beberapa file yang tidak boleh dapat diakses melalui browser.

Tambahkan konfigurasi berikut.

```apache
<FilesMatch "^(\.env|artisan|composer\.(json|lock)|phpunit\.xml)$">
    Require all denied
</FilesMatch>

<FilesMatch "^\.">
    Require all denied
</FilesMatch>
```

Konfigurasi tersebut akan memblokir akses ke file seperti.

```
.env
artisan
composer.json
composer.lock
phpunit.xml
.git
.editorconfig
```

---

# Menonaktifkan Directory Listing

Directory listing dapat menampilkan isi direktori apabila file index tidak ditemukan.

Pastikan seluruh direktori menggunakan.

```apache
Options -Indexes
```

Dengan demikian pengguna tidak dapat melihat daftar file pada suatu direktori.

---

# Hardening Global Apache

Edit file berikut.

```
/etc/apache2/conf-available/security.conf
```

Gunakan konfigurasi berikut.

```apache
ServerTokens Prod
ServerSignature Off
TraceEnable Off
FileETag None

Header always unset X-Powered-By

Header always set X-Content-Type-Options "nosniff"
Header always set X-Frame-Options "SAMEORIGIN"
Header always set Referrer-Policy "strict-origin-when-cross-origin"
Header always set Permissions-Policy "geolocation=(), microphone=(), camera=()"
```

Konfigurasi tersebut memiliki fungsi sebagai berikut.

| Konfigurasi | Fungsi |
|-------------|--------|
| ServerTokens Prod | Menyembunyikan versi Apache. |
| ServerSignature Off | Menghilangkan informasi Apache pada halaman error. |
| TraceEnable Off | Menonaktifkan metode HTTP TRACE. |
| FileETag None | Mengurangi informasi yang dikirim melalui ETag. |
| Header always unset X-Powered-By | Menyembunyikan informasi versi PHP. |
| X-Content-Type-Options | Mencegah MIME Sniffing. |
| X-Frame-Options | Mengurangi risiko Clickjacking. |
| Referrer-Policy | Mengontrol informasi Referrer. |
| Permissions-Policy | Membatasi akses terhadap fitur browser. |

---

# Mengaktifkan Modul Headers

Agar Security Header dapat diterapkan, aktifkan modul berikut.

```bash
sudo a2enmod headers
```

Restart Apache.

```bash
sudo systemctl restart apache2
```

---

# Validasi Konfigurasi Apache

Periksa konfigurasi sebelum melakukan restart.

```bash
apachectl configtest
```

Apabila konfigurasi benar.

```
Syntax OK
```

Restart Apache.

```bash
sudo systemctl restart apache2
```

---

# Menguji Security Header

Periksa response header menggunakan `curl`.

```bash
curl -I http://example.com
```

Contoh hasil.

```
Server: Apache

X-Content-Type-Options: nosniff

X-Frame-Options: SAMEORIGIN

Referrer-Policy: strict-origin-when-cross-origin

Permissions-Policy: geolocation=(), microphone=(), camera=()
```

Informasi versi Apache maupun PHP seharusnya tidak lagi ditampilkan.

---

# Menguji Hardening Upload

Sebagai pengujian, buat file berikut pada direktori upload.

```
public/storage/test.php
```

Isi file.

```php
<?php phpinfo();
```

Kemudian akses melalui browser.

```
http://example.com/storage/test.php
```

Apabila konfigurasi telah benar, Apache akan mengembalikan.

```
403 Forbidden
```

atau.

```
404 Not Found
```

Bukan menampilkan halaman **phpinfo()**.

Hal ini membuktikan bahwa Apache telah berhasil memblokir eksekusi PHP pada direktori upload.

---

# Checklist Hardening

Pastikan seluruh konfigurasi berikut telah diterapkan.

- DocumentRoot mengarah ke folder `public`.
- PHP dijalankan menggunakan PHP-FPM.
- Directory listing dinonaktifkan.
- Eksekusi PHP diblok pada direktori upload.
- Eksekusi PHP diblok pada direktori `storage`.
- Eksekusi PHP diblok pada direktori `bootstrap/cache`.
- File `.env` tidak dapat diakses.
- File `composer.json` tidak dapat diakses.
- HTTP TRACE dinonaktifkan.
- Informasi versi Apache disembunyikan.
- Informasi versi PHP disembunyikan.
- HTTP Security Header telah diterapkan.

Dengan menerapkan seluruh langkah tersebut, konfigurasi Apache menjadi lebih aman untuk menjalankan aplikasi Laravel pada lingkungan produksi.

---

## Bagian Selanjutnya

Pada **Bagian 4**, kita akan membahas proses pengujian keamanan, troubleshooting, optimasi performa Apache dan PHP-FPM, serta rekomendasi praktik terbaik untuk deployment Laravel pada server produksi.

---

# Pengujian Konfigurasi Apache

Setelah seluruh konfigurasi selesai diterapkan, langkah berikutnya adalah memastikan bahwa Apache dan PHP-FPM berjalan dengan baik serta seluruh hardening berfungsi sebagaimana mestinya.

Sebelum melakukan pengujian, validasi konfigurasi Apache.

```bash
apachectl configtest
```

Apabila konfigurasi benar, akan muncul pesan berikut.

```text
Syntax OK
```

Selanjutnya lakukan restart service.

```bash
sudo systemctl restart php8.3-fpm
sudo systemctl restart apache2
```

Pastikan kedua service berjalan normal.

```bash
systemctl status apache2
systemctl status php8.3-fpm
```

---

# Menguji Koneksi PHP-FPM

Buat sebuah file sederhana.

```bash
sudo nano /var/www/laravel-app/public/info.php
```

Isi file.

```php
<?php
phpinfo();
```

Kemudian buka melalui browser.

```
http://example.com/info.php
```

Apabila halaman informasi PHP muncul, berarti Apache telah berhasil meneruskan request PHP ke PHP-FPM.

> **Catatan**
>
> Setelah pengujian selesai, segera hapus file tersebut.

```bash
rm /var/www/laravel-app/public/info.php
```

File `phpinfo()` sebaiknya tidak dibiarkan pada server produksi karena dapat memberikan informasi mengenai konfigurasi PHP kepada pihak yang tidak berwenang.

---

# Menguji Permission Laravel

Laravel membutuhkan hak tulis pada beberapa direktori.

Pastikan direktori berikut dapat ditulis.

```
storage
bootstrap/cache
```

Lakukan pengujian.

```bash
php artisan optimize:clear
```

Apabila tidak muncul error **Permission denied**, maka permission telah dikonfigurasi dengan benar.

---

# Menguji Logging Laravel

Periksa apakah Laravel dapat menulis file log.

```bash
tail -f storage/logs/laravel.log
```

Kemudian akses aplikasi melalui browser.

Apabila terdapat aktivitas baru pada file log, berarti proses logging telah berjalan dengan baik.

---

# Menguji Security Header

Periksa response header menggunakan `curl`.

```bash
curl -I http://example.com
```

Contoh hasil.

```text
HTTP/1.1 200 OK

Server: Apache

X-Frame-Options: SAMEORIGIN

X-Content-Type-Options: nosniff

Referrer-Policy: strict-origin-when-cross-origin

Permissions-Policy: geolocation=(), microphone=(), camera=()
```

Perhatikan bahwa informasi versi Apache maupun PHP tidak lagi ditampilkan.

---

# Menguji File Sensitif

Pastikan file sensitif tidak dapat diakses.

Contoh.

```
http://example.com/.env

http://example.com/composer.json

http://example.com/composer.lock

http://example.com/artisan

http://example.com/phpunit.xml
```

Semua URL tersebut seharusnya menghasilkan.

```
403 Forbidden
```

atau.

```
404 Not Found
```

---

# Menguji Directory Listing

Buat sebuah direktori tanpa file `index`.

Kemudian akses melalui browser.

Apabila konfigurasi telah benar, Apache tidak akan menampilkan daftar file.

Sebaliknya akan muncul.

```
403 Forbidden
```

---

# Menguji Direktori Upload

Sebagai simulasi, buat file berikut.

```
public/storage/test.php
```

Isi file.

```php
<?php
phpinfo();
```

Kemudian akses.

```
http://example.com/storage/test.php
```

Apabila konfigurasi hardening telah diterapkan dengan benar, Apache akan menolak permintaan tersebut.

Hasil yang diharapkan.

```
403 Forbidden
```

atau.

```
404 Not Found
```

Apabila halaman **phpinfo()** masih muncul, berarti konfigurasi Apache masih mengizinkan eksekusi PHP pada direktori upload dan harus segera diperbaiki.

---

# Menguji Upload File

Lakukan upload sebuah file melalui aplikasi.

Pastikan file berhasil tersimpan pada direktori upload.

Selanjutnya coba akses file tersebut melalui browser untuk memastikan file dapat diunduh atau ditampilkan sesuai fungsinya.

Terakhir, lakukan pengujian menggunakan file PHP.

Apabila konfigurasi telah benar, Apache tetap akan menolak eksekusi file tersebut meskipun file berhasil diunggah.

---

# Troubleshooting

## Apache Tidak Dapat Dijalankan

Periksa konfigurasi.

```bash
apachectl configtest
```

Lihat log Apache.

```bash
journalctl -u apache2
```

atau.

```bash
tail -100 /var/log/apache2/error.log
```

---

## PHP Tidak Diproses

Pastikan PHP-FPM berjalan.

```bash
systemctl status php8.3-fpm
```

Pastikan socket tersedia.

```bash
ls -l /run/php/
```

Periksa apakah modul `proxy_fcgi` telah aktif.

```bash
apachectl -M | grep proxy_fcgi
```

---

## Laravel Menampilkan Error Permission Denied

Pastikan ownership telah sesuai.

```bash
sudo chown -R www-data:www-data storage
sudo chown -R www-data:www-data bootstrap/cache
```

Periksa permission.

```bash
sudo find storage -type d -exec chmod 775 {} \;
sudo find storage -type f -exec chmod 664 {} \;
```

---

## Upload File Gagal

Pastikan direktori upload dimiliki oleh user yang menjalankan PHP-FPM.

Contoh.

```bash
sudo chown -R www-data:www-data public/storage
```

Periksa permission.

```bash
sudo chmod -R 775 public/storage
```

---

# Optimasi Apache

Selain hardening, beberapa optimasi berikut dapat meningkatkan performa server.

Aktifkan kompresi HTTP.

```bash
sudo a2enmod deflate
```

Aktifkan cache browser.

```bash
sudo a2enmod expires
```

Restart Apache.

```bash
sudo systemctl restart apache2
```

---

# Optimasi PHP-FPM

Aktifkan OPcache.

Pastikan konfigurasi berikut telah tersedia.

```ini
opcache.enable=1
opcache.memory_consumption=256
opcache.max_accelerated_files=20000
opcache.validate_timestamps=0
```

Sesuaikan konfigurasi dengan kebutuhan aplikasi.

Setelah melakukan perubahan, restart PHP-FPM.

```bash
sudo systemctl restart php8.3-fpm
```

---

# Rekomendasi Keamanan Tambahan

Hardening Apache merupakan salah satu lapisan keamanan. Untuk meningkatkan keamanan server secara menyeluruh, beberapa langkah berikut juga direkomendasikan.

- Mengaktifkan HTTPS menggunakan sertifikat TLS.
- Melakukan pembaruan sistem secara berkala.
- Mengaktifkan firewall.
- Membatasi akses SSH.
- Melakukan backup aplikasi dan database secara rutin.
- Mengaktifkan monitoring log server.
- Menghapus file pengujian seperti `phpinfo()`.
- Menggunakan password yang kuat untuk seluruh akun administrasi.
- Melakukan audit keamanan secara berkala.

Pendekatan **defense in depth** akan memberikan perlindungan yang lebih baik dibandingkan hanya mengandalkan satu mekanisme keamanan.

---

# Kesimpulan

Menggunakan Apache bersama PHP-FPM merupakan solusi yang stabil dan efisien untuk menjalankan aplikasi Laravel pada lingkungan produksi. Dengan memisahkan proses web server dan proses PHP, pengelolaan resource menjadi lebih baik sekaligus memudahkan proses pemeliharaan.

Namun performa saja tidak cukup. Konfigurasi keamanan yang tepat merupakan bagian penting dari proses deployment. Membatasi akses hanya ke direktori `public`, melindungi file sensitif, mencegah eksekusi PHP pada direktori writable, serta menyembunyikan informasi versi Apache dan PHP merupakan beberapa langkah sederhana yang dapat mengurangi risiko eksploitasi akibat kesalahan konfigurasi.

Melalui penerapan praktik-praktik tersebut, server tidak hanya mampu menjalankan aplikasi Laravel dengan baik, tetapi juga memiliki tingkat keamanan yang lebih tinggi sehingga lebih siap digunakan pada lingkungan produksi.

---
{{< saweria >}}