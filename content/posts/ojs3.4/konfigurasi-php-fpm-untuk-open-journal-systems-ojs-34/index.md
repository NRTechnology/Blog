---
title: "Konfigurasi PHP-FPM untuk Open Journal Systems (OJS) 3.4"
date: 2026-08-02
draft: false
description: "Panduan lengkap konfigurasi PHP-FPM untuk Open Journal Systems (OJS) 3.4 menggunakan Docker, mulai dari php.ini, www.conf, optimasi performa, hingga hardening keamanan."
tags:
  - OJS
  - PHP
  - PHP-FPM
  - Docker
  - Linux
  - Nginx
categories:
  - OJS
  - Docker
  - Linux

series:
  - "Membangun Open Journal Systems (OJS) 3.4"
weight: 3

author: "NR Technology"
cover:
  image: "../assets/ojs-cover.png"
  alt: "Open Journal Systems (OJS) 3.4"
  caption: "Seri Membangun Open Journal Systems (OJS) 3.4"
---

# Konfigurasi PHP-FPM untuk Open Journal Systems (OJS) 3.4

## Pendahuluan

Setelah container PHP-FPM berhasil dibangun, langkah berikutnya adalah melakukan konfigurasi PHP agar sesuai dengan kebutuhan Open Journal Systems (OJS). Walaupun PHP dapat dijalankan menggunakan konfigurasi bawaan (default), konfigurasi tersebut umumnya ditujukan untuk lingkungan pengembangan dan belum tentu sesuai untuk server produksi.

Open Journal Systems merupakan aplikasi yang cukup kompleks. Selain melayani permintaan HTTP, OJS juga menangani proses unggah dokumen, pengolahan metadata, translasi bahasa, pembuatan cache, pengiriman email, serta berbagai proses lain yang memerlukan konfigurasi PHP yang tepat.

Pada artikel ini akan dibahas konfigurasi PHP-FPM yang digunakan pada implementasi sebelumnya sehingga menghasilkan lingkungan yang aman, stabil, dan mudah dipelihara.

---

# Arsitektur Konfigurasi

Pada implementasi ini konfigurasi PHP dipisahkan dari image Docker.

```
/opt/docker/apps/ojs
├── docker-compose.yml
├── php
│   ├── php.ini
│   └── www.conf
└── logs
```

Container hanya menggunakan file konfigurasi tersebut melalui mekanisme bind mount.

```yaml
volumes:

- ./php/php.ini:/usr/local/etc/php/conf.d/99-custom.ini:ro

- ./php/www.conf:/usr/local/etc/php-fpm.d/www.conf:ro
```

Dengan pendekatan tersebut, administrator cukup mengubah file konfigurasi pada host kemudian me-restart container tanpa perlu membangun ulang image Docker.

---

# Mengapa Konfigurasi Dipisahkan?

Pemisahan konfigurasi memberikan beberapa keuntungan.

- Mudah dipelihara.
- Mudah dibackup.
- Tidak perlu rebuild image.
- Dapat digunakan kembali pada server lain.
- Seluruh perubahan dapat didokumentasikan menggunakan Git.

Selain itu, proses upgrade PHP juga menjadi jauh lebih sederhana karena konfigurasi tetap berada pada host.

---

# Konfigurasi php.ini

File `php.ini` merupakan pusat konfigurasi PHP.

Pada implementasi ini file tersebut berada pada direktori berikut.

```text
/opt/docker/apps/ojs/php/php.ini
```

Docker kemudian melakukan mount menuju lokasi standar PHP.

```yaml
/usr/local/etc/php/conf.d/99-custom.ini
```

Penggunaan nama **99-custom.ini** memastikan konfigurasi tersebut dibaca setelah seluruh konfigurasi bawaan image.

---

# Contoh Konfigurasi php.ini

Berikut konfigurasi yang digunakan.

```ini
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; General
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

date.timezone = Asia/Jakarta
expose_php = Off
short_open_tag = Off
cgi.fix_pathinfo = 0

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; Resource Limits
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

memory_limit = 512M
max_execution_time = 300
max_input_time = 300
max_input_vars = 5000

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; File Upload
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

file_uploads = On
upload_max_filesize = 256M
post_max_size = 256M
max_file_uploads = 20

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; Error Handling
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

display_errors = Off
display_startup_errors = Off
log_errors = On
error_reporting = E_ALL

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; Session
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

session.save_handler = files
session.gc_maxlifetime = 1440
session.cookie_httponly = On
session.cookie_samesite = Lax

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; OPcache
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

opcache.enable = 1
opcache.enable_cli = 0
opcache.memory_consumption = 256
opcache.interned_strings_buffer = 16
opcache.max_accelerated_files = 20000
opcache.validate_timestamps = 1
opcache.revalidate_freq = 2
```

Konfigurasi di atas merupakan konfigurasi dasar yang telah disesuaikan untuk menjalankan Open Journal Systems pada lingkungan produksi.

Pada bagian berikutnya setiap parameter akan dibahas secara rinci.

---

# Konfigurasi Zona Waktu

```ini
date.timezone = Asia/Jakarta
```

Zona waktu menentukan bagaimana PHP menampilkan waktu, mencatat log, serta menjalankan berbagai proses internal.

Penggunaan zona waktu yang benar sangat penting karena OJS banyak menggunakan informasi waktu, misalnya:

- tanggal publikasi artikel
- waktu login pengguna
- waktu unggah dokumen
- penjadwalan penerbitan
- pencatatan log aplikasi

Apabila zona waktu tidak dikonfigurasi dengan benar, informasi tersebut dapat menjadi tidak konsisten.

---

# Menyembunyikan Versi PHP

```ini
expose_php = Off
```

Secara bawaan PHP akan mengirimkan header HTTP berikut.

```
X-Powered-By: PHP/8.x.x
```

Header tersebut memberikan informasi kepada pihak luar mengenai versi PHP yang sedang digunakan.

Walaupun terlihat sederhana, informasi tersebut dapat dimanfaatkan untuk melakukan fingerprinting terhadap server.

Dengan mengubah nilainya menjadi **Off**, header tersebut tidak lagi dikirimkan sehingga informasi mengenai versi PHP tidak mudah diketahui.

---

# Menonaktifkan Short Tag

```ini
short_open_tag = Off
```

PHP mendukung dua bentuk penulisan tag.

```php
<?php
```

dan

```php
<?
```

Penulisan menggunakan short tag sudah tidak direkomendasikan karena dapat menyebabkan konflik dengan format lain seperti XML.

Open Journal Systems menggunakan sintaks PHP standar sehingga fitur ini dapat dinonaktifkan dengan aman.

---

# Mencegah Celah PATH_INFO

```ini
cgi.fix_pathinfo = 0
```

Parameter ini merupakan salah satu konfigurasi keamanan yang paling penting ketika PHP dijalankan bersama Nginx.

Nilai bawaan memungkinkan PHP mencoba mengeksekusi file berdasarkan informasi PATH_INFO.

Konfigurasi yang tidak tepat dapat membuka peluang eksekusi file yang tidak semestinya.

Dengan mengubah nilainya menjadi **0**, PHP hanya akan menjalankan file yang secara eksplisit diminta oleh Nginx.

Konfigurasi ini juga melengkapi pembatasan yang telah diterapkan pada konfigurasi Nginx, yaitu hanya mengizinkan eksekusi terhadap file **index.php**.

---

# Resource Limits

PHP menyediakan beberapa parameter untuk membatasi penggunaan resource.

Pembatasan ini penting agar satu proses tidak menghabiskan seluruh sumber daya server.

Pada implementasi ini digunakan konfigurasi berikut.

```ini
memory_limit = 512M
max_execution_time = 300
max_input_time = 300
max_input_vars = 5000
```

Keempat parameter tersebut akan dibahas secara rinci pada bagian berikutnya.

---

## Memory Limit

```ini
memory_limit = 512M
```

Parameter ini menentukan jumlah memori maksimum yang dapat digunakan oleh satu proses PHP.

Setiap request yang diproses oleh PHP-FPM akan memiliki batas memori sesuai nilai tersebut.

Open Journal Systems melakukan berbagai proses yang cukup berat, antara lain:

- membaca metadata artikel
- menghasilkan halaman HTML
- mengelola plugin
- melakukan import dan export XML
- menghasilkan cache template
- mengolah dokumen berukuran besar

Apabila nilai `memory_limit` terlalu kecil, pengguna dapat mengalami pesan kesalahan seperti berikut.

```text
Allowed memory size exhausted
```

Nilai **512 MB** umumnya sudah lebih dari cukup untuk sebagian besar implementasi OJS.

Memberikan nilai yang terlalu besar juga tidak selalu lebih baik karena setiap proses PHP dapat menggunakan memori hingga batas tersebut.

---

## Maximum Execution Time

```ini
max_execution_time = 300
```

Parameter ini menentukan lama maksimum sebuah script PHP dijalankan.

Nilai tersebut dihitung dalam satuan detik.

Secara bawaan PHP menggunakan nilai yang relatif kecil.

Pada OJS terdapat beberapa proses yang membutuhkan waktu lebih lama, misalnya:

- upload artikel berukuran besar
- import artikel
- export XML
- indexing metadata
- instalasi plugin

Dengan nilai **300 detik**, proses tersebut memiliki waktu lima menit untuk diselesaikan.

Apabila waktu tersebut terlampaui, PHP akan menghentikan proses secara otomatis.

---

## Maximum Input Time

```ini
max_input_time = 300
```

Berbeda dengan `max_execution_time`, parameter ini membatasi waktu yang digunakan PHP untuk membaca data dari pengguna.

Contohnya:

- upload file
- membaca POST request
- membaca multipart form
- membaca request XML

Semakin besar file yang diunggah, semakin lama waktu yang diperlukan.

Nilai lima menit umumnya sudah mencukupi untuk server jurnal dengan ukuran artikel yang besar.

---

## Maximum Input Variables

```ini
max_input_vars = 5000
```

Parameter ini menentukan jumlah maksimum variabel yang diterima PHP dari request.

Nilai bawaan PHP biasanya sekitar seribu variabel.

Pada OJS terdapat halaman administrasi yang menghasilkan banyak field, terutama pada:

- pengaturan jurnal
- metadata
- konfigurasi plugin
- formulir panjang

Apabila nilai terlalu kecil, sebagian data tidak akan diproses.

Karena itu digunakan nilai **5000** agar seluruh data dapat diterima dengan baik.

---

# Konfigurasi Upload File

Salah satu fungsi utama Open Journal Systems adalah menyimpan dokumen ilmiah.

Oleh karena itu konfigurasi upload menjadi bagian yang sangat penting.

Konfigurasi yang digunakan adalah sebagai berikut.

```ini
file_uploads = On
upload_max_filesize = 256M
post_max_size = 256M
max_file_uploads = 20
```

Keempat parameter tersebut saling berhubungan.

---

## Mengaktifkan Upload File

```ini
file_uploads = On
```

Parameter ini mengaktifkan kemampuan PHP menerima file yang dikirim melalui browser.

Apabila nilainya diubah menjadi **Off**, seluruh fitur upload pada OJS tidak akan dapat digunakan.

---

## Upload Maximum File Size

```ini
upload_max_filesize = 256M
```

Parameter ini menentukan ukuran maksimum setiap file yang dapat diterima.

Contoh file yang diunggah ke OJS antara lain:

- artikel PDF
- supplementary files
- gambar
- dataset penelitian
- cover jurnal

Nilai **256 MB** sudah lebih dari cukup untuk sebagian besar jurnal ilmiah.

Apabila institusi memiliki kebutuhan khusus, nilai tersebut dapat disesuaikan.

---

## POST Maximum Size

```ini
post_max_size = 256M
```

Seluruh proses upload menggunakan metode HTTP POST.

Parameter ini menentukan ukuran maksimum request POST yang diterima PHP.

Nilai `post_max_size` sebaiknya sama atau lebih besar dibandingkan `upload_max_filesize`.

Apabila nilainya lebih kecil, proses upload akan gagal walaupun ukuran file masih berada di bawah batas `upload_max_filesize`.

---

## Maximum Uploaded Files

```ini
max_file_uploads = 20
```

Parameter ini menentukan jumlah maksimum file yang dapat diunggah dalam satu request.

Walaupun sebagian besar proses OJS hanya mengunggah satu file, terdapat beberapa fitur yang memungkinkan pengguna mengunggah beberapa lampiran sekaligus.

Nilai **20** memberikan ruang yang cukup tanpa membuka peluang penggunaan resource yang berlebihan.

---

# Konfigurasi Error Handling

Pada lingkungan produksi, informasi kesalahan tidak boleh ditampilkan kepada pengguna.

Konfigurasi yang digunakan adalah sebagai berikut.

```ini
display_errors = Off
display_startup_errors = Off
log_errors = On
error_reporting = E_ALL
```

Konfigurasi tersebut memungkinkan administrator tetap memperoleh informasi kesalahan tanpa membocorkan informasi sensitif kepada pengguna.

---

## Display Errors

```ini
display_errors = Off
```

Parameter ini menentukan apakah pesan kesalahan PHP ditampilkan pada browser.

Pada server produksi, nilai ini harus dinonaktifkan.

Apabila diaktifkan, pengguna dapat melihat:

- path direktori server
- nama file
- query database
- informasi internal aplikasi

Informasi tersebut dapat dimanfaatkan untuk melakukan fingerprinting terhadap aplikasi.

---

## Display Startup Errors

```ini
display_startup_errors = Off
```

Parameter ini mengatur apakah kesalahan pada saat proses inisialisasi PHP ditampilkan kepada pengguna.

Kesalahan startup umumnya berkaitan dengan:

- extension PHP
- konfigurasi PHP
- modul yang gagal dimuat

Sama seperti `display_errors`, parameter ini sebaiknya dimatikan pada lingkungan produksi.

---

## Log Errors

```ini
log_errors = On
```

Walaupun pesan kesalahan tidak ditampilkan kepada pengguna, administrator tetap memerlukan informasi tersebut.

Dengan mengaktifkan `log_errors`, seluruh error PHP akan dicatat ke dalam log sehingga memudahkan proses troubleshooting.

Pendekatan ini memberikan keseimbangan antara keamanan dan kemudahan administrasi.

---

## Error Reporting

```ini
error_reporting = E_ALL
```

Parameter ini menentukan jenis kesalahan yang akan dicatat.

Menggunakan `E_ALL` memastikan seluruh kesalahan penting tetap tercatat di dalam log.

Administrator dapat mengetahui adanya warning maupun deprecated feature sebelum berkembang menjadi masalah yang lebih besar.

---

# Konfigurasi Session

Open Journal Systems menggunakan session untuk mengelola autentikasi pengguna.

Konfigurasi yang digunakan adalah sebagai berikut.

```ini
session.save_handler = files
session.gc_maxlifetime = 1440
session.cookie_httponly = On
session.cookie_samesite = Lax
```

Konfigurasi tersebut cukup sederhana, namun memiliki peran yang sangat penting dalam menjaga keamanan sesi pengguna.

Pada bagian berikutnya kita akan membahas setiap parameter tersebut secara rinci, kemudian melanjutkan dengan konfigurasi **OPcache** yang berfungsi meningkatkan performa PHP secara signifikan.

---

## Session Save Handler

```ini
session.save_handler = files
```

PHP mendukung berbagai mekanisme penyimpanan session, antara lain:

- files
- redis
- memcached
- database

Pada implementasi ini digunakan penyimpanan berbasis file karena sederhana, stabil, dan tidak memerlukan layanan tambahan.

Seluruh session pengguna akan disimpan pada direktori yang telah ditentukan oleh PHP.

Pendekatan ini sudah mencukupi untuk sebagian besar implementasi Open Journal Systems dengan satu server aplikasi.

Apabila OJS dijalankan pada beberapa web server secara bersamaan (load balancing), administrator dapat mempertimbangkan penggunaan Redis atau Memcached agar session dapat dibagikan antar server.

---

## Session Lifetime

```ini
session.gc_maxlifetime = 1440
```

Parameter ini menentukan berapa lama session dianggap masih aktif.

Nilai tersebut dihitung dalam satuan detik.

```
1440 detik = 24 menit
```

Apabila pengguna tidak melakukan aktivitas selama waktu tersebut, PHP dapat menghapus session secara otomatis.

Nilai ini harus disesuaikan dengan kebutuhan organisasi.

Semakin lama session dipertahankan maka semakin nyaman bagi pengguna, namun risiko penyalahgunaan session yang ditinggalkan juga meningkat.

---

## HTTP Only Cookie

```ini
session.cookie_httponly = On
```

Cookie session merupakan salah satu target utama serangan terhadap aplikasi web.

Dengan mengaktifkan opsi ini, cookie session tidak dapat diakses menggunakan JavaScript.

Sebagai contoh, kode berikut tidak dapat membaca cookie session.

```javascript
document.cookie
```

Konfigurasi ini membantu mengurangi dampak apabila terjadi serangan Cross Site Scripting (XSS).

Walaupun tidak menghilangkan seluruh risiko XSS, pengaturan ini memberikan lapisan perlindungan tambahan terhadap pencurian session.

---

## SameSite Cookie

```ini
session.cookie_samesite = Lax
```

Parameter ini mengatur bagaimana browser mengirim cookie ketika terjadi permintaan lintas situs.

Pilihan yang tersedia antara lain:

- Strict
- Lax
- None

Pada implementasi ini digunakan nilai **Lax** karena memberikan keseimbangan antara keamanan dan kompatibilitas.

Dengan konfigurasi tersebut, sebagian besar serangan Cross Site Request Forgery (CSRF) dapat dikurangi tanpa mengganggu pengalaman pengguna.

---

# Menggunakan OPcache

Salah satu peningkatan performa terbesar pada PHP modern berasal dari OPcache.

Tanpa OPcache, setiap request yang diterima PHP akan melalui proses berikut.

```
Source Code PHP

↓

Lexer

↓

Parser

↓

Opcode

↓

Eksekusi
```

Proses tersebut akan diulang untuk setiap request.

Pada aplikasi sebesar Open Journal Systems, proses parsing ribuan file PHP secara terus-menerus tentu memerlukan waktu dan sumber daya yang tidak sedikit.

OPcache menyimpan hasil kompilasi opcode di dalam memori sehingga PHP tidak perlu melakukan parsing ulang terhadap file yang sama.

Alur kerjanya menjadi seperti berikut.

```
Request Pertama

PHP Source

↓

Compile

↓

OPcache

↓

Eksekusi

──────────────

Request Berikutnya

↓

OPcache

↓

Eksekusi
```

Pendekatan tersebut mampu mengurangi penggunaan CPU secara signifikan.

---

# Konfigurasi OPcache

Konfigurasi yang digunakan adalah sebagai berikut.

```ini
opcache.enable = 1
opcache.enable_cli = 0
opcache.memory_consumption = 256
opcache.interned_strings_buffer = 16
opcache.max_accelerated_files = 20000
opcache.validate_timestamps = 1
opcache.revalidate_freq = 2
```

Selanjutnya kita akan membahas setiap parameter tersebut.

---

## Mengaktifkan OPcache

```ini
opcache.enable = 1
```

Parameter ini mengaktifkan OPcache pada PHP-FPM.

Pada lingkungan produksi, fitur ini hampir selalu direkomendasikan karena mampu meningkatkan performa aplikasi secara signifikan.

---

## OPcache CLI

```ini
opcache.enable_cli = 0
```

Parameter ini menentukan apakah OPcache digunakan ketika menjalankan PHP melalui command line.

Contohnya.

```bash
php artisan

php index.php

php tools/importExport.php
```

Karena sebagian besar proses CLI hanya dijalankan sesekali, penggunaan OPcache pada CLI umumnya tidak memberikan keuntungan yang berarti.

Oleh karena itu parameter ini dinonaktifkan.

---

## Memory Consumption

```ini
opcache.memory_consumption = 256
```

Parameter ini menentukan jumlah memori yang disediakan untuk menyimpan opcode.

Open Journal Systems terdiri dari ribuan file PHP.

Memberikan ruang memori yang cukup akan mengurangi kemungkinan cache cepat penuh.

Apabila kapasitas terlalu kecil, PHP harus menghapus cache lama dan melakukan kompilasi ulang sehingga performa aplikasi menurun.

---

## Interned Strings Buffer

```ini
opcache.interned_strings_buffer = 16
```

PHP banyak menggunakan string yang sama secara berulang.

Misalnya:

- nama class
- namespace
- nama fungsi
- nama plugin
- path file

Dengan menggunakan interned string, PHP cukup menyimpan satu salinan string tersebut di memori.

Penggunaan memori menjadi lebih efisien sekaligus mengurangi waktu pemrosesan.

---

## Maximum Cached Files

```ini
opcache.max_accelerated_files = 20000
```

Parameter ini menentukan jumlah maksimum file PHP yang dapat disimpan pada OPcache.

Open Journal Systems terdiri dari ribuan file.

Selain source code inti, masih terdapat:

- plugin
- tema
- library pihak ketiga
- framework Laravel
- komponen PKP

Memberikan nilai yang cukup besar memastikan seluruh file penting dapat masuk ke dalam cache.

---

## Validate Timestamp

```ini
opcache.validate_timestamps = 1
```

Parameter ini menentukan apakah PHP memeriksa perubahan file.

Nilai **1** berarti PHP akan memeriksa apakah source code berubah.

Pendekatan ini sangat cocok untuk server yang masih sesekali menerima pembaruan aplikasi.

Apabila administrator yakin source code tidak pernah berubah kecuali ketika proses deployment, parameter ini dapat diubah menjadi:

```ini
opcache.validate_timestamps = 0
```

Namun setiap perubahan source code mengharuskan administrator melakukan restart PHP-FPM agar cache diperbarui.

---

## Revalidate Frequency

```ini
opcache.revalidate_freq = 2
```

Parameter ini menentukan interval pemeriksaan perubahan file.

Pada konfigurasi ini PHP akan memeriksa perubahan setiap dua detik.

Nilai tersebut memberikan keseimbangan antara performa dan kemudahan administrasi.

Pada server yang benar-benar statis, nilai ini dapat diperbesar atau bahkan menggunakan `validate_timestamps = 0`.

---

# Memverifikasi Konfigurasi PHP

Setelah selesai mengubah file `php.ini`, lakukan restart container.

```bash
docker compose restart
```

Masuk ke dalam container.

```bash
docker exec -it ojs-php bash
```

Periksa file konfigurasi yang sedang digunakan.

```bash
php --ini
```

Pastikan file berikut muncul pada daftar konfigurasi.

```text
/usr/local/etc/php/conf.d/99-custom.ini
```

Hal ini menunjukkan bahwa file konfigurasi pada host telah berhasil dimuat oleh PHP.

Pada bagian berikutnya kita akan melakukan validasi seluruh konfigurasi menggunakan `php -i`, memeriksa status OPcache, kemudian melanjutkan konfigurasi **www.conf** yang mengatur cara kerja PHP-FPM.

---

# Konfigurasi PHP-FPM (www.conf)

Selain `php.ini`, PHP-FPM menggunakan file konfigurasi lain yang bernama **www.conf**.

File ini mengatur bagaimana PHP-FPM menerima request dari Nginx, jumlah worker yang dijalankan, penggunaan socket, logging, serta berbagai parameter lain yang memengaruhi performa aplikasi.

Pada implementasi ini file tersebut berada pada direktori berikut.

```text
/opt/docker/apps/ojs/php/www.conf
```

Kemudian dipasang ke dalam container menggunakan bind mount.

```yaml
- ./php/www.conf:/usr/local/etc/php-fpm.d/www.conf:ro
```

Dengan pendekatan tersebut administrator cukup mengubah file pada host kemudian melakukan restart container.

---

# Konfigurasi www.conf

Berikut konfigurasi yang digunakan.

```ini
[www]

user = www-data
group = www-data

listen = /run/php/ojs.sock

listen.owner = www-data
listen.group = www-data
listen.mode = 0660

pm = dynamic

pm.max_children = 30
pm.start_servers = 6
pm.min_spare_servers = 4
pm.max_spare_servers = 12

pm.max_requests = 500

request_terminate_timeout = 300

clear_env = no

catch_workers_output = yes

decorate_workers_output = no

access.log = /var/log/php/access.log

slowlog = /var/log/php/slow.log

request_slowlog_timeout = 10s
```

Konfigurasi tersebut cukup sederhana, namun sudah sangat memadai untuk menjalankan Open Journal Systems pada server produksi dengan beban kerja menengah.

---

# User dan Group

```ini
user = www-data
group = www-data
```

Seluruh worker PHP-FPM dijalankan menggunakan user **www-data**.

Penggunaan user khusus ini memberikan beberapa keuntungan.

- tidak menjalankan proses sebagai root
- hak akses lebih terbatas
- lebih aman
- kompatibel dengan Nginx

Menjalankan PHP sebagai root tidak direkomendasikan karena meningkatkan risiko apabila aplikasi berhasil dieksploitasi.

---

# Socket PHP-FPM

```ini
listen = /run/php/ojs.sock
```

PHP-FPM dapat menerima request menggunakan dua metode.

Menggunakan TCP.

```text
127.0.0.1:9000
```

atau menggunakan Unix Socket.

```text
/run/php/ojs.sock
```

Pada implementasi ini digunakan Unix Socket.

Beberapa keuntungan penggunaan Unix Socket antara lain.

- lebih cepat
- latensi lebih rendah
- tidak menggunakan port jaringan
- lebih mudah dikontrol menggunakan permission file
- lebih aman

Karena Nginx dan PHP-FPM berada pada server yang sama, penggunaan Unix Socket merupakan pilihan terbaik.

---

# Permission Socket

```ini
listen.owner = www-data

listen.group = www-data

listen.mode = 0660
```

Ketiga parameter tersebut menentukan permission socket.

Socket yang dihasilkan akan terlihat seperti berikut.

```text
srw-rw---- www-data www-data ojs.sock
```

Dengan permission tersebut hanya proses yang berada pada group **www-data** yang dapat mengakses socket.

Hal ini memberikan perlindungan tambahan terhadap akses yang tidak diinginkan.

---

# Process Manager

PHP-FPM menyediakan beberapa mode pengelolaan worker.

```
static

dynamic

ondemand
```

Pada implementasi ini digunakan mode.

```ini
pm = dynamic
```

Mode **dynamic** memungkinkan PHP-FPM menyesuaikan jumlah worker sesuai beban kerja.

Ketika trafik meningkat, PHP-FPM dapat membuat worker tambahan.

Sebaliknya ketika server sedang tidak sibuk, sebagian worker akan dihentikan sehingga penggunaan memori menjadi lebih efisien.

Mode ini merupakan pilihan yang paling umum digunakan pada server produksi.

---

# Maximum Children

```ini
pm.max_children = 30
```

Parameter ini menentukan jumlah maksimum worker PHP yang dapat berjalan secara bersamaan.

Setiap worker hanya dapat melayani satu request dalam satu waktu.

Sebagai contoh.

```
30 Worker

↓

30 Request Bersamaan
```

Apabila request ke-31 datang sementara seluruh worker masih sibuk, request tersebut akan menunggu hingga salah satu worker selesai.

Nilai ini harus disesuaikan dengan kapasitas RAM server.

Semakin besar nilainya, semakin banyak memori yang digunakan.

---

# Start Servers

```ini
pm.start_servers = 6
```

Ketika PHP-FPM pertama kali dijalankan, enam worker langsung dibuat.

Pendekatan ini mengurangi waktu tunggu ketika request pertama diterima.

Apabila nilainya terlalu kecil, pengguna pertama mungkin mengalami sedikit keterlambatan karena PHP harus membuat worker baru terlebih dahulu.

---

# Minimum Spare Servers

```ini
pm.min_spare_servers = 4
```

Parameter ini menentukan jumlah minimum worker yang selalu siap melayani request.

Apabila jumlah worker yang menganggur kurang dari empat, PHP-FPM akan membuat worker baru.

Dengan demikian server selalu siap menerima lonjakan request secara tiba-tiba.

---

# Maximum Spare Servers

```ini
pm.max_spare_servers = 12
```

Sebaliknya, apabila jumlah worker yang menganggur melebihi dua belas, PHP-FPM akan menghentikan sebagian worker.

Pendekatan ini membantu menghemat penggunaan memori ketika server sedang tidak sibuk.

---

# Maximum Requests

```ini
pm.max_requests = 500
```

Setiap worker PHP akan dihentikan setelah memproses lima ratus request.

Kemudian PHP-FPM membuat worker baru sebagai penggantinya.

Mengapa perlu dilakukan?

Beberapa library PHP dapat mengalami memory leak setelah berjalan cukup lama.

Walaupun memory leak sangat kecil, akumulasi dalam jangka panjang dapat menyebabkan penggunaan memori terus meningkat.

Dengan melakukan restart worker secara berkala, penggunaan memori menjadi lebih stabil.

---

# Request Timeout

```ini
request_terminate_timeout = 300
```

Parameter ini menentukan waktu maksimum satu request dijalankan.

Apabila melebihi lima menit, worker akan dihentikan.

Hal ini melindungi server dari request yang macet ataupun proses yang berjalan tanpa batas.

---

# Clear Environment

```ini
clear_env = no
```

Secara bawaan PHP-FPM akan membersihkan seluruh environment variable.

Pada implementasi berbasis Docker sering kali diperlukan beberapa environment variable tambahan.

Karena itu parameter ini diubah menjadi.

```ini
clear_env = no
```

Dengan demikian seluruh environment yang diberikan Docker tetap tersedia bagi PHP.

---

# Worker Output

```ini
catch_workers_output = yes
```

Parameter ini mengarahkan output worker menuju log PHP-FPM.

Ketika terjadi kesalahan pada aplikasi, informasi tersebut akan lebih mudah ditemukan pada log.

Administrator tidak perlu mencari output dari masing-masing worker secara terpisah.

---

# Decorate Worker Output

```ini
decorate_workers_output = no
```

Secara bawaan PHP-FPM dapat menambahkan informasi tambahan pada setiap baris log.

Pada implementasi ini fitur tersebut dimatikan agar log menjadi lebih bersih dan mudah dibaca.

---

# Access Log

```ini
access.log = /var/log/php/access.log
```

PHP-FPM mampu mencatat seluruh request yang diterima.

Log tersebut sangat berguna ketika administrator ingin menganalisis:

- jumlah request
- waktu eksekusi
- aktivitas worker
- performa aplikasi

Log dipisahkan dari source code sehingga tetap tersedia walaupun container dibuat ulang.

---

# Slow Log

```ini
slowlog = /var/log/php/slow.log
```

PHP-FPM dapat mencatat request yang berjalan terlalu lama.

Fitur ini sangat berguna ketika aplikasi mulai terasa lambat.

Administrator dapat mengetahui script mana yang membutuhkan waktu paling lama untuk dijalankan.

---

# Slow Request Timeout

```ini
request_slowlog_timeout = 10s
```

Apabila sebuah request membutuhkan waktu lebih dari sepuluh detik, PHP-FPM akan mencatat informasi tersebut ke dalam slow log.

Dengan demikian proses optimasi dapat dilakukan berdasarkan data nyata, bukan sekadar perkiraan.

Pada bagian berikutnya kita akan melakukan validasi konfigurasi `www.conf`, menguji socket PHP-FPM, serta melakukan tuning performa sebelum container digunakan pada lingkungan produksi.

---

# Memverifikasi Konfigurasi PHP

Setelah seluruh konfigurasi `php.ini` dan `www.conf` selesai dibuat, langkah berikutnya adalah memastikan PHP benar-benar menggunakan konfigurasi tersebut.

Restart terlebih dahulu container.

```bash
cd /opt/docker/apps/ojs

docker compose restart
```

Tunggu hingga container kembali berjalan.

```bash
docker ps
```

Pastikan status container menjadi.

```text
Up (healthy)
```

---

# Masuk ke Dalam Container

Masuk ke shell container.

```bash
docker exec -it ojs-php bash
```

Prompt akan berubah menjadi.

```text
root@xxxxxxxx:/var/www/html#
```

Seluruh pemeriksaan berikut dilakukan dari dalam container.

---

# Memastikan php.ini Terbaca

Jalankan perintah berikut.

```bash
php --ini
```

Contoh hasil.

```text
Configuration File (php.ini) Path:
/usr/local/etc/php

Loaded Configuration File:
(none)

Scan for additional .ini files in:
/usr/local/etc/php/conf.d

Additional .ini files parsed:
/usr/local/etc/php/conf.d/99-custom.ini
```

Pastikan file **99-custom.ini** muncul pada daftar tersebut.

Apabila file tidak muncul, kemungkinan volume Docker belum dipasang dengan benar.

---

# Memastikan Konfigurasi Aktif

Selanjutnya lakukan pengecekan beberapa parameter.

Misalnya.

```bash
php -i | grep memory_limit
```

Hasilnya.

```text
memory_limit => 512M
```

Kemudian.

```bash
php -i | grep upload_max_filesize
```

Contoh.

```text
upload_max_filesize => 256M
```

Lanjutkan.

```bash
php -i | grep post_max_size
```

Hasil.

```text
post_max_size => 256M
```

Periksa pula timezone.

```bash
php -i | grep "Default timezone"
```

Contoh.

```text
Default timezone => Asia/Jakarta
```

Dengan langkah tersebut administrator dapat memastikan bahwa PHP benar-benar menggunakan konfigurasi yang telah dibuat.

---

# Memverifikasi PHP-FPM

PHP-FPM menyediakan fasilitas validasi konfigurasi.

Jalankan.

```bash
php-fpm -tt
```

Apabila konfigurasi benar akan muncul.

```text
configuration file test is successful
```

Perintah ini sangat berguna setelah melakukan perubahan pada file `www.conf`.

---

# Memverifikasi Socket

Pastikan socket PHP berhasil dibuat.

```bash
ls -lah /run/php
```

Contoh.

```text
srw-rw---- 1 www-data www-data ojs.sock
```

Socket tersebut nantinya digunakan Nginx untuk meneruskan request PHP.

Apabila socket tidak muncul, Nginx tidak akan dapat memproses file PHP.

---

# Memverifikasi Permission Socket

Lakukan pemeriksaan lebih lanjut.

```bash
stat /run/php/ojs.sock
```

Contoh.

```text
Owner : www-data

Group : www-data

Mode  : 0660
```

Permission tersebut memastikan hanya proses yang memiliki hak akses yang sesuai yang dapat menggunakan socket PHP.

---

# Memastikan Extension PHP

Open Journal Systems membutuhkan beberapa extension PHP.

Periksa extension yang aktif.

```bash
php -m
```

Pastikan minimal extension berikut tersedia.

```text
ctype
curl
dom
fileinfo
filter
gd
gettext
iconv
intl
json
libxml
mbstring
mysqli
openssl
pdo_mysql
session
simplexml
tokenizer
xml
xmlreader
xmlwriter
xsl
zip
zlib
Zend OPcache
```

Apabila salah satu extension tidak tersedia, beberapa fitur OJS mungkin tidak berjalan sebagaimana mestinya.

---

# Memverifikasi OPcache

Periksa apakah OPcache aktif.

```bash
php -i | grep opcache.enable
```

Contoh.

```text
opcache.enable => On
```

Periksa pula kapasitas memorinya.

```bash
php -i | grep opcache.memory_consumption
```

Contoh.

```text
opcache.memory_consumption => 256
```

Hal ini memastikan proses caching opcode telah aktif.

---

# Monitoring PHP-FPM

Administrator sebaiknya melakukan monitoring secara berkala.

Melihat proses PHP.

```bash
ps aux | grep php-fpm
```

Melihat penggunaan memori.

```bash
free -h
```

Melihat penggunaan CPU.

```bash
top
```

atau.

```bash
htop
```

Monitoring sederhana tersebut sangat membantu ketika melakukan analisis performa server.

---

# Melihat Log PHP

Seluruh log PHP disimpan di luar container.

Misalnya.

```text
/opt/docker/apps/ojs/logs
```

Lihat isi direktori.

```bash
ls -lah /opt/docker/apps/ojs/logs
```

Apabila terjadi kesalahan, administrator cukup membaca log tersebut tanpa harus masuk ke dalam container.

---

# Restart PHP-FPM

Setiap perubahan konfigurasi memerlukan restart container.

```bash
docker compose restart
```

Apabila melakukan perubahan pada image.

```bash
docker compose down

docker compose up -d
```

Dengan pendekatan ini seluruh perubahan dapat diterapkan tanpa memengaruhi source code aplikasi.

---

# Troubleshooting

Berikut beberapa masalah yang paling sering dijumpai.

## php.ini Tidak Terbaca

Periksa.

```bash
php --ini
```

Pastikan file **99-custom.ini** muncul.

Kemudian periksa volume Docker.

```bash
docker inspect ojs-php
```

---

## PHP-FPM Tidak Mau Start

Lakukan validasi.

```bash
php-fpm -tt
```

Periksa pula log.

```bash
docker logs ojs-php
```

---

## Socket Tidak Muncul

Periksa.

```bash
ls -lah /run/php
```

Pastikan file berikut tersedia.

```text
ojs.sock
```

---

## Permission Denied

Periksa owner.

```bash
stat /run/php/ojs.sock
```

Pastikan owner dan group sesuai.

```text
www-data
```

---

## OPcache Tidak Aktif

Periksa.

```bash
php -i | grep opcache.enable
```

Kemudian restart container.

```bash
docker compose restart
```

---

# Best Practices

Beberapa praktik terbaik yang diterapkan pada implementasi ini antara lain.

- Gunakan PHP versi yang masih didukung.
- Pisahkan konfigurasi dari image Docker.
- Gunakan Unix Socket dibandingkan TCP.
- Jalankan worker menggunakan user **www-data**.
- Aktifkan OPcache.
- Nonaktifkan `display_errors`.
- Gunakan `log_errors`.
- Batasi penggunaan resource.
- Gunakan filesystem read-only pada container.
- Lakukan backup konfigurasi secara berkala.
- Simpan source code di luar container.
- Pisahkan direktori `ojsdata` dari source code.
- Dokumentasikan setiap perubahan konfigurasi.

Pendekatan tersebut menghasilkan lingkungan PHP-FPM yang stabil, aman, dan mudah dipelihara.

---

# Kesimpulan

Konfigurasi PHP-FPM merupakan salah satu komponen terpenting dalam implementasi Open Journal Systems. Konfigurasi yang tepat tidak hanya meningkatkan performa aplikasi, tetapi juga memperkuat keamanan serta mempermudah proses administrasi server.

Pada artikel ini telah dibahas penyusunan file `php.ini` dan `www.conf`, konfigurasi OPcache, pengaturan worker PHP-FPM, penggunaan Unix Socket, proses validasi konfigurasi, hingga berbagai praktik terbaik yang dapat diterapkan pada lingkungan produksi.

Dengan konfigurasi tersebut, PHP-FPM siap diintegrasikan dengan Nginx untuk menjalankan Open Journal Systems secara optimal.

---

## Ringkasan

Pada artikel ini telah dibahas:

- Struktur konfigurasi PHP-FPM
- Penyusunan `php.ini`
- Penyusunan `www.conf`
- Pengaturan upload file
- Pengaturan session
- Optimasi OPcache
- Pengelolaan worker PHP-FPM
- Penggunaan Unix Socket
- Validasi konfigurasi
- Monitoring
- Troubleshooting
- Best Practices

Pada artikel berikutnya kita akan membahas **Konfigurasi Nginx untuk Open Journal Systems (OJS) 3.4**, meliputi pembuatan virtual host, konfigurasi FastCGI, hardening Nginx, reverse proxy, SSL, hingga optimasi performa agar OJS siap digunakan pada lingkungan produksi.

---
{{< saweria >}}