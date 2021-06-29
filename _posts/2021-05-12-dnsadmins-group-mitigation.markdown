---
layout: post
title: "How to mitigate DnsAdmins privilege escalation ?"
date: 2021-05-12
category: Pentesting
excerpt_separator: <!--more-->
lang: en
---  
Hello,  
  
Glad to see you in this second part of this [post](https://phackt.com/dnsadmins-group-exploitation-write-permissions). In our previous [article](https://phackt.com/dnsadmins-group-exploitation-write-permissions) we showed which rights were involved in the DnsAdmins privilege escalation. Now let's talk about how to properly mitigate this.  
<!--more--> 
  
First of all, a few words about the [Security Descriptor Propagator](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/05c8a4b6-43aa-49f7-8c31-df3ac72230f3) (SDProp) ;  
  
<p class="note">
Within Active Directory, a default set of highly privileged accounts and groups are considered <b>protected accounts and groups.</b><br><br>
  
With protected accounts and groups, the objects' permissions are set and enforced via an automatic process that ensures the permissions on the objects remains consistent even if the objects are moved the directory. Even if somebody manually changes a protected object's permissions, <b>this process ensures that permissions are returned to their defaults quickly.</b><br><br>

Additionally, permissions inheritance is disabled on protected groups and accounts, which means that even if the accounts and groups are moved to different locations in the directory, they do not inherit permissions from their new parent objects.<br><br>
Inheritance is disabled on the AdminSDHolder object so that permission changes to the parent objects do not change the permissions of AdminSDHolder.<br><br>  
</p>

This last sentence makes sense because the security descriptor used as a template to protect sensitive groups and users is stored in the ```nTSecurityDescriptor``` attribute from the ```AdminSDHolder``` object in Active Directory. The AdminSDHolder object is of class
container and has a DN of ```CN=AdminSDHolder,CN=System,<Domain NC DN>```.  
  
Even if this is not a post about detailing the SDProp mechanism, why is this important ? Because this security mechanism called [Security Descriptor Propagator](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/05c8a4b6-43aa-49f7-8c31-df3ac72230f3) (SDProp) aims at cutting any potential control paths to these **Protected Users and Groups** as they are highly privileged.  
  
Unfortunately, DnsAdmins is not targeted by this security mechanism <i class="fa fa-frown-o" aria-hidden="true"></i>.  
  
What is the algorithm to define which object deserves to be protected by the SDProp ? (*taken from the ADTS - 3.1.1.6.1.2 Protected Objects*) ;
```
d = domain;
o = protected security principal object

if  (
        (o!objectClass = group AND attribute o!groupType & GROUP_TYPE_SECURITY_ENABLED != 0) OR (o!objectClass = user)
    )
AND 
    (
        o!objectSid = d!objectSid + RID
    )
AND 
    (
        o is a member, directly or transitively, of any group in the set:
            built-in well-known group with RID = DOMAIN_ALIAS_RID_ADMINS
            built-in well-known group with RID = DOMAIN_ALIAS_RID_ACCOUNT_OPS
            built-in well-known group with RID = DOMAIN_ALIAS_RID_SYSTEM_OPS
            built-in well-known group with RID = DOMAIN_ALIAS_RID_PRINT_OPS
            built-in well-known group with RID = DOMAIN_ALIAS_RID_BACKUP_OPS
            built-in well-known group with RID = DOMAIN_ALIAS_RID_REPLICATOR
            account domain well-known group with RID = DOMAIN_GROUP_RID_ADMINS
            account domain well-known group with RID = DOMAIN_GROUP_RID_SCHEMA_ADMINS
            account domain well-known group with RID = DOMAIN_GROUP_RID_ENTERPRISE_ADMINS
    )
OR  
    (
        is one of the following well-known security principals:
            of class user with RID = DOMAIN_USER_RID_ADMIN
            of class user with RID = DOMAIN_USER_RID_KRBTGT
            of class group with RID = DOMAIN_GROUP_RID_CONTROLLERS
            of class group with RID = DOMAIN_GROUP_RID_READONLY_CONTROLLERS
    )
```

No mention to DnsAdmins here, even if we know that being member of DnsAdmins <i class="fa fa-long-arrow-right" aria-hidden="true"></i> compromised domain.  
  
At least we can manually set ACEs as restricted as the AdminSDHolder ones thanks to the cmdlet ```Set-ADSyncRestrictedPermissions``` from the ```AdSyncConfig.psm1``` module. This is part of the recommendations shared by this [Active Directory guide](https://www.cert.ssi.gouv.fr/uploads/guide-ad.html#dnsadmins).  
<p class="note">
The AzureADConnect.msi package can be downloaded from <a href="https://go.microsoft.com/fwlink/?LinkId=615771">https://go.microsoft.com/fwlink/?LinkId=615771</a>.<br><br>
The AdSyncConfig.psm1 module may be extracted from the Azure AD Connect installer with msiexec:<br>
<code>msiexec /a "AzureADConnect.msi" /qb TARGETDIR=c:\temp\</code>  
</p>  
  
# Mitigations

## DnsAdmins cleared
Keep the DnsAdmins group empty, prefer a delegation group to manage your DNS service and to allow DNS zone updates. This is done thanks to the two following steps ;  
  
- STEP 1: ALLOW ACCESS TO RPC USED BY DNS MANAGEMENT MMC SNAP-INS  

In the domain naming context, under the ```CN=System``` container, set the following access rights on the ```CN=MicrosoftDNS``` container for your trustee delegation group ;
<img class="dropshadowclass" src="{{ site.url }}/public/images/dnsadmins/step1.png" style="margin-top:1.5rem;margin-bottom:1.5rem;">


- STEP 2: ALLOW DNS ZONE MODIFICATIONS  
  
In the ```DC=DomainDnsZones``` naming context, on every zone you wish to delegate management, enable inheritance and set the following access rights on the ```CN=MicrosoftDNS``` container for your trustee delegation group ;
<img class="dropshadowclass" src="{{ site.url }}/public/images/dnsadmins/step2.png" style="margin-top:1.5rem;margin-bottom:1.5rem;">

## DnsAdmins security descriptor
This is an overview of the DnsAdmins security descriptor ;  
<img class="dropshadowclass" src="{{ site.url }}/public/images/dnsadmins/dnsadmins_sd_orig.png" style="margin-top:1.5rem;margin-bottom:1.5rem;">
  
### Owner
The group should be changed, DnsAdmins should be owned by Domain Admins ; 
<img class="dropshadowclass" src="{{ site.url }}/public/images/dnsadmins/owner_dnsadmins.png" style="margin-top:1.5rem;margin-bottom:1.5rem;">
  
### ACL
Set rights at least as strict as the AdminSDHolder ones ;
```powershell
Import-Module "C:\Program Files\Microsoft Azure Active Directory Connect\AdSyncConfig\AdSyncConfig.psm1"
$credential = Get-Credential
Set-ADSyncRestrictedPermissions -ADConnectorAccountDN "CN=DnsAdmins,CN=Users,DC=phackt,DC=local" -Credential $credential
```
  
You can check that the DnsAdmins security descriptor is blocking inheritable ACEs ;
```powershell
Get-ADSyncObjectsWithInheritanceDisabled -SearchBase "CN=DnsAdmins,CN=Users,DC=phackt,DC=local" -ObjectClass '*'

DistinguishedName : CN=DnsAdmins,CN=Users,DC=phackt,DC=local
Name              : DnsAdmins
ObjectClass       : group
ObjectGUID        : 54d747e8-f9c2-461b-b8f2-5cc0b99249cc
ObjectSID         : S-1-5-21-3816950244-2414788102-2833019223-1101
sAMAccountName    : DnsAdmins
``` 
  
You should get a security descriptor which should look like this ;  
<img class="dropshadowclass" src="{{ site.url }}/public/images/dnsadmins/dnsadmins_sd_after.png" style="margin-top:1.5rem;margin-bottom:1.5rem;">
  
<p class="note">
The <code>DACL_PROTECTED</code> parameter shows that the descriptor is blocking inheritable ACEs
</p>
  
As you see it is a way more restrictive than the original AdminSDHolder security descriptor shown below ;  
<img class="dropshadowclass" src="{{ site.url }}/public/images/dnsadmins/adminsdholder_sd.png" style="margin-top:1.5rem;margin-bottom:1.5rem;">
  
Also you can set, on the DnsAdmins group, a ```Deny``` ACE forbidding ```Everyone``` for the ```WRITE_OWNER``` right (remember that being owner of an object provides the ```WRITE_DAC``` and ```READ_CONTROL``` over the object).  

However if you do this remember that replaying the ```Set-ADSyncRestrictedPermissions``` cmdlet will reset the DACL and delete this ACE (on the other hand you could update the AdSyncConfig.psm1 module to match your needs).  
  
## MicrosoftDNS security descriptor
As seen previously, deleting the DnsAdmins ```RIGHT_DS_WRITE_PROPERTY_EXTENDED``` right for the MicrosoftDNS container will prevent an attacker from writing the registry key to set the path of an arbitrary DLL to be loaded by the DNS service.  
<img class="dropshadowclass" src="{{ site.url }}/public/images/dnsadmins/withoutextwrite.png" style="margin-top:1.5rem;margin-bottom:1.5rem;">
  
However i don't know what is the impact if you benefits from the Microsoft support and if they need to exceptionally perform a maintenance operation, from DnsAdmins, which needs this right.  
  
# Conclusion
Firstly i thought it was possible to add a new security group to be targeted by the SDProp mechanism but no, it's not (and as i was said, it's not for tomorrow). At least we can manually set some restricted rights on it. Also i'm having a look at creating a routine which will frequently override the DnsAdmins ```ntSecurityDescriptor``` based on the AdminSDHolder one to simulate the SDProp mechanism for custom groups.  
  
DnsAdmins is one of the so many ways an attacker can gain control of your Active Directory. If you are looking for a complete Active Directory security assessment checklist : [https://www.cert.ssi.gouv.fr/uploads/guide-ad.html](https://www.cert.ssi.gouv.fr/uploads/guide-ad.html).  
    
Thanks folks for reading,  
Cheers.
