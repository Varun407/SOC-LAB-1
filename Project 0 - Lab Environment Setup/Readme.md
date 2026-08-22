# Project 0: SOC Lab Environment Setup

**A fully virtualized, segmented security lab built from scratch: firewall, monitored endpoints, and an attacker machine, as the foundation for a hands-on SOC and security engineering portfolio.**

## Why I built this

Every project after this one (SIEM correlation, intrusion detection, honeypots, cloud IAM) needs somewhere safe to actually happen. Rather than spinning up throwaway VMs for each one, I built a proper lab once: a firewall separating an isolated internal network from the internet, with a Linux server, a Windows endpoint, and an attacker machine sitting behind it. Everything here is reusable, so the next five projects all build directly on top of this environment instead of starting from zero.

**Tech stack:** VirtualBox, pfSense (FreeBSD), Ubuntu Server 26.04 LTS, Windows 11 Enterprise, Kali Linux

## Architecture

![Network topology diagram](screenshots/soc_lab_network_topology_v2.png)

```
                         Internet
                            │
                          WAN (NAT)
                            │
                       ┌─────────┐
                       │ pfSense │
                       └────┬────┘
                            │
                     soc-lab-lan
                   192.168.10.0/24
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
      ┌───────┐        ┌──────────┐      ┌──────────┐
      │ Kali  │        │ Windows  │      │ Ubuntu   │
      │ Linux │        │  11 VM   │      │ Server   │
      │.100   │        │  .103    │      │  .102    │
      └───────┘        └──────────┘      └──────────┘
```

All four machines run on a single Windows host under VirtualBox. The three internal VMs sit on a private, non-routable segment (`soc-lab-lan`) behind pfSense, isolated from my home network, so later projects can safely run scans, exploits, and traffic captures without any real-world blast radius.

| Machine | Role | IP | Assignment |
|---|---|---|---|
| pfSense | Firewall / router, DHCP server | `192.168.10.1` | Static |
| Kali Linux | Attacker / testing machine | `192.168.10.100` | DHCP |
| Ubuntu Server | Future Elastic Stack host | `192.168.10.102` | DHCP |
| Windows 11 VM | Future Sysmon-monitored endpoint | `192.168.10.103` | DHCP |

## How I built it

### Planning and tooling

Before touching any VM settings, I made sure VirtualBox was installed and working, and set up a simple folder structure to track the build as I went.

### Downloading the install media

I gathered Ubuntu Server 26.04 LTS, pfSense CE, and a Windows 11 Enterprise evaluation ISO. Kali was already set up from earlier testing.

![Ubuntu Server download](screenshots/ubuntu-download.png)
![pfSense download page](screenshots/pfsense-download-1.png)
![pfSense installer download in progress](screenshots/pfsense-download-2.png)
![Windows 11 Enterprise evaluation download](screenshots/windows-download.png)

### Building and configuring pfSense

pfSense is the center of the whole lab; everything else routes through it. I gave it two virtual NICs: WAN on NAT for internet access, and LAN on a dedicated VirtualBox Internal Network I named `soc-lab-lan`.

![pfSense VM creation](screenshots/pfsense-setup-1.png)
![pfSense hardware settings](screenshots/pfsense-setup-2.png)
![pfSense virtual disk settings](screenshots/pfsense-setup-3.png)
![pfSense network adapter settings](screenshots/pfsense-setup-network.png)
![pfSense network adapter settings, adapter 2](screenshots/pfsense-setup-network-2.png)

<details>
<summary>Full installation walkthrough</summary>

![pfSense copyright notice](screenshots/pfsense-installation-1.png)
![pfSense installer welcome, install or rescue shell](screenshots/pfsense-installation-2.png)
![pfSense WAN interface selection](screenshots/pfsense-installation-3.png)
![pfSense LAN interface selection](screenshots/pfsense-installation-4.png)
![pfSense interface assignment confirmation](screenshots/pfsense-installation-5.png)
![pfSense filesystem and partition scheme options](screenshots/pfsense-installation-6.png)
![pfSense disk selection](screenshots/pfsense-installation-7.png)
![pfSense post-installation setup details](screenshots/pfsense-installation-8.png)
![pfSense installation complete, reboot prompt](screenshots/pfsense-installation-9.png)

</details>

During the guided install I assigned `em0` to WAN and `em1` to LAN. Once it booted, I set a static LAN address (`192.168.10.1/24`) and turned on DHCP across `192.168.10.100 to 200`, so every machine I added afterward would provision itself automatically.

![pfSense console after first boot](screenshots/pfsense-config-1.png)
![pfSense LAN IP configuration](screenshots/pfsense-config-2.png)
![pfSense DHCP and IPv6 configuration](screenshots/pfsense-config-3.png)
![pfSense LAN configuration complete](screenshots/pfsense-config-4.png)

Then I confirmed the web dashboard was reachable from the LAN and rotated the default admin password off `pfsense`.

![pfSense web login](screenshots/pfsense-check-1.png)
![pfSense setup wizard](screenshots/pfsense-check-2.png)

### Connecting Kali to the internal network

Kali was still sitting on NAT from earlier testing, so I moved its adapter onto `soc-lab-lan` and confirmed it picked up a lease from pfSense.

![Kali network adapter, before, on NAT](screenshots/kali-vm-network.png)
![Kali network adapter, switched to Internal Network](screenshots/kali-vm-network-mode.png)
![Kali network interface with an assigned address](screenshots/kali-vm-config-1.png)
![Kali requesting an additional lease with dhcpcd](screenshots/kali-vm-config-2.png)
![Kali interface showing both DHCP-assigned addresses](screenshots/kali-vm-config-3.png)

### Building the Ubuntu Server VM

This machine will eventually host the Elastic Stack, so getting its networking right mattered. I built it, put it on the LAN segment, patched it, and confirmed it could reach the internet through pfSense.

![Ubuntu Server VM settings, memory and boot order](screenshots/ubuntu-installation-1.png)
![Ubuntu Server network adapter set to Internal Network](screenshots/ubuntu-network-mode.png)
![Ubuntu Server first login](screenshots/ubuntu-setup-1.png)
![Ubuntu Server update command entered](screenshots/ubuntu-setup-2.png)
![Ubuntu Server update running](screenshots/ubuntu-setup-3.png)
![Ubuntu Server ping test confirming connectivity](screenshots/ubuntu-config.png)

### Building the Windows VM

Windows 11 has real hardware requirements: TPM 2.0, UEFI, Secure Boot. I made sure all three were enabled before install.

![Windows VM creation with ISO selected](screenshots/windows-vm-installation-1.png)
![Windows VM unattended setup details](screenshots/windows-vm-installation-2.png)
![Windows VM hardware settings, EFI enabled](screenshots/windows-vm-installation-3.png)
![Windows VM virtual disk settings](screenshots/windows-vm-installation-4.png)
![Windows VM created, TPM, EFI, and Secure Boot confirmed](screenshots/windows-vm-installation-5.png)

The first install attempt stalled indefinitely at the EFI boot stage. Digging into the VirtualBox VM log pointed to the real cause: Windows' Core Isolation (Memory Integrity) was enabled on my host, which forced VirtualBox to fall back to unaccelerated software emulation instead of using hardware virtualization (VT-x). Turning Memory Integrity off fixed it immediately, a good reminder to check host-level virtualization settings whenever a VM's performance doesn't add up.

![Disabling Memory Integrity in Windows Security](screenshots/windows-vm-installation-6.png)
![Windows installation progress](screenshots/windows-vm-installation-7.png)
![Windows installation progressing further](screenshots/windows-vm-installation-8.png)
![Windows installation nearing completion](screenshots/windows-vm-installation-9.png)

Once it was up, I confirmed the system specs and validated LAN and internet connectivity.

![Windows VM system info](screenshots/windows-vm-check-1.png)
![Windows VM ipconfig and ping test](screenshots/windows-vm-check-2.png)

### Validating the whole network

With all four machines built, the real test was checking that they all actually talk to each other through pfSense, not just individually, but as a network. pfSense's DHCP lease table gave me that in one view.

![pfSense DHCP leases showing all connected VMs](screenshots/devices-check.png)

### Snapshotting a clean baseline

Last step: a `clean-baseline` snapshot of every VM. The next few projects intentionally break things (disabled services, live exploits, malicious traffic), so having a known-good state to roll back to isn't optional. It's how repeatable testing works at all.

![Taking a clean-baseline snapshot](screenshots/snapshots.png)

## Lessons learned

**Networking.** Building this from scratch made concepts like WAN/LAN separation, DHCP scoping, and gateway routing click in a way that reading about them never did, especially watching pfSense's console output confirm each change in real time.

**Virtualization.** VM networking modes (NAT vs. Internal Network) look similar in the settings menu but behave completely differently in practice. And the Windows install issue was a good lesson that a "stuck" VM isn't always a VM problem. Sometimes it's the host's own security settings getting in the way.

**Lab design.** Isolating the lab network from my home network, keeping snapshots as recovery points, and never putting real credentials anywhere in the environment aren't just best practices. They're what make it safe to actually run attacks and exploits here later without worry.

## What's next

This environment is the base layer for the rest of the portfolio:

1. **Attack Surface Reduction**: hardening the Ubuntu Server target
2. **Network Intrusion Detection Engineering**: custom Snort rules validated against Kali-driven attacks
3. **SOC Mini Lab with SIEM Correlation**: Elastic Stack on Ubuntu Server, Sysmon on Windows
4. **Honeypot Threat Intelligence Lab**: Cowrie + ELK for real attacker behavior capture
5. **Cloud IAM Hardening Lab**: AWS IAM privilege escalation and remediation

## Disclaimer

This lab is for educational purposes and authorized testing only. All attack simulations in this and future projects run strictly inside this isolated environment, against systems I own.