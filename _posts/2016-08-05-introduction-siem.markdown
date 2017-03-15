---
layout: post
title:  "Introduction au SIEM"
date:   2016-08-05
categories: siem
---
SIEM, le monitoring de la sécurité
===================
Nous avons vu dans un précédent article quel était le contexte réglementaire de la [Loi de Programmation Militaire]({{ site.url }}/introduction-cybersecurite) française.  

**Que dites-vous d’une petite révision ?**  

La LPM impose aux **OIV** (Opérateurs d’Importance Vitale) de renforcer la sécurité des systèmes d’information critiques qu’ils exploitent et de notifier les incidents de sécurité à l’[ANSSI](http://www.ssi.gouv.fr/) (Agence Nationale de la Sécurité des Systèmes d'Information).  
Une première vague d’arrêtés a été publiée le 1er juillet 2016. Ces arrêtés concernent les secteurs d’activité des produits de santé, la gestion de l’eau et l’alimentation. Plusieurs autres arrêtés, concernant d’autres secteurs d’activité, devraient paraître d’ici la fin de cette année.  

Notez que la Commission européenne a également adopté, en février 2013, une proposition de directive visant à assurer un niveau élevé commun de sécurité des réseaux et de l’information dans l’Union.  
  
Les OIV se retrouvent donc avec l'obligation de mettre en œuvre des systèmes qualifiés de détection des évènements susceptibles d’affecter la sécurité de leurs systèmes d’information. En cas de manquement, ceci peut coûter cher : la LPM 2014-2019 sanctionne les manquements à la loi d’une amende de 150.000€, s’élevant à 750.000€ pour les personnes morales. La loi française ne distingue pas selon que le manquement est ou non intentionnel. La simple négligence est donc en principe condamnable.  

Nous voyons donc qu’en sus d'un réel besoin, le cadre réglementaire concernant la supervision de la sécurité des entreprises est posé.  

**Mais en quoi consiste ce monitoring de la sécurité ?**  

Nous allons aborder la notion de **SIEM** : Security Information and Event Management.  
Le SIEM est le résultat de la volonté de superviser l’activité de son système d’information.  
Il a donc pour objectif de collecter les fichiers d’évènements (logs) de différentes sources, de les uniformiser, et de les corréler. On parle de corrélation car ces solutions sont munies de moteurs de corrélation qui permettent de relier plusieurs évènements à une même cause.  

Un SIEM impliquera également la création d’une équipe dédiée, capable d’assurer plusieurs niveaux de services autour de l’outil. En effet, il faudra être en capacité de lire, d’analyser, de vérifier, de valider les alertes de sécurité remontées par la solution 7j/7 24H/24, les assaillants n’attendant pas patiemment les heures ouvrées pour commettre leurs exactions.  

En somme, il faudra créer un SOC ([Security Operation Center](https://fr.wikipedia.org/wiki/Security_Operations_Center "https://fr.wikipedia.org/wiki/Security_Operations_Center")).

**Quel rapport entre journaux et sécurité ?**  
 
Le monitoring du SI implique deux besoins : __la gestion des logs et la supervision de la sécurité__.  

*La gestion des logs*, plus souvent appelée « Log Management », consiste en la mise en place d’une ou plusieurs solutions permettant à une entreprise de
collecter, stocker, sécuriser et archiver les journaux d’information provenant des différents équipements de sécurité, d’authentification, réseaux...  

Ce besoin fait très souvent suite à des contraintes légales de conservation et de non-répudiation des informations. Un exemple est l’analyse post incident et la visualisation de requêtes d’exfiltration de données par le hacker. La brique d’indexation et de recherche est assurée par un système d’indexation très rapide (exemple ElasticSearch en open source).  

*La supervision de la sécurité* quant à elle, relève d’un réel besoin des entreprises de suivre en temps réel l’activité de leur SI, de corréler tous les évènements qui s’y passent et d’être alertés en cas de problème de sécurité. Cette partie sera assurée par une couche **HIDS** (Host Based Intrusion Detection System - ex: OSSEC) / **NIDS** (Network Intrusion Detection System - ex: SNORT). Les alertes de ces systèmes seront remontées au SIEM qui effectuera des **corrélations** (règles de détection mises à jours régulièrement, expressions régulières) permettant de détecter des comportements suspicieux probants, et ainsi d’anticiper une attaque (exemple de scan de ports – voir [nmap](https://nmap.org/book/man-port-scanning-techniques.html "https://nmap.org/book/man-port-scanning-techniques.html")), ou de remonter une alerte sur une attaque en cours.  De surcroît, les informations remontées par les NIDS et HIDS sont **complémentaires** : le NIDS remontera la source de l'attaque (IP) et le payload envoyé, le HIDS alertera sur l'action effectuée sur le système ciblé et si cette dernière a abouti.  
  
Ainsi des règles de corrélations pourront être mises en place pour déterminer la criticité d'une alerte (évitez les règles trop génériques pouvant mener à des faux positifs). Voici un exemple de règle de corrélation d'un SIEM impliquant des alertes de SNORT et d'OSSEC sur une attaque LFI ([Local File Inclusion](https://www.owasp.org/index.php/Testing_for_Local_File_Inclusion)):

![Corrélation LFI]({{ site.url }}/public/images/introduction-siem/events_flowchart1.png)  
  
Le SIEM dispose d'une partie graphique pour l'affichage des Key Risk Indicators(reportings et dashboards).  
Ces composantes sont donc intimement liées. Les solutions doivent faire appel à des technologies de Big Data pour assurer la rapidité de traitement et garantir l’intégrité de ces gros volumes de données qui transitent par le SIEM. 
 
**Quelles sont donc ces solutions ?**  

Selon le Gartner Magic Quadrant 2015 :  

![Gartner Magic Quadrant SIEM 2015]({{ site.url }}/public/images/introduction-siem/2015-siem-mq-LG.png)  


And the winner is…. IBM Security QRadar, suivi de HP ArcSight et de Splunk.  
Ces solutions sont commerciales, mais le geek qui est en vous se demande déjà ce qu’il en est des solutions libres ? Effectivement il existe des solutions comme [OSSIM](https://www.alienvault.com/products/ossim "https://www.alienvault.com/products/ossim") qui est un SIEM open source, mais cependant limité (absence de règles de corrélations, reporting basique, ...).  

Il existe également une solution d’agrégation de logs, ou plutôt une stack open source de produits Elastic, la stack **ELK (ElasticSearch, Logstash, Kibana)** :

 - [ElasticSearch](https://www.elastic.co/fr/products/elasticsearch "https://www.elastic.co/fr/products/elasticsearch") : moteur de stockage et d’indexation de documents et moteur de requête/d’analyse de ceux-ci.
 - [Logstash](https://www.elastic.co/products/logstash "https://www.elastic.co/products/logstash"): analyse, filtrage et découpage des logs pour les transformer en documents, parfaitement formatés notamment pour ElasticSearch.
 - [Kibana](https://www.elastic.co/products/kibana "https://www.elastic.co/products/kibana"): dashboard interactif et paramétrable permettant de visualiser les données stockées dans ElasticSearch.

Notre premier besoin est couvert, mais concernant notre supervision de la sécurité ?

Pour notre HIDS, vous trouverez la solution open source [OSSEC](http://ossec.github.io/ "http://ossec.github.io/"). Concernant le NIDS, voici [SNORT](https://www.snort.org/ "https://www.snort.org/").

Notre prochain article consacré au SIEM se concentrera donc sur la mise en place de cette solution ELK + surcouche OSSEC. Nous ferons également le parallèle avec les enjeux [Big Data](http://www.bgfi-groupe.com/ "http://www.bgfi-groupe.com/") concernant ces solutions.  
  
A bientôt.
<br />
<br />
Références:  
[http://www.orange-business.com/fr/blogs/securite/securite-organisationnelle-et-humaine/securite-siem-ou-pas-siem-](http://www.orange-business.com/fr/blogs/securite/securite-organisationnelle-et-humaine/securite-siem-ou-pas-siem-)  
[https://www.itrust.fr/SIEM](https://www.itrust.fr/SIEM)   
[http://www.itrmanager.com/articles/164117/siem-element-incontournable-oiv.html](http://www.itrmanager.com/articles/164117/siem-element-incontournable-oiv.html)  
[http://www.village-justice.com/articles/Quelles-obligations-pour-les-OIV,16739.html](http://www.village-justice.com/articles/Quelles-obligations-pour-les-OIV,16739.html)  
[http://www.wazuh.com/elk-stack/](http://www.wazuh.com/elk-stack/)  
[http://connect.ed-diamond.com/MISC/MISC-069/SIEM-IDS-l-union-fait-elle-la-force](http://connect.ed-diamond.com/MISC/MISC-069/SIEM-IDS-l-union-fait-elle-la-force)
