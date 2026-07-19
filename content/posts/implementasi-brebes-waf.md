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

## Persiapan Server

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