# Cybersecurity Home Lab Setup Report
**Date:** June 27, 2026

---

## Lab Overview

| Role | Machine | OS | Virtualization |
|---|---|---|---|
| Attacker | Lenovo LOQ (Ryzen 7, RTX 5060, 16GB RAM) | Windows + Kali Linux | VMware Workstation Pro |
| Victim | Dell Inspiron 15 3541 (AMD A6, 4GB RAM, 256GB SSD) | Debian 13 Trixie + XFCE | VirtualBox 7.1 |

---

## Victim Machine Setup (Dell)

### OS
- **Debian 13 Trixie** (selected during install from desktop environment menu)
- XFCE desktop environment
- User: `dell`

### Issues encountered & resolved
1. **sudo not configured** — user `dell` was not in sudoers after install
   - Fixed via: `su -` → `usermod -aG sudo dell` → reboot
2. **VirtualBox not in default Debian repos** — added Oracle's official repo
   - Used `trixie` repo (not `bookworm`) since Debian 13 is installed
   - Installed `virtualbox-7.1`
3. **VERR_SVM_IN_USE** — KVM kernel module conflicting with VirtualBox AMD-V
   - Fixed via: `sudo modprobe -r kvm_amd && sudo modprobe -r kvm`
   - Made permanent: blacklisted both modules in `/etc/modprobe.d/blacklist-kvm.conf`
4. **Battery cycling issue** — old battery causing charge loop
   - Resolved by physically removing the battery; machine runs on AC only

### VirtualBox commands used
```bash
# Add Oracle VirtualBox repo
curl -fsSL https://www.virtualbox.org/download/oracle_vbox_2016.asc | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/virtualbox.gpg
echo "deb [arch=amd64] https://download.virtualbox.org/virtualbox/debian trixie contrib" | sudo tee /etc/apt/sources.list.d/virtualbox.list
sudo apt update
sudo apt install virtualbox-7.1 -y

# Disable KVM (run before starting VM every time, or blacklist permanently)
sudo modprobe -r kvm_amd
sudo modprobe -r kvm

# Blacklist permanently
echo "blacklist kvm_amd" | sudo tee /etc/modprobe.d/blacklist-kvm.conf
echo "blacklist kvm" | sudo tee -a /etc/modprobe.d/blacklist-kvm.conf
```

---

## Metasploitable 2 Setup

### Source
Downloaded from SourceForge, extracted on Windows, transferred to Dell via USB.

### Files used
- `Metasploitable.vmdk`
- `Metasploitable.nvram`, `.vmsd`, `.vmx`, `.vmxf` (supporting files)

### VirtualBox VM config
| Setting | Value |
|---|---|
| Name | Metasploitable2 |
| Type | Linux / Ubuntu (32-bit) |
| RAM | 512MB |
| CPU | 1 core |
| Disk | Existing `Metasploitable.vmdk` |
| Network | Bridged Adapter → `wlp2s0` (WiFi) |

### Credentials
```
Username: msfadmin
Password: msfadmin
```

### Current IP
```
10.14.88.150
```
*(assigned by phone hotspot — may change on reconnect)*

---

## Network Architecture

```
Phone Hotspot (10.14.88.x subnet)
        |
        |--- Dell (Debian host) 
        |         └── Metasploitable2 VM (10.14.88.150) [Bridged via wlp2s0]
        |
        |--- Lenovo LOQ (Windows host)
                  └── Kali Linux VM (VMware, Bridged) 
```

Both VMs bridged to the phone hotspot — same subnet, direct connectivity confirmed.

---

## Connectivity Verified

```bash
# From Kali
ping 10.14.88.150       # SUCCESS
nmap -sV 10.14.88.150   # SUCCESS
```

---

## Nmap Scan Results (from Kali)

```
PORT     STATE  SERVICE      VERSION
21/tcp   open   ftp          vsftpd 2.3.4
22/tcp   open   ssh          OpenSSH 4.7p1 Debian 8ubuntu1
23/tcp   open   telnet       Linux telnetd
25/tcp   open   smtp         Postfix smtpd
53/tcp   open   domain       ISC BIND 9.4.2
80/tcp   open   http         Apache httpd 2.2.8 (Ubuntu) DAV/2
111/tcp  open   rpcbind      2 (RPC #100000)
139/tcp  open   netbios-ssn  Samba smbd 3.X-4.X
445/tcp  open   netbios-ssn  Samba smbd 3.X-4.X
512/tcp  open   exec         netkit-rsh rexecd
513/tcp  open   login        OpenBSD or Solaris rlogind
514/tcp  open   tcpwrapped
1099/tcp open   java-rmi     GNU Classpath grmiregistry
1524/tcp open   bindshell    Metasploitable root shell
2049/tcp open   nfs          2-4 (RPC #100003)
2121/tcp open   ftp          ProFTPD 1.3.1
3306/tcp open   mysql        MySQL 5.0.51a-3ubuntu5
5432/tcp open   postgresql   PostgreSQL DB 8.3.0-8.3.7
5900/tcp open   vnc          VNC (protocol 3.3)
6000/tcp open   X11          (access denied)
6667/tcp open   irc          UnrealIRCd
8009/tcp open   ajp13        Apache Jserv (Protocol v1.3)
8180/tcp open   http         Apache Tomcat/Coyote JSP engine 1.1
```

---

## Next Steps / Notable Exploits to Try

| Port | Service | Exploit |
|---|---|---|
| 21 | vsftpd 2.3.4 | Backdoor — `exploit/unix/ftp/vsftpd_234_backdoor` |
| 139/445 | Samba | MS08-067 style Samba exploits |
| 1524 | bindshell | Direct root shell — just netcat to port 1524 |
| 6667 | UnrealIRCd | Backdoor — `exploit/unix/irc/unreal_ircd_3281_backdoor` |
| 512 | rexec | Login with msfadmin credentials |

### First exploit (recommended starting point)
```bash
msfconsole
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS 10.14.88.150
run
# Expected result: root shell
```

---

## Important Notes for Next Session

- Run `sudo modprobe -r kvm_amd && sudo modprobe -r kvm` on Dell before starting VM (unless blacklist is already applied)
- Metasploitable IP (`10.14.88.150`) may change if phone hotspot reassigns — always check with `ifconfig` inside the VM
- Both machines must be on the same hotspot network for connectivity
- Dell runs on AC power only (battery removed)
