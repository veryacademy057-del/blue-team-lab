# 🎯 Metasploitable 2 — Konfigurasi IP Static

> **Seri Lab:** Blue Team SOC Lab  
> **YouTube:** [▶️ Tonton Tutorial di YouTube](https://youtu.be/3DQyQVyRMII?si=Ql3UYyRMII)  
> **Tingkat Kesulitan:** `Pemula` | **OS:** `Metasploitable 2 (Ubuntu 8.04)` | **Peran:** `Vulnerable Target`

---

## 📋 Deskripsi

Lab ini mengkonfigurasi **Metasploitable 2** sebagai mesin target yang sengaja dibuat rentan (*intentionally vulnerable*) dalam jaringan SOC Lab. Metasploitable digunakan untuk simulasi serangan dari Kali Linux dan monitoring deteksi dari sisi Blue Team.

> ⚠️ **Peringatan:** Metasploitable adalah mesin yang **sengaja dibuat tidak aman**. Jangan pernah menghubungkannya ke internet atau jaringan publik. Gunakan hanya di jaringan internal/lab yang terisolasi.

---

## 🎬 Video Tutorial

[![Metasploitable 2 - Setup IP Static](https://img.youtube.com/vi/3DQyQVyRMII/maxresdefault.jpg)](https://youtu.be/3DQyQVyRMII?si=Ql3UQyQVyRMII)

> 📺 **[Tonton di YouTube → Metasploitable 2 Setup IP Static](https://youtu.be/3DQyQVyRMII?si=Ql3UQyQVyRMII)**

---

## 🗺️ Topologi Jaringan

```
                        Internet
                           ↓
                  [VirtualBox NAT]
                           ↓
              OPNsense Firewall (10.200.200.254)
                           ↓
            [Internal Network — intnet]
                           |
      ┌───────┬────────┬───┴────┬────────┬───────────┐
      ↓       ↓        ↓        ↓        ↓           ↓
  Win Server Win 10  Ubuntu   Kali   Metasploitable
   (.20)     (.30)   (.100)   (.10)     (.50)
    DC       Client   SIEM   Attacker  ← Kita di sini
```

---

## 📊 Tabel Alokasi IP Jaringan

| Perangkat | IP Address | Peran |
|-----------|------------|-------|
| **OPNsense** | `10.200.200.254` | Gateway / Firewall |
| **Windows Server 2012** | `10.200.200.20` | Domain Controller |
| **Windows 10 Pro** | `10.200.200.30` | Domain Client |
| **Ubuntu Server** | `10.200.200.100` | Splunk SIEM |
| **Kali Linux** | `10.200.200.10` | Mesin Penyerang |
| **Metasploitable 2** | `10.200.200.50` | **Target Rentan** |

---

## ⚙️ Langkah-Langkah Konfigurasi

### Langkah 1 — Konfigurasi Network di VirtualBox

Pastikan VM Metasploitable dalam keadaan **mati (OFF)** sebelum mengubah pengaturan.

```
VirtualBox → Pilih VM Metasploitable → Settings
→ Network → Adapter 1
```

Atur sebagai berikut:

| Parameter | Nilai |
|-----------|-------|
| Enable Network Adapter | ✅ Dicentang |
| Attached to | `Internal Network` |
| Name | `intnet` |

> ⚠️ **Penting:** Nama jaringan (`intnet`) harus **sama persis** dengan semua VM lain agar bisa saling terhubung.

---

### Langkah 2 — Jalankan VM dan Login

Start VM Metasploitable, lalu login dengan kredensial default:

```
Username : msfadmin
Password : msfadmin
```

Jika berhasil login, akan muncul shell prompt:

```
msfadmin@metasploitable:~$
```

---

### Langkah 3 — Cek Nama Network Interface

```bash
# Lihat semua interface
ifconfig

# Alternatif
ip a
```

Catat nama interface yang muncul. Pada Metasploitable 2, biasanya:

| Interface | Keterangan |
|-----------|------------|
| `eth0` | Paling umum di Metasploitable 2 |
| `ens33` | Jika menggunakan VMware |
| `enp0s3` | Versi Ubuntu lebih baru |

---

### Langkah 4 — Edit File Konfigurasi Jaringan

Metasploitable 2 berbasis Ubuntu 8.04 (versi lama), sehingga konfigurasi jaringan menggunakan file `/etc/network/interfaces`.

```bash
sudo nano /etc/network/interfaces
```

Password sudo: `msfadmin`

Hapus konfigurasi lama, lalu ganti dengan:

```bash
# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet static
    address 10.200.200.50
    netmask 255.255.255.0
    gateway 10.200.200.254
    dns-nameservers 10.200.200.20 8.8.8.8
```

> 💡 Ganti `eth0` dengan nama interface kamu jika berbeda (cek hasil Langkah 3).

Simpan file:

```
Ctrl + O   → Simpan
Enter      → Konfirmasi nama file
Ctrl + X   → Keluar dari nano
```

---

### Langkah 5 — Restart Jaringan

```bash
# Cara 1 — Restart service networking
sudo /etc/init.d/networking restart

# Cara 2 — Bring down lalu up interface
sudo ifdown eth0 && sudo ifup eth0

# Cara 3 — Reboot VM (paling aman)
sudo reboot
```

---

### Langkah 6 — Verifikasi IP Address

Setelah restart, login kembali dan cek IP:

```bash
# Cek IP address
ifconfig eth0

# Alternatif
ip addr show eth0
```

**Output yang diharapkan:**

```
eth0  Link encap:Ethernet  HWaddr xx:xx:xx:xx:xx:xx
      inet addr:10.200.200.50  Bcast:10.200.200.255  Mask:255.255.255.0
      ...
```

> ✅ Jika `inet addr:10.200.200.50` muncul, konfigurasi **berhasil!**

---

### Langkah 7 — Uji Koneksi Jaringan

```bash
# Ping ke gateway (OPNsense)
ping -c 4 10.200.200.254

# Ping ke Domain Controller
ping -c 4 10.200.200.20

# Ping ke Ubuntu/Splunk
ping -c 4 10.200.200.100

# Ping ke internet
ping -c 4 8.8.8.8

# Test DNS
ping -c 4 google.com
```

> ✅ Semua ping harus berhasil sebelum melanjutkan.

---

### Langkah 8 — Uji dari Mesin Lain

**Dari Kali Linux:**

```bash
# Ping ke Metasploitable
ping -c 4 10.200.200.50

# Scan port (akan muncul banyak port terbuka — ini normal!)
nmap 10.200.200.50
```

**Dari Windows 10 / Windows Server:**

```powershell
# Ping ke Metasploitable
Test-NetConnection -ComputerName 10.200.200.50
```

> ✅ `PingSucceeded : True` artinya Metasploitable sudah terhubung ke jaringan lab.

---

### Langkah 9 — Verifikasi Services yang Berjalan

```bash
# Cek semua port yang sedang listen
netstat -tuln | grep LISTEN
```

**Port yang seharusnya terbuka di Metasploitable 2:**

| Port | Protokol | Layanan |
|------|----------|---------|
| `21` | TCP | FTP (vsftpd 2.3.4 — backdoor!) |
| `22` | TCP | SSH (OpenSSH) |
| `23` | TCP | Telnet |
| `25` | TCP | SMTP |
| `80` | TCP | HTTP (Apache) |
| `139` | TCP | NetBIOS |
| `445` | TCP | SMB (Samba) |
| `3306` | TCP | MySQL |
| `5432` | TCP | PostgreSQL |
| `8180` | TCP | Apache Tomcat |

---

### Langkah 10 — Akses via Browser

Dari Kali Linux atau Windows, buka browser dan akses:

```
http://10.200.200.50
```

Halaman web Metasploitable akan muncul berisi berbagai **vulnerable web application** untuk latihan:

- **DVWA** (Damn Vulnerable Web App)
- **Mutillidae**
- **phpMyAdmin**
- **TWiki**

---

## 🛠️ Troubleshooting

### Masalah 1 — IP Tidak Tersimpan Setelah Edit

```bash
# Cek isi file konfigurasi
cat /etc/network/interfaces

# Set IP secara manual (sementara)
sudo ifconfig eth0 10.200.200.50 netmask 255.255.255.0
sudo route add default gw 10.200.200.254

# Paksa restart interface
sudo ifconfig eth0 down
sudo ifconfig eth0 up
```

---

### Masalah 2 — Tidak Bisa Ping Gateway

```bash
# Cek routing table
route -n
```

Harus ada baris dengan `Destination: 0.0.0.0` dan `Gateway: 10.200.200.254`. Jika tidak ada:

```bash
sudo route add default gw 10.200.200.254
```

---

### Masalah 3 — Network Interface Tidak Ditemukan

```bash
# Lihat semua interface (termasuk yang down)
ifconfig -a

# Atau
ip link show

# Jika ada tapi statusnya down, aktifkan
sudo ifconfig eth0 up
```

---

### Alternatif — Set IP Sementara (Tanpa Edit File)

Jika mengalami kesulitan mengedit file, gunakan cara ini. **IP akan hilang setelah reboot.**

```bash
# Set IP address
sudo ifconfig eth0 10.200.200.50 netmask 255.255.255.0

# Set default gateway
sudo route add default gw 10.200.200.254

# Set DNS
echo "nameserver 10.200.200.20" | sudo tee /etc/resolv.conf
echo "nameserver 8.8.8.8" | sudo tee -a /etc/resolv.conf

# Verifikasi
ifconfig eth0
route -n
```

## 📚 Referensi

- [Metasploitable 2 Official Documentation](https://docs.rapid7.com/metasploit/metasploitable-2/)
- [Metasploitable 2 Exploitability Guide](https://docs.rapid7.com/metasploit/metasploitable-2-exploitability-guide/)
- [DVWA Documentation](https://dvwa.co.uk/)

---

*📺 Ikuti tutorialnya di [YouTube](https://youtu.be/3DQyQVyRMII?si=Ql3UQyQVyRMII) | ⭐ Star repo ini jika membantu!*
