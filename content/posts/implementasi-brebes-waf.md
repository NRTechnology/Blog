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


## Instalasi ModSecurity

## Memastikan ModSecurity Berjalan pada Nginx

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