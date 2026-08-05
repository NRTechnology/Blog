---
title: "Hardening SSH dengan nftables: Membatasi Akses dan Mencegah SSH Jumping"
date: 2026-07-29
draft: false
description: "Panduan lengkap melakukan hardening SSH pada Ubuntu Server menggunakan nftables untuk membatasi akses, mencegah SSH Jumping, mengurangi risiko lateral movement, dan meningkatkan keamanan server produksi."

tags:
  - SSH
  - OpenSSH
  - nftables
  - Linux Security
  - Server Hardening
  - Firewall
  - Ubuntu Server
  - Cyber Security
  - SSH Jumping
  - Lateral Movement
  - Secure Shell
  - Access Control
  - Network Security

categories:
  - Cyber Security
  - Linux
  - Ubuntu Server

series:
  - "Hardening Linux Server"

weight: 3

author: "NR Technology"

cover:
  image: "sshjumping-cover.png"
  alt: "Hardening SSH menggunakan nftables untuk membatasi akses dan mencegah SSH Jumping pada Ubuntu Server"
  caption: "Panduan implementasi hardening SSH menggunakan nftables untuk membatasi akses, mencegah SSH Jumping, dan meningkatkan keamanan server Linux."
---

SSH (**Secure Shell**) merupakan salah satu layanan yang hampir selalu tersedia pada server Linux. Administrator menggunakannya untuk melakukan remote administration, maintenance, deployment aplikasi, troubleshooting, hingga transfer file.

Kemudahan tersebut sekaligus menjadikan SSH salah satu jalur yang perlu mendapat perhatian serius dalam pengamanan server.

Hardening SSH biasanya berfokus pada bagaimana seseorang **masuk ke server**, misalnya dengan menonaktifkan login root, menggunakan SSH key, membatasi user, atau membatasi alamat IP yang diperbolehkan mengakses port 22.

Namun ada satu hal lain yang tidak kalah penting:

> **Apa yang terjadi jika sebuah server sudah berhasil dikuasai dan kemudian digunakan untuk melakukan SSH ke server lainnya?**

Kondisi tersebut dapat menjadi bagian dari **lateral movement** di dalam jaringan. Server yang awalnya menjadi korban dapat berubah menjadi titik loncat untuk menyerang sistem lain.

Pada artikel ini kita akan melakukan hardening SSH menggunakan **nftables** dengan dua kebijakan utama:

1. SSH inbound hanya diperbolehkan dari jaringan VPN.
2. SSH outbound TCP port 22 dari server diblokir.

Tujuannya bukan hanya mempersempit akses menuju SSH server, tetapi juga mengurangi kemungkinan server digunakan sebagai **SSH jumping point** menuju server lainnya.

---

## 1. Memahami Risiko SSH pada Server

Bayangkan sebuah infrastruktur memiliki beberapa server:

~~~text
Administrator
     |
     | VPN
     v
+-------------+
| VPN Network |
+-------------+
     |
     | SSH
     v
+-------------+
| Web Server  |
+-------------+
     |
     | SSH
     v
+-------------+
| DB Server   |
+-------------+
~~~

Administrator memang hanya dapat mengakses Web Server melalui VPN.

Tetapi apabila Web Server masih diperbolehkan membuat koneksi SSH ke mana saja, server tersebut berpotensi digunakan sebagai batu loncatan menuju sistem lain.

Misalnya seorang penyerang berhasil mendapatkan akses shell pada Web Server melalui:

- webshell;
- vulnerability pada aplikasi;
- credential yang bocor;
- privilege escalation;
- malware atau backdoor;
- konfigurasi aplikasi yang tidak aman.

Setelah memperoleh shell, penyerang dapat mencoba:

~~~bash
ssh user@server-lain
~~~

Jika tersedia credential, SSH key, atau password yang dapat digunakan, penyerang dapat mencoba berpindah ke server berikutnya.

Aktivitas berpindah dari satu sistem yang telah dikuasai menuju sistem lain dikenal sebagai **lateral movement**.

---

## 2. Apa Itu SSH Jumping?

Dalam administrasi sistem, konsep SSH jumping sebenarnya bukan sesuatu yang buruk.

SSH menyediakan mekanisme **Jump Host** atau **Bastion Host**.

Contohnya:

~~~text
Laptop Administrator
        |
        | SSH
        v
+----------------+
|  Bastion Host  |
+----------------+
        |
        | SSH
        v
+-----------------+
| Internal Server |
+-----------------+
~~~

Administrator tidak dapat mengakses Internal Server secara langsung. Semua koneksi harus melewati Bastion Host.

Dengan OpenSSH, mekanisme tersebut dapat dilakukan menggunakan `ProxyJump`, misalnya:

~~~bash
ssh -J admin@bastion.example.com admin@internal-server
~~~

Dalam arsitektur yang dirancang dengan benar, jump host merupakan mekanisme keamanan yang sangat berguna.

Masalah muncul ketika **server biasa yang tidak dirancang sebagai jump host dapat digunakan sebagai titik loncat setelah server tersebut dikompromikan**.

Sebagai contoh:

~~~text
Internet
   |
   | Exploit Web Application
   v
+----------------+
|   Web Server   |  <-- compromised
+----------------+
        |
        | SSH
        v
+-----------------+
| Database Server |
+-----------------+
        |
        | SSH
        v
+----------------+
| Backup Server  |
+----------------+
~~~

Penyerang yang awalnya hanya mendapatkan Web Server sekarang memiliki kemungkinan untuk mencoba menjangkau Database Server, Backup Server, atau server internal lainnya.

Karena itu kita perlu membedakan antara **SSH Jump Host yang memang dirancang secara resmi** dengan **server yang tanpa sengaja dapat digunakan sebagai jumping point oleh attacker**.

Artikel ini membahas pencegahan skenario kedua.

---

## 3. Prinsip Hardening yang Digunakan

Kita akan menerapkan kebijakan berikut:

~~~text
                    VPN
              10.10.10.0/24
                     |
                     | TCP/22
                     v
              +-------------+
              |             |
Internet ---->|   SERVER    |
              |             |
              +-------------+
                     |
                     | TCP/22
                     X
                  BLOCKED
~~~

Kebijakannya adalah:

~~~text
VPN Network -> Server TCP/22      ALLOW
Non-VPN     -> Server TCP/22      REJECT
Server      -> Destination TCP/22 REJECT
~~~

Dengan demikian SSH hanya dapat digunakan untuk **administrasi masuk melalui jaringan VPN yang telah ditentukan**.

Pada saat yang sama server tidak diperbolehkan membuka koneksi menuju layanan SSH pada server lain melalui TCP port 22.

---

## 4. Mengapa Menggunakan nftables?

Pada Linux modern, salah satu framework firewall yang dapat digunakan adalah **nftables**.

Administrasinya dilakukan menggunakan perintah:

~~~bash
nft
~~~

Ruleset dapat dilihat menggunakan:

~~~bash
sudo nft list ruleset
~~~

Dalam konfigurasi artikel ini kita menggunakan sebuah table:

~~~text
inet filter
~~~

dengan dua chain utama:

~~~text
input
output
~~~

Chain `input` digunakan untuk mengatur paket yang **menuju server**.

Chain `output` digunakan untuk mengatur paket yang **berasal dari server**.

Inilah yang memungkinkan kita membuat dua lapisan kontrol SSH sekaligus.

---

## 5. Menentukan Network VPN

Pada script, jaringan VPN didefinisikan melalui:

~~~bash
VPN_NETWORK="10.10.10.0/24"
~~~

Artinya seluruh host dalam network tersebut diperbolehkan mengakses SSH server.

Contohnya:

~~~text
10.10.10.5  -> TCP/22 -> ALLOW
10.10.10.10 -> TCP/22 -> ALLOW
10.10.10.20 -> TCP/22 -> ALLOW
~~~

Sedangkan koneksi SSH dari network lainnya akan ditolak.

Network tersebut harus disesuaikan dengan konfigurasi VPN yang digunakan pada lingkungan masing-masing.

---

## 6. Membuat Table dan Chain nftables

Script terlebih dahulu memastikan table `inet filter` tersedia.

Apabila belum tersedia, table dibuat menggunakan:

~~~bash
nft add table inet filter
~~~

Kemudian dibuat chain `input`:

~~~bash
nft 'add chain inet filter input {
    type filter hook input priority filter;
    policy accept;
}'
~~~

dan chain `output`:

~~~bash
nft 'add chain inet filter output {
    type filter hook output priority filter;
    policy accept;
}'
~~~

Perhatikan bahwa konfigurasi ini menggunakan:

~~~text
policy accept
~~~

Artinya kita **tidak sedang membuat firewall default-deny untuk seluruh trafik server**.

Kita hanya menambahkan pembatasan khusus terhadap SSH.

Pendekatan ini berguna ketika hardening ingin diterapkan secara bertahap tanpa langsung mengubah seluruh kebijakan firewall server.

---

## 7. Mengizinkan SSH dari VPN

Rule pertama digunakan untuk mengizinkan SSH dari jaringan VPN:

~~~bash
nft add rule inet filter input \
    ip saddr "$VPN_NETWORK" \
    tcp dport 22 \
    accept \
    comment \"SSH_FROM_VPN\"
~~~

Secara sederhana rule tersebut berarti:

~~~text
Source IP berada di VPN_NETWORK
            +
Protocol TCP
            +
Destination Port 22
            =
ACCEPT
~~~

Sebagai contoh:

~~~text
10.10.10.20 -> Server:22
~~~

akan diterima karena alamat sumber berada dalam network:

~~~text
10.10.10.0/24
~~~

---

## 8. Memblokir SSH dari Luar VPN

Setelah rule `accept` untuk VPN dibuat, kita menambahkan:

~~~bash
nft add rule inet filter input \
    tcp dport 22 \
    reject \
    comment \"BLOCK_SSH_NON_VPN\"
~~~

Rule ini tidak menentukan source IP.

Artinya koneksi menuju TCP port 22 yang belum diterima oleh rule sebelumnya akan ditolak.

Urutan rule sangat penting:

~~~text
ip saddr 10.10.10.0/24 tcp dport 22 accept
tcp dport 22 reject
~~~

Proses evaluasinya dapat dibayangkan seperti:

~~~text
Incoming TCP/22
      |
      v
Source dari VPN?
   /       \
 YES        NO
  |          |
ACCEPT     REJECT
~~~

Karena itu rule `SSH_FROM_VPN` harus berada **sebelum** `BLOCK_SSH_NON_VPN`.

---

## 9. Memblokir SSH Outbound

Bagian berikutnya merupakan pembatasan yang sering tidak diterapkan pada hardening SSH sederhana.

~~~bash
nft add rule inet filter output \
    tcp dport 22 \
    reject \
    comment \"BLOCK_SSH_OUTBOUND\"
~~~

Rule tersebut berarti:

> Paket keluar dari server menuju TCP destination port 22 akan ditolak.

Contohnya:

~~~text
Web Server -> Server A:22    REJECT
Web Server -> Server B:22    REJECT
Web Server -> Internet:22    REJECT
~~~

Jika seseorang mendapatkan shell pada server kemudian menjalankan:

~~~bash
ssh user@192.168.1.20
~~~

koneksi menuju TCP port 22 akan ditolak oleh firewall.

---

## 10. Mencegah Server Menjadi SSH Jumping Point

Misalkan attacker berhasil mengeksploitasi aplikasi web:

~~~text
Attacker
   |
   | exploit
   v
+-----------------+
| Web Application |
+-----------------+
        |
        v
+----------------+
|   Web Server   |
|  compromised   |
+----------------+
~~~

Tanpa outbound filtering:

~~~text
Web Server
    |
    +---- SSH ----> DB Server
    |
    +---- SSH ----> Backup Server
    |
    +---- SSH ----> Application Server
~~~

Server yang telah dikompromikan dapat digunakan sebagai jumping point.

Dengan outbound TCP/22 diblokir:

~~~text
Web Server
    |
    +---- SSH ----X DB Server
    |
    +---- SSH ----X Backup Server
    |
    +---- SSH ----X Application Server
~~~

Pembatasan tersebut tidak membuat lateral movement menjadi mustahil, tetapi **menghilangkan salah satu jalur yang dapat digunakan attacker**.

Konsep pentingnya adalah:

> **Server sebaiknya hanya diperbolehkan membuat koneksi jaringan yang memang dibutuhkan oleh fungsinya.**

Prinsip tersebut merupakan salah satu bentuk **egress filtering**.

---

## 11. Hardening sshd Saja Tidak Cukup

Hardening OpenSSH tetap penting.

Misalnya:

~~~text
PermitRootLogin no
PasswordAuthentication no
AllowUsers administrator
~~~

Namun konfigurasi tersebut terutama mengatur bagaimana SSH server menerima autentikasi.

Firewall memberikan kontrol pada lapisan jaringan.

~~~text
VPN
 |
 v
Firewall
 |
 v
TCP/22
 |
 v
OpenSSH
 |
 v
Authentication
 |
 v
Linux User
~~~

Keduanya merupakan lapisan keamanan yang berbeda dan dapat digunakan secara bersamaan sebagai bagian dari prinsip **defense in depth**.

---

## 12. Script Hardening SSH

Berikut script lengkap yang digunakan pada implementasi ini.

~~~bash
#!/bin/bash

# ============================================================
# SSH NETWORK HARDENING - nftables
#
# Fungsi:
# 1. SSH INBOUND hanya dari network VPN
# 2. SSH OUTBOUND TCP/22 diblokir
#
# PENTING:
# Ubah VPN_NETWORK sebelum menjalankan script.
# script by NR Technology & ChatGPT
# ============================================================

set -e

# ============================================================
# KONFIGURASI
# ============================================================

VPN_NETWORK="10.10.10.0/24"

# ============================================================
# CEK ROOT
# ============================================================

if [ "$(id -u)" -ne 0 ]; then
    echo "[ERROR] Script harus dijalankan sebagai root."
    exit 1
fi

echo "================================================"
echo " SSH NETWORK HARDENING"
echo "================================================"
echo
echo "VPN Network : $VPN_NETWORK"
echo
echo "Kebijakan:"
echo "  INBOUND  TCP/22 dari VPN  : ALLOW"
echo "  INBOUND  TCP/22 selain VPN: REJECT"
echo "  OUTBOUND TCP/22           : REJECT"
echo

# ============================================================
# KONFIRMASI
# ============================================================

read -p "Apakah VPN_NETWORK sudah benar? (yes/no): " CONFIRM

if [ "$CONFIRM" != "yes" ]; then
    echo
    echo "[CANCEL] Tidak ada perubahan firewall."
    exit 0
fi

# ============================================================
# INSTALL NFTABLES JIKA BELUM TERSEDIA
# ============================================================

if ! command -v nft >/dev/null 2>&1; then

    echo "[INFO] nftables belum tersedia."
    echo "[INFO] Menginstall nftables..."

    apt-get update
    apt-get install -y nftables

fi

echo "[OK] nftables tersedia."

# ============================================================
# VALIDASI VPN NETWORK
# ============================================================

if ! echo "$VPN_NETWORK" | grep -Eq \
'^([0-9]{1,3}\.){3}[0-9]{1,3}/[0-9]{1,2}$'; then

    echo "[ERROR] Format VPN_NETWORK tidak valid."
    echo "Contoh: 10.10.10.0/24"
    exit 1

fi

# ============================================================
# BACKUP NFTABLES
# ============================================================

if [ -f /etc/nftables.conf ]; then

    BACKUP="/etc/nftables.conf.backup-$(date +%Y%m%d-%H%M%S)"

    cp -a /etc/nftables.conf "$BACKUP"

    echo "[OK] Backup dibuat:"
    echo "     $BACKUP"

fi

# ============================================================
# TABLE inet filter
# ============================================================

if ! nft list table inet filter >/dev/null 2>&1; then

    nft add table inet filter

    echo "[OK] Table inet filter dibuat."

else

    echo "[OK] Table inet filter sudah ada."

fi

# ============================================================
# CHAIN INPUT
# ============================================================

if ! nft list chain inet filter input >/dev/null 2>&1; then

    nft 'add chain inet filter input {
        type filter hook input priority filter;
        policy accept;
    }'

    echo "[OK] Chain input dibuat."

else

    echo "[OK] Chain input sudah ada."

fi

# ============================================================
# CHAIN OUTPUT
# ============================================================

if ! nft list chain inet filter output >/dev/null 2>&1; then

    nft 'add chain inet filter output {
        type filter hook output priority filter;
        policy accept;
    }'

    echo "[OK] Chain output dibuat."

else

    echo "[OK] Chain output sudah ada."

fi

# ============================================================
# SSH INBOUND DARI VPN
# ============================================================

if nft list chain inet filter input | \
grep -Fq "SSH_FROM_VPN"; then

    echo "[OK] Rule SSH_FROM_VPN sudah ada."

else

    nft add rule inet filter input \
        ip saddr "$VPN_NETWORK" \
        tcp dport 22 \
        accept \
        comment \"SSH_FROM_VPN\"

    echo "[OK] SSH dari VPN diizinkan."

fi

# ============================================================
# BLOCK SSH INBOUND NON-VPN
# ============================================================

if nft list chain inet filter input | \
grep -Fq "BLOCK_SSH_NON_VPN"; then

    echo "[OK] Rule BLOCK_SSH_NON_VPN sudah ada."

else

    nft add rule inet filter input \
        tcp dport 22 \
        reject \
        comment \"BLOCK_SSH_NON_VPN\"

    echo "[OK] SSH dari luar VPN diblokir."

fi

# ============================================================
# BLOCK SSH OUTBOUND
# ============================================================

if nft list chain inet filter output | \
grep -Fq "BLOCK_SSH_OUTBOUND"; then

    echo "[OK] Rule BLOCK_SSH_OUTBOUND sudah ada."

else

    nft add rule inet filter output \
        tcp dport 22 \
        reject \
        comment \"BLOCK_SSH_OUTBOUND\"

    echo "[OK] SSH outbound TCP/22 diblokir."

fi

# ============================================================
# SIMPAN RULESET
# ============================================================

nft list ruleset > /etc/nftables.conf

echo "[OK] Ruleset disimpan ke /etc/nftables.conf"

# ============================================================
# ENABLE NFTABLES
# ============================================================

systemctl enable nftables >/dev/null 2>&1

echo "[OK] nftables diaktifkan saat boot."

# ============================================================
# TAMPILKAN HASIL
# ============================================================

echo
echo "================================================"
echo " CHAIN INPUT"
echo "================================================"

nft list chain inet filter input

echo
echo "================================================"
echo " CHAIN OUTPUT"
echo "================================================"

nft list chain inet filter output

echo
echo "================================================"
echo " HASIL HARDENING"
echo "================================================"
echo
echo "VPN NETWORK:"
echo "  $VPN_NETWORK"
echo
echo "SSH INBOUND:"
echo "  VPN       -> Server TCP/22 : ALLOW"
echo "  NON-VPN   -> Server TCP/22 : REJECT"
echo
echo "SSH OUTBOUND:"
echo "  Server -> TCP/22           : REJECT"
echo
echo "================================================"
echo " PERINGATAN"
echo "================================================"
echo
echo "JANGAN logout dari SSH saat ini."
echo
echo "Buka terminal kedua melalui VPN dan pastikan"
echo "SSH ke server berhasil sebelum menutup sesi ini."
echo
echo "================================================"
~~~

Simpan script, misalnya:

~~~bash
nano ssh-hardening.sh
~~~

Berikan permission executable:

~~~bash
chmod +x ssh-hardening.sh
~~~

Kemudian jalankan:

~~~bash
sudo ./ssh-hardening.sh
~~~

---

## 13. Memeriksa Hasil Konfigurasi

Setelah konfigurasi diterapkan, periksa keseluruhan ruleset:

~~~bash
sudo nft list ruleset
~~~

Untuk memeriksa chain `input`:

~~~bash
sudo nft list chain inet filter input
~~~

Untuk memeriksa chain `output`:

~~~bash
sudo nft list chain inet filter output
~~~

Secara konseptual hasilnya akan terlihat seperti:

~~~text
table inet filter {
    chain input {
        type filter hook input priority filter; policy accept;

        ip saddr 10.10.10.0/24 tcp dport 22 accept comment "SSH_FROM_VPN"
        tcp dport 22 reject comment "BLOCK_SSH_NON_VPN"
    }

    chain output {
        type filter hook output priority filter; policy accept;

        tcp dport 22 reject comment "BLOCK_SSH_OUTBOUND"
    }
}
~~~

---

## 14. Pengujian

### Pengujian dari VPN

Dari komputer yang terhubung ke VPN:

~~~bash
ssh administrator@IP-SERVER
~~~

Koneksi seharusnya **berhasil**.

### Pengujian dari Luar VPN

Dari jaringan yang tidak termasuk dalam network VPN:

~~~bash
ssh administrator@IP-SERVER
~~~

Koneksi seharusnya **ditolak**.

### Pengujian SSH Outbound

Dari server yang telah di-hardening:

~~~bash
ssh user@IP-SERVER-LAIN
~~~

Koneksi menuju TCP port 22 seharusnya **ditolak**.

Hasil akhir yang diharapkan:

~~~text
VPN      -> Server:22    ALLOW
Non-VPN  -> Server:22    REJECT
Server   -> Remote:22    REJECT
~~~

---

## 15. Jangan Langsung Menutup Sesi SSH

Setelah menjalankan script:

> **Jangan langsung logout dari sesi SSH yang sedang digunakan.**

Buka terminal kedua, hubungkan komputer ke VPN, kemudian lakukan koneksi baru:

~~~bash
ssh administrator@IP-SERVER
~~~

Pastikan login berhasil sebelum menutup sesi SSH pertama.

Kesalahan konfigurasi firewall dapat menyebabkan administrator kehilangan akses remote ke server.

Pada server produksi, sebaiknya tersedia akses alternatif seperti console hypervisor, console VM, IPMI, iDRAC, iLO, atau akses fisik.

---

## 16. Apakah SSH Outbound Selalu Harus Diblokir?

Tidak.

Beberapa server memang membutuhkan SSH outbound untuk:

- deployment;
- Git melalui SSH;
- SFTP;
- SCP;
- rsync over SSH;
- backup;
- automation;
- Ansible.

Dalam kondisi tersebut lebih baik menggunakan **allowlist**.

Misalnya SSH hanya diperbolehkan menuju backup server `10.20.30.40`:

~~~bash
nft add rule inet filter output \
    ip daddr 10.20.30.40 \
    tcp dport 22 \
    accept
~~~

Kemudian blokir destination TCP/22 lainnya:

~~~bash
nft add rule inet filter output \
    tcp dport 22 \
    reject
~~~

Hasilnya:

~~~text
Server -> Backup Server:22    ALLOW
Server -> TCP/22 lainnya      REJECT
~~~

Pendekatan tersebut lebih sesuai dengan prinsip **least privilege**.

---

## 17. Memblokir Port 22 Bukan Berarti Memblokir Semua SSH

Rule:

~~~bash
tcp dport 22 reject
~~~

memblokir **TCP destination port 22**, bukan mendeteksi seluruh trafik yang menggunakan protokol SSH.

SSH dapat dijalankan pada port lain, misalnya:

~~~text
TCP/2222
TCP/2200
TCP/443
~~~

Contohnya:

~~~bash
ssh -p 2222 user@server
~~~

tidak akan terkena rule yang hanya memblokir destination port 22.

Karena itu outbound filtering TCP/22 merupakan salah satu lapisan mitigasi, **bukan solusi tunggal untuk mencegah lateral movement**.

Pada lingkungan dengan kebutuhan keamanan lebih tinggi, kontrol dapat diperluas menggunakan:

- network segmentation;
- internal firewall;
- ACL;
- application-aware firewall;
- IDS/IPS;
- SIEM;
- endpoint security;
- network monitoring.

---

## 18. Defense in Depth

Pembatasan menggunakan nftables sebaiknya menjadi bagian dari strategi keamanan yang lebih luas.

~~~text
                 Internet
                    |
                    X
                 TCP/22
                    |
              +-----------+
              |    VPN    |
              +-----------+
                    |
                    v
              +-----------+
              | nftables  |
              +-----------+
                    |
                    v
              +-----------+
              | OpenSSH   |
              +-----------+
                    |
                    v
              +-----------+
              | Linux OS  |
              +-----------+
                    |
             SSH outbound
                    X
~~~

Lapisan hardening dapat mencakup:

- VPN sebagai jalur administrasi;
- firewall restriction;
- SSH key authentication;
- menonaktifkan root login;
- menonaktifkan password authentication;
- `AllowUsers` atau `AllowGroups`;
- proteksi brute-force;
- logging dan monitoring;
- network segmentation;
- outbound filtering.

Pendekatan berlapis seperti ini merupakan implementasi prinsip **defense in depth**.

---

## 19. Kesimpulan

Hardening SSH tidak hanya berbicara tentang bagaimana melindungi **pintu masuk** menuju server.

Kita juga perlu mempertimbangkan apa yang dapat dilakukan server setelah berhasil dikompromikan.

Dalam implementasi ini, nftables digunakan untuk menerapkan tiga kebijakan:

~~~text
SSH dari VPN         -> ALLOW
SSH dari luar VPN    -> REJECT
SSH outbound TCP/22  -> REJECT
~~~

Pembatasan inbound memperkecil permukaan serangan terhadap layanan SSH.

Sementara pembatasan outbound membantu mengurangi kemungkinan server yang telah dikompromikan digunakan sebagai **SSH jumping point** untuk melakukan lateral movement menuju sistem lainnya.

Namun outbound blocking harus disesuaikan dengan fungsi server. Server yang membutuhkan SSH untuk backup, deployment, SFTP, Git, atau automation dapat diberikan pengecualian secara spesifik.

Prinsip akhirnya sederhana:

> **Sebuah server hanya boleh menerima dan membuat koneksi jaringan yang memang diperlukan untuk menjalankan fungsinya.**

Dengan pendekatan tersebut, firewall tidak hanya menjadi penjaga trafik yang datang dari luar, tetapi juga menjadi mekanisme untuk membatasi apa yang dapat dilakukan sebuah server apabila suatu saat server tersebut berhasil dikompromikan.

---
{{< saweria >}}