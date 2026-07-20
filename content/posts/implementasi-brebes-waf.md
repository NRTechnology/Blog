+++
date = '2026-07-19T15:00:00+07:00'
draft = false
title = 'Implementasi Brebes WAF dengan Nginx dan ModSecurity'
categories = ['Cyber Security']
tags = [
    'Brebes WAF',
    'Web Application Firewall',
    'WAF',
    'Nginx',
    'ModSecurity',
    'Reverse Proxy',
    'Web Security',
    'Ubuntu Server'
]
+++

Pada artikel sebelumnya kita telah membahas konsep dan tujuan pembangunan Brebes WAF sebagai lapisan perlindungan terpusat untuk aplikasi web.

Pada artikel kali ini, kita akan mulai melakukan implementasi Brebes WAF menggunakan Nginx sebagai reverse proxy dan ModSecurity sebagai Web Application Firewall.

Secara umum, tahapan yang akan dilakukan adalah:

1. Menyiapkan Ubuntu Server
2. Menginstal Nginx
3. Menginstal ModSecurity
4. Mengaktifkan ModSecurity pada Nginx
5. Melakukan clone repository Brebes WAF
6. Mengaktifkan konfigurasi dan rule Brebes WAF
7. Menghubungkan WAF dengan origin web server
8. Melakukan pengujian reverse proxy
9. Melakukan pengujian deteksi serangan
10. Memeriksa log ModSecurity

Arsitektur yang akan dibangun adalah:

```text
Internet / Client
        │
        ▼
┌─────────────────────┐
│     Brebes WAF      │
│                     │
│  Nginx              │
│      │              │
│      ▼              │
│  ModSecurity        │
│      │              │
│  Brebes WAF Rules   │
└─────────┬───────────┘
          │
          │ Reverse Proxy
          ▼
┌─────────────────────┐
│   Origin Server     │
│                     │
│ Nginx / Apache      │
│ Aplikasi Web        │
└─────────────────────┘
```

## Persiapan Server

Pada implementasi Brebes WAF ini, kita akan menggunakan Ubuntu Server sebagai sistem operasi. Server Brebes WAF akan ditempatkan di depan origin web server dan berfungsi sebagai reverse proxy sekaligus Web Application Firewall.

Sebelum memulai proses instalasi, pastikan server memiliki koneksi internet dan dapat berkomunikasi dengan origin web server yang nantinya akan dilindungi.

Arsitektur jaringan yang digunakan kurang lebih sebagai berikut:

```text
Internet / Client
        │
        ▼
    Firewall
        │
        ▼
┌─────────────────────┐
│     Brebes WAF      │
│    Ubuntu Server    │
│                     │
│ Nginx + ModSecurity │
└─────────┬───────────┘
          │
          │ HTTP / HTTPS
          ▼
┌─────────────────────┐
│   Origin Server     │
│                     │
│ Nginx / Apache      │
│ Aplikasi Web        │
└─────────────────────┘
```

### System Requirements

Untuk kebutuhan pengujian atau lingkungan lab, Brebes WAF dapat dijalankan pada Virtual Machine dengan spesifikasi minimal:

- 2 vCPU
- 2 GB RAM
- 20 GB storage
- 1 Network Interface
- Ubuntu Server 24.04 LTS

Spesifikasi tersebut ditujukan untuk kebutuhan pengujian atau lingkungan lab. Untuk lingkungan production, kebutuhan resource perlu disesuaikan dengan jumlah aplikasi yang dilindungi, jumlah traffic, jumlah request, serta rule ModSecurity yang digunakan.

### Memperbarui Sistem

Untuk memastikan proses instalasi berjalan dengan lancar, sebelum melakukan instalasi paket yang dibutuhkan, sangat disarankan untuk memperbarui sistem terlebih dahulu.

```bash
sudo apt update
sudo apt upgrade -y
```

Setelah proses pembaruan selesai, lakukan restart server apabila diperlukan:

```bash
sudo reboot
```

### Mengatur Timezone

Timezone atau zona waktu perlu dikonfigurasi agar waktu pada sistem dan log keamanan yang dihasilkan sesuai dengan zona waktu yang digunakan.

Pada implementasi ini, kita akan menggunakan zona waktu `Asia/Jakarta`.

```bash
sudo timedatectl set-timezone Asia/Jakarta
```

Untuk memastikan konfigurasi timezone telah diterapkan, jalankan:

```bash
timedatectl
```

Pengaturan waktu yang tepat sangat penting karena Brebes WAF akan menghasilkan berbagai log keamanan. Timestamp yang konsisten akan mempermudah proses monitoring dan analisis apabila terjadi insiden keamanan.

### Instalasi Package Pendukung

Selanjutnya, instal beberapa package pendukung yang akan dibutuhkan selama proses implementasi Brebes WAF:

```bash
sudo apt install -y \
    curl \
    wget \
    git \
    unzip \
    ca-certificates \
    gnupg \
    lsb-release
```

Package `git` nantinya akan digunakan untuk melakukan clone repository Brebes WAF, sedangkan package lainnya digunakan untuk mendukung proses instalasi dan konfigurasi.

## Instalasi Nginx

Setelah proses persiapan server selesai, langkah berikutnya adalah melakukan instalasi Nginx. Pada arsitektur Brebes WAF, Nginx akan berfungsi sebagai reverse proxy yang menerima request HTTP dan HTTPS dari client sebelum request diperiksa oleh ModSecurity dan diteruskan menuju origin server.

Untuk melakukan instalasi Nginx, jalankan perintah berikut:

```bash
sudo apt install -y nginx
```

Setelah proses instalasi selesai, aktifkan Nginx agar berjalan secara otomatis ketika server melakukan boot:

```bash
sudo systemctl enable nginx
```

Kemudian jalankan service Nginx:

```bash
sudo systemctl start nginx
```

Periksa status service Nginx dengan perintah:

```bash
sudo systemctl status nginx
```

Apabila Nginx berjalan dengan normal, akan terlihat status:

```text
Active: active (running)
```

Selanjutnya, periksa versi Nginx yang telah terinstal:

```bash
nginx -v
```

Untuk memastikan konfigurasi Nginx tidak memiliki kesalahan syntax, jalankan:

```bash
sudo nginx -t
```

Apabila konfigurasi Nginx valid, akan muncul informasi kurang lebih seperti berikut:

```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

### Pengujian Nginx

Untuk memastikan Nginx dapat menerima request HTTP, lakukan pengujian dari server menggunakan:

```bash
curl -I http://localhost
```

Apabila Nginx berjalan dengan normal, server akan memberikan HTTP response seperti:

```text
HTTP/1.1 200 OK
Server: nginx
```

Pengujian juga dapat dilakukan melalui web browser dengan mengakses alamat IP server Brebes WAF:

```text
http://IP-SERVER-WAF
```

Apabila halaman default **Welcome to nginx!** muncul, maka instalasi Nginx telah berhasil.

### Memeriksa Port Nginx

Secara default, Nginx akan mendengarkan koneksi HTTP pada port `80`. Untuk memastikannya, jalankan:

```bash
sudo ss -lntp | grep nginx
```

Hasilnya kurang lebih akan menunjukkan:

```text
LISTEN 0 511 0.0.0.0:80 0.0.0.0:*
```

Hal tersebut menunjukkan bahwa Nginx telah siap menerima koneksi HTTP.

Pada tahap ini, Nginx masih menggunakan konfigurasi default dan belum berfungsi sebagai Web Application Firewall. Pada langkah berikutnya, kita akan melakukan instalasi ModSecurity dan mengintegrasikannya dengan Nginx agar setiap HTTP request yang melewati Brebes WAF dapat diperiksa berdasarkan rule keamanan yang digunakan.


## Instalasi ModSecurity

Setelah Nginx berhasil diinstal dan berjalan dengan normal, langkah berikutnya adalah melakukan instalasi ModSecurity.

ModSecurity merupakan Web Application Firewall (WAF) engine yang digunakan untuk melakukan inspeksi terhadap HTTP request dan response berdasarkan rule keamanan yang telah dikonfigurasi.

Pada implementasi Brebes WAF ini, kita menggunakan ModSecurity v3 yang tersedia melalui repository Ubuntu serta modul connector yang menghubungkan ModSecurity dengan Nginx.

Untuk melakukan instalasi, jalankan:

```bash
sudo apt install -y \
    libmodsecurity3t64 \
    libmodsecurity-dev \
    libnginx-mod-http-modsecurity \
    modsecurity-crs
```

Setelah proses instalasi selesai, periksa package ModSecurity yang telah terinstal:

```bash
dpkg -l | grep -i modsecurity
```

Hasilnya kurang lebih akan terlihat seperti:

```text
ii  libmodsecurity-dev
ii  libmodsecurity3t64
ii  libnginx-mod-http-modsecurity
ii  modsecurity-crs
```

Untuk melihat versi package secara lengkap, gunakan:

```bash
dpkg -l | grep -i modsecurity
```

Pada implementasi ini, komponen yang digunakan adalah:

```text
ModSecurity Engine       : 3.0.12
Nginx ModSecurity Module : 1.0.3
OWASP Core Rule Set      : 3.3.5
```

Versi package dapat berbeda tergantung versi Ubuntu dan repository yang digunakan.

### Memastikan Module ModSecurity Tersedia

Periksa keberadaan dynamic module ModSecurity untuk Nginx:

```bash
find /usr/lib/nginx/modules -iname '*modsecurity*'
```

Kemudian periksa apakah konfigurasi untuk memuat module ModSecurity tersedia:

```bash
grep -R "load_module.*modsecurity" /etc/nginx/
```

Pada instalasi package Ubuntu, dynamic module Nginx umumnya dikelola melalui konfigurasi module yang disediakan oleh package.

Selanjutnya, lakukan pengujian konfigurasi Nginx:

```bash
sudo nginx -t
```

Apabila tidak terdapat kesalahan konfigurasi, restart Nginx:

```bash
sudo systemctl restart nginx
```

Kemudian periksa status Nginx:

```bash
sudo systemctl status nginx
```

Sampai tahap ini, ModSecurity engine dan connector untuk Nginx telah terinstal. Namun, ModSecurity belum tentu melakukan inspeksi terhadap seluruh virtual host secara otomatis.

Pada tahap berikutnya, kita akan melakukan konfigurasi ModSecurity, mengaktifkan rule yang digunakan oleh Brebes WAF, dan memastikan bahwa request yang melewati Nginx benar-benar diperiksa oleh ModSecurity.

## Memastikan ModSecurity Terintegrasi dengan Nginx

Setelah proses instalasi selesai, langkah berikutnya adalah memastikan module ModSecurity telah tersedia dan dimuat oleh Nginx.

Pada instalasi menggunakan package Ubuntu, module ModSecurity untuk Nginx akan dipasang sebagai dynamic module.

Periksa module yang aktif dengan menjalankan:

```bash
ls -lah /etc/nginx/modules-enabled/
```

Apabila instalasi berhasil, akan ditemukan konfigurasi module ModSecurity seperti berikut:

```text
50-mod-http-modsecurity.conf -> /usr/share/nginx/modules-available/mod-http-modsecurity.conf
```

File tersebut merupakan symbolic link menuju konfigurasi dynamic module ModSecurity yang disediakan oleh package Nginx.

Untuk melihat konfigurasi tersebut, jalankan:

```bash
cat /usr/share/nginx/modules-available/mod-http-modsecurity.conf
```

Konfigurasi tersebut akan memuat module ModSecurity ke dalam Nginx.

Selanjutnya, lakukan pengujian konfigurasi Nginx:

```bash
sudo nginx -t
```

Apabila tidak ditemukan kesalahan, konfigurasi akan menghasilkan:

```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Sampai tahap ini, dynamic module ModSecurity telah berhasil dimuat oleh Nginx.

Perlu diperhatikan bahwa module ModSecurity yang berhasil dimuat belum berarti setiap request yang masuk ke Nginx otomatis diperiksa oleh ModSecurity. ModSecurity masih harus diaktifkan pada konfigurasi Nginx dan diberikan file konfigurasi serta rule yang akan digunakan.

### Memeriksa OWASP Core Rule Set

Pada proses instalasi sebelumnya, kita juga telah menginstal package:

```text
modsecurity-crs
```

Package tersebut menyediakan OWASP ModSecurity Core Rule Set atau OWASP CRS.

OWASP CRS merupakan kumpulan rule generik yang digunakan untuk membantu mendeteksi berbagai pola serangan terhadap aplikasi web, seperti SQL Injection, Cross-Site Scripting, Local File Inclusion, Remote File Inclusion, dan beberapa jenis serangan aplikasi web lainnya.

Untuk melihat konfigurasi CRS, jalankan:

```bash
ls -lah /etc/modsecurity/crs/
```

Pada instalasi ini, konfigurasi CRS berada pada:

```text
/etc/modsecurity/crs/
├── crs-setup.conf
├── REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf
└── RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf
```

Sedangkan kumpulan rule OWASP CRS berada pada:

```text
/usr/share/modsecurity-crs/rules/
```

Untuk melihat rule yang tersedia:

```bash
ls -lah /usr/share/modsecurity-crs/rules/
```

Di dalam direktori tersebut terdapat berbagai rule untuk mendeteksi pola serangan, antara lain:

```text
REQUEST-930-APPLICATION-ATTACK-LFI.conf
REQUEST-931-APPLICATION-ATTACK-RFI.conf
REQUEST-932-APPLICATION-ATTACK-RCE.conf
REQUEST-933-APPLICATION-ATTACK-PHP.conf
REQUEST-941-APPLICATION-ATTACK-XSS.conf
REQUEST-942-APPLICATION-ATTACK-SQLI.conf
```

Rule tersebut digunakan sebagai salah satu lapisan deteksi terhadap request yang melewati Web Application Firewall.

### Memahami Komponen ModSecurity

Secara sederhana, komponen yang digunakan pada implementasi ini dapat digambarkan sebagai berikut:

```text
HTTP/HTTPS Request
        │
        ▼
┌──────────────────────┐
│        Nginx         │
│       1.24.0         │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ ModSecurity Connector│
│        1.0.3         │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  ModSecurity Engine  │
│        3.0.12        │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│      WAF Rules       │
│                      │
│   OWASP Core Rule    │
│        Set 3.3.5     │
│          +           │
│ Brebes WAF Rules     │
└──────────────────────┘
```

Nginx bertugas menerima request HTTP dan HTTPS serta berfungsi sebagai reverse proxy.

ModSecurity connector menghubungkan Nginx dengan ModSecurity Engine, sedangkan ModSecurity Engine melakukan inspeksi terhadap request berdasarkan rule yang telah dikonfigurasi.

OWASP Core Rule Set menyediakan rule keamanan generik, sedangkan Brebes WAF dapat menambahkan konfigurasi dan rule yang disesuaikan dengan kebutuhan implementasi.

Pada tahap berikutnya, kita akan melakukan clone repository Brebes WAF dan mulai menerapkan konfigurasi yang digunakan oleh Brebes WAF.

## Clone Repository Brebes WAF

Setelah Nginx dan ModSecurity berhasil diinstal, langkah berikutnya adalah melakukan clone repository Brebes WAF.

Repository Brebes WAF berisi konfigurasi dan rule tambahan yang akan digunakan dalam implementasi Web Application Firewall.

Pada implementasi ini, repository akan ditempatkan pada direktori `/opt`.

Masuk ke direktori `/opt`:

```bash
cd /opt
```

Kemudian lakukan clone repository Brebes WAF:

```bash
sudo git clone https://github.com/NRTechnology/Brebes-WAF.git
```

Setelah proses clone selesai, masuk ke direktori Brebes WAF:

```bash
cd /opt/Brebes-WAF
```

Periksa isi repository:

```bash
ls -lah
```

Untuk melihat struktur direktori secara lebih lengkap, kita dapat menggunakan perintah:

```bash
find /opt/Brebes-WAF -maxdepth 2 -type f | sort
```

Repository Brebes WAF yang digunakan pada tutorial ini dapat diakses melalui:

https://github.com/NRTechnology/Brebes-WAF

Dengan menggunakan Git, konfigurasi Brebes WAF dapat dikelola menggunakan version control. Apabila terdapat pembaruan konfigurasi atau rule pada repository, administrator dapat melihat perubahan yang dilakukan sebelum menerapkannya pada server production.

Untuk melihat informasi repository yang digunakan, jalankan:

```bash
git remote -v
```

Hasilnya akan menunjukkan repository sumber:

```text
origin  https://github.com/NRTechnology/Brebes-WAF.git (fetch)
origin  https://github.com/NRTechnology/Brebes-WAF.git (push)
```

Kita juga dapat melihat branch yang sedang digunakan dengan menjalankan:

```bash
git branch --show-current
```

Sedangkan commit terakhir dapat diperiksa menggunakan:

```bash
git log -1 --oneline
```

Sampai tahap ini, repository Brebes WAF telah berhasil di-clone ke dalam server.

Pada tahap berikutnya, kita akan mempelajari struktur direktori Brebes WAF dan menentukan file konfigurasi serta rule yang perlu diintegrasikan dengan ModSecurity dan Nginx.

## Struktur Direktori Brebes WAF

Setelah repository Brebes WAF berhasil di-clone, langkah berikutnya adalah memahami struktur direktori yang terdapat di dalam repository.

Untuk melihat isi direktori utama Brebes WAF, jalankan:

```bash
cd /opt/Brebes-WAF
ls -lah
```

Struktur utama repository Brebes WAF adalah sebagai berikut:

```text
/opt/Brebes-WAF
├── docs
├── nginx
├── rules
├── samples
└── scripts
```

Masing-masing direktori memiliki fungsi yang berbeda dalam implementasi Brebes WAF.

### Direktori docs

Direktori:

```text
/opt/Brebes-WAF/docs
```

digunakan untuk menyimpan dokumentasi yang berkaitan dengan Brebes WAF.

Pada versi repository yang digunakan dalam tutorial ini, terdapat file:

```text
docs/
├── BREBES-WAF_Alfa_Rule_ID.xlsx
└── README.md
```

File `BREBES-WAF_Alfa_Rule_ID.xlsx` digunakan sebagai dokumentasi rule ID Brebes WAF, sedangkan `README.md` berisi dokumentasi yang berkaitan dengan implementasi Brebes WAF.

### Direktori nginx

Direktori:

```text
/opt/Brebes-WAF/nginx
```

berisi konfigurasi yang digunakan untuk mengintegrasikan Nginx dengan ModSecurity dan Brebes WAF.

File yang tersedia antara lain:

```text
nginx/
├── modsecurity.conf
├── modsecurity_includes.conf
└── nginx.conf
```

File `modsecurity.conf` berisi konfigurasi utama ModSecurity yang digunakan oleh Brebes WAF.

File `modsecurity_includes.conf` digunakan untuk mengatur file konfigurasi dan rule yang akan dimuat oleh ModSecurity.

Sedangkan `nginx.conf` berisi konfigurasi Nginx yang digunakan sebagai bagian dari implementasi Brebes WAF.

### Direktori rules

Direktori:

```text
/opt/Brebes-WAF/rules
```

digunakan untuk menyimpan rule keamanan yang digunakan atau dikembangkan untuk Brebes WAF.

Pada direktori tersebut juga terdapat file:

```text
9999000-debug-response.conf.off
```

Ekstensi `.off` menunjukkan bahwa rule tersebut tidak diaktifkan secara default. File debug seperti ini sebaiknya hanya digunakan ketika dibutuhkan untuk proses pengujian atau troubleshooting dan tidak diaktifkan tanpa memahami dampaknya pada lingkungan production.

### Direktori samples

Direktori:

```text
/opt/Brebes-WAF/samples
```

digunakan untuk menyimpan contoh konfigurasi atau file pendukung yang dapat digunakan sebagai referensi dalam implementasi Brebes WAF.

Isi direktori ini akan kita bahas lebih lanjut sesuai dengan kebutuhan konfigurasi yang akan diterapkan.

### Direktori scripts

Direktori:

```text
/opt/Brebes-WAF/scripts
```

berisi script yang digunakan untuk membantu proses deployment dan operasional Brebes WAF.

Pada versi repository yang digunakan dalam tutorial ini terdapat:

```text
scripts/
├── check-last-detection.sh
└── deploy.sh
```

Script `deploy.sh` digunakan untuk membantu proses deployment konfigurasi Brebes WAF.

Sedangkan `check-last-detection.sh` digunakan sebagai utilitas untuk membantu melakukan pemeriksaan terhadap deteksi yang dihasilkan oleh WAF.

Sebelum menjalankan script dari repository, sangat disarankan untuk memeriksa terlebih dahulu isi script:

```bash
cat /opt/Brebes-WAF/scripts/deploy.sh
```

dan:

```bash
cat /opt/Brebes-WAF/scripts/check-last-detection.sh
```

Langkah ini penting agar administrator mengetahui perubahan apa saja yang akan dilakukan oleh script terhadap konfigurasi sistem.

Secara sederhana, struktur Brebes WAF dapat digambarkan sebagai berikut:

```text
Brebes-WAF
    │
    ├── docs
    │     └── Dokumentasi
    │
    ├── nginx
    │     └── Konfigurasi Nginx dan ModSecurity
    │
    ├── rules
    │     └── Rule keamanan Brebes WAF
    │
    ├── samples
    │     └── Contoh konfigurasi
    │
    └── scripts
          └── Deployment dan utilitas
```

Dengan struktur tersebut, konfigurasi Nginx, konfigurasi ModSecurity, rule keamanan, dokumentasi, dan script operasional dapat dikelola secara terpisah di dalam satu repository.

Pada tahap berikutnya, kita akan melakukan deployment konfigurasi Brebes WAF dan mengintegrasikannya dengan Nginx serta ModSecurity yang telah terinstal pada server.

## Deployment Brebes WAF

Setelah memahami struktur direktori Brebes WAF, langkah berikutnya adalah melakukan deployment rule Brebes WAF agar dapat digunakan oleh ModSecurity.

Brebes WAF menyediakan script deployment yang berada pada:

```text
/opt/Brebes-WAF/scripts/deploy.sh
```

Sebelum menjalankan script deployment, kita dapat melihat terlebih dahulu isi script:

```bash
cat /opt/Brebes-WAF/scripts/deploy.sh
```

Script tersebut akan melakukan beberapa proses secara otomatis, yaitu:

1. Memastikan script dijalankan menggunakan user `root`.
2. Memastikan direktori `/opt/Brebes-WAF/rules` tersedia.
3. Menghitung jumlah file rule dengan ekstensi `.conf`.
4. Membuat file include ModSecurity secara otomatis.
5. Memuat OWASP Core Rule Set.
6. Memuat seluruh rule Brebes WAF.
7. Membuat backup konfigurasi sebelumnya.
8. Melakukan pengujian konfigurasi Nginx menggunakan `nginx -t`.
9. Melakukan reload Nginx apabila konfigurasi valid.
10. Mengembalikan konfigurasi sebelumnya apabila pengujian konfigurasi gagal.

### File Include yang Dihasilkan

Script deployment akan menghasilkan file:

```text
/etc/nginx/modsecurity_includes.conf
```

File tersebut digunakan sebagai file utama yang mengatur rule yang akan dimuat oleh ModSecurity.

Secara sederhana, rule akan dimuat dengan urutan:

```text
/etc/nginx/modsecurity.conf
        │
        ▼
OWASP Core Rule Set
/etc/nginx/modsecurity/crs-load.conf
        │
        ▼
Brebes WAF Rules
/opt/Brebes-WAF/rules/**/*.conf
```

Urutan pemuatan rule penting karena konfigurasi utama ModSecurity dimuat terlebih dahulu, kemudian OWASP CRS, dan dilanjutkan dengan rule tambahan Brebes WAF.

### Menyiapkan Konfigurasi ModSecurity

Sebelum menjalankan deployment, pastikan file konfigurasi yang dibutuhkan telah tersedia pada Nginx.

Periksa keberadaan file berikut:

```bash
ls -lah /etc/nginx/modsecurity.conf
ls -lah /etc/nginx/modsecurity/crs-load.conf
```

File `/etc/nginx/modsecurity.conf` merupakan konfigurasi utama ModSecurity.

Sedangkan:

```text
/etc/nginx/modsecurity/crs-load.conf
```

digunakan untuk memuat konfigurasi OWASP Core Rule Set yang digunakan oleh Brebes WAF.

Kedua file tersebut harus tersedia karena file hasil deployment akan memuat:

```text
include modsecurity.conf
include /etc/nginx/modsecurity/crs-load.conf
```

Apabila salah satu file tersebut tidak tersedia atau memiliki konfigurasi yang tidak valid, proses pengujian `nginx -t` dapat mengalami kegagalan.

### Memeriksa Rule Brebes WAF

Sebelum melakukan deployment, kita dapat melihat jumlah rule Brebes WAF yang akan dimuat:

```bash
find /opt/Brebes-WAF/rules -type f -name "*.conf" | wc -l
```

Untuk melihat daftar rule:

```bash
find /opt/Brebes-WAF/rules -type f -name "*.conf" | sort
```

Script deployment hanya akan memuat file dengan ekstensi:

```text
*.conf
```

Dengan mekanisme tersebut, file yang memiliki ekstensi `.conf.off`, seperti:

```text
9999000-debug-response.conf.off
```

tidak akan dimuat secara otomatis.

Hal ini memungkinkan rule tertentu disimpan di dalam repository tanpa harus langsung diaktifkan pada lingkungan production.

### Menjalankan Deployment

Pastikan script memiliki permission untuk dieksekusi:

```bash
chmod +x /opt/Brebes-WAF/scripts/deploy.sh
```

Kemudian jalankan:

```bash
sudo /opt/Brebes-WAF/scripts/deploy.sh
```

Script akan menghitung seluruh rule Brebes WAF dan menghasilkan file:

```text
/etc/nginx/modsecurity_includes.conf
```

Setelah file berhasil dibuat, script akan menjalankan:

```bash
nginx -t
```

Apabila konfigurasi Nginx valid, script akan melakukan reload Nginx secara otomatis:

```bash
systemctl reload nginx
```

Output deployment kurang lebih akan terlihat seperti:

```text
=========================================================
 BREBES-WAF Deployment
=========================================================
[1/4] Generating ModSecurity Include File...
[2/4] Testing Nginx Configuration...
[3/4] Reloading Nginx...
[4/4] Deployment Successful
```

Setelah deployment selesai, periksa file yang dihasilkan:

```bash
cat /etc/nginx/modsecurity_includes.conf
```

File tersebut akan berisi konfigurasi utama ModSecurity, OWASP CRS, dan seluruh rule Brebes WAF yang ditemukan pada direktori:

```text
/opt/Brebes-WAF/rules
```

Dengan mekanisme deployment ini, administrator tidak perlu menambahkan setiap file rule secara manual ke dalam konfigurasi ModSecurity. Setiap file `.conf` yang berada di dalam direktori `rules` akan ditemukan, diurutkan berdasarkan nama file, kemudian ditambahkan ke dalam file include secara otomatis.

Pada tahap berikutnya, kita akan mengaktifkan ModSecurity pada konfigurasi virtual host Nginx dan menghubungkan Brebes WAF dengan origin web server menggunakan mekanisme reverse proxy.

### Verifikasi Deployment Brebes WAF

Setelah proses deployment selesai, periksa file konfigurasi yang dihasilkan:

```bash
cat /etc/nginx/modsecurity_includes.conf
```

File tersebut akan memuat konfigurasi dengan urutan sebagai berikut:

```text
ModSecurity Configuration
        │
        ▼
OWASP Core Rule Set
        │
        ▼
Brebes WAF Custom Rules
        │
        ├── Upload Protection
        └── Webshell Protection
```

Pada implementasi ini, script deployment berhasil menemukan dan memuat sebanyak 23 file rule Brebes WAF.

Jumlah rule yang dimuat dapat diperiksa menggunakan:

```bash
grep "^Include /opt/Brebes-WAF/rules" /etc/nginx/modsecurity_includes.conf | wc -l
```

Untuk melihat seluruh rule Brebes WAF yang aktif:

```bash
grep "^Include /opt/Brebes-WAF/rules" /etc/nginx/modsecurity_includes.conf
```

Rule Brebes WAF akan dimuat setelah OWASP Core Rule Set. Dengan demikian, request yang diperiksa oleh ModSecurity akan mendapatkan perlindungan dari OWASP CRS serta rule tambahan yang dikembangkan pada Brebes WAF.

Setelah memastikan seluruh rule berhasil dimuat, lakukan pengujian konfigurasi Nginx:

```bash
sudo nginx -t
```

Apabila konfigurasi valid, reload Nginx:

```bash
sudo systemctl reload nginx
```

Sampai tahap ini, konfigurasi ModSecurity, OWASP Core Rule Set, dan rule Brebes WAF telah siap digunakan.

Langkah berikutnya adalah mengaktifkan ModSecurity pada virtual host Nginx dan mengonfigurasi Nginx sebagai reverse proxy menuju origin web server.

## Konfigurasi Nginx sebagai Reverse Proxy

Setelah ModSecurity, OWASP Core Rule Set, dan rule Brebes WAF berhasil dikonfigurasi, langkah berikutnya adalah mengonfigurasi Nginx sebagai reverse proxy.

Pada implementasi Brebes WAF, ModSecurity diaktifkan secara global pada konfigurasi utama Nginx yang berada di:

```text
/etc/nginx/nginx.conf
```

Konfigurasi ModSecurity ditempatkan di dalam context `http {}`:

```nginx
http {

    ...

    ##
    # ModSecurity
    ##

    modsecurity on;
    modsecurity_rules_file /etc/nginx/modsecurity_includes.conf;

    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

Directive berikut digunakan untuk mengaktifkan ModSecurity:

```nginx
modsecurity on;
```

Sedangkan directive:

```nginx
modsecurity_rules_file /etc/nginx/modsecurity_includes.conf;
```

digunakan untuk menentukan file konfigurasi dan rule yang akan dimuat oleh ModSecurity.

File `/etc/nginx/modsecurity_includes.conf` sebelumnya telah dibuat oleh script deployment Brebes WAF dan berisi konfigurasi utama ModSecurity, OWASP Core Rule Set, serta custom rule Brebes WAF.

Secara sederhana, konfigurasi tersebut membentuk alur:

```text
/etc/nginx/nginx.conf
        │
        ├── modsecurity on
        │
        └── modsecurity_rules_file
                    │
                    ▼
      /etc/nginx/modsecurity_includes.conf
                    │
          ┌─────────┴─────────┐
          │                   │
          ▼                   ▼
      OWASP CRS       Brebes WAF Rules
```

Karena ModSecurity diaktifkan pada context `http {}`, virtual host yang dimuat melalui:

```nginx
include /etc/nginx/sites-enabled/*;
```

akan berada di bawah konfigurasi HTTP tersebut. Oleh karena itu, kita tidak perlu menambahkan directive `modsecurity on` dan `modsecurity_rules_file` secara berulang pada setiap virtual host, kecuali apabila diperlukan konfigurasi khusus untuk virtual host tertentu.

### Membuat Virtual Host Reverse Proxy

Sebagai contoh, kita akan melindungi aplikasi dengan domain:

```text
app.example.com
```

Sedangkan aplikasi berada pada origin server:

```text
192.168.10.20
```

Buat konfigurasi virtual host:

```bash
sudo nano /etc/nginx/sites-available/app.example.com.conf
```

Masukkan konfigurasi berikut:

```nginx
server {
    listen 80;
    server_name app.example.com;

    access_log /var/log/nginx/app.example.com.access.log reverser;
    error_log  /var/log/nginx/app.example.com.error.log;

    location / {
        proxy_pass http://192.168.10.20;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_http_version 1.1;
    }
}
```

Pada konfigurasi tersebut, `proxy_pass` menentukan alamat origin web server yang akan menerima request setelah melewati Brebes WAF.

Directive:

```nginx
proxy_set_header Host $host;
```

digunakan untuk meneruskan hostname yang diakses oleh client menuju origin server.

Sedangkan:

```nginx
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

digunakan untuk meneruskan informasi alamat IP client menuju origin server.

Directive:

```nginx
proxy_set_header X-Forwarded-Proto $scheme;
```

memberikan informasi kepada origin server mengenai protokol yang digunakan oleh client untuk mengakses Brebes WAF.

### Mengaktifkan Virtual Host

Setelah konfigurasi selesai dibuat, aktifkan virtual host dengan membuat symbolic link:

```bash
sudo ln -s /etc/nginx/sites-available/app.example.com.conf \
    /etc/nginx/sites-enabled/app.example.com.conf
```

Lakukan pengujian konfigurasi Nginx:

```bash
sudo nginx -t
```

Apabila konfigurasi valid, reload Nginx:

```bash
sudo systemctl reload nginx
```

Dengan konfigurasi tersebut, request yang masuk menuju `app.example.com` akan diterima oleh Nginx pada Brebes WAF dan diperiksa oleh ModSecurity menggunakan OWASP CRS serta custom rule Brebes WAF.

Request yang diperbolehkan akan diteruskan menuju origin server, sedangkan request yang memenuhi kondisi blocking pada rule dapat dihentikan oleh ModSecurity sebelum mencapai aplikasi.

## Menghubungkan Brebes WAF dengan Origin Server

Setelah Nginx dikonfigurasi sebagai reverse proxy, langkah berikutnya adalah memastikan Brebes WAF dapat berkomunikasi dengan origin server yang menjalankan aplikasi web.

Dalam arsitektur ini, origin server merupakan server tempat aplikasi web sebenarnya berjalan. Origin server dapat menggunakan Nginx, Apache, IIS, atau web server lainnya.

Alur komunikasi yang digunakan adalah:

```text
Internet / Client
        │
        ▼
┌─────────────────────┐
│     Brebes WAF      │
│                     │
│ Nginx + ModSecurity │
│ OWASP CRS           │
│ Brebes WAF Rules    │
└──────────┬──────────┘
           │
           │ Reverse Proxy
           │ HTTP / HTTPS
           ▼
┌─────────────────────┐
│    Origin Server    │
│                     │
│ Nginx / Apache / IIS│
│    Aplikasi Web     │
└─────────────────────┘
```

Client tidak berkomunikasi secara langsung dengan origin server. Request terlebih dahulu diterima oleh Brebes WAF untuk diperiksa oleh ModSecurity. Request yang diperbolehkan kemudian diteruskan oleh Nginx menuju origin server.

### Memastikan Konektivitas ke Origin Server

Sebelum melakukan pengujian melalui reverse proxy, pastikan server Brebes WAF dapat terhubung ke origin server.

Sebagai contoh, apabila origin server memiliki alamat IP:

```text
192.168.10.20
```

lakukan pengujian konektivitas:

```bash
ping -c 4 192.168.10.20
```

Perlu diperhatikan bahwa beberapa server atau firewall dapat memblokir ICMP. Oleh karena itu, kegagalan `ping` tidak selalu menunjukkan bahwa layanan web pada origin server tidak dapat diakses.

Pengujian yang lebih relevan adalah melakukan koneksi langsung ke layanan HTTP:

```bash
curl -I http://192.168.10.20
```

Apabila origin server menggunakan port tertentu, misalnya port `8080`, gunakan:

```bash
curl -I http://192.168.10.20:8080
```

Jika origin server menggunakan virtual host berdasarkan nama domain, sertakan header `Host`:

```bash
curl -I \
    -H "Host: app.example.com" \
    http://192.168.10.20
```

Apabila origin server dapat diakses dengan benar, server akan memberikan HTTP response, misalnya:

```text
HTTP/1.1 200 OK
```

Response lain seperti `301`, `302`, atau response aplikasi tertentu juga dapat menunjukkan bahwa koneksi ke origin server telah berhasil, tergantung konfigurasi aplikasi.

### Mengarahkan Reverse Proxy ke Origin Server

Pada konfigurasi virtual host yang telah dibuat sebelumnya, alamat origin server ditentukan melalui directive `proxy_pass`:

```nginx
location / {
    proxy_pass http://192.168.10.20;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    proxy_http_version 1.1;
}
```

Apabila origin server berjalan pada port `8080`, konfigurasi dapat diubah menjadi:

```nginx
proxy_pass http://192.168.10.20:8080;
```

Dengan konfigurasi tersebut, request yang diterima oleh Brebes WAF akan diteruskan ke alamat dan port yang telah ditentukan.

### Memastikan Origin Server Menerima Host yang Benar

Pada implementasi yang menggunakan beberapa website dalam satu origin server, Nginx atau Apache pada origin server biasanya menentukan aplikasi berdasarkan hostname.

Oleh karena itu, Brebes WAF meneruskan header `Host` menggunakan:

```nginx
proxy_set_header Host $host;
```

Sebagai contoh, ketika client mengakses:

```text
app.example.com
```

Brebes WAF akan meneruskan request menuju:

```text
192.168.10.20
```

dengan header:

```text
Host: app.example.com
```

Origin server kemudian menggunakan informasi tersebut untuk menentukan virtual host atau aplikasi yang harus menangani request.

Secara sederhana:

```text
Client
  │
  │ Host: app.example.com
  ▼
Brebes WAF
  │
  │ proxy_pass http://192.168.10.20
  │ Host: app.example.com
  ▼
Origin Server
  │
  ├── app.example.com  → Aplikasi A
  ├── web.example.com  → Aplikasi B
  └── api.example.com  → Aplikasi C
```

Dengan mekanisme ini, satu origin server dapat menjalankan beberapa aplikasi dan tetap menerima hostname asli yang diakses oleh client.

### Pengujian Melalui Brebes WAF

Setelah koneksi langsung ke origin server berhasil, lakukan pengujian melalui Brebes WAF.

Dari server Brebes WAF, jalankan:

```bash
curl -I \
    -H "Host: app.example.com" \
    http://127.0.0.1
```

Request tersebut akan masuk melalui Nginx Brebes WAF, diperiksa oleh ModSecurity, kemudian diteruskan menuju origin server.

Alur request yang terjadi adalah:

```text
curl
  │
  │ Host: app.example.com
  ▼
Nginx Brebes WAF
  │
  ▼
ModSecurity
  │
  ├── OWASP CRS
  │
  └── Brebes WAF Rules
  │
  ▼
Origin Server
  │
  ▼
Aplikasi Web
```

Apabila aplikasi memberikan response yang sesuai, maka komunikasi antara Brebes WAF dan origin server telah berhasil.

### Mencegah Akses Langsung ke Origin Server

Pada lingkungan production, sangat disarankan agar origin server tidak dapat diakses secara langsung dari internet.

Idealnya, firewall pada origin server hanya mengizinkan koneksi HTTP atau HTTPS yang berasal dari alamat IP Brebes WAF atau jaringan reverse proxy yang telah ditentukan.

Secara sederhana:

```text
Internet
   │
   │ HTTP/HTTPS
   ▼
Brebes WAF
   │
   │ ALLOW
   ▼
Origin Server


Internet
   │
   │ HTTP/HTTPS
   ▼
Origin Server
   │
   └── BLOCK
```

Pembatasan tersebut penting untuk mencegah pengguna atau penyerang melewati perlindungan WAF dengan mengakses alamat IP origin server secara langsung.

Dengan demikian, seluruh traffic publik menuju aplikasi dipaksa melewati Brebes WAF terlebih dahulu.

Setelah koneksi antara Brebes WAF dan origin server berhasil, langkah berikutnya adalah melakukan pengujian reverse proxy untuk memastikan request dari client benar-benar diteruskan melalui Brebes WAF sebelum mencapai aplikasi.

## Pengujian Reverse Proxy

Setelah Brebes WAF berhasil dihubungkan dengan origin server, langkah berikutnya adalah melakukan pengujian untuk memastikan mekanisme reverse proxy berjalan dengan benar.

Tujuan pengujian pada tahap ini adalah memastikan bahwa request yang dikirim oleh client diterima terlebih dahulu oleh Brebes WAF, kemudian diteruskan menuju origin server.

Alur yang akan diuji adalah:

```text
Client
   │
   │ HTTP Request
   ▼
Brebes WAF
   │
   │ Nginx Reverse Proxy
   │
   │ ModSecurity
   ▼
Origin Server
   │
   ▼
Aplikasi Web
```

Pada tahap ini, kita belum melakukan pengujian serangan. Pengujian difokuskan untuk memastikan request normal dapat melewati Brebes WAF dan mendapatkan response dari origin server.

### Menguji Akses Melalui Brebes WAF

Dari server Brebes WAF, lakukan pengujian menggunakan `curl`.

Sebagai contoh, apabila virtual host yang digunakan adalah:

```text
app.example.com
```

jalankan:

```bash
curl -I \
    -H "Host: app.example.com" \
    http://127.0.0.1
```

Perintah tersebut mengirimkan request ke Nginx pada server Brebes WAF dengan header:

```text
Host: app.example.com
```

Nginx akan mencocokkan header tersebut dengan konfigurasi:

```nginx
server_name app.example.com;
```

Kemudian request akan diteruskan menuju origin server melalui konfigurasi:

```nginx
proxy_pass http://192.168.10.20;
```

Apabila reverse proxy berjalan dengan benar, kita akan mendapatkan HTTP response dari aplikasi atau origin server.

Sebagai contoh:

```text
HTTP/1.1 200 OK
Server: nginx
Content-Type: text/html
```

Status HTTP yang diterima dapat berbeda tergantung konfigurasi aplikasi. Response seperti `200`, `301`, atau `302` dapat menjadi response yang valid.

### Menguji Konten Aplikasi

Selain menggunakan opsi `-I` untuk melihat HTTP response header, kita juga dapat melakukan request biasa:

```bash
curl \
    -H "Host: app.example.com" \
    http://127.0.0.1
```

Apabila reverse proxy bekerja dengan benar, output yang ditampilkan merupakan response yang berasal dari aplikasi pada origin server.

### Memeriksa Access Log Brebes WAF

Untuk memastikan request telah diterima oleh Brebes WAF, periksa access log virtual host:

```bash
sudo tail -f /var/log/nginx/app.example.com.access.log
```

Kemudian, dari terminal lain, kirim kembali request:

```bash
curl -I \
    -H "Host: app.example.com" \
    http://127.0.0.1
```

Request tersebut seharusnya muncul pada access log Brebes WAF.

Apabila konfigurasi menggunakan format log `reverser`, informasi yang tercatat dapat mencakup:

```text
Alamat IP client
HTTP request
HTTP status
Hostname
X-Forwarded-For
Request time
Upstream response time
Alamat upstream
```

Keberadaan nilai `upstream` pada log dapat membantu memastikan bahwa Nginx meneruskan request menuju origin server.

### Memeriksa Access Log Origin Server

Selanjutnya, periksa access log pada origin server.

Apabila origin server menggunakan Nginx, perintah yang digunakan dapat berupa:

```bash
sudo tail -f /var/log/nginx/access.log
```

Kemudian kirim kembali request melalui Brebes WAF.

Request yang sama seharusnya tercatat pada:

```text
Brebes WAF Access Log
        │
        ▼
Origin Server Access Log
```

Hal tersebut menunjukkan bahwa request telah melewati reverse proxy dan berhasil diteruskan menuju origin server.

Perlu diperhatikan bahwa alamat IP yang terlihat pada access log origin server secara default dapat berupa alamat IP Brebes WAF. Informasi alamat IP client asli diteruskan melalui header:

```text
X-Real-IP
X-Forwarded-For
```

Apabila origin server perlu menggunakan alamat IP client asli untuk logging atau kebutuhan aplikasi, web server pada origin harus dikonfigurasi untuk mempercayai header tersebut hanya dari alamat IP Brebes WAF yang tepercaya.

### Menguji dari Komputer Client

Setelah pengujian dari server Brebes WAF berhasil, lakukan pengujian dari komputer client.

Pastikan DNS domain:

```text
app.example.com
```

mengarah ke alamat IP Brebes WAF, bukan langsung ke origin server.

Kemudian akses menggunakan browser:

```text
http://app.example.com
```

Atau lakukan pengujian menggunakan `curl` dari komputer client:

```bash
curl -I http://app.example.com
```

Apabila DNS belum dikonfigurasi, pengujian dapat dilakukan menggunakan `curl --resolve`.

Sebagai contoh, apabila alamat IP Brebes WAF adalah `192.168.10.10`:

```bash
curl --resolve app.example.com:80:192.168.10.10 \
    http://app.example.com/
```

Dengan cara tersebut, `curl` akan mengarahkan domain `app.example.com` menuju alamat IP Brebes WAF tanpa harus melakukan perubahan DNS.

### Memastikan Request Melewati Brebes WAF

Pengujian dapat dianggap berhasil apabila kondisi berikut terpenuhi:

1. Client dapat mengakses domain aplikasi melalui Brebes WAF.
2. Request tercatat pada access log Nginx Brebes WAF.
3. Brebes WAF meneruskan request menuju origin server.
4. Request tercatat pada access log origin server.
5. Origin server memberikan response kepada Brebes WAF.
6. Response berhasil dikembalikan kepada client.

Alur komunikasi yang telah berhasil diuji menjadi:

```text
Client
   │
   │ Request
   ▼
Brebes WAF Access Log
   │
   ▼
ModSecurity Inspection
   │
   ▼
Nginx Reverse Proxy
   │
   ▼
Origin Server Access Log
   │
   ▼
Application
   │
   │ Response
   ▼
Brebes WAF
   │
   ▼
Client
```

Apabila seluruh pengujian tersebut berhasil, mekanisme reverse proxy Brebes WAF telah berjalan dengan benar.

Tahap berikutnya adalah melakukan pengujian ModSecurity untuk memastikan request berbahaya dapat dideteksi atau diblokir sebelum mencapai origin server.

## Pengujian ModSecurity

Setelah reverse proxy berhasil diuji, langkah berikutnya adalah memastikan ModSecurity benar-benar aktif dan dapat melakukan inspeksi terhadap request yang masuk melalui Brebes WAF.

Pada tahap ini, pengujian dilakukan menggunakan rule sederhana yang dibuat khusus untuk kebutuhan pengujian. Tujuannya adalah membuktikan bahwa ModSecurity dapat menerima request, memproses rule, dan melakukan blocking sebelum kita melakukan pengujian menggunakan OWASP Core Rule Set dan rule Brebes WAF.

Pengujian dilakukan pada lingkungan lab dan sebaiknya tidak dilakukan terhadap sistem yang tidak memiliki izin untuk diuji.

### Memastikan ModSecurity Aktif

Sebelum melakukan pengujian, pastikan ModSecurity berada dalam mode aktif.

Jalankan:

```bash
grep -E "^SecRuleEngine" /etc/nginx/modsecurity.conf
```

Pada konfigurasi Brebes WAF yang digunakan dalam tutorial ini, hasilnya adalah:

```text
SecRuleEngine On
```

Konfigurasi tersebut menunjukkan bahwa ModSecurity berada dalam mode aktif dan dapat melakukan tindakan disruptive seperti memblokir request apabila rule yang digunakan menentukan tindakan tersebut.

Apabila konfigurasi menggunakan:

```text
SecRuleEngine DetectionOnly
```

ModSecurity hanya akan melakukan deteksi dan pencatatan tanpa melakukan blocking terhadap request.

### Membuat Rule Pengujian

Untuk memastikan ModSecurity benar-benar bekerja, kita dapat membuat sebuah rule sederhana yang akan memblokir request apabila URL mengandung parameter tertentu.

Buat file:

```bash
sudo nano /opt/Brebes-WAF/rules/99-testing/9999999-modsecurity-test.conf
```

Apabila direktori belum tersedia, buat terlebih dahulu:

```bash
sudo mkdir -p /opt/Brebes-WAF/rules/99-testing
```

Kemudian masukkan rule berikut:

```apache
SecRule ARGS:test "@streq BREBES-WAF-TEST" \
    "id:9999999,\
    phase:2,\
    deny,\
    status:403,\
    log,\
    msg:'BREBES-WAF ModSecurity Test Rule'"
```

Rule tersebut akan memeriksa parameter bernama `test`.

Apabila request memiliki parameter:

```text
test=BREBES-WAF-TEST
```

ModSecurity akan memberikan response:

```text
403 Forbidden
```

Rule ID `9999999` pada contoh ini digunakan khusus untuk pengujian. Pastikan ID tersebut tidak bertabrakan dengan rule lain yang digunakan pada sistem.

### Deployment Ulang Rule Brebes WAF

Karena kita menambahkan file rule baru ke dalam direktori:

```text
/opt/Brebes-WAF/rules/
```

jalankan kembali script deployment:

```bash
sudo /opt/Brebes-WAF/scripts/deploy.sh
```

Script akan menemukan file `.conf` yang baru dan membuat ulang:

```text
/etc/nginx/modsecurity_includes.conf
```

Jumlah rule yang sebelumnya:

```text
Rule Count: 23
```

seharusnya bertambah menjadi:

```text
Rule Count: 24
```

Verifikasi bahwa rule pengujian telah dimuat:

```bash
grep "9999999-modsecurity-test.conf" \
    /etc/nginx/modsecurity_includes.conf
```

Hasilnya seharusnya menunjukkan:

```text
Include /opt/Brebes-WAF/rules/99-testing/9999999-modsecurity-test.conf
```

Script deployment juga akan menjalankan pengujian konfigurasi Nginx dan melakukan reload apabila konfigurasi valid.

### Mengirim Request Normal

Sebelum mengirim request yang akan memicu rule, lakukan pengujian request normal.

Sebagai contoh:

```bash
curl -i \
    -H "Host: app.example.com" \
    "http://127.0.0.1/?test=NORMAL"
```

Request tersebut tidak memenuhi kondisi rule pengujian sehingga seharusnya diteruskan menuju origin server.

Response yang diterima dapat berupa:

```text
HTTP/1.1 200 OK
```

atau status HTTP lain yang memang diberikan oleh aplikasi.

### Mengirim Request yang Memicu Rule

Selanjutnya, kirim request yang sesuai dengan kondisi rule:

```bash
curl -i \
    -H "Host: app.example.com" \
    "http://127.0.0.1/?test=BREBES-WAF-TEST"
```

Karena parameter `test` memiliki nilai yang sesuai dengan rule:

```text
BREBES-WAF-TEST
```

ModSecurity seharusnya menghentikan request dan memberikan response:

```text
HTTP/1.1 403 Forbidden
```

Alur yang terjadi adalah:

```text
Client
   │
   │ ?test=BREBES-WAF-TEST
   ▼
Nginx
   │
   ▼
ModSecurity
   │
   │ Rule ID 9999999 Match
   ▼
403 Forbidden
   │
   X
Origin Server
```

Apabila response `403 Forbidden` diterima, hal tersebut membuktikan bahwa ModSecurity telah aktif dan mampu melakukan blocking terhadap request berdasarkan rule yang telah ditentukan.

### Memeriksa Log ModSecurity

Setelah request diblokir, periksa audit log ModSecurity.

Lokasi audit log bergantung pada konfigurasi directive `SecAuditLog` yang digunakan.

Untuk mengetahui lokasi audit log:

```bash
grep -E "^SecAuditLog " /etc/nginx/modsecurity.conf
```

Kemudian periksa file log yang ditampilkan oleh konfigurasi tersebut.

Sebagai contoh, apabila menggunakan:

```text
SecAuditLog /var/log/modsecurity/audit.log
```

kita dapat menjalankan:

```bash
sudo tail -f /var/log/modsecurity/audit.log
```

Cari informasi yang berkaitan dengan:

```text
9999999
```

atau pesan:

```text
BREBES-WAF ModSecurity Test Rule
```

Informasi tersebut menunjukkan bahwa request berhasil dideteksi oleh rule pengujian.

### Menghapus Rule Pengujian

Setelah pengujian selesai, rule pengujian sebaiknya dihapus karena rule tersebut hanya digunakan untuk membuktikan bahwa mekanisme blocking ModSecurity bekerja.

Hapus file:

```bash
sudo rm /opt/Brebes-WAF/rules/99-testing/9999999-modsecurity-test.conf
```

Kemudian jalankan kembali deployment:

```bash
sudo /opt/Brebes-WAF/scripts/deploy.sh
```

Jumlah rule seharusnya kembali menjadi:

```text
Rule Count: 23
```

Pastikan rule pengujian sudah tidak terdapat dalam file include:

```bash
grep "9999999-modsecurity-test.conf" \
    /etc/nginx/modsecurity_includes.conf
```

Apabila tidak menghasilkan output, rule pengujian sudah tidak dimuat.

Dengan pengujian tersebut, kita telah memastikan bahwa ModSecurity dapat melakukan inspeksi dan memblokir request sebelum request diteruskan menuju origin server.

Tahap berikutnya adalah melakukan pengujian terhadap OWASP Core Rule Set menggunakan beberapa pola serangan pada lingkungan lab, dimulai dengan pengujian SQL Injection.

## Pengujian SQL Injection

Setelah memastikan ModSecurity dapat melakukan inspeksi dan blocking terhadap request, langkah berikutnya adalah melakukan pengujian terhadap OWASP Core Rule Set menggunakan pola SQL Injection.

SQL Injection merupakan salah satu jenis serangan yang memanfaatkan input aplikasi untuk menyisipkan perintah atau ekspresi SQL. Apabila aplikasi tidak melakukan validasi dan parameterisasi query dengan benar, serangan ini dapat digunakan untuk memanipulasi query yang dijalankan oleh database.

Dalam arsitektur Brebes WAF, OWASP Core Rule Set menyediakan rule yang dapat membantu mendeteksi berbagai pola SQL Injection pada HTTP request.

Rule SQL Injection pada instalasi OWASP CRS yang digunakan dalam tutorial ini berada pada:

```text
/usr/share/modsecurity-crs/rules/REQUEST-942-APPLICATION-ATTACK-SQLI.conf
```

Pengujian berikut hanya dilakukan pada server atau aplikasi lab yang kita miliki dan digunakan untuk memastikan konfigurasi WAF bekerja dengan benar.

### Memastikan Rule SQL Injection Tersedia

Sebelum melakukan pengujian, pastikan rule SQL Injection OWASP CRS tersedia:

```bash
ls -lah \
    /usr/share/modsecurity-crs/rules/REQUEST-942-APPLICATION-ATTACK-SQLI.conf
```

Kemudian pastikan OWASP CRS dimuat melalui konfigurasi:

```bash
cat /etc/nginx/modsecurity/crs-load.conf
```

Pada implementasi ini, terdapat konfigurasi:

```text
Include /usr/share/modsecurity-crs/rules/*.conf
```

Dengan konfigurasi tersebut, seluruh file rule OWASP CRS dengan ekstensi `.conf`, termasuk rule SQL Injection, akan dimuat oleh ModSecurity.

### Mengirim Request Normal

Sebelum melakukan pengujian pola SQL Injection, kirim terlebih dahulu request normal.

Sebagai contoh:

```bash
curl -i \
    -H "Host: app.example.com" \
    "http://127.0.0.1/?id=1"
```

Request tersebut seharusnya dapat diteruskan menuju origin server dan mendapatkan response normal dari aplikasi.

Status HTTP yang diterima dapat berupa:

```text
HTTP/1.1 200 OK
```

atau status lain yang memang diberikan oleh aplikasi.

### Mengirim Request dengan Pola SQL Injection

Selanjutnya, kita dapat mengirim request pengujian yang mengandung pola SQL Injection sederhana.

Gunakan `curl` dengan parameter yang telah di-URL-encode:

```bash
curl -i \
    -H "Host: app.example.com" \
    "http://127.0.0.1/?id=1%27%20OR%20%271%27%3D%271"
```

Setelah URL decoding, nilai parameter tersebut merepresentasikan pola:

```text
1' OR '1'='1
```

Request akan diterima oleh Nginx dan diperiksa oleh ModSecurity menggunakan OWASP Core Rule Set.

Secara sederhana, alur pemeriksaannya adalah:

```text
Client
   │
   │ Request dengan pola SQL Injection
   ▼
Brebes WAF
   │
   ▼
ModSecurity
   │
   ▼
OWASP CRS
   │
   └── REQUEST-942-APPLICATION-ATTACK-SQLI.conf
             │
             ▼
      SQL Injection Detected
             │
             ▼
        403 Forbidden
             │
             X
       Origin Server
```

Apabila request memenuhi kondisi blocking berdasarkan konfigurasi OWASP CRS dan anomaly score yang digunakan, response yang diterima seharusnya berupa:

```text
HTTP/1.1 403 Forbidden
```

Hal tersebut menunjukkan bahwa request dihentikan oleh WAF sebelum diteruskan menuju origin server.

### Membandingkan Request Normal dan Request Berbahaya

Kita dapat membandingkan kedua pengujian:

```text
Request Normal
?id=1
   │
   ▼
Brebes WAF
   │
   ▼
Diizinkan
   │
   ▼
Origin Server


Request SQL Injection
?id=1' OR '1'='1
   │
   ▼
Brebes WAF
   │
   ▼
OWASP CRS
   │
   ▼
Diblokir
   │
   X
Origin Server
```

Dengan pengujian tersebut, kita dapat memastikan bahwa request normal tetap dapat diteruskan, sedangkan pola request yang terdeteksi sebagai SQL Injection dapat dihentikan oleh WAF.

### Memeriksa ModSecurity Audit Log

Setelah melakukan pengujian, periksa audit log ModSecurity untuk melihat rule yang terpicu.

Untuk mengetahui lokasi audit log yang digunakan:

```bash
grep -E "^SecAuditLog " /etc/nginx/modsecurity.conf
```

Kemudian periksa log tersebut.

Sebagai contoh:

```bash
sudo tail -f /var/log/modsecurity/audit.log
```

Lokasi file harus disesuaikan dengan nilai `SecAuditLog` pada konfigurasi ModSecurity.

Untuk mencari deteksi SQL Injection pada audit log, kita juga dapat menggunakan:

```bash
sudo grep -i "SQL Injection" /var/log/modsecurity/audit.log | tail
```

Pada log tersebut biasanya dapat ditemukan informasi mengenai rule OWASP CRS yang terpicu, pesan deteksi, request yang diperiksa, dan tindakan yang dilakukan oleh ModSecurity.

### Memastikan Request Tidak Mencapai Origin Server

Untuk membuktikan bahwa request yang diblokir tidak mencapai aplikasi, kita dapat memantau access log pada origin server.

Pada origin server, jalankan:

```bash
sudo tail -f /var/log/nginx/access.log
```

Kemudian kirim kembali request pengujian SQL Injection melalui Brebes WAF.

Apabila request mendapatkan `403 Forbidden` dari Brebes WAF dan tidak muncul pada access log origin server, hal tersebut menunjukkan bahwa request telah dihentikan sebelum mencapai aplikasi.

Dengan demikian, alur blocking yang terjadi adalah:

```text
Client
   │
   │ SQL Injection Pattern
   ▼
Brebes WAF
   │
   ▼
ModSecurity + OWASP CRS
   │
   ├── Detection
   └── Blocking
         │
         ▼
    403 Forbidden

Origin Server
   ▲
   X
Request tidak diteruskan
```

Perlu diperhatikan bahwa keberhasilan WAF dalam mendeteksi pola SQL Injection tidak menggantikan kebutuhan untuk mengamankan aplikasi. Aplikasi tetap harus menggunakan mekanisme seperti parameterized query atau prepared statement, validasi input, serta pembatasan hak akses database.

WAF berfungsi sebagai lapisan perlindungan tambahan dalam pendekatan defense in depth.

Setelah pengujian SQL Injection berhasil, tahap berikutnya adalah melakukan pengujian Cross-Site Scripting atau XSS untuk memastikan OWASP CRS dapat mendeteksi pola serangan pada input HTML dan JavaScript.

## Pengujian Cross-Site Scripting

Setelah melakukan pengujian SQL Injection, langkah berikutnya adalah menguji kemampuan OWASP Core Rule Set dalam mendeteksi pola Cross-Site Scripting atau XSS.

Cross-Site Scripting merupakan jenis serangan yang memanfaatkan input aplikasi untuk menyisipkan kode atau script berbahaya ke dalam halaman web. Apabila aplikasi tidak melakukan validasi input dan output encoding dengan benar, script tersebut berpotensi dijalankan pada browser pengguna.

Dalam arsitektur Brebes WAF, OWASP Core Rule Set menyediakan rule untuk membantu mendeteksi berbagai pola XSS pada HTTP request.

Rule XSS pada instalasi OWASP CRS yang digunakan dalam tutorial ini berada pada:

```text
/usr/share/modsecurity-crs/rules/REQUEST-941-APPLICATION-ATTACK-XSS.conf
```

Pengujian berikut dilakukan pada lingkungan lab untuk memastikan konfigurasi ModSecurity dan OWASP CRS bekerja dengan benar.

### Memastikan Rule XSS Tersedia

Sebelum melakukan pengujian, pastikan rule XSS tersedia:

```bash
ls -lah \
    /usr/share/modsecurity-crs/rules/REQUEST-941-APPLICATION-ATTACK-XSS.conf
```

Karena konfigurasi:

```text
/etc/nginx/modsecurity/crs-load.conf
```

telah memuat:

```text
Include /usr/share/modsecurity-crs/rules/*.conf
```

maka rule XSS tersebut akan ikut dimuat oleh ModSecurity.

### Mengirim Request Normal

Sebelum melakukan pengujian pola XSS, kirim terlebih dahulu request normal.

Sebagai contoh:

```bash
curl -i \
    -H "Host: app.example.com" \
    "http://127.0.0.1/?search=hello"
```

Request tersebut seharusnya dapat melewati Brebes WAF dan diteruskan menuju origin server.

Response yang diterima dapat berupa:

```text
HTTP/1.1 200 OK
```

atau status HTTP lain sesuai dengan response aplikasi.

### Mengirim Request dengan Pola XSS

Selanjutnya, kirim request yang mengandung pola XSS sederhana.

Untuk menghindari masalah karakter khusus pada shell dan URL, gunakan parameter yang telah di-URL-encode:

```bash
curl -i \
    -H "Host: app.example.com" \
    "http://127.0.0.1/?search=%3Cscript%3Ealert%281%29%3C%2Fscript%3E"
```

Setelah proses URL decoding, nilai parameter tersebut merepresentasikan:

```html
<script>alert(1)</script>
```

Request akan diterima oleh Nginx dan diperiksa oleh ModSecurity menggunakan OWASP Core Rule Set.

Secara sederhana, proses yang terjadi adalah:

```text
Client
   │
   │ Request dengan pola XSS
   ▼
Brebes WAF
   │
   ▼
ModSecurity
   │
   ▼
OWASP CRS
   │
   └── REQUEST-941-APPLICATION-ATTACK-XSS.conf
             │
             ▼
        XSS Detected
             │
             ▼
        403 Forbidden
             │
             X
       Origin Server
```

Apabila pola tersebut terdeteksi dan memenuhi kondisi blocking berdasarkan konfigurasi OWASP CRS yang digunakan, Brebes WAF seharusnya memberikan response:

```text
HTTP/1.1 403 Forbidden
```

Dengan demikian, request dihentikan pada lapisan WAF dan tidak diteruskan menuju aplikasi.

### Membandingkan Request Normal dan Request XSS

Hasil pengujian dapat dibandingkan sebagai berikut:

```text
Request Normal
?search=hello
      │
      ▼
Brebes WAF
      │
      ▼
Diizinkan
      │
      ▼
Origin Server


Request XSS
?search=<script>alert(1)</script>
      │
      ▼
Brebes WAF
      │
      ▼
OWASP CRS
      │
      ▼
Diblokir
      │
      X
Origin Server
```

Request normal seharusnya tetap diteruskan menuju aplikasi, sedangkan request yang mengandung pola XSS dapat dideteksi dan diblokir oleh ModSecurity.

### Memeriksa ModSecurity Audit Log

Setelah melakukan pengujian, periksa audit log ModSecurity untuk mengetahui rule yang terpicu.

Pertama, pastikan lokasi audit log:

```bash
grep -E "^SecAuditLog " /etc/nginx/modsecurity.conf
```

Kemudian periksa log tersebut. Sebagai contoh:

```bash
sudo tail -f /var/log/modsecurity/audit.log
```

Lokasi file harus disesuaikan dengan konfigurasi `SecAuditLog` yang digunakan.

Untuk mencari deteksi yang berkaitan dengan XSS, dapat menggunakan:

```bash
sudo grep -i "Cross Site Scripting" \
    /var/log/modsecurity/audit.log | tail
```

atau:

```bash
sudo grep -i "XSS" \
    /var/log/modsecurity/audit.log | tail
```

Audit log dapat digunakan untuk mengetahui rule ID yang terpicu, pesan deteksi, request yang diperiksa, serta tindakan yang dilakukan oleh ModSecurity.

### Memastikan Request Tidak Mencapai Origin Server

Untuk memastikan request XSS benar-benar dihentikan oleh Brebes WAF, pantau access log pada origin server.

Pada origin server, jalankan:

```bash
sudo tail -f /var/log/nginx/access.log
```

Kemudian kirim kembali request pengujian:

```bash
curl -i \
    -H "Host: app.example.com" \
    "http://127.0.0.1/?search=%3Cscript%3Ealert%281%29%3C%2Fscript%3E"
```

Apabila Brebes WAF memberikan response `403 Forbidden` dan request tidak muncul pada access log origin server, hal tersebut menunjukkan bahwa request telah dihentikan sebelum mencapai aplikasi.

Alur blocking yang terjadi menjadi:

```text
Client
   │
   │ XSS Pattern
   ▼
Brebes WAF
   │
   ▼
ModSecurity + OWASP CRS
   │
   ├── Detection
   └── Blocking
         │
         ▼
    403 Forbidden

Origin Server
   ▲
   X
Request tidak diteruskan
```

Perlu dipahami bahwa perlindungan WAF terhadap XSS merupakan lapisan keamanan tambahan dan tidak menggantikan mekanisme keamanan pada aplikasi.

Aplikasi tetap harus menerapkan validasi input sesuai konteks, output encoding atau escaping, Content Security Policy apabila sesuai, serta menghindari penggunaan API browser yang tidak aman dalam memproses input pengguna.

Dengan pengujian ini, kita dapat memastikan OWASP CRS mampu melakukan inspeksi terhadap request yang mengandung pola Cross-Site Scripting.

Tahap berikutnya adalah melakukan pengujian Path Traversal untuk memastikan WAF dapat mendeteksi pola request yang mencoba mengakses file atau direktori di luar lokasi yang seharusnya.

## Pengujian Path Traversal

Setelah melakukan pengujian SQL Injection dan Cross-Site Scripting, langkah berikutnya adalah menguji kemampuan OWASP Core Rule Set dalam mendeteksi pola Path Traversal.

Path Traversal merupakan teknik yang mencoba mengakses file atau direktori di luar lokasi yang seharusnya dapat diakses oleh aplikasi. Serangan ini biasanya memanfaatkan karakter seperti `../` untuk berpindah ke direktori di atasnya.

Apabila aplikasi memiliki celah keamanan dan menggunakan input pengguna secara tidak aman untuk menentukan lokasi file, serangan Path Traversal dapat memungkinkan penyerang mencoba mengakses file sensitif pada sistem.

Dalam arsitektur Brebes WAF, OWASP Core Rule Set menyediakan rule Local File Inclusion atau LFI yang dapat membantu mendeteksi berbagai pola serangan yang berkaitan dengan akses file dan Path Traversal.

Rule tersebut pada instalasi OWASP CRS yang digunakan dalam tutorial ini berada pada:

```text
/usr/share/modsecurity-crs/rules/REQUEST-930-APPLICATION-ATTACK-LFI.conf
```

Pengujian berikut dilakukan pada lingkungan lab yang kita miliki untuk memastikan konfigurasi ModSecurity dan OWASP CRS bekerja dengan benar.

### Memastikan Rule LFI Tersedia

Sebelum melakukan pengujian, pastikan rule Local File Inclusion tersedia:

```bash
ls -lah \
    /usr/share/modsecurity-crs/rules/REQUEST-930-APPLICATION-ATTACK-LFI.conf
```

Karena konfigurasi:

```text
/etc/nginx/modsecurity/crs-load.conf
```

telah memuat:

```text
Include /usr/share/modsecurity-crs/rules/*.conf
```

maka rule tersebut akan ikut dimuat oleh ModSecurity.

### Mengirim Request Normal

Sebelum mengirim request dengan pola Path Traversal, lakukan pengujian menggunakan parameter normal.

Sebagai contoh:

```bash
curl -i \
    -H "Host: app.example.com" \
    "http://127.0.0.1/?file=document.pdf"
```

Request tersebut seharusnya dapat melewati Brebes WAF dan diteruskan menuju origin server.

Response yang diterima dapat berupa:

```text
HTTP/1.1 200 OK
```

atau status HTTP lain yang memang diberikan oleh aplikasi.

### Mengirim Request dengan Pola Path Traversal

Selanjutnya, lakukan pengujian menggunakan pola Path Traversal yang telah di-URL-encode:

```bash
curl -i \
    -H "Host: app.example.com" \
    "http://127.0.0.1/?file=..%2F..%2F..%2F..%2Fetc%2Fpasswd"
```

Setelah proses URL decoding, nilai parameter tersebut merepresentasikan:

```text
../../../../etc/passwd
```

String tersebut digunakan sebagai pola pengujian untuk mengetahui apakah OWASP CRS dapat mendeteksi upaya Path Traversal.

Request akan diterima oleh Nginx dan diperiksa oleh ModSecurity menggunakan OWASP Core Rule Set.

Secara sederhana, proses pemeriksaannya adalah:

```text
Client
   │
   │ Request dengan pola Path Traversal
   ▼
Brebes WAF
   │
   ▼
ModSecurity
   │
   ▼
OWASP CRS
   │
   └── REQUEST-930-APPLICATION-ATTACK-LFI.conf
             │
             ▼
   Path Traversal / LFI Detected
             │
             ▼
        403 Forbidden
             │
             X
       Origin Server
```

Apabila request terdeteksi dan memenuhi kondisi blocking berdasarkan konfigurasi OWASP CRS yang digunakan, Brebes WAF seharusnya memberikan response:

```text
HTTP/1.1 403 Forbidden
```

Dengan demikian, request dihentikan oleh WAF sebelum diteruskan menuju origin server.

### Membandingkan Request Normal dan Path Traversal

Hasil kedua pengujian dapat dibandingkan sebagai berikut:

```text
Request Normal
?file=document.pdf
        │
        ▼
   Brebes WAF
        │
        ▼
    Diizinkan
        │
        ▼
  Origin Server


Request Path Traversal
?file=../../../../etc/passwd
        │
        ▼
   Brebes WAF
        │
        ▼
ModSecurity + OWASP CRS
        │
        ▼
    Diblokir
        │
        X
  Origin Server
```

Dengan pengujian tersebut, request normal seharusnya tetap dapat diteruskan menuju aplikasi, sedangkan request yang mengandung pola Path Traversal dapat dideteksi dan diblokir oleh ModSecurity.

### Memeriksa ModSecurity Audit Log

Setelah melakukan pengujian, periksa audit log ModSecurity untuk mengetahui rule yang terpicu.

Pastikan terlebih dahulu lokasi audit log:

```bash
grep -E "^SecAuditLog " /etc/nginx/modsecurity.conf
```

Kemudian periksa file log tersebut. Sebagai contoh:

```bash
sudo tail -f /var/log/modsecurity/audit.log
```

Lokasi file harus disesuaikan dengan konfigurasi `SecAuditLog` yang digunakan.

Untuk mencari log yang berkaitan dengan Local File Inclusion atau Path Traversal, dapat menggunakan:

```bash
sudo grep -i "Local File Inclusion" \
    /var/log/modsecurity/audit.log | tail
```

atau mencari berdasarkan rule ID yang tercatat pada hasil deteksi.

Audit log dapat memberikan informasi mengenai request yang diperiksa, rule yang terpicu, pesan deteksi, anomaly score, dan tindakan yang dilakukan oleh ModSecurity.

### Memastikan Request Tidak Mencapai Origin Server

Untuk membuktikan bahwa request Path Traversal telah dihentikan oleh Brebes WAF, pantau access log pada origin server:

```bash
sudo tail -f /var/log/nginx/access.log
```

Kemudian kirim kembali request pengujian:

```bash
curl -i \
    -H "Host: app.example.com" \
    "http://127.0.0.1/?file=..%2F..%2F..%2F..%2Fetc%2Fpasswd"
```

Apabila Brebes WAF memberikan response `403 Forbidden` dan request tersebut tidak muncul pada access log origin server, hal tersebut menunjukkan bahwa request telah dihentikan pada lapisan WAF.

Alur blocking yang terjadi adalah:

```text
Client
   │
   │ Path Traversal Pattern
   ▼
Brebes WAF
   │
   ▼
ModSecurity + OWASP CRS
   │
   ├── Detection
   └── Blocking
         │
         ▼
    403 Forbidden

Origin Server
   ▲
   X
Request tidak diteruskan
```

Perlu dipahami bahwa WAF merupakan lapisan perlindungan tambahan dan tidak menggantikan pengamanan pada aplikasi.

Aplikasi tetap harus melakukan validasi terhadap nama dan lokasi file, menggunakan allowlist apabila memungkinkan, membatasi direktori yang dapat diakses, serta memastikan input pengguna tidak digunakan secara langsung untuk membentuk filesystem path.

Dengan pengujian ini, kita telah melakukan pengujian dasar terhadap beberapa kategori serangan yang dapat dideteksi oleh OWASP Core Rule Set, yaitu SQL Injection, Cross-Site Scripting, dan Path Traversal.

Tahap berikutnya adalah mempelajari ModSecurity Audit Log secara lebih detail untuk mengetahui bagaimana Brebes WAF mencatat request yang terdeteksi dan rule keamanan yang terpicu.

## Memeriksa ModSecurity Audit Log

Setelah melakukan pengujian terhadap SQL Injection, Cross-Site Scripting, dan Path Traversal, langkah berikutnya adalah memeriksa ModSecurity Audit Log.

Audit log merupakan salah satu komponen penting dalam implementasi Web Application Firewall karena menyimpan informasi mengenai request yang diperiksa oleh ModSecurity. Informasi tersebut dapat digunakan untuk melakukan analisis terhadap serangan, mengetahui rule yang terpicu, melakukan troubleshooting, serta membantu proses tuning rule untuk mengurangi false positive.

### Memeriksa Konfigurasi Audit Log

Lokasi dan mekanisme pencatatan audit log ditentukan pada konfigurasi ModSecurity.

Untuk melihat konfigurasi audit log yang digunakan, jalankan:

```bash
grep -E "^SecAudit" /etc/nginx/modsecurity.conf
```

Hasil perintah tersebut akan menampilkan konfigurasi yang berkaitan dengan audit log, seperti:

```text
SecAuditEngine RelevantOnly
SecAuditLogRelevantStatus "^(?:5|4(?!04))"
SecAuditLogParts ABIJDEFHZ
SecAuditLogType Serial
SecAuditLog /var/log/modsec_audit.log
```

Konfigurasi pada masing-masing implementasi dapat berbeda. Oleh karena itu, lokasi file audit log harus disesuaikan dengan nilai `SecAuditLog` yang digunakan pada konfigurasi ModSecurity.

Sebagai contoh, apabila konfigurasi menunjukkan:

```text
SecAuditLog /var/log/modsec_audit.log
```

maka audit log dapat diperiksa menggunakan:

```bash
sudo tail -f /var/log/modsec_audit.log
```

### Memantau Deteksi Secara Real-Time

Untuk memantau audit log secara langsung, jalankan:

```bash
sudo tail -f /var/log/modsec_audit.log
```

Kemudian, dari terminal lain, lakukan salah satu pengujian yang telah dilakukan sebelumnya.

Sebagai contoh, kirim request pengujian SQL Injection:

```bash
curl -i \
    -H "Host: app.example.com" \
    "http://127.0.0.1/?id=1%27%20OR%20%271%27%3D%271"
```

Apabila request terdeteksi oleh ModSecurity, informasi mengenai request dan rule yang terpicu akan dicatat pada audit log.

### Informasi yang Terdapat pada Audit Log

ModSecurity Audit Log dapat menyimpan berbagai informasi mengenai transaksi HTTP yang diperiksa.

Informasi tersebut dapat mencakup:

```text
Transaction ID
Timestamp
Client IP Address
HTTP Method
Request URI
HTTP Request Header
HTTP Response Status
Rule ID
Rule Message
Matched Data
Severity
File Rule
Anomaly Score
```

Informasi yang tersedia bergantung pada konfigurasi `SecAuditLogParts` yang digunakan.

Salah satu informasi penting yang perlu diperhatikan adalah `id`, yang menunjukkan rule ModSecurity yang terpicu.

Sebagai contoh:

```text
[id "942100"]
[msg "SQL Injection Attack Detected"]
```

Informasi tersebut menunjukkan bahwa request memicu rule dengan ID `942100`.

Selain rule ID, audit log juga dapat menunjukkan file tempat rule tersebut didefinisikan:

```text
[file "/usr/share/modsecurity-crs/rules/REQUEST-942-APPLICATION-ATTACK-SQLI.conf"]
```

Informasi ini dapat membantu administrator mengetahui apakah deteksi berasal dari OWASP Core Rule Set atau custom rule Brebes WAF.

### Mencari Deteksi Berdasarkan Jenis Serangan

Untuk mempermudah analisis, kita dapat menggunakan `grep` untuk mencari jenis deteksi tertentu.

Untuk mencari deteksi SQL Injection:

```bash
sudo grep -i "SQL Injection" /var/log/modsec_audit.log | tail
```

Untuk mencari deteksi Cross-Site Scripting:

```bash
sudo grep -i "Cross Site Scripting" /var/log/modsec_audit.log | tail
```

Untuk mencari deteksi Local File Inclusion:

```bash
sudo grep -i "Local File Inclusion" /var/log/modsec_audit.log | tail
```

Kita juga dapat mencari berdasarkan rule ID tertentu:

```bash
sudo grep 'id "942100"' /var/log/modsec_audit.log
```

Sedangkan untuk mencari deteksi yang berasal dari custom rule Brebes WAF, pencarian dapat dilakukan menggunakan rentang atau rule ID yang digunakan oleh Brebes WAF.

### Memahami Transaction ID

Setiap transaksi yang dicatat oleh ModSecurity memiliki transaction ID yang dapat digunakan untuk menghubungkan berbagai bagian log dari request yang sama.

Audit log ModSecurity dapat terdiri dari beberapa bagian yang dipisahkan menggunakan identifier transaksi.

Secara sederhana:

```text
Request Client
      │
      ▼
Transaction ID
      │
      ├── Request Header
      ├── Request Body
      ├── Rule yang Terpicu
      ├── Response Header
      └── Informasi Audit
```

Transaction ID sangat berguna ketika satu request memicu beberapa rule sekaligus.

Sebagai contoh, sebuah request dapat memicu rule deteksi SQL Injection dan pada saat yang sama meningkatkan anomaly score. Dengan melihat transaction ID yang sama, administrator dapat menganalisis seluruh rangkaian deteksi sebagai satu transaksi.

### Memahami Anomaly Score

OWASP Core Rule Set menggunakan pendekatan anomaly scoring untuk menentukan tingkat risiko sebuah request.

Sebuah request dapat memicu satu atau beberapa rule. Setiap rule dapat menambahkan nilai tertentu pada anomaly score.

Secara sederhana:

```text
HTTP Request
      │
      ▼
Rule A Match
      │
      ├── + Anomaly Score
      │
Rule B Match
      │
      ├── + Anomaly Score
      │
      ▼
Total Anomaly Score
      │
      ├── Di bawah threshold → Request diteruskan
      │
      └── Mencapai threshold → Request diblokir
```

Oleh karena itu, ketika melakukan analisis audit log, administrator tidak hanya perlu melihat satu rule yang terpicu, tetapi juga keseluruhan rule dan anomaly score yang dihasilkan oleh transaksi tersebut.

### Membedakan Rule OWASP CRS dan Brebes WAF

Dalam implementasi ini, ModSecurity menggunakan dua kelompok rule utama:

```text
ModSecurity
    │
    ├── OWASP Core Rule Set
    │       │
    │       ├── SQL Injection
    │       ├── XSS
    │       ├── LFI
    │       ├── RFI
    │       └── Kategori serangan lainnya
    │
    └── Brebes WAF Rules
            │
            ├── Upload Protection
            └── Webshell Detection
```

Audit log dapat digunakan untuk mengetahui rule mana yang melakukan deteksi dengan melihat informasi `id`, `msg`, dan `file`.

Apabila nilai `file` mengarah ke:

```text
/usr/share/modsecurity-crs/rules/
```

maka deteksi berasal dari OWASP Core Rule Set.

Sedangkan apabila mengarah ke:

```text
/opt/Brebes-WAF/rules/
```

maka deteksi berasal dari custom rule Brebes WAF.

### Menggunakan Script Check Last Detection

Repository Brebes WAF juga menyediakan script:

```text
/opt/Brebes-WAF/scripts/check-last-detection.sh
```

Script tersebut dapat digunakan sebagai utilitas untuk membantu memeriksa deteksi terakhir yang tercatat oleh WAF.

Sebelum menggunakannya, periksa terlebih dahulu isi script:

```bash
cat /opt/Brebes-WAF/scripts/check-last-detection.sh
```

Pastikan lokasi log yang digunakan oleh script sesuai dengan konfigurasi `SecAuditLog` pada server.

Apabila script telah memiliki permission execute, script dapat dijalankan menggunakan:

```bash
sudo /opt/Brebes-WAF/scripts/check-last-detection.sh
```

Jika belum, berikan permission terlebih dahulu:

```bash
sudo chmod +x /opt/Brebes-WAF/scripts/check-last-detection.sh
```

Kemudian jalankan kembali script tersebut.

### Pentingnya Monitoring Audit Log

Audit log tidak hanya digunakan ketika melakukan pengujian WAF. Pada lingkungan production, log merupakan salah satu sumber informasi penting untuk mengetahui aktivitas keamanan yang terjadi pada aplikasi.

Data audit log dapat digunakan untuk:

- Mengidentifikasi pola serangan yang sering terjadi.
- Mengetahui aplikasi yang menjadi target serangan.
- Mengetahui rule yang paling sering terpicu.
- Menganalisis false positive.
- Melakukan tuning terhadap rule WAF.
- Membantu proses investigasi insiden keamanan.
- Menjadi sumber data untuk sistem monitoring atau SIEM.

Namun, audit log ModSecurity dapat bertambah besar apabila traffic aplikasi tinggi. Oleh karena itu, administrator perlu memperhatikan kapasitas penyimpanan, mekanisme log rotation, retensi log, dan pengiriman log ke sistem monitoring terpusat apabila diperlukan.

Dengan melakukan pemeriksaan terhadap ModSecurity Audit Log, administrator dapat mengetahui tidak hanya apakah sebuah request diblokir, tetapi juga alasan request tersebut terdeteksi, rule yang terpicu, serta informasi lain yang diperlukan untuk melakukan analisis keamanan.

Pada tahap berikutnya, kita akan membahas troubleshooting beberapa permasalahan yang umum ditemukan ketika mengimplementasikan Nginx, ModSecurity, OWASP Core Rule Set, dan Brebes WAF.

## Troubleshooting

Dalam proses implementasi Brebes WAF, beberapa permasalahan dapat terjadi pada konfigurasi Nginx, ModSecurity, OWASP Core Rule Set, custom rule Brebes WAF, maupun koneksi antara reverse proxy dengan origin server.

Bagian ini membahas beberapa permasalahan umum yang dapat ditemukan selama proses implementasi dan langkah awal yang dapat dilakukan untuk melakukan troubleshooting.

### Nginx Gagal Melakukan Reload

Setelah melakukan perubahan konfigurasi, Nginx mungkin gagal melakukan reload karena terdapat kesalahan pada konfigurasi.

Sebelum melakukan reload, selalu lakukan pengujian konfigurasi:

```bash
sudo nginx -t
```

Apabila terdapat kesalahan, Nginx akan menampilkan informasi mengenai file dan baris konfigurasi yang bermasalah.

Setelah masalah diperbaiki, jalankan kembali:

```bash
sudo nginx -t
```

Jika konfigurasi sudah valid, lakukan reload:

```bash
sudo systemctl reload nginx
```

Untuk melihat status service Nginx:

```bash
sudo systemctl status nginx
```

Sedangkan untuk melihat log service:

```bash
sudo journalctl -u nginx -n 100 --no-pager
```

### Directive ModSecurity Tidak Dikenali

Apabila muncul error seperti:

```text
unknown directive "modsecurity"
```

kemungkinan module ModSecurity untuk Nginx belum dimuat.

Periksa module yang aktif:

```bash
ls -lah /etc/nginx/modules-enabled/
```

Pastikan terdapat symbolic link:

```text
50-mod-http-modsecurity.conf
```

Kemudian periksa konfigurasi module:

```bash
cat /usr/share/nginx/modules-available/mod-http-modsecurity.conf
```

Setelah itu lakukan pengujian:

```bash
sudo nginx -t
```

Pada instalasi Ubuntu yang digunakan dalam tutorial ini, module ModSecurity dipasang melalui package:

```text
libnginx-mod-http-modsecurity
```

Status package dapat diperiksa menggunakan:

```bash
dpkg -l | grep -i modsecurity
```

### ModSecurity Aktif tetapi Request Tidak Diblokir

Apabila request berbahaya tidak diblokir, periksa terlebih dahulu mode ModSecurity:

```bash
grep -E "^SecRuleEngine" /etc/nginx/modsecurity.conf
```

Untuk melakukan blocking, konfigurasi yang digunakan adalah:

```text
SecRuleEngine On
```

Apabila menggunakan:

```text
SecRuleEngine DetectionOnly
```

ModSecurity akan melakukan deteksi dan logging, tetapi tidak melakukan blocking terhadap request.

Selanjutnya, pastikan ModSecurity telah diaktifkan pada konfigurasi utama Nginx:

```bash
sudo nginx -T | grep -E "modsecurity|modsecurity_rules_file"
```

Pada implementasi Brebes WAF dalam tutorial ini, konfigurasi yang digunakan adalah:

```nginx
modsecurity on;
modsecurity_rules_file /etc/nginx/modsecurity_includes.conf;
```

### OWASP CRS Tidak Termuat

Apabila pengujian SQL Injection, XSS, atau Path Traversal tidak menghasilkan deteksi, periksa konfigurasi CRS:

```bash
cat /etc/nginx/modsecurity/crs-load.conf
```

Pada implementasi ini, konfigurasi yang digunakan adalah:

```text
Include /etc/modsecurity/crs/crs-setup.conf
Include /etc/modsecurity/crs/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf
Include /usr/share/modsecurity-crs/rules/*.conf
Include /etc/modsecurity/crs/RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf
```

Pastikan direktori rule tersedia:

```bash
ls -lah /usr/share/modsecurity-crs/rules/
```

Kemudian pastikan `crs-load.conf` dimuat oleh:

```text
/etc/nginx/modsecurity_includes.conf
```

Periksa menggunakan:

```bash
grep "crs-load.conf" /etc/nginx/modsecurity_includes.conf
```

### Rule Brebes WAF Tidak Termuat

Apabila custom rule Brebes WAF tidak terdeteksi, periksa jumlah file rule:

```bash
find /opt/Brebes-WAF/rules \
    -type f \
    -name "*.conf" | wc -l
```

Kemudian periksa rule yang dimuat:

```bash
grep "^Include /opt/Brebes-WAF/rules" \
    /etc/nginx/modsecurity_includes.conf
```

Perlu diperhatikan bahwa script deployment hanya memuat file dengan ekstensi:

```text
.conf
```

File dengan ekstensi:

```text
.conf.off
```

tidak akan dimuat.

Apabila terdapat perubahan atau penambahan rule pada direktori Brebes WAF, jalankan kembali:

```bash
sudo /opt/Brebes-WAF/scripts/deploy.sh
```

Script akan membuat ulang file:

```text
/etc/nginx/modsecurity_includes.conf
```

kemudian melakukan pengujian konfigurasi dan reload Nginx.

### Reverse Proxy Menghasilkan 502 Bad Gateway

Response:

```text
502 Bad Gateway
```

biasanya menunjukkan bahwa Nginx Brebes WAF tidak dapat berkomunikasi dengan origin server.

Lakukan pengujian langsung dari server Brebes WAF:

```bash
curl -I http://192.168.10.20
```

Apabila origin menggunakan port `8080`:

```bash
curl -I http://192.168.10.20:8080
```

Jika origin menggunakan virtual host berdasarkan domain:

```bash
curl -I \
    -H "Host: app.example.com" \
    http://192.168.10.20
```

Periksa juga konfigurasi:

```nginx
proxy_pass http://192.168.10.20;
```

Pastikan alamat IP dan port sesuai dengan layanan yang berjalan pada origin server.

Selain itu, periksa kemungkinan firewall yang membatasi komunikasi antara Brebes WAF dan origin server.

### Reverse Proxy Menghasilkan 504 Gateway Timeout

Response:

```text
504 Gateway Timeout
```

dapat terjadi apabila origin server tidak memberikan response dalam waktu yang diharapkan.

Periksa terlebih dahulu koneksi langsung:

```bash
curl -v http://192.168.10.20
```

Kemudian periksa error log Nginx:

```bash
sudo tail -f /var/log/nginx/error.log
```

Masalah timeout tidak selalu disebabkan oleh konfigurasi WAF. Penyebabnya dapat berasal dari aplikasi yang lambat, database, konektivitas jaringan, atau proses pada origin server.

Hindari langsung meningkatkan nilai timeout tanpa mengetahui penyebab utama masalah.

### Domain Mengarah Langsung ke Origin Server

Apabila domain masih mengarah langsung ke origin server, traffic dapat melewati Brebes WAF.

Periksa resolusi DNS:

```bash
dig app.example.com
```

atau:

```bash
nslookup app.example.com
```

Alamat IP publik yang digunakan oleh domain seharusnya mengarah ke Brebes WAF atau perangkat jaringan yang meneruskan traffic menuju Brebes WAF.

Pada lingkungan production, akses langsung dari internet menuju origin server sebaiknya dibatasi menggunakan firewall.

### Origin Server Menerima IP Brebes WAF sebagai Client IP

Karena Brebes WAF bertindak sebagai reverse proxy, origin server secara default akan melihat alamat IP Brebes WAF sebagai sumber koneksi.

Brebes WAF telah meneruskan informasi IP client melalui:

```nginx
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

Apabila origin server perlu menggunakan IP client asli, web server pada origin harus dikonfigurasi untuk memproses header tersebut.

Konfigurasi trust hanya boleh diberikan kepada alamat IP reverse proxy Brebes WAF yang diketahui. Jangan mempercayai `X-Forwarded-For` dari semua sumber karena header tersebut dapat dipalsukan oleh client.

### Terjadi False Positive

False positive terjadi ketika request yang sebenarnya valid terdeteksi sebagai serangan.

Apabila hal ini terjadi, jangan langsung menonaktifkan seluruh ModSecurity atau OWASP CRS.

Periksa terlebih dahulu audit log untuk mengetahui:

```text
Transaction ID
Rule ID
Rule Message
Matched Data
Request URI
Parameter
Anomaly Score
```

Cari rule yang menyebabkan blocking dan tentukan apakah request tersebut benar-benar merupakan false positive.

Pengecualian rule sebaiknya dibuat secara spesifik berdasarkan kebutuhan aplikasi, misalnya hanya untuk parameter atau endpoint tertentu.

Secara konseptual:

```text
False Positive
      │
      ▼
Periksa Audit Log
      │
      ▼
Identifikasi Rule ID
      │
      ▼
Identifikasi Endpoint / Parameter
      │
      ▼
Buat Exclusion Spesifik
      │
      ▼
Uji Kembali
```

Hindari menonaktifkan seluruh kelompok rule hanya untuk menyelesaikan satu false positive.

### Audit Log Tidak Muncul

Apabila ModSecurity melakukan blocking tetapi audit log tidak ditemukan, periksa konfigurasi:

```bash
grep -E "^SecAudit" /etc/nginx/modsecurity.conf
```

Perhatikan nilai:

```text
SecAuditEngine
SecAuditLog
SecAuditLogType
SecAuditLogParts
```

Kemudian pastikan direktori atau file tujuan log dapat ditulis oleh proses yang menjalankan Nginx.

Periksa juga error log:

```bash
sudo tail -f /var/log/nginx/error.log
```

### Memeriksa Log Secara Bersamaan

Ketika melakukan troubleshooting, akan lebih mudah apabila beberapa log dipantau secara bersamaan.

Pada Brebes WAF:

```bash
sudo tail -f /var/log/nginx/access.log
```

Pada terminal lain:

```bash
sudo tail -f /var/log/nginx/error.log
```

Kemudian pantau ModSecurity Audit Log sesuai dengan lokasi yang dikonfigurasi:

```bash
sudo tail -f /var/log/modsec_audit.log
```

Pada origin server, pantau access log:

```bash
sudo tail -f /var/log/nginx/access.log
```

Dengan membandingkan log tersebut, administrator dapat mengetahui pada bagian mana sebuah request mengalami masalah.

Alur troubleshooting dapat dilakukan sebagai berikut:

```text
Client
   │
   ▼
Brebes WAF Access Log
   │
   ├── Tidak masuk
   │      └── Periksa DNS / Firewall / Network
   │
   ▼
ModSecurity
   │
   ├── Diblokir
   │      └── Periksa Audit Log dan Rule ID
   │
   ▼
Reverse Proxy
   │
   ├── 502 / 504
   │      └── Periksa konektivitas Origin Server
   │
   ▼
Origin Access Log
   │
   ├── Request masuk
   │      └── Periksa aplikasi / web server
   │
   ▼
Application Response
```

Pendekatan troubleshooting secara berurutan akan membantu mempersempit sumber permasalahan tanpa harus menonaktifkan mekanisme keamanan yang telah diterapkan.

Setelah seluruh proses instalasi, konfigurasi, deployment, pengujian, dan troubleshooting selesai dilakukan, tahap terakhir adalah menarik kesimpulan dari implementasi Brebes WAF sebagai lapisan perlindungan aplikasi web.

## Kesimpulan

Pada artikel ini kita telah melakukan implementasi Brebes WAF menggunakan Nginx sebagai reverse proxy dan ModSecurity sebagai Web Application Firewall.

Implementasi dimulai dengan menyiapkan Ubuntu Server, menginstal Nginx dan ModSecurity, mengaktifkan OWASP Core Rule Set, kemudian melakukan clone repository Brebes WAF yang berisi custom rule dan script deployment.

Dalam arsitektur yang telah dibangun, seluruh request HTTP yang menuju aplikasi terlebih dahulu melewati Brebes WAF sebelum diteruskan menuju origin server.

Secara sederhana, arsitektur yang telah diimplementasikan adalah:

```text
Internet / Client
        │
        │ HTTP / HTTPS Request
        ▼
┌───────────────────────────┐
│        Brebes WAF         │
│                           │
│          Nginx            │
│            │              │
│            ▼              │
│       ModSecurity         │
│            │              │
│      ┌─────┴─────┐        │
│      │           │        │
│      ▼           ▼        │
│  OWASP CRS   Brebes WAF   │
│                Rules      │
└─────────────┬─────────────┘
              │
              │ Reverse Proxy
              ▼
┌───────────────────────────┐
│       Origin Server       │
│                           │
│   Nginx / Apache / IIS    │
│       Aplikasi Web        │
└───────────────────────────┘
```

Nginx menerima request dari client dan bertindak sebagai reverse proxy, sedangkan ModSecurity melakukan inspeksi terhadap request berdasarkan rule keamanan yang telah dimuat.

Dalam implementasi ini terdapat dua kelompok rule utama, yaitu OWASP Core Rule Set dan custom rule Brebes WAF.

OWASP Core Rule Set memberikan perlindungan umum terhadap berbagai pola serangan aplikasi web, sedangkan custom rule Brebes WAF dikembangkan untuk menambahkan mekanisme deteksi yang lebih spesifik sesuai dengan kebutuhan dan pola ancaman yang ditemukan.

Pada proses pengujian, kita telah mencoba beberapa pola serangan seperti:

- SQL Injection
- Cross-Site Scripting
- Path Traversal

Pengujian tersebut digunakan untuk memastikan bahwa ModSecurity dan OWASP Core Rule Set dapat melakukan inspeksi terhadap request yang masuk dan melakukan blocking ketika request memenuhi kondisi yang ditentukan oleh rule.

Selain melakukan blocking, ModSecurity juga menghasilkan audit log yang dapat digunakan untuk mengetahui berbagai informasi mengenai aktivitas keamanan, seperti rule yang terpicu, jenis serangan yang terdeteksi, request yang diperiksa, dan informasi lain yang dapat membantu proses analisis keamanan.

Dengan mekanisme tersebut, alur perlindungan aplikasi menjadi:

```text
HTTP Request
     │
     ▼
Brebes WAF
     │
     ▼
ModSecurity Inspection
     │
     ├── Request Normal
     │       │
     │       ▼
     │   Origin Server
     │
     └── Request Berbahaya
             │
             ▼
        Block / Log
```

Namun perlu dipahami bahwa implementasi WAF bukan merupakan solusi tunggal untuk mengamankan aplikasi web.

WAF tidak dapat menggantikan penerapan secure coding, patch management, vulnerability assessment, hardening sistem operasi dan web server, pengamanan database, serta mekanisme autentikasi dan otorisasi yang baik.

WAF sebaiknya ditempatkan sebagai salah satu bagian dari strategi **defense in depth**, di mana setiap lapisan infrastruktur memiliki mekanisme keamanan masing-masing.

```text
Internet
   │
   ▼
Network Firewall
   │
   ▼
Brebes WAF
   │
   ▼
Reverse Proxy
   │
   ▼
Web Server
   │
   ▼
Application
   │
   ▼
Database
```

Dalam lingkungan production, implementasi Brebes WAF juga membutuhkan proses monitoring dan tuning secara berkelanjutan.

Administrator perlu melakukan analisis terhadap audit log untuk mengetahui pola serangan yang terjadi dan mengidentifikasi kemungkinan false positive. Rule yang terlalu agresif dapat mengganggu fungsi aplikasi, sedangkan rule yang terlalu longgar dapat mengurangi efektivitas perlindungan.

Oleh karena itu, pengelolaan WAF merupakan proses yang terus berkembang mengikuti perubahan aplikasi dan pola ancaman keamanan.

Custom rule Brebes WAF juga dapat terus dikembangkan berdasarkan hasil analisis serangan yang ditemukan di lapangan. Pola serangan yang sebelumnya belum terdeteksi dapat dianalisis dan digunakan sebagai dasar untuk membuat rule baru.

Dengan pendekatan tersebut, Brebes WAF tidak hanya berfungsi sebagai reverse proxy yang dilengkapi Web Application Firewall, tetapi juga dapat dikembangkan menjadi platform perlindungan aplikasi web yang disesuaikan dengan kebutuhan lingkungan tempat sistem tersebut diimplementasikan.

Implementasi yang dibahas dalam artikel ini merupakan fondasi awal. Pengembangan selanjutnya dapat mencakup peningkatan custom rule, pengelolaan false positive, rate limiting, integrasi threat intelligence, sentralisasi log, monitoring keamanan, integrasi dengan SIEM, high availability, serta otomatisasi deployment dan pembaruan rule.

Pada akhirnya, tujuan utama pembangunan Brebes WAF adalah menambahkan satu lapisan pertahanan sebelum request dari internet mencapai aplikasi.

Dengan kombinasi Nginx, ModSecurity, OWASP Core Rule Set, custom rule Brebes WAF, serta monitoring yang dilakukan secara berkelanjutan, kita dapat membangun mekanisme perlindungan terpusat yang membantu meningkatkan keamanan aplikasi web tanpa menghilangkan tanggung jawab untuk tetap memperbaiki keamanan pada sisi aplikasi dan infrastruktur.