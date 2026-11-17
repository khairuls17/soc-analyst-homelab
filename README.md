# SOC Analyst Homelab

Dokumentasi teknis pembangunan dan konfigurasi SOC homelab berbasis VMware Workstation, MikroTik CHR, Ubuntu Server, Splunk Enterprise, Splunk Universal Forwarder, dan MITRE Caldera.

> **Status dokumentasi:** konfigurasi dasar pengumpulan log telah dibuat dan event pengujian dari beberapa sumber berhasil ditemukan di Splunk. Beberapa hal operasional, terutama verifikasi autostart setelah reboot dan pengumpulan telemetry endpoint Windows, masih menjadi pekerjaan lanjutan.
>
> **Prinsip dokumentasi:** langkah yang sudah dilakukan dan diuji ditandai sebagai *terkonfirmasi*. Nilai yang belum diketahui secara pasti tidak ditebak dan harus diisi dari konfigurasi aktual.
>
> **Format:** seluruh panduan menggunakan teks, tabel, dan command line. Tidak memerlukan screenshot.

---

## Daftar Isi

- [1. Gambaran Proyek](#1-gambaran-proyek)
- [2. Status Implementasi](#2-status-implementasi)
- [3. Arsitektur dan Topologi](#3-arsitektur-dan-topologi)
- [4. Port dan Alur Komunikasi](#4-port-dan-alur-komunikasi)
- [5. Persiapan VMware Workstation](#5-persiapan-vmware-workstation)
- [6. Konfigurasi MikroTik CHR](#6-konfigurasi-mikrotik-chr)
- [7. Konfigurasi Jaringan Ubuntu](#7-konfigurasi-jaringan-ubuntu)
- [8. Persiapan dan Instalasi Splunk Enterprise](#8-persiapan-dan-instalasi-splunk-enterprise)
- [9. Receiving Port dan Index Splunk](#9-receiving-port-dan-index-splunk)
- [10. Instalasi dan Konfigurasi Universal Forwarder](#10-instalasi-dan-konfigurasi-universal-forwarder)
- [11. Pengumpulan Log Linux server-general](#11-pengumpulan-log-linux-server-general)
- [12. Integrasi Caldera dan Splunk](#12-integrasi-caldera-dan-splunk)
- [13. Integrasi MikroTik Syslog ke Splunk](#13-integrasi-mikrotik-syslog-ke-splunk)
- [14. Monitoring Sistem Operasi Splunk Server Sendiri](#14-monitoring-sistem-operasi-splunk-server-sendiri)
- [15. Validasi Data dengan SPL](#15-validasi-data-dengan-spl)
- [16. Prosedur Pemeriksaan Operasional](#16-prosedur-pemeriksaan-operasional)
- [17. Troubleshooting yang Ditemui](#17-troubleshooting-yang-ditemui)
- [18. Backup dan Pemeliharaan Konfigurasi](#18-backup-dan-pemeliharaan-konfigurasi)
- [19. Keamanan dan Batasan Lab](#19-keamanan-dan-batasan-lab)
- [20. Pekerjaan Berikutnya](#20-pekerjaan-berikutnya)
- [21. Checklist Implementasi](#21-checklist-implementasi)
- [22. Referensi](#22-referensi)
- [Lampiran A — Lembar Inventaris](#lampiran-a--lembar-inventaris)

---

## 1. Gambaran Proyek

SOC (Security Operations Center) homelab ini digunakan untuk mempraktikkan alur kerja dasar seorang SOC Analyst. Fokus tahap awal adalah membangun infrastruktur dan memastikan telemetry dapat dikirim ke satu tempat terpusat sebelum membuat detection rule atau alert.

### Tujuan

1. Membuat beberapa segmen jaringan virtual menggunakan VMware Workstation.
2. Menggunakan MikroTik CHR sebagai router antar-subnet lab.
3. Mengoperasikan Splunk Enterprise sebagai SIEM.
4. Mengirim log Linux menggunakan Splunk Universal Forwarder.
5. Mengirim syslog MikroTik ke Splunk melalui UDP.
6. Mengumpulkan log OS pada mesin Splunk Server itu sendiri.
7. Menyiapkan MITRE Caldera sebagai platform emulasi adversary terkontrol.
8. Melatih pencarian log, validasi event, dan investigasi menggunakan SPL.

### Ruang lingkup saat ini

- Splunk Enterprise berada pada Ubuntu Server dengan alamat `192.168.90.100`.
- Splunk Web diakses melalui TCP `8000`.
- Universal Forwarder mengirim data ke Splunk melalui TCP `9997`.
- MikroTik mengirim syslog ke Splunk melalui UDP `5514`.
- VM `server-general` mengirim `/var/log/syslog` dan `/var/log/auth.log` ke index `server_general`.
- VM `red-tim` yang menjalankan Caldera mengirim log OS yang sama ke index `caldera`.
- MikroTik mengirim log kategori `system`, `account`, `firewall`, dan `warning` ke index `mikrotik`.
- Splunk Server dikonfigurasi untuk membaca log OS lokal ke index `splunk_server_os`.

### Yang belum termasuk dalam hasil terkonfirmasi

- Windows endpoint dan Windows Security Event Log belum dibangun/diintegrasikan pada tahap dokumentasi ini.
- Belum semua aktivitas aplikasi Caldera dipastikan masuk ke Splunk. Yang terverifikasi adalah log OS VM Caldera dan event uji Universal Forwarder.
- Detection rule dan alert SOC belum dinyatakan selesai.
- Pengujian autostart setelah reboot belum diklaim berhasil sampai ada bukti event baru pascareboot.
- Pemetaan setiap VMnet VMware terhadap interface MikroTik perlu disimpan secara terpisah berdasarkan konfigurasi aktif.

---

## 2. Status Implementasi

| Komponen | Status | Detail |
|---|---|---|
| VMware Workstation | Digunakan | Menjalankan router dan VM lab |
| MikroTik CHR | Terkonfigurasi | RouterOS 7.20.1; interface dan IP tercatat pada bagian topologi |
| Splunk Enterprise | Terpasang dan berjalan | `/opt/splunk`, IP `192.168.90.100`, versi tercatat `10.6.0.5` |
| Splunk Web | Berhasil diakses | `http://192.168.90.100:8000` |
| Receiving TCP 9997 | Aktif dan diuji | Digunakan Universal Forwarder |
| Input UDP 5514 | Aktif dan diuji | Digunakan syslog MikroTik |
| `server-general` UF | Berhasil mengirim event uji | Index `server_general` |
| Caldera UF | Berhasil mengirim event uji | Index `caldera` |
| MikroTik syslog | Berhasil mengirim event uji | Index `mikrotik` |
| Splunk Server OS logs | Akses file diperbaiki dan input lokal disiapkan | Index `splunk_server_os`; pastikan hasil event terbaru terlihat |
| Caldera service | Dikonfigurasi dengan systemd | `/etc/systemd/system/caldera.service` |
| Windows endpoint | Belum diimplementasikan | Rencana untuk tahap berikutnya |
| Detection dan alert | Belum diimplementasikan penuh | Menunggu cakupan telemetry dan rule yang divalidasi |
| Autostart setelah reboot | Belum terverifikasi | Wajib diuji dengan reboot dan event baru |

---

## 3. Arsitektur dan Topologi

### 3.1 Diagram logis

```text
                       HOST WINDOWS
                            |
                  VMware Workstation / VMnet
                            |
                     MikroTik CHR Router
        ____________________|____________________________
       |          |              |             |         |
   ether2      ether3         ether4         ether5    ether6
       |          |              |             |         |
  70.0/24     80.0/24        90.0/24        99.0/24   11.11.11.0/24
       |          |              |             |
   red-tim    Windows       Splunk Server   server-general
   Caldera    endpoint      192.168.90.100  Ubuntu + UF
   + UF       (rencana)          |              |
       |                         |              |
       +-------------------------+--------------+
                                 |
                         Splunk Enterprise
                                 |
                  +--------------+---------------+
                  |              |               |
            Index caldera  Index mikrotik  Index server_general
                           (UDP 5514)       (TCP 9997)
```

Diagram ini adalah gambaran logis. Lokasi fisik VMnet untuk setiap interface perlu dicocokkan dengan pengaturan VMware aktual. Jangan menganggap subnet rencana otomatis sama dengan mapping VMnet yang sekarang aktif.

### 3.2 IP interface MikroTik yang tercatat

| Interface | IP address | Keterangan |
|---|---|---|
| `ether1` | DHCP `192.168.82.128/24` | WAN/upstream; gateway tercatat `192.168.82.2` |
| `ether2` | `192.168.70.1/24` | Segmen lab 70 |
| `ether3` | `192.168.80.1/24` | Segmen lab 80 |
| `ether4` | `192.168.90.1/24` | Segmen Splunk |
| `ether5` | `192.168.99.1/24` | Segmen server-general |
| `ether6` | `11.11.11.1/24` | Segmen tambahan |

Informasi tersebut mengikuti konfigurasi yang tercatat selama pengerjaan. Pemeriksaan ulang RouterOS dapat dilakukan dengan:

```routeros
/interface print
/ip address print
/ip route print detail
```

### 3.3 Alamat dan path komponen

| Komponen | Nilai tercatat |
|---|---|
| Splunk Server | `192.168.90.100` |
| Gateway segmen Splunk | `192.168.90.1` |
| Splunk Web | `http://192.168.90.100:8000` |
| Splunk Enterprise home | `/opt/splunk` |
| Universal Forwarder home | `/opt/splunkforwarder` |
| Caldera project | `/home/hairul/caldera` |
| Caldera Python environment | `/home/hairul/caldera/venv` |
| Caldera systemd unit | `/etc/systemd/system/caldera.service` |
| Caldera UI port | `8888/TCP` |

IP aktual `server-general` dan `red-tim` tidak ditetapkan dalam README ini karena nilai finalnya belum tercatat secara pasti. Ambil dari tiap VM dengan `ip -br address`.

### 3.4 Rencana subnet

| Segmen | Subnet | Gateway yang tercatat/rencana |
|---|---|---|
| Lab segment 70 | `192.168.70.0/24` | `192.168.70.1` |
| Lab segment 80 | `192.168.80.0/24` | `192.168.80.1` |
| Splunk | `192.168.90.0/24` | `192.168.90.1` |
| server-general | `192.168.99.0/24` | `192.168.99.1` |
| Segmen tambahan | `11.11.11.0/24` | `11.11.11.1` |

### 3.5 Inventaris VMnet

Isi tabel ini dari **VMware Virtual Network Editor** dan pengaturan Network Adapter setiap VM. Nomor VMnet tidak direka karena belum ada pemetaan final yang terkonfirmasi.

| Fungsi | VMnet VMware | Interface MikroTik | Subnet | Catatan |
|---|---|---|---|---|
| WAN/upstream | Isi dari VMware | `ether1` | DHCP `192.168.82.0/24` | Gateway tercatat `.82.2` |
| Segment 70 | Isi dari VMware | `ether2` | `192.168.70.0/24` | Verifikasi VM yang terhubung |
| Segment 80 | Isi dari VMware | `ether3` | `192.168.80.0/24` | Endpoint belum selesai |
| Segment 90 | Isi dari VMware | `ether4` | `192.168.90.0/24` | Splunk Server |
| Segment 99 | Isi dari VMware | `ether5` | `192.168.99.0/24` | server-general |
| Segment 11 | Isi dari VMware | `ether6` | `11.11.11.0/24` | Fungsi/VM perlu dicatat |

---

## 4. Port dan Alur Komunikasi

| Sumber | Tujuan | Port/protokol | Fungsi |
|---|---|---|---|
| Browser admin | Splunk Server | `8000/TCP` | Splunk Web |
| `server-general` UF | Splunk Server | `9997/TCP` | Forwarding event |
| Caldera VM UF | Splunk Server | `9997/TCP` | Forwarding log OS |
| MikroTik | Splunk Server | `5514/UDP` | Syslog router |
| Browser admin/lab | Caldera VM | `8888/TCP` | Caldera UI |
| Admin host | Ubuntu VM | `22/TCP` | SSH, jika diaktifkan |

`9997/TCP` dan `5514/UDP` merupakan dua mekanisme input yang berbeda. Universal Forwarder mengirim ke receiving port Splunk, sedangkan MikroTik mengirim syslog ke input UDP yang dibuat di Splunk.

Aturan firewall harus membatasi sumber dan tujuan sesuai kebutuhan. Jangan membuka seluruh port untuk semua subnet.

---

## 5. Persiapan VMware Workstation

### 5.1 Periksa Virtual Network Editor

1. Buka VMware Workstation.
2. Buka **Edit → Virtual Network Editor**.
3. Catat VMnet, tipe jaringan, subnet, dan apakah adapter host ikut dibuat.
4. Cocokkan setiap VMnet dengan interface virtual MikroTik yang akan menjadi gateway segmen tersebut.
5. Tentukan jalur host Windows menuju Splunk Web.
6. Jangan mengubah VMnet yang sedang dipakai banyak VM sebelum mencatat konfigurasi awalnya.

### 5.2 Hubungkan adapter virtual setiap VM

Untuk setiap VM:

1. Matikan VM sebelum mengubah konfigurasi adapter bila memungkinkan.
2. Buka **VM Settings → Network Adapter**.
3. Pilih VMnet yang sudah ditentukan.
4. Aktifkan **Connected** dan **Connect at power on** sesuai kebutuhan.
5. Cocokkan adapter virtual dengan interface di guest OS/RouterOS melalui MAC address dan status link.

Untuk MikroTik CHR, satu adapter virtual mewakili satu interface jaringan. Urutan `ether1`, `ether2`, dan seterusnya perlu diverifikasi; jangan menebak hanya dari urutan klik.

### 5.3 Pemeriksaan dari guest OS

Ubuntu:

```bash
ip -br address
ip link
ip route
```

MikroTik:

```routeros
/interface print
/interface ethernet print detail
```

---

## 6. Konfigurasi MikroTik CHR

**Perangkat:** MikroTik CHR, RouterOS `7.20.1`.

### 6.1 Verifikasi interface dan alamat

```routeros
/interface print
/ip address print
/ip route print detail
```

Pastikan interface yang benar terhubung ke VMnet yang tepat sebelum melakukan perubahan.

### 6.2 Contoh penambahan alamat interface

Alamat berikut adalah alamat yang tercatat pada konfigurasi lab. Jalankan perintah `add` hanya jika alamat tersebut belum ada. Jika sudah ada, jangan menambahkan duplikat.

```routeros
/ip address add address=192.168.70.1/24 interface=ether2 comment="LAB-70"
/ip address add address=192.168.80.1/24 interface=ether3 comment="LAB-80"
/ip address add address=192.168.90.1/24 interface=ether4 comment="SPLUNK-LAN"
/ip address add address=192.168.99.1/24 interface=ether5 comment="SERVER-LAN"
/ip address add address=11.11.11.1/24 interface=ether6 comment="LAB-11"
```

Perintah di atas menunjukkan bentuk konfigurasi, bukan instruksi untuk menambah ulang IP pada router yang sudah berjalan. Cek lebih dulu:

```routeros
/ip address print
```

### 6.3 Verifikasi route

```routeros
/ip route print detail
```

RouterOS seharusnya memiliki connected route untuk setiap subnet yang terpasang pada interface aktif. Default route WAN perlu diperiksa terpisah.

### 6.4 Periksa firewall dan NAT

```routeros
/ip firewall filter print
/ip firewall nat print
```

Detail aturan firewall/NAT yang sedang aktif harus disimpan dari konfigurasi aktual. Tidak ada aturan drop/accept generik yang aman untuk diterapkan tanpa melihat urutan rule dan kebutuhan routing lab.

Prinsip kebijakan:

- Akses administrasi router hanya dari jaringan pengelolaan.
- Izinkan Universal Forwarder menuju `192.168.90.100:9997/TCP`.
- Izinkan MikroTik mengirim UDP ke `192.168.90.100:5514`.
- Izinkan admin membuka Splunk Web ke `192.168.90.100:8000/TCP`.
- Batasi akses Caldera UI `8888/TCP` ke admin/lab.
- Jangan membuka akses Caldera langsung ke internet publik.

### 6.5 Pemeriksaan akhir router

```routeros
/interface print
/ip address print
/ip route print detail
/ip firewall filter print
/ip firewall nat print
/system logging print
/system logging action print
```

---

## 7. Konfigurasi Jaringan Ubuntu

Bagian ini berlaku untuk Ubuntu `server-general`, Splunk Server, dan VM Caldera. Gunakan nilai IP aktual masing-masing VM.

### 7.1 Identifikasi interface dan route

```bash
ip -br address
ip route
ip link
```

Catat nama interface, misalnya `ens33`, alamat/prefix, gateway default, dan status `UP`.

### 7.2 Periksa Netplan

```bash
ls -l /etc/netplan/
```

Buka file yang benar-benar ada. Contoh:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Jangan mengasumsikan nama file selalu sama.

### 7.3 Contoh konfigurasi IP statis

Berikut contoh untuk host pada subnet server `192.168.99.0/24`. Nilai IP host `192.168.99.10` hanya contoh konfigurasi, bukan klaim IP aktual `server-general`.

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 192.168.99.10/24
      routes:
        - to: default
          via: 192.168.99.1
      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8
```

Ganti `ens33`, IP/prefix, gateway, dan DNS sesuai kondisi jaringan. Pastikan tidak ada duplikasi IP dan tidak bertabrakan dengan DHCP pool.

### 7.4 Validasi dan terapkan Netplan

Simpan salinan sebelum melakukan perubahan:

```bash
sudo cp -a /etc/netplan "/etc/netplan.backup.$(date +%Y%m%d-%H%M%S)"
```

Validasi syntax:

```bash
sudo netplan generate
```

Uji perubahan:

```bash
sudo netplan try
```

Jika koneksi tetap benar, konfirmasi perubahan sesuai prompt. Setelah itu:

```bash
ip -br address
ip route
ping -c 4 192.168.90.100
```

Ping ke Splunk hanya relevan untuk VM yang seharusnya dapat mencapai subnet Splunk.

### 7.5 Uji route dan port

```bash
ip route get 192.168.90.100
```

Uji TCP receiving port:

```bash
timeout 3 bash -c '</dev/tcp/192.168.90.100/9997' \
  && echo "PORT TERBUKA" \
  || echo "PORT GAGAL"
```

Ping yang berhasil tidak otomatis membuktikan port TCP `9997` terbuka.

---

## 8. Persiapan dan Instalasi Splunk Enterprise

### 8.1 Informasi Splunk Server

| Parameter | Nilai tercatat |
|---|---|
| Hostname | `splunk-server` |
| OS | Ubuntu Server |
| IP | `192.168.90.100` |
| Web UI | `http://192.168.90.100:8000` |
| Instalasi | `/opt/splunk` |
| Versi tercatat | `10.6.0.5` |

### 8.2 Update Ubuntu dan pemeriksaan storage

```bash
sudo apt update
sudo apt upgrade -y
ip -br address
ip route
df -h
lsblk
```

Pastikan ruang disk memadai sebelum memasang Splunk. Simpan output `lsblk` dan `df -h` sebagai catatan awal.

### 8.3 Unduh paket Splunk Enterprise

Gunakan halaman resmi:

<https://www.splunk.com/en_us/download/splunk-enterprise.html>

Pilih paket Linux yang cocok dengan arsitektur dan distribusi VM. Versi portal dapat berubah; versi yang tercatat pada instalasi lab ini adalah `10.6.0.5`.

Setelah paket `.deb` tersedia di VM, masuk ke folder unduhan dan periksa nama file:

```bash
cd ~/Downloads
ls -lh
```

Pasang paket dengan nama file sebenarnya:

```bash
sudo dpkg -i ./NAMA_PAKET_SPLUNK.deb
```

`NAMA_PAKET_SPLUNK.deb` adalah placeholder, harus diganti dengan nama paket yang diunduh.

### 8.4 Inisialisasi Splunk Enterprise

```bash
sudo /opt/splunk/bin/splunk start --accept-license
```

Ikuti prompt awal dan buat kredensial administrator. Jangan menyimpan password pada README atau repository.

Periksa status:

```bash
sudo /opt/splunk/bin/splunk status
```

### 8.5 Akses Splunk Web

Buka melalui browser dari host/jaringan admin yang diizinkan:

```text
http://192.168.90.100:8000
```

Jika tidak terbuka, periksa status service, IP, route, firewall, dan listener:

```bash
sudo /opt/splunk/bin/splunk status
sudo ss -lntp | grep ':8000'
ip -br address
ip route
```

### 8.6 Catatan menjalankan Splunk sebagai user layanan

Splunk pada VM ini berjalan sebagai user `splunk`. Hindari menjalankan proses layanan sebagai root. Ketika diperlukan pemeriksaan proses:

```bash
ps -eo user,group,comm | grep splunkd
```

Jangan menganggap pesan warning dari perintah yang dijalankan dengan `sudo` sebagai bukti restart sukses. Selalu periksa status aktual.

---

## 9. Receiving Port dan Index Splunk

### 9.1 Aktifkan receiving TCP 9997

Pada Splunk Web:

1. Buka **Settings → Forwarding and receiving**.
2. Masuk ke bagian **Receive data** atau **Configure receiving** (label menu dapat berbeda antarversi).
3. Pilih **Add new port**.
4. Masukkan `9997`.
5. Simpan.

Verifikasi listener pada Splunk Server:

```bash
sudo ss -lntp | grep ':9997'
```

### 9.2 Buat index

Index yang digunakan di lab:

| Index | Sumber | Tujuan |
|---|---|---|
| `server_general` | Ubuntu `server-general` | Log sistem dan autentikasi server umum |
| `caldera` | Ubuntu VM `red-tim`/Caldera | Log sistem dan autentikasi VM Caldera |
| `mikrotik` | MikroTik | Syslog router |
| `splunk_server_os` | Ubuntu Splunk Server sendiri | Log OS mesin SIEM |

Buat dari **Settings → Indexes**. Pastikan spelling sama persis dengan tujuan input. Index `server_general` berbeda dari `server-general`.

### 9.3 Pemeriksaan ringkas

```bash
sudo ss -lntp | grep -E ':8000|:9997'
sudo ss -lunp | grep ':5514'
```

---

## 10. Instalasi dan Konfigurasi Universal Forwarder

### 10.1 Gambaran

Universal Forwarder dipasang pada mesin Linux sumber log dan mengirim data ke Splunk Enterprise.

Lokasi pada lab:

```text
/opt/splunkforwarder
```

Tujuan forwarding:

```text
192.168.90.100:9997
```

### 10.2 Unduh paket

Gunakan halaman resmi:

<https://www.splunk.com/en_us/download/universal-forwarder.html>

Pilih paket `.deb` Linux yang sesuai arsitektur VM. Versi yang tercatat pada salah satu instalasi UF lab adalah `10.4.3`; versi mesin lain perlu diperiksa langsung.

Contoh jika paket disimpan ke `/tmp/splunkforwarder.deb`:

```bash
ls -lh /tmp/splunkforwarder.deb
sudo dpkg -i /tmp/splunkforwarder.deb
```

Jika nama paket berbeda, gunakan nama file sebenarnya.

### 10.3 Jalankan UF untuk pertama kali

```bash
sudo /opt/splunkforwarder/bin/splunk start --accept-license
```

Ikuti prompt pembuatan kredensial admin Forwarder bila diminta. Jangan memasukkan password ke script atau dokumentasi publik.

### 10.4 Tambahkan target Splunk

```bash
sudo /opt/splunkforwarder/bin/splunk add forward-server 192.168.90.100:9997
```

Perintah ini digunakan pada konfigurasi UF lab. Sebelum mengulangnya, periksa target yang sudah ada agar tidak membuat konfigurasi ganda.

### 10.5 Tambahkan input file log

Untuk `server-general` dan VM Caldera, file yang dipantau adalah:

```text
/var/log/syslog
/var/log/auth.log
```

Contoh perintah pada `server-general` dengan index `server_general`:

```bash
sudo /opt/splunkforwarder/bin/splunk add monitor /var/log/syslog -index server_general -sourcetype syslog
sudo /opt/splunkforwarder/bin/splunk add monitor /var/log/auth.log -index server_general -sourcetype linux_secure
```

Pada VM Caldera, index tujuan yang digunakan adalah `caldera`:

```bash
sudo /opt/splunkforwarder/bin/splunk add monitor /var/log/syslog -index caldera -sourcetype syslog
sudo /opt/splunkforwarder/bin/splunk add monitor /var/log/auth.log -index caldera -sourcetype linux_secure
```

Jalankan perintah sesuai host tempat UF terpasang. Jika input sudah terdaftar, jangan menambahkannya ulang. Periksa daftar input terlebih dahulu.

### 10.6 Verifikasi UF

```bash
sudo /opt/splunkforwarder/bin/splunk status
sudo /opt/splunkforwarder/bin/splunk list forward-server
sudo /opt/splunkforwarder/bin/splunk list monitor
```

Jika CLI meminta autentikasi, gunakan kredensial administrator Forwarder yang dibuat saat instalasi. Jangan menuliskan password pada command yang akan tersimpan dalam shell history.

---

## 11. Pengumpulan Log Linux server-general

### 11.1 Sumber log

| File | Sourcetype | Fungsi umum |
|---|---|---|
| `/var/log/syslog` | `syslog` | Log layanan dan sistem Linux |
| `/var/log/auth.log` | `linux_secure` | Aktivitas autentikasi dan otorisasi |

Index tujuan: `server_general`.

### 11.2 Periksa file log

```bash
ls -lh /var/log/syslog /var/log/auth.log
sudo tail -n 20 /var/log/syslog
sudo tail -n 20 /var/log/auth.log
```

### 11.3 Generate event uji

```bash
logger "SOC_LAB_TEST server-general"
```

### 11.4 Cari event uji

Di Splunk Search & Reporting:

```spl
index=server_general "SOC_LAB_TEST"
```

Event `SOC_LAB_TEST server-general` sudah terkonfirmasi ditemukan di Splunk. Ini menjadi tes end-to-end bahwa event dari `server-general` diterima pada index yang dituju.

### 11.5 Troubleshooting sumber log

Jika event tidak terlihat:

1. Pastikan perintah `logger` dijalankan pada VM yang benar.
2. Pastikan `/var/log/syslog` ada dan menerima event baru.
3. Pastikan file tercantum pada `list monitor`.
4. Pastikan target UF mengarah ke `192.168.90.100:9997`.
5. Pastikan receiving port aktif.
6. Pastikan index `server_general` tersedia.
7. Atur time range pencarian untuk mencakup waktu event dibuat.

---

## 12. Integrasi Caldera dan Splunk

### 12.1 Informasi VM dan direktori Caldera

| Parameter | Nilai tercatat |
|---|---|
| Host/VM | `red-tim` |
| Direktori proyek | `/home/hairul/caldera` |
| Python virtual environment | `/home/hairul/caldera/venv` |
| Python executable untuk service | `/home/hairul/caldera/venv/bin/python` |
| Caldera server script | `/home/hairul/caldera/server.py` |
| systemd unit | `/etc/systemd/system/caldera.service` |
| UI port | `8888/TCP` |
| UF home | `/opt/splunkforwarder` |
| Index log OS | `caldera` |

Alamat IP aktual VM `red-tim` tidak dicantumkan karena perlu diverifikasi dari VM. Periksa:

```bash
ip -br address
ip route
```

### 12.2 Persiapan runtime dan proses build

Selama pemasangan Caldera, Python virtual environment digunakan, dependency Python dipasang, dan kebutuhan frontend Node/npm diproses. Proses build Caldera menggunakan:

```bash
cd /home/hairul/caldera
/home/hairul/caldera/venv/bin/python server.py build
```

Node/npm juga telah digunakan untuk memasang paket frontend yang diperlukan. Lokasi direktori frontend dan perintah npm rinci tidak disimpan secara pasti pada riwayat, sehingga README ini tidak mengarang path atau command `npm` yang tidak terverifikasi. Jika membangun ulang, ikuti instruksi `README` pada versi/tag Caldera yang dipilih dan catat versi Python, Node, npm, serta hasil build.

### 12.3 systemd service Caldera

Service dibuat pada:

```text
/etc/systemd/system/caldera.service
```

Isi service yang tercatat:

```ini
[Unit]
Description=MITRE Caldera Server
After=network-online.target docker.service
Wants=network-online.target
Requires=docker.service

[Service]
Type=simple
User=hairul
WorkingDirectory=/home/hairul/caldera
ExecStart=/home/hairul/caldera/venv/bin/python /home/hairul/caldera/server.py
Restart=on-failure
RestartSec=10
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```

Setelah mengubah file unit, systemd perlu membaca ulang unit file dan service perlu diaktifkan sesuai kebutuhan. Perintah administrasi:

```bash
sudo systemctl daemon-reload
sudo systemctl enable caldera
sudo systemctl start caldera
sudo systemctl status caldera
```

Status `enable` menunjukkan service dikonfigurasi untuk diaktifkan saat boot; tetap uji reboot untuk memastikan perilakunya.

Log service:

```bash
sudo journalctl -u caldera -n 100 --no-pager
sudo journalctl -u caldera -f
```

Pemeriksaan listener UI:

```bash
sudo ss -lntp | grep ':8888'
```

### 12.4 Universal Forwarder pada Caldera VM

UF berada di `/opt/splunkforwarder`, diarahkan ke:

```text
192.168.90.100:9997
```

File yang dipantau:

```text
/var/log/syslog
/var/log/auth.log
```

Index tujuan: `caldera`.

Verifikasi:

```bash
sudo /opt/splunkforwarder/bin/splunk status
sudo /opt/splunkforwarder/bin/splunk list forward-server
sudo /opt/splunkforwarder/bin/splunk list monitor
```

### 12.5 Uji forwarding Caldera

Event uji yang telah digunakan dan ditemukan:

```spl
index=caldera "CALDERA_UF_TEST"
```

Hasil ini membuktikan jalur pengiriman data UF dari VM Caldera berjalan untuk event uji dan log OS yang dipantau.

**Batasan telemetry:** log OS Caldera bukan hal yang sama dengan seluruh log internal aplikasi Caldera, dan bukan pula telemetry dari endpoint yang menjadi target emulasi. Untuk mendeteksi aktivitas pada endpoint, endpoint tersebut harus memiliki audit/security logging yang sesuai dan mengirimkannya ke Splunk.

---

## 13. Integrasi MikroTik Syslog ke Splunk

### 13.1 Siapkan index

Pastikan index berikut tersedia di Splunk:

```text
mikrotik
```

### 13.2 Buat UDP input di Splunk

Pada Splunk Web:

1. Buka **Settings → Data inputs → UDP**.
2. Pilih tambah input UDP baru.
3. Isi port `5514`.
4. Gunakan source type `syslog`.
5. Pilih index `mikrotik`.
6. Simpan.

Verifikasi listener di Splunk Server:

```bash
sudo ss -lunp | grep ':5514'
```

Listener sebelumnya terlihat pada `0.0.0.0:5514` dengan proses `splunkd`.

### 13.3 Buat remote logging action MikroTik

Command yang berhasil digunakan:

```routeros
/system/logging/action/add name=splunktarget target=remote remote=192.168.90.100 remote-port=5514
```

Nama action `splunktarget` dipilih tanpa tanda hubung. Dalam percobaan, nama dengan tanda hubung ditolak oleh RouterOS.

Sebelum menambah action, periksa konfigurasi yang ada:

```routeros
/system/logging/action/print
```

### 13.4 Tambahkan logging rules

Rules yang berhasil dibuat:

```routeros
/system/logging/add topics=system action=splunktarget
/system/logging/add topics=account action=splunktarget
/system/logging/add topics=firewall action=splunktarget
/system/logging/add topics=warning action=splunktarget
```

Rule `warning` ditambahkan saat troubleshooting karena event uji awal belum terkirim melalui rule yang tersedia. Setelah `warning` ditambahkan, paket terlihat pada `tcpdump` dan event uji ditemukan.

Periksa rule sebelum menambahkan ulang:

```routeros
/system/logging/print
```

### 13.5 Uji event dari MikroTik

```routeros
:log warning "MIKROTIK_REMOTE_TEST_06"
```

Cari di Splunk:

```spl
index=mikrotik "MIKROTIK_REMOTE_TEST_06"
```

Event tersebut sudah terkonfirmasi ditemukan.

### 13.6 Periksa paket UDP dengan tcpdump

Di Splunk Server:

```bash
sudo tcpdump -ni any 'host 192.168.90.1 and udp port 5514'
```

Kemudian generate event baru dari MikroTik. Jika paket terlihat di `tcpdump` tetapi tidak muncul di Splunk, fokuskan pemeriksaan ke konfigurasi input UDP, index, source type, dan time range.

### 13.7 Pencarian log akun MikroTik

Event account yang teramati di Splunk memiliki teks seperti `user admin logged in` dan `user admin logged out`.

Contoh pencarian:

```spl
index=mikrotik "logged in"
```

```spl
index=mikrotik "logged out"
```

Atau:

```spl
index=mikrotik ("logged in" OR "logged out")
| sort - _time
```

Pencarian login gagal belum terbukti menghasilkan event pada pengujian terdahulu. Jangan menyimpulkan logging gagal hanya dari pencarian kosong; pastikan event kegagalan benar-benar dihasilkan, dikirim, dan termasuk dalam kategori yang diteruskan.

### 13.8 Duplikasi event

Sebagian event dapat diteruskan lebih dari sekali jika beberapa logging rule mempunyai cakupan kategori yang bertumpang tindih. Jangan menghapus semua rule. Periksa `/system/logging/print`, identifikasi kategori yang dibutuhkan, kemudian rapikan rules secara terukur.

---

## 14. Monitoring Sistem Operasi Splunk Server Sendiri

Splunk dapat mencari log internal aplikasinya sendiri dan juga menerima log OS dari Ubuntu yang menjalankan Splunk. Kedua sumber tersebut berbeda.

### 14.1 Log internal Splunk

Operasional Splunk:

```spl
index=_internal
```

Audit aktivitas Splunk:

```spl
index=_audit
```

### 14.2 Log OS Ubuntu Splunk Server

Log OS yang ditambahkan sebagai input lokal:

```text
/var/log/auth.log
/var/log/syslog
```

Index tujuan:

```text
splunk_server_os
```

Konfigurasi dilakukan melalui Splunk Web:

1. Buat index `splunk_server_os` melalui **Settings → Indexes** jika belum ada.
2. Buka **Settings → Data inputs → Files & Directories**.
3. Pilih **New Local File & Directory**.
4. Tambahkan `/var/log/auth.log` dengan sourcetype `linux_secure` dan index `splunk_server_os`.
5. Tambahkan `/var/log/syslog` dengan sourcetype `syslog` dan index yang sama.
6. Gunakan host `splunk-server` bila field host tersedia pada form.
7. Simpan lalu validasi data.

### 14.3 Trouble permission yang ditemukan

Splunk Web sempat menolak path dengan pesan:

```text
Parameter name: Path is not readable
```

Pemeriksaan file menunjukkan file dimiliki `syslog:adm` dengan permission `640`, sementara proses Splunk berjalan sebagai user `splunk`.

Pemeriksaan yang dilakukan:

```bash
ls -l /var/log/auth.log /var/log/syslog
ps -eo user,group,comm | grep splunkd
```

Solusi yang digunakan adalah menambahkan user layanan `splunk` ke grup `adm`:

```bash
sudo usermod -aG adm splunk
```

Verifikasi grup:

```bash
id splunk
```

Verifikasi akses baca sebagai user service:

```bash
sudo -u splunk test -r /var/log/auth.log && echo "auth.log readable"
sudo -u splunk test -r /var/log/syslog && echo "syslog readable"
```

Hasil yang telah dilaporkan: user `splunk` tergabung dalam grup `adm` dan kedua file dapat dibaca.

### 14.4 Restart layanan dengan user yang benar

Perubahan supplementary group hanya akan terlihat oleh proses baru. Jika Splunk Web masih menyatakan path tidak dapat dibaca, periksa apakah proses `splunkd` yang sedang berjalan sudah memperoleh group membership terbaru.

```bash
ps -eo user,group,comm | grep splunkd
sudo /opt/splunk/bin/splunk status
```

Dalam troubleshooting, perintah berikut sempat dijalankan:

```bash
sudo /opt/splunk/bin/splunk restart
```

Perintah tersebut menampilkan peringatan bahwa menjalankan Splunk sebagai root sudah deprecated/tidak dianjurkan. Peringatan itu bukan bukti restart berhasil. **Jangan menjalankan Splunk sebagai root.** Gunakan mekanisme service yang sesuai dengan instalasi dan pastikan proses berjalan sebagai user `splunk`.

Setelah layanan berjalan dengan permission yang benar, ulangi proses menambahkan input lokal di Splunk Web.

### 14.5 Validasi input OS lokal

```spl
index=splunk_server_os
| stats count by host source sourcetype
```

Status input perlu dinilai dari event aktual yang muncul. Keberhasilan perintah `test -r` hanya membuktikan izin baca, bukan bahwa event sudah terindeks.

---

## 15. Validasi Data dengan SPL

Gunakan time range yang sesuai. Untuk event yang baru saja dibuat, pilih misalnya **Last 15 minutes** atau **Last 60 minutes**.

### 15.1 Ringkasan event per index dan sumber

```spl
(index=server_general OR index=caldera OR index=mikrotik OR index=splunk_server_os)
| stats count by index host source sourcetype
```

### 15.2 Event uji `server-general`

```spl
index=server_general "SOC_LAB_TEST"
```

### 15.3 Event uji Caldera

```spl
index=caldera "CALDERA_UF_TEST"
```

### 15.4 Event uji MikroTik

```spl
index=mikrotik "MIKROTIK_REMOTE_TEST_06"
```

### 15.5 Ringkasan data MikroTik

```spl
index=mikrotik
| stats count by host source sourcetype
```

### 15.6 Aktivitas login/logout MikroTik

```spl
index=mikrotik ("logged in" OR "logged out")
| sort - _time
```

### 15.7 Log autentikasi Linux

```spl
(index=server_general OR index=caldera OR index=splunk_server_os)
source="/var/log/auth.log"
| sort - _time
```

### 15.8 Log sistem Linux

```spl
(index=server_general OR index=caldera OR index=splunk_server_os)
source="/var/log/syslog"
| sort - _time
```

### 15.9 Pemeriksaan internal Splunk

```spl
index=_internal
| stats count by sourcetype source
```

### 15.10 Audit Splunk

```spl
index=_audit
| sort - _time
```

> Query di atas adalah pencarian awal. Detection rule perlu disusun setelah field, format event, volume, dan kualitas data dipahami. Jangan membuat alert produksi hanya berdasarkan satu event uji.

---

## 16. Prosedur Pemeriksaan Operasional

### 16.1 Splunk Enterprise

```bash
sudo /opt/splunk/bin/splunk status
sudo ss -lntp | grep -E ':8000|:9997'
sudo ss -lunp | grep ':5514'
df -h /
```

### 16.2 Universal Forwarder pada VM sumber

```bash
sudo /opt/splunkforwarder/bin/splunk status
sudo /opt/splunkforwarder/bin/splunk list forward-server
sudo /opt/splunkforwarder/bin/splunk list monitor
```

### 16.3 Caldera service

```bash
sudo systemctl status caldera
sudo journalctl -u caldera -n 100 --no-pager
sudo ss -lntp | grep ':8888'
```

### 16.4 MikroTik

```routeros
/interface print
/ip address print
/ip route print detail
/system logging print
/system logging action print
```

### 16.5 Sesudah reboot

Setelah reboot masing-masing VM:

1. Periksa status service.
2. Periksa listener port yang diperlukan.
3. Periksa status dan target Universal Forwarder.
4. Buat event uji baru dari sumbernya.
5. Pastikan event pascareboot terlihat di index yang benar.

Jangan menandai autostart berhasil hanya karena konfigurasi `enable` pernah dijalankan. Bukti yang paling berguna adalah service aktif dan event baru setelah reboot berhasil ditemukan.

---

## 17. Troubleshooting yang Ditemui

Bagian ini berisi masalah yang memang muncul saat pengerjaan dan cara penyelesaiannya. Pemeriksaan lain yang belum pernah terbukti bukan dicatat sebagai hasil perbaikan.

### 17.1 Disk Splunk penuh

**Gejala:** filesystem root kehabisan ruang; Splunk terhambat untuk berjalan/menulis data.

**Pemeriksaan:**

```bash
df -h
lsblk
sudo pvs
sudo vgs
sudo lvs
```

**Perbaikan yang dilakukan:** virtual disk diperbesar dari sekitar 10 GB menjadi 50 GB, kemudian partisi/PV/LV diperluas sesuai layout VM.

**Catatan:** perintah `growpart`/LVM harus disesuaikan dengan layout aktual. Jangan menjalankan command resize tanpa memeriksa disk dan LV yang benar.

### 17.2 Universal Forwarder tidak terlihat mengirim event

**Pemeriksaan utama:**

```bash
sudo /opt/splunkforwarder/bin/splunk status
sudo /opt/splunkforwarder/bin/splunk list forward-server
sudo /opt/splunkforwarder/bin/splunk list monitor
timeout 3 bash -c '</dev/tcp/192.168.90.100/9997' && echo OK || echo FAIL
```

Periksa receiving port TCP `9997`, IP tujuan, input monitor, index, dan time range. Buat event uji baru menggunakan `logger` untuk membuktikan jalur end-to-end.

### 17.3 Syslog MikroTik belum masuk

**Gejala:** event uji belum terlihat di Splunk.

**Langkah diagnosis:**

```bash
sudo tcpdump -ni any 'host 192.168.90.1 and udp port 5514'
```

Buat event uji dari MikroTik. Dalam pengerjaan, penambahan logging rule `warning` membuat paket mulai terlihat dan event uji berhasil ditemukan di Splunk.

### 17.4 Nama remote logging action RouterOS ditolak

Nama action menggunakan tanda hubung ditolak pada percobaan. Nama yang berhasil dipakai adalah:

```routeros
/system/logging/action/add name=splunktarget target=remote remote=192.168.90.100 remote-port=5514
```

### 17.5 Splunk tidak dapat membaca `/var/log/auth.log`

**Gejala:**

```text
Parameter name: Path is not readable
```

**Penyebab:** user `splunk` tidak memiliki akses baca terhadap file yang dimiliki grup `adm`.

**Solusi:**

```bash
sudo usermod -aG adm splunk
id splunk
sudo -u splunk test -r /var/log/auth.log && echo "auth.log readable"
sudo -u splunk test -r /var/log/syslog && echo "syslog readable"
```

Jika permission sudah benar namun Splunk Web masih menolak file, cek apakah proses layanan telah memperoleh grup tambahan. Restart harus dilakukan dengan user layanan yang benar, bukan root.

### 17.6 Search menghasilkan nol event

Jangan langsung mengubah seluruh konfigurasi. Periksa berurutan:

1. Index yang dicari benar.
2. Time range mencakup event.
3. Event baru memang dibuat.
4. File atau input terpantau.
5. Forwarder terhubung ke server.
6. Receiving port atau UDP input aktif.
7. Event tiba di Splunk dan diarahkan ke index yang sesuai.

### 17.7 Login failure belum muncul

Pencarian login failure sebelumnya belum menghasilkan event yang terkonfirmasi. Hal ini belum membuktikan konfigurasi rusak. Perlu dibuat percobaan gagal yang terkontrol pada router lab, lalu diperiksa apakah event dicatat secara lokal, diteruskan, dan diterima Splunk.

### 17.8 Event duplikat dari MikroTik

Periksa apakah beberapa logging rule bertumpang tindih:

```routeros
/system/logging/print
```

Rapikan hanya rule yang sudah dipahami dampaknya. Jangan menghapus seluruh aturan karena dapat menghentikan pengiriman kategori log yang masih dibutuhkan.

---

## 18. Backup dan Pemeliharaan Konfigurasi

### 18.1 Backup konfigurasi MikroTik

Simpan export konfigurasi secara aman:

```routeros
/export file=soc-lab-backup
```

File export dapat berisi informasi konfigurasi jaringan dan kredensial/secret tertentu tergantung metode dan versi. Jangan commit file backup mentah ke repository publik. Simpan pada lokasi privat dan sanitasi sebelum dibagikan.

### 18.2 Backup konfigurasi Splunk

Sebelum mengubah file konfigurasi manual, buat salinan file yang akan diedit. Contoh:

```bash
sudo cp -a /opt/splunk/etc/system/local /opt/splunk/etc/system/local.backup.$(date +%Y%m%d-%H%M%S)
```

Jangan mengedit file konfigurasi yang tidak diperlukan. Untuk konfigurasi input lokal Splunk Server pada tahap ini, input ditambahkan melalui Splunk Web.

### 18.3 Backup konfigurasi Universal Forwarder

Lokasi konfigurasi lokal:

```text
/opt/splunkforwarder/etc/system/local/
```

Sebelum perubahan:

```bash
sudo cp -a /opt/splunkforwarder/etc/system/local \
  "/opt/splunkforwarder/etc/system/local.backup.$(date +%Y%m%d-%H%M%S)"
```

### 18.4 Catatan perubahan

Setiap kali konfigurasi berubah, catat:

- Tanggal dan waktu.
- VM/perangkat yang diubah.
- File atau menu konfigurasi.
- Perintah yang dijalankan.
- Alasan perubahan.
- Hasil validasi sebelum dan sesudah perubahan.
- Cara rollback jika perubahan gagal.

---

## 19. Keamanan dan Batasan Lab

1. Jalankan Caldera dan aktivitas emulasi hanya pada VM/endpoint lab yang dimiliki atau diizinkan.
2. Jangan mengekspos Caldera UI/API ke internet publik tanpa kontrol akses yang ketat.
3. Ganti kredensial default dan lindungi akun admin Splunk serta Caldera.
4. Jangan menulis password, token, private key, secret, atau session cookie ke README.
5. Jangan commit file konfigurasi mentah yang berisi credential atau secret.
6. Batasi akses Splunk Web dan index log autentikasi ke admin yang berwenang.
7. Perhatikan bahwa `/var/log/auth.log` dan audit log dapat mengandung nama akun, alamat IP, serta informasi operasional sensitif.
8. Gunakan snapshot VMware sebelum perubahan besar atau percobaan yang berpotensi merusak VM.
9. Jangan menjalankan perintah resize disk/LVM tanpa memeriksa layout storage.
10. Dokumentasikan batasan sumber data agar hasil pencarian tidak dianggap sebagai telemetry lengkap.

---

## 20. Pekerjaan Berikutnya

### Tahap 1 — Memastikan fondasi tetap stabil

- Validasi bahwa index `splunk_server_os` menerima event terbaru dari `/var/log/auth.log` dan `/var/log/syslog`.
- Uji reboot Splunk Server, `server-general`, dan Caldera VM.
- Pastikan Splunk, Universal Forwarder, dan Caldera service aktif kembali setelah reboot.
- Buat event uji baru setelah reboot dan pastikan masuk ke index yang benar.
- Lengkapi pemetaan VMnet ke interface MikroTik dan IP aktual semua VM.

### Tahap 2 — Endpoint Windows

- Buat Windows VM pada segmen endpoint yang direncanakan.
- Aktifkan Windows audit policy yang sesuai kebutuhan.
- Tentukan Security Event Log yang relevan.
- Pasang Universal Forwarder Windows jika sesuai rancangan.
- Buat index terpisah untuk log Windows.
- Validasi event autentikasi dan event keamanan yang benar-benar dihasilkan.

### Tahap 3 — Detection engineering dasar

- Buat query untuk aktivitas login gagal setelah event sumber tersedia.
- Buat query untuk login sukses dan perubahan akun/router.
- Buat query untuk aktivitas firewall MikroTik.
- Validasi pola normal dan anomali menggunakan event uji.
- Catat false positive dan keterbatasan data.
- Buat alert setelah query tervalidasi.

### Tahap 4 — Skenario emulasi terkontrol

- Hubungkan Caldera dengan endpoint lab yang sesuai.
- Tentukan operasi yang aman dan tujuan pembelajaran.
- Pastikan endpoint merekam telemetry yang dibutuhkan.
- Jalankan satu skenario pada satu waktu.
- Cocokkan waktu, host, event, dan hasil aktivitas pada Splunk.
- Dokumentasikan teknik, bukti log, hasil deteksi, dan gap telemetry.

---

## 21. Checklist Implementasi

### A. VMware dan jaringan

- [x] VMware Workstation digunakan untuk menjalankan VM lab.
- [x] MikroTik CHR digunakan sebagai router virtual.
- [ ] Pemetaan VMnet setiap interface didokumentasikan lengkap.
- [ ] IP/prefix/gateway aktual seluruh VM dicatat.
- [ ] Route dan firewall antarsegmen diverifikasi sesuai kebutuhan.
- [ ] Akses Splunk Web dari host admin diuji setelah reboot.

### B. MikroTik

- [x] Interface dengan alamat segmen lab dikonfigurasi.
- [x] Remote logging action `splunktarget` dibuat.
- [x] Logging rules `system`, `account`, `firewall`, dan `warning` dibuat.
- [x] Event uji `MIKROTIK_REMOTE_TEST_06` ditemukan di Splunk.
- [ ] Login failure terkontrol menghasilkan event yang dapat dicari.
- [ ] Rules diperiksa dari duplikasi dan dirapikan bila diperlukan.

### C. Splunk Enterprise

- [x] Splunk Enterprise terpasang di `/opt/splunk`.
- [x] Splunk Web berjalan pada TCP `8000`.
- [x] Receiving TCP `9997` aktif.
- [x] Index `server_general`, `caldera`, dan `mikrotik` digunakan.
- [x] Disk VM diperbesar setelah kapasitas awal tidak mencukupi.
- [ ] Input lokal `splunk_server_os` divalidasi dengan event terbaru.
- [ ] Autostart setelah reboot diuji.

### D. Universal Forwarder

- [x] UF pada `server-general` mengirim event uji.
- [x] UF pada Caldera VM mengirim event uji.
- [x] `/var/log/syslog` dan `/var/log/auth.log` dipantau pada VM sumber.
- [ ] Autostart UF pada seluruh VM diuji setelah reboot.
- [ ] Event baru pascareboot ditemukan pada index masing-masing.

### E. Caldera

- [x] Caldera berjalan pada VM `red-tim`.
- [x] Python virtual environment digunakan.
- [x] systemd service `caldera` dibuat.
- [x] Port UI `8888` digunakan.
- [x] UF mengirim log OS Caldera ke index `caldera`.
- [ ] Akses UI dibatasi dan diverifikasi.
- [ ] Reboot test service dilakukan.
- [ ] Endpoint target dan telemetry-nya diintegrasikan.

### F. Pengembangan SOC

- [ ] Endpoint Windows dibuat.
- [ ] Windows Security Event Log masuk ke Splunk.
- [ ] Detection query login failure diuji.
- [ ] Detection query aktivitas akun diuji.
- [ ] Detection query perubahan konfigurasi MikroTik diuji.
- [ ] Alert dibuat setelah query tervalidasi.
- [ ] Skenario Caldera dikorelasikan dengan telemetry endpoint.

---

## 22. Referensi

- Splunk Enterprise: <https://www.splunk.com/en_us/download/splunk-enterprise.html>
- Splunk Universal Forwarder: <https://www.splunk.com/en_us/download/universal-forwarder.html>
- Dokumentasi Splunk: <https://docs.splunk.com/>
- Apache Caldera repository: <https://github.com/apache/caldera>
- Apache Caldera releases: <https://github.com/apache/caldera/releases>
- Ubuntu Server documentation: <https://documentation.ubuntu.com/server/>
- MikroTik RouterOS documentation: <https://help.mikrotik.com/docs/>
- VMware Workstation documentation: <https://docs.vmware.com/>

Periksa dokumentasi resmi sesuai versi yang benar-benar terpasang karena nama menu, opsi CLI, dependency, dan prosedur instalasi dapat berubah antarversi.

---

## Lampiran A — Lembar Inventaris

Lengkapi tabel ini dari konfigurasi aktual sebelum repository dijadikan dokumentasi final untuk publik.

| VM/perangkat | Hostname | OS/versi | IP/prefix | Gateway | VMnet/interface | Fungsi |
|---|---|---|---|---|---|---|
| MikroTik CHR | MikroTik | RouterOS 7.20.1 | Multi-interface | WAN DHCP | Catat VMnet tiap ether | Router lab |
| Splunk Server | `splunk-server` | Ubuntu Server | `192.168.90.100` | `192.168.90.1` (tercatat) | ether4/segmen 90 | SIEM |
| server-general | `server-general` | Ubuntu Server | Isi dari VM | Isi dari VM | ether5/segmen 99 (tercatat) | Linux log source |
| Caldera VM | `red-tim` | Ubuntu Server | Isi dari VM | Isi dari VM | Segmen 70 direncanakan | Caldera + UF |
| Windows endpoint | Belum dibuat | Belum ditentukan | Belum ditentukan | Belum ditentukan | Segmen 80 direncanakan | Endpoint monitoring |

### Perintah inventaris Ubuntu

```bash
hostnamectl
ip -br address
ip route
df -h
lsblk
```

### Perintah inventaris MikroTik

```routeros
/system identity print
/system resource print
/interface print
/ip address print
/ip route print detail
```

---

