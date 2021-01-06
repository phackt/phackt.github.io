---
layout: post
title:  "What solutions to prevent git leaks ?"
date:   2020-05-27
category: Dev
excerpt_separator: <!--more-->
---
Hello,  
  
I will do a quick and dirty post about what's out there to find / prevent leaks of secrets in your git repositories.  
I did not try all of these tools. For the search part, i'm mainly using a fork of Trufflehog with some added features (search in filenames, commits comments, also with custom regexes).  
  
## Objectives :  
  
 - Look into the commits history for sensitive information publicly accessible by an attacker ;
 - Prevent secrets leaks ;
 - Monitoring and integrating these checks in the Continous Delivery process - aka DevSecOps
<!--more-->
  
## Tools to look for sensitive data :
<div>
  <span class="fa fa-arrow-circle-o-right fa-2x" style=" vertical-align: middle;"></span>
  <span style="margin-left: 5px;"><a href="https://github.com/dxa4481/truffleHog">TRUFFLEHOG</a></span>
</div>
   - Written in Python
   - Also works for local repo
   - Strings with high entropy
   - Custom regex rules
   - No search in commits comments or filenames  
  
<div>
  <span class="fa fa-arrow-circle-o-right fa-2x" style=" vertical-align: middle;"></span>
  <span style="margin-left: 5px;"><a href="http://michenriksen.com/blog/gitrob-putting-the-open-source-in-osint/">GITROB</a></span>
</div>
   - Written in Go
   - Renders results in a Web App
   - Do not work for local repo AFAIK
  
<div>
  <span class="fa fa-arrow-circle-o-right fa-2x" style=" vertical-align: middle;"></span>
  <span style="margin-left: 5px;"><a href="https://github.com/auth0/repo-supervisor">REPO-SUPERVISOR</a></span>
</div>
   - Written in NodeJS
   - Generates HTML reports

<div>
  <span class="fa fa-arrow-circle-o-right fa-2x" style=" vertical-align: middle;"></span>
  <span style="margin-left: 5px;"><a href="https://github.com/anshumanbh/git-all-secrets">GIT-ALL-SECRETS</a></span>
</div>
   - Written in Go
   - Using / merging results of [Trufflehog](https://github.com/dxa4481/truffleHog) and [Repo-Supervisor](https://github.com/dxa4481/truffleHog)

<div>
  <span class="fa fa-arrow-circle-o-right fa-2x" style=" vertical-align: middle;"></span>
  <span style="margin-left: 5px;"><a href="https://github.com/zricethezav/gitleaks">GITLEAKS</a></span>
</div>
   - Written in Go
   - regex and entropy

<div>
  <span class="fa fa-arrow-circle-o-right fa-2x" style=" vertical-align: middle;"></span>
  <span style="margin-left: 5px;"><a href="https://github.com/eth0izzle/shhgit">SSHGIT</a></span>
</div>
   - Written in Go
   - Renders results in a Web App
   - Monitors GIT repositories - *Be the first to catch the secret before it gets deleted from git history*
   - *shhgit will watch real-time stream and pull out any accidentally committed secrets*
    
## Prevent commits :

What is a Git Hook ? As described [here](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks):  

*Like many other Version Control Systems, Git has a way to fire off custom scripts when certain important actions occur. There are two groups of these hooks: client-side and server-side. Client-side hooks are triggered by operations such as committing and merging, while server-side hooks run on network operations such as receiving pushed commits. You can use these hooks for all sorts of reasons.*
  
Security is a good one.  
  
What kind of hooks can used to prevent leak at an early stage of the git workflow? :  
```
pre-commit: Used to check if any of the files changed in the commit use prohibited patterns.

commit-msg: Used to determine if a commit message contains a prohibited patterns.

prepare-commit-msg: Used to determine if a merge commit will introduce a history that contains a prohibited pattern at any point. Please note that this hook is only invoked for non fast-forward merges.
```  

<u>Client-side hooks :</u>  

<div>
  <span class="fa fa-arrow-circle-o-right fa-2x" style=" vertical-align: middle;"></span>
  <span style="margin-left: 5px;"><a href="https://github.com/ezekg/git-hound">GITHOUND</a></span>
</div>
  - You have to set GitHound command into a *pre-commit* hook
  - Regexes are configured in ```.githound.yml```  

<div>
  <span class="fa fa-arrow-circle-o-right fa-2x" style=" vertical-align: middle;"></span>
  <span style="margin-left: 5px;"><a href="https://github.com/awslabs/git-secrets">GIT-SECRETS</a></span>
</div>
  - From **AWS** team
  - You also can manually scan for secrets before making your repo public: ```git secrets --scan-history```
  
<u>GITHUB actions :</u>  

*[GitHub Actions](https://help.github.com/en/actions/getting-started-with-github-actions/about-github-actions) enables you to create custom software development life cycle (SDLC) workflows directly in your GitHub repository.*  
  
 - [Gitleaks Github Action](https://github.com/marketplace/actions/gitleaks)
 - [Trufflehog Github Action](https://github.com/marketplace/actions/trufflehog-actions-scan)


<u>The GITHUB.COM scanning project :</u>  

Github has a project which aims at monitoring leaked third parties tokens from your repositories: [https://help.github.com/en/github/administering-a-repository/about-secret-scanning](https://help.github.com/en/github/administering-a-repository/about-secret-scanning).  

Once identified, Github will warn you and will request the provider (from the following list) in an automated way to ask for the immediate revokation of your leaked tokens:  

```
Adafruit
Alibaba Cloud
Amazon Web Services (AWS)
Atlassian
Azure
Clojars
CloudBees CodeShip
Databricks
Datadog
Discord
Doppler
Dropbox
Dynatrace
Finicity
Frame.io
GitHub
GoCardless
Google Cloud
Hashicorp Terraform
Hubspot
Mailchimp
Mailgun
MessageBird
npm
NuGet
Palantir
Plivo
Postman
Proctorio
Pulumi
Samsara
Shopify
Slack
SSLMate
Stripe
Tencent Cloud
Twilio
```
  
### Finally what to conclude :
  
For each leaked secrets linked to an environment which could be targeted by an adversary :  
 - Revoke / update the secret (password, any token / private key, ...) as quick as possible ;
 - [Update the git commits history](https://help.github.com/en/github/authenticating-to-github/removing-sensitive-data-from-a-repository)  
  
More globally :  
 - Warn the stakeholders involved in this data leak (users, providers, ...) ;  
 - Check log files to detect former fraudulent access ;
 - Prevent the versioning of sensitive data (via .gitignore and set hooks to monitor your commits) ;  
 - Include the secret scanning process as part as your continuous delivery process (build factory). For example for Github CI see the Github actions Trufflehog and Gitleaks as aforementioned. 
  
  
See you soon, next posts will be more internal pentesting / windows oriented,  
Cheers.
