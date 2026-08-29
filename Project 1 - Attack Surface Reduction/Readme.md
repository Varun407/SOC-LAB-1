# Project 1: Attack Surface Reduction & Network Segmentation Lab

**A two-phase project on top of my SOC lab: first reducing a deliberately vulnerable Ubuntu server's attack surface, then redesigning the network into segmented USER, SERVER, and ADMIN zones to demonstrate how firewall policy limits lateral movement after a compromise.**

## Why I built this

Closing open ports on a server is the easy part. The harder, more interesting question is: why was that service exposed in the first place, does the server actually need it, and if it gets compromised anyway, how far can an attacker actually get? This project answers that in two stages. Phase 1 treats Ubuntu Server as a neglected box with services piling up over time, assessed and hardened the way a real analyst would. Phase 2 goes further: even after hardening, a flat network still lets a compromised machine reach everything else directly, so I rebuilt the lab into three separated zones and used pfSense firewall rules to prove that lateral movement is actually blocked, not just theoretically reduced.

**Tech stack:** VirtualBox, pfSense (FreeBSD), Ubuntu Server 26.04 LTS, Windows 11, Kali Linux, Nmap, Netcat, OpenSSH

## Phase 1 architecture

![Phase 1 topology](screenshots/phase1-topology.svg)

A single flat LAN behind pfSense. Kali and Ubuntu Server share the same subnet, which is realistic for a small lab but also the exact problem Phase 2 addresses: any machine on this network can reach any other machine directly.

## Phase 2 architecture

![Phase 2 segmentation](screenshots/phase2-segmentation.svg)

| Zone | Network | Gateway | Machine | Address |
|---|---|---|---|---|
| USER | `192.168.10.0/24` | `192.168.10.1` | Kali (attacker) | `192.168.10.100` |
| SERVER | `192.168.20.0/24` | `192.168.20.1` | Ubuntu (target) | `192.168.20.102` |
| ADMIN | `192.168.30.0/24` | `192.168.30.1` | Windows (admin) | `192.168.30.10` |

pfSense now has four interfaces instead of two: WAN for internet, and three internal interfaces (LAN, OPT1, OPT2) each mapped to its own isolated VirtualBox internal network.

## Phase 1: building and reducing a vulnerable baseline

### Baseline check

Before building anything, I confirmed Kali's own network position and ran a full port scan against Ubuntu Server to establish a true starting point: nothing installed yet, every port closed.

<details>
<summary>Screenshots: Initial port scanning</summary>

![Kali network check](screenshots/01.png)

![Kali network check](screenshots/02.png)

![Kali network check](screenshots/03.png)

![Kali network check](screenshots/04.png)
</details>

### Setting up the vulnerable server

Rather than starting from a hardened box and pretending there was something to fix, I deliberately built Ubuntu Server the way a neglected real-world server tends to look: SSH installed with defaults, a web server left running from an old test, an FTP server nobody remembers enabling, and Telnet still reachable.

```
sudo apt update
sudo apt install openssh-server apache2 vsftpd telnetd -y
```

<details>
<summary>Screenshots: installing the services and confirming each is active</summary>

![Ubuntu login and starting the install](screenshots/05.png)
![Package install continuing, sudo retry](screenshots/06.png)
![Apache modules and services enabling](screenshots/07.png)
![SSH and Apache confirmed active](screenshots/08.png)
![vsftpd active, inetutils-inetd inactive](screenshots/09.png)

</details>

### The Telnet snag, and the fix

`telnetd` installed cleanly, but the underlying `inetutils-inetd` super-server kept showing `inactive (dead)`. Telnet doesn't run as its own daemon: it's launched on demand by inetd based on `/etc/inetd.conf`, and every line in that file ships commented out by default. Once I found the right line and uncommented it, the service came up immediately.

```
sudo sed -i 's/^#<off># telnet/telnet/' /etc/inetd.conf
sudo systemctl restart inetutils-inetd
```

<details>
<summary>Screenshots: diagnosing and fixing the inetd/Telnet issue</summary>

![inetd showing inactive, skipped due to exec-condition](screenshots/10.png)
![Reviewing inetd.conf and applying the fix](screenshots/11.png)
![inetd restarted, now active and running](screenshots/12.png)
![Port 23 confirmed listening](screenshots/13.png)

</details>

### Attacker reconnaissance from Kali

With the baseline in place, I switched to Kali and worked through this the way an attacker actually would: find live hosts first, then scan the one that looks interesting, then validate what was found by hand rather than trusting the scanner blindly.

```
ip a
sudo nmap -sn 192.168.10.0/24
sudo nmap -sV -p- 192.168.10.102
sudo nmap -sC -sV -oN ubuntu-baseline-scan.txt 192.168.10.102
```

<details>
<summary>Screenshots: host discovery and full port scan</summary>

![Kali interface check](screenshots/14.png)
![Kali interface, second view](screenshots/15.png)
![Ping sweep finding 3 live hosts](screenshots/16.png)
![Full port and version scan against Ubuntu](screenshots/17.png)
![Scan results saved to ubuntu-baseline-scan.txt](screenshots/18.png)

</details>

The scan came back with four open ports:

| Port | Service | Version |
|---|---|---|
| 21/tcp | FTP | vsftpd 3.0.5 |
| 22/tcp | SSH | OpenSSH 10.2p1 (Ubuntu) |
| 23/tcp | Telnet | inetutils-telnetd |
| 80/tcp | HTTP | Apache 2.4.66 |

### Validating each finding manually

A port showing "open" in Nmap isn't the same as confirming it's actually usable. I connected to each service by hand to see what an attacker would really get.

**FTP**: connected and tried anonymous login, got rejected. Checked the browser view of the Apache default page and grabbed its headers with curl. Connected to Telnet directly and got a real login prompt. Grabbed the SSH banner with netcat.

```
ftp 192.168.10.102
curl -I http://192.168.10.102
telnet 192.168.10.102 23
nc -nv 192.168.10.102 22
```

<details>
<summary>Screenshots: manual FTP, HTTP, Telnet, and SSH validation</summary>

![FTP anonymous login attempt, rejected](screenshots/19.png)
![Apache default page in the browser](screenshots/20.png)
![Apache headers via curl](screenshots/21.png)
![Telnet manual connection, login prompt exposed](screenshots/22.png)
![SSH banner grab via netcat](screenshots/23.png)

</details>

Then I switched to the defender side and reviewed the actual configuration behind each finding: `/etc/inetd.conf` for Telnet, `vsftpd.conf` for FTP (anonymous access was off, but TLS was too, a real finding), and the effective SSH config via `sshd -T`, which showed password authentication still enabled.

<details>
<summary>Screenshots: defender-side config review for each service</summary>

![Telnet config confirmed in inetd.conf](screenshots/24.png)
![dpkg package check and SSH config with reconnaissance in the logs](screenshots/25.png)
![vsftpd config: anonymous off, SSL off](screenshots/26.png)
![Apache status and sshd -T effective config](screenshots/27.png)
![User account and full listening-socket baseline](screenshots/28.png)
![Full running-services baseline](screenshots/29.png)

</details>

### Hardening: Telnet, FTP, and Apache

For each of these three, I followed the same pattern: confirm the service isn't actually needed, disable it on Ubuntu, then go back to Kali and prove the port is genuinely closed, not just assumed closed.

**Telnet**: disabled the entry in `/etc/inetd.conf` and restarted the service.

<details>
<summary>Screenshots: Telnet hardening and Kali-side verification</summary>

![Disabling the Telnet entry and confirming port 23 no longer listens](screenshots/30.png)
![Kali: Telnet connection refused](screenshots/31.png)

</details>

**FTP**: stopped and disabled the service outright.
```
sudo systemctl disable vsftpd
sudo systemctl stop vsftpd
```

<details>
<summary>Screenshots: FTP hardening and Kali-side verification</summary>

![Full vsftpd.conf review, then disable and stop, port 21 no longer listening](screenshots/32.png)
![vsftpd disabled and stopped](screenshots/33.png)
![Kali: FTP connection refused](screenshots/34.png)

</details>

**Apache**: same pattern.
```
sudo systemctl disable apache2
sudo systemctl stop apache2
```

<details>
<summary>Screenshots: Apache hardening and Kali-side verification</summary>

![Apache banner re-check before hardening](screenshots/35.png)
![Apache disabled after a failed sudo retry](screenshots/36.png)
![Kali: HTTP 200 OK, then connection refused after hardening](screenshots/37.png)
![Kali: Apache connection refused, second confirmation](screenshots/38.png)

</details>

### SSH: the one service we kept, and hardened instead of removed

SSH is the odd one out here: it's the only service with a real administrative purpose, so the right move isn't to disable it, it's to harden it properly. That means moving off password authentication entirely, which meant first setting up a legitimate key-based login path so hardening the server couldn't accidentally lock out the admin.

I set up Windows as the legitimate admin workstation (separate from Kali, which stays purely as the attacker). Getting there took an unexpected detour: Windows reported OpenSSH Client as "Installed," but `ssh` wasn't recognized anywhere. The cause was a 32-bit PowerShell session: Windows silently redirects `System32` access for 32-bit processes to `SysWOW64`, so the shell literally couldn't see the real OpenSSH folder even though File Explorer could. Opening a proper 64-bit PowerShell session fixed it immediately.

<details>
<summary>Screenshots: the Windows OpenSSH client troubleshooting detour</summary>

![Installing OpenSSH Client via Add-WindowsCapability](screenshots/39.png)
![ssh -V not recognized, despite a successful install](screenshots/40.png)
![Correct 64-bit PowerShell session: Test-Path true, ssh -V works](screenshots/41.png)

</details>

With SSH working on Windows, I generated a key pair, added the public key to Ubuntu, confirmed passwordless login worked, and only then hardened the server config.

```
ssh-keygen -t ed25519
# public key added to ~/.ssh/authorized_keys on Ubuntu
ssh socadmin@192.168.10.102          # confirms passwordless login
# then, on Ubuntu:
sudo nano /etc/ssh/sshd_config       # PasswordAuthentication no
sudo systemctl restart ssh
```

<details>
<summary>Screenshots: SSH key generation, authorized_keys setup, and hardening</summary>

![Generating an Ed25519 key pair on Windows](screenshots/42.png)
![Displaying the public key](screenshots/43.png)
![Adding the key to Ubuntu's authorized_keys](screenshots/44.png)
![Passwordless SSH login confirmed from Windows](screenshots/45.png)
![Editing sshd_config](screenshots/46.png)
![PasswordAuthentication set to no](screenshots/47.png)
![Config reloaded, sshd -T confirms the change](screenshots/48.png)

</details>

Then verified from Kali, the actual attacker machine, that password authentication was refused entirely, with no password prompt offered at all, tried twice to be sure:

<details>
<summary>Screenshots: Kali attempting password login, refused both times</summary>

![First attempt: permission denied, publickey only](screenshots/49.png)
![Second attempt, same result](screenshots/50.png)

</details>

### Phase 1 result

Ran the exact same `nmap -sC -sV` scan that produced the original baseline, this time against the hardened server.

![Final hardened scan: only port 22 open](screenshots/51.png)

| Port | Before | After |
|---|---|---|
| 21 (FTP) | Open | Closed |
| 22 (SSH) | Open, password auth allowed | Open, key-only |
| 23 (Telnet) | Open | Closed |
| 80 (HTTP) | Open | Closed |

Four exposed services down to one, and the one that remains is the one that actually needs to be there, hardened instead of just left alone.

## Phase 2: why hardening alone wasn't enough

Even fully hardened, Ubuntu Server was still sitting on the same flat LAN as Kali. SSH being locked down doesn't matter much if an attacker who compromises any other machine on that same subnet can still reach the server directly for whatever *is* exposed, and can freely probe it for anything that gets exposed later. The real fix is architectural: separate the zones so that reaching the server requires actually passing through firewall policy, not just being on the same wire.

Before making any changes, I captured the starting point: pfSense's default LAN rules (allow everything) and the DHCP lease table showing all three machines still on one subnet.

<details>
<summary>Screenshots: the flat-network starting point</summary>

![pfSense default LAN rules before segmentation](screenshots/52.png)
![DHCP leases showing all three VMs on one subnet](screenshots/53.png)

</details>

### Building the SERVER zone

Added a second network adapter to Ubuntu Server and to pfSense, both connected to a new isolated VirtualBox internal network (`SERVER-LAN`), then assigned it as pfSense's OPT1 interface at `192.168.20.1/24`.

<details>
<summary>Screenshots: adding the SERVER-LAN adapter and configuring OPT1</summary>

![VirtualBox: pfSense's new SERVER-LAN adapter](screenshots/54.png)
![Ubuntu shows the new enp0s8 interface, no IP yet](screenshots/55.png)
![pfSense console: assigning em2 as OPT1](screenshots/56.png)
![Setting OPT1's IP to 192.168.20.1/24](screenshots/57.png)
![OPT1 configuration complete, DHCP left off](screenshots/58.png)
![Reviewing and backing up the existing Netplan config](screenshots/59.png)

</details>

On Ubuntu, I edited Netplan directly to give the new interface a static address, since this segment doesn't hand out DHCP.

```yaml
enp0s8:
  dhcp4: false
  dhcp6: false
  addresses:
    - 192.168.20.102/24
```

<details>
<summary>Screenshots: Netplan configuration and applying it</summary>

![Adding the static enp0s8 block](screenshots/60.png)
![netplan apply, interface confirmed with 192.168.20.102](screenshots/61.png)
![Confirming the OPT1 interface via the pfSense web GUI](screenshots/64.png)
![Static IPv4 192.168.20.1/24, no upstream gateway](screenshots/65.png)

</details>

The very first ping to the new gateway failed completely, a hundred percent packet loss, before the interface and firewall settings were fully in place. Once resolved, connectivity came up cleanly.

<details>
<summary>Screenshots: initial ping failure, then success</summary>

![First ping: 100% packet loss](screenshots/62.png)
![Related pfSense ICMP rule while troubleshooting](screenshots/63.png)
![Confirming the OPT1 interface via the pfSense web GUI](screenshots/64.png)
![Static IPv4 192.168.20.1/24, no upstream gateway](screenshots/65.png)
![Ping working after the fix](screenshots/66.png)
![Ping and traceroute succeed](screenshots/67.png)

</details>

### Testing SERVER-zone connectivity, then blocking it

First confirmed Kali could still reach Ubuntu across the new segment, expected, since no firewall rule existed yet.

```
ping -c 4 192.168.20.102
traceroute -n 192.168.20.102
nc -nv 192.168.20.102 22
```

<details>
<summary>Screenshots: pre-block connectivity from Kali</summary>

![SSH banner reachable across the new segment](screenshots/68.png)
![Creating the block rule: LAN interface, TCP, block action](screenshots/69.png)

</details>

Then created a pfSense rule blocking USER LAN traffic to Ubuntu on port 22, and tested again.

<details>
<summary>Screenshots: creating and applying the block rule</summary>

![Destination 192.168.20.102, port 22, description added](screenshots/70.png)
![Rule saved and active in the LAN ruleset](screenshots/71.png)
![Rule saved and active in the LAN ruleset](screenshots/72.png)

</details>

```
nc -nv 192.168.20.102 22
```

![Before and after: SSH open, then connection timed out](screenshots/73.png)

That single before/after pair, SSH banner grabbed cleanly, then the exact same command timing out, is the clearest evidence in the whole project that firewall policy, not just service hardening, is what actually stops lateral movement. As a side demonstration, I also restarted Apache to show that other services can still be selectively exposed while SSH stays policy-controlled.

![Apache restarted, port 80 listening again](screenshots/74.png)
![Apache restarted, port 80 listening again](screenshots/75.png)

### Building the ADMIN zone

Added a fourth interface to pfSense (`em3`, mapped to OPT2) and moved Windows onto its own isolated network at `192.168.30.0/24`. Windows came up with an APIPA address (`169.254.x.x`) since this new network had no DHCP configured, so I assigned it a static IP instead.

```
netsh interface ipv4 set address name="Ethernet" static 192.168.30.10 255.255.255.0 192.168.30.1
```

<details>
<summary>Screenshots: pfSense OPT2 setup, Windows moved to ADMIN LAN</summary>

![Windows adapter moved to SOC-Admin-LAN](screenshots/76.png)
![pfSense's new Adapter 4, same internal network](screenshots/77.png)
![pfSense console: assigning em3 as OPT2](screenshots/78.png)
![Interface assignment: WAN/LAN/OPT1/OPT2 all confirmed](screenshots/79.png)
![Setting OPT2's IP to 192.168.30.1/24](screenshots/80.png)
![OPT2 config complete, DHCP left off](screenshots/81.png)
![ifconfig em3 confirms OPT2 active](screenshots/82.png)
![Windows APIPA address before the fix](screenshots/83.png)

</details>

### Granting the ADMIN zone controlled access

Created two pfSense rules on the new OPT2 interface: allow ICMP from ADMIN to the pfSense gateway (for basic reachability testing), and allow TCP/22 specifically from ADMIN to the Ubuntu server. Nothing else.

<details>
<summary>Screenshots: creating the ADMIN firewall rules</summary>

![Creating the ICMP pass rule for OPT2 to the gateway](screenshots/84.png)
![Rule description: ADMIN to pfSense gateway](screenshots/85.png)
![Creating the TCP/22 rule, destination 192.168.20.102](screenshots/86.png)
![Rule description: ADMIN to Ubuntu (SSH)](screenshots/87.png)
![Both OPT2 rules live and applied](screenshots/88.png)

</details>

### A routing detail worth documenting

The first ADMIN → SERVER SSH attempt failed even with the firewall rule in place. The cause: Ubuntu's default route still pointed back toward the original USER-LAN gateway, so return traffic to the ADMIN subnet had nowhere correct to go. A temporary route fixed it immediately.

```
sudo ip route add 192.168.30.0/24 via 192.168.20.1 dev enp0s8
```

![Route table before and after the fix](screenshots/89.png)

The Ubuntu return route to the ADMIN subnet was initially implemented as a runtime-only route during troubleshooting. It was subsequently made persistent through Netplan and verified, ensuring that the ADMIN-to-SERVER routing survives an Ubuntu reboot.

<details>
<summary>Making route persistent through Netplan and verifying</summary>

![Route table before and after the fix](screenshots/91.png)
![Route table before and after the fix](screenshots/92.png)
![Route table before and after the fix](screenshots/93.png)
![Route table before and after the fix](screenshots/94.png)
![Route table before and after the fix](screenshots/95.png)
![Route table before and after the fix](screenshots/96.png)
![Route table before and after the fix](screenshots/97.png)

</details>

### Phase 2 result

The final verification is the strongest evidence in the project:

```
PS> ping 192.168.20.102
Request timed out.

PS> Test-NetConnection 192.168.20.102 -Port 22
TcpTestSucceeded : True
```

![Final ADMIN to SERVER verification: ping fails, TCP/22 succeeds](screenshots/90.png)

ICMP is blocked, SSH is allowed, on the exact same machine, from the exact same source. That's not an accident, it's the firewall enforcing policy at the protocol level, which is a more accurate real-world picture than "ping works so it's reachable."

| Path | Result |
|---|---|
| USER (Kali) → SERVER, TCP/22 | Blocked |
| ADMIN (Windows) → SERVER, TCP/22 | Allowed |
| ADMIN (Windows) → pfSense, ICMP | Allowed |

## Lessons learned

**Attack surface reduction is a judgment call, not a checklist.** The instinct to "just disable everything that isn't SSH" is wrong. The actual skill is asking, for each service, whether it has a legitimate purpose here, and treating "no" and "yes, but insecurely configured" as two different problems with two different fixes.

**A closed port and a segmented network solve different problems.** Hardening Ubuntu Server closed three unnecessary services, but it didn't change the fact that Kali could still reach whatever remained open. Segmentation is what actually limits what a compromised machine can reach, independent of how well any individual service is configured.

**Ping is not connectivity.** The clearest single lesson from this whole project: ICMP and TCP are controlled independently by firewall policy, and a failed ping tells you nothing definitive about whether a specific service is reachable. `Test-NetConnection -Port` (or `nc`) is the real test.

**Routing problems hide behind firewall problems.** When the ADMIN-to-SERVER SSH rule didn't work at first, the instinct was to suspect the firewall rule. The actual cause was Ubuntu's return route. Worth remembering that "the rule looks right" and "the traffic actually round-trips" are two different things to verify.

**Environment quirks cost real time and are worth documenting.** The inetd/Telnet service-naming confusion and the 32-bit PowerShell/OpenSSH issue weren't security findings, but they were genuine troubleshooting work, and writing them down is more honest, and more useful to future me, than pretending everything worked on the first try.

## What's next
1. **Network Intrusion Detection Engineering**: custom Snort rules validated against Kali-driven attacks
2. **SOC Mini Lab with SIEM Correlation**: Elastic Stack on Ubuntu Server, Sysmon on Windows
3. **Honeypot Threat Intelligence Lab**: Cowrie + ELK for real attacker behavior capture
4. **Cloud IAM Hardening Lab**: AWS IAM privilege escalation and remediatio

## Disclaimer

This lab is for educational purposes and authorized testing only. All attack simulations in this project run strictly inside this isolated lab environment, against systems I own.
