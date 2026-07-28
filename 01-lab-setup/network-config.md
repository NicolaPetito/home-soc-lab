# VMware Workstation Network Configuration

This document covers the network setup required before any VMs are created. Read [`architecture.md`](architecture.md) first for the reasoning behind this design.

---

## Overview

Every VM in this lab uses **two network adapters**:

| Adapter | Mode | VMware Name | Purpose |
|---|---|---|---|
| Adapter 1 | NAT | VMnet8 | Outbound internet access (setup and updates only) |
| Adapter 2 | Host-Only | VMnet1 | Internal lab traffic (attacks, logs, SIEM comms) |

This separation means VMs can download packages during setup while all attack and detection traffic stays completely isolated on the internal subnet.

---

## Step 1: Configuring VMnet1 in Virtual Network Editor

This only needs to be done once. All VMs will share the same Host-Only adapter (VMnet1).

1. Open VMware Workstation
2. Go to Virtual Network Editor
3. Click Change Settings
4. Select **VMnet1** from the list
5. This was configured as follow:

| Setting | Value |
|---|---|
| Type | Host-only |
| Connect a host virtual adapter to this network | Ticked |
| Use local DHCP service to distribute IP addresses | Unticked |
| Subnet IP | `192.168.56.0` |
| Subnet Mask | `255.255.255.0` |

6. Click Apply and OK

VMware automatically assigns the Windows host the address `192.168.56.1` on VMnet1.

> **DHCP must be disabled** because the Wazuh agents on victim machines need to know the fixed IP of the Wazuh manager (`192.168.56.10`) to register and send logs. If DHCP assigned addresses dynamically, IPs could change between reboots and break agent connectivity. In this case we simulate an enterprise environment, where servers normally use static IPs for exactly this reason.

### Verifying VMnet1 on the Host

On the Windows host:

```cmd
ipconfig
```

**VMware Network Adapter VMnet1**. It should show:

```
IPv4 Address: 192.168.56.1
Subnet Mask:  255.255.255.0
```

---

## Step 2: Assign Adapters When Creating Each VM

When creating each VM in VMware Workstation, and then set the network adapter during the wizard to **NAT**.

1. Right click the VM **Settings**
2. Click **Add a Network Adapter**
3. Select **Host-only (VMnet1)**
4. Click **Finish and OK**

Each VM must have two adapters: NAT (VMnet8) for internet and Host-Only (VMnet1) for lab traffic.

---

## Step 3: Configure Static IPs Inside Each VM

After each VM's OS is installed, assign a static IP to the Host-Only interface from inside the VM. The NAT interface gets its IP automatically from VMware's internal DHCP.

### On Ubuntu (Wazuh Server)

Ubuntu 22.04 uses Netplan for network configuration.

Identify your network interfaces:
```bash
ip a
```

It shows two interfaces:
- One with a `192.168.232.x` address. In this build, this was the NAT adapter (`ens37`) and it remained on DHCP.
- One with no IP or the lab static IP. In this build, this was the Host-Only adapter (`ens33`) and the static IP was assigned here.

The netplan configuration must be edited accordingly to the interfaces and the static IP:
```bash
sudo micro /etc/netplan/00-installer-config.yaml
```

Configuration used for the **Wazuh Server**:

```yaml
network:
  version: 2
  ethernets:
    ens37:
      dhcp4: true
    ens33:
      dhcp4: false
      addresses:
        - 192.168.56.10/24
```

Apply:
```bash
sudo netplan apply
```

This can be verified if needed:
```bash
ip a show ens33
```

### On Debian (Linux Victim)

Debian uses `/etc/network/interfaces` instead of Netplan.

Identify interfaces:
```bash
ip a
```

Edit the interfaces file:
```bash
nano /etc/network/interfaces
```

Add the following:

```
auto lo
iface lo inet loopback

# Host-Only adapter (static) - VMnet1
auto ens33
iface ens33 inet static
  address 192.168.56.30
  netmask 255.255.255.0

# NAT adapter (dhcp) - VMnet8 
auto ens38
iface ens38 inet dhcp
```

> **Note:** Interface names on Debian may differ, verify with `ip a` before editing. In this case, `ens33` was Host-Only and `ens38` was NAT.

Apply:
```bash
systemctl restart networking
```

Verify:
```bash
ip a
```

### On Windows 10 Pro (Windows Victim)

1. Open **Control Panel -> Network and Sharing Centre -> Change adapter settings**
2. You will see two adapters. In this lab, Host-Only was Ethernet0, and NAT was Ethernet1, but this may vary.
3. Right-click the **Host-Only adapter (VMnet1)** -> **Properties**
4. Select **Internet Protocol Version 4 (TCP/IPv4)** -> **Properties**
5. Select **Use the following IP address** and enter:

| Field | Value |
|---|---|
| IP address | `192.168.56.20` |
| Subnet mask | `255.255.255.0` |
| Default gateway | *(blank)* |
| Preferred DNS | *(blank)* |

6. Click OK and close

### On Kali Linux (Attacker)

Kali uses NetworkManager. Easiest via the terminal:

```bash
# Find the Host-Only interface name
ip a

# Set static IP permanently via nmcli (replace eth1 with actual interface name)
sudo nmcli con mod "Wired connection 2" ipv4.addresses 192.168.56.40/24 ipv4.method manual
sudo nmcli con up "Wired connection 2"
```

Or via the network icon in the Kali taskbar:
1. Click network icon -> **Edit Connections**
2. Select the Host-Only adapter -> Edit
3. IPv4 Settings -> Method: **Manual**
4. Add address: `192.168.56.40`, netmask `24`, gateway blank
5. Save and reconnect

---

## IP Address Reference

| VM | Host-Only IP | NAT IP | Role |
|---|---|---|---|
| Host Machine | `192.168.56.1` | — | VMnet1 gateway (auto-assigned) |
| Wazuh Server | `192.168.56.10` | `192.168.232.x` | SIEM manager — agents connect here |
| Windows Victim | `192.168.56.20` | `192.168.232.x` | RDP target (port 3389) |
| Linux Victim | `192.168.56.30` | `192.168.232.x` | SSH target (port 22) |
| Kali Attacker | `192.168.56.40` | `192.168.232.x` | Attack source |

---

## Verifying Connectivity

Once all VMs have their static IPs set, verify the network before proceeding to Wazuh installation.

From **Kali**, we can ping each victim and the Wazuh server:
```bash
ping -c 3 192.168.56.10   # Wazuh
ping -c 3 192.168.56.20   # Windows
ping -c 3 192.168.56.30   # Linux
```

From the **Wazuh Server**, we can ping both victims:
```bash
ping -c 3 192.168.56.20   # Windows
ping -c 3 192.168.56.30   # Linux
```

---

## Security Note

VMnet1 is completely isolated from the real home network and the internet by design. Attack traffic generated during simulations (brute force, port scans, etc.) stays within the `192.168.56.0/24` subnet and never reaches your physical network. The NAT adapters (VMnet8) only provide outbound access to the internet.
