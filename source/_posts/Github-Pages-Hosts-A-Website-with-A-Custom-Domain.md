---
title: Github Pages Hosts A Website with A Custom Domain
date: 2022-06-10 17:26:45
Categories: [IT, Web]
tags: [Github Page, web hosting, domain, DNS]
cover: /img/customDomain_githubpage_hosting.jpg
thumbnail: /img/githubpages.png
---

From previous posting, **"Online Document/Resume Deployment on Github Pges with Docsify"**, we discussed the way to deploy static web pages and online documents onto `Github Pages` so that we can save our cost on web hosting, but what if we want to use custom domain like `usernameresume.net` or `resume.username.net` rather than `username.github.io` or `username.github.io/pagename`? The answer is we can because github provides that functionality to host website by using custom domain, all we need to do is simply 3 steps

1. buy a custom domain from vendor
2. config domain DNS setting
3. config a custom domain on your Github Pages site

Now, let's dive-in the details of those steps in order to host your website with your favorite domain on github.

<!-- more -->

The first thing first, we need to buy a custom domain, there are a lot of choices from hosting service vendor like `goDaddy`, `bluehost`, `name cheap`, this time we use `Google Domains`. The good thing is google provides a very simple and intuitive user interface to be able to finish the entire purchase by just several clicks as long as you have a google account. 

[![google_domain.md.png](https://o.130014.xyz/2022/06/14/google_domain.md.png)](https://www.wailian.work/image/Q4QP2k)

after we buy a domain, we are redirected to `My Domain` page and DNS setting is now enabled so that we can add our subdomain record, for example, we want to host resume.username.net on Github Pages. 

click `manage custom records` 

![manage_custom_records.png](https://o.130014.xyz/2022/06/14/manage_custom_records.png)

fill in subdomain name in `Host name`, ***CNAME*** IN `Type`, keep default value on `TTL`, `Data` field should be Github Pages name **hermanteng19.github.io**,

![custom_records.png](https://o.130014.xyz/2022/06/14/custom_records.png)

if you have more than one subdomain hosting on Github Pages, click **Create new record** to add it into custom records.

The last thing we do is Github Pages configuration. Login our Github repository which contains website files, go to `Setting` page

![github_setting.png](https://o.130014.xyz/2022/06/14/github_setting.png)

then click `Pages` on the left side menu bar, fill in our subdomain name recorded on DNS setting, then click Save. Github Page will check DNS, download TLS/SSL certificates about 5-10 minutes, after all things are ready then we can visit our website in our favorite domain name

![custom_domain_GithubPages.png](https://o.130014.xyz/2022/06/14/custom_domain_GithubPages.png)

![github_pages_host_site.png](https://o.130014.xyz/2022/06/14/github_pages_host_site.png)

click our custom domain URL the we can visit our web pages.

![resume_custom_domain.png](https://o.130014.xyz/2022/06/14/resume_custom_domain.png)

