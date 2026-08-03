---
title: "Konfigurasi Nginx untuk Open Journal Systems (OJS) 3.4"
date: 2026-08-02
draft: false
description: "Panduan lengkap konfigurasi Nginx untuk Open Journal Systems (OJS) 3.4 menggunakan Docker PHP-FPM, mulai dari Virtual Host, FastCGI, Unix Socket, hardening, optimasi performa hingga best practice."
tags:
  - Nginx
  - OJS
  - Docker
  - PHP-FPM
  - Linux
  - Reverse Proxy
categories:
  - Nginx
  - Docker
  - OJS

series:
  - "Membangun Open Journal Systems (OJS) 3.4"
weight: 4

author: "NR Technology"
---

# Konfigurasi Nginx untuk Open Journal Systems (OJS) 3.4

## Pendahuluan

Setelah PHP-FPM selesai dikonfigurasi, langkah berikutnya adalah membangun web server menggunakan Nginx.

Nginx bertugas menerima seluruh permintaan HTTP maupun HTTPS dari pengguna kemudian meneruskannya kepada PHP-FPM melalui Unix Socket.

Pada implementasi ini Nginx **tidak dijalankan di dalam Docker**, melainkan berjalan langsung pada sistem operasi (host). Pendekatan tersebut memberikan beberapa keuntungan dibandingkan menjalankan Nginx di dalam container.

Beberapa keuntungan tersebut antara lain:

- konfigurasi lebih sederhana
- logging lebih mudah
- performa lebih baik
- mudah diintegrasikan dengan Reverse Proxy
- lebih mudah melakukan troubleshooting
- SSL dapat dipusatkan pada Reverse Proxy

Container Docker hanya digunakan untuk menjalankan runtime PHP sehingga tanggung jawab masing-masing komponen menjadi lebih jelas.

---

# Arsitektur Implementasi

Implementasi yang digunakan pada artikel ini terdiri dari empat komponen utama.

```
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

Alur request yang terjadi adalah sebagai berikut.

1. Browser mengakses website menggunakan HTTPS.
2. Reverse Proxy menerima koneksi SSL.
3. Reverse Proxy meneruskan request menuju Application Server.
4. Nginx memeriksa apakah request merupakan file statis atau request PHP.
5. Request PHP diteruskan menuju PHP-FPM menggunakan Unix Socket.
6. PHP memproses source code Open Journal Systems.
7. OJS mengambil data dari MariaDB.
8. Hasil dikembalikan ke browser melalui Reverse Proxy.

Pemisahan tersebut membuat proses maintenance maupun troubleshooting jauh lebih mudah dibandingkan seluruh layanan dijalankan pada satu container.

---

# Struktur Direktori

Pada implementasi ini digunakan struktur direktori berikut.

```
/var/apps/ojs
├── htdocs
├── data
│   └── ojsdata
├── backup
└── logs
```

Sedangkan konfigurasi Docker berada pada direktori berikut.

```
/opt/docker/apps/ojs
├── docker-compose.yml
├── php
│   ├── php.ini
│   └── www.conf
└── logs
```

Konfigurasi Nginx berada pada lokasi standar Ubuntu.

```
/etc/nginx
├── nginx.conf
├── snippets
├── sites-available
└── sites-enabled
```

---

# Struktur Konfigurasi Nginx

Konfigurasi virtual host OJS ditempatkan pada direktori berikut.

```
/etc/nginx/sites-available/ojs.conf
```

Kemudian diaktifkan menggunakan symbolic link.

```
/etc/nginx/sites-enabled/ojs.conf
```

Pendekatan ini memudahkan administrator mengelola beberapa website dalam satu server.

---

# Membuat Virtual Host

Masuk ke direktori konfigurasi Nginx.

```bash
cd /etc/nginx/sites-available
```

Buat file konfigurasi baru.

```bash
nano ojs.conf
```

Pada implementasi ini Nginx hanya menerima koneksi dari Reverse Proxy.

Karena itu server mendengarkan pada port **8080**.

```nginx
server {

    listen 8080;

    server_name jurnal.example.go.id;

}
```

Konfigurasi tersebut merupakan fondasi awal virtual host yang akan dikembangkan pada bagian berikutnya.

---

# Mengapa Menggunakan Port 8080?

Server aplikasi tidak menerima koneksi langsung dari Internet.

Seluruh koneksi HTTPS diterima oleh Reverse Proxy.

```
Internet

↓

Reverse Proxy

↓

HTTP 8080

↓

Application Server
```

Dengan pendekatan tersebut, Application Server tidak perlu mengelola:

- sertifikat SSL
- redirect HTTP ke HTTPS
- keamanan TLS
- pembaruan sertifikat

Seluruh proses tersebut dilakukan oleh Reverse Proxy.

Pendekatan ini banyak digunakan pada lingkungan produksi karena mempermudah administrasi dan meningkatkan fleksibilitas arsitektur.

---

# Menentukan Server Name

Directive berikut menentukan nama host yang akan dilayani oleh virtual host.

```nginx
server_name jurnal.example.go.id;
```

Nginx menggunakan nilai tersebut untuk menentukan virtual host mana yang harus digunakan ketika menerima sebuah request.

Apabila server menjalankan beberapa website sekaligus, setiap website harus memiliki nilai `server_name` yang berbeda.

Contoh.

```nginx
server_name jurnal.example.go.id;
```

```nginx
server_name blog.example.go.id;
```

```nginx
server_name api.example.go.id;
```

Dengan demikian satu server Nginx dapat melayani banyak aplikasi secara bersamaan.

---

# Menentukan Document Root

Source code Open Journal Systems berada pada direktori berikut.

```nginx
root /var/apps/ojs/htdocs;
```

Directive `root` menentukan lokasi source code yang akan digunakan oleh Nginx ketika melayani file statis maupun meneruskan request PHP.

Pada implementasi ini source code tetap berada pada host, kemudian dipasang ke dalam container Docker menggunakan bind mount.

Pendekatan tersebut mempermudah proses backup dan upgrade karena source code tidak berada di dalam container.

---

# Menentukan Default Index

Tambahkan directive berikut.

```nginx
index index.php;
```

Open Journal Systems menggunakan **index.php** sebagai entry point utama.

Seluruh request yang diteruskan kepada aplikasi pada akhirnya akan diproses melalui file tersebut.

Karena itu tidak diperlukan daftar index yang panjang seperti.

```nginx
index index.php index.html index.htm;
```

Menggunakan hanya satu entry point membuat konfigurasi menjadi lebih sederhana dan lebih jelas.

---

# Konfigurasi Dasar Virtual Host

Sampai tahap ini konfigurasi virtual host menjadi seperti berikut.

```nginx
server {

    listen 8080;

    server_name jurnal.example.go.id;

    root /var/apps/ojs/htdocs;

    index index.php;

}
```

Walaupun konfigurasi tersebut sudah dapat dibaca oleh Nginx, virtual host tersebut belum dapat digunakan untuk menjalankan Open Journal Systems.

Masih diperlukan konfigurasi tambahan seperti:

- security header
- static file
- FastCGI
- Unix Socket
- upload file
- hardening
- logging

Semua konfigurasi tersebut akan dibahas secara rinci pada bagian berikutnya.

---

# Memvalidasi Konfigurasi

Setelah membuat file virtual host, lakukan pengujian sintaks.

```bash
nginx -t
```

Apabila tidak ditemukan kesalahan, hasilnya akan terlihat seperti berikut.

```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok

nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Jangan melakukan restart Nginx sebelum konfigurasi dinyatakan valid.

Dengan membiasakan menjalankan `nginx -t` setiap kali melakukan perubahan konfigurasi, administrator dapat mengurangi risiko kesalahan yang menyebabkan layanan web tidak dapat dijalankan.

Pada bagian berikutnya kita akan mulai menambahkan konfigurasi **Security Header**, pengaturan file statis, perlindungan terhadap file sensitif, serta berbagai teknik hardening dasar yang direkomendasikan untuk Open Journal Systems pada lingkungan produksi.

---

# Menambahkan Security Header

Salah satu langkah awal dalam melakukan hardening Nginx adalah menambahkan HTTP Security Header.

Security Header membantu browser menerapkan kebijakan keamanan tertentu sehingga dapat mengurangi risiko berbagai jenis serangan terhadap aplikasi web.

Pada implementasi ini digunakan konfigurasi berikut.

```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

Tambahkan konfigurasi tersebut di dalam blok `server`.

Contohnya.

```nginx
server {

    listen 8080;

    server_name jurnal.example.go.id;

    root /var/apps/ojs/htdocs;

    index index.php;

    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

}
```

Walaupun hanya terdiri dari beberapa baris, konfigurasi tersebut memberikan perlindungan tambahan terhadap browser yang mengakses aplikasi.

---

# X-Frame-Options

```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
```

Header ini mengatur apakah halaman web diperbolehkan ditampilkan di dalam sebuah **frame** atau **iframe**.

Tanpa perlindungan tersebut, sebuah website dapat ditampilkan di dalam halaman milik pihak lain.

Kondisi tersebut dapat dimanfaatkan untuk melakukan serangan **Clickjacking**, yaitu teknik yang menipu pengguna agar melakukan klik pada elemen yang sebenarnya tidak mereka sadari.

Dengan nilai berikut.

```text
SAMEORIGIN
```

Browser hanya mengizinkan halaman ditampilkan oleh website yang berasal dari origin yang sama.

Karena itu Open Journal Systems tetap dapat menggunakan iframe internal apabila diperlukan, namun tidak dapat ditampilkan oleh domain lain.

---

# X-Content-Type-Options

```nginx
add_header X-Content-Type-Options "nosniff" always;
```

Secara bawaan beberapa browser mencoba menebak tipe file yang diterima.

Sebagai contoh.

Sebuah file dikirim dengan tipe.

```text
text/plain
```

Namun browser mencoba menganggap file tersebut sebagai.

```text
text/html
```

atau.

```text
application/javascript
```

Perilaku tersebut berpotensi membuka peluang eksekusi konten yang tidak diharapkan.

Dengan menggunakan nilai.

```text
nosniff
```

Browser hanya akan menggunakan tipe konten yang benar-benar dikirimkan oleh server.

Pendekatan ini membantu mengurangi berbagai serangan yang memanfaatkan MIME Type Sniffing.

---

# Referrer Policy

```nginx
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

Browser biasanya mengirimkan informasi halaman asal ketika pengguna berpindah menuju halaman lain.

Informasi tersebut dikenal sebagai **Referrer**.

Tanpa pengaturan yang tepat, URL lengkap suatu halaman dapat ikut dikirim ke website lain.

Hal tersebut berpotensi membocorkan informasi seperti.

- parameter URL
- struktur aplikasi
- informasi pencarian
- token tertentu pada URL

Nilai berikut.

```text
strict-origin-when-cross-origin
```

merupakan pilihan yang seimbang.

Untuk navigasi dalam website yang sama, browser tetap mengirimkan informasi lengkap.

Sedangkan ketika berpindah menuju domain lain, browser hanya mengirimkan origin tanpa menyertakan path maupun parameter.

---

# Mengelola Static File

Nginx sangat baik dalam melayani file statis.

Karena itu file seperti gambar, stylesheet, maupun JavaScript sebaiknya tidak diteruskan menuju PHP.

Tambahkan konfigurasi berikut.

```nginx
location ~* \.(css|js|jpg|jpeg|png|gif|svg|ico|webp|woff|woff2|ttf|eot)$ {

    expires 30d;

    access_log off;

    log_not_found off;

}
```

Konfigurasi tersebut memungkinkan browser melakukan cache terhadap file statis sehingga jumlah request menuju server dapat dikurangi.

---

# Directive Expires

```nginx
expires 30d;
```

Directive ini memberi tahu browser bahwa file statis dapat disimpan selama tiga puluh hari.

Contohnya.

- CSS
- JavaScript
- Font
- Logo
- Banner
- Ikon

Karena file-file tersebut jarang berubah, browser tidak perlu mengunduh ulang setiap kali halaman dibuka.

Akibatnya.

- waktu loading menjadi lebih cepat
- penggunaan bandwidth menurun
- jumlah request ke server berkurang

---

# Menonaktifkan Access Log

```nginx
access_log off;
```

File statis dapat menghasilkan ribuan request setiap hari.

Misalnya.

- logo
- favicon
- stylesheet
- JavaScript
- font

Mencatat seluruh request tersebut ke dalam access log hanya akan memperbesar ukuran log tanpa memberikan informasi yang berarti.

Karena itu pencatatan access log untuk file statis dapat dimatikan.

---

# Menonaktifkan Log File Tidak Ditemukan

```nginx
log_not_found off;
```

Browser sering kali meminta file yang sebenarnya tidak tersedia.

Misalnya.

```
favicon.ico
```

atau.

```
apple-touch-icon.png
```

Tanpa konfigurasi tersebut, file log akan dipenuhi pesan **404 Not Found** yang sebenarnya tidak memerlukan perhatian administrator.

---

# Konfigurasi favicon.ico

Tambahkan blok berikut.

```nginx
location = /favicon.ico {

    access_log off;

    log_not_found off;

}
```

Browser hampir selalu meminta file `favicon.ico`.

Apabila file tersebut tidak ada, administrator biasanya tidak memerlukan informasi tersebut di dalam log.

Karena itu request tersebut diperlakukan secara khusus.

---

# Konfigurasi robots.txt

Tambahkan pula konfigurasi berikut.

```nginx
location = /robots.txt {

    access_log off;

    log_not_found off;

}
```

File `robots.txt` digunakan oleh mesin pencari untuk mengetahui halaman mana yang boleh diindeks.

Apabila file tersebut belum tersedia, administrator umumnya tidak memerlukan pencatatan setiap request ke dalam log.

---

# Melindungi Hidden File

Linux memiliki konsep **hidden file**, yaitu file yang diawali dengan karakter titik (`.`).

Contohnya.

```
.git

.env

.htaccess

.htpasswd
```

File-file tersebut tidak boleh dapat diakses melalui browser.

Tambahkan konfigurasi berikut.

```nginx
location ~ /\.(?!well-known).* {

    deny all;

}
```

---

# Mengapa Hidden File Harus Dilindungi?

Beberapa hidden file dapat berisi informasi sensitif.

Sebagai contoh.

```
.env
```

dapat berisi.

- username database
- password database
- API Key
- Secret Token

Sedangkan direktori.

```
.git
```

dapat memperlihatkan seluruh riwayat source code apabila tidak sengaja ikut dipublikasikan.

Dengan menggunakan.

```nginx
deny all;
```

Nginx akan langsung menolak request sebelum diteruskan ke PHP.

---

# Melindungi File Sensitif

Selain hidden file, beberapa file aplikasi juga tidak boleh diakses secara langsung.

Tambahkan konfigurasi berikut.

```nginx
location ~* ^/(config\.inc\.php|README|INSTALL|UPGRADE|docs|cache|lib/pkp/tests) {

    deny all;

}
```

Konfigurasi tersebut mencegah akses terhadap berbagai file maupun direktori yang hanya digunakan oleh administrator.

---

# Mengapa File Tersebut Harus Dilindungi?

Beberapa contoh file sensitif antara lain.

### config.inc.php

Berisi konfigurasi utama Open Journal Systems.

Di dalamnya terdapat.

- konfigurasi database
- lokasi file upload
- konfigurasi aplikasi

File tersebut tidak boleh dapat diakses melalui browser.

---

### README

File README sering kali memperlihatkan.

- versi aplikasi
- informasi deployment
- struktur project

Informasi tersebut dapat dimanfaatkan untuk melakukan fingerprinting terhadap server.

---

### INSTALL

File INSTALL hanya diperlukan ketika proses instalasi.

Setelah aplikasi berjalan, file tersebut tidak lagi dibutuhkan oleh pengguna.

---

### UPGRADE

File ini berisi prosedur upgrade aplikasi.

Walaupun tidak mengandung data sensitif, file tersebut tidak perlu dapat diakses dari Internet.

---

### docs

Direktori dokumentasi sering kali memuat informasi internal mengenai aplikasi.

Pada server produksi, direktori tersebut sebaiknya tidak dapat diakses secara langsung.

---

### cache

Direktori cache digunakan oleh Open Journal Systems untuk menyimpan data sementara.

Direktori tersebut bukan merupakan bagian yang perlu diakses oleh pengguna.

---

### lib/pkp/tests

Direktori ini berisi berbagai file pengujian yang digunakan selama proses pengembangan.

Direktori tersebut tidak memiliki fungsi pada lingkungan produksi dan sebaiknya tidak dapat diakses melalui browser.

---

Sampai tahap ini virtual host telah memiliki perlindungan dasar terhadap berbagai jenis file yang seharusnya tidak pernah dapat diakses dari Internet.

Pada bagian berikutnya kita akan mulai membahas konfigurasi **FastCGI**, komunikasi menggunakan **Unix Socket**, serta berbagai parameter yang digunakan Nginx ketika meneruskan request menuju PHP-FPM.

---

# Menghubungkan Nginx dengan PHP-FPM

Setelah virtual host selesai dibuat dan hardening dasar diterapkan, langkah berikutnya adalah menghubungkan Nginx dengan PHP-FPM.

Pada implementasi ini komunikasi dilakukan menggunakan **Unix Socket**, bukan TCP.

```
Browser

↓

Nginx

↓

/run/php/ojs.sock

↓

PHP-FPM

↓

Open Journal Systems
```

Pendekatan ini memberikan performa yang lebih baik sekaligus meningkatkan keamanan karena komunikasi tidak melewati jaringan TCP.

---

# Mengapa Menggunakan Unix Socket?

PHP-FPM mendukung dua metode komunikasi.

Menggunakan TCP.

```
127.0.0.1:9000
```

atau menggunakan Unix Socket.

```
/run/php/ojs.sock
```

Pada implementasi ini dipilih Unix Socket karena memiliki beberapa keuntungan.

- Tidak membuka port jaringan.
- Latensi lebih rendah.
- Throughput lebih tinggi.
- Permission dapat dikontrol menggunakan filesystem.
- Mengurangi attack surface.

Karena Nginx dan PHP-FPM berjalan pada host yang sama, penggunaan Unix Socket merupakan pilihan yang paling tepat.

---

# Konfigurasi FastCGI

Tambahkan blok berikut.

```nginx
location ~ ^/index\.php($|/) {

    fastcgi_split_path_info ^(.+?\.php)(/.*)$;

    include fastcgi_params;

    fastcgi_index index.php;

    fastcgi_param SCRIPT_FILENAME /var/www/html$fastcgi_script_name;

    fastcgi_param DOCUMENT_ROOT /var/www/html;

    fastcgi_param PATH_INFO $fastcgi_path_info;

    fastcgi_pass unix:/run/php/ojs.sock;

}
```

Konfigurasi tersebut menjadi jembatan antara Nginx dengan PHP-FPM.

Selanjutnya kita akan membahas setiap directive secara rinci.

---

# Mengapa Hanya index.php?

Perhatikan baris berikut.

```nginx
location ~ ^/index\.php($|/)
```

Open Journal Systems menggunakan pola **Front Controller**.

Artinya seluruh request diproses melalui satu file.

```
index.php
```

Contohnya.

```
/index.php/index

/index.php/login

/index.php/user

/index.php/article/view/10
```

Tidak ada file PHP lain yang perlu dapat diakses secara langsung.

Dengan membatasi hanya `index.php`, kemungkinan eksekusi file PHP lain menjadi jauh lebih kecil.

Pendekatan ini merupakan salah satu teknik hardening yang direkomendasikan.

---

# FastCGI Split Path Info

```nginx
fastcgi_split_path_info ^(.+?\.php)(/.*)$;
```

Directive ini memisahkan URL menjadi dua bagian.

Sebagai contoh.

```
/index.php/index/article/view/10
```

akan dipisahkan menjadi.

```
SCRIPT_FILENAME

↓

/index.php
```

dan.

```
PATH_INFO

↓

/index/article/view/10
```

Informasi tersebut kemudian diteruskan kepada PHP-FPM.

Open Journal Systems menggunakan informasi tersebut untuk menentukan halaman yang harus diproses.

---

# Include fastcgi_params

```nginx
include fastcgi_params;
```

Nginx menyediakan file bawaan yang berisi berbagai parameter FastCGI.

Misalnya.

- QUERY_STRING
- REQUEST_METHOD
- REQUEST_URI
- CONTENT_TYPE
- CONTENT_LENGTH

Daripada menuliskan seluruh parameter tersebut secara manual, administrator cukup menggunakan file bawaan.

Pendekatan ini membuat konfigurasi lebih ringkas dan mudah dipelihara.

---

# FastCGI Index

```nginx
fastcgi_index index.php;
```

Directive ini menentukan file PHP bawaan yang digunakan apabila request menuju sebuah direktori.

Walaupun pada implementasi OJS hampir seluruh request telah diarahkan menuju `index.php`, directive ini tetap digunakan agar konfigurasi menjadi lebih konsisten.

---

# SCRIPT_FILENAME

```nginx
fastcgi_param SCRIPT_FILENAME /var/www/html$fastcgi_script_name;
```

Parameter ini merupakan salah satu parameter terpenting pada konfigurasi FastCGI.

Nilai tersebut menentukan lokasi file PHP di dalam container.

Perhatikan bahwa Nginx berjalan pada host.

Sedangkan PHP berjalan di dalam Docker.

Source code dipasang menggunakan bind mount.

```
Host

/var/apps/ojs/htdocs

↓

Docker

/var/www/html
```

Karena itu `SCRIPT_FILENAME` harus menggunakan path yang dikenali oleh PHP di dalam container.

Bukan path pada host.

Apabila menggunakan.

```
/var/apps/ojs/htdocs
```

PHP tidak akan menemukan source code.

---

# DOCUMENT_ROOT

```nginx
fastcgi_param DOCUMENT_ROOT /var/www/html;
```

Parameter ini menentukan document root yang diketahui oleh PHP.

Nilai tersebut juga menggunakan path di dalam container.

```
/var/www/html
```

Bukan.

```
/var/apps/ojs/htdocs
```

Kesalahan pada parameter ini sering menyebabkan.

```
Primary script unknown
```

atau.

```
File not found.
```

---

# PATH_INFO

```nginx
fastcgi_param PATH_INFO $fastcgi_path_info;
```

Setelah `fastcgi_split_path_info` dijalankan, informasi PATH_INFO diteruskan menuju PHP.

Sebagai contoh.

```
Request

↓

/index.php/article/view/100
```

PHP menerima.

```
SCRIPT_FILENAME

↓

index.php
```

dan.

```
PATH_INFO

↓

article/view/100
```

Open Journal Systems menggunakan informasi tersebut untuk menentukan route aplikasi.

---

# FastCGI Pass

```nginx
fastcgi_pass unix:/run/php/ojs.sock;
```

Directive ini menginstruksikan Nginx agar meneruskan request menuju socket PHP-FPM.

Socket tersebut dibuat oleh container Docker.

```
Docker PHP

↓

/run/php/ojs.sock

↓

Nginx
```

Selama socket tersedia, Nginx dapat memproses seluruh request PHP tanpa memerlukan koneksi TCP.

---

# Mengapa Tidak Menggunakan fastcgi_pass 127.0.0.1:9000?

Sebagian besar tutorial menggunakan konfigurasi berikut.

```nginx
fastcgi_pass 127.0.0.1:9000;
```

Konfigurasi tersebut memang benar apabila PHP-FPM berjalan menggunakan TCP.

Namun pada implementasi ini PHP-FPM menggunakan Unix Socket sehingga konfigurasi tersebut tidak digunakan.

Selain memberikan performa yang lebih baik, penggunaan socket juga mengurangi kebutuhan membuka port tambahan pada server.

---

# Alur Request PHP

Secara sederhana alur request dapat digambarkan sebagai berikut.

```
Browser

↓

Nginx

↓

Location index.php

↓

FastCGI

↓

Unix Socket

↓

PHP-FPM

↓

Open Journal Systems

↓

MariaDB

↓

PHP-FPM

↓

Nginx

↓

Browser
```

Seluruh proses tersebut berlangsung dalam waktu yang sangat singkat sehingga pengguna hanya melihat halaman web yang telah selesai diproses.

---

# Mengapa Tidak Mengeksekusi Seluruh File PHP?

Banyak tutorial menggunakan konfigurasi seperti berikut.

```nginx
location ~ \.php$ {

    ...

}
```

Konfigurasi tersebut akan mencoba mengeksekusi seluruh file PHP.

Pada Open Journal Systems pendekatan tersebut tidak diperlukan.

Sebaliknya, implementasi ini hanya mengizinkan eksekusi terhadap.

```
index.php
```

Keuntungan pendekatan tersebut antara lain.

- Attack surface lebih kecil.
- Mengurangi risiko eksekusi file PHP yang tidak diinginkan.
- Selaras dengan arsitektur Front Controller OJS.
- Lebih mudah diaudit.

Pendekatan ini merupakan salah satu teknik hardening yang direkomendasikan untuk aplikasi modern.

---

# Konfigurasi Virtual Host Sampai Tahap Ini

Setelah seluruh konfigurasi FastCGI ditambahkan, virtual host akan memiliki struktur sebagai berikut.

```
Server

├── Security Header
├── Static File
├── Hidden File Protection
├── Sensitive File Protection
└── FastCGI
        │
        ▼
    PHP-FPM
```

Walaupun sudah dapat menjalankan Open Journal Systems, masih terdapat beberapa konfigurasi penting yang perlu ditambahkan agar server siap digunakan pada lingkungan produksi.

Pada bagian berikutnya kita akan membahas konfigurasi upload file, timeout, buffering, logging, reverse proxy header, serta berbagai teknik hardening lanjutan yang digunakan pada implementasi produksi.

---

# Mengatur Ukuran Upload File

Open Journal Systems digunakan untuk mengunggah berbagai jenis dokumen seperti artikel, lampiran, dataset, maupun gambar.

Karena itu Nginx perlu dikonfigurasi agar mampu menerima file berukuran besar.

Tambahkan konfigurasi berikut.

```nginx
client_max_body_size 256M;
```

Nilai tersebut menentukan ukuran maksimum request yang diterima oleh Nginx.

Apabila pengguna mengunggah file yang melebihi batas tersebut, Nginx akan langsung mengembalikan pesan.

```
413 Request Entity Too Large
```

Sebaiknya nilai ini disesuaikan dengan konfigurasi PHP.

Sebagai contoh.

```
client_max_body_size = 256M

↓

upload_max_filesize = 256M

↓

post_max_size = 256M
```

Dengan konfigurasi tersebut seluruh komponen memiliki batas upload yang sama.

---

# Client Body Timeout

Tambahkan konfigurasi berikut.

```nginx
client_body_timeout 300;
```

Directive ini menentukan berapa lama Nginx menunggu browser mengirimkan request.

Parameter ini sangat penting ketika pengguna mengunggah file yang besar menggunakan koneksi Internet yang lambat.

Apabila waktu tersebut terlampaui sebelum upload selesai, Nginx akan menghentikan koneksi.

Nilai lima menit umumnya sudah mencukupi untuk sebagian besar implementasi Open Journal Systems.

---

# Send Timeout

Tambahkan.

```nginx
send_timeout 300;
```

Directive ini menentukan waktu maksimum ketika Nginx mengirimkan response kepada browser.

Nilai yang terlalu kecil dapat menyebabkan download artikel PDF terputus pada koneksi yang lambat.

Karena OJS sering digunakan untuk mengunduh dokumen berukuran besar, nilai lima menit merupakan pilihan yang cukup aman.

---

# Konfigurasi Buffer FastCGI

FastCGI menggunakan buffer untuk menampung response dari PHP sebelum dikirimkan kepada browser.

Tambahkan konfigurasi berikut.

```nginx
fastcgi_buffer_size 32k;

fastcgi_buffers 8 16k;

fastcgi_busy_buffers_size 64k;
```

Konfigurasi tersebut membantu Nginx menangani response yang relatif besar tanpa harus langsung menggunakan media penyimpanan sementara.

---

# FastCGI Buffer Size

```nginx
fastcgi_buffer_size 32k;
```

Parameter ini menentukan ukuran buffer pertama yang digunakan untuk menerima header response dari PHP.

Nilai tiga puluh dua kilobyte umumnya sudah mencukupi untuk aplikasi seperti Open Journal Systems.

---

# FastCGI Buffers

```nginx
fastcgi_buffers 8 16k;
```

Directive ini menentukan jumlah buffer tambahan.

Pada konfigurasi tersebut tersedia.

```
8 Buffer

↓

16 KB

↓

Total 128 KB
```

Buffer tersebut digunakan untuk menyimpan isi response sebelum dikirim menuju browser.

---

# Busy Buffer

```nginx
fastcgi_busy_buffers_size 64k;
```

Ketika sebagian buffer sedang dikirim kepada browser, sebagian buffer lainnya masih menerima data dari PHP.

Busy buffer menentukan jumlah maksimum buffer yang boleh digunakan secara bersamaan.

Nilai yang terlalu kecil dapat mengurangi performa pada response yang besar.

---

# FastCGI Request Buffering

Tambahkan.

```nginx
fastcgi_request_buffering on;
```

Dengan konfigurasi tersebut Nginx menerima seluruh request dari browser terlebih dahulu sebelum meneruskannya menuju PHP.

Keuntungan pendekatan ini.

- PHP tidak perlu menunggu upload selesai.
- Worker PHP lebih cepat tersedia.
- Penggunaan resource lebih efisien.

Pendekatan tersebut sangat sesuai untuk aplikasi berbasis upload seperti Open Journal Systems.

---

# FastCGI Connect Timeout

```nginx
fastcgi_connect_timeout 60s;
```

Directive ini menentukan lama waktu Nginx menunggu koneksi menuju PHP-FPM.

Apabila socket tidak dapat diakses dalam waktu tersebut, request akan dianggap gagal.

---

# FastCGI Send Timeout

```nginx
fastcgi_send_timeout 300s;
```

Parameter ini menentukan lama waktu Nginx mengirimkan request menuju PHP.

Nilai lima menit memberikan ruang yang cukup ketika request berisi upload dokumen yang besar.

---

# FastCGI Read Timeout

```nginx
fastcgi_read_timeout 300s;
```

Setelah request diterima PHP, Nginx akan menunggu response.

Apabila PHP membutuhkan waktu cukup lama, misalnya ketika.

- import XML
- upload artikel
- export metadata
- instalasi plugin

Nginx tetap mempertahankan koneksi hingga lima menit.

Tanpa konfigurasi tersebut pengguna dapat mengalami pesan.

```
504 Gateway Timeout
```

---

# FastCGI Intercept Errors

```nginx
fastcgi_intercept_errors off;
```

Directive ini menentukan apakah Nginx mengambil alih halaman error yang dikirim PHP.

Pada implementasi ini nilainya dimatikan.

Dengan demikian Open Journal Systems tetap dapat menampilkan halaman kesalahan miliknya sendiri.

Pendekatan tersebut memberikan pengalaman pengguna yang lebih konsisten.

---

# Reverse Proxy Header

Karena Nginx Application Server berada di belakang Reverse Proxy, beberapa header perlu diteruskan menuju PHP.

Tambahkan konfigurasi berikut.

```nginx
fastcgi_param HTTP_HOST              $http_host;

fastcgi_param SERVER_NAME            $host;

fastcgi_param REQUEST_SCHEME         $scheme;

fastcgi_param SERVER_PORT            $server_port;

fastcgi_param HTTP_X_FORWARDED_HOST  $http_x_forwarded_host;

fastcgi_param HTTP_X_FORWARDED_PROTO $http_x_forwarded_proto;

fastcgi_param HTTP_X_FORWARDED_PORT  $http_x_forwarded_port;

fastcgi_param HTTP_X_FORWARDED_FOR   $proxy_add_x_forwarded_for;
```

Header tersebut memungkinkan Open Journal Systems mengetahui alamat asli yang digunakan oleh pengguna.

---

# HTTP_HOST

```nginx
fastcgi_param HTTP_HOST $http_host;
```

Parameter ini meneruskan nilai Host dari browser menuju PHP.

Open Journal Systems menggunakan informasi tersebut ketika membentuk URL internal.

---

# REQUEST_SCHEME

```nginx
fastcgi_param REQUEST_SCHEME $scheme;
```

Directive ini memberi tahu PHP apakah request diterima menggunakan HTTP atau HTTPS.

Walaupun Application Server hanya menerima HTTP dari Reverse Proxy, informasi tersebut tetap diperlukan.

Reverse Proxy akan meneruskan nilai sebenarnya melalui.

```
X-Forwarded-Proto
```

---

# X-Forwarded-Proto

```nginx
fastcgi_param HTTP_X_FORWARDED_PROTO $http_x_forwarded_proto;
```

Parameter ini sangat penting.

Open Journal Systems menggunakan informasi tersebut untuk mengetahui apakah pengguna sebenarnya mengakses website menggunakan HTTPS.

Tanpa parameter ini OJS dapat menganggap koneksi masih menggunakan HTTP sehingga menghasilkan.

- redirect yang salah
- mixed content
- URL HTTP
- CSS tidak termuat
- JavaScript tidak termuat

Permasalahan tersebut cukup sering dijumpai ketika OJS berada di belakang Reverse Proxy.

---

# X-Forwarded-Port

```nginx
fastcgi_param HTTP_X_FORWARDED_PORT $http_x_forwarded_port;
```

Header ini memberi tahu PHP port asli yang digunakan oleh pengguna.

Pada implementasi ini nilainya umumnya.

```
443
```

Walaupun Nginx menerima request melalui port.

```
8080
```

---

# X-Forwarded-Host

```nginx
fastcgi_param HTTP_X_FORWARDED_HOST $http_x_forwarded_host;
```

Header ini meneruskan nama host asli dari Reverse Proxy.

Open Journal Systems menggunakan informasi tersebut ketika menghasilkan berbagai URL internal.

---

# X-Forwarded-For

```nginx
fastcgi_param HTTP_X_FORWARDED_FOR $proxy_add_x_forwarded_for;
```

Tanpa konfigurasi tersebut PHP hanya akan melihat alamat IP Reverse Proxy.

Dengan meneruskan header ini administrator tetap dapat mengetahui alamat IP asli pengguna.

Informasi tersebut juga sangat berguna ketika melakukan audit maupun analisis keamanan.

---

# Mitigasi HTTPoxy

Tambahkan konfigurasi berikut.

```nginx
fastcgi_param HTTP_PROXY "";
```

Konfigurasi tersebut digunakan untuk mencegah kerentanan **HTTPoxy**.

Serangan tersebut memanfaatkan header.

```
Proxy:
```

yang dikirim oleh pengguna.

Dengan mengosongkan variabel.

```
HTTP_PROXY
```

PHP tidak akan menggunakan nilai yang dikirim oleh pengguna.

Walaupun kerentanan ini sudah cukup lama dikenal, praktik ini masih direkomendasikan pada lingkungan produksi.

---

# Konfigurasi FastCGI Lengkap

Sampai tahap ini blok FastCGI akan terlihat seperti berikut.

```
FastCGI

├── Unix Socket
├── SCRIPT_FILENAME
├── DOCUMENT_ROOT
├── PATH_INFO
├── Upload
├── Timeout
├── Buffer
├── Reverse Proxy Header
└── HTTPoxy Protection
```

Konfigurasi tersebut telah memenuhi sebagian besar kebutuhan Open Journal Systems pada lingkungan produksi dan memberikan dasar yang kuat untuk komunikasi antara Nginx dan PHP-FPM.

Pada bagian terakhir kita akan membahas proses validasi konfigurasi, pengujian Nginx, troubleshooting, optimasi akhir, best practices, serta menyusun konfigurasi virtual host secara lengkap sehingga siap digunakan pada server produksi.

---

# Konfigurasi Virtual Host Lengkap

Setelah seluruh konfigurasi dijelaskan pada bagian-bagian sebelumnya, berikut adalah contoh konfigurasi virtual host yang siap digunakan pada lingkungan produksi.

```nginx
server {

    listen 8080 ssl http2;

    server_name jurnal.example.go.id;

    root /var/apps/ojs/htdocs;

    index index.php;

    charset utf-8;

    server_tokens off;

    ssl_certificate     /var/ssl_cert/star.example.go.id.crt;
    ssl_certificate_key /var/ssl_cert/star.example.go.id.key;

    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:20m;
    ssl_session_tickets off;

    ssl_protocols TLSv1.2 TLSv1.3;

    ssl_prefer_server_ciphers off;

    ssl_ciphers HIGH:!aNULL:!MD5;

    client_max_body_size 256M;
    client_body_timeout 300;
    send_timeout 300;

    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    location / {

        try_files $uri $uri/ /index.php$is_args$args;

    }

    location ~* \.(css|js|jpg|jpeg|png|gif|svg|ico|webp|woff|woff2|ttf|eot)$ {

        expires 30d;

        access_log off;

        log_not_found off;

    }

    location = /favicon.ico {

        access_log off;

        log_not_found off;

    }

    location = /robots.txt {

        access_log off;

        log_not_found off;

    }

    location ~ /\.(?!well-known).* {

        deny all;

    }

    location ~* ^/(config\.inc\.php|README|INSTALL|UPGRADE|docs|cache|lib/pkp/tests) {

        deny all;

    }

    location ~* ^/(public|files)/.*\.php$ {

        deny all;

    }

    location ~ ^/index\.php($|/) {

        fastcgi_split_path_info ^(.+?\.php)(/.*)$;

        include fastcgi_params;

        fastcgi_index index.php;

        fastcgi_param SCRIPT_FILENAME /var/www/html$fastcgi_script_name;

        fastcgi_param DOCUMENT_ROOT /var/www/html;

        fastcgi_param PATH_INFO $fastcgi_path_info;

        fastcgi_param HTTP_HOST              $http_host;
        fastcgi_param SERVER_NAME            $host;
        fastcgi_param REQUEST_SCHEME         $scheme;
        fastcgi_param SERVER_PORT            $server_port;

        fastcgi_param HTTP_X_FORWARDED_HOST  $http_x_forwarded_host;
        fastcgi_param HTTP_X_FORWARDED_PROTO $http_x_forwarded_proto;
        fastcgi_param HTTP_X_FORWARDED_PORT  $http_x_forwarded_port;
        fastcgi_param HTTP_X_FORWARDED_FOR   $proxy_add_x_forwarded_for;

        fastcgi_param HTTP_PROXY "";

        fastcgi_connect_timeout 60s;
        fastcgi_send_timeout 300s;
        fastcgi_read_timeout 300s;

        fastcgi_request_buffering on;

        fastcgi_buffer_size 32k;
        fastcgi_buffers 8 16k;
        fastcgi_busy_buffers_size 64k;

        fastcgi_intercept_errors off;

        fastcgi_pass unix:/run/php/ojs.sock;

    }

}
```

Konfigurasi tersebut dirancang untuk digunakan bersama arsitektur yang telah dibangun pada seri artikel ini, yaitu Nginx berjalan langsung pada host, sedangkan PHP-FPM dijalankan di dalam container Docker dan berkomunikasi menggunakan Unix Socket.

---

# Mengaktifkan Virtual Host

Setelah file konfigurasi selesai dibuat, aktifkan virtual host menggunakan symbolic link.

```bash
ln -s /etc/nginx/sites-available/ojs.conf \
      /etc/nginx/sites-enabled/ojs.conf
```

Apabila symbolic link sudah ada, perintah tersebut tidak perlu dijalankan kembali.

Daftar virtual host aktif dapat dilihat menggunakan.

```bash
ls -lah /etc/nginx/sites-enabled
```

---

# Memvalidasi Konfigurasi

Sebelum melakukan reload ataupun restart Nginx, lakukan validasi sintaks.

```bash
nginx -t
```

Apabila konfigurasi benar, hasilnya akan terlihat seperti berikut.

```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok

nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Jangan melakukan reload apabila masih terdapat kesalahan sintaks.

---

# Reload Nginx

Apabila konfigurasi telah dinyatakan valid, muat ulang konfigurasi.

```bash
systemctl reload nginx
```

Reload hanya memuat ulang konfigurasi tanpa menghentikan koneksi yang sedang aktif.

Hal ini berbeda dengan restart yang akan menghentikan seluruh worker kemudian menjalankannya kembali.

Pada lingkungan produksi, penggunaan **reload** lebih disarankan dibandingkan **restart**.

---

# Memverifikasi PHP-FPM

Pastikan socket PHP tersedia.

```bash
ls -lah /run/php
```

Contoh.

```text
srw-rw---- 1 www-data www-data ojs.sock
```

Selanjutnya pastikan container PHP berjalan.

```bash
docker ps
```

Contoh.

```text
STATUS

Up (healthy)
```

---

# Menguji Melalui Browser

Buka alamat website.

```text
https://jurnal.example.go.id
```

Pastikan halaman utama Open Journal Systems berhasil ditampilkan.

Kemudian lakukan beberapa pengujian sederhana.

- Membuka halaman login.
- Membuka halaman About.
- Membuka daftar artikel.
- Mengunduh artikel PDF.
- Mengunggah file melalui Dashboard.
- Membuka halaman administrasi.

Seluruh halaman tersebut seharusnya dapat diakses tanpa menghasilkan kesalahan.

---

# Menguji Menggunakan curl

Pengujian juga dapat dilakukan melalui terminal.

Periksa header HTTP.

```bash
curl -I https://jurnal.example.go.id
```

Pastikan header keamanan telah muncul.

Sebagai contoh.

```text
X-Frame-Options: SAMEORIGIN

X-Content-Type-Options: nosniff

Referrer-Policy: strict-origin-when-cross-origin
```

Selanjutnya pastikan aplikasi menghasilkan status.

```text
HTTP/2 200 OK
```

---

# Memverifikasi Reverse Proxy

Apabila server berada di belakang Reverse Proxy, lakukan pemeriksaan apakah Open Journal Systems menghasilkan URL HTTPS.

Periksa.

- CSS berhasil dimuat.
- JavaScript berhasil dimuat.
- Tidak terdapat Mixed Content.
- Redirect tidak berulang.
- URL menggunakan HTTPS.

Apabila salah satu pengujian tersebut gagal, periksa kembali konfigurasi.

```
X-Forwarded-Proto

X-Forwarded-Port

X-Forwarded-Host
```

Ketiga header tersebut sangat penting ketika OJS dijalankan di belakang Reverse Proxy.

---

# Troubleshooting

## 502 Bad Gateway

Penyebab yang paling sering.

- PHP-FPM tidak berjalan.
- Socket tidak ditemukan.
- Permission socket salah.

Periksa.

```bash
docker ps

ls -lah /run/php

docker logs ojs-php
```

---

## 404 Not Found

Periksa.

```nginx
root

try_files

index
```

Kesalahan pada salah satu directive tersebut dapat menyebabkan request tidak diteruskan menuju `index.php`.

---

## Primary Script Unknown

Kesalahan ini hampir selalu berkaitan dengan.

```
SCRIPT_FILENAME
```

Pastikan path yang digunakan merupakan path di dalam container.

```
/var/www/html
```

Bukan.

```
/var/apps/ojs/htdocs
```

---

## CSS dan JavaScript Tidak Termuat

Periksa.

- `base_url` pada `config.inc.php`
- `X-Forwarded-Proto`
- `X-Forwarded-Host`
- `X-Forwarded-Port`

Permasalahan ini paling sering terjadi ketika OJS berada di belakang Reverse Proxy.

---

## Upload File Gagal

Pastikan ketiga komponen berikut menggunakan nilai yang sesuai.

Nginx.

```text
client_max_body_size
```

PHP.

```text
upload_max_filesize
```

PHP.

```text
post_max_size
```

Ketiga parameter tersebut harus saling konsisten.

---

# Best Practices

Implementasi pada artikel ini menerapkan beberapa praktik terbaik sebagai berikut.

- Nginx berjalan langsung pada host.
- PHP-FPM berjalan di dalam Docker.
- MariaDB berjalan langsung pada host.
- Menggunakan Unix Socket.
- Menggunakan satu entry point (`index.php`).
- Menggunakan filesystem read-only pada container.
- Memisahkan source code dan data aplikasi.
- Memisahkan konfigurasi Docker dan aplikasi.
- Menggunakan security header.
- Melindungi hidden file.
- Melindungi file sensitif.
- Menonaktifkan `server_tokens`.
- Menggunakan bind mount hanya pada direktori yang diperlukan.
- Memvalidasi konfigurasi sebelum melakukan reload.
- Menggunakan `systemctl reload nginx` setelah perubahan konfigurasi.

Pendekatan tersebut menghasilkan lingkungan yang lebih aman, mudah dipelihara, serta mudah direproduksi pada server lain.

---

# Kesimpulan

Nginx merupakan komponen yang sangat penting dalam implementasi Open Journal Systems karena bertanggung jawab menerima seluruh permintaan dari pengguna, melayani file statis, serta meneruskan request PHP menuju PHP-FPM.

Melalui konfigurasi yang tepat, Nginx tidak hanya berfungsi sebagai web server, tetapi juga menjadi lapisan keamanan pertama sebelum request diproses oleh aplikasi.

Pada artikel ini telah dibahas konfigurasi virtual host, FastCGI, Unix Socket, pengelolaan upload file, security header, perlindungan terhadap file sensitif, optimasi buffering, konfigurasi timeout, integrasi dengan Reverse Proxy, serta berbagai praktik terbaik yang direkomendasikan untuk lingkungan produksi.

Dengan konfigurasi tersebut, Open Journal Systems siap dijalankan menggunakan arsitektur **Nginx + Docker PHP-FPM + MariaDB** yang aman, ringan, mudah dipelihara, dan sesuai untuk implementasi pada lingkungan produksi.

---

# Ringkasan

Pada artikel ini telah dibahas:

- Arsitektur Nginx untuk OJS
- Struktur konfigurasi
- Virtual Host
- Security Header
- Static File
- Hidden File Protection
- Sensitive File Protection
- FastCGI
- Unix Socket
- Reverse Proxy
- Upload File
- Timeout
- Buffering
- Logging
- Validasi Konfigurasi
- Troubleshooting
- Best Practices
  