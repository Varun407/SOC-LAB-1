# Project 0 : SOC Lab Environment Setup

## Overview

Before diving into detection engineering, threat hunting, or attack simulation, every SOC analyst needs a safe, isolated environment to practice in. This project documents how I built a self-contained mini network from scratch, a firewall, a Linux server, a Windows endpoint, and an attacker machine, all running as virtual machines on a single host.

This lab is the foundation for the rest of my security projects: attack surface reduction, SIEM correlation, intrusion detection, and more all build on top of what's set up here.

## What I Built

A small, segmented network with a firewall sitting between the internet and three internal machines:

- **pfSense** -- acts as the firewall and router, controlling what traffic is allowed in and out
- **Ubuntu Server** -- the machine that will later run the Elastic Stack and act as a monitored target
- **Windows VM** -- a Windows endpoint that will later run Sysmon for event logging
- **Kali Linux** -- the "attacker" machine used to simulate scans and attacks in later projects

Everything runs inside VirtualBox on a single Windows host, with the three internal machines sitting behind pfSense on their own private network, not directly exposed to the internet or my home network.

## Network Architecture

![Network topology diagram](screenshots/soc_lab_network_topology_v2.png)

Internet traffic comes in through pfSense's WAN interface. pfSense then routes to an internal LAN segment where the three other VMs live. This means Kali, Ubuntu Server, and the Windows VM can all reach each other and the internet, but they're isolated from my actual home network, a safe sandbox to break things in later projects without any real-world risk.

## How I Built It

### 1. Planning and tooling

Before touching any VM settings, I made sure my hypervisor (VirtualBox) was installed and working, and set up a simple folder structure to track the build as I went, screenshots, notes, and a running status list for each step.

### 2. Downloading the install media

I gathered the ISOs I'd need: Ubuntu Server (for the Linux target), pfSense (for the firewall), and a Windows 11 Enterprise evaluation ISO (for the Windows endpoint). Kali Linux was already set up from earlier testing.

![Ubuntu Server download](screenshots/ubuntu-download.png)
![pfSense download page](screenshots/pfsense-download-1.png)
![pfSense installer download in progress](screenshots/pfsense-download-2.png)
![Windows 11 Enterprise evaluation download](screenshots/windows-download.png)

### 3. Building and configuring pfSense

I created the pfSense VM (FreeBSD 64-bit, 2GB RAM, 20GB disk) and pointed it at the pfSense ISO.

![pfSense VM creation](screenshots/pfsense-setup-1.png)
![pfSense hardware settings](screenshots/pfsense-setup-2.png)
![pfSense virtual disk settings](screenshots/pfsense-setup-3.png)
![pfSense Network Adapter settings](screenshots/pfsense-setup-network.png),![pfSense Network Adapter settings](screenshots/pfsense-setup-network-2.png)

<details>
<summary>Installation walkthrough</summary>

![pfSense copyright notice](screenshots/pfsense-installation-1.png)
![pfSense installer welcome — install or rescue shell](screenshots/pfsense-installation-2.png)
![pfSense WAN interface selection](screenshots/pfsense-installation-3.png)
![pfSense LAN interface selection](screenshots/pfsense-installation-4.png)
![pfSense interface assignment confirmation](screenshots/pfsense-installation-5.png)
![pfSense filesystem and partition scheme options](screenshots/pfsense-installation-6.png)
![pfSense disk selection](screenshots/pfsense-installation-7.png)
![pfSense post-installation setup details](screenshots/pfsense-installation-8.png)
![pfSense installation complete — reboot prompt](screenshots/pfsense-installation-9.png)

</details>

I gave pfSense two network adapters,one facing the internet (WAN, on NAT) and one facing the internal lab network (LAN, on an Internal Network I named `soc-lab-lan`) and then assigned `em0` as WAN and `em1` as LAN during setup.

After the first boot, the console showed the default interface status before any reconfiguration:

![pfSense console after first boot](screenshots/pfsense-config-1.png)

I then configured the LAN interface with a static IP (`192.168.10.1/24`) and turned on DHCP so the other VMs would automatically get addresses in the `192.168.10.100–200` range.

![pfSense LAN IP configuration](screenshots/pfsense-config-2.png)
![pfSense DHCP and IPv6 configuration](screenshots/pfsense-config-3.png)
![pfSense LAN configuration complete](screenshots/pfsense-config-4.png)

To reach pfSense's web dashboard, I connected Kali to the internal network, let it pick up a DHCP lease, and logged in through the browser to finish setup and secure the admin account.

![pfSense web login](screenshots/pfsense-check-1.png)
![pfSense setup wizard](screenshots/pfsense-check-2.png)

### 4. Connecting Kali to the internal network

Kali was originally set up on NAT for earlier testing, so I switched its network adapter over to the same internal network as pfSense.

![Kali network adapter - before, on NAT](screenshots/kali-vm-network.png)
![Kali network adapter - switched to Internal Network](screenshots/kali-vm-network-mode.png)

With the adapter switched, Kali picked up a DHCP-assigned address from pfSense and I confirmed connectivity from the command line.

![Kali network interface with an assigned address](screenshots/kali-vm-config-1.png)
![Kali requesting an additional lease with dhcpcd](screenshots/kali-vm-config-2.png)
![Kali interface showing both DHCP-assigned addresses](screenshots/kali-vm-config-3.png)

### 5. Building the Ubuntu Server VM

I built the Ubuntu Server VM, connected it to the internal LAN, and confirmed it picked up an IP from pfSense.

![Ubuntu Server VM settings — memory and boot order](screenshots/ubuntu-installation-1.png)
![Ubuntu Server network adapter set to Internal Network](screenshots/ubuntu-network-mode.png)
![Ubuntu Server first login](screenshots/ubuntu-setup-1.png)

I ran a system update and confirmed internet access through pfSense with a ping test.

![Ubuntu Server update command entered](screenshots/ubuntu-setup-2.png)
![Ubuntu Server update running](screenshots/ubuntu-setup-3.png)
![Ubuntu Server ping test confirming connectivity](screenshots/ubuntu-config.png)

### 6. Building the Windows VM

I built the Windows VM with TPM 2.0, EFI, and Secure Boot enabled (required for Windows 11), pointed at the Windows evaluation ISO, and connected it to the internal LAN.

![Windows VM creation with ISO selected](screenshots/windows-vm-installation-1.png)
![Windows VM unattended setup details](screenshots/windows-vm-installation-2.png)
![Windows VM hardware settings — EFI enabled](screenshots/windows-vm-installation-3.png)
![Windows VM virtual disk settings](screenshots/windows-vm-installation-4.png)
![Windows VM created — TPM, EFI, and Secure Boot confirmed](screenshots/windows-vm-installation-5.png)

Partway through, the install stalled because my host had Windows' Core Isolation (Memory Integrity) enabled, which forced VirtualBox into an unaccelerated emulation mode instead of using hardware virtualization. Disabling it in Windows Security resolved the slowdown.

![Disabling Memory Integrity in Windows Security](screenshots/windows-vm-installation-6.png)

With that fixed, the install completed normally.

![Windows installation progress](screenshots/windows-vm-installation-7.png)
![Windows installation progressing further](screenshots/windows-vm-installation-8.png)
![Windows installation nearing completion](screenshots/windows-vm-installation-9.png)

Once installed, I confirmed the system specs and verified the Windows VM had a LAN IP from pfSense with working internet access.

![Windows VM system info](screenshots/windows-vm-check-1.png)
![Windows VM ipconfig and ping test](screenshots/windows-vm-check-2.png)

### 7. Validating the whole network

With all four machines built, I checked pfSense's DHCP lease table to confirm every VM was properly connected and getting addresses from the firewall, a clean, single view proving the whole lab was talking to itself correctly.

![pfSense DHCP leases showing all connected VMs](screenshots/devices-check.png)

### 8. Snapshotting a clean baseline

Finally, I took a "clean baseline" snapshot of each VM. Since upcoming projects involve intentionally breaking things like disabling services, running attacks, tuning firewall rules.This gives me a known-good state to reset back to at any point.

![Taking a clean-baseline snapshot](screenshots/snapshots.png)

## Result

A fully working, isolated 4-machine lab:

| Machine | Role | IP |
|---|---|---|
| pfSense | Firewall / router | `192.168.10.1` |
| Ubuntu Server | Future Elastic Stack host | `192.168.10.102` (DHCP) |
| Windows VM | Future Sysmon-monitored endpoint | `192.168.10.103` (DHCP) |
| Kali Linux | Attacker machine | `192.168.10.100` (DHCP) |

This lab is now ready to serve as the foundation for the next projects.

