+++
title = 'Cara Install Docker di Ubuntu Server'
date = 2026-07-18T12:00:00+07:00
draft = false
categories = ['Linux']
tags = ['Ubuntu', 'Docker', 'Server']
+++

Docker adalah platform container yang memungkinkan kita menjalankan aplikasi dalam lingkungan yang terisolasi.

## Persiapan

Pastikan sistem Ubuntu sudah diperbarui.

```bash
sudo apt update
sudo apt upgrade -y
```

## Install Docker

Install Docker:

```bash
sudo apt install docker.io
```

Cek versi Docker:

```bash
docker --version
```

## Menjalankan Docker

Aktifkan service Docker:

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Cek status Docker:

```bash
sudo systemctl status docker
```

## Kesimpulan

Docker telah berhasil diinstal dan siap digunakan.