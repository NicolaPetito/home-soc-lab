# VM Setup Guide

This document records VM used for the SOC lab and the main setup decisions behind them. This lab runs in VMware on a Windows host, using VMnet for isolated traffica dn VMnet8 for internet access during setup.

---

For network details, see [`network-config.md`](network-config.md). For the overall design, see [`architecture.md`](architecture.md).

**Images:**
- Ubuntu 22.04 LTS Server ISO: [ubuntu.com/download/server](https://ubuntu.com/download/server)
- Debian 12 netinst ISO: [debian.org/distrib/netinst](https://www.debian.org/distrib/netinst)  (amd64 netinst)
- Windows 10 Pro ISO: via Microsoft's Media Creation Tool
- Kali Linux pre-built VMware image: [kali.org/get-kali/#kali-virtual-machines](https://www.kali.org/get-kali/#kali-virtual-machines) (select VMware, 64-bit)

---

## VM 1: Wazuh Server (Ubuntu 22.04 LTS)

### VMware Settings

| Setting | Value | Reason |
|---|---|---|
| Name | `wazuh-server` | Clear identification |
| Guest OS | Linux / Ubuntu 64-bit | Correct virtualisation optimisations |
| RAM | 4096 MB (4GB) | Wazuh manager + indexer + dashboard need margin to work properly|
| Disk | 50GB, single file | Wazuh indexer stores log data; grows over time |
| Adapter 1 | NAT (VMnet8) | Internet access for package installation |
| Adapter 2 | Host-Only (VMnet1) | Internal lab network for agent communications |

### Installation Notes
- Select **Ubuntu Server** (no desktop)
- When prompted for network during install: ens33 (NAT) gets DHCP automatically; ens37 (Host-Only) will show autoconfiguration failed, which is expected
- Create user: `name` / password of your choice
- Skip all optional snaps

### Post-Install
- Update: `sudo apt update && sudo apt upgrade -y`
- Set static IP `192.168.56.10` on ens37 (see [`network-config.md`](network-config.md))
- Verify internet: `ping -c 4 8.8.8.8`
- Verify Host-Only IP: `ip a show ens37`
- Take **Snapshot 1: `01-clean-install`**
- Install Wazuh (see [`wazuh-install.md`](wazuh-install.md))
- Take **Snapshot 2: `02-wazuh-configured`**

---

## VM 2: Linux Victim (Debian 12)

### VMware Settings

| Setting | Value | Reason |
|---|---|---|
| Name | `linux-victim` | Clear identification |
| Guest OS | Linux / Debian 10.x 64-bit | Closest available |
| RAM | 2048 MB (2GB) | 2GB sufficient |
| Disk | 25GB, single file | Minimal footprint needed |
| Adapter 1 | NAT (VMnet8) | Internet access during setup |
| Adapter 2 | Host-Only (VMnet1) | Internal lab network |

### Installation Notes
- Use the **netinst ISO** minimal base install, no desktop
- Primary network interface during install: select **ens33** (NAT)
- If network autoconfiguration fails during install: select **Do not configure the network at this time** and continue, this can be fixed post-install
- Partition method: **Guided - use entire disk**
- Create at least one user with a weak password: `victim` / `Password123` (intentionally weak for a Hydra Brute Force)

### Known Issue: NAT DHCP Failure
The Debian minimal install does not include `isc-dhcp-client`. The NAT adapter may fail to get an IP automatically after install. This is resolved by configuring the static IP on the Host-Only adapter first, then installing the DHCP client via package transfer from the Wazuh Server over the Host-Only network.

### Post-Install Steps
- Log in as root
- Bring up interfaces: `ip link set ens33 up && ip link set ens34 up`
- Set static IP `192.168.56.30` on Host-Only interface (see [`network-config.md`](network-config.md))
- Install SSH server: `apt install openssh-server -y`
- Verify SSH is running: `systemctl status ssh`
- Take **Snapshot 1: `01-clean-install`**
- Install Wazuh agent and register to `192.168.56.10`
- Take **Snapshot 2: `02-agent-configured`**

### Why Debian Instead of Ubuntu?
This was a design choice taken to be able to work on different distros, which reflects a more realistic enterprise environment where machines may not run identical OS configurations.

---

## VM 3: Windows 10 Pro Victim

### VMware Settings

| Setting | Value | Reason |
|---|---|---|
| Name | `windows-victim` | Clear identification |
| Guest OS | Windows 10 x64 | Correct virtualisation optimisations |
| Firmware | BIOS | Avoids UEFI boot issues with VMware |
| RAM | 4096 MB (4GB) | Windows needs headroom to remain responsive under load |
| Disk | 50GB, single file | Windows base install uses minimun 20GB |
| Adapter 1 | NAT (VMnet8) | Internet access during setup |
| Adapter 2 | Host-Only (VMnet1) | Internal lab network |

### Installation Notes
- ISO must be **Windows 10 Pro** as the Home edition does not support Remote Desktop Protocol (RDP) server natively
- During setup, when prompted to sign in with a Microsoft account: disconnect the NAT adapter in VMware settings temporarily to force the offline account option
- Create local account: username `victim` / password `Password123` (intentionally weak)
- Skip all Microsoft account prompts

### Post-Install Steps
- Install VMware Tools: VM menu -> Install VMware Tools -> DVD Drive -> `setup64.exe` -> Typical → restart
- Enable RDP: Settings -> System -> Remote Desktop -> toggle **On**
- Pause Windows Update: Settings -> Update & Security -> Advanced Options -> pause to max date
- Set static IP `192.168.56.20` on Host-Only adapter (see [`network-config.md`](network-config.md))
- Verify RDP port: `netstat -an | find "3389"` should show `0.0.0.0:3389`
- Take **Snapshot 1: `01-clean-install`**
- Install Wazuh agent and register to `192.168.56.10`
- Take **Snapshot 2: `02-agent-configured`**

### Why Enable RDP?
RDP (port 3389) is one of the most commonly attacked services in real enterprise environments. Enabling it here creates a realistic attack surface for T1110 (Brute Force) simulation. In a production environment, RDP could be protected with MFA, VPN, and network segmentation.

---

## VM 4: Kali Linux Attacker

### Import Method
Kali provides a pre-built VMware image. Therefore, no OS installation needed.

1. Download the `.7z` file from [kali.org/get-kali/#kali-virtual-machines](https://www.kali.org/get-kali/#kali-virtual-machines) (VMware, 64-bit)
2. Extract with 7-Zip
3. In VMware: **File -> Open** -> select the `.vmx` file
4. VMware imports it automatically

### Post-Import Settings
Right click the imported VM -> **Settings** and adjust:

| Setting | Value |
|---|---|
| RAM | 4096 MB (4GB) |
| Existing Network Adapter | Change to NAT (VMnet8) |
| Add Network Adapter | Host-Only (VMnet1) |

### Post-Import Steps
- Boot and log in: `kali` / `kali`
- Install VMware Tools: `sudo apt install open-vm-tools-desktop -y`
- Update: `sudo apt update && sudo apt upgrade -y`
- Set static IP `192.168.56.40` on Host-Only interface (see [`network-config.md`](network-config.md))
- Verify tools: `hydra -h` and `nmap --version`
- Take **Snapshot 1: `01-clean-install`**

### Note on Wazuh Agent
The Kali attacker VM does **not** have a Wazuh agent installed. In a real scenario, the SOC team does not have visibility into the attacker's machine. Thus, the attacker is visible only through the footprints it leaves on the victim systems.

---

