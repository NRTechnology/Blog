---
title: "Cara Install Docker di Ubuntu Server"
date: 2026-07-18
draft: false
description: "Panduan lengkap instalasi Docker Engine dan Docker Compose di Ubuntu Server, mulai dari persiapan sistem, konfigurasi repository resmi Docker, hingga pengujian instalasi untuk lingkungan development maupun production."

tags:
  - Docker
  - Docker Engine
  - Docker Compose
  - Ubuntu
  - Ubuntu Server
  - Linux
  - Container
  - DevOps
  - Server
  - Virtualisasi
  - Self Hosting
  - Docker CLI

categories:
  - Linux
  - Docker

series:
  - "Belajar Docker"

weight: 1

author: "NR Technology"

cover:
  image: "ubuntudocker-cover.png"
  alt: "Cara Install Docker Engine dan Docker Compose di Ubuntu Server"
  caption: "Panduan instalasi Docker Engine dan Docker Compose di Ubuntu Server untuk membangun lingkungan container yang aman, ringan, dan siap digunakan."
---

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

---
{{< saweria >}}