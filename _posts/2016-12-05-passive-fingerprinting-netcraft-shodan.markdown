---
layout: post
title:  "Passive Gathering Information - Netcraft and Shodan"
date:   2016-12-05
categories: fingerprint
excerpt_separator: <!--more-->
---
Bonjour à tous, 
  
Aujourd'hui nous parlerons de la prise d'empreinte passive et des plateformes [netcraft](https://www.netcraft.com/) et [shodan](https://www.shodan.io/). La prise d'empreinte passive consiste en l'agrégation d'informations publiques concernant une cible (sans requêtage direct de ses serveurs).  
<!--more-->
  
La prise d'empreinte au sens large devra être la plus exhaustive possible pour maximiser la probabilité de réussite d'une attaque:  
 
 - prise d'informations sur l'entreprise et son activité (énumération des employés et adresses mail ([theharvester](https://code.google.com/archive/p/theharvester/))
 - base de données Whois
 - Google dorks
 - SMTP, SMB, SNMP, DNS enumeration (recon-ng, dnsrecon, nbtscan, enum4linux, snmpwalk)
 - Scan des services et des bannières
 - Scan des vulnérabilités (metasploit, nikto, OpenVAS, NSE vuln)
  
Il est possible de collecter beaucoup d'informations via les moteurs de recherche. Pour cela je vous redirige vers l'excellent site d'Offensive Security [https://www.exploit-db.com/google-hacking-database/](https://www.exploit-db.com/google-hacking-database/) et la Google Cheat Sheet du SANS [https://www.sans.org/security-resources/GoogleCheatSheet.pdf](https://www.sans.org/security-resources/GoogleCheatSheet.pdf).  
  
Quelques exemples de recherches:  
  
```site:microsoft.com -site:www.microsoft.com``` (tous les sous domaines de microsoft)  
```site:ameli.fr inurl:phpinfo.php``` (version de php)  
```site:ameli.fr inurl:(cgi|api|webservice|private|portail) | (login OR pass OR admin)``` (potentielles pages de login)  
  
D'autres informations sur la prise d'empreinte passive:  
  
**Base de données Whois:**  
  
Selon Wikipedia: *Each registrar must maintain a Whois database containing all contact information for the domains they host. These databases are usually published by a Whois server over TCP port 43. The whois client can also perform reverse lookups. Rather than inputting a domain name, you can provide an IP address.*  
  
```whois microsoft.com```  
  
**recon-ng:**  
  
Outil complet de prise d'empreinte: [https://bitbucket.org/LaNMaSteR53/recon-ng/wiki/Home](https://bitbucket.org/LaNMaSteR53/recon-ng/wiki/Home)
  
```
$ recon-ng
[recon-ng][default] > use recon/domains-contacts/whois_pocs
[recon-ng][default][whois_pocs] > show options

  Name    Current Value  Required  Description
  ------  -------------  --------  -----------
  SOURCE  cisco.com      yes       source of input (see 'show info' for details)

[recon-ng][default][whois_pocs] > set SOURCE cisco.com
SOURCE => cisco.com
[recon-ng][default][whois_pocs] > run

---------
CISCO.COM
---------
[*] URL: http://whois.arin.net/rest/pocs;domain=cisco.com
[*] URL: http://whois.arin.net/rest/poc/GAB42-ARIN
[*] [contact] Gary Abbott (gabbott@cisco.com) - Whois contact
[*] URL: http://whois.arin.net/rest/poc/SMA-ARIN
[*] [contact] Steve Acheson (satch@cisco.com) - Whois contact
[*] URL: http://whois.arin.net/rest/poc/ACKER5-ARIN
[*] [contact] Barry Ackerman (backerma@cisco.com) - Whois contact
...
```  
  
recon-ng modules:  
```
show modules (pour lister tous les modules)
recon/domains-hosts/google_site_web (récupération des sous-domaines)
recon/domains-vulnerabilities/xssed (cherche dans la database http://xssed.com, sites vulnérables au XSS)
...
```
  
**theharvester**  
  
*The objective of this program is to gather emails, subdomains, hosts, employee names, open ports and banners from different public sources like search engines, PGP key servers and SHODAN computer database* - [http://www.edge-security.com/theharvester.php](http://www.edge-security.com/theharvester.php).  
  
```
*******************************************************************
*                                                                 *
* | |_| |__   ___    /\  /\__ _ _ ____   _____  ___| |_ ___ _ __  *
* | __| '_ \ / _ \  / /_/ / _` | '__\ \ / / _ \/ __| __/ _ \ '__| *
* | |_| | | |  __/ / __  / (_| | |   \ V /  __/\__ \ ||  __/ |    *
*  \__|_| |_|\___| \/ /_/ \__,_|_|    \_/ \___||___/\__\___|_|    *
*                                                                 *
* TheHarvester Ver. 2.7                                           *
* Coded by Christian Martorella                                   *
* Edge-Security Research                                          *
* cmartorella@edge-security.com                                   *
*******************************************************************


Usage: theharvester options 

       -d: Domain to search or company name
       -b: data source: google, googleCSE, bing, bingapi, pgp, linkedin,
                        google-profiles, jigsaw, twitter, googleplus, all

       -s: Start in result number X (default: 0)
       -v: Verify host name via dns resolution and search for virtual hosts
       -f: Save the results into an HTML and XML file (both)
       -n: Perform a DNS reverse query on all ranges discovered
       -c: Perform a DNS brute force for the domain name
       -t: Perform a DNS TLD expansion discovery
       -e: Use this DNS server
       -l: Limit the number of results to work with(bing goes from 50 to 50 results,
            google 100 to 100, and pgp doesn't use this option)
       -h: use SHODAN database to query discovered hosts

Examples:
        theharvester -d microsoft.com -l 500 -b google -h myresults.html
        theharvester -d microsoft.com -b pgp
        theharvester -d microsoft -l 200 -b linkedin
        theharvester -d apple.com -b googleCSE -l 500 -s 300
```  
  
Exemples de commandes:  
  
```
theharvester -d mycompany.com -l 500 -b google -t -h -f results_google.html
theharvester -d mycompany.com -l 500 -b linkedin > results_linkedin.txt
```
  
L'option -h utilise la base de données Shodan.io [https://www.shodan.io/](https://www.shodan.io/).  
  
**www.shodan.io**  
  
*Shodan is a search engine that lets the user find specific types of computers (web cams, routers, servers, etc.) connected to the internet using a variety of filters. Some have also described it as a search engine of service banners, which are meta-data the server sends back to the client.[1] This can be information about the server software, what options the service supports, a welcome message or anything else that the client can find out before interacting with the server.*  
  
Shodan est un site de Data Mining dont les informations proviennent du scan d'adresses ip publiques. Leur service agrége toutes les informations sur les services exposés et leurs bannières. Un exemple, Shodan.io permet de détecter de nombreux [IoT](https://www.owasp.org/index.php/OWASP_Internet_of_Things_Project), webcam, Industrial Control Systems, et j'en passe.  
  
Voici par exemple une capture d'écran d'une recherche sur les Caméras connectées ayant une ip géolocalisée sur Paris:
  
![shodan]({{ site.url }}/public/images/passive-fingerprinting/shodan.png)  
  
**www.netcraft.com**  
  
*Netcraft provides web server and web hosting market-share analysis, including web server and operating system detection. Netcraft also provides security testing, and publishes news releases about the state of various networks that make up the Internet.*  
  
Si vous souhaitez rechercher par nom de domaine: [https://searchdns.netcraft.com/](https://searchdns.netcraft.com/).  
Cliquez ensuite sur *Site Report*:  
  
![netcraft]({{ site.url }}/public/images/passive-fingerprinting/netcraft.png)  
  
Ces bases de connaissance sont utiles pour qu'une entreprise puisse prendre connaissance des informations à disposition d'un assaillant. Exemple un simple mail pro utilisé dans un forum peut être utilisé pour du phising ciblé.  
  
**Passive DNS**  
  
Les bases de passive DNS permettront d'obtenir passivement différent types d'enregistrements DNS ([Passive Mnemonic](https://passivedns.mnemonic.no/search) est gratuit). D'autres solutions payantes existent comme DNSDB ou RisqIQ.  
  
A bientôt.
