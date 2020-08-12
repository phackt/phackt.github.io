---
layout: post
title:  "MITM Part 1 - Attaque MITM sur HTTPS"
date:   2016-10-01
category: Pentesting
excerpt_separator: <!--more-->
---
Bonjour à tous,

L'objectif est de partager avec vous un [script](https://github.com/phackt/mitm) créé pour automatiser une attaque 'Man In The Middle' permettant l'obtention des identifiants de la victime de la façon la plus transparente possible.  
<!--more-->
  
**Préambule:**  
  
Nous allons donc cibler un site possédant au moins une page non sécurisée (HTTP), permettant ainsi l'interception en clair des liens HTTPS et de les transformer en liens HTTP classiques (*HTTP Strict Transport Security* (HSTS) ne doit pas s'appliquer comme nous le verrons par la suite). Certains outils comme [sslstrip](https://github.com/moxie0/sslstrip) permettent ce genre d'opérations, en éliminant le caching des pages et en traitant également les headers response **Location** lors des redirections.  
  
De notre coté nous utiliserons un script *Python*, [sslstrip.py](https://github.com/phackt/mitm/blob/master/script/sslstrip.py), développé pour l'occasion, que nous utiliserons avec l'outil [mitmproxy](https://mitmproxy.org/).  
  
Pour rappel:  

 > L'attaque de l'homme du milieu ou man-in-the-middle attack (MITM) est une attaque qui a pour but d'intercepter les communications entre deux parties, sans que ni l'une ni l'autre ne puisse se douter que le canal de communication entre elles a été compromis.  
  
![Mitm]({{ site.url }}/public/images/mitm-phishing/owasp-man_in_the_middle.jpg)  
Cette attaque se base sur la corruption du cache ARP enregistrant les correspondances @IP <-> @MAC sur un réseau local.  
  
Toujours dans un contexte MITM, l'objectif est d'identifier les liens sécurisés et redirections (**https**), de les 'stripper' en **http**, dans le but de maintenir le type de connexion suivante:  
  
```
victime <-- HTTP --> mitmproxy <-- HTTPS --> website
```
<!--more-->
  
Le but est qu'à partir d'une page non sécurisée d'un site, nous puissions continuer en proposant à notre victime une connexion en clair (http) pour analyser son trafic, **mais que notre proxy communique en SSL/TLS avec le site concerné**. Cette attaque ne pourra pas aboutir sur des sites implémentant le [HSTS](https://https.cio.gov/hsts/) (sauf si l'utilisateur ne s'est jamais connecté au site et que le HSTS n'est pas préchargé - mécanisme de *Trust On First Use*).  
  
Le stripping est effectué par le script [sslstrip.py](https://github.com/phackt/mitm/blob/master/script/sslstrip.py), qui permet également de supprimer d'autres headers de sécurité, notamment les fameux cookies **secure** que nous avons abordés dans un [article précédent]({{ site.url }}/xss-cors-csrf-partie-2-xss-cookies-session).  
  
**Premier test:**  

Vous trouverez le projet full et à jour (de nouvelles features apparaissent régulièrement) sur mon [github](https://github.com/phackt/mitm).  
  
Le script [chk_poison.py](https://github.com/phackt/mitm/blob/master/bin/chk_poison.py) va également permettre de vérifier que votre ARP poisoning est opérationnel dans les deux sens (Victime \<-\> Passerelle). N'oubliez pas que certaines protections filtrent les résolutions ARP non légitimes.  
  
Vous pouvez donc tester mitmproxy sans attaque MITM, en l'utilisant en mode [Regular](http://docs.mitmproxy.org/en/stable/modes.html). La commande sera la suivante:  

```bash
mitmproxy --anticache --host --anticomp --noapp --script ./sslstrip.py --eventlog
```
  
Il convient ensuite de configurer votre navigateur avec le proxy http **127.0.0.1:8080**. **Ne mettez rien pour le proxy https**, l'objectif ici n'est pas de générer à la volée de faux certificats et donc nous ne souhaitons pas capturer le trafic chiffré.  
  
```mitm.sh``` permet également d'injecter un payload javascript (ex Beef) dans les pages de la victime et d'effectuer une attaque de social engineering par DNS spoofing.  
  
Vous trouverez une véritable attaque de bout en bout sur ce [post](https://phackt.com/mitm-part2-hands-on-mitm-attack-and-https).  
  
  
Je vous dis à très bientôt!
