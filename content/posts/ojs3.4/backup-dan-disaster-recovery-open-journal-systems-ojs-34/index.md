---
title: "Backup dan Restore Open Journal Systems (OJS) 3.4"
date: 2026-08-02
draft: false
description: "Panduan lengkap strategi backup dan restore Open Journal Systems (OJS) 3.4 untuk menjaga ketersediaan data dan mendukung proses disaster recovery."
tags:
  - OJS
  - Open Journal Systems
  - Backup
  - Restore
  - Disaster Recovery
  - MariaDB
  - Linux
categories:
  - OJS
  - Backup
  - Linux
series:
  - "Membangun Open Journal Systems (OJS) 3.4"

weight: 7

author: "NR Technology"
---

# Backup dan Restore Open Journal Systems (OJS) 3.4

## Pendahuluan

Data merupakan aset terpenting pada sebuah sistem jurnal elektronik. Artikel ilmiah, hasil review, metadata, informasi pengguna, hingga konfigurasi aplikasi merupakan data yang harus dijaga keberadaannya.

Gangguan terhadap server dapat disebabkan oleh berbagai faktor, seperti.

- Kerusakan perangkat keras.
- Kesalahan konfigurasi.
- Kegagalan pembaruan sistem.
- Human error.
- Malware.
- Ransomware.
- Defacement.
- Kegagalan media penyimpanan.
- Bencana alam.

Tanpa strategi backup yang baik, seluruh data tersebut dapat hilang secara permanen.

Karena itu, backup harus menjadi bagian dari prosedur operasional standar dan bukan hanya dilakukan ketika terjadi masalah.

---

# Tujuan Backup

Strategi backup bertujuan untuk memastikan data dapat dipulihkan apabila terjadi kegagalan sistem.

Beberapa tujuan utama backup antara lain.

- Melindungi data jurnal.
- Mempercepat proses pemulihan layanan.
- Mengurangi kehilangan data.
- Mendukung proses migrasi server.
- Mendukung proses upgrade aplikasi.
- Mendukung disaster recovery.
- Memenuhi kebutuhan audit dan tata kelola.

Backup yang baik tidak hanya menghasilkan salinan data, tetapi juga memastikan data tersebut dapat dipulihkan dengan benar.

---

# Memahami Backup dan Restore

Backup merupakan proses membuat salinan data.

Restore merupakan proses mengembalikan data dari backup ke sistem.

Kedua proses tersebut merupakan satu kesatuan.

```text
Data Produksi
      │
      ▼
   Backup
      │
      ▼
 Media Penyimpanan
      │
      ▼
   Restore
      │
      ▼
 Sistem Pulih
```

Sebuah backup baru dapat dianggap berhasil apabila telah diuji melalui proses restore.

---

# Komponen Open Journal Systems

Sebelum menyusun strategi backup, administrator harus memahami komponen yang membentuk Open Journal Systems.

Secara umum terdiri atas.

```text
Open Journal Systems

├── Database
├── Source Code
├── config.inc.php
├── ojsdata
├── public
├── Plugin
└── Theme
```

Setiap komponen memiliki fungsi yang berbeda dan memerlukan perlakuan backup yang berbeda pula.

---

# Komponen yang Harus Dibackup

Pada implementasi produksi, seluruh komponen berikut harus dicadangkan.

## Database

Database menyimpan.

- Data pengguna.
- Data jurnal.
- Metadata artikel.
- Workflow editorial.
- Konfigurasi aplikasi.
- Riwayat aktivitas.

Apabila database hilang, sebagian besar informasi Open Journal Systems tidak dapat dipulihkan.

---

## Direktori ojsdata

Direktori `ojsdata` berisi seluruh dokumen yang diunggah pengguna.

Contohnya.

- Artikel PDF.
- Supplementary Files.
- Dataset.
- Cover Journal.
- Lampiran.

Backup database tanpa `ojsdata` akan menghasilkan sistem yang kehilangan seluruh dokumen jurnal.

---

## File Konfigurasi

File.

```text
config.inc.php
```

berisi konfigurasi utama aplikasi.

Di dalamnya terdapat.

- Base URL.
- Konfigurasi Database.
- Lokasi Upload.
- Pengaturan Session.
- Pengaturan Email.

Backup file konfigurasi akan mempercepat proses pemulihan apabila server harus dibangun kembali.

---

## Source Code

Source code OJS juga perlu dicadangkan, terutama apabila terdapat.

- Modifikasi aplikasi.
- Plugin tambahan.
- Theme khusus.
- Patch internal.

Apabila source code berasal dari paket resmi dan tidak pernah dimodifikasi, backup source code bukan merupakan prioritas utama. Namun pada lingkungan produksi tetap disarankan untuk menyimpan salinan versi yang sedang digunakan.

---

## Plugin dan Theme

Plugin maupun theme tambahan sering kali dikembangkan secara terpisah dari OJS.

Komponen tersebut sebaiknya ikut dimasukkan ke dalam strategi backup.

Dengan demikian seluruh fungsi aplikasi dapat dipulihkan tanpa harus melakukan konfigurasi ulang.

---

# Arsitektur Backup

Pada seri artikel ini digunakan arsitektur sebagai berikut.

```text
                    OJS Server
                        │
        ┌───────────────┼────────────────┐
        │               │                │
        ▼               ▼                ▼
   Database        config.inc.php     ojsdata
        │               │                │
        └───────────────┼────────────────┘
                        ▼
                  Backup Server
```

Seluruh komponen dicadangkan secara terpisah sehingga proses restore dapat dilakukan dengan lebih fleksibel.

---

# Strategi Backup

Backup yang baik tidak hanya dilakukan sekali.

Administrator sebaiknya memiliki beberapa jenis backup.

## Full Backup

Full Backup merupakan salinan lengkap seluruh komponen.

Meliputi.

- Database.
- ojsdata.
- config.inc.php.
- Plugin.
- Theme.
- Source Code.

Full Backup biasanya dilakukan secara berkala, misalnya mingguan atau bulanan.

---

## Incremental Backup

Incremental Backup hanya menyimpan perubahan sejak backup sebelumnya.

Keuntungannya.

- Ukuran backup lebih kecil.
- Proses lebih cepat.
- Menghemat media penyimpanan.

Strategi ini umumnya digunakan pada lingkungan dengan volume data yang besar.

---

## Differential Backup

Differential Backup menyimpan seluruh perubahan sejak Full Backup terakhir.

Metode ini menghasilkan ukuran backup yang lebih besar dibandingkan Incremental Backup, namun proses restore menjadi lebih sederhana.

---

# Menentukan Frekuensi Backup

Frekuensi backup bergantung pada aktivitas jurnal.

Sebagai contoh.

| Aktivitas Jurnal | Frekuensi Backup |
|------------------|------------------:|
| Sangat Tinggi | Setiap Hari |
| Tinggi | Setiap 2–3 Hari |
| Sedang | Mingguan |
| Rendah | Bulanan |

Semakin tinggi aktivitas jurnal, semakin sering backup perlu dilakukan.

---

# Memahami RPO

Recovery Point Objective (RPO) merupakan jumlah maksimum data yang masih dapat ditoleransi untuk hilang ketika terjadi insiden.

Sebagai contoh.

Apabila backup dilakukan setiap malam.

```text
23.00
```

dan server mengalami kerusakan pada pukul.

```text
16.00
```

maka data yang dibuat setelah backup terakhir berpotensi hilang.

Semakin kecil nilai RPO, semakin kecil pula kehilangan data.

---

# Memahami RTO

Recovery Time Objective (RTO) merupakan waktu yang dibutuhkan untuk memulihkan layanan.

Sebagai contoh.

```text
Target RTO

2 Jam
```

Artinya administrator harus mampu mengembalikan Open Journal Systems hingga kembali beroperasi dalam waktu maksimal dua jam setelah terjadi gangguan.

Nilai RTO dipengaruhi oleh.

- Prosedur restore.
- Kapasitas server.
- Ukuran database.
- Ukuran direktori `ojsdata`.
- Pengalaman administrator.

---

# Prinsip 3-2-1 Backup

Salah satu praktik terbaik dalam pengelolaan backup adalah menerapkan prinsip **3-2-1**.

Prinsip tersebut terdiri atas.

- Memiliki minimal tiga salinan data.
- Menggunakan minimal dua media penyimpanan yang berbeda.
- Menyimpan minimal satu salinan pada lokasi yang berbeda.

Contoh implementasi.

```text
Server Produksi
        │
        ├── Backup Lokal
        │
        ├── NAS
        │
        └── Backup Offsite
```

Pendekatan ini membantu mengurangi risiko kehilangan data akibat kerusakan perangkat maupun bencana.

---

# Ruang Lingkup Artikel

Pada artikel ini akan dibahas.

- Backup Database.
- Backup `config.inc.php`.
- Backup `ojsdata`.
- Backup Source Code.
- Backup Plugin.
- Backup Theme.
- Verifikasi Backup.
- Otomasi Backup.
- Restore Database.
- Restore Direktori Upload.
- Restore Server.
- Disaster Recovery.
- Best Practices.

Seluruh contoh menggunakan placeholder sehingga dapat disesuaikan dengan lingkungan masing-masing tanpa membawa informasi sensitif dari server produksi.

Pada bagian berikutnya kita akan mulai membahas proses backup terhadap setiap komponen Open Journal Systems secara terpisah, dimulai dari database, file konfigurasi, direktori `ojsdata`, source code, plugin, dan theme.

---

# Backup Database

Database merupakan komponen terpenting pada Open Journal Systems.

Seluruh informasi berikut disimpan di dalam database.

- Data jurnal.
- Data pengguna.
- Metadata artikel.
- Workflow editorial.
- Reviewer.
- Editor.
- Konfigurasi aplikasi.
- Statistik.
- Log aktivitas.

Apabila database hilang, sebagian besar informasi Open Journal Systems tidak dapat dipulihkan.

Karena itu database harus menjadi prioritas utama dalam strategi backup.

---

# Memastikan Database Dapat Diakses

Sebelum melakukan backup, pastikan database dapat diakses menggunakan akun yang memiliki hak baca.

Masuk ke MariaDB.

```bash
mysql -u ojs_user -p db_ojs
```

Apabila login berhasil, keluar kembali.

```sql
EXIT;
```

Langkah sederhana ini memastikan proses backup tidak gagal akibat kesalahan autentikasi.

---

# Melakukan Full Backup Database

Backup penuh dilakukan menggunakan utilitas `mysqldump`.

Contoh.

```bash
mysqldump \
--single-transaction \
--routines \
--triggers \
db_ojs \
> backup-db_ojs.sql
```

Parameter yang digunakan memiliki fungsi sebagai berikut.

| Parameter | Fungsi |
|-----------|---------|
| `--single-transaction` | Menjaga konsistensi backup tanpa mengunci tabel InnoDB |
| `--routines` | Membackup Stored Procedure dan Function apabila ada |
| `--triggers` | Membackup Trigger apabila ada |

Walaupun OJS umumnya tidak menggunakan Trigger maupun Stored Procedure, penggunaan parameter tersebut membuat backup lebih lengkap.

---

# Mengompresi Backup Database

Backup database dapat dikompresi untuk menghemat ruang penyimpanan.

Contoh.

```bash
mysqldump \
--single-transaction \
db_ojs \
| gzip \
> backup-db_ojs.sql.gz
```

Metode ini menghasilkan ukuran file yang jauh lebih kecil dibandingkan file SQL biasa.

---

# Memverifikasi File Backup

Setelah proses backup selesai, pastikan file berhasil dibuat.

```bash
ls -lh backup-db_ojs.sql
```

atau.

```bash
ls -lh backup-db_ojs.sql.gz
```

Periksa ukuran file.

Ukuran file yang sangat kecil biasanya mengindikasikan proses backup gagal atau database kosong.

---

# Menguji Isi Backup

Backup yang baik tidak hanya dibuat, tetapi juga diperiksa.

Apabila menggunakan file SQL.

```bash
head backup-db_ojs.sql
```

Contoh.

```text
-- MariaDB dump
--
-- Host:
-- Database: db_ojs
```

Kemudian lihat bagian akhir.

```bash
tail backup-db_ojs.sql
```

Pastikan file tidak terpotong.

---

# Backup File Konfigurasi

File konfigurasi utama OJS adalah.

```text
config.inc.php
```

File tersebut berisi.

- Base URL.
- Database.
- Direktori Upload.
- Pengaturan Session.
- Email.
- Konfigurasi aplikasi.

Backup dilakukan menggunakan.

```bash
cp \
/var/apps/ojs/htdocs/config.inc.php \
backup-config.inc.php
```

Selalu lakukan backup sebelum melakukan perubahan konfigurasi.

---

# Memverifikasi Backup Konfigurasi

Periksa file.

```bash
ls -lh backup-config.inc.php
```

Kemudian bandingkan dengan file asli.

```bash
diff \
backup-config.inc.php \
/var/apps/ojs/htdocs/config.inc.php
```

Apabila tidak terdapat perbedaan, proses backup berhasil.

---

# Backup Direktori Upload

Direktori upload berisi seluruh dokumen yang diunggah pengguna.

Contohnya.

- Artikel PDF.
- Lampiran.
- Dataset.
- Cover Journal.
- Gambar.

Direktori tersebut umumnya berada pada.

```text
/var/apps/ojs/data/ojsdata
```

---

# Membuat Backup ojsdata

Gunakan utilitas `tar`.

```bash
tar \
-czf \
backup-ojsdata.tar.gz \
/var/apps/ojs/data/ojsdata
```

Parameter.

```text
-c
```

membuat arsip baru.

```text
-z
```

mengaktifkan kompresi gzip.

```text
-f
```

menentukan nama file arsip.

---

# Memverifikasi Backup ojsdata

Periksa ukuran file.

```bash
ls -lh backup-ojsdata.tar.gz
```

Kemudian lihat isi arsip.

```bash
tar -tf \
backup-ojsdata.tar.gz
```

Pastikan struktur direktori sesuai dengan direktori asli.

---

# Backup Source Code

Apabila source code tidak pernah dimodifikasi, administrator dapat mengunduh ulang paket resmi OJS.

Namun apabila terdapat.

- Patch internal.
- Modifikasi aplikasi.
- Penyesuaian tampilan.
- Integrasi tambahan.

Source code juga harus dicadangkan.

---

# Membuat Backup Source Code

Gunakan.

```bash
tar \
-czf \
backup-source.tar.gz \
/var/apps/ojs/htdocs
```

File tersebut dapat digunakan untuk mempercepat proses pemulihan server.

---

# Backup Plugin

Plugin tambahan sering kali dikembangkan secara terpisah dari OJS.

Apabila menggunakan plugin pihak ketiga, lakukan backup.

```bash
tar \
-czf \
backup-plugin.tar.gz \
/var/apps/ojs/htdocs/plugins
```

Backup plugin menghindari kebutuhan mengunduh ulang maupun melakukan konfigurasi ulang ketika proses restore.

---

# Backup Theme

Apabila menggunakan theme khusus.

Lakukan backup.

```bash
tar \
-czf \
backup-theme.tar.gz \
/var/apps/ojs/htdocs/plugins/themes
```

Langkah ini sangat penting apabila theme telah dimodifikasi.

---

# Menyusun Struktur Backup

Agar mudah dikelola, gunakan struktur direktori seperti berikut.

```text
backup/

├── database
│   └── backup-db_ojs.sql.gz
│
├── config
│   └── config.inc.php
│
├── upload
│   └── backup-ojsdata.tar.gz
│
├── source
│   └── backup-source.tar.gz
│
├── plugin
│   └── backup-plugin.tar.gz
│
└── theme
    └── backup-theme.tar.gz
```

Pemisahan tersebut mempermudah proses pencarian file backup maupun proses restore.

---

# Melakukan Verifikasi Backup

Backup tidak boleh langsung dianggap berhasil setelah file dibuat.

Lakukan pemeriksaan terhadap.

- Ukuran file.
- Tanggal pembuatan.
- Isi arsip.
- Integritas file.

Sebagai contoh.

```bash
gzip -t backup-db_ojs.sql.gz
```

dan.

```bash
tar -tf backup-ojsdata.tar.gz
```

Apabila kedua perintah tersebut berhasil dijalankan tanpa error, file backup dapat dianggap valid.

---

# Mendokumentasikan Backup

Setiap backup sebaiknya disertai dokumentasi.

Minimal informasi berikut.

- Tanggal Backup.
- Waktu Backup.
- Versi OJS.
- Versi Database.
- Nama Server.
- Administrator yang melakukan backup.

Dokumentasi tersebut sangat membantu ketika organisasi memiliki banyak server OJS maupun ketika melakukan proses audit.

---

# Ringkasan

Pada bagian ini telah dibahas proses backup terhadap seluruh komponen utama Open Journal Systems.

- Backup Database.
- Backup `config.inc.php`.
- Backup `ojsdata`.
- Backup Source Code.
- Backup Plugin.
- Backup Theme.
- Verifikasi Backup.
- Dokumentasi Backup.

Dengan mencadangkan seluruh komponen tersebut, administrator memiliki fondasi yang kuat untuk melakukan pemulihan sistem apabila terjadi kegagalan maupun ketika melakukan migrasi server.

Pada bagian berikutnya kita akan membahas otomasi backup menggunakan Bash Script, penjadwalan dengan Cron, rotasi backup, kompresi, backup offsite, serta praktik terbaik dalam mengelola arsip backup pada lingkungan produksi.

---

# Mengotomatisasi Backup

Melakukan backup secara manual hanya cocok untuk kebutuhan sesekali, misalnya sebelum melakukan upgrade atau perubahan konfigurasi.

Pada lingkungan produksi, backup sebaiknya dilakukan secara otomatis sehingga administrator tidak bergantung pada proses manual.

Otomasi backup memberikan beberapa keuntungan.

- Backup dilakukan secara konsisten.
- Mengurangi human error.
- Memastikan backup tetap berjalan meskipun administrator tidak sedang melakukan pemeliharaan.
- Memudahkan penerapan kebijakan retensi backup.

---

# Menyiapkan Direktori Backup

Buat direktori khusus untuk menyimpan seluruh file backup.

```bash
mkdir -p /var/apps/ojs/backup
```

Selanjutnya buat struktur direktori.

```bash
mkdir -p /var/apps/ojs/backup/{database,config,upload,source,logs}
```

Periksa hasilnya.

```bash
tree /var/apps/ojs/backup
```

Contoh.

```text
backup

├── config
├── database
├── logs
├── source
└── upload
```

Pemisahan direktori mempermudah pengelolaan backup.

---

# Penamaan File Backup

Gunakan format penamaan yang konsisten.

Contohnya.

```text
backup-db_ojs-20260802-010000.sql.gz
```

```text
backup-ojsdata-20260802-010000.tar.gz
```

Format tersebut memudahkan administrator mengetahui waktu pembuatan backup tanpa harus membuka isi file.

---

# Menggunakan Timestamp

Timestamp dapat dibuat menggunakan.

```bash
date +"%Y%m%d-%H%M%S"
```

Contoh hasil.

```text
20260802-010000
```

Timestamp tersebut dapat digunakan sebagai bagian dari nama file backup.

---

# Membuat Script Backup

Buat direktori untuk script.

```bash
mkdir -p /opt/scripts
```

Kemudian buat file.

```bash
nano /opt/scripts/ojs-backup.sh
```

Contoh isi script.

```bash
#!/bin/bash

BACKUP_DIR="/var/apps/ojs/backup"

DATE=$(date +"%Y%m%d-%H%M%S")

mkdir -p ${BACKUP_DIR}/database
mkdir -p ${BACKUP_DIR}/config
mkdir -p ${BACKUP_DIR}/upload

mysqldump \
--single-transaction \
db_ojs \
| gzip \
> ${BACKUP_DIR}/database/db-${DATE}.sql.gz

cp \
/var/apps/ojs/htdocs/config.inc.php \
${BACKUP_DIR}/config/config-${DATE}.inc.php

tar \
-czf \
${BACKUP_DIR}/upload/ojsdata-${DATE}.tar.gz \
/var/apps/ojs/data/ojsdata
```

Script tersebut melakukan backup terhadap tiga komponen utama.

- Database.
- File konfigurasi.
- Direktori upload.

---

# Memberikan Hak Eksekusi

Ubah permission script.

```bash
chmod +x \
/opt/scripts/ojs-backup.sh
```

Selanjutnya lakukan pengujian.

```bash
/opt/scripts/ojs-backup.sh
```

Pastikan seluruh file backup berhasil dibuat.

---

# Menjadwalkan Backup Menggunakan Cron

Backup otomatis dapat dijalankan menggunakan Cron.

Buka konfigurasi Cron.

```bash
crontab -e
```

Tambahkan.

```cron
0 1 * * * /opt/scripts/ojs-backup.sh
```

Konfigurasi tersebut menjalankan backup setiap hari pukul.

```text
01:00
```

Administrator dapat menyesuaikan jadwal sesuai kebutuhan operasional.

---

# Memisahkan Jadwal Backup

Pada lingkungan dengan aktivitas tinggi, administrator dapat memisahkan jadwal backup.

Contohnya.

| Komponen | Jadwal |
|----------|---------|
| Database | Harian |
| config.inc.php | Harian |
| ojsdata | Harian |
| Source Code | Mingguan |
| Plugin | Mingguan |
| Theme | Mingguan |

Pendekatan tersebut membantu mengurangi waktu backup sekaligus menjaga efisiensi penggunaan media penyimpanan.

---

# Melakukan Rotasi Backup

Backup yang tidak pernah dihapus akan memenuhi media penyimpanan.

Karena itu diperlukan kebijakan rotasi.

Sebagai contoh.

- Simpan backup harian selama 7 hari.
- Simpan backup mingguan selama 4 minggu.
- Simpan backup bulanan selama 12 bulan.

Kebijakan retensi dapat disesuaikan dengan kebutuhan organisasi.

---

# Menghapus Backup Lama

Backup lama dapat dihapus secara otomatis.

Contoh.

```bash
find \
/var/apps/ojs/backup \
-type f \
-mtime +30 \
-delete
```

Perintah tersebut menghapus file yang berumur lebih dari tiga puluh hari.

Sebelum menerapkan kebijakan ini, pastikan retensi telah disepakati oleh organisasi.

---

# Menggunakan Kompresi

Sebagian besar file backup dapat dikompresi.

Contohnya.

- SQL
- TAR
- Log

Gunakan.

```text
gzip
```

atau.

```text
xz
```

Kompresi membantu menghemat kapasitas media penyimpanan terutama apabila database dan direktori upload berukuran besar.

---

# Memisahkan Backup dari Server Produksi

Backup sebaiknya tidak hanya disimpan pada server produksi.

Apabila server mengalami kerusakan fisik, seluruh backup juga berpotensi hilang.

Minimal gunakan dua lokasi penyimpanan.

```text
Server Produksi

↓

NAS

↓

Backup Offsite
```

Pemisahan lokasi backup merupakan bagian dari penerapan prinsip 3-2-1 Backup.

---

# Backup Offsite

Backup Offsite merupakan salinan yang disimpan di lokasi berbeda.

Contohnya.

- Data Center lain.
- NAS pada lokasi berbeda.
- Penyimpanan objek (Object Storage).
- Penyimpanan arsip organisasi.

Tujuannya adalah melindungi data dari risiko.

- Kebakaran.
- Banjir.
- Pencurian.
- Kerusakan perangkat.
- Ransomware.

---

# Enkripsi Backup

Backup sering kali berisi informasi sensitif.

Apabila backup dipindahkan ke media eksternal atau lokasi lain, administrator sebaiknya mempertimbangkan penggunaan enkripsi.

Enkripsi membantu memastikan isi backup tidak dapat dibaca apabila media penyimpanan hilang atau jatuh ke tangan pihak yang tidak berwenang.

Pengelolaan kunci enkripsi harus dilakukan secara terpisah dari media backup.

---

# Menguji Backup Secara Berkala

Backup yang tidak pernah diuji belum dapat dianggap andal.

Lakukan pengujian secara berkala.

- Pastikan file dapat dibuka.
- Pastikan arsip dapat diekstrak.
- Pastikan file SQL dapat dibaca.
- Pastikan ukuran file sesuai.

Selanjutnya lakukan pengujian restore pada lingkungan pengujian.

Pengujian ini memastikan backup benar-benar dapat digunakan ketika terjadi insiden.

---

# Monitoring Proses Backup

Administrator sebaiknya memantau setiap proses backup.

Minimal periksa.

- Waktu mulai.
- Waktu selesai.
- Ukuran file.
- Status berhasil atau gagal.
- Pesan kesalahan apabila ada.

Catatan tersebut sangat membantu ketika melakukan audit maupun investigasi kegagalan backup.

---

# Ringkasan

Pada bagian ini telah dibahas otomasi proses backup menggunakan Bash Script dan Cron, strategi penamaan file, rotasi backup, kompresi, backup offsite, serta pengujian hasil backup.

Dengan menerapkan otomasi dan kebijakan retensi yang tepat, proses backup menjadi lebih konsisten, mudah diaudit, dan siap digunakan ketika diperlukan.

Pada bagian berikutnya kita akan membahas proses **restore**, mulai dari pemulihan database, direktori `ojsdata`, file konfigurasi, source code, hingga validasi aplikasi setelah proses restore selesai.

---

# Restore Open Journal Systems

Restore merupakan proses mengembalikan Open Journal Systems menggunakan data hasil backup.

Restore dapat dilakukan pada berbagai kondisi.

- Kerusakan sistem operasi.
- Kerusakan database.
- Kegagalan upgrade.
- Migrasi server.
- Penggantian perangkat keras.
- Pemulihan setelah insiden keamanan.

Tujuan utama restore adalah mengembalikan layanan hingga dapat digunakan kembali dengan kehilangan data seminimal mungkin.

---

# Komponen yang Direstore

Proses restore harus memperhatikan seluruh komponen Open Journal Systems.

```text
Open Journal Systems

├── Database
├── config.inc.php
├── ojsdata
├── Source Code
├── Plugin
└── Theme
```

Restore hanya terhadap salah satu komponen sering kali menghasilkan aplikasi yang tidak dapat berjalan dengan benar.

Sebagai contoh.

- Database baru tanpa `ojsdata` menyebabkan artikel tidak dapat diunduh.
- `ojsdata` tanpa database menyebabkan metadata artikel hilang.
- Source code yang berbeda versi dengan database dapat menyebabkan aplikasi gagal dijalankan.

---

# Persiapan Restore

Sebelum melakukan restore, lakukan beberapa langkah berikut.

- Pastikan seluruh file backup tersedia.
- Pastikan media penyimpanan backup dapat diakses.
- Pastikan ruang penyimpanan server mencukupi.
- Pastikan versi Open Journal Systems sesuai.
- Pastikan versi database kompatibel.

Restore tidak boleh dilakukan menggunakan file backup yang belum pernah diverifikasi.

---

# Menghentikan Layanan

Untuk menjaga konsistensi data, hentikan akses pengguna selama proses restore.

Apabila menggunakan Docker PHP-FPM.

```bash
docker stop ojs-php
```

Kemudian hentikan web server apabila diperlukan.

```bash
systemctl stop nginx
```

Dengan demikian tidak terdapat perubahan data selama proses restore berlangsung.

---

# Restore Database

Masuk ke MariaDB.

```bash
mysql -u root -p
```

Apabila database lama masih tersedia dan akan diganti seluruhnya.

```sql
DROP DATABASE db_ojs;
```

Buat kembali database.

```sql
CREATE DATABASE db_ojs
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;
```

Keluar dari MariaDB.

```sql
EXIT;
```

---

# Mengembalikan Database

Apabila menggunakan file SQL.

```bash
mysql \
-u root \
-p \
db_ojs \
< backup-db_ojs.sql
```

Apabila menggunakan file terkompresi.

```bash
gunzip -c \
backup-db_ojs.sql.gz \
| mysql \
-u root \
-p \
db_ojs
```

Tunggu hingga proses selesai.

Lamanya proses bergantung pada ukuran database.

---

# Memverifikasi Database

Masuk ke database.

```bash
mysql \
-u root \
-p
```

Pilih database.

```sql
USE db_ojs;
```

Periksa tabel.

```sql
SHOW TABLES;
```

Pastikan seluruh tabel telah muncul.

Selanjutnya periksa jumlah data.

Contoh.

```sql
SELECT COUNT(*)
FROM users;
```

dan.

```sql
SELECT COUNT(*)
FROM submissions;
```

Langkah ini memastikan data telah berhasil dipulihkan.

---

# Restore config.inc.php

Apabila file konfigurasi ikut dibackup.

Salin kembali file tersebut.

```bash
cp \
backup-config.inc.php \
/var/apps/ojs/htdocs/config.inc.php
```

Selanjutnya atur ownership.

```bash
chown \
www-data:www-data \
/var/apps/ojs/htdocs/config.inc.php
```

Kemudian permission.

```bash
chmod 640 \
/var/apps/ojs/htdocs/config.inc.php
```

---

# Memverifikasi Konfigurasi

Buka file konfigurasi.

```bash
nano \
/var/apps/ojs/htdocs/config.inc.php
```

Pastikan beberapa parameter berikut sesuai.

- Database Host.
- Database Name.
- Database Username.
- Base URL.
- Files Directory.
- Installed.

Apabila server tujuan menggunakan konfigurasi yang berbeda, sesuaikan parameter tersebut sebelum aplikasi dijalankan.

---

# Restore Direktori Upload

Apabila backup menggunakan TAR.

Ekstrak kembali.

```bash
tar \
-xzf \
backup-ojsdata.tar.gz \
-C /
```

Periksa hasilnya.

```bash
ls -lah \
/var/apps/ojs/data/ojsdata
```

Pastikan seluruh file artikel telah kembali.

---

# Mengatur Ownership Direktori Upload

Pastikan owner menggunakan user web server.

```bash
chown -R \
www-data:www-data \
/var/apps/ojs/data/ojsdata
```

Kemudian.

```bash
chmod -R 755 \
/var/apps/ojs/data/ojsdata
```

Ownership yang salah sering menyebabkan proses upload maupun download gagal.

---

# Restore Source Code

Apabila source code ikut dibackup.

Ekstrak kembali.

```bash
tar \
-xzf \
backup-source.tar.gz \
-C /
```

Periksa struktur direktori.

```bash
ls \
/var/apps/ojs/htdocs
```

Pastikan file utama seperti.

```text
index.php
```

dan.

```text
config.inc.php
```

telah tersedia.

---

# Restore Plugin

Apabila terdapat plugin tambahan.

Ekstrak.

```bash
tar \
-xzf \
backup-plugin.tar.gz \
-C /
```

Pastikan plugin berada pada direktori yang benar.

```text
plugins/
```

---

# Restore Theme

Apabila menggunakan theme khusus.

Ekstrak.

```bash
tar \
-xzf \
backup-theme.tar.gz \
-C /
```

Selanjutnya pastikan theme dapat dikenali oleh Open Journal Systems.

---

# Memeriksa Permission

Setelah seluruh file dikembalikan.

Periksa ownership.

```bash
ls -lah \
/var/apps/ojs
```

Pastikan owner.

```text
www-data
```

Kemudian periksa permission.

```bash
find \
/var/apps/ojs/htdocs \
-type f
```

Permission yang tidak sesuai dapat menyebabkan aplikasi gagal berjalan.

---

# Menjalankan Kembali Layanan

Aktifkan kembali PHP.

```bash
docker start ojs-php
```

Kemudian Nginx.

```bash
systemctl start nginx
```

Pastikan seluruh service berjalan normal.

---

# Validasi Restore

Buka browser.

```text
https://jurnal.example.go.id
```

Lakukan pengujian.

- Login Administrator.
- Membuka Dashboard.
- Membuka Artikel.
- Mengunduh PDF.
- Mengunggah Dokumen.
- Membuka Pengaturan.

Seluruh fungsi utama harus berjalan tanpa menghasilkan kesalahan.

---

# Memeriksa Log

Lakukan pemantauan log selama proses validasi.

Log Nginx.

```bash
tail -f \
/var/log/nginx/error.log
```

Log PHP.

```bash
docker logs -f ojs-php
```

Log MariaDB.

```bash
journalctl \
-u mariadb \
-f
```

Pastikan tidak terdapat.

- PHP Fatal Error.
- Database Error.
- Permission Denied.
- File Not Found.

---

# Pengujian Akhir

Sebelum layanan dibuka kembali kepada pengguna.

Pastikan.

- Database berhasil dipulihkan.
- Artikel dapat dibuka.
- PDF dapat diunduh.
- Login berhasil.
- Dashboard dapat diakses.
- Upload berhasil.
- Plugin aktif.
- Theme berjalan dengan baik.
- Tidak terdapat error pada log.

Apabila seluruh pengujian berhasil, proses restore dapat dinyatakan selesai.

---

# Ringkasan

Pada bagian ini telah dibahas proses restore terhadap seluruh komponen utama Open Journal Systems.

- Restore Database.
- Restore `config.inc.php`.
- Restore `ojsdata`.
- Restore Source Code.
- Restore Plugin.
- Restore Theme.
- Validasi Restore.
- Pengujian Pasca Restore.

Restore yang dilakukan secara sistematis membantu memastikan Open Journal Systems dapat kembali beroperasi dengan konfigurasi dan data yang konsisten.

Pada bagian terakhir kita akan membahas **Disaster Recovery**, migrasi server, pemindahan Open Journal Systems ke server baru, checklist pemulihan, serta praktik terbaik untuk menjaga ketersediaan layanan dalam jangka panjang.

---

# Disaster Recovery

Backup merupakan salah satu bagian dari strategi Disaster Recovery (DR).

Disaster Recovery adalah serangkaian prosedur untuk memulihkan layanan setelah terjadi gangguan yang menyebabkan sistem tidak dapat beroperasi.

Gangguan tersebut dapat berupa.

- Kerusakan perangkat keras.
- Kerusakan media penyimpanan.
- Kegagalan sistem operasi.
- Kegagalan pembaruan aplikasi.
- Human Error.
- Serangan Malware.
- Ransomware.
- Defacement.
- Bencana alam.

Tujuan utama Disaster Recovery adalah mengembalikan layanan secepat mungkin dengan kehilangan data seminimal mungkin.

---

# Mempersiapkan Server Pengganti

Salah satu keuntungan menggunakan arsitektur yang telah dibangun pada seri artikel ini adalah proses pemindahan ke server baru menjadi lebih sederhana.

Server baru cukup memiliki komponen berikut.

- Ubuntu Server.
- Nginx.
- Docker Engine.
- Docker Compose.
- PHP-FPM.
- MariaDB.

Setelah seluruh komponen tersebut tersedia, proses restore dapat dilakukan menggunakan file backup.

---

# Migrasi ke Server Baru

Migrasi Open Journal Systems pada dasarnya terdiri atas beberapa tahapan.

```text
Backup Server Lama

↓

Salin Backup

↓

Bangun Server Baru

↓

Restore Database

↓

Restore ojsdata

↓

Restore config.inc.php

↓

Restore Source Code

↓

Validasi

↓

Produksi
```

Dengan pendekatan tersebut, downtime dapat diminimalkan.

---

# Restore Berdasarkan Prioritas

Apabila seluruh komponen harus dipulihkan, lakukan restore dengan urutan berikut.

1. Database.
2. Source Code.
3. Plugin.
4. Theme.
5. config.inc.php.
6. ojsdata.
7. Permission.
8. Validasi.

Urutan tersebut membantu mengurangi kemungkinan terjadinya inkonsistensi konfigurasi.

---

# Melakukan Validasi Setelah Migrasi

Setelah seluruh proses restore selesai, lakukan pengujian terhadap fungsi utama aplikasi.

Pastikan.

- Halaman utama dapat diakses.
- Login berhasil.
- Dashboard muncul.
- Artikel dapat dibuka.
- PDF dapat diunduh.
- Upload berhasil.
- Plugin berjalan.
- Theme tampil dengan benar.

Seluruh pengujian tersebut sebaiknya dilakukan sebelum DNS diarahkan menuju server baru.

---

# Melakukan Pengujian Disaster Recovery

Backup tidak cukup hanya dibuat.

Administrator juga harus melakukan simulasi pemulihan secara berkala.

Contoh.

- Restore pada Virtual Machine.
- Restore pada Server Pengujian.
- Restore pada lingkungan staging.

Melalui simulasi tersebut administrator dapat memastikan bahwa seluruh prosedur pemulihan telah terdokumentasi dengan baik.

---

# Mendokumentasikan Proses Recovery

Setiap proses recovery sebaiknya didokumentasikan.

Minimal informasi berikut.

- Tanggal.
- Penyebab Recovery.
- Jenis Backup.
- Lokasi Backup.
- Waktu Restore.
- Waktu Sistem Kembali Beroperasi.
- Kendala yang Ditemukan.
- Solusi yang Dilakukan.

Dokumentasi tersebut akan sangat membantu apabila terjadi insiden serupa pada masa mendatang.

---

# Menentukan Recovery Point

Setelah proses restore selesai, lakukan pemeriksaan terhadap data terbaru.

Pastikan.

- Artikel terakhir tersedia.
- Submission terakhir tersedia.
- Pengguna terbaru tersedia.
- Konfigurasi terbaru telah dipulihkan.

Langkah ini membantu memastikan bahwa kehilangan data masih berada dalam batas Recovery Point Objective (RPO) yang telah ditetapkan.

---

# Menentukan Recovery Time

Catat waktu yang dibutuhkan selama proses pemulihan.

Sebagai contoh.

| Tahapan | Durasi |
|---------|-------:|
| Restore Database | 15 Menit |
| Restore ojsdata | 20 Menit |
| Restore Source Code | 5 Menit |
| Validasi | 20 Menit |

Informasi tersebut berguna untuk mengevaluasi apakah target Recovery Time Objective (RTO) telah tercapai.

---

# Checklist Recovery

Sebelum layanan dibuka kembali kepada pengguna, pastikan seluruh komponen telah diperiksa.

- Database berhasil direstore.
- config.inc.php sesuai.
- Source Code sesuai versi.
- Plugin aktif.
- Theme aktif.
- Direktori `ojsdata` lengkap.
- Permission benar.
- Ownership benar.
- Login berhasil.
- Upload berhasil.
- Download berhasil.
- Dashboard Administrator berjalan.
- Tidak terdapat error pada log.
- HTTPS berfungsi.
- Scheduled Task berjalan.

Checklist tersebut dapat digunakan sebagai dokumen serah terima setelah proses recovery selesai.

---

# Evaluasi Backup

Setelah proses restore berhasil dilakukan, lakukan evaluasi terhadap strategi backup yang digunakan.

Beberapa pertanyaan yang dapat dijadikan bahan evaluasi.

- Apakah backup selalu berhasil dibuat?
- Apakah backup mudah ditemukan?
- Apakah ukuran backup masih wajar?
- Apakah proses restore berjalan sesuai prosedur?
- Apakah target RPO tercapai?
- Apakah target RTO tercapai?

Evaluasi berkala membantu meningkatkan kualitas strategi backup pada implementasi berikutnya.

---

# Best Practices

Beberapa praktik terbaik yang direkomendasikan dalam pengelolaan backup dan restore Open Journal Systems antara lain.

- Terapkan prinsip 3-2-1 Backup.
- Lakukan backup secara otomatis.
- Simpan backup pada lokasi yang berbeda dari server produksi.
- Enkripsi backup yang berisi data sensitif.
- Lakukan verifikasi integritas setiap file backup.
- Uji proses restore secara berkala.
- Dokumentasikan seluruh prosedur backup dan restore.
- Simpan dokumentasi bersama arsip backup.
- Pantau kapasitas media penyimpanan backup.
- Gunakan penamaan file yang konsisten.
- Terapkan kebijakan retensi backup.
- Hapus backup yang telah melewati masa retensi sesuai kebijakan organisasi.

Penerapan praktik-praktik tersebut membantu memastikan backup tetap dapat digunakan ketika benar-benar dibutuhkan.

---

# Kesimpulan

Backup dan restore merupakan bagian yang tidak terpisahkan dari pengelolaan Open Journal Systems pada lingkungan produksi.

Backup yang dilakukan secara rutin, terdokumentasi, diverifikasi, dan diuji melalui proses restore akan memberikan tingkat kesiapan yang jauh lebih baik ketika terjadi kegagalan sistem.

Pada artikel ini telah dibahas strategi backup terhadap seluruh komponen penting Open Journal Systems, otomasi backup, penyimpanan arsip, proses restore, migrasi server, hingga penerapan prinsip Disaster Recovery.

Dengan menerapkan prosedur tersebut, administrator dapat mengurangi risiko kehilangan data, mempercepat proses pemulihan layanan, serta menjaga ketersediaan sistem jurnal dalam jangka panjang.

---

# Ringkasan

Pada artikel ini telah dibahas.

- Konsep Backup dan Restore.
- Komponen yang Harus Dibackup.
- Strategi Backup.
- Full Backup.
- Incremental Backup.
- Differential Backup.
- Backup Database.
- Backup `config.inc.php`.
- Backup `ojsdata`.
- Backup Source Code.
- Backup Plugin.
- Backup Theme.
- Otomasi Backup.
- Cron Job.
- Rotasi Backup.
- Backup Offsite.
- Enkripsi Backup.
- Restore Database.
- Restore Direktori Upload.
- Restore Source Code.
- Restore Konfigurasi.
- Disaster Recovery.
- Migrasi Server.
- Validasi Restore.
- Best Practices.

Strategi backup yang baik tidak hanya melindungi data dari kehilangan, tetapi juga memastikan layanan dapat dipulihkan dengan cepat dan konsisten ketika terjadi gangguan.

---