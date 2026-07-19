+++
date = '2026-07-19T14:00:00+07:00'
draft = false
title = 'Membangun Brebes WAF: Perlindungan Terpusat untuk Aplikasi Web'
categories = ['Cyber Security']
tags = [
    'Brebes WAF',
    'Web Application Firewall',
    'WAF',
    'Nginx',
    'ModSecurity',
    'Reverse Proxy',
    'Web Security',
    'Cyber Security',
    'Defense in Depth'
]
+++

Aplikasi berbasis web merupakan salah satu layanan yang banyak digunakan dalam penyelenggaraan sistem pemerintahan. Semakin banyak aplikasi yang dipublikasikan melalui internet, semakin besar pula kebutuhan untuk memberikan perlindungan terhadap berbagai ancaman keamanan yang menargetkan aplikasi web.

Serangan terhadap aplikasi web tidak selalu terjadi karena kelemahan pada infrastruktur server. Celah keamanan juga dapat ditemukan pada source code aplikasi, framework, library, maupun komponen pihak ketiga yang digunakan oleh aplikasi.

Beberapa jenis serangan yang umum ditemukan antara lain SQL Injection, Cross-Site Scripting (XSS), Local File Inclusion (LFI), Remote File Inclusion (RFI), Remote Code Execution (RCE), Command Injection, Path Traversal, hingga eksploitasi terhadap celah keamanan yang belum diperbaiki.

Untuk menambahkan lapisan perlindungan terhadap aplikasi web yang dipublikasikan ke internet, dibutuhkan sebuah mekanisme keamanan yang dapat melakukan inspeksi terhadap HTTP request sebelum request tersebut diteruskan ke aplikasi.

Salah satu teknologi yang dapat digunakan adalah Web Application Firewall atau WAF.

## Apa Itu Brebes WAF?

Brebes WAF merupakan konsep platform Web Application Firewall yang ditempatkan sebagai lapisan keamanan di depan aplikasi web.

Dalam arsitektur ini, pengguna dari internet tidak berkomunikasi secara langsung dengan web server aplikasi. Seluruh HTTP dan HTTPS request terlebih dahulu melewati WAF untuk dilakukan pemeriksaan berdasarkan rule dan kebijakan keamanan yang telah ditentukan.

Secara sederhana, arsitekturnya adalah sebagai berikut:

Internet
    │
    ▼
Firewall
    │
    ▼
Brebes WAF
    │
    ├── HTTP/HTTPS Inspection
    ├── Security Rules
    ├── Attack Detection
    ├── Request Filtering
    └── Security Logging
    │
    ▼
Web Server / Reverse Proxy
    │
    ▼
Aplikasi Web

Dengan arsitektur tersebut, WAF berfungsi sebagai salah satu lapisan pertahanan sebelum request mencapai aplikasi.

## Mengapa Membutuhkan WAF?

Dalam pengelolaan banyak aplikasi web, tidak semua aplikasi dikembangkan menggunakan teknologi, framework, dan standar keamanan yang sama.

Sebagian aplikasi mungkin merupakan aplikasi baru yang dikembangkan menggunakan framework modern, sedangkan aplikasi lainnya merupakan aplikasi lama yang masih harus tetap beroperasi.

Kondisi tersebut membuat penerapan standar keamanan pada level aplikasi menjadi lebih kompleks.

WAF dapat digunakan sebagai lapisan keamanan tambahan yang diterapkan secara terpusat tanpa harus melakukan perubahan langsung terhadap seluruh source code aplikasi.

Namun perlu dipahami bahwa WAF bukan pengganti keamanan aplikasi. Celah keamanan pada aplikasi tetap harus diperbaiki pada source code. WAF berfungsi sebagai bagian dari pendekatan defense in depth untuk membantu mengurangi risiko eksploitasi.

## Konsep Reverse Proxy

Brebes WAF bekerja dengan konsep reverse proxy.

Ketika pengguna mengakses sebuah aplikasi, request akan diterima terlebih dahulu oleh WAF. Setelah request diperiksa dan dinyatakan sesuai dengan kebijakan keamanan, request akan diteruskan menuju origin server.

Alur request menjadi:

Client
    │
    │ HTTPS Request
    ▼
Brebes WAF
    │
    │ Security Inspection
    ▼
Origin Web Server
    │
    ▼
Application

Apabila request terdeteksi sebagai serangan, WAF dapat menghentikan request tersebut sebelum mencapai aplikasi.

## Ancaman yang Dapat Dideteksi

Dengan rule yang tepat, WAF dapat membantu mendeteksi atau memblokir berbagai pola serangan terhadap aplikasi web, seperti:

- SQL Injection
- Cross-Site Scripting (XSS)
- Local File Inclusion (LFI)
- Remote File Inclusion (RFI)
- Path Traversal
- Command Injection
- Beberapa pola Remote Code Execution (RCE)
- Malicious Bot
- HTTP Protocol Abuse

Efektivitas perlindungan tetap bergantung pada teknologi WAF, rule yang digunakan, konfigurasi, tuning, dan karakteristik masing-masing aplikasi.

## Defense in Depth

Brebes WAF sebaiknya tidak berdiri sendiri sebagai satu-satunya mekanisme keamanan.

Perlindungan aplikasi web dapat dibangun menggunakan beberapa lapisan:

```text
Internet
    │
    ▼
Network Firewall
    │
    ▼
Web Application Firewall
    │
    ▼
Reverse Proxy / Nginx
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

Setiap lapisan memiliki fungsi keamanan yang berbeda.

Network firewall membatasi akses pada level jaringan, sedangkan WAF melakukan inspeksi terhadap komunikasi HTTP dan HTTPS.

Web server menerapkan hardening pada level server, sementara aplikasi tetap bertanggung jawab terhadap validasi input, autentikasi, otorisasi, session management, dan keamanan proses bisnis.

## Logging dan Monitoring

Salah satu fungsi penting dari WAF adalah menghasilkan security log.

Log tersebut dapat digunakan untuk mengetahui pola serangan yang diarahkan terhadap aplikasi web.

Informasi yang dapat dianalisis antara lain:

- Waktu terjadinya serangan
- Alamat IP sumber
- Domain yang menjadi target
- URL yang diakses
- HTTP method
- Rule keamanan yang terpicu
- Jenis serangan yang terdeteksi
- Tindakan yang dilakukan oleh WAF

Data tersebut dapat menjadi sumber informasi untuk proses monitoring dan incident response.

## Tantangan Implementasi WAF

Implementasi WAF tidak hanya sebatas mengaktifkan rule sebanyak mungkin.

Rule keamanan yang terlalu agresif dapat menyebabkan false positive, yaitu request aplikasi yang sebenarnya valid tetapi dianggap sebagai serangan.

Sebaliknya, rule yang terlalu longgar dapat menyebabkan serangan tidak terdeteksi.

Oleh karena itu, implementasi WAF membutuhkan proses tuning secara bertahap.

Pada tahap awal, rule tertentu dapat dijalankan dalam mode monitoring untuk melihat dampaknya terhadap aplikasi. Setelah dipastikan tidak mengganggu fungsi aplikasi, rule dapat diterapkan dalam mode blocking.

## Kesimpulan

Brebes WAF merupakan salah satu upaya untuk membangun lapisan perlindungan terpusat bagi aplikasi web yang dipublikasikan melalui internet.

Dengan menempatkan WAF di depan origin web server, HTTP dan HTTPS request dapat diperiksa terlebih dahulu sebelum diteruskan ke aplikasi.

Pendekatan ini dapat membantu mendeteksi dan mengurangi berbagai serangan terhadap aplikasi web sekaligus menyediakan security logging yang dapat digunakan untuk kebutuhan monitoring dan analisis keamanan.

Namun, WAF tetap merupakan salah satu bagian dari strategi keamanan secara keseluruhan.

Keamanan aplikasi tetap membutuhkan penerapan secure coding, patch management, hardening server, vulnerability assessment, monitoring, backup, serta mekanisme incident response yang baik.

Pada artikel berikutnya, kita akan mulai membangun arsitektur Brebes WAF dan membahas komponen yang digunakan untuk melakukan reverse proxy, inspeksi HTTP request, serta penerapan Web Application Firewall rule.