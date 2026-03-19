# 📊 Windows Log Monitoring — Splunk + Sysmon

> **Seri Lab:** Blue Team SOC Lab  
> **Tingkat Kesulitan:** `Menengah` | **OS:** `Windows Server 2012 + Windows 10 + Ubuntu Server` | **Tools:** `Splunk, Sysmon, Universal Forwarder`

---

## 📋 Deskripsi

Lab ini mengkonfigurasi **Splunk Universal Forwarder** di Windows Server dan Windows 10 untuk mengirimkan log ke **Splunk Server** yang berjalan di Ubuntu Server. Ditambah dengan **Sysmon** untuk logging yang lebih detail di level sistem operasi. Tujuannya adalah membangun kemampuan monitoring dan deteksi ancaman dari sisi Blue Team.

---

## 🗺️ Topologi Lab

```
┌─────────────────────────────────────────────────────────────┐
│                    Internal Network (intnet)                 │
│                                                             │
│  ┌──────────────────┐        ┌──────────────────┐          │
│  │  Windows Server  │        │   Windows 10     │          │
│  │  10.200.200.20   │        │   10.200.200.30  │          │
│  │                  │        │                  │          │
│  │  [UF Agent]      │        │  [UF Agent]      │          │
│  │  Port 9997 →     │        │  Port 9997 →     │          │
│  └────────┬─────────┘        └────────┬─────────┘          │
│           │                           │                     │
│           └─────────────┬─────────────┘                     │
│                         │ Log Stream                        │
│                         ↓                                   │
│              ┌──────────────────────┐                       │
│              │    Ubuntu Server     │                       │
│              │    10.200.200.100    │                       │
│              │                     │                       │
│              │   [Splunk Server]   │                       │
│              │   Port 9997 (input) │                       │
│              │   Port 8000 (web)   │                       │
│              └──────────────────────┘                       │
└─────────────────────────────────────────────────────────────┘
```

---

## 📦 Komponen yang Diinstall

| Komponen | Mesin | Fungsi |
|----------|-------|--------|
| **Splunk Enterprise** | Ubuntu Server (`10.200.200.100`) | Menerima, menyimpan, dan menganalisis log |
| **Splunk Universal Forwarder** | Windows Server + Windows 10 | Mengirim log ke Splunk Server |
| **Sysmon** | Windows Server + Windows 10 | Logging aktivitas sistem yang detail |

---

## ⚙️ Bagian 1 — Install Splunk Universal Forwarder di Windows

Universal Forwarder adalah agent ringan yang bertugas mengumpulkan dan mengirimkan log ke Splunk Server.

### 1.1 — Download Splunk Universal Forwarder

Buka browser di Windows dan akses:

```
https://www.splunk.com/en_us/download/universal-forwarder.html
```

Pilih:
- Platform: **Windows**
- Arsitektur: **64-bit**
- File: `splunkforwarder-9.x.x-x64-release.msi`

---

### 1.2 — Install di Windows Server 2012

Double-click file `.msi` yang sudah didownload, lalu ikuti wizard:

```
1. License Agreement     → Accept & Next
2. Customize Options     → Next
3. Credentials           → Username: admin
                           Password: [buat password, catat!]
4. Deployment Server     → Kosongkan → Next
5. Receiving Indexer     → Host: 10.200.200.100
                           Port: 9997
6. Install               → Tunggu proses selesai
7. Finish
```

> 💡 **Receiving Indexer** adalah alamat Splunk Server (Ubuntu). Port `9997` adalah port default untuk penerimaan data dari forwarder.

---

### 1.3 — Install di Windows 10

Ulangi langkah yang **sama persis** seperti pada Windows Server di atas. Pastikan Receiving Indexer tetap mengarah ke `10.200.200.100:9997`.

---

## ⚙️ Bagian 2 — Konfigurasi Splunk Server untuk Menerima Data

### 2.1 — Via Web Interface

Login ke Splunk Web di Ubuntu Server:

```
http://10.200.200.100:8000
```

Navigasi ke:

```
Settings → Forwarding and receiving
→ Configure receiving → New Receiving Port
→ Port: 9997
→ Save
```

---

### 2.2 — Via Command Line (Ubuntu Server)

```bash
cd /opt/splunk/bin

# Aktifkan port penerima
sudo ./splunk enable listen 9997 -auth admin:passwordkamu

# Restart Splunk
sudo ./splunk restart
```

---

## ⚙️ Bagian 3 — Konfigurasi Sumber Data (Windows Event Logs)

Jalankan perintah berikut di **masing-masing Windows** (Server dan Windows 10) via **PowerShell sebagai Administrator**.

### 3.1 — Daftarkan Splunk Server sebagai Tujuan

```powershell
cd "C:\Program Files\SplunkUniversalForwarder\bin"

.\splunk add forward-server 10.200.200.100:9997 -auth admin:changeme
```

---

### 3.2 — Tambahkan Windows Event Logs

```powershell
# Security logs — Login, akses, privilege
.\splunk add monitor "WinEventLog://Security" -index main -sourcetype WinEventLog:Security

# System logs — Hardware, driver, service
.\splunk add monitor "WinEventLog://System" -index main -sourcetype WinEventLog:System

# Application logs — Error & info dari aplikasi
.\splunk add monitor "WinEventLog://Application" -index main -sourcetype WinEventLog:Application

# Restart Universal Forwarder
.\splunk restart
```

> ⚠️ Ulangi langkah ini di **Windows 10** juga.

---

## ⚙️ Bagian 4 — Install Sysmon (Advanced Logging)

Sysmon (*System Monitor*) adalah tool dari Microsoft Sysinternals yang mencatat aktivitas sistem secara sangat detail — proses yang berjalan, koneksi jaringan, perubahan registry, dan lainnya.

### 4.1 — Download Sysmon

Buka browser dan akses:

```
https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon
```

Download `Sysmon.zip`, lalu extract ke `C:\Sysmon`.

---

### 4.2 — Download Konfigurasi Sysmon (SwiftOnSecurity)

```powershell
# Download konfigurasi yang sudah teruji dan digunakan secara luas
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" -OutFile "C:\sysmonconfig.xml"
```

---

### 4.3 — Install Sysmon

```powershell
# Masuk ke folder Sysmon
cd C:\Sysmon

# Install dengan konfigurasi SwiftOnSecurity
.\Sysmon64.exe -accepteula -i C:\sysmonconfig.xml
```

---

### 4.4 — Daftarkan Sysmon ke Universal Forwarder

```powershell
cd "C:\Program Files\SplunkUniversalForwarder\bin"

# Tambahkan Sysmon log sebagai sumber data
.\splunk.exe add monitor "WinEventLog://Microsoft-Windows-Sysmon/Operational" `
  -index main `
  -sourcetype XmlWinEventLog:Microsoft-Windows-Sysmon/Operational `
  -auth admin:changeme

# Restart forwarder
.\splunk.exe restart
```

> ⚠️ Ulangi langkah 4.1 – 4.4 di **Windows 10** juga.

---

## ✅ Bagian 5 — Verifikasi Forwarder Terkoneksi

Di Splunk Web Interface:

```
Settings → Forwarding and receiving
→ Configure receiving → Klik port 9997
→ Lihat bagian "Forwarders connected"
```

Seharusnya muncul dua forwarder:

```
✅ WIN-SERVER-DC
✅ WIN10-CLIENT
```

---

## 🔍 Bagian 6 — Search & Query di Splunk

### 6.1 — Cek Log Masuk

Login ke Splunk Web, klik **Search & Reporting**, lalu jalankan query:

```splunk
index=main sourcetype=WinEventLog:*
```

Jika log muncul, artinya forwarder sudah bekerja dengan benar. ✅

---

### 6.2 — Query Penting untuk Blue Team

```splunk
# Semua Windows Security events
index=main sourcetype=WinEventLog:Security

# Login GAGAL (Event ID 4625) — indikasi brute force
index=main sourcetype=WinEventLog:Security EventCode=4625

# Login BERHASIL (Event ID 4624)
index=main sourcetype=WinEventLog:Security EventCode=4624

# Akun terkunci (Event ID 4740)
index=main sourcetype=WinEventLog:Security EventCode=4740

# Proses baru dibuat (Event ID 4688)
index=main sourcetype=WinEventLog:Security EventCode=4688
```

---

### 6.3 — Query Sysmon

```splunk
# Semua log Sysmon
index=main sourcetype="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational"

# Event ID 1 — Proses baru dibuat
index=main sourcetype="XmlWinEventLog:*Sysmon*" EventCode=1

# Event ID 3 — Koneksi jaringan terdeteksi
index=main sourcetype="XmlWinEventLog:*Sysmon*" EventCode=3
```

---

## 🧪 Bagian 7 — Generate Test Events

### Test: Simulasi Brute Force (Login Gagal)

Di Windows 10, coba login dengan **password salah sebanyak 5 kali**.

Lalu cari hasilnya di Splunk:

```splunk
index=main sourcetype=WinEventLog:Security EventCode=4625
| stats count by host, Account_Name
```

Jika muncul hasil dengan `count >= 5`, artinya Splunk berhasil mendeteksi percobaan brute force. ✅

---

## 📊 Bagian 8 — Membuat Dashboard

### 8.1 — Cara Membuat Panel Dashboard

```
1. Jalankan query di Search & Reporting
2. Klik Save As → Dashboard Panel
3. Buat dashboard baru: "SOC Lab Monitoring"
4. Tambahkan panel
```

---

### 8.2 — Query untuk Panel Dashboard

```splunk
# Panel 1 — Jumlah event per host (timeline)
index=main | timechart count by host

# Panel 2 — Login gagal per akun
index=main sourcetype=WinEventLog:Security EventCode=4625
| stats count by host, Account_Name

# Panel 3 — Top 10 proses yang berjalan (Sysmon)
index=main sourcetype="XmlWinEventLog:*Sysmon*" EventCode=1
| stats count by Image | sort -count | head 10

# Panel 4 — Koneksi jaringan mencurigakan (Sysmon)
index=main sourcetype="XmlWinEventLog:*Sysmon*" EventCode=3
| stats count by DestinationIp, DestinationPort | sort -count

# Panel 5 — Traffic yang diblokir firewall
index=main sourcetype=syslog "block"
| stats count by src_ip, dest_ip
```

---

## 🚨 Bagian 9 — Setup Alert Brute Force

Buat alert otomatis ketika ada percobaan login gagal lebih dari 5 kali:

### Query Alert:

```splunk
index=main sourcetype=WinEventLog:Security EventCode=4625
| stats count by host, Account_Name
| where count > 5
```

### Langkah Membuat Alert:

```
1. Jalankan query di atas di Search & Reporting
2. Klik Save As → Alert
3. Isi:
   - Title       : Brute Force Detection
   - Alert type  : Real-time
   - Trigger     : Number of Results > 0
   - Actions     : Log event (opsional: Send email)
4. Save
```

> ✅ Alert ini akan aktif otomatis setiap kali ada percobaan brute force yang terdeteksi.

---

## 📌 Referensi Event ID Windows

| Event ID | Keterangan | Tingkat Perhatian |
|----------|------------|-------------------|
| `4624` | Login berhasil | ℹ️ Normal |
| `4625` | Login gagal | ⚠️ Waspadai jika berulang |
| `4634` | User logout | ℹ️ Normal |
| `4648` | Login dengan kredensial eksplisit | ⚠️ Perhatikan |
| `4740` | Akun terkunci | 🚨 Investigasi |
| `4688` | Proses baru dibuat | ⚠️ Monitor |

## 📌 Referensi Sysmon Event ID

| Event ID | Keterangan |
|----------|------------|
| `1` | Proses baru dibuat |
| `3` | Koneksi jaringan terdeteksi |
| `7` | Image/DLL di-load |
| `11` | File dibuat |
| `13` | Registry value diubah |

---

## 📚 Referensi

- [Splunk Universal Forwarder Docs](https://docs.splunk.com/Documentation/Forwarder)
- [Sysmon Documentation — Microsoft](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)
- [SwiftOnSecurity Sysmon Config](https://github.com/SwiftOnSecurity/sysmon-config)
- [Splunk Search Reference](https://docs.splunk.com/Documentation/Splunk/latest/SearchReference)

---

*⭐ Star repo ini jika membantu!*
