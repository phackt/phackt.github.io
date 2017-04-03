---
layout: post
title:  "XSS, CORS, CSRF (Partie 1)"
date:   2016-08-09
categories: web
---
Le XSS, CORS, CSRF... Késako?
----------
**Que se cache-t-il derrière ces acronymes barbares ?**  
  
Bienvenue dans cette saga qui traitera des notions de XSS, CORS, CSRF et du lien entre elles.  
Vous en avez forcément entendu parler si vous avez été impliqués dans la création d’applications WEB. L’idée de cet article m’est venue suite à la création d’une application WEB "cas d’école" en Java [https://github.com/phackt/DemoWebApp](https://github.com/phackt/DemoWebApp). La question que nous nous posons est la suivante : **quelles sont les bonnes pratiques pour sécuriser une application WEB ?**  
  
Dans ce premier volet, nous traiterons des attaques de cross-site scripting. Nous ferons le lien par la suite avec les protections CSRF (Cross Site Request Forgery) et les requêtes CORS (Cross Origin Ressource Sharing).    
  
XSS 
====
Définition de Wikipédia :  
*" Le cross-site scripting (abrégé XSS), est un type de faille de sécurité des sites web permettant d'injecter du contenu dans une page, permettant ainsi de provoquer des actions sur les navigateurs web visitant la page. Les possibilités des XSS sont très larges puisque l'attaquant peut utiliser tous les langages pris en charge par le navigateur (JavaScript, Java, Flash...) et de nouvelles possibilités sont régulièrement découvertes notamment avec l'arrivée de nouvelles technologies comme HTML5. Il est par exemple possible de rediriger vers un autre site pour de l'hameçonnage ou encore de voler la session en récupérant les cookies."* 
  
L’objectif est donc l’injection de code dans la page HTML pouvant être interprété par le navigateur. Si les balises permettant l’interprétation de code arbitraire ne sont pas bien filtrées, exemple classique avec la balise ```<script>```, alors nous pourrons exécuter du code à l’insu de l’utilisateur.   
Il existe deux types d’attaques XSS, les réfléchies, et les stockées. Les réfléchies sont injectées dans la requête et le code interprétable n’est pas stocké de façon permanente sur le serveur.  
  
Un exemple basique sur une page PHP :  
  
```php
<?php
  $name = $_GET['name'];
  echo "Welcome $name";
?>
```
   
Un autre exemple en JSP :  
  
```java
<% String name = request.getParameter("name"); %> 
Employee name: <%= name %>  
```
   
Si vous appelez ```/index?name=<script>prompt(1)</script>```, votre navigateur interprétera la balise ```<script>``` et affichera un donc un prompt. Aucun intérêt ici mais on vous laisse deviner la portée d’une telle attaque : un payload javascript pourra voler vos informations de session (*document.cookie*), envoyer des requêtes à votre insu sur le site vulnérable ou faire de la redirection dans un but d’hameçonnage. Les attaques XSS Reflected sont souvent associées à de l’ingénierie sociale car l’utilisateur doit accéder à un lien provoquant l’exécution du code. 
  
Cependant ce code peut être stocké de façon permanente dans les attaques XSS Stored. Le principe est le même sauf que le code est stocké sur le serveur, le cas classique étant un forum vulnérable qui sauvegarde les messages infectés ([OWASP (Open Web Application Security Project) Cross-Site Scripting](https://www.owasp.org/index.php/Cross-site_Scripting_%28XSS%29)).
   
Comment s’en prémunir ? 
====
  
Il convient d'assainir en entrée les données (Filter) et d'encoder l'information pour les réponses HTTP. Tout dépend du langage de programmation, framework utilisé, et également de l'endroit où se situe point d'injection.  
  
Par exemple si vous travaillez avec Spring MVC:
```
<context-param>
   <param-name>defaultHtmlEscape</param-name>
   <param-value>true</param-value>
</context-param>
```
  
Cependant ceci n'échappera que les Spring tags: ```<form:input path="formField" htmlEscape="true" />```  
  
Si vous codez des pages JSP :  

 - ```${description}``` est vulnérable en Expression Language, idem pour  ```<%=description%>``` en scriptlet. Préférez ```${fn:escapeXml(file.description)}``` ou encore la JSTL (JavaServer Pages Standard Tag Library) ```<c:out value="${file.description}```. La balise JSTL ```<c:out/>``` permet d’afficher une variable et possède l'attribut ```escapeXml``` à *true* par défaut (caractères <,>,&,’,").  
  
 - Tester les fonctions d’échappement du framework de rendu que vous utilisez et si besoin échappez vos entrées coté serveur : [HtmlUtils.htmlEscape]( http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/util/HtmlUtils.html#htmlEscape-java.lang.String- "http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/util/HtmlUtils.html#htmlEscape-java.lang.String-") avec Spring, [StringEscapeUtils.escapeHtml4]( https://commons.apache.org/proper/commons-lang/javadocs/api-release/ "https://commons.apache.org/proper/commons-lang/javadocs/api-release/") d’Apache Commons, ou le [Java HTML Sanitizer]( https://github.com/OWASP/java-html-sanitizer/blob/master/docs/getting_started.md "https://github.com/OWASP/java-html-sanitizer/blob/master/docs/getting_started.md") d’OWASP.  
  
Autre exemple si vous codez en PHP, utilisez les fonctions ```htmlentities``` (avec l'option ENT_QUOTES, pour échapper les simple quotes), ou ```htmlspecialchars```.  
  
Vous pouvez également protéger votre application derrière un **Web Application Firewall** (ex: [ModSecurity](http://www.modsecurity.org/ "http://www.modsecurity.org/")). ModSecurity met à disposition une [page](https://www.modsecurity.org/crs-demo.html "https://www.modsecurity.org/crs-demo.html") permettant d’éprouver leur moteur de détection d’injection XSS.

Si vous utilisez la couche **Spring Security** (ce qui a été mon cas ;)), le framework inclut par défaut de nombreux headers dans la réponse HTTP:  

 - **X-Content-Type-Options**: Stipule de ne pas deviner le MIME-Type si mal renseigné (spécifique à certaines attaques XSS) – voir [OWASP XSS Filter Evasion Cheat Sheet](https://www.owasp.org/index.php/XSS_Filter_Evasion_Cheat_Sheet).  
 
 - **X-XSS-Protection**: Stipule d'activer l'auditeur XSS du navigateur. Si votre site est vulnérable au XSS, mais que votre réponse contient le header ```X-XSS-Protection: 1; mode=block```, vous aurez le message suivant dans la console de votre navigateur (ici Chrome): <span style="color: red">```Refused to execute inline script because it violates the following Content Security Policy Directive```</span>.
  
 - **X-Frame-Options**: Spécifie au navigateur qu'une page ne peut être rendue dans un ```<frame>```, ```<iframe>``` ou ```<object>```.  
  
Ces headers, ainsi que HTTP **Strict-Transport-Security** (abrégé HSTS – oblige le navigateur à requêter sur du HTTPS, utile pour lutter contre le blocage des connexions sécurisées HTTPS avec des outils comme sslstrip lors d'attaques "Man In The Middle"), ou **Cache-Control** sont par défaut inclus et activés dans la couche Spring Security ([Spring Security Headers](http://docs.spring.io/spring-security/site/docs/current/reference/html/headers.html "http://docs.spring.io/spring-security/site/docs/current/reference/html/headers.html")).  
Il convient également de configurer l'ajout du header **[Content-Security-Policy](https://www.w3.org/TR/CSP/)** qui propose de nombreuses directives de sécurité.  
  
Si vous utilisez une autre technologie ou framework, pensez à inclure ces response headers en fonction de vos besoins.  
  
Pensez également à sécuriser vos **cookies session**. Les requêtes XSS ont pour objectif le vol de ces cookies. Stipulez au navigateur que ce dernier ne peut pas y accéder via javascript (flag **HttpOnly**). Pour un site HTTPS, ces derniers ne doivent pas être accessibles sur des connexions non cryptées (flag **Secure**): [https://tools.ietf.org/html/rfc6265#section-5.2.4](https://tools.ietf.org/html/rfc6265#section-5.2.4).
  
Des outils pour vos développements sécurisés 
====
  
> Eprouvez votre application avec un outil tel que [Xenotix]( https://www.owasp.org/index.php/OWASP_Xenotix_XSS_Exploit_Framework "https://www.owasp.org/index.php/OWASP_Xenotix_XSS_Exploit_Framework").  
 Vous disposez aussi des supports [OWASP XSS Filter Evasion Cheat Sheet]( https://www.owasp.org/index.php/XSS_Filter_Evasion_Cheat_Sheet "https://www.owasp.org/index.php/XSS_Filter_Evasion_Cheat_Sheet") et [OWASP Prevention Cheat Sheet](https://www.owasp.org/index.php/XSS_(Cross_Site_Scripting)_Prevention_Cheat_Sheet).  
  
Il est également important de parler de l’api d’OWASP [ESAPI (The OWASP Enterprise Security API)](https://www.owasp.org/index.php/Category:OWASP_Enterprise_Security_API "https://www.owasp.org/index.php/Category:OWASP_Enterprise_Security_API") qui est une API libre, open source, qui facilite le développement sécurisé d’applications, facilement intégrable à une application existante ou pour de nouveaux développements. Cette API offre de nombreuses implémentations permettant les validations des entrées, l'authentification, l'encoding, l'échappement, ... l'API existe pour différents langages, JAVA EE, .NET, PHP, Python, Javascript, ASP, Coldfusion & CFML.
  
Il existe également d’autres projets qui peuvent répondre à vos besoins :  
 
 - Output encoding: [OWASP Java Encoder Project](https://www.owasp.org/index.php/OWASP_Java_Encoder_Project "https://www.owasp.org/index.php/OWASP_Java_Encoder_Project")  
 - General HTML sanitization: [OWASP Java HTML Sanitizer](https://www.owasp.org/index.php/OWASP_Java_HTML_Sanitizer_Project "https://www.owasp.org/index.php/OWASP_Java_HTML_Sanitizer_Project")
 - Validation: [JSR-303/JSR-349 Bean Validation](http://beanvalidation.org/ "http://beanvalidation.org/")
 - Strong cryptography: [Keyczar](https://github.com/google/keyczar "https://github.com/google/keyczar")
 - Authentication / authorization: [Apache Shiro](https://shiro.apache.org/ "https://shiro.apache.org/")
 - CSRF protection: [OWASP CSRFGuard Project](https://www.owasp.org/index.php/Category:OWASP_CSRFGuard_Project "https://www.owasp.org/index.php/Category:OWASP_CSRFGuard_Project") or [OWASP CSRFProtector Project](https://www.owasp.org/index.php/CSRFProtector_Project "https://www.owasp.org/index.php/CSRFProtector_Project").  
  
Les attaques possibles sont nombreuses sur une application web, nous avons mis en avant celles de type XSS. Nous verrons dans un prochain article comment ces attaques peuvent détourner les protections CSRF, voler les cookies de session et dans quelle mesure les requêtes Cross-Origin impactent la sécurité d’une application web.  
  
A bientôt!  
  
### [PARTIE 2](https://phackt.com/xss-cors-csrf-partie-2-xss-cookies-session)  
  
Références:
 - [http://stackoverflow.com/questions/2147958/how-do-i-prevent-people-from-doing-xss-in-spring-mvc](http://stackoverflow.com/questions/2147958/how-do-i-prevent-people-from-doing-xss-in-spring-mvc)
