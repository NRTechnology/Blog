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

## Struktur Direktori Brebes WAF

## Konfigurasi Brebes WAF

## Mengaktifkan Rule ModSecurity

## Konfigurasi Nginx sebagai Reverse Proxy

## Menghubungkan Brebes WAF dengan Origin Server

## Pengujian Reverse Proxy

## Pengujian ModSecurity

## Pengujian SQL Injection

## Pengujian Cross-Site Scripting

## Pengujian Path Traversal

## Memeriksa ModSecurity Audit Log

## Mengubah DetectionOnly menjadi Blocking

## Troubleshooting

## Kesimpulan