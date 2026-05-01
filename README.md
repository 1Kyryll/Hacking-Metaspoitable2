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

