---
layout: post
title:  "Dyn DNS DDOS"
date:   2016-10-23
categories: general
visible: 1
---
<br />
Bonjour à tous,
  
Peut être avez-vous tenté vendredi dernier (21 octobre 2016) d'accéder à certains de vos sites ou services préférés comme Netflix, Twitter, Spotify, Paypal, Ebay, Playstation Network,... ces derniers ayant été rendus inaccessibles. 

**Mais comment autant de services peuvent-ils tomber simultanément?**

Une attaque par déni de service distribué a eu lieu ce vendredi paralysant le service DNS de la société Dyn qui gère les services DNS de colosses du Web. Dyn ne gère les DNS que de 4.7% des sites selon [Datanyze](https://www.datanyze.com/market-share/dns/), mais ces derniers feraient partie des plus importants (Paypal, Twitter, Github, Etsy, Soundcloud, Spotify, Heroku, Shopify). Les Etats-Unis et l'Europe occidentale ont été particulièrement touchés. Dyn a communiqué dans la nuit de Vendredi à Samedi que le problème était résolu, l'attaque par déni de service ayant très probablement été orchestrée à partir d'objets connectés **piratés**.  
  
 > Le [déni de service](https://fr.wikipedia.org/wiki/Attaque_par_d%C3%A9ni_de_service) consiste à envoyer un très gros volume de paquets formatés de telle façon que le service cible saturera et deviendra indisponible.  
  
L'attaque de Dyn, complexe et sophistiquée,  met en évidence l'enjeu de la **sécurisation des objets connectés**. Une attaque similaire avait touché récemment l'hébergeur français OVH ([http://www.zdnet.fr/actualites/ovh-noye-par-une-attaque-ddos-sans-precedent-39842490.htm](http://www.zdnet.fr/actualites/ovh-noye-par-une-attaque-ddos-sans-precedent-39842490.htm)).  
  
On peut également se poser la question de la centralisation du Web. En effet ici le manque de redondance est critique. Cependant certains DNS n'ayant pas autorité ont pu garder en cache l'ip des sites touchés, permettant de continuer à accéder aux sites. Pour rappel, Le [Domain Name System](https://fr.wikipedia.org/wiki/Domain_Name_System) (ou DNS, système de noms de domaine) est un service permettant de traduire un nom de domaine en informations de plusieurs types qui y sont associées, notamment en adresses IP de la machine portant ce nom. Ainsi lorsque vous tapez www.twitter.com, si la résolution nom de domaine <=> adresse IP échoue, votre ordinateur ne connaitra pas la destination de vos paquets.  
   
**Qui a orchestré cette attaque est dans quel but?**  
  
Certains avancent l'hypothèse qu'un état testerait la cyberdéfense de compagnies qui jouent des rôles clés sur le Web. Un article très intéressant sur le sujet est le suivant: [https://www.schneier.com/blog/archives/2016/09/someone_is_lear.html](https://www.schneier.com/blog/archives/2016/09/someone_is_lear.html).  
  
**Comment pouvons-nous identifier que le DNS est tombé?**  
  
Les commandes DNS nslookup ou dig. Un exemple avec dig pour nous permettre de connaitre l'ip associéé à un nom de domaine:

```bash
dig twitter.com +noall +answer

; <<>> DiG 9.8.3-P1 <<>> twitter.com +noall +answer
;; global options: +cmd
twitter.com.	 1005	 IN	A	104.244.42.1
twitter.com.	 1005	 IN	A	104.244.42.65
```  
  
Regardons qui sont les serveurs ayant autorité, donc à contacter pour résoudre le domaine twitter.com:  
  
```bash
dig twitter.com NS +noall +answer

; <<>> DiG 9.8.3-P1 <<>> twitter.com NS +noall +answer
;; global options: +cmd
twitter.com.	5469	IN	NS	ns2.p34.dynect.net.
twitter.com.	5469	IN	NS	ns4.p34.dynect.net.
twitter.com.	5469	IN	NS	ns1.p34.dynect.net.
twitter.com.	5469	IN	NS	ns3.p34.dynect.net.
```  
  
Et qui retrouvons-nous? Avec un petit whois: ```whois dynect.net```  
  
```
Admin Name: Dynamic Network Services
Admin Organization: Dyn
...
```  

**Quelle alternative pour continuer à surfer sur la toile ?**
  
[OpenNIC](https://www.opennicproject.org/) est un Network Information Center/Serveur racine du DNS qui se pose comme une alternative à l'ICANN (société américaine qui contrôle et délègue l'attribution des noms de domaines de premier niveau (Top Level Domain, .com, .net, .org, ...) ainsi que l'adressage IP).   
  
Voici une liste de serveurs DNS que vous pouvez configurer sur votre machine dans l'espoir de pallier au problème: [https://servers.opennicproject.org](https://servers.opennicproject.org).  
  
Mais retenons que ces attaques massives sont possibles car commises par de **nombreux objets connectés compromis** et transformés en botnet. Peut-être que votre balance connectée, caméra ou autre était de la partie?  
  
Je vous dis à très bientôt et que la force soit avec vous !!!!  

