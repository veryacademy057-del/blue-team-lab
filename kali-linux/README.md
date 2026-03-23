# ⚔️ Kali Linux — Konfigurasi IP Static ke Jaringan SOC Lab

> **Seri Lab:** Blue Team SOC Lab  
> **YouTube:** [▶️ Tonton Tutorial di YouTube](https://youtu.be/3DQyQVyRMII?si=PYh0Zjf5bYSS6GW3)  
> **Tingkat Kesulitan:** `Pemula` | **OS:** `Kali Linux` | **Peran:** `Mesin Penyerang`

---

## 📋 Deskripsi

Lab ini mengkonfigurasi **Kali Linux** agar terhubung ke jaringan internal SOC Lab (`intnet`) dengan IP static `10.200.200.10`. Kali Linux berperan sebagai mesin penyerang (*attacker machine*) yang digunakan untuk simulasi serangan terhadap target di jaringan lab.

---

## 🎬 Video Tutorial

[![Kali Linux - Konfigurasi IP Static SOC Lab](https://img.youtube.com/vi/3DQyQVyRMII/maxresdefault.jpg)](https://youtu.be/3DQyQVyRMII?si=PYh0Zjf5bYSS6GW3)

> 📺 **[Tonton di YouTube → Kali Linux Konfigurasi IP Static SOC Lab](https://youtu.be/3DQyQVyRMII?si=PYh0Zjf5bYSS6GW3)**

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
      ┌───────┬────────┬───┴────┬────────────────────┐
      ↓       ↓        ↓        ↓                    ↓
  Win Server Win 10  Ubuntu  Metasploitable       Kali Linux
   (.20)     (.30)   (.100)    (.50)               (.10)
    DC       Client   SIEM     Target         ← Kita di sini
```

---

## 📊 Tabel Alokasi IP Jaringan

| Perangkat | IP Address | Peran |
|-----------|------------|-------|
| **OPNsense** | `10.200.200.254` | Gateway / Firewall |
| **Windows Server 2012** | `10.200.200.20` | Domain Controller |
| **Windows 10 Pro** | `10.200.200.30` | Domain Client |
| **Ubuntu Server** | `10.200.200.100` | Splunk SIEM |
| **Metasploitable 2** | `10.200.200.50` | Target Rentan |
| **Kali Linux** | `10.200.200.10` | **Mesin Penyerang** |

---

## ⚙️ Langkah-Langkah Konfigurasi

### Langkah 1 — Konfigurasi Network di VirtualBox

Pastikan VM Kali Linux dalam keadaan **mati (OFF)** sebelum mengubah pengaturan.

```
VirtualBox → Pilih VM Kali Linux → Settings
→ Network → Adapter 1
```

| Parameter | Nilai |
|-----------|-------|
| Enable Network Adapter | ✅ Dicentang |
| Attached to | `Internal Network` |
| Name | `intnet` |

> ⚠️ **Penting:** Nama jaringan (`intnet`) harus **sama persis** dengan semua VM lain di lab.

---

### Langkah 2 — Cek Nama Network Interface

Buka terminal di Kali Linux dan jalankan:

```bash
ip a
```

Catat nama interface yang muncul:

| Interface | Keterangan |
|-----------|------------|
| `eth0` | Umum di VirtualBox |
| `ens33` | Umum di VMware |
| `enp0s3` | Versi kernel tertentu |

---

### Langkah 3 — Konfigurasi IP Static

Kali Linux modern menggunakan **NetworkManager** untuk mengelola jaringan — bukan file `/etc/network/interfaces` (file ini kosong, itu **normal**).

Ada dua cara konfigurasi, pilih salah satu:

---

#### Cara A — Menggunakan `nmcli` (Terminal)

```bash
# Langkah 1 — Cek nama koneksi yang tersedia
nmcli con show

# Langkah 2 — Set IP static (ganti nama koneksi jika berbeda)
sudo nmcli con mod "Wired connection 1" ipv4.addresses 10.200.200.10/24
sudo nmcli con mod "Wired connection 1" ipv4.gateway 10.200.200.254
sudo nmcli con mod "Wired connection 1" ipv4.dns "8.8.8.8 8.8.4.4"
sudo nmcli con mod "Wired connection 1" ipv4.method manual

# Langkah 3 — Restart koneksi untuk menerapkan perubahan
sudo nmcli con down "Wired connection 1"
sudo nmcli con up "Wired connection 1"
```

> 💡 Jika nama koneksi bukan `"Wired connection 1"`, sesuaikan dengan hasil `nmcli con show`.

---

#### Cara B — Menggunakan GUI (Lebih Mudah)

```
1. Klik icon jaringan di taskbar Kali Linux
2. Pilih → Edit Connections (atau Advanced Network Configuration)
3. Pilih koneksi → Klik icon Edit (roda gigi ⚙️)
4. Buka tab → IPv4 Settings
5. Ubah Method → Manual
6. Klik Add, lalu isi:
   - Address  : 10.200.200.10
   - Netmask  : 255.255.255.0  (atau /24)
   - Gateway  : 10.200.200.254
7. DNS servers : 8.8.8.8, 8.8.4.4
8. Klik Save
9. Toggle koneksi OFF → ON
```

---

### Langkah 4 — Restart Network Service

```bash
# Cara 1 — Restart NetworkManager
sudo systemctl restart NetworkManager

# Cara 2 — Restart networking
sudo systemctl restart networking
```

---

### Langkah 5 — Verifikasi Konfigurasi

```bash
# Cek IP address yang terpasang
ip a

# Cek routing table (harus ada default via 10.200.200.254)
ip route

# Cek DNS resolver
cat /etc/resolv.conf
```

**Output `ip route` yang diharapkan:**

```
default via 10.200.200.254 dev eth0
10.200.200.0/24 dev eth0 proto kernel scope link src 10.200.200.10
```

---

### Langkah 6 — Uji Koneksi Jaringan

```bash
# Ping ke gateway OPNsense
ping -c 4 10.200.200.254

# Ping ke internet
ping -c 4 8.8.8.8

# Test resolusi DNS
ping -c 4 google.com

# Ping ke mesin lain di lab
ping -c 4 10.200.200.20   # Windows Server
ping -c 4 10.200.200.50   # Metasploitable
```

> ✅ Semua ping harus berhasil sebelum melanjutkan ke lab berikutnya.

---

### Langkah 7 — Verifikasi di OPNsense (Opsional)

Login ke web interface OPNsense dari Kali Linux:

```
http://10.200.200.254
```

Periksa:

```
Firewall → Rules → LAN
→ Pastikan ada rule: "Default allow LAN to any rule"
```

> Rule ini mengizinkan semua traffic dari jaringan internal ke internet secara default.

---

## 🛠️ Troubleshooting

### Masalah 1 — Tidak Bisa Ping Gateway

```bash
# Cek routing table
ip route

# Jika tidak ada default route, tambahkan manual
sudo ip route add default via 10.200.200.254

# Cek kembali
ping -c 4 10.200.200.254
```

---

### Masalah 2 — DNS Tidak Bekerja

```bash
# Cek isi resolv.conf
cat /etc/resolv.conf

# Jika kosong atau salah, set manual
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
echo "nameserver 8.8.4.4" | sudo tee -a /etc/resolv.conf

# Test DNS
nslookup google.com
```

---

### Masalah 3 — Interface Tidak Ditemukan atau Down

```bash
# Lihat semua interface termasuk yang non-aktif
ip link show

# Aktifkan interface
sudo ip link set eth0 up

# Atau via nmcli
sudo nmcli con up "Wired connection 1"
```

---

### Masalah 4 — IP Tidak Tersimpan Setelah Reboot

Pastikan menggunakan **nmcli** (bukan `ifconfig` manual) karena perubahan via `nmcli` bersifat permanen. Perubahan via `ifconfig` bersifat sementara dan hilang setelah reboot.

---

## 📌 Kesimpulan

Setelah mengikuti lab ini, kamu telah berhasil:

- ✅ Menghubungkan Kali Linux ke jaringan internal SOC Lab
- ✅ Mengkonfigurasi IP static menggunakan NetworkManager
- ✅ Memverifikasi koneksi ke seluruh node di jaringan lab
- ✅ Kali Linux siap digunakan sebagai mesin penyerang dalam simulasi

---

## 📚 Referensi

- [Kali Linux Network Configuration Docs](https://www.kali.org/docs/general-use/network-manager/)
- [nmcli Command Reference](https://networkmanager.dev/docs/api/latest/nmcli.html)
- [OPNsense Firewall Rules Documentation](https://docs.opnsense.org/manual/firewall.html)

---

*📺 Ikuti tutorialnya di [YouTube](https://youtu.be/3DQyQVyRMII?si=PYh0Zjf5bYSS6GW3) | ⭐ Star repo ini jika membantu!*
