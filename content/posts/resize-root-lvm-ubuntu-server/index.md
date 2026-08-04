---
title: "Cara Menambah Kapasitas Root Partition Ubuntu Server Menggunakan LVM Tanpa Downtime"
description: "Panduan langkah demi langkah memperbesar kapasitas root (/) Ubuntu Server dengan menambahkan hard disk baru ke LVM tanpa menghapus data dan tanpa reboot."
slug: resize-root-lvm-ubuntu-server
date: 2026-08-04
draft: false
tags:
  - Ubuntu
  - Linux
  - LVM
  - Storage
  - Server
  - DevOps
categories:
  - Linux
series:
  - Administrasi Linux
weight: 5
cover:
  image: "cover.png"
  alt: "Resize LVM Ubuntu Server"
  caption: "Menambah kapasitas root filesystem Ubuntu menggunakan LVM"
---

# Cara Menambah Kapasitas Root Partition Ubuntu Server Menggunakan LVM Tanpa Downtime

Salah satu kelebihan **Logical Volume Manager (LVM)** adalah kemampuannya untuk memperbesar kapasitas penyimpanan tanpa perlu melakukan reinstall sistem operasi ataupun memindahkan data.

Pada artikel ini kita akan menambahkan sebuah hard disk baru ke dalam **Volume Group (VG)** sehingga kapasitas partisi root (`/`) bertambah tanpa downtime.

## Topologi Awal

Misalkan kondisi server sebagai berikut.

| Disk | Ukuran | Keterangan |
|------|---------|------------|
| `/dev/sda` | 100 GB | Digunakan sistem operasi |
| `/dev/sdb` | 250 GB | Hard disk baru |

Kondisi awal:

```
/dev/sda
 ├── EFI
 ├── /boot
 └── LVM
      └── /
```

Target akhir:

```
/dev/sda
 └── LVM
         \
          +------------+
                       |
/dev/sdb --------------+
                       |
                  Volume Group
                       |
                 Logical Volume
                       |
                       /
```

---

# Langkah 1 — Periksa Hard Disk

Pastikan disk baru sudah dikenali oleh sistem.

```bash
lsblk
```

Contoh:

```text
NAME                      SIZE TYPE MOUNTPOINT
sda                       100G disk
├─sda1                      1G part /boot/efi
├─sda2                      2G part /boot
└─sda3                   96.9G part
  └─ubuntu--vg-ubuntu--lv
                         96.9G lvm  /

sdb                       250G disk
```

---

# Langkah 2 — Pastikan Disk Masih Kosong

```bash
sudo fdisk -l /dev/sdb
```

Apabila belum terdapat partisi, disk siap digunakan.

---

# Langkah 3 — Membuat Physical Volume (PV)

Karena seluruh hard disk akan digunakan untuk LVM, langsung buat Physical Volume.

```bash
sudo pvcreate /dev/sdb
```

Output:

```text
Physical volume "/dev/sdb" successfully created.
```

Verifikasi:

```bash
sudo pvs
```

Contoh:

```text
PV         VG        PSize   PFree
/dev/sda3  ubuntu-vg <96.95g      0
/dev/sdb             250.00g 250.00g
```

---

# Langkah 4 — Menambahkan Disk ke Volume Group

Lihat nama Volume Group.

```bash
sudo vgs
```

Contoh:

```text
VG         VSize   VFree
ubuntu-vg  96.95g     0
```

Tambahkan disk baru.

```bash
sudo vgextend ubuntu-vg /dev/sdb
```

Output:

```text
Volume group "ubuntu-vg" successfully extended
```

Verifikasi kembali.

```bash
sudo vgs
```

Contoh:

```text
VG         VSize    VFree
ubuntu-vg 346.94g 250.00g
```

---

# Langkah 5 — Perbesar Logical Volume

Lihat nama Logical Volume.

```bash
sudo lvs
```

Contoh:

```text
LV         VG
ubuntu-lv  ubuntu-vg
```

Gunakan seluruh ruang kosong.

```bash
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
```

Output:

```text
Logical volume ubuntu-vg/ubuntu-lv successfully resized.
```

---

# Langkah 6 — Perbesar Filesystem

Periksa jenis filesystem.

```bash
df -Th /
```

Misalnya:

```text
Filesystem Type
ext4
```

Karena menggunakan **ext4**, jalankan:

```bash
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
```

Output:

```text
Filesystem on /dev/ubuntu-vg/ubuntu-lv is now ...
```

Apabila menggunakan **XFS**, gunakan:

```bash
sudo xfs_growfs /
```

---

# Langkah 7 — Verifikasi

Periksa kapasitas filesystem.

```bash
df -h /
```

Contoh hasil:

```text
Filesystem                         Size  Used Avail Use%
/dev/mapper/ubuntu--vg-ubuntu--lv  341G   87G  241G  27%
```

---

# Verifikasi Struktur LVM

Lihat struktur storage.

```bash
lsblk
```

Periksa status LVM.

```bash
sudo pvs
sudo vgs
sudo lvs
```

---

# Ringkasan Perintah

```bash
sudo pvcreate /dev/sdb

sudo vgextend ubuntu-vg /dev/sdb

sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv

sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
```

---

# Hal yang Perlu Diperhatikan

- Pastikan hard disk yang digunakan benar-benar kosong.
- Lakukan backup sebelum melakukan perubahan pada storage.
- Logical Volume yang menggunakan lebih dari satu hard disk akan bergantung pada seluruh disk tersebut. Apabila salah satu disk mengalami kerusakan, data pada Logical Volume dapat ikut terdampak.
- Pastikan monitoring kapasitas penyimpanan berjalan dengan baik agar penggunaan disk dapat dipantau sejak dini.

---

# Kesimpulan

Dengan memanfaatkan **Logical Volume Manager (LVM)**, administrator dapat memperbesar kapasitas root filesystem tanpa melakukan reinstall sistem operasi maupun memindahkan data. Seluruh proses dapat dilakukan secara **online** sehingga layanan tetap berjalan selama proses resize berlangsung.

Pendekatan ini sangat cocok digunakan pada server produksi yang membutuhkan fleksibilitas dalam pengelolaan kapasitas penyimpanan.