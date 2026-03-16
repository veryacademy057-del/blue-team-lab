🖧 Ubuntu Server — Static IP Network Configuration

Lab Series: Blue Team SOC Lab Setup
YouTube: Ubuntu Server Network Config - Simple & To The Point
Difficulty: Beginner | OS: Ubuntu Server 22.04 LTS | Tool: Netplan


📋 Overview
This lab covers configuring a static IP address on Ubuntu Server using Netplan — the default network configuration utility for Ubuntu 17.10+. In a SOC/Blue Team environment, static IPs are essential for predictable, reliable communication between all nodes in your lab.
🧪 Lab Environment
ComponentValueUbuntu Server IP10.200.200.100/24Default Gateway10.200.200.254 (OPNsense Firewall)DNS Primary10.200.200.20 (Windows Server DC)DNS Secondary8.8.8.8 (Google)DomainSOCLABS.localConfig File/etc/netplan/00-installer-config.yaml
🗺️ Network Topology (Simplified)
[ Internet ]
      |
[ OPNsense Firewall ] ← 10.200.200.254 (Gateway)
      |
  [ LAN: 10.200.200.0/24 ]
      |          |            |
[ Ubuntu ]  [ Win DC ]   [ Other Nodes ]
.100        .20           .xxx

⚙️ Prerequisites

Ubuntu Server installed (22.04 LTS recommended)
Root or sudo access
Know your network interface name (check with ip link show)

Common Interface Names
InterfaceTypical Useenp0s3VirtualBox VMens33VMware VMeth0Bare metal / older systems
bash# Check your interface name
ip link show

📄 Configuration File
File path: /etc/netplan/00-installer-config.yaml
yamlnetwork:
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

🔍 Configuration Breakdown
Top-Level Structure
network:                    ← Root block (0 spaces)
  version: 2                ← Netplan schema version (2 spaces)
  renderer: networkd        ← Backend engine (2 spaces)
  ethernets:                ← Wired interface section (2 spaces)
    enp0s3:                 ← Your interface name (4 spaces)
      dhcp4: no             ← Disable DHCP (6 spaces)
      addresses:            ← Static IP list (6 spaces)
        - 10.200.200.100/24 ← IP/CIDR (8 spaces + dash)

⚠️ Critical: Netplan uses SPACES only — never TAB. Wrong indentation = parse error.


Key Fields Explained
FieldValueDescriptionversion: 22Netplan config schema versionrenderer: networkdnetworkdUses systemd-networkd — best for servers (no GUI)dhcp4: nonoDisables DHCP; we assign IP manuallyaddresses10.200.200.100/24Static IP + CIDR notation (/24 = 255.255.255.0)routes.to: defaultdefaultApplies to all outbound traffic (default route)routes.via10.200.200.254Gateway — all internet traffic routes through OPNsensenameservers.addresses[0]10.200.200.20Windows Server DC → resolves SOCLABS.localnameservers.addresses[1]8.8.8.8Google DNS → resolves public domainssearchSOCLABS.localAuto-appends domain (e.g., ping win-server → win-server.SOCLABS.local)

🚀 Step-by-Step Implementation
Step 1 — Open the Netplan Config File
bashsudo nano /etc/netplan/00-installer-config.yaml

If the file doesn't exist yet, it will be created. On fresh installs, a default file may already be present — replace its content entirely.


Step 2 — Enter the Configuration
Type or paste the configuration from the Configuration File section above.

💡 Tip: Double-check your interface name first with ip link show and replace enp0s3 if needed.


Step 3 — Save the File
Ctrl+O    ← Write/save
Enter     ← Confirm filename
Ctrl+X    ← Exit nano

Step 4 — Test Configuration (Safe Apply)
bashsudo netplan try
This applies the config temporarily for 120 seconds:

✅ Network works → Press Enter to confirm and make permanent
❌ Network breaks → Wait 120 seconds → Netplan auto-reverts (safe!)


⚠️ Always use netplan try before netplan apply — especially on remote/SSH sessions. A broken config can lock you out permanently.


Step 5 — Apply Permanently
bashsudo netplan apply

✅ Verification
Run these commands to confirm everything is working correctly:
bash# 1. Check IP address assignment
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
Expected Output — ip a
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    inet 10.200.200.100/24 brd 10.200.200.255 scope global enp0s3
Expected Output — ip route
default via 10.200.200.254 dev enp0s3 proto static
10.200.200.0/24 dev enp0s3 proto kernel scope link src 10.200.200.100

🛠️ Troubleshooting
ProblemCauseFixInvalid YAML errorWrong indentation (TAB used)Replace all TABs with spacesNo internet after applyWrong gateway IPVerify via: matches OPNsense LAN IPDNS not resolving .localSearch domain missingAdd search: - SOCLABS.localSSH drops after applyIP conflict or wrong IPUse netplan try next time; fix via consoleInterface name not foundWrong name in configRun ip link show and update enp0s3

📌 Quick Reference Summary
ItemValueIP Address10.200.200.100Subnet Mask/24 (255.255.255.0)Default Gateway10.200.200.254DNS Primary10.200.200.20 (Windows DC)DNS Secondary8.8.8.8 (Google)Search DomainSOCLABS.localConfig File/etc/netplan/00-installer-config.yamlRenderersystemd-networkd



📚 References

Netplan Official Documentation
Ubuntu Server Networking Guide
systemd-networkd Man Page


📺 Follow along on YouTube | ⭐ Star this repo if it helped you!
