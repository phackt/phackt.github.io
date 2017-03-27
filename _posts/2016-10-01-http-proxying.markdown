---
layout: post
title:  "HTTP Proxying with Mitmproxy"
date:   2016-10-01
categories: mitm
---
<br />
Bonjour à tous,
  
Pour faire suite à l'[article]({{ site.url }}/mitm-phishing) que j'avais rédigé sur une attaque MITM redirigeant vers un site web spoofé, je me suis penché sur une autre méthode plus générique qui utilise l'outil [mitmproxy](https://mitmproxy.org/).  
  
Toujours dans un contexte MITM, l'objectif est d'identifier les liens sécurisés et redirections (**https**), de les 'stripper' en **http**, dans le but de maintenir le type de connexion suivante:  
  
```
victime <-- HTTP --> mitmproxy <-- HTTPS --> website
```
  
Le but est, qu'à partir d'une page non sécurisée d'un site, nous puissions continuer en proposant à notre victime une connexion en clair (http) pour analyser son trafic, mais que notre proxy rétablisse la connexion sécurisée avec le site concerné. Cette attaque ne pourra pas aboutir sur des sites implémentant le [HSTS](https://https.cio.gov/hsts/) (sauf si l'utilisateur ne s'est jamais connecté au site et que le HSTS n'est pas préchargé).  
  
Le stripping est effectué par le script [sslstrip.py](https://github.com/phackt/Workshops/blob/master/mitm/http_proxy/sslstrip.py), qui permet également de supprimer d'autres headers de sécurité, notamment les fameux cookies **secure** que nous avons abordés dans un [article précédent]({{ site.url }}/xss-cors-csrf-partie-2-xss-cookies-session).  
  
Vous trouverez le projet full et à jour (de nouvelles features apparaissent régulièrement) sur mon [github](https://github.com/phackt/mitm).  
  
J'ai également créé un petit script python, [chk_poison.py](https://github.com/phackt/Workshops/blob/master/mitm/http_proxy/chk_poison.py), qui va vérifier que votre ARP poisoning est opérationnel dans les deux sens (Victime \<-\> Passerelle). N'oubliez pas que certaines protections filtrent les résolutions ARP non sollicitées.  
  
Vous pouvez tester mitmproxy sans attaque MITM, en l'utilisant en mode [Regular](http://docs.mitmproxy.org/en/stable/modes.html). La commande sera la suivante:  

```bash
mitmproxy --anticache --host --anticomp --noapp --script ./sslstrip.py --eventlog
```
  
Il convient ensuite de configurer votre navigateur avec le proxy http **127.0.0.1:8080**. **Ne mettez rien pour le proxy https**, l'objectif ici n'est pas de générer à la volée de faux certificats et donc nous ne souhaitons pas capturer le trafic chiffré.  
  
*UPDATES: mitm.sh permet également d'injecter un payload javascript (ex Beef) dans les pages de la victime et d'effectuer une attaque de social engineering par DNS spoofing.  
  
Voir un example du projet sur ce post: [https://phackt.com/mitm-non-hsts-example](https://phackt.com/mitm-non-hsts-example)*  
  
**CONCLUSION:**  
  
**Sur vos sites web, sécurisez toutes vos pages (domaines et sous domaines). Ne laissez aucune opportunité à un assaillant de manipuler le trafic.**
  
**N'hésitez pas à soumettre vos idées, à contribuer au github, et à partager.**
  
Je vous dis à très bientôt!
