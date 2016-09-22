---
layout: post
title:  "Anonymat avec TOR et Proxychains sous Kali"
date:   2016-09-17
categories: web
---
<br />
Salut à tous,

Après m'être demandé comment lancer toutes mes commandes derrière un proxy [SOCKS](https://fr.wikipedia.org/wiki/SOCKS) pour masquer mon ip (certaines commandes ne proposent pas d'option pour rediriger vers un proxy SOCKS), vous pouvez vous en sortir grâce au réseau [TOR](https://fr.wikipedia.org/wiki/Tor_(r%C3%A9seau)) et un outil appelé [Proxychains](https://github.com/haad/proxychains/blob/master/src/proxychains.conf).  
  
Installez tor et proxychains:  

```
apt-get install tor
apt-get install proxychains
```
  
Tor écoute par défaut sur le port **9050**. Editer le fichier */etc/tor/torrc* et décommentez la ligne ```SOCKSPort 9050```.  
  
Lancer ensuite TOR as a service:  ```service tor start```  
  
Vous pouvez vérifier que TOR écoute en lançant la commande ```netstat -tulpn```, vous devriez voir:  

```
Proto Recv-Q Send-Q Adresse locale          Adresse distante        Etat        PID/Program name
tcp        0      0 127.0.0.1:9050          0.0.0.0:*               LISTEN      -
```
  
**Maintenant comment linker nos commandes sur TOR?**  
  
Proxychains va prendre en paramètre notre commande et linker le processus vers le(s) proxy(ies) de votre choix.  
  
 > *ProxyChains is a UNIX program, that hooks network-related libc functions in dynamically linked programs via a preloaded DLL and redirects the
   connections through SOCKS4a/5 or HTTP proxies.*  
  
Utilisation:  

```
ProxyChains-3.1 (http://proxychains.sf.net)
	usage:
		proxychains <prog> [args]
```
  
Proxychains demande de modifier sa configuration: ```gedit /etc/proxychains.conf```
  
Voici ma configuration:

```
# proxychains.conf  VER 3.1
#
#        HTTP, SOCKS4, SOCKS5 tunneling proxifier with DNS.
#	

# The option below identifies how the ProxyList is treated.
# only one option should be uncommented at time,
# otherwise the last appearing option will be accepted
#
dynamic_chain
#
# Dynamic - Each connection will be done via chained proxies
# all proxies chained in the order as they appear in the list
# at least one proxy must be online to play in chain
# (dead proxies are skipped)
# otherwise EINTR is returned to the app
#
#strict_chain
#
# Strict - Each connection will be done via chained proxies
# all proxies chained in the order as they appear in the list
# all proxies must be online to play in chain
# otherwise EINTR is returned to the app
#
#random_chain
#
# Random - Each connection will be done via random proxy
# (or proxy chain, see  chain_len) from the list.
# this option is good to test your IDS :)

# Make sense only if random_chain
#chain_len = 2

# Quiet mode (no output from library)
quiet_mode

# Proxy DNS requests - no leak for DNS data
proxy_dns

# Some timeouts in milliseconds
tcp_read_time_out 15000
tcp_connect_time_out 8000

# ProxyList format
#       type  host  port [user pass]
#       (values separated by 'tab' or 'blank')
#
#
#        Examples:
#
#            	socks5	192.168.67.78	1080	lamer	secret
#		http	192.168.89.3	8080	justu	hidden
#	 	socks4	192.168.1.49	1080
#	        http	192.168.39.93	8080	
#		
#
#       proxy types: http, socks4, socks5
#        ( auth types supported: "basic"-http  "user/pass"-socks )
#
[ProxyList]
# add proxy here ...
# meanwile
# defaults set to "tor"
socks5 	127.0.0.1 9050
```
  
**dynamic_chain**: pertinent surtout si vous chainez plusieurs proxies, les proxies down ne seront simplement pas pris en compte.  
**quiet_mode**: sortie non verbeuse.  
**proxy_dns**: demandera au proxy d'effectuer les résolutions DNS (SOCKS4a et SOCKS5).  
**tcp_read_time_out 15000, tcp_connect_time_out 8000**: socket timeout.  
**socks5 127.0.0.1 9050**: ici nous utilisons le réseau TOR.  
  
J'ai eu perso une erreur du type:  
<code>ERROR: ld.so: object 'libproxychains.so.3' from LD_PRELOAD cannot be preloaded: ignored.</code>  
  
La librairie partagée *libproxychains.so.3* n'a pas été trouvée. La variable d'environnement *LD_PRELOAD* n'est donc pas correctement initialisée (variable qui permet d'effectuer le hook des appels aux sockets).  Editez le fichier */usr/bin/proxychains* et remplacez  
```export LD_PRELOAD=libproxychains.so.3```  
par  
```export LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libproxychains.so.3```.  
  
Au besoin effectuez un ```locate libproxychains.so.3```.  
  
Je vous dis à très bientôt!
