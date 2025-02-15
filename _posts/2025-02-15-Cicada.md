---
title: HackTheBox - Cicada
date: 2025-02-15 16:00:00 +/-0000
categories: [HackTheBox, Windows]
tags: [hackthebox, easy, smb, windows, rpc, activedirectory]     # TAG names should always be lowercase
description: In Cicada we attack a Domain Controller starting with abusing SMB null authentication.
image: "/assets/img/posts/cicada/cicada.png"
---

## The attack path

Cicada from HackTheBox involves attacking a domain controller with various misconfigurations which result in us gaining winRM access as a low privilege user. 

We start out by abusing SMB null authentication to gain the default password for new users in the domain.

We are then able to gather local user information using null auth on rpcclient.

With a potential user we are able to carry out a `rid bruteforce` using `nxc` which gives us the full list of domain users.

We then carry out a password spray using the default password against the user list.

This provides us access to a new user that has access to new shares on the SMB service where we find a `PSCredential` giving us a new username and cleartext password.

After accessing the DC using `evilwinRM` it turns out that this user has the powerful `SeBackupPrivilege` due to being part of the `Backup Operators` group which we can use to quickly grab the root flag. 

In beyond root I demonstrate the risk of excessive permissions by dumping the local registry hives giving us access to the `Domain Administrator` account using a `pass the hash attack`.

## User

The first step is to use `nmap` to carry out a scan of the IP address provided.

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Cicada]
└─$ nmap -sV -sC -p- 10.129.62.235 -Pn                                                                                                 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-02-15 10:07 UTC
Nmap scan report for 10.129.62.235
Host is up (0.031s latency).
Not shown: 65522 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-02-15 17:10:15Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: cicada.htb0., Site: Default-First-Site-Name)
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: cicada.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
|_ssl-date: TLS randomness does not represent time
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: cicada.htb0., Site: Default-First-Site-Name)
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: cicada.htb0., Site: Default-First-Site-Name)
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
61921/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: CICADA-DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2025-02-15T17:11:06
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
|_clock-skew: 7h00m01s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 257.06 seconds

                                                              
```


After the initial nmap scan we find the DC is running a `SMB` service which allows `Null Authentication`. 

First we listed the SMB shares with `smbclient -N -L //IP/` which revealed the interesting HR share.

![image](/assets/img/posts/cicada/1.png)

Exploiting this we manage to find a file containing the `default password` for new users in the domain.

![image](/assets/img/posts/cicada/2.png)

We find that we are able to log an anonymously using rpcclient, which we can use to query LSA for potential users.

![image](/assets/img/posts/cicada/3.png)

We are able to use the `Dev Support` account with no password to perform a `rid-bruteforce` attack to enumerate potential domain users.

![image](/assets/img/posts/cicada/4.png)

We can use these users to make a userlist and attempt a `password spray` against the list and we get a hit.

![image](/assets/img/posts/cicada/5.png)

With valid credentials we are able to access RPC using `rpcclient` and enumerate domain users and we find a user with a password saved in their description field.

![image](/assets/img/posts/cicada/6.png)

The new user has higher access permissions on the `SMB shares` and inside one of them we find a Powershell backup script containing a PSCredential string for another user.

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Cicada]
└─$ nxc smb 10.129.62.235 -u david.orelious -p 'aRt$Lp#7t*VQ!3' -M spider_plus
SMB         10.129.62.235   445    CICADA-DC        [*] Windows Server 2022 Build 20348 x64 (name:CICADA-DC) (domain:cicada.htb) (signing:True) (SMBv1:False)
SMB         10.129.62.235   445    CICADA-DC        [+] cicada.htb\david.orelious:aRt$Lp#7t*VQ!3 
SPIDER_PLUS 10.129.62.235   445    CICADA-DC        [*] Started module spidering_plus with the following options:
SPIDER_PLUS 10.129.62.235   445    CICADA-DC        [*]  DOWNLOAD_FLAG: False
SPIDER_PLUS 10.129.62.235   445    CICADA-DC        [*]     STATS_FLAG: True
SPIDER_PLUS 10.129.62.235   445    CICADA-DC        [*] EXCLUDE_FILTER: ['print$', 'ipc$']
SPIDER_PLUS 10.129.62.235   445    CICADA-DC        [*]   EXCLUDE_EXTS: ['ico', 'lnk']
SPIDER_PLUS 10.129.62.235   445    CICADA-DC        [*]  MAX_FILE_SIZE: 50 KB
SPIDER_PLUS 10.129.62.235   445    CICADA-DC        [*]  OUTPUT_FOLDER: /tmp/nxc_hosted/nxc_spider_plus
SMB         10.129.62.235   445    CICADA-DC        [*] Enumerated shares
SMB         10.129.62.235   445    CICADA-DC        Share           Permissions     Remark
SMB         10.129.62.235   445    CICADA-DC        -----           -----------     ------
SMB         10.129.62.235   445    CICADA-DC        ADMIN$                          Remote Admin
SMB         10.129.62.235   445    CICADA-DC        C$                              Default share
SMB         10.129.62.235   445    CICADA-DC        DEV             READ            
SMB         10.129.62.235   445    CICADA-DC        HR              READ            
SMB         10.129.62.235   445    CICADA-DC        IPC$            READ            Remote IPC
SMB         10.129.62.235   445    CICADA-DC        NETLOGON        READ            Logon server share 
SMB         10.129.62.235   445    CICADA-DC        SYSVOL          READ            Logon server share 
SPIDER_PLUS 10.129.62.235   445    CICADA-DC        [+] Saved share-file metadata to "/tmp/nxc_hosted/nxc_spider_plus/10.129.62.235.json".
SPIDER_PLUS 10.129.62.235   445    CICADA-DC        [*] SMB Shares:           7 (ADMIN$, C$, DEV, HR, IPC$, NETLOGON, SYSVOL)
SPIDER_PLUS 10.129.62.235   445    CICADA-DC        [*] SMB Readable Shares:  5 (DEV, HR, IPC$, NETLOGON, SYSVOL)
SPIDER_PLUS 10.129.62.235   445    CICADA-DC        [*] SMB Filtered Shares:  1
SPIDER_PLUS 10.129.62.235   445    CICADA-DC        [*] Total folders found:  33
SPIDER_PLUS 10.129.62.235   445    CICADA-DC        [*] Total files found:    12
SPIDER_PLUS 10.129.62.235   445    CICADA-DC        [*] File size average:    1.09 KB
SPIDER_PLUS 10.129.62.235   445    CICADA-DC        [*] File size min:        23 B
SPIDER_PLUS 10.129.62.235   445    CICADA-DC        [*] File size max:        5.22 KB

```

Looking at the json file we see something interesting...

```bash
{
    "DEV": {
        "Backup_script.ps1": {
            "atime_epoch": "2024-08-28 18:28:22",
            "ctime_epoch": "2024-03-14 12:31:38",
            "mtime_epoch": "2024-08-28 18:28:22",
            "size": "601 B"
```


```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Cicada]
└─$ smbclient //10.129.62.235/DEV -U david.orelious              
Password for [WORKGROUP\david.orelious]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Thu Mar 14 12:31:39 2024
  ..                                  D        0  Thu Mar 14 12:21:29 2024
  Backup_script.ps1                   A      601  Wed Aug 28 18:28:22 2024

                4168447 blocks of size 4096. 326165 blocks available
smb: \> get Backup_script.ps1 
getting file \Backup_script.ps1 of size 601 as Backup_script.ps1 (5.3 KiloBytes/sec) (average 5.3 KiloBytes/sec)
smb: \> exit

```

Looking at the contents of the backup script we find a Powershell Credential.

![image](/assets/img/posts/cicada/7.png)

We can log in with this user and get the user flag.

![image](/assets/img/posts/cicada/8.png)

## Root

We check the users permissions and find out that we have `SeBackupPrivilege and SeRestorePrivilege` due to being a member of the `Backup Operators` group which allows us full access to the file system.

![image](/assets/img/posts/cicada/9.png)

Using [SebackupPrivilege](https://github.com/giuliano108/SeBackupPrivilege) we can copy the root flag from the Administrator desktop into a folder that we control and read the file contents.

![image](/assets/img/posts/cicada/10.png)

## Beyond Root

Since we have access to the full system we can take copies of the `SAM and SYSTEM` registry hives.

![image](/assets/img/posts/cicada/11.png)

Using `impacket-secretsdump` we can dump dump the NT hash of every local user on the DC. We can then use the domain Administrator hash in a `Pass the hash` attack to gain full control over the Domain.

![image](/assets/img/posts/cicada/12.png)

## Thoughts

I found this to be quite an interesting box as I tried doing it just before I finished the pentester path on HTB and to land on a DC with no obvious way in was a bit strange at first but I manged to get through it and was very happy about it as some of the attacks like using rpcclient are slightly out of scope of the CPTS exam. Its interesting to look back at my original write up for this and see how much my skills and knowledge have improved.

Thanks for reading! 
