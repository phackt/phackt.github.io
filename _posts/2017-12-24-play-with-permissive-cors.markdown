---
layout: post
title:  "Play with permissive CORS"
date:   2017-12-24
category: Web Security
excerpt_separator: <!--more-->
---
Hello folks,  
  
While i was playing with a 'code search engine' tool called [publicwww](https://publicwww.com), i decided to gather some top Alexa domains and to look for some permissive CORS. I tried the following researches, ```site:fr "type=\"password\""```, the same for the TLDs *.org* and *.com*, in order to spot webpages with a login form, meaning authenticated users..  
<!--more-->
  
I will let you play with publicwww, remember that you are limited with basic access but if you pay you will have access to the top 200000 Alexa websites dealing with your request. May be you also wanna play with Shodan or Censys.  
  
**Automate permissive CORS detection**  

We already talked about **Cross Origin Ressource Sharing** in some [previous posts](https://phackt.com/xss-cors-csrf-partie-3-cors-csrf). You also have a great explanation about the **Same Origin Policy** on [mozilla.org](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy). A website with permissive CORS will allow an attacker to perform any requests to this website from another ressource and this request can include the victim's cookies (xhr.withCredentials = true).  
  
So if the cookie's session is reflected, you will be able to steal it and forge the victim's session. You will be able to change the victim's password if the former one is not requested, read some sensitive account information, .... Remember that because now you can READ the response, you also **can read the CSRF token** and bypass the protection.  
  
So once i gathered enough interesting domains and urls from OSINT tools, and got a bunch of ones dealing with bug some bounty programs, i decided to be original and to create a tool called [cors.py](https://github.com/phackt/pentest/tree/master/fingerprint/web/cors) (*DEPRECATED right now, check below*).  
  
This tool reads a list of domains and/or urls and will HTTP request them, adding the **Origin** header of your choice, example:  
```Origin: https://phackt.com```.  
  
It will finally detects a permissive cors if the response header **Access-Control-Allow-Origin** is set to *null* or if the origin header's value is reflected, and if the response header **Access-Control-Allow-Credentials** is set to *true* (permissive CORS is relevant only if you can read from request including victim's cookies). You also can detect some strings in the response headers (see config.json) and configure other parameters like the number of threads, the user agent, the response timeout, and so on.  
  
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
  
**UPDATE 01/07/2020:I'm strongly suggesting right now to use this complete tool of which i merged most of CORS detecting features: [https://github.com/chenjj/CORScanner](https://github.com/chenjj/CORScanner).**  
  
This tool gathers everything you need to detect an exploitable CORS misconfiguration <i class="fa fa-usd"></i><i class="fa fa-usd"></i><i class="fa fa-usd"></i> :  
  
Misconfiguration type    | Description
------------------------ | --------------------------
Reflect_any_origin       | Blindly reflect the Origin header value in `Access-Control-Allow-Origin headers` in responses, which means any website can read its secrets by sending cross-orign requests.
Prefix_match             | `wwww.example.com` trusts `example.com.evil.com`, which is an attacker's domain.
Suffix_match             | `wwww.example.com` trusts `evilexample.com`, which could be registered by an attacker.
Not_escape_dot           | `wwww.example.com` trusts `wwwaexample.com`, which could be registered by an attacker.
Substring match          | `wwww.example.com` trusts `example.co`, which could be registered by an attacker.
Trust_null               | `wwww.example.com` trusts `null`, which can be forged by iframe sandbox scripts
HTTPS_trust_HTTP         | Risky trust dependency, a MITM attacker may steal HTTPS site secrets
Trust_any_subdomain      | Risky trust dependency, a subdomain XSS may steal its secrets
Custom_third_parties     | Custom unsafe third parties origins like `github.io`, see more in [origins.json](./origins.json) file. Thanks [@phackt](https://github.com/phackt)!
Special_characters_bypass| Exploiting browsers’ handling of special characters. Most can only work in Safari except `_`, which can also work in Chrome and Firefox. See more in [Advanced CORS Exploitation Techniques](https://www.corben.io/advanced-cors-techniques/). Thanks [@Malayke](https://github.com/Malayke).

  
Cheers.
