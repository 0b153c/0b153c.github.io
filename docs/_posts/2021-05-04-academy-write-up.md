---
layout: post
title:  "HTB CTF Write-Up: Academy"
date:   2021-05-04 20:00:00 +0000
author: 0b153c
categories: HTB CTFs
---

## Terms

The following two terms will be used throughout the write-up:

* **Attacker machine**: local burner laptop, loaded with Kali Linux 2020.4, using an IP of 10.10.15.166

* **Target machine**: remote server, Academy (hosted by HTB), using an IP of 10.10.10.215

## 1 - Reconnaissance

### 1.1 – Network scanning

#### 1.1.1

Nmap (short for “Network Mapper”) is used to scan the target IP, identify all open service ports, create a comma separated list of these service ports, and print out the list to confirm scan results.

* On the attacker machine (within a terminal emulator), create a variable called “ports” and define it with the results of the nmap scan:

``` bash
ports=$(nmap -p- --min-rate=1000 -T4 10.10.10.215 | grep ^[0-9] | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)
```

* Once the scan completes, print the contents of the ports variable:

``` bash
echo $ports
```

> ![Screenshot_1.1.1](https://i.ibb.co/Ydw0Dgj/1-1-1.png)

Now that the open ports on the target machine have been discovered, additional network scanning may identify more details regarding what is running on the service ports.

#### 1.1.2

A second Nmap scan of the target IP is executed to uncover more information regarding the open ports from section 1.1.1.

* Pass the “ports” variable into a follow-up nmap scan:

``` bash
nmap -sC -sV -p$ports 10.10.10.215
```

> ![Screenshot_1.1.2](https://i.ibb.co/fdx6WcR/1-1-2.png)

Knowing that the target machine is running OpenSSH Server, Apache Web Server, and—potentially—a MySQL Database Server helps with identifying what to focus on next. At this point, the “easiest” place to start would be the Apache Web Server (80/tcp/http).

### 1.2 – Setting up a local proxy and browsing to the web server

### 1.3 – Local network routing modification / bypassing DNS

### 1.4 – Testing and analyzing site registration

### 1.5 – Discovering site directories and files

## 2 – Intrusion

### 2.1 – Manipulating site registration

### 2.2 – Post intrusion reconnaissance

## 3 – Exploitation

### 3.1 – Executing de-serialization exploit for reverse-shell

### 3.2 – Upgrading the reverse-shell

### 3.3 – Post exploitation reconnaissance

## 4 – Lateral Movement (1 of 2)

### 4.1 – Breaking authentication

### 4.2 – Post lateral movement reconnaissance

## 5 – Lateral Movement (2 of 2)

### 5.1 – Breaking authentication

### 5.2 - Post lateral movement reconnaissance

## 6 – Privilege Escalation

### 6.1 – Exploiting privilege escalation vulnerability

### 6.2 – Post privilege escalation exploit reconnaissance
