---
layout: post
title:  "Fingerprint Web Application static files"
date:   2017-06-05
categories: web
---
Hello folks,  
  
Today i would like to share with you the first version of a script i wrote to help determining the version of a software.  
  
**So what the hell am i talking about?**  
  
Ok let me explain the context: You are looking at a website and trying to get the exact version of the underlying CMS, let say Drupal. We know that Drupal has suffered from the [Drupalgeddon](https://www.drupal.org/project/drupalgeddon) for versions 7.X before 7.32. So you can have at look at some obvious files (CHANGELOG.txt), but what happens if the webmaster deleted these files? How to maximize the chance to find the right version or at least the smallest delta as possible ?  
  
A tool was existing for that purpose, [BlindElephant](https://github.com/lokifer/BlindElephant), but it is not maintained anymore. So i chose to write my own script matching my needs (hand made with love). Of course don't hesitate to contact me and let me know if you took time to have BlindElephant in a working state with the CMS hashes databases up-to-date.  
  
So now after your recon phase you have your target using Drupal CMS (it will work with any CMS files versioned on GIT). Here we will install a Drupal on our local machine.  
[WebApp Information Gatherer](https://github.com/jekyc/wig) may helps your to identify it:  
```bash
./wig.py http://localhost/drupal
...
Name           Versions          Type               
Drupal         7                 CMS   
...
```
  
Of course if you can hit some files leaking the CMS version, in this case everything is already folded:  
```
curl --silent http://localhost/drupal/CHANGELOG.txt | head -1
Drupal 7.29, 2014-07-16
```
  
You also can look for some obvious information in the response headers or for any meta information in the HTML page. But if the webmaster did its job, nothing will leak so how can we fingerprint our CMS?  
  
What do you need:  
 - access to versioned project
 - find the most relevant (updated) files as input files (git log)
 - download these files from your target
 - hash all these input files and their GIT releases equivalent
 - compare input files hashes with the releases ones
  
Following all these steps will help to determine the most likely versions of the underlying CMS.  
  
So first download the last version of [versionchecker.sh](https://raw.githubusercontent.com/phackt/pentest/master/fingerprint/versionchecker.sh):  
```bash
cd /tmp && wget https://raw.githubusercontent.com/phackt/pentest/master/fingerprint/versionchecker.sh
chmod +x versionchecker.sh
```
  
Then clone the [Drupal repo](https://github.com/drupal/drupal):
```bash
git clone https://github.com/drupal/drupal.git
cd drupal/
```
  
Now we will look for our relevant input files:  
```bash
git log --all --pretty=format: --name-only | egrep -v "^core/" | egrep -i "(.js$|.html$|.css$)" | sort | uniq -c | sort -rg | head -30 | tee /tmp/relevant_files.txt
warning: inexact rename detection was skipped due to too many files.
warning: you may want to set your diff.renameLimit variable to at least 4708 and retry the command.
    170 misc/drupal.css
    106 themes/seven/style.css
    106 themes/garland/style.css
    102 misc/drupal.js
     85 modules/system/system.css
     78 modules/overlay/overlay-parent.js
     72 themes/bartik/css/style.css
     67 misc/tabledrag.js
     57 misc/autocomplete.js
     55 themes/xtemplate/xtemplate.css
     55 modules/system/system.js
     52 misc/ajax.js
     42 misc/collapse.js
     41 misc/textarea.js
     41 misc/tableheader.js
     37 themes/pushbutton/style.css
     35 themes/bluemarine/style.css
     35 modules/user/user.js
     35 misc/progress.js
     33 modules/user/user.css
     32 modules/toolbar/toolbar.css
     30 misc/tableselect.js
     29 modules/system/admin.css
     28 themes/garland/style-rtl.css
     28 misc/jquery.js
     26 modules/color/color.js
     26 modules/block/block.js
     25 misc/teaser.js
     24 themes/bartik/css/style-rtl.css
     24 modules/node/node.css
     23 themes/chameleon/common.css
     23 modules/toolbar/toolbar.js
     23 modules/system/system-rtl.css
     22 modules/dashboard/dashboard.css
     22 misc/ahah.js
     21 themes/garland/fix-ie.css
     20 themes/xtemplate/pushbutton/xtemplate.css
     20 modules/overlay/overlay-parent.css
     20 modules/openid/openid.js
     20 modules/book/book.css
```
  
We are looking for the most commited static files (.js, .html, .css), not beginning by *core/* (Drupal 8 hierarchy, we are looking for a Drupal 7 version here) and saving the results in */tmp/relevant_files.txt*.  
  
Let's download these files from our target in order to be processed by the **versionchecker.sh**:  
```bash
mkdir ../input && cd ../input
for file in $(cat /tmp/relevant_files.txt | sed -r 's/^ +//g' | cut -d' ' -f2);do mkdir -p $(dirname $file) 2>/dev/null;wget -O $file "http://localhost/drupal/$file";done
```
  
In the input directory we need to keep the hierarchy of relevant files in order to find them in the GIT repo. Some files may not be found for the target release. As an option, you can pass a pattern to grep the tags you want after the script has executed the ```git tag``` command (to find all the releases).  
  
Let's run our **versionchecker.sh** (currently only working for git versioning system):  
```bash
cd .. && ./versionchecker.sh -s ./input/ -g ./drupal/ -p "^[78]\.[0-9.]+$"
[*] Input files directory: /tmp/input
[*] Cleaning empty files and directory in /tmp/input - Done
[*] GIT repository: /root/Documents/repo/drupal
[*] Grep pattern: ^7\.[0-9]+$
        ___           ___           ___           ___                       ___           ___       
       /\__\         /\  \         /\  \         /\  \          ___        /\  \         /\__\      
      /:/  /        /::\  \       /::\  \       /::\  \        /\  \      /::\  \       /::|  |     
     /:/  /        /:/\:\  \     /:/\:\  \     /:/\ \  \       \:\  \    /:/\:\  \     /:|:|  |     
    /:/__/  ___   /::\~\:\  \   /::\~\:\  \   _\:\~\ \  \      /::\__\  /:/  \:\  \   /:/|:|  |__   
    |:|  | /\__\ /:/\:\ \:\__\ /:/\:\ \:\__\ /\ \:\ \ \__\  __/:/\/__/ /:/__/ \:\__\ /:/ |:| /\__\  
    |:|  |/:/  / \:\~\:\ \/__/ \/_|::\/:/  / \:\ \:\ \/__/ /\/:/  /    \:\  \ /:/  / \/__|:|/:/  /  
    |:|__/:/  /   \:\ \:\__\      |:|::/  /   \:\ \:\__\   \::/__/      \:\  /:/  /      |:/:/  /   
     \::::/__/     \:\ \/__/      |:|\/__/     \:\/:/  /    \:\__\       \:\/:/  /       |::/  /    
      ~~~~          \:\__\        |:|  |        \::/  /      \/__/        \::/  /        /:/  /     
                     \/__/         \|__|         \/__/                     \/__/         \/__/      
        ___           ___           ___           ___           ___           ___           ___     
       /\  \         /\__\         /\  \         /\  \         /\__\         /\  \         /\  \    
      /::\  \       /:/  /        /::\  \       /::\  \       /:/  /        /::\  \       /::\  \   
     /:/\:\  \     /:/__/        /:/\:\  \     /:/\:\  \     /:/__/        /:/\:\  \     /:/\:\  \  
    /:/  \:\  \   /::\  \ ___   /::\~\:\  \   /:/  \:\  \   /::\__\____   /::\~\:\  \   /::\~\:\  \ 
   /:/__/ \:\__\ /:/\:\  /\__\ /:/\:\ \:\__\ /:/__/ \:\__\ /:/\:::::\__\ /:/\:\ \:\__\ /:/\:\ \:\__\
   \:\  \  \/__/ \/__\:\/:/  / \:\~\:\ \/__/ \:\  \  \/__/ \/_|:|~~|~    \:\~\:\ \/__/ \/_|::\/:/  /
    \:\  \            \::/  /   \:\ \:\__\    \:\  \          |:|  |      \:\ \:\__\      |:|::/  / 
     \:\  \           /:/  /     \:\ \/__/     \:\  \         |:|  |       \:\ \/__/      |:|\/__/  
      \:\__\         /:/  /       \:\__\        \:\__\        |:|  |        \:\__\        |:|  |    
       \/__/         \/__/         \/__/         \/__/         \|__|         \/__/         \|__|    


[*] Hint: for choosing relevant files to compare from a GIT repository:
[*] git log --all --pretty=format: --name-only | egrep -i "(.js$|.html$|.css$)" | sort | uniq -c | sort -rg | head -20

[*] /tmp/work/hashes.txt found. Are you sure you want to overwrite and compute hashes [Y/n]: 
[*] -----------------------
[*] Looking for tag: 7.0
[*] -----------------------
[*] Looking for tag: 7.1
[*] -----------------------
[*] Looking for tag: 7.2
[*] -----------------------
[*] Looking for tag: 7.3
...
[*] -----------------------
[*] Checking filename /tmp/input/modules/color/color.js: abce1b13050367ea1c8806888c29b383
[*] -----------------------
[*] Checking filename /tmp/input/modules/user/user.css: 1162bec186856e63a6ca207b04282816
[*] -----------------------
[*] Checking filename /tmp/input/modules/user/user.js: 0409afa4203df9e19e5754663bf27ba8
[*] -----------------------
[*] Checking filename /tmp/input/modules/overlay/overlay-parent.js: 9e7f29219143a79e528a59f1e5e2ab6e
[*] Thanks to strong and costly mathematical and statistical calculation:
[*] min version: 7.29
[*] max version: 7.32
[*] Number of input files: 23
[*] All input files have been found in the following versions: 
7.32
7.31
7.30
7.29
```
  
The script is computing hashes for every input files found and for every git tags. If one input file is not found, the script will fail because we can not precisely determine a version that matches all input files.  
  
In our example, we finally have 23 input files from 30 relevant files (some have not been found on our target). The final comparison between the input files hashes and the git tags hashes shows that the whole 23 input hashes match the versions 7.32, 7.31, 7.30, **7.29**.  
  
So we found the real version in our script suggestions (7.29).  
You also can try with some others CMS like Wordpress. Here is what we got for example with a **Wordpress 4.6**:  
```bash
./versionchecker.sh -s ./input/ -g ~/Documents/repo/WordPress/ -p "^4(\.[0-9])+$"
[*] Input files directory: /tmp/input
[*] Cleaning empty files and directory in /tmp/input - Done
[*] GIT repository: /root/Documents/repo/WordPress
[*] Grep pattern: ^4(\.[0-9])+$
...
[*] min version: 4.6
[*] max version: 4.6
[*] Number of input files: 24
[*] All input files have been found in the following versions: 
4.6
```
  
For wordpress we have only one possible version matching all the input files we provided thanks to the same procedure as seen for the Drupal CMS. We clearly know right now which version the CMS is running, **4.6**.  
  
Hope it may helps you, give you some ideas, make you think about how to reduce your fingerprinting entropy.  
  
Cheers,  
Phackt.
