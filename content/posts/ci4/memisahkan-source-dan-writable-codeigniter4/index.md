---
title: "Memisahkan Source Code dan Direktori Writable pada CodeIgniter 4"
slug: "memisahkan-source-dan-writable-codeigniter4"
date: 2026-08-03T22:00:00+07:00
lastmod: 2026-08-03T22:00:00+07:00
draft: false

description: "Panduan memisahkan source code dan direktori writable pada CodeIgniter 4 agar deployment lebih aman, mudah di-backup, dan sesuai untuk lingkungan produksi."

author: "NR Technology"

series:
  - Docker PHP Production
weight: 7

tags:
  - CodeIgniter4
  - Docker
  - PHP-FPM
  - Linux
  - Security
  - DevOps

categories:
  - CodeIgniter
  - Docker

keywords:
  - codeigniter writable
  - codeigniter deployment
  - docker codeigniter
  - php docker
  - source code readonly

toc: true
showReadingTime: true
showWordCount: true
showCodeCopyButtons: true
---

# Memisahkan Source Code dan Direktori Writable pada CodeIgniter 4

Salah satu praktik terbaik ketika melakukan deployment aplikasi **CodeIgniter 4** adalah memisahkan **source code** dari **direktori runtime**.

Banyak implementasi masih meletakkan seluruh aplikasi, termasuk direktori `writable`, di dalam satu lokasi. Pendekatan tersebut memang sederhana, tetapi kurang ideal untuk lingkungan produksi karena seluruh aplikasi menjadi dapat ditulis oleh proses PHP.

Pada artikel ini kita akan membangun struktur deployment yang memisahkan source code dan data runtime sehingga aplikasi menjadi lebih aman, mudah diperbarui, dan lebih mudah dibackup.

---

# Mengenal Direktori Writable

CodeIgniter 4 menggunakan direktori:

```text
writable/
```

Direktori tersebut digunakan untuk menyimpan berbagai data yang berubah selama aplikasi berjalan, antara lain:

- Cache aplikasi
- Session
- File log
- File upload
- Temporary file
- Debugbar

Karena isi direktori ini selalu berubah, maka sebaiknya tidak disimpan bersama source code.

---

# Mengapa Source Code Tidak Boleh Dapat Ditulis?

Source code aplikasi seharusnya hanya berubah ketika administrator melakukan deployment atau update aplikasi.

Apabila proses PHP memiliki hak tulis terhadap seluruh project, maka berbagai risiko dapat muncul, misalnya:

- Web shell dapat mengubah source code.
- Framework dapat dimodifikasi.
- File aplikasi dapat dihapus.
- Backdoor dapat ditanamkan.
- Sulit memverifikasi integritas aplikasi.

Prinsip yang baik adalah:

> **Source code bersifat statis, sedangkan data runtime bersifat dinamis.**

---

# Struktur Deployment yang Direkomendasikan

Berikut struktur direktori yang digunakan pada artikel ini.

```text
/opt/docker/apps/
└── myapp/
    ├── docker-compose.yml
    └── zz-custom.conf

/var/apps/
└── myapp/
    ├── backup/
    ├── data/
    │   └── writable/
    ├── htdocs/
    └── logs/
```

Penjelasan setiap direktori.

| Direktori | Fungsi |
|-----------|--------|
| htdocs | Source Code CodeIgniter 4 |
| data/writable | Data runtime aplikasi |
| backup | Backup aplikasi dan database |
| logs | Log tambahan aplikasi |
| docker-compose.yml | Konfigurasi Docker Compose |
| zz-custom.conf | Konfigurasi PHP-FPM |

---

# Membuat Struktur Direktori

Buat struktur direktori berikut.

```bash
mkdir -p /var/apps/myapp

mkdir -p /var/apps/myapp/backup

mkdir -p /var/apps/myapp/data/writable

mkdir -p /var/apps/myapp/htdocs

mkdir -p /var/apps/myapp/logs

mkdir -p /opt/docker/apps/myapp
```

Hasilnya.

```text
/opt/docker/apps/
└── myapp/
    ├── docker-compose.yml
    └── zz-custom.conf

/var/apps/
└── myapp/
    ├── backup/
    ├── data/
    │   └── writable/
    ├── htdocs/
    └── logs/
```

---

# Menyalin Source Code

Misalkan source aplikasi berada pada direktori lain.

Salin seluruh project ke direktori `htdocs`.

```bash
cp -a /path/source/. \
      /var/apps/myapp/htdocs/
```

Perintah tersebut akan menyalin seluruh file beserta atributnya.

---

# Memindahkan Direktori Writable

Selanjutnya salin isi direktori `writable`.

```bash
cp -a \
/var/apps/myapp/htdocs/writable/. \
/var/apps/myapp/data/writable/
```

Direktori `data/writable` kini menjadi lokasi utama penyimpanan seluruh data runtime.

---

# Mengosongkan Writable pada Source

Setelah seluruh isi berhasil disalin, kosongkan isi direktori `htdocs/writable`.

```bash
find /var/apps/myapp/htdocs/writable \
    -mindepth 1 \
    -exec rm -rf {} +
```

Perintah tersebut hanya menghapus isi direktori.

Direktori `writable` tetap dipertahankan sebagai **mount point** Docker.

---

# Mengapa Direktori Writable Tidak Dihapus?

Docker melakukan bind mount berdasarkan direktori tujuan.

Apabila direktori `writable` tidak ada di dalam source code, Docker akan mencoba membuat direktori tersebut.

Karena source code dipasang sebagai **read-only**, Docker tidak dapat membuat mount point baru sehingga container gagal dijalankan.

Oleh karena itu:

- isi direktori dihapus,
- tetapi direktorinya tetap dipertahankan.

---

# Menggunakan Docker Compose

Mount source code sebagai **read-only**.

```yaml
- /var/apps/myapp/htdocs:/var/www/html:ro
```

Sedangkan direktori writable dipasang sebagai **read-write**.

```yaml
- /var/apps/myapp/data/writable:/var/www/html/writable:rw
```

Dengan konfigurasi tersebut:

- Framework tidak dapat diubah.
- Data runtime tetap dapat ditulis.
- Deployment menjadi lebih aman.

---

# Mengatur Permission

PHP-FPM biasanya dijalankan menggunakan user:

```text
www-data
```

Pastikan kepemilikan direktori writable sesuai.

```bash
chown -R 33:33 \
/var/apps/myapp/data/writable
```

Kemudian atur permission.

```bash
chmod -R 775 \
/var/apps/myapp/data/writable
```

Hindari penggunaan permission `777` karena memberikan hak tulis kepada semua pengguna.

---

# Verifikasi di Dalam Container

Setelah container dijalankan.

```bash
docker compose up -d
```

Masuk ke container.

```bash
docker exec -it myapp-php bash
```

Periksa struktur direktori.

```bash
ls -lah /var/www/html
```

Pastikan direktori `writable` berasal dari bind mount host.

---

# Melakukan Backup

Dengan struktur yang baru, proses backup menjadi jauh lebih sederhana.

Backup source code.

```bash
tar czf source.tar.gz \
/var/apps/myapp/htdocs
```

Backup data runtime.

```bash
tar czf writable.tar.gz \
/var/apps/myapp/data/writable
```

Backup database.

```bash
mysqldump db_myapp > database.sql
```

Administrator dapat memilih hanya membackup source code, hanya data runtime, atau keduanya.

---

# Update Aplikasi

Ketika terdapat versi baru aplikasi, cukup lakukan update pada direktori:

```text
htdocs/
```

Sedangkan direktori:

```text
data/writable/
```

tetap dipertahankan.

Dengan cara ini:

- Session pengguna tetap tersedia.
- File upload tidak hilang.
- Cache dapat dipertahankan atau dibersihkan sesuai kebutuhan.

---

# Keuntungan Pendekatan Ini

Beberapa keuntungan yang diperoleh antara lain:

- Source code tidak dapat dimodifikasi oleh PHP.
- Risiko web shell berkurang.
- Framework tetap bersih.
- Backup lebih sederhana.
- Restore lebih cepat.
- Update aplikasi lebih mudah.
- Struktur deployment lebih konsisten.

Pendekatan ini juga mempermudah integrasi dengan Git karena direktori runtime tidak lagi bercampur dengan source code.

---

# Best Practice

Pada lingkungan produksi saya menerapkan beberapa prinsip berikut.

- Source code bersifat read-only.
- Direktori writable dipisahkan.
- Nginx hanya mengakses direktori `public`.
- PHP-FPM hanya memiliki hak tulis pada direktori runtime.
- Setiap aplikasi memiliki container PHP-FPM sendiri.
- Backup source code dan data runtime dilakukan secara terpisah.

Dengan struktur tersebut, pengelolaan banyak aplikasi pada satu server menjadi lebih sederhana dan lebih aman.

---

# Kesimpulan

Memisahkan source code dan direktori `writable` merupakan salah satu praktik terbaik dalam deployment CodeIgniter 4.

Dengan memanfaatkan bind mount Docker, source code dapat dipasang sebagai **read-only**, sedangkan seluruh data runtime ditempatkan pada direktori terpisah yang lebih mudah dikelola.

Pendekatan ini tidak hanya meningkatkan keamanan, tetapi juga mempermudah proses backup, restore, update aplikasi, dan pemeliharaan jangka panjang.

Pada artikel berikutnya kita akan membahas bagaimana menghubungkan aplikasi CodeIgniter 4 yang berjalan di dalam Docker dengan MariaDB yang berjalan di host menggunakan koneksi TCP secara aman dan efisien.
