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
  
### Objectives :  
  
 - Look into the commits history for sensitive information publicly accessible by an attacker ;
 - Prevent secrets leaks ;
 - Monitoring and integrating these checks in the Continous Delivery process - aka DevSecOps
<!--more-->
  
### Tools to look for sensitive data :

=> [TRUFFLEHOG](https://github.com/dxa4481/truffleHog)
   - Python
   - also works for local repo
   - strings with high entropy
   - custom regex rules
   - no search in commits comments or filenames  
  
=> [GITROB](http://michenriksen.com/blog/gitrob-putting-the-open-source-in-osint/)  
   - Go
   - Web App
   - do not work for local repo AFAIK
  
=> [REPO-SUPERVISOR](https://github.com/auth0/repo-supervisor)
   - NodeJS
   - generates HTML reports

=> [GIT-ALL-SECRETS](https://github.com/anshumanbh/git-all-secrets)
   - Go
   - using / merging results of [Trufflehog](https://github.com/dxa4481/truffleHog) and [Repo-Supervisor](https://github.com/dxa4481/truffleHog)

=> [SECRETS ANALYZER](https://gitlab.com/gitlab-org/security-products/analyzers/secrets)
   - Go
   - on GITLAB-ORG repo 
   - using / merging results of [Trufflehog](https://github.com/dxa4481/truffleHog) and [Gitleaks](https://github.com/zricethezav/gitleaks)

=> [GITLEAKS](https://github.com/zricethezav/gitleaks)
   - Go
   - regex and entropy

=> [SSHGIT](https://github.com/eth0izzle/shhgit)
   - Go
   - Web App
   - monitoring git repos - Be the first to catch the secret before it gets deleted from git history
   - *shhgit will watch real-time stream and pull out any accidentally committed secrets*
    
### Prevent commits :

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

=> [GITHOUND](https://github.com/ezekg/git-hound)
  - set githound command into a *pre-commit* hook
  - regexes configured in ```.githound.yml```  

=> [GIT-SECRETS](https://github.com/awslabs/git-secrets)
  - from AWS
  - you also can manually scan for secrets before making your repo public: ```git secrets --scan-history```
  
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
CloudBees CodeShip
Databricks
Datadog
Discord
Dropbox
Dynatrace
GitHub
GoCardless
Google Cloud
Hashicorp Terraform
Hubspot
Mailgun
npm
NuGet
Palantir
Postman
Proctorio
Pulumi
Samsara
Slack
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
