# OpenSOC Lab — Complete Build Journal
## Chronological Record: Every Step, Every Error, Every Fix

---

## PROJECT OVERVIEW

**Lab Name:** OpenSOC Lab
**Full Title:** Open-Source Security Operations & Detection Engineering Home Lab
**Host System:** Arch Linux
**Hypervisor:** VirtualBox
**Target Role:** SOC Analyst

---

## TOOLS USED

| Tool | Version | Purpose | Cost |
|------|---------|---------|------|
| VirtualBox | 7.x | Hypervisor | Free |
| Ubuntu Server | 22.04 LTS | Wazuh SIEM OS | Free |
| Wazuh | 4.7.5 | SIEM + EDR + Dashboard | Free GPL v2 |
| Windows 11 LTSC | 2021 Eval | Target endpoint | Free eval |
| Sysmon | Latest | Deep Windows telemetry | Free Microsoft |
| SwiftOnSecurity Config | Latest | Sysmon ruleset | Free MIT |

---

## CHRONOLOGICAL BUILD LOG

---

### STAGE 1 — VirtualBox Verification on Arch Linux

Verified VirtualBox was correctly installed and kernel modules loaded.

**Commands:**
```bash
virtualbox --version
lsmod | grep vboxdrv
sudo modprobe vboxdrv
sudo pacman -S virtualbox-host-modules-arch
sudo usermod -aG vboxusers $(whoami)
```

**Note:** Must log out and back in after adding user to vboxusers group.

---

### STAGE 2 — Creating Isolated Lab Network

Created isolated NAT Network so VMs communicate with each other but are hidden from the real home network.

**ERROR 1: NAT Network GUI creation failed**
```
Failed to change NAT network parameter.
192.168.100.0/24 is not a valid IPv4 CIDR notation.
Result Code: NS_ERROR_FAILURE (0x80004005)
```
Cause: VirtualBox GUI bug with CIDR input on Linux.

**Fix — Used CLI instead:**
```bash
VBoxManage natnetwork add --netname soclab --network "192.168.100.0/24" --enable --dhcp on
VBoxManage natnetwork list
VBoxManage natnetwork start --netname soclab
```

Network: `192.168.100.0/24` | Gateway: `192.168.100.1`

---

### STAGE 3 — Ubuntu Server 22.04 VM Creation

**VM specs:**
- Name: wazuh-siem | RAM: 3072 MB | CPUs: 2 | Disk: 40 GB
- Network Adapter 1: NAT Network (soclab)
- Network Adapter 2: Host-only (vboxnet0) — added later

Ubuntu profile: username=soc, hostname=wazuh-siem, SSH enabled.
VM IP assigned: 192.168.100.4

---

### STAGE 4 — Wazuh SIEM Installation (Multiple Errors Encountered)

---

**ERROR 2: curl could not resolve host**
```
curl: (6) Could not resolve host: packages.wazuh.com
```
Cause: Ubuntu had no DNS server configured. VirtualBox NAT Network DHCP assigned IP but did not push DNS info. /etc/resolv.conf was empty — VM had no "phone book" to translate domain names to IPs.

**Fix:**
```bash
sudo bash -c 'echo "nameserver 8.8.8.8" > /etc/resolv.conf'
sudo bash -c 'echo "nameserver 8.8.4.4" >> /etc/resolv.conf'
```

SOC relevance: DNS is critical in security — malware uses DNS to contact C2 servers. Sysmon Event ID 22 logs all DNS queries.

---

**ERROR 3: Hardware requirements check failed**
```
ERROR: Your system does not meet minimum hardware requirements of 4GB RAM
```
Cause: Wazuh checks for 4GB minimum. VM has 3GB (fine for a lab).

**Fix:**
```bash
sudo bash wazuh-install.sh -a -i
# -i = ignore hardware requirements
```

---

**ERROR 4: apt locked by background process**
```
ERROR: Cannot install dependency: apt-transport-https
Another process is using APT. Waiting for it to release the lock.
```
Cause: Ubuntu's unattended-upgrades auto-updater was running and had locked apt.

**Fix:**
```bash
sudo killall apt apt-get 2>/dev/null
sudo rm -f /var/lib/dpkg/lock-frontend /var/lib/dpkg/lock /var/cache/apt/archives/lock
sudo dpkg --configure -a
sudo systemctl disable --now unattended-upgrades apt-daily.timer apt-daily-upgrade.timer
sudo apt update && sudo apt install -y apt-transport-https curl wget
```

Key lesson: Always disable unattended-upgrades before long installations in Ubuntu lab VMs.

---

**ERROR 5: No space left on device**
```
E: Write error - write (28: No space left on device)
```
Cause: Multiple failed installs left behind large incomplete packages that filled the disk.

**Fix:**
```bash
sudo apt clean
sudo apt autoremove -y
sudo rm -f ~/wazuh-install.sh ~/config.yml
df -h  # confirmed 12GB free after cleanup
```

---

**ERROR 6: Ports already occupied**
```
ERROR: Port 1515 is being used by another process
ERROR: Port 55000 is being used by another process
```
Cause: Previous failed install left background processes holding ports.

**Fix:**
```bash
sudo systemctl stop wazuh-manager wazuh-indexer wazuh-dashboard filebeat 2>/dev/null
sudo fuser -k 1515/tcp 2>/dev/null
sudo fuser -k 55000/tcp 2>/dev/null
sudo fuser -k 9200/tcp 2>/dev/null
sudo ss -tlnp | grep -E '1515|55000|9200|443'  # verify empty
```

---

**ERROR 7: Wazuh 4.9.2 keystore bug**
```
/var/ossec/bin/wazuh-keystore: No such file or directory
ERROR: wazuh-manager could not be started.
```
Cause: Confirmed bug in Wazuh 4.9.2 on Ubuntu 22.04.

**Fix — Switched to Wazuh 4.7.5 (stable):**
```bash
rm -f wazuh-install.sh wazuh-install-files.tar config.yml
curl -O https://packages.wazuh.com/4.7/wazuh-install.sh
curl -O https://packages.wazuh.com/4.7/config.yml
```

---

**ERROR 8: VM in unrecoverable corrupted state**
Cause: Too many partial installs. No snapshot existed to restore from.

**Fix:** Deleted VM completely. Created fresh Ubuntu VM.

Critical lesson learned: ALWAYS take a VirtualBox snapshot immediately after OS installation and before installing any software. This is now standard procedure.

**Correct procedure established:**
1. Install Ubuntu
2. Log in
3. Fix DNS immediately
4. Disable unattended-upgrades immediately
5. TAKE SNAPSHOT (ubuntu-clean-before-wazuh)
6. Then install Wazuh

---

**SUCCESSFUL Wazuh 4.7.5 Installation:**

config.yml used:
```yaml
nodes:
  indexer:
    - name: node-1
      ip: "192.168.100.4"
  server:
    - name: wazuh-1
      ip: "192.168.100.4"
  dashboard:
    - name: dashboard
      ip: "192.168.100.4"
```

```bash
sudo bash wazuh-install.sh -a -i
```

Result:
```
INFO: Wazuh dashboard installation finished.
INFO: wazuh-manager service started.
INFO: wazuh-indexer service started.
INFO: wazuh-dashboard service started.
INFO: You can access the web interface https://192.168.100.4:443
    User: admin
    Password: [generated]
INFO: Installation finished.
```

Snapshot taken: wazuh-fully-installed-working

---

**ERROR 9: Dashboard unreachable from Arch host**
```
ping 192.168.100.4 — 100% packet loss
```
Cause: VirtualBox NAT Network by design does not allow the host machine to reach VMs. It only allows VMs to reach the internet.

**Fix — Added Host-only Network Adapter:**
- Added Adapter 2 (Host-only, vboxnet0) to both VMs
- VMs now have two IPs each:
  - 192.168.100.X — VM to VM (soclab)
  - 192.168.56.X — Arch host to VM (vboxnet0)

Dashboard accessed at: https://192.168.56.X

---

### STAGE 5 — Windows 11 Enterprise LTSC VM

**VM specs:**
- Name: win-target | RAM: 2048 MB | CPUs: 2 | Disk: 30 GB (expanded to 50 GB)
- EFI: enabled | TPM: 2.0 | Paravirtualization: Hyper-V
- Network Adapter 1: NAT Network (soclab)
- Network Adapter 2: Host-only (vboxnet0)

---

**ERROR 10: VM failed to boot from ISO**
```
BdsDxe: No bootable option or device was found.
```
Cause: Windows 11 ISO was on SATA controller. VirtualBox EFI firmware expects optical drives on IDE, not SATA. SATA is treated as hard disk interface.

**Fix:** Moved ISO from SATA to IDE controller (PIIX4).
```
Controller: SATA → win-target.vdi (hard disk)
Controller: IDE  → Windows11_LTSC.iso (optical)
```

---

**ERROR 11: Windows 11 system requirements check**
```
This PC doesn't currently meet Windows 11 system requirements
```
Cause: Windows 11 checks TPM 2.0, Secure Boot, RAM. VMs don't fully pass even with TPM enabled.

**Fix — Registry bypass via Shift+F10:**
```
regedit → HKEY_LOCAL_MACHINE\SYSTEM\Setup → New Key: LabConfig
```
Created DWORD values:
- BypassTPMCheck = 1
- BypassSecureBootCheck = 1
- BypassRAMCheck = 1

---

**ERROR 12: Insufficient disk space for Wazuh agent**
```
Used: 28 GB | Free: 1.1 GB (of 30 GB disk)
```
Cause: Windows 11 installation consumed nearly all virtual disk space.

**Fix — Expanded virtual disk:**
```bash
# On Arch host:
VBoxManage modifymedium disk /path/to/win-target.vdi --resize 51200
```
Inside Windows: diskmgmt.msc → Extend Volume on C:

Cleanup also performed:
```cmd
cleanmgr /d C:
dism /online /cleanup-image /startcomponentcleanup
```

---

**ERROR 13: auditpol parameter incorrect on Windows 11**
```
Error 0x00000057: The parameter is incorrect.
auditpol /set /category:"Logon/Logoff"  ← fails
auditpol /set /category:"Process Tracking"  ← fails
```
Cause: Windows 11 removed legacy category names from auditpol.

**Fix — Used subcategory names:**
```cmd
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
auditpol /set /subcategory:"Logoff" /success:enable /failure:enable
auditpol /set /subcategory:"Account Lockout" /success:enable /failure:enable
auditpol /set /subcategory:"User Account Management" /success:enable /failure:enable
auditpol /set /subcategory:"Security Group Management" /success:enable /failure:enable
auditpol /set /subcategory:"Process Creation" /success:enable /failure:enable
auditpol /set /subcategory:"Credential Validation" /success:enable /failure:enable
auditpol /set /subcategory:"Special Logon" /success:enable /failure:enable
```

PowerShell logging:
```cmd
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" /v EnableScriptBlockLogging /t REG_DWORD /d 1 /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ModuleLogging" /v EnableModuleLogging /t REG_DWORD /d 1 /f
```

---

### STAGE 6 — Sysmon Installation

**What Sysmon adds (default Windows logging is almost useless):**

| Event ID | What it captures |
|----------|-----------------|
| 1 | Process creation + full command line + file hash |
| 3 | Network connections + destination IP + port |
| 11 | File creation in sensitive directories |
| 12/13 | Registry modifications |
| 22 | All DNS queries |

**Installation:**
```cmd
cd C:\Users\WinTarget\Downloads
Sysmon64.exe -accepteula -i sysmonconfig.xml
sc query sysmon64
```
Config: SwiftOnSecurity sysmonconfig-export.xml (community standard ruleset)
Result: STATE: 4 RUNNING

---

### STAGE 7 — Wazuh Agent on Windows

Downloaded: wazuh-agent-4.7.5-1.msi
Manager IP configured: 192.168.100.4

```cmd
NET START WazuhSvc
sc query WazuhSvc
```
Result: STATE: 4 RUNNING

---

## FINAL ARCHITECTURE

```
Arch Linux Host
      |
   VirtualBox
      |
  -----------------------------------------------
  |                                             |
wazuh-siem VM                            win-target VM
Ubuntu 22.04                          Windows 11 LTSC
Wazuh 4.7.5                           Sysmon + Wazuh Agent
192.168.100.4 (internal)              192.168.100.X (internal)
192.168.56.X  (host access)           192.168.56.102 (host access)
  |                                             |
  |<---- Wazuh Agent ships logs (port 1514) ----|
  |
  Dashboard: https://192.168.56.X
```

---

## LOG FLOW PIPELINE

```
Windows Events + Sysmon Telemetry
         |
   Wazuh Agent (WinTarget)
         |
   Wazuh Manager (wazuh-siem)
         |
   Wazuh Indexer (OpenSearch DB)
         |
   Wazuh Dashboard (browser)
         |
   Alerts + MITRE ATT&CK Mapping
```

---

## ALL ERRORS SUMMARY

| # | Error | Root Cause | Solution |
|---|-------|-----------|---------|
| 1 | NAT network GUI bug | VirtualBox Linux bug | VBoxManage CLI |
| 2 | Could not resolve host | No DNS in resolv.conf | Added 8.8.8.8 |
| 3 | Hardware requirements | VM RAM < 4GB | -i ignore flag |
| 4 | apt locked | unattended-upgrades | Disabled auto-updater |
| 5 | No disk space | Failed installs filled disk | apt clean + autoremove |
| 6 | Ports occupied | Old Wazuh processes | fuser -k kill ports |
| 7 | Keystore missing | Wazuh 4.9.2 bug | Switched to 4.7.5 |
| 8 | Corrupted VM | No snapshot + repeated fails | Fresh VM + snapshot |
| 9 | Dashboard unreachable | NAT blocks host | Host-only adapter |
| 10 | VM won't boot ISO | ISO on SATA not IDE | Moved to IDE controller |
| 11 | Win11 requirements | TPM/SecureBoot check | LabConfig registry bypass |
| 12 | No space for agent | Win11 used 28/30 GB | Expanded VDI to 50 GB |
| 13 | auditpol error | Win11 changed names | Used subcategory names |

---

## SNAPSHOTS TAKEN

| Name | VM | Purpose |
|------|----|---------|
| ubuntu-clean-before-wazuh | wazuh-siem | Clean restore point |
| wazuh-fully-installed-working | wazuh-siem | Working SIEM |
| windows-clean-install | win-target | Clean Windows |
| sysmon-wazuh-agent-installed | win-target | Working endpoint |

