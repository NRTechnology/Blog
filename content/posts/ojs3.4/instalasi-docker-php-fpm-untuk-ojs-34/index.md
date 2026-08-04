---
title: "Instalasi Docker PHP-FPM untuk Open Journal Systems (OJS) 3.4"
date: 2026-08-02
draft: false
description: "Panduan lengkap membangun lingkungan Docker PHP-FPM untuk Open Journal Systems (OJS) 3.4 menggunakan Nginx, MariaDB, dan Docker dengan pendekatan production-ready."
tags:
  - Docker
  - PHP-FPM
  - OJS
  - Open Journal Systems
  - Linux
  - Nginx
  - MariaDB
categories:
  - Docker
  - Linux
  - OJS

series:
  - "Membangun Open Journal Systems (OJS) 3.4"
weight: 2

author: "NR Technology"
cover:
  image: "../assets/ojs-cover.png"
  alt: "Open Journal Systems (OJS) 3.4"
  caption: "Seri Membangun Open Journal Systems (OJS) 3.4"
---

# Instalasi Docker PHP-FPM untuk Open Journal Systems (OJS) 3.4

## Pendahuluan

Open Journal Systems (OJS) merupakan salah satu aplikasi manajemen jurnal ilmiah yang paling banyak digunakan oleh perguruan tinggi, lembaga penelitian, maupun institusi pemerintah. Sebagai aplikasi berbasis PHP, OJS dapat dijalankan menggunakan berbagai pendekatan, mulai dari instalasi PHP langsung pada sistem operasi hingga menggunakan teknologi container seperti Docker.

Pada implementasi ini, PHP tidak diinstal secara langsung pada host, melainkan dijalankan di dalam sebuah container Docker menggunakan PHP-FPM. Pendekatan ini memberikan fleksibilitas yang lebih baik dalam proses deployment, maintenance, upgrade, maupun migrasi server.

Arsitektur yang digunakan memisahkan setiap komponen layanan sehingga masing-masing memiliki tanggung jawab yang jelas.

- Reverse Proxy bertugas menerima koneksi dari Internet.
- Nginx bertugas melayani static content dan meneruskan request PHP.
- PHP-FPM dijalankan di dalam Docker.
- MariaDB berjalan langsung pada host.
- Seluruh data aplikasi berada di luar container.

Dengan pendekatan tersebut, container hanya berfungsi sebagai runtime PHP sehingga proses upgrade maupun penggantian image dapat dilakukan tanpa memengaruhi source code aplikasi.

---

# Mengapa Menggunakan Docker PHP-FPM?

Docker memberikan banyak keuntungan dibandingkan menjalankan PHP langsung pada sistem operasi.

Beberapa keuntungan tersebut antara lain:

- Runtime PHP terisolasi dari host.
- Mudah melakukan upgrade versi PHP.
- Setiap aplikasi dapat menggunakan versi PHP yang berbeda.
- Konfigurasi menjadi konsisten antar server.
- Mudah dipindahkan ke server lain.
- Backup menjadi lebih sederhana.
- Risiko konflik dependency jauh lebih kecil.

Selain itu, pendekatan ini memungkinkan satu server menjalankan beberapa aplikasi PHP secara bersamaan tanpa saling mengganggu.

Sebagai contoh:

```

Ubuntu Server

├── Laravel
│ └── PHP 8.4 Container
│
├── OJS
│ └── PHP 8.3 Container
│
├── WordPress
│ └── PHP 8.2 Container
│
└── Nextcloud
└── PHP 8.4 Container

```

Masing-masing aplikasi memiliki runtime sendiri sehingga proses upgrade tidak memengaruhi aplikasi lainnya.

---

# Arsitektur yang Digunakan

Implementasi pada artikel ini menggunakan arsitektur sebagai berikut.

```

Internet
│
HTTPS 443
│
Reverse Proxy (Nginx)
│
HTTP 8080
│
Nginx Application Server
│
Unix Socket
│
Docker PHP-FPM
│
MariaDB

```

Perjalanan sebuah request dapat dijelaskan sebagai berikut.

1. Pengguna membuka alamat jurnal menggunakan HTTPS.
2. Reverse Proxy menerima koneksi SSL.
3. Reverse Proxy meneruskan request menuju Nginx Application Server.
4. Nginx memeriksa apakah request merupakan file statis atau request PHP.
5. Apabila merupakan request PHP, Nginx meneruskan request melalui Unix Socket menuju PHP-FPM yang berjalan di dalam Docker.
6. PHP-FPM menjalankan source code OJS.
7. OJS mengambil data dari MariaDB.
8. Hasil diproses kembali oleh Nginx dan dikirim kepada pengguna.

Pendekatan tersebut menghasilkan pemisahan tanggung jawab yang jelas sehingga proses troubleshooting menjadi jauh lebih mudah.

---

# Persiapan Server

Sebelum membangun container PHP-FPM, pastikan server telah memiliki komponen berikut.

- Ubuntu Server 24.04 LTS
- Docker Engine
- Docker Compose
- Nginx
- MariaDB
- Source code Open Journal Systems (OJS) 3.4

Pastikan pula Docker telah berjalan dengan baik.

```bash
docker version
```

Periksa Docker Compose.

```bash
docker compose version
```

Pastikan service Docker aktif.

```bash
systemctl status docker
```

Apabila Docker belum aktif, jalankan perintah berikut.

```bash
systemctl enable docker
systemctl start docker
```

---

# Struktur Direktori

Untuk mempermudah proses backup dan pemeliharaan, source code aplikasi dipisahkan dari konfigurasi Docker.

Source code aplikasi berada pada direktori berikut.

```text
/var/apps/ojs
├── htdocs
├── data
│   └── ojsdata
├── backup
└── logs
```

Sedangkan konfigurasi Docker ditempatkan pada lokasi yang berbeda.

```text
/opt/docker/apps/ojs
├── docker-compose.yml
├── php
└── logs
```

Pemisahan ini membuat konfigurasi container tidak bercampur dengan source code aplikasi.

Selain itu, proses migrasi server juga menjadi jauh lebih sederhana karena administrator cukup menyalin direktori aplikasi beserta konfigurasi Docker.

---

# Membuat Struktur Direktori

Buat terlebih dahulu struktur direktori yang akan digunakan.

```bash
mkdir -p /opt/docker/apps/ojs/php
mkdir -p /opt/docker/apps/ojs/logs

mkdir -p /var/apps/ojs
mkdir -p /var/apps/ojs/data/ojsdata
mkdir -p /var/apps/ojs/backup
```

Setelah direktori dibuat, struktur yang dihasilkan akan terlihat seperti berikut.

```text
/opt/docker/apps/ojs
├── logs
└── php

/var/apps/ojs
├── backup
├── data
│   └── ojsdata
└── htdocs
```

Direktori `htdocs` nantinya berisi seluruh source code Open Journal Systems, sedangkan direktori `ojsdata` digunakan untuk menyimpan seluruh file yang diunggah melalui aplikasi, seperti artikel PDF, file reviewer, lampiran, serta berbagai dokumen pendukung lainnya.

Pada bagian berikutnya kita akan mulai membangun proyek Docker dan menyusun konfigurasi `docker-compose.yml` yang akan digunakan sebagai fondasi container PHP-FPM.

---

# Membangun Docker Project

Seluruh konfigurasi Docker untuk OJS akan ditempatkan pada direktori berikut.

```text
/opt/docker/apps/ojs
```

Masuk ke direktori tersebut.

```bash
cd /opt/docker/apps/ojs
```

Kemudian buat file `docker-compose.yml`.

```bash
touch docker-compose.yml
```

Selanjutnya buat direktori untuk konfigurasi PHP.

```bash
mkdir -p php
```

Buat dua file konfigurasi yang nantinya digunakan oleh PHP-FPM.

```bash
touch php/php.ini
touch php/www.conf
```

Struktur direktori sekarang menjadi seperti berikut.

```text
/opt/docker/apps/ojs
├── docker-compose.yml
├── logs
└── php
    ├── php.ini
    └── www.conf
```

Seluruh konfigurasi PHP akan ditempatkan di dalam direktori `php`, sedangkan file `docker-compose.yml` menjadi pusat konfigurasi seluruh container.

---

# Menyusun Docker Compose

Docker Compose merupakan komponen utama yang mengatur bagaimana container dijalankan.

Pada implementasi ini digunakan konfigurasi sebagai berikut.

```yaml
name: ojs

services:
  php:
    image: local/php:8.3

    container_name: ojs-php

    restart: unless-stopped

    init: true

    working_dir: /var/www/html

    read_only: true

    security_opt:
      - no-new-privileges=true

    stop_grace_period: 30s

    environment:
      TZ: Asia/Jakarta

    ulimits:
      nofile:
        soft: 65535
        hard: 65535

    pids_limit: 256

    tmpfs:
      - /tmp:size=256m,mode=1777
      - /run:size=32m,mode=755

    volumes:
      - /run/php:/run/php
      - /var/apps/ojs/htdocs:/var/www/html:ro
      - ./php/php.ini:/usr/local/etc/php/conf.d/99-custom.ini:ro
      - ./php/www.conf:/usr/local/etc/php-fpm.d/www.conf:ro
      - /var/apps/ojs/data/ojsdata:/var/ojsdata:rw
      - /var/apps/ojs/htdocs/public:/var/www/html/public:rw
      - /var/apps/ojs/htdocs/cache:/var/www/html/cache:rw
      - ./logs:/var/log/php:rw

    healthcheck:
      test:
        - CMD
        - test
        - -S
        - /run/php/ojs.sock
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s

    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "5"

networks:
  default:
    name: ojs-network
    driver: bridge
```

Konfigurasi tersebut merupakan fondasi utama container PHP-FPM yang akan digunakan untuk menjalankan Open Journal Systems 3.4.

Meskipun terlihat sederhana, setiap parameter memiliki fungsi yang berbeda dan saling melengkapi untuk menghasilkan container yang aman, ringan, dan mudah dipelihara.

Pada bagian berikutnya kita akan membahas setiap parameter tersebut secara rinci.

---

# Penjelasan Parameter Docker Compose

## Name

```yaml
name: ojs
```

Parameter ini menentukan nama project Docker Compose.

Dengan konfigurasi tersebut Docker akan membuat resource menggunakan awalan **ojs**, misalnya:

```text
ojs-network
```

atau

```text
ojs-php
```

Penggunaan nama project akan mempermudah administrasi apabila satu server menjalankan banyak aplikasi Docker.

---

## Service

```yaml
services:
```

Bagian ini mendefinisikan seluruh container yang akan dijalankan.

Pada implementasi ini hanya terdapat satu service.

```yaml
php
```

Container tersebut hanya bertugas menjalankan PHP-FPM.

Seluruh fungsi web server tetap dijalankan oleh Nginx yang berada di host.

Pendekatan ini membuat proses debugging menjadi lebih sederhana karena setiap komponen memiliki fungsi yang jelas.

---

## Image

```yaml
image: local/php:8.3
```

Image yang digunakan merupakan image PHP yang telah dibangun sebelumnya.

Menggunakan image sendiri memiliki beberapa keuntungan dibandingkan langsung menggunakan image resmi.

- Extension PHP sudah lengkap.
- Konfigurasi dapat distandarkan.
- Proses deployment lebih cepat.
- Seluruh server menggunakan image yang sama.

Administrator cukup memperbarui image ketika ingin melakukan upgrade PHP tanpa harus mengubah konfigurasi aplikasi.

---

## Container Name

```yaml
container_name: ojs-php
```

Docker sebenarnya mampu membuat nama container secara otomatis.

Namun penggunaan nama yang konsisten akan mempermudah proses administrasi.

Sebagai contoh.

```bash
docker logs ojs-php

docker exec -it ojs-php bash

docker restart ojs-php
```

Nama container yang mudah diingat akan mempercepat proses troubleshooting.

---

## Restart Policy

```yaml
restart: unless-stopped
```

Parameter ini menginstruksikan Docker agar secara otomatis menjalankan kembali container apabila terjadi reboot server.

Dengan demikian administrator tidak perlu menjalankan container secara manual setelah server hidup kembali.

Pilihan `unless-stopped` menjadi pilihan yang paling umum digunakan pada lingkungan produksi karena container tetap berjalan setelah restart sistem, tetapi tetap dapat dihentikan secara manual apabila diperlukan.

---

## Init

```yaml
init: true
```

Opsi ini mengaktifkan proses init di dalam container.

Fungsinya adalah menangani proses zombie yang mungkin muncul selama PHP-FPM berjalan.

Walaupun terlihat sederhana, penggunaan `init` membantu menjaga stabilitas container dalam jangka panjang, terutama pada aplikasi yang berjalan terus-menerus seperti OJS.

---

## Working Directory

```yaml
working_dir: /var/www/html
```

Direktori ini menjadi lokasi kerja utama PHP-FPM.

Seluruh source code Open Journal Systems akan dipasang pada lokasi tersebut menggunakan mekanisme bind mount.

Dengan demikian seluruh proses PHP selalu berjalan pada direktori aplikasi yang benar.

---

## Time Zone

```yaml
environment:
  TZ: Asia/Jakarta
```

Penentuan zona waktu sangat penting agar seluruh log aplikasi, PHP, dan proses internal memiliki waktu yang konsisten dengan lokasi server.

Pada implementasi ini digunakan zona waktu Asia/Jakarta sehingga seluruh log dan proses penjadwalan mengikuti waktu Indonesia Barat.

---

# Hardening Container

Salah satu tujuan utama penggunaan Docker pada implementasi ini bukan hanya untuk mempermudah deployment, tetapi juga meningkatkan keamanan aplikasi.

Container PHP-FPM yang digunakan menerapkan beberapa teknik hardening sehingga apabila terjadi kompromi pada aplikasi, dampaknya dapat diminimalkan.

Beberapa hardening yang diterapkan antara lain:

- Read-only filesystem
- No New Privileges
- Resource Limitation
- Temporary Filesystem (tmpfs)
- Unix Socket
- Isolasi Volume
- Health Check

Masing-masing akan dijelaskan pada bagian berikut.

---

## Read Only Filesystem

```yaml
read_only: true
```

Parameter ini merupakan salah satu fitur keamanan yang paling penting.

Secara default, filesystem container dapat ditulis oleh seluruh proses yang berjalan di dalam container.

Apabila terjadi eksploitasi terhadap aplikasi PHP, penyerang dapat dengan mudah:

- membuat file PHP baru
- memodifikasi source code
- memasang web shell
- mengubah konfigurasi aplikasi

Dengan mengaktifkan mode **read-only**, seluruh filesystem container menjadi hanya dapat dibaca.

Artinya source code aplikasi tidak dapat diubah walaupun aplikasi berhasil dieksploitasi.

Hanya direktori tertentu yang tetap diberikan hak tulis melalui mekanisme bind mount.

Pendekatan ini mengikuti prinsip **Least Privilege**, yaitu memberikan hak akses seminimal mungkin kepada aplikasi.

---

## Security Option

```yaml
security_opt:
  - no-new-privileges=true
```

Parameter ini menginstruksikan kernel Linux agar proses di dalam container tidak dapat memperoleh hak akses tambahan.

Sebagai contoh, apabila terdapat binary yang memiliki bit **setuid**, proses tersebut tetap tidak dapat meningkatkan hak aksesnya menjadi root.

Walaupun fitur ini terlihat sederhana, penerapannya mampu mengurangi risiko privilege escalation apabila terjadi kompromi terhadap aplikasi.

Pada lingkungan produksi, opsi ini sangat direkomendasikan untuk seluruh container yang tidak memerlukan hak akses istimewa.

---

## Stop Grace Period

```yaml
stop_grace_period: 30s
```

Ketika container dihentikan, Docker tidak langsung mematikan proses PHP.

Docker terlebih dahulu mengirimkan sinyal penghentian dan memberikan waktu selama tiga puluh detik agar seluruh request yang sedang diproses dapat diselesaikan.

Hal ini mengurangi risiko:

- request terputus
- data tidak tersimpan
- proses upload gagal

Apabila setelah tiga puluh detik proses masih berjalan, Docker akan menghentikan container secara paksa.

---

## Resource Limitation

Container tidak boleh menggunakan seluruh resource server.

Oleh karena itu diterapkan pembatasan resource.

```yaml
pids_limit: 256
```

Parameter tersebut membatasi jumlah proses maksimum yang dapat dibuat oleh container.

Apabila terjadi bug ataupun serangan yang menyebabkan proses PHP membuat child process secara berlebihan, container tidak akan menghabiskan seluruh PID pada host.

---

Selain itu digunakan pula pengaturan file descriptor.

```yaml
ulimits:
  nofile:
    soft: 65535
    hard: 65535
```

Parameter ini menentukan jumlah maksimum file yang dapat dibuka oleh PHP.

Nilai yang cukup besar diperlukan karena OJS dapat melayani banyak koneksi secara bersamaan, terutama ketika melakukan upload file ataupun mengakses banyak artikel secara bersamaan.

---

# Temporary Filesystem

```yaml
tmpfs:
  - /tmp:size=256m,mode=1777
  - /run:size=32m,mode=755
```

Direktori **/tmp** dan **/run** tidak memerlukan penyimpanan permanen.

Oleh karena itu kedua direktori tersebut ditempatkan pada **tmpfs**, yaitu filesystem yang berada di memori.

Keuntungan penggunaan tmpfs antara lain:

- akses lebih cepat
- mengurangi aktivitas disk
- data otomatis hilang ketika container dihentikan
- mengurangi kemungkinan file sementara tertinggal

Pendekatan ini sangat sesuai untuk PHP-FPM karena sebagian besar file pada direktori tersebut memang hanya bersifat sementara.

---

# Volume Mapping

Seluruh data penting tidak disimpan di dalam container.

Sebaliknya, seluruh data ditempatkan pada host kemudian dipasang menggunakan bind mount.

Pendekatan ini memiliki beberapa keuntungan.

- mudah dibackup
- mudah dipindahkan
- container dapat diganti kapan saja
- source code tetap berada di host

Diagram sederhananya sebagai berikut.

```text
Host

├── htdocs
├── cache
├── public
├── ojsdata
└── logs

        │

        ▼

Docker PHP-FPM
```

Dengan demikian container dapat dianggap sebagai runtime yang bersifat sementara.

---

## Mount Source Code

```yaml
- /var/apps/ojs/htdocs:/var/www/html:ro
```

Source code dipasang menggunakan mode **read only**.

Keuntungan pendekatan ini:

- aplikasi tidak dapat memodifikasi source code
- risiko web shell lebih kecil
- source code tetap identik dengan repository

Apabila administrator ingin melakukan upgrade OJS, cukup mengganti source code pada host tanpa perlu membangun ulang container.

---

## Mount Cache

```yaml
- /var/apps/ojs/htdocs/cache:/var/www/html/cache:rw
```

Direktori cache merupakan satu-satunya bagian source code yang memang memerlukan hak tulis.

OJS menggunakan direktori ini untuk:

- template compile
- cache konfigurasi
- cache database
- cache aplikasi

Karena itu direktori ini diberikan akses **read-write**.

---

## Mount Public

```yaml
- /var/apps/ojs/htdocs/public:/var/www/html/public:rw
```

Direktori public digunakan untuk menyimpan file yang memang boleh diakses pengguna.

Contohnya:

- logo jurnal
- favicon
- banner
- gambar homepage

Karena administrator dapat mengubah aset tersebut melalui antarmuka OJS, direktori ini juga memerlukan hak tulis.

---

## Mount OJS Data

```yaml
- /var/apps/ojs/data/ojsdata:/var/ojsdata:rw
```

Direktori ini merupakan lokasi penyimpanan seluruh dokumen jurnal.

Beberapa contoh file yang disimpan antara lain:

- artikel PDF
- file submission
- revisi reviewer
- supplementary files
- cover issue
- lampiran

Direktori ini merupakan salah satu komponen terpenting dalam proses backup.

Tanpa direktori **ojsdata**, seluruh dokumen jurnal akan hilang walaupun database masih tersedia.

---

## Mount PHP Socket

```yaml
- /run/php:/run/php
```

PHP-FPM berkomunikasi dengan Nginx menggunakan Unix Socket.

Pendekatan ini dipilih karena memiliki beberapa keuntungan dibandingkan TCP.

- lebih cepat
- latensi lebih rendah
- tidak memerlukan port tambahan
- lebih mudah diamankan menggunakan permission file

Setelah container berjalan, socket yang dihasilkan akan terlihat seperti berikut.

```text
/run/php/ojs.sock
```

Nginx cukup meneruskan request menuju socket tersebut.

---

## Mount Log

```yaml
- ./logs:/var/log/php:rw
```

Log PHP dipisahkan dari source code aplikasi.

Dengan demikian administrator dapat melakukan monitoring maupun troubleshooting tanpa harus masuk ke dalam container.

Selain itu log tetap tersedia walaupun container dihapus dan dibuat kembali.

---

# Health Check

Docker menyediakan mekanisme untuk memantau kesehatan container.

Konfigurasi yang digunakan adalah sebagai berikut.

```yaml
healthcheck:
  test:
    - CMD
    - test
    - -S
    - /run/php/ojs.sock
```

Docker akan memeriksa apakah socket PHP-FPM berhasil dibuat.

Apabila socket tidak tersedia, container akan dianggap tidak sehat (**unhealthy**).

Pendekatan ini jauh lebih baik dibandingkan hanya memeriksa apakah proses PHP masih berjalan.

Karena pada praktiknya proses PHP dapat tetap hidup walaupun socket tidak berhasil dibuat.

Pada bagian berikutnya kita akan menjalankan container, melakukan verifikasi PHP-FPM, serta memastikan seluruh konfigurasi bekerja dengan baik sebelum diintegrasikan dengan Nginx.

---

# Menjalankan Container

Setelah seluruh konfigurasi selesai dibuat, langkah berikutnya adalah membangun dan menjalankan container PHP-FPM.

Pastikan berada pada direktori project Docker.

```bash
cd /opt/docker/apps/ojs
```

Jalankan container.

```bash
docker compose up -d
```

Apabila seluruh konfigurasi benar, Docker akan membuat network kemudian menjalankan container.

Contoh output.

```text
[+] Running 2/2
 ✔ Network ojs-network  Created
 ✔ Container ojs-php    Started
```

Selanjutnya pastikan container benar-benar berjalan.

```bash
docker ps
```

Contoh hasilnya.

```text
CONTAINER ID   IMAGE            STATUS
2a4f0d7ab12d   local/php:8.3    Up 10 seconds (healthy)
```

Perhatikan kolom **STATUS**.

Container sebaiknya berada pada status:

```text
healthy
```

Apabila status masih **starting**, tunggu beberapa saat hingga proses health check selesai.

---

# Melihat Informasi Container

Docker menyediakan berbagai perintah untuk melihat kondisi container.

Melihat seluruh container.

```bash
docker ps -a
```

Melihat detail container.

```bash
docker inspect ojs-php
```

Melihat penggunaan resource.

```bash
docker stats ojs-php
```

Melihat log.

```bash
docker logs ojs-php
```

Apabila ingin mengikuti log secara real time.

```bash
docker logs -f ojs-php
```

---

# Masuk ke Dalam Container

Kadang administrator perlu melakukan pemeriksaan langsung.

Masuk ke shell container.

```bash
docker exec -it ojs-php bash
```

Prompt akan berubah menjadi.

```text
root@xxxxxxxx:/var/www/html#
```

Selanjutnya seluruh pemeriksaan PHP dilakukan dari dalam container.

---

# Memeriksa Versi PHP

Pastikan versi PHP sesuai dengan yang digunakan.

```bash
php -v
```

Contoh.

```text
PHP 8.3.x
```

Selain versi PHP, periksa pula Zend OPcache.

Output biasanya menampilkan.

```text
Zend OPcache
```

Hal tersebut menunjukkan bahwa OPcache telah aktif.

---

# Memeriksa Extension PHP

OJS membutuhkan sejumlah extension PHP.

Periksa extension yang tersedia.

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
```

Extension tersebut digunakan oleh berbagai fitur OJS seperti:

- upload file
- XML import
- export metadata
- translasi bahasa
- kompresi ZIP
- pengolahan gambar
- database

Apabila salah satu extension tidak tersedia, OJS kemungkinan tidak dapat berjalan secara normal.

---

# Memeriksa Konfigurasi PHP

Seluruh konfigurasi aktif dapat dilihat menggunakan.

```bash
php --ini
```

Perintah tersebut menampilkan lokasi file konfigurasi.

Contoh.

```text
Loaded Configuration File

/usr/local/etc/php/php.ini
```

Selanjutnya periksa nilai konfigurasi tertentu.

Misalnya.

```bash
php -i | grep memory_limit
```

atau

```bash
php -i | grep upload_max_filesize
```

Hal ini memastikan file `php.ini` yang digunakan benar-benar telah dimuat.

---

# Memeriksa Konfigurasi PHP-FPM

PHP-FPM menyediakan fasilitas validasi konfigurasi.

```bash
php-fpm -tt
```

Apabila seluruh konfigurasi benar, hasil akhirnya akan menampilkan.

```text
configuration file test is successful
```

Perintah ini sangat berguna setelah melakukan perubahan pada file `www.conf`.

---

# Memeriksa Socket PHP

Pastikan socket berhasil dibuat.

```bash
ls -lah /run/php
```

Contoh.

```text
srw-rw---- 1 www-data www-data ojs.sock
```

Socket inilah yang digunakan Nginx untuk berkomunikasi dengan PHP-FPM.

Apabila socket tidak muncul, Nginx tidak akan dapat memproses request PHP.

---

# Memeriksa Permission Socket

Pastikan owner socket sesuai.

```bash
stat /run/php/ojs.sock
```

Output akan memperlihatkan owner serta permission file.

Idealnya socket dimiliki oleh user yang sama dengan proses PHP-FPM.

Sebagai contoh.

```text
Owner: www-data
Group: www-data
```

Dengan demikian Nginx dapat mengakses socket tanpa harus membuka permission secara berlebihan.

---

# Memeriksa Direktori Mount

Pastikan seluruh volume berhasil dipasang.

```bash
mount | grep var/www/html
```

atau

```bash
df -h
```

Kemudian periksa direktori aplikasi.

```bash
ls -lah /var/www/html
```

Pastikan source code OJS telah muncul.

Selanjutnya periksa direktori cache.

```bash
ls -lah /var/www/html/cache
```

Pastikan direktori tersebut dapat ditulis.

---

# Memeriksa Direktori ojsdata

Pastikan direktori upload tersedia.

```bash
ls -lah /var/ojsdata
```

Apabila direktori belum ada, buat terlebih dahulu pada host.

```bash
mkdir -p /var/apps/ojs/data/ojsdata
```

Kemudian sesuaikan hak akses.

```bash
chown -R www-data:www-data /var/apps/ojs/data/ojsdata
```

Direktori ini nantinya akan menyimpan seluruh artikel dan dokumen jurnal.

---

# Restart Container

Apabila terdapat perubahan konfigurasi Docker Compose.

Restart container.

```bash
docker compose restart
```

Apabila mengubah image.

```bash
docker compose down

docker compose up -d
```

Perintah tersebut akan membuat ulang container menggunakan konfigurasi terbaru.

---

# Troubleshooting

Berikut beberapa masalah yang paling sering ditemui.

## Container Tidak Mau Berjalan

Periksa log.

```bash
docker logs ojs-php
```

Kemudian periksa konfigurasi.

```bash
docker compose config
```

---

## Socket Tidak Muncul

Periksa konfigurasi PHP-FPM.

```bash
php-fpm -tt
```

Periksa pula direktori socket.

```bash
ls -lah /run/php
```

---

## Permission Denied

Periksa owner direktori.

```bash
ls -lah
```

atau

```bash
namei -om /var/apps/ojs/htdocs/cache
```

Pastikan user `www-data` memiliki hak akses yang sesuai.

---

## Health Check Gagal

Periksa apakah socket berhasil dibuat.

```bash
test -S /run/php/ojs.sock
```

Apabila menghasilkan exit code selain nol, berarti socket belum tersedia.

---

# Best Practices

Beberapa praktik terbaik yang diterapkan pada implementasi ini antara lain:

- Gunakan image PHP yang telah distandarkan.
- Jalankan satu aplikasi dalam satu container.
- Pisahkan source code dan data aplikasi.
- Gunakan bind mount hanya pada direktori yang diperlukan.
- Terapkan filesystem read-only.
- Gunakan Unix Socket dibanding TCP.
- Aktifkan health check.
- Batasi resource container.
- Pisahkan log dari source code.
- Backup direktori `ojsdata` secara berkala.

Pendekatan tersebut menghasilkan container yang ringan, aman, mudah dipelihara, serta mudah dipindahkan ke server lain.

---

# Kesimpulan

Pada artikel ini kita telah membangun lingkungan Docker PHP-FPM yang siap digunakan untuk menjalankan Open Journal Systems (OJS) 3.4.

Container yang dibangun tidak hanya berfungsi sebagai runtime PHP, tetapi juga menerapkan berbagai praktik terbaik seperti penggunaan filesystem read-only, pembatasan resource, health check, Unix Socket, serta pemisahan data aplikasi dari container.

Dengan pendekatan ini, proses backup, upgrade PHP, migrasi server, maupun pemeliharaan aplikasi menjadi jauh lebih mudah dibandingkan instalasi PHP secara langsung pada sistem operasi.

Pada artikel berikutnya kita akan membahas konfigurasi **PHP-FPM**, meliputi penyusunan file **php.ini** dan **www.conf**, optimasi performa, serta konfigurasi yang direkomendasikan agar Open Journal Systems dapat berjalan secara optimal pada lingkungan produksi.

---
{{< saweria >}}