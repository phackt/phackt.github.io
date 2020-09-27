---
layout: post
title:  "SecurityTube Advanced Red Team Lab training - Worth it ?"
date:   2020-09-22
category: Certification
excerpt_separator: <!--more-->
---
Quick answer: **Totally** !

Hello everybody,  

I would like to talk a bit about the [SecurityTube red team labs](https://www.pentesteracademy.com/redlabs), specifically the [Advanced Red Team Lab](https://www.pentesteracademy.com/redteamlab) which leads to the **CRTE** (Certified Red Team Expert) certification.  

Some great [reviews](https://www.google.com/search?q=review+red+team+lab+pentester+academy+%22CRTE%22) are already existing, so i will focus on why i chose this lab and certification. I will give you some hints about how to approach your targets. Most importantly, i would like to introduce you a tool that i developped which will help you during your journey, [Invoke-Recon](https://github.com/phackt/Invoke-Recon).  
<!--more-->


## Context

I was looking for a another **professional and challenging cert** focused on **Windows AD**. My prerequisites were:  
 - no jeopardy style
 - real life scenarios which will be found in real engagements
 - a environment not too crowded. I met this situation with others platforms and i spent much of my time contacting the support to revert the VMs because of others bad exploitation or persistence, getting the machines you're working on totally unstable.  

Finally the *Advanced Red Team Lab* was the answer. Companies are more and more looking for online qualitative trainings because of the actual sanitary crisis. Think to ask to your company for a financial support.  


## What i really liked  

 - Reactive and helpful support
 - Updated servers and workstations with AppLocker, AV running and Powershell in Constrained Language Mode.  
 - Relevant AD exploitation paths and local privilege escalation
 - Nikhil Mittal and his team are really friendly and you definitely can contact them on twitter to share your thoughts on this lab

And why not, while you're diving into this journey, you may probably see how the tools you are using may be improved to match your needs (don't hesitate to PR - mine with [PowerUpSQL](https://github.com/NetSPI/PowerUpSQL/pull/62)) or may be you will develop your own tool. What i mean is that this is a virtuous circle, you will learn and you will share with the infosec community.  

## What should be / will be improved

 - The possibility to revert VMs on our own during lab and cert (in dev on their side)


## What you will learn

Obviously, the difficulty level depends on your experience as a pentester, but be sure that after this cert you will be able to :  
 - Properly enumerate an AD (basic / more complex stuff) and which tools to use (we will about it later)
 - Find AD exploitation paths, weak ACLs
 - Spearphish users  
 - Exploit Kerberos delegation and to know how the S4U extensions are working (for ex: kerberos constrained delegation with protocol transition)
 - Exploit domain / forest trusts (SID Filtering, SID History, ...)
 - Exploit MSSQL servers (this part was really interesting as i did not have so many opportunities to exploit MSSQL instances during engagements)
 - Find your way to perform local privilege escalation on fully patched servers with AppLocker and AV running
 - Many other cool stuff


## Hints (that you may not find in others blog posts)

Your will start as domain user on a Windows Server where you will be able to RDP. I advice you to **get SYSTEM** on this machine, to **disable Defender and some restrictive Firewall and AppLocker rules**. Then take off your shoes and feel comfortable at home.  

I mainly used this Windows VM but i also had my Kali. In order to be able to proxyfy my Kali tools, i used [ssf](https://github.com/securesocketfunneling/ssf/releases) and [proxychains](https://translate.google.fr/translate?hl=fr&sl=fr&tl=en&u=https%3A%2F%2Fphackt.com%2Ftor-proxychains).  

Sometimes in this lab you will be able to move forward the unintended ways, i mean with your own kind of exploitation (to give you just an idea, read this blog post from [itm4n](https://itm4n.github.io/printspoofer-abusing-impersonate-privileges/)). Even if you succeed in compromising all the forests, i would strongly recommend you to try to get the all the boxes the intended way, because you may miss something for the D-day.  

Also do not overlook the **post exploitation** part. And don't forget to note where you struggled at, during the exam it will be a pity to lose time on a thing you already faced.  

Also not so many services will be exposed, and you also will be firewalled.  

Talking about firewalling, we agree that sometimes it's more comfortable to have a **RDP** session rather than any command line session. In the following example we will assume that we have a shell on a server and we have just a few ports which are accessible. **5985** is a good candidate but **WinRM** is already running on the server.  

*In the following example, be aware that if you execute the following commands from a Remote Powershell session, you will be disconnected because we set the RDP listen port to 5985, so we will have to ```sc.exe stop WinRM``` before running Remote Desktop Service.*  

**Set RDP listen port and start the Remote Desktop service 'without rebooting'**:  
```powershell
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -name "UserAuthentication" -Value 1
Set-ItemProperty -Path "HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp\" -Name PortNumber -Value 5985
sc.exe stop WinRM
```

Then run:  
```
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections /t REG_DWORD /d 0 /f
```

## Tools

Now let's talk a bit about an interesting part, what kind of tools did i have on my Windows VM (did not mean i used all of them) ? Here is my list :

| Tool          | Description  |
| ------------- | :------------|
|[ADModule](https://github.com/samratashok/ADModule)|Use AD powershell module for enumeration without installing RSAT|
|[amsi-bypass.ps1](https://github.com/phackt/pentest/blob/master/bypass/amsi-bypass.ps1)|```[Bypass.A_M_S_I]::Disable()```|
|[nmap](https://nmap.org/download.html)|Network Mapper|
|[agentransack](https://www.mythicsoft.com/agentransack/)|Fast files / directories crawler|
|[mimikatz](https://github.com/gentilkiwi/mimikatz/releases)|Do we really have to introduce this tool  ?|
|[nc64.exe](https://github.com/phackt/pentest/blob/master/privesc/windows/nc64.exe)|Netcat|
|[Nishang](https://github.com/samratashok/nishang)|Exploitation framework from Nikhil "SamratAshok" Mittal|
|[Empire -> Invoke-DCSync.ps1](https://github.com/EmpireProject/Empire/blob/master/data/module_source/credentials/Invoke-DCSync.ps1)|DCSync|
|[PowerSploit -> PowerView.ps1](https://github.com/PowerShellMafia/PowerSploit/blob/dev/Recon/PowerView.ps1)|AD Enumeration - [Harmjoy Powerview 3 tricks](https://gist.github.com/HarmJ0y/184f9822b195c52dd50c379ed3117993)|
|[PowerSploit -> Invoke-Mimikatz.ps1](https://github.com/PowerShellMafia/PowerSploit/blob/master/Exfiltration/Invoke-Mimikatz.ps1)|Mimikatz thanks to Powershell|
|[PowerSploit -> Invoke-PortScan.ps1](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/Invoke-Portscan.ps1)|Ports scanner in Powershell|
|[PowerSploit -> Invoke-TokenManipulation.ps1](https://github.com/PowerShellMafia/PowerSploit/blob/master/Exfiltration/Invoke-TokenManipulation.ps1)|Manipulate logon token and create impersonation token|
|[Invoke-SDPropagator.ps1](https://gallery.technet.microsoft.com/invoke-sdpropagator-to-c99ae41c)|Invoke SDPropagator mechanism - for example if you backdoored the AdminSDHolder|
|[Invoke-SocksProxy.psm1](https://github.com/p3nt4/Invoke-SocksProxy/blob/master/Invoke-SocksProxy.psm1)|Proxy socks in Powershell|
|[IPv4PortScan.ps1](https://github.com/BornToBeRoot/PowerShell_IPv4PortScanner/blob/master/Scripts/IPv4PortScan.ps1)|Another ports scanner in Powershell|
|[powercat.ps1](https://github.com/besimorhino/powercat/blob/master/powercat.ps1)|Netcat in Powershell|
|[Set-LHSTokenPrivilege.ps1](https://gallery.technet.microsoft.com/Adjusting-Token-Privileges-9b6724fc)|Enable / disable your local privileges|
|[PowerUpSQL](https://github.com/NetSPI/PowerUpSQL)|MSSQL exploitation framework|
|[PrintSpoofer.exe](https://github.com/itm4n/PrintSpoofer)|Exploit the printer bug locally|
|[PrivescCheck](https://github.com/itm4n/PrivescCheck)|Find privesc|
|[Rubeus](https://github.com/GhostPack/Rubeus)|Useful when you will have to deal with kerberos tickets|
|[BloodHound](https://github.com/BloodHoundAD/BloodHound)|Find AD exploitation paths|
|[SpoolSample](https://github.com/leechristensen/SpoolSample)|Connect to the RPC RpcRemoteFindFirstPrinterChangeNotification to trigger the printer bug|
|[ssf](https://github.com/securesocketfunneling/ssf)|Tunneling tool|
|[SysinternalsSuite](https://docs.microsoft.com/en-us/sysinternals/downloads/sysinternals-suite)|All MS utilities|
|[lolbas](https://lolbas-project.github.io/)|Living Off The Land Binaries and Scripts|

And last but not least, as i said previously this journey was a good opportunity to gather a lot of my recon commands in one single Powershell tool: [Invoke-Recon](https://github.com/phackt/Invoke-Recon).  
Believe me, if you are doing this lab and want to pass this cert, [Invoke-Recon](https://github.com/phackt/Invoke-Recon) + [BloodHound](https://github.com/BloodHoundAD/BloodHound) will be the perfect journey companions. *Invoke-Recon* has been developed based on the *Advanced Red Team Lab*.  

### Invoke-recon

Give a try to [Invoke-Recon](https://github.com/phackt/Invoke-Recon) and let me know, i'm still developping this tool, and i will introduce it later in a next blog post.
But if you are already asking yourself what kind of stuff you can enumerate with it:  

#### Domain Enumeration  

 - Find all DCs (check if ADWS are accessible in order to be able to use the Active Directory powershell module)
 - Password domain policy
 - Domains / forests trusts
 - All domain users / groups / computers
 - Privileged users with RID >= 1000 (recursive lookups for nested members of privileged groups, not AdminCount = 1 to avoid orphans)
 - Users / computers / Managed Service Accounts with unconstrained (T4D) and constrained delegation (also look for constrained delegation with protocol transition (T2A4D))
 - Services with msDS-AllowedToActOnBehalfOfOtherIdentity
 - Exchange servers
 - Users with mailboxes

#### Quick Wins  

- Exchange vulnerable to PrivExchange and CVE-2020-0688  
- Computers with deprecated OS
- Users with Kerberos PreAuth disables (AS_REP Roasting)
- Kerberoastable users
- Principals (RID >= 1000) with Replicating Directory Changes / Replicating Directory Changes All

#### MSSQL Enumeration  

- Enumerates MSSQL instances (looking for SPN service class MSSQL)
- Find MSSQL instances accessible within current security context and get their versions
- Find linked servers from each accessible MSSQL instances
- Bruteforce common credentials
- Look for xp_cmdshell enabled through linked servers of each accessible instances
- Audit each accessible MSSQL Instances for common high impact vulnerabilities and weak configurations


## More to come

To keep on talking about AD enumeration, i would like to share with you some of my **Cypher** queries i use with **BloodHound** to spot some very interesting stuff during an engagement. I will also dive a bit more into [Invoke-Recon](https://github.com/phackt/Invoke-Recon).  

Keep in touch, and don't hesitate if you wanna more information about this lab, if you are stuck or anything else, i'm always glad to help. I answer back to a lot of questions by email or twitter, some dealing with my switch from web developer to technical auditor for the french gouvernment. [Contact me](https://phackt.com/about/).  

*P.S: if you wanna buy me a coffee to stay awoke while i will be writing my next blogs, thank you so much.*  

Wish you a nice day, may the force be with you (i let you choose which side).  

Phackt.
