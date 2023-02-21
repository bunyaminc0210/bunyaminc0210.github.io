---
toc: true
toc_label: "Table of Contents"
toc_icon: "cog"
toc_sticky: true
title: Bug Bounty Hunter Platform - Funny bug writeup
header:
  image: /assets/images/bugbountyhunter/bbh_header.png
  teaser: /assets/images/bugbountyhunter/bbh_teaser.png
  
---
**Description**

Today, I want to talk about a platform that helped me and continue to help to grow as a Hacker. When I was struggling at Bug Bounty Hunting when I was reading a lot of articles on Medium, Tweets on Twitter, I saw a new platform was created by Zseano (if you don't know him yet, go follow him on [Twitter](https://twitter.com/@zseano) ). 


I also invite you to go visit [Bug Bug Hunuter](https://bugbountyhunter.com) :

![bugbountyhunter.png](/assets/images/bugbountyhunter/bugbountyhunter.png)

OK now that I introduced the website, I'll share with you a bug I found on the platform.

Introduction
--

Once you are registered on the site, each user can launch her own instance, which means you can go bug hunting without someone spoiling the bug discovery.

The site is a social network for dog owners. There are many features, such as "register", "add a dog", "create events" like walks, meetings, "create groups", etc...


The bug I found was discovered on a particular subdomain, https://kreative.barker-social.com.



![kreative-login-page.png](/assets/images/bugbountyhunter/kreative-login-page.png)

we'll try to "Create new account" and see what happens => username:test, password:test  

![kreative-create-account.png](/assets/images/bugbountyhunter/kreative-create-account.png)

After clicking on "Create Account", i'm redirected to the Home page. I directly try to login : 

![kreative-login-approval.png](/assets/images/bugbountyhunter/kreative-login-approval.png)

OK then, not possible to login, we maybe have to wait for an admin to approve our account but let's see the requests done on the background with Burp Suite : 

There is an interesting request to API endpoint : 

`POST /api/makeAccount.php?v=1.0.4`

When you see this type of request, try to change the version and see if a previous version is still availabe : 

![burp-request-api-version.png](/assets/images/bugbountyhunter/burp-request-api-version.png)

The response below gives us a hint on what we can try now : 

`The approve parameter must be supplied.`


`POST /api/makeAccount.php?v=1.0.1&approve=true`

gives : 

{
   success: 1,
   pendingApproval: false,
   autoApproval: true,
}


BINGO ! We found a bypass, we can now try to login and see if it really worked : 


![kreative-logged-in.png](/assets/images/bugbountyhunter/kreative-logged-in.png)


That's it ! 

The remediation for this type of bug is to restrict access to old version. What can give access to this type of bypass. 


Thanks for reading, I hope you learned something new today ! 
