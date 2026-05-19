# Detecting Basic Network Reconnaissance in a Dual-Machine Home Lab

**AttackBox:** Kali Linux VM  
**DefenseBox:** Windows 11 + Wireshark  
**Completed:** 2026-05-11  
**Author:** Malakh Fuller

> **Privacy note:** Internal lab IP addresses have been anonymized in this writeup and related screenshots. The original testing was performed only on my own local home lab network.

## Objective

After obtaining my Security+ certification and beginning to study CySA+ concepts, I wanted to move beyond theory and put those skills into practice in a home lab.

I had been learning from TryHackMe rooms, but I wanted to build something of my own: a small attack/defense lab where I could generate controlled network activity, observe it from the defender’s perspective, and document the results in a SOC-style format.

## Tools and Technologies

- **AttackBox:** Oracle VirtualBox, Kali Linux VM, Nmap
- **DefenseBox:** Windows 11, Wireshark
- **Concepts:** IP addressing, ICMP, TCP ports, Windows networking services, basic reconnaissance detection
- **Prior Knowledge:** CompTIA A+, Network+, Security+, CySA+ study, and TryHackMe SOC-related rooms

## Lab Addressing

For privacy, the IP addresses below are anonymized:

- **AttackBox:** `192.168.56.10`
- **DefenseBox:** `192.168.56.20`
- **Lab Network:** `192.168.56.0/24`

## Summary

This project documents the setup of a dual-machine home lab for practicing basic attack and defense techniques using industry-standard tools.

The first test was simple: use Kali Linux to generate controlled reconnaissance traffic against the Windows DefenseBox, then use Wireshark to confirm that the activity could be observed from the defender’s perspective.

## Methodology

### 1. Preparing the AttackBox

I started by setting up Kali Linux in a virtualized environment rather than installing it directly onto physical hardware.

- Downloaded and installed Oracle VirtualBox for Windows.
- Downloaded a pre-built Kali Linux VirtualBox image from Kali’s official “Get Kali” page.
- Opened VirtualBox and imported the `kali-linux-2026.1-virtualbox-amd64` VirtualBox image.
- Logged into Kali using the default credentials.
- Opened a terminal and ran:

```bash
sudo apt update
sudo apt upgrade -y
```

After the update completed, I created a VirtualBox snapshot so I would have a clean reset point before moving further into lab work.

<img width="2066" height="761" alt="github_WiresharkWorking" src="https://github.com/user-attachments/assets/a8517b86-218d-479c-9291-12723dc0dc40" />

### 2. Configuring the Network

After the initial setup, I checked the network configuration and found that the Kali VM was behind VirtualBox’s default NAT network. That worked for internet access, but it was not ideal for this lab because I wanted the Kali VM to appear as a separate system on the same local network as the Windows DefenseBox.

To correct this:

- Shut down Kali using:

```bash
sudo shutdown now
```

- Opened VirtualBox Manager.
- Selected the Kali VM.
- Went to **Settings → Network**.
- Changed the adapter from **NAT** to **Bridged Adapter**.
- Restarted Kali and ran:

```bash
ip addr
```

At this point, the Kali AttackBox and Windows DefenseBox were on the same lab network.

### 3. Basic Reconnaissance - Part I

The first goal was to confirm that the Kali AttackBox could reach the Windows DefenseBox.

From Kali, I ran:

```bash
ping -c 4 192.168.56.20
```

The DefenseBox responded successfully. I also confirmed the Kali VM’s own network address with:

```bash
ip addr
```

With both checks working, I confirmed a functioning two-machine lab path between the AttackBox and DefenseBox.

Next, I ran a small Nmap scan against only two ports on the DefenseBox:

```bash
nmap -Pn -p 80,443 192.168.56.20
```

Both ports returned as `filtered`.

In this lab context, the filtered result was consistent with firewall filtering or dropped probes. The important part, however, was that Wireshark on the DefenseBox showed the attempted traffic from the Kali AttackBox.

<img width="1877" height="838" alt="github_project1_nmap_2port_scan" src="https://github.com/user-attachments/assets/e7daf952-80af-46bc-8e4e-b2d221fded94" />

### 4. Basic Reconnaissance - Part II: Nmap Scan

After confirming basic connectivity, I ran a slightly broader but still controlled Nmap scan from the Kali AttackBox to the Windows DefenseBox:

```bash
nmap -Pn -p 22,80,135,139,443,445,3389 192.168.56.20
```

I used this scan to check a small set of common Windows and remote-access ports:

- `22` - SSH
- `80` - HTTP
- `135` - Windows RPC
- `139` - NetBIOS
- `443` - HTTPS
- `445` - SMB
- `3389` - Remote Desktop

<img width="1386" height="1135" alt="github_nmap_common_ports_kali_result" src="https://github.com/user-attachments/assets/10352982-9697-4a1a-9c7b-30398d9d2446" />

Nmap identified the host as up and reported three open Windows-related services:

- `135/tcp` - msrpc
- `139/tcp` - netbios-ssn
- `445/tcp` - microsoft-ds

The following ports were reported as filtered:

- `22/tcp` - SSH
- `80/tcp` - HTTP
- `443/tcp` - HTTPS
- `3389/tcp` - RDP

From a SOC perspective, this activity represents basic network reconnaissance. The source host attempted to identify exposed services on the target system. Wireshark on the DefenseBox confirmed TCP traffic between the Kali AttackBox and the Windows host during the scan window.

To refine the Wireshark view, I used the following display filter:

```text
ip.src == 192.168.56.10 && ip.dst == 192.168.56.20 && tcp
```

This allowed me to focus only on TCP traffic from the AttackBox to the DefenseBox.

## Key Findings

- Successfully built a dual-machine attack/defense lab using a Kali Linux VM as the AttackBox and a Windows 11 desktop as the DefenseBox.
- Confirmed both systems were on the same local network after changing the Kali VM from NAT to Bridged Adapter mode.
- Verified basic connectivity from Kali to the Windows DefenseBox using ICMP ping.
- Captured baseline ICMP traffic in Wireshark to confirm the DefenseBox could observe traffic from the AttackBox.
- Ran a limited two-port Nmap scan against ports `80` and `443`; both ports returned as filtered.
- Ran a second limited scan against common Windows and remote-access ports: `22`, `80`, `135`, `139`, `443`, `445`, and `3389`.
- Nmap reported `135/tcp`, `139/tcp`, and `445/tcp` as open, which is consistent with common Windows networking services.
- Wireshark confirmed TCP traffic from the Kali AttackBox to the Windows DefenseBox during the scan window.

## Key Competencies Demonstrated

- Home lab setup and troubleshooting
- Virtual machine configuration
- Basic Linux command-line usage
- Network adapter configuration in VirtualBox
- Local network IP identification
- Controlled use of Nmap for service discovery
- Wireshark packet capture and filtering
- Basic TCP/IP traffic analysis
- Documentation of lab activity and findings
- SOC-style interpretation of reconnaissance behavior

## Employer-Relevant Skills

This project demonstrates the ability to set up a small lab environment, generate controlled network activity, capture that activity from a defender’s perspective, and document the results clearly.

These skills are directly relevant to entry-level SOC work. Analysts are often expected to review alerts, validate source and destination activity, identify exposed services, and determine whether observed behavior is expected or suspicious.

This project also shows basic familiarity with tools and concepts commonly seen in SOC environments, including Kali Linux, Nmap, Wireshark, IP addressing, TCP ports, Windows services, and network reconnaissance.

## SOC Relevance

From a SOC perspective, this activity represents basic internal reconnaissance. In this lab, the scan was authorized and expected. In a production environment, similar activity would require context before assigning severity.

A scan from an approved vulnerability scanner or administrator workstation may be normal. A scan from an unknown endpoint, user workstation, or newly observed host would be more suspicious, especially if followed by authentication attempts, SMB activity, RDP attempts, or additional scans against multiple systems.

For this lab, the final outcome was:

- **Verdict:** Authorized lab activity / confirmed reconnaissance behavior
- **Severity:** Low in lab context; potentially medium in a production environment if unauthorized
- **Recommended Action:** Validate source ownership, confirm whether scanning was approved, review exposed services, and monitor for follow-on activity such as authentication attempts against SMB or RDP.

## HUMINT to SOC Translation

My previous work in competitive intelligence and HUMINT-style research involved collecting fragmented information, validating sources, reconciling conflicting signals, and turning incomplete data into clear written assessments.

This lab used a similar workflow in a technical setting.

Instead of evaluating human-source information, I reviewed network traffic and scan results. Instead of validating interview data, I compared Nmap output against Wireshark evidence.

The core process was familiar: identify the source, review the activity, validate what happened, assess the significance, and document the finding in a way another person could understand.

## Author

- GitHub: [Malakh Fuller](https://github.com/MalakhFuller)
- LinkedIn: [Malakh Fuller](https://www.linkedin.com/in/malakhfuller/)

