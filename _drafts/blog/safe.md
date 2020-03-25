---
layout: post
title: "HTB-Safe"
categories: HTB-Walkthrough
---

![safe](/assets/img/safe/safe1.png)


Safe is another box from TJNull's list of OSCP-like boxes from the HTB 'retired' archive. It is rated
as 'more challenging than OSCP, but good practice'.

nmap first.


<h3>Nmap</h3>

```
nmap -sV -Pn -p- --min-rate 10000 10.10.10.147 |tee -a safe.txt
```

```

Nmap scan report for 10.10.10.147
Host is up (0.19s latency).
Not shown: 65423 closed ports, 109 filtered ports
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4p1 Debian 10+deb9u6 (protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.25 ((Debian))
1337/tcp open  waste?
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port1337-TCP:V=7.80%I=7%D=3/19%Time=5E73E3C1%P=x86_64-pc-linux-gnu%r(NU
SF:LL,3E,"\x2017:28:40\x20up\x203\x20min,\x20\x200\x20users,\x20\x20load\x
SF:20average:\x200\.02,\x200\.02,\x200\.00\n")%r(GenericLines,63,"\x2017:2
SF:8:40\x20up\x203\x20min,\x20\x200\x20users,\x20\x20load\x20average:\x200
SF:\.02,\x200\.02,\x200\.00\n\nWhat\x20do\x20you\x20want\x20me\x20to\x20ec
SF:ho\x20back\?\x20\r\n")%r(GetRequest,71,"\x2017:28:46\x20up\x203\x20min,
SF:\x20\x200\x20users,\x20\x20load\x20average:\x200\.02,\x200\.02,\x200\.0
SF:0\n\nWhat\x20do\x20you\x20want\x20me\x20to\x20echo\x20back\?\x20GET\x20
SF:/\x20HTTP/1\.0\r\n")%r(HTTPOptions,75,"\x2017:28:46\x20up\x203\x20min,\
SF:x20\x200\x20users,\x20\x20load\x20average:\x200\.02,\x200\.02,\x200\.00
SF:\n\nWhat\x20do\x20you\x20want\x20me\x20to\x20echo\x20back\?\x20OPTIONS\
SF:x20/\x20HTTP/1\.0\r\n")%r(RTSPRequest,75,"\x2017:28:47\x20up\x203\x20mi
SF:n,\x20\x200\x20users,\x20\x20load\x20average:\x200\.02,\x200\.02,\x200\
SF:.00\n\nWhat\x20do\x20you\x20want\x20me\x20to\x20echo\x20back\?\x20OPTIO
SF:NS\x20/\x20RTSP/1\.0\r\n")%r(RPCCheck,3E,"\x2017:28:47\x20up\x203\x20mi
SF:n,\x20\x200\x20users,\x20\x20load\x20average:\x200\.02,\x200\.02,\x200\
SF:.00\n")%r(DNSVersionBindReqTCP,3E,"\x2017:28:52\x20up\x203\x20min,\x20\
SF:x200\x20users,\x20\x20load\x20average:\x200\.01,\x200\.02,\x200\.00\n")
SF:%r(DNSStatusRequestTCP,3E,"\x2017:28:58\x20up\x203\x20min,\x20\x200\x20
SF:users,\x20\x20load\x20average:\x200\.01,\x200\.01,\x200\.00\n")%r(Help,
SF:67,"\x2017:29:03\x20up\x203\x20min,\x20\x200\x20users,\x20\x20load\x20a
SF:verage:\x200\.01,\x200\.01,\x200\.00\n\nWhat\x20do\x20you\x20want\x20me
SF:\x20to\x20echo\x20back\?\x20HELP\r\n")%r(SSLSessionReq,64,"\x2017:29:03
SF:\x20up\x203\x20min,\x20\x200\x20users,\x20\x20load\x20average:\x200\.01
SF:,\x200\.01,\x200\.00\n\nWhat\x20do\x20you\x20want\x20me\x20to\x20echo\x
SF:20back\?\x20\x16\x03\n")%r(TerminalServerCookie,63,"\x2017:29:04\x20up\
SF:x203\x20min,\x20\x200\x20users,\x20\x20load\x20average:\x200\.01,\x200\
SF:.01,\x200\.00\n\nWhat\x20do\x20you\x20want\x20me\x20to\x20echo\x20back\
SF:?\x20\x03\n")%r(TLSSessionReq,64,"\x2017:29:04\x20up\x203\x20min,\x20\x                                         
SF:200\x20users,\x20\x20load\x20average:\x200\.01,\x200\.01,\x200\.00\n\nW                                         
SF:hat\x20do\x20you\x20want\x20me\x20to\x20echo\x20back\?\x20\x16\x03\n");                                         
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

```
gobuster dir -u http://10.10.10.147/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t40 -x .php,.txt
```

The webpage is the default Apache index.html, but in the source there is a hint.

![web-hint](/assets/img/safe/safe-web-hint.png)


Browsing to http://10.10.10.147/myapp gives us a binary to download.


```
 file myapp
myapp: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, BuildID[sha1]=fcbd5450d23673e92c8b716200762ca7d282c73a, not stripped
```



###########






<h3>Curl</h3>

```
curl -i http://10.10.10.147:1337
curl: (1) Received HTTP/0.9 when not allowed
```

`HTTP/0.9` ?


[http wiki](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol)


#########

