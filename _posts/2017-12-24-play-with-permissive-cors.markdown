---
layout: post
title:  "Play with permissive CORS"
date:   2017-12-24
categories: web
---
Hello folks,  
  
While i was playing with a 'search code search engine' tool called [publicwww](https://publicwww.com), i decided to gather some top Alexa domains and to look for some permissive CORS. I tried the following researches, ```site:fr "type=\"password\""```, the same for the TLDs *.org* and *.com*, in order to spot webpages with a login form.  
  
I also looked for some other stuff like websites with french language and so on. I will let you play with publicwww, remember that you are limited with basic access but if you pay you will have access to the top 200000 Alexa websites dealing with your request. May be you also wanna play with Shodan or Censys.  
  
**Python tool to detect permissive CORS**  

We already talked about **Cross Origin Ressource Sharing** in some [previous posts](https://phackt.com/xss-cors-csrf-partie-3-cors-csrf). You also have a great explanation about the **Same Origin Policy** on [mozilla.org](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy). A website with permissive CORS will allow an attacker to perform any requests to this website from another ressource and this request can include the victim's cookies (xhr.withCredentials = true). So if the cookie's session is reflected, you will be able to steal it and forge the victim's session. You will be able to change the victim's password if the former one is not requested, read some sensitive account information, .... Remember that because now you can READ the response, you also can read the CSRF token and bypass the protection.  
  
So once i gathered enough interesting domains and urls from OSINT tools, and got a bunch of ones dealing with bug some bounty programs, i decided to be original and to create a tool called [cors.py](https://github.com/phackt/pentest/tree/master/fingerprint/cors).  
  
This tool reads a list of domains and/or urls and will HTTP request them, adding the **Origin** header of your choice, example: ```Origin: https://phackt.com```. It will finally detects a permissive cors if the response header **Access-Control-Allow-Origin** is set to *null* or if the origin header's value is reflected, and if the response header **Access-Control-Allow-Credentials** is set to *true* (permissive CORS is relevant only if you can read from request including victim's cookies). You also can detect some strings in the response headers (see config.json) and configure other parameters like the number of threads, the user agent, the response timeout, and so on.  
  
**HOW TO**  

```
$ ./cors.py -h
usage: cors.py [-h] [-f URLSFILE] [-r]

Looking for some permissive CORS

optional arguments:
  -h, --help            show this help message and exit
  -f URLSFILE, --file URLSFILE
                        file with urls
  -r, --redirect        allow redirect in requests
```
  
Example of output:  
```
$ ./cors.py -f /tmp/sitecomtypepassword.txt 
Processing 230 lines
Launching 20 threads
open cors found for url http://asiaroom.com/
open cors found for url http://aksekiyapi.com/
open cors found for url http://baseshare.com/
open cors found for url http://fitwall.com/
open cors found for url http://h10.edmarkreadingonline.com/
open cors found for url http://ermresearch.com/
open cors found for url http://ellysdirectory.com/
...
```
  
I also provide some [lists](https://github.com/phackt/pentest/tree/master/fingerprint/cors/results) of vulnerable domains (total of 11435) that i found.  
  
Don't hesitate to contact me if you have any idea or comments, or if you played with one of the domains, for example **www.attractiveworld.com** ;).  
  
Cheers.