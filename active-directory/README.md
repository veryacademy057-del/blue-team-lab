# 🏛️ Active Directory Domain Controller — Lab Setup

> **Seri Lab:** Blue Team SOC Lab  
> **YouTube:** [▶️ Tonton Tutorial di YouTube](https://youtu.be/MsW7DhdrsP4?si=avd6Xqz9US__K-h1)  
> **Tingkat Kesulitan:** `Menengah` | **OS:** `Windows Server 2012 + Windows 10 Pro` | **Domain:** `soclabs.local`

---

## 📋 Deskripsi

Lab ini bertujuan membangun **Active Directory Domain Controller** menggunakan Windows Server dan menghubungkan Windows 10 sebagai client ke dalam domain `soclabs.local`. Ini adalah fondasi utama dari SOC Lab — semua autentikasi dan manajemen user akan terpusat di sini.

---

## 🎬 Video Tutorial

[![Active Directory Lab - soclabs.local](https://img.youtube.com/vi/MsW7DhdrsP4/maxresdefault.jpg)](https://youtu.be/MsW7DhdrsP4?si=avd6Xqz9US__K-h1)

> 📺 **[Tonton di YouTube → Active Directory Domain Controller Lab](https://youtu.be/MsW7DhdrsP4?si=avd6Xqz9US__K-h1)**

---

## 🧱 Arsitektur Lab

```
┌─────────────────────────────────────────────────────────┐
│                   JARINGAN: soc-net                     │
│                  (Internal Network)                     │
│                                                         │
│   ┌──────────────────────┐   ┌──────────────────────┐  │
│   │   Windows Server     │   │    Windows 10 Pro    │  │
│   │   2012               │   │    (Client/Agent)    │  │
│   │                      │   │                      │  │
│   │  IP: 10.200.200.20   │   │  IP: 10.200.200.30   │  │
│   │  Role: DC + AD DS    │   │  Domain: soclabs\... │  │
│   └──────────────────────┘   └──────────────────────┘  │
│              │                          │               │
│              └──────────┬───────────────┘               │
│                         │                               │
│              ┌──────────────────┐                       │
│              │  OPNsense        │                       │
│              │  Gateway         │                       │
│              │  10.200.200.254  │                       │
│              └──────────────────┘                       │
└─────────────────────────────────────────────────────────┘
```

### 🗂️ Ringkasan Konfigurasi

| Komponen | Nilai |
|----------|-------|
| **Domain** | `soclabs.local` |
| **Windows Server IP** | `10.200.200.20` |
| **Windows 10 IP** | `10.200.200.30` |
| **Gateway** | `10.200.200.254` (OPNsense) |
| **DNS Utama** | `10.200.200.20` (Domain Controller) |
| **DNS Cadangan** | `8.8.8.8` (Google) |
| **Jaringan** | Internal Network (`soc-net`) |

---

## ⚙️ Bagian 1 — Konfigurasi Windows Server (Domain Controller)

### 1.1 — Set IP Static

Masuk ke pengaturan jaringan:

```
Control Panel → Network and Sharing Center
→ Change adapter settings
→ Klik kanan adapter → Properties → IPv4
```

Isi konfigurasi berikut:

| Parameter | Nilai |
|-----------|-------|
| IP Address | `10.200.200.20` |
| Subnet Mask | `255.255.255.0` |
| Default Gateway | `10.200.200.254` |
| DNS Utama | `8.8.8.8` |
| DNS Cadangan | `8.8.4.4` |

> 💡 **Catatan:** DNS akan diubah ke `127.0.0.1` setelah AD DS terinstall, karena server ini akan menjadi DNS server untuk domain.

---

### 1.2 — Install Active Directory Domain Services (AD DS)

Buka Server Manager dan install role AD DS:

```
Server Manager → Manage → Add Roles and Features
```

Ikuti langkah-langkah berikut:

```
1. Before You Begin         → Next
2. Installation Type        → Role-based or feature-based installation → Next
3. Server Selection         → Pilih server lokal → Next
4. Server Roles             → ✅ Active Directory Domain Services → Next
5. Features                 → Next (biarkan default)
6. AD DS                    → Next
7. Confirmation             → Install
```

> ⏳ Tunggu proses instalasi selesai. Jangan restart dulu.

---

### 1.3 — Promote Server menjadi Domain Controller

Setelah instalasi selesai, klik notifikasi di Server Manager:

```
🔔 Notification (tanda seru kuning) → Promote this server to a domain controller
```

Pada wizard yang muncul, isi sebagai berikut:

**Deployment Configuration:**
```
Pilih: Add a new forest
Root domain name: soclabs.local
```

**Domain Controller Options:**
```
Forest functional level : Windows Server 2012 R2
Domain functional level : Windows Server 2012 R2
✅ Domain Name System (DNS) server
✅ Global Catalog (GC)

DSRM Password: (buat password yang kuat, catat baik-baik!)
```

**Lanjutkan hingga selesai:**
```
DNS Options      → Next (abaikan warning)
Additional Options → NetBIOS name: SOCLABS → Next
Paths            → Next (biarkan default)
Review Options   → Next
Prerequisites    → Install (jika semua hijau ✅)
```

> 🔄 Server akan **restart otomatis** setelah proses selesai. Ini normal.

---

### 1.4 — Verifikasi Domain Controller

Setelah restart, login menggunakan akun domain:

```
Username: soclabs\Administrator
Password: (password yang dibuat saat instalasi)
```

Buka **Active Directory Users and Computers** untuk verifikasi:

```
Server Manager → Tools → Active Directory Users and Computers
```

Pastikan struktur berikut muncul:

```
soclabs.local
├── Builtin
├── Computers
├── Domain Controllers
│   └── WIN-SERVER (komputer ini)
└── Users
    ├── Administrator
    └── (user lainnya)
```

---

### 1.5 — Membuat User Domain

Buat akun user baru untuk digunakan login dari client:

```
Active Directory Users and Computers
→ soclabs.local → Users
→ Klik kanan → New → User
```

Isi form user:

| Field | Nilai |
|-------|-------|
| First Name | `Hunter` |
| Last Name | `Two` |
| User logon name | `hunter2` |
| Password | `(buat password kuat)` |

Setting password:
```
✅ User must change password at next logon  → (opsional)
✅ Password never expires                   → (untuk lab)
```

> ✅ User `soclabs\hunter2` siap digunakan untuk login dari Windows 10.

---

## 💻 Bagian 2 — Konfigurasi Windows 10 (Client)

### 2.1 — Set IP Static & DNS

Arahkan DNS ke Domain Controller agar bisa resolve domain `soclabs.local`:

```
Control Panel → Network and Sharing Center
→ Change adapter settings
→ Klik kanan adapter → Properties → IPv4
```

| Parameter | Nilai |
|-----------|-------|
| IP Address | `10.200.200.30` |
| Subnet Mask | `255.255.255.0` |
| Default Gateway | `10.200.200.254` |
| DNS Utama | `10.200.200.20` ← IP Domain Controller |
| DNS Cadangan | `8.8.8.8` |

> ⚠️ **Penting:** DNS Utama **harus** diisi IP Domain Controller (`10.200.200.20`), bukan `8.8.8.8`. Tanpa ini, Windows 10 tidak bisa menemukan domain `soclabs.local`.

---

### 2.2 — Cek Koneksi ke Domain Controller

Buka **Command Prompt** dan jalankan:

```cmd
:: Test koneksi ke gateway
ping 10.200.200.254

:: Test koneksi ke Domain Controller
ping 10.200.200.20

:: Test resolusi domain (PALING PENTING)
nslookup soclabs.local
```

**Output `nslookup` yang diharapkan:**
```
Server:   soclabs.local
Address:  10.200.200.20

Name:     soclabs.local
Address:  10.200.200.20
```

> ❌ Jika `nslookup` gagal, periksa kembali pengaturan DNS di langkah 2.1.

---

### 2.3 — Join ke Domain `soclabs.local`

Buka System Properties:

```
Windows + R → ketik: sysdm.cpl → Enter
```

Navigasi ke:
```
Tab: Computer Name → Change...
```

Ubah pengaturan:
```
✅ Member of: Domain
   soclabs.local
```

Klik **OK** → Masukkan kredensial Administrator domain:

```
Username: Administrator
Password: (password DC)
```

Jika berhasil, akan muncul pesan:

```
✅ Welcome to the soclabs.local domain.
```

> 🔄 **Restart Windows 10** setelah berhasil join domain.

---

### 2.4 — Login dengan Akun Domain

Setelah restart, pada layar login:

```
Pilih: Other user
```

Login menggunakan akun domain yang dibuat sebelumnya:

```
Username: soclabs\hunter2
Password: (password hunter2)
```

---

### 2.5 — Verifikasi Login Domain

Buka **Command Prompt** dan jalankan:

```cmd
whoami
```

**Output yang diharapkan:**
```
soclabs\hunter2
```

Cek informasi lengkap user:

```cmd
whoami /all
net user hunter2 /domain
```

> ✅ Jika output menampilkan `soclabs\hunter2`, artinya Windows 10 sudah berhasil bergabung dan login ke domain.

---

## 🧪 Bagian 3 — Pengujian Lengkap

Jalankan semua perintah berikut dari **Windows 10 (client)** untuk memastikan semua berjalan normal:

```cmd
:: 1. Cek identitas user saat ini
whoami

:: 2. Test koneksi ke domain
ping soclabs.local

:: 3. Test resolusi DNS domain
nslookup soclabs.local

:: 4. Cek informasi komputer dalam domain
net computer

:: 5. Cek daftar user domain (dari DC)
net user /domain
```

### Hasil yang Diharapkan

| Pengujian | Perintah | Hasil Normal |
|-----------|----------|--------------|
| Identitas user | `whoami` | `soclabs\hunter2` |
| Koneksi domain | `ping soclabs.local` | Reply dari `10.200.200.20` |
| Resolusi DNS | `nslookup soclabs.local` | Address: `10.200.200.20` |
| Login domain | Login screen | Berhasil masuk desktop |

---

## 🔐 Bagian 4 — Monitoring Login (Opsional)

Pantau aktivitas login dari **Domain Controller** menggunakan Event Viewer:

```
Server Manager → Tools → Event Viewer
→ Windows Logs → Security
```

### Event ID Penting

| Event ID | Keterangan | Tingkat |
|----------|------------|---------|
| `4624` | Login berhasil | ℹ️ Info |
| `4625` | Login gagal (password salah) | ⚠️ Warning |
| `4634` | User logout | ℹ️ Info |
| `4648` | Login dengan kredensial eksplisit | ⚠️ Perhatikan |
| `4768` | Kerberos ticket diminta (TGT) | ℹ️ Info |
| `4769` | Kerberos service ticket diminta | ℹ️ Info |

> 💡 **Tips SOC:** Event ID `4625` yang muncul berulang kali dalam waktu singkat bisa mengindikasikan **brute force attack**. Ini yang akan kita monitor di lab Wazuh SIEM nanti.

---

## 🛠️ Troubleshooting

| Masalah | Kemungkinan Penyebab | Solusi |
|---------|----------------------|--------|
| Tidak bisa join domain | DNS salah di Windows 10 | Set DNS ke `10.200.200.20` |
| `nslookup` gagal | DC tidak terjangkau | Cek ping ke `10.200.200.20` |
| Login domain gagal | Password salah / user belum dibuat | Cek di AD Users and Computers |
| Domain tidak ditemukan | Firewall memblokir | Nonaktifkan firewall sementara untuk test |
| Server tidak bisa promote | Nama domain salah format | Gunakan format `nama.local` (harus ada titik) |
| Komputer tidak muncul di AD | Belum restart setelah join | Restart Windows 10 |

---

## 📌 Kesimpulan

Setelah mengikuti lab ini, kamu telah berhasil:

- ✅ Menginstall dan mengkonfigurasi **Active Directory Domain Services**
- ✅ Mempromosikan Windows Server menjadi **Domain Controller**
- ✅ Membuat domain baru `soclabs.local`
- ✅ Menghubungkan Windows 10 ke domain
- ✅ Membuat dan menggunakan **akun domain** untuk login
- ✅ Memahami dasar **monitoring login** via Event Viewer

---

## 🚀 Pengembangan Selanjutnya

Lab ini adalah fondasi. Berikut langkah selanjutnya dalam seri ini:

- [ ] `Lab 03` — Integrasi Wazuh SIEM untuk monitoring log domain
- [ ] `Lab 04` — Konfigurasi Sysmon untuk enhanced logging
- [ ] `Lab 05` — Simulasi serangan dengan Kali Linux
- [ ] `Lab 06` — Deteksi brute force via Event ID 4625

---

## 📚 Referensi

- [Microsoft Docs — AD DS Installation](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/deploy/install-active-directory-domain-services)
- [Microsoft Docs — Join Computer to Domain](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/virtual-dc/active-directory-domain-services-overview)
- [Windows Security Event ID Reference](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/)

---

*📺 Ikuti tutorialnya di [YouTube](https://youtu.be/MsW7DhdrsP4?si=avd6Xqz9US__K-h1) | ⭐ Star repo ini jika membantu!*
