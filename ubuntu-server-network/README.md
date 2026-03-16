# 🖧 Ubuntu Server — Static IP Network Configuration

> **Lab Series:** Blue Team SOC Lab Setup  
> **YouTube:** [▶️ Ubuntu Server Network Config - Simple & To The Point](https://youtu.be/08Sz69p3bFg?si=YbcH1B-UFlFkKkPh)  
> **Difficulty:** `Beginner` | **OS:** `Ubuntu Server 22.04 LTS` | **Tool:** `Netplan`

---

## 📋 Overview

This lab covers configuring a **static IP address** on Ubuntu Server using **Netplan** — the default network configuration utility for Ubuntu 17.10+. In a SOC/Blue Team environment, static IPs are essential for predictable, reliable communication between all nodes in your lab.

### 🧪 Lab Environment

| Component | Value |
|-----------|-------|
| **Ubuntu Server IP** | `10.200.200.100/24` |
| **Default Gateway** | `10.200.200.254` (OPNsense Firewall) |
| **DNS Primary** | `10.200.200.20` (Windows Server DC) |
| **DNS Secondary** | `8.8.8.8` (Google) |
| **Domain** | `SOCLABS.local` |
| **Config File** | `/etc/netplan/00-installer-config.yaml` |

### 🗺️ Network Topology (Simplified)

```
[ Internet ]
      |
[ OPNsense Firewall ] ← 10.200.200.254 (Gateway)
      |
  [ LAN: 10.200.200.0/24 ]
      |          |            |
[ Ubuntu ]  [ Win DC ]   [ Other Nodes ]
.100        .20           .xxx
```

---

## ⚙️ Prerequisites

- Ubuntu Server installed (22.04 LTS recommended)
- Root or `sudo` access
- Know your network interface name (check with `ip link show`)

### Common Interface Names

| Interface | Typical Use |
|-----------|-------------|
| `enp0s3` | VirtualBox VM |
| `ens33` | VMware VM |
| `eth0` | Bare metal / older systems |

```bash
# Check your interface name
ip link show
```

---

## 📄 Configuration File

**File path:** `/etc/netplan/00-installer-config.yaml`

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 10.200.200.100/24
      routes:
        - to: default
          via: 10.200.200.254
      nameservers:
        addresses:
          - 10.200.200.20
          - 8.8.8.8
        search:
          - SOCLABS.local
```

---

## 🔍 Configuration Breakdown

### Top-Level Structure

```
network:                    ← Root block (0 spaces)
  version: 2                ← Netplan schema version (2 spaces)
  renderer: networkd        ← Backend engine (2 spaces)
  ethernets:                ← Wired interface section (2 spaces)
    enp0s3:                 ← Your interface name (4 spaces)
      dhcp4: no             ← Disable DHCP (6 spaces)
      addresses:            ← Static IP list (6 spaces)
        - 10.200.200.100/24 ← IP/CIDR (8 spaces + dash)
```

> ⚠️ **Critical:** Netplan uses **SPACES only — never TAB**. Wrong indentation = parse error.

---

### Key Fields Explained

| Field | Value | Description |
|-------|-------|-------------|
| `version: 2` | `2` | Netplan config schema version |
| `renderer: networkd` | `networkd` | Uses `systemd-networkd` — best for servers (no GUI) |
| `dhcp4: no` | `no` | Disables DHCP; we assign IP manually |
| `addresses` | `10.200.200.100/24` | Static IP + CIDR notation (`/24` = `255.255.255.0`) |
| `routes.to: default` | `default` | Applies to all outbound traffic (default route) |
| `routes.via` | `10.200.200.254` | Gateway — all internet traffic routes through OPNsense |
| `nameservers.addresses[0]` | `10.200.200.20` | Windows Server DC → resolves `SOCLABS.local` |
| `nameservers.addresses[1]` | `8.8.8.8` | Google DNS → resolves public domains |
| `search` | `SOCLABS.local` | Auto-appends domain (e.g., `ping win-server` → `win-server.SOCLABS.local`) |

---

## 🚀 Step-by-Step Implementation

### Step 1 — Open the Netplan Config File

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

> If the file doesn't exist yet, it will be created. On fresh installs, a default file may already be present — replace its content entirely.

---

### Step 2 — Enter the Configuration

Type or paste the configuration from the [Configuration File](#-configuration-file) section above.

> 💡 **Tip:** Double-check your interface name first with `ip link show` and replace `enp0s3` if needed.

---

### Step 3 — Save the File

```
Ctrl+O    ← Write/save
Enter     ← Confirm filename
Ctrl+X    ← Exit nano
```

---

### Step 4 — Test Configuration (Safe Apply)

```bash
sudo netplan try
```

This applies the config **temporarily for 120 seconds**:
- ✅ Network works → Press `Enter` to confirm and make permanent
- ❌ Network breaks → Wait 120 seconds → Netplan **auto-reverts** (safe!)

> ⚠️ **Always use `netplan try` before `netplan apply`** — especially on remote/SSH sessions. A broken config can lock you out permanently.

---

### Step 5 — Apply Permanently

```bash
sudo netplan apply
```

---

## ✅ Verification

Run these commands to confirm everything is working correctly:

```bash
# 1. Check IP address assignment
ip a

# 2. Check routing table (should show default via 10.200.200.254)
ip route

# 3. Check DNS resolver config
cat /etc/resolv.conf

# 4. Test gateway reachability
ping -c 4 10.200.200.254

# 5. Test internet connectivity
ping -c 4 8.8.8.8

# 6. Test DNS resolution (public)
ping -c 4 google.com

# 7. Test internal domain resolution
ping -c 4 SOCLABS.local
```

### Expected Output — `ip a`

```
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    inet 10.200.200.100/24 brd 10.200.200.255 scope global enp0s3
```

### Expected Output — `ip route`

```
default via 10.200.200.254 dev enp0s3 proto static
10.200.200.0/24 dev enp0s3 proto kernel scope link src 10.200.200.100
```

---

## 🛠️ Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `Invalid YAML` error | Wrong indentation (TAB used) | Replace all TABs with spaces |
| No internet after apply | Wrong gateway IP | Verify `via:` matches OPNsense LAN IP |
| DNS not resolving `.local` | Search domain missing | Add `search: - SOCLABS.local` |
| SSH drops after apply | IP conflict or wrong IP | Use `netplan try` next time; fix via console |
| Interface name not found | Wrong name in config | Run `ip link show` and update `enp0s3` |

---

## 📌 Quick Reference Summary

| Item | Value |
|------|-------|
| **IP Address** | `10.200.200.100` |
| **Subnet Mask** | `/24` (`255.255.255.0`) |
| **Default Gateway** | `10.200.200.254` |
| **DNS Primary** | `10.200.200.20` (Windows DC) |
| **DNS Secondary** | `8.8.8.8` (Google) |
| **Search Domain** | `SOCLABS.local` |
| **Config File** | `/etc/netplan/00-installer-config.yaml` |
| **Renderer** | `systemd-networkd` |

---

## 🔗 Related Labs

> *(Coming soon — links will be updated as the series progresses)*

- [ ] `01` — OPNsense Firewall Setup & LAN Configuration
- [x] `02` — **Ubuntu Server Static IP (this lab)**
- [ ] `03` — Windows Server 2022 — AD DS & DNS Setup
- [ ] `04` — Wazuh SIEM Installation & Agent Deployment
- [ ] `05` — Sysmon Configuration for Enhanced Logging
- [ ] `06` — Attack Simulation with Kali Linux

---

## 🎬 Video Tutorial

[![Ubuntu Server Network Config - Simple & To The Point](https://img.youtube.com/vi/08Sz69p3bFg/maxresdefault.jpg)](https://youtu.be/08Sz69p3bFg?si=YbcH1B-UFlFkKkPh)

> 📺 **[Watch on YouTube → Ubuntu Server Network Config - Simple & To The Point](https://youtu.be/08Sz69p3bFg?si=YbcH1B-UFlFkKkPh)**  
> Follow along with the video while using this documentation as your reference guide.

---

## 📚 References

- [Netplan Official Documentation](https://netplan.io/reference)
- [Ubuntu Server Networking Guide](https://ubuntu.com/server/docs/network-configuration)
- [systemd-networkd Man Page](https://www.freedesktop.org/software/systemd/man/systemd.network.html)

---

*📺 Follow along on [YouTube](https://youtu.be/08Sz69p3bFg?si=YbcH1B-UFlFkKkPh) | ⭐ Star this repo if it helped you!*
