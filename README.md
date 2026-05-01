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
```bash
nmap -sn <target-ip>/24
```

```-sn``` means "ping scan, no port scan" — it just finds live hosts. You'll see Kali, the host, and Metasploitable.

Now port scan the target: 

```bash
nmap -sV -sC -p- <target-ip> -oN nmap_full.txt
```
Breaking that down: ```-sV``` does service/version detection (not just "port 21 is open" but "port 21 is running vsftpd 2.3.4"). ```-sC``` runs default safe scripts that probe for common issues. ```-p-``` scans all 65535 ports instead of just the top 1000. ```-oN``` saves output to a file. This will take a few minutes.

You'll see something like 20+ open ports — FTP, SSH, Telnet, SMTP, HTTP, RPC, Samba, MySQL, PostgreSQL, VNC, IRC, and more. **This is your attack surface.** Each open port is a door, each service version is a clue, each unusual service is potentially something interesting.
