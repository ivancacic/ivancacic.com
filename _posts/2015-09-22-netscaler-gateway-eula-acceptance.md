---
id: 178
title: 'NetScaler Gateway - Automatic EULA Acceptance'
date: 2015-09-22T19:03:58+00:00
author: Ivan Cacic
layout: post
comments: true
guid: https://ivancacic.com/?p=178
permalink: /netscaler-gateway-eula-acceptance
categories:
  - Citrix
  - NetScaler
tags:
  - Cookie
  - EULA
  - Rewrite
---
Citrix has recently released NetScaler firmware version 11 which brings us the Portal Themes feature. This in turn makes customisation a lot easier, particularly as we upgrade the firmware version on the appliance - no more custom themes ([kind of](http://discussions.citrix.com/topic/366451-aaa-vserver-green-bubble-theme/?p=1881908), there is a 'fix' [here](http://discussions.citrix.com/topic/367441-netscaler-11-gateway-uithemeportal-theme-bug/?p=1886197)).

As part of the Portal Themes page, Citrix now allows us to prove and End User License Agreement to our users on the page which looks something like this (with the X1 theme):

[![X1 Theme]({{ site.baseurl }}/assets/2015/09/2015-09-22-19_07_06-NetScaler-Gateway.png)]({{ site.baseurl }}/assets/2015/09/2015-09-22-19_07_06-NetScaler-Gateway.png)

The only issue with the above is the fact that our users have to select the checkbox each any every time before they can use the _Log On_ button, pair this with multifactor authentication and you start to run into some pretty annoyed users. So the solution? Well we could use the custom theme method from yesteryear to check the box, but a more fluid and upgrade persistent method (hopefully) is to use a re-write to force the check box ticked.

Using the great developer tools in Google Chrome (F12 or Ctrl+I) and WinSCP (to inspect the index.html file) I was able to find out how the log on form is generated, and what we need to target to change the code so the checkbox is ticked automatically. Firstly, this is how the checkbox is coded:

[![eula_check]({{ site.baseurl }}/assets/2015/09/2015-09-22-23_17_15-NetScaler-Gateway.png)]({{ site.baseurl }}/assets/2015/09/2015-09-22-23_17_15-NetScaler-Gateway.png)

So we know we are targeting the _eula_check_ name and that we need to add the _checked_ attribute to the element. The Sources tab is an easy way to browse the JavaScript, we can see the `gateway_login_form_view.js` file is generating the checkbox:

[![Checkbox generation]({{ site.baseurl }}/assets/2015/09/2015-09-22-23_19_31-NetScaler-Gateway1.png)]({{ site.baseurl }}/assets/2015/09/2015-09-22-23_19_31-NetScaler-Gateway1.png)

Great, so this is our first rewrite, the action will be to add the _checked_ attribute to the _type='checkbox'_ section, this can be done with a rewrite such as this:

`add rewrite action RW_ACT_EulaChecked replace_all "http.RES.BODY(120000).SET_TEXT_MODE(ignorecase)" "\"type=\'checkbox\' checked\"" -pattern "type=\'checkbox\'"`

[![Configure Rewrite Action]({{ site.baseurl }}/assets/2015/09/2015-09-22-23_25_49-Citrix-NetScaler-VPX-Configuration.png)

We need to bind this action to a policy which acts on the `gateway_login_form_view.js` object:

`add rewrite policy RW_POL_EulaChecked "HTTP.REQ.URL.CONTAINS(\"gateway_login_form_view.js\")" RW_ACT_EulaChecked`

Lastly let's bind this Rewrite Response policy to our NetScaler Gateway:

`bind vpn vserver citrix.ivancacic.com -policy RW_POL_EulaChecked -priority 100 -gotoPriorityExpression NEXT -type RESPONSE`

The above rewrite should result in the checkbox being ticked for all users visiting the site, however this won't set the _Log On_ button into an enabled state, this is because the rewrite occurs after the JavaScript has loaded, which means at the time of 'testing' the checkbox was unticked and thus the _Log On_ button remains disabled. Just like before the Log On disabled section is part of the `gateway_login_form_view.js` file.

[![Log On disabled]({{ site.baseurl }}/assets/2015/09/2015-09-22-23_36_42-NetScaler-Gateway.png)]({{ site.baseurl }}/assets/2015/09/2015-09-22-23_36_42-NetScaler-Gateway.png)

The following rewrite action and policy will enable the _Log On_ button by removing the _disabled_ element:

`add rewrite action RW_ACT_LogonAutoEnable replace_all "http.RES.BODY(120000).SET_TEXT_MODE(ignorecase)" "\"\'disabled\':\'\'\"" -pattern "\'disabled\':\'disabled\'"`

`add rewrite policy RW_POL_LogonAutoEnable "HTTP.REQ.URL.CONTAINS(\"gateway_login_form_view.js\")" RW_ACT_LogonAutoEnable`

`bind vpn vserver citrix.ivancacic.com -policy RW_POL_LogonAutoEnable -priority 110 -gotoPriorityExpression NEXT -type RESPONSE`

That's about it, we should now have all our users defaulting to a ticked EULA and enabled _Log On_ button. If you are having issues with it remember to clear the _loginstaticobjects_ Content Group from within the Integrated Caching feature, this is applied even if Integrated Caching is not enabled: 

[![Content Group invalidation]({{ site.baseurl }}/assets/2015/09/2015-09-22-23_48_14-Clipboard.png)]({{ site.baseurl }}/assets/2015/09/2015-09-22-23_48_14-Clipboard.png)

## Extra Points

The ideal method to treat the above situation would be to use a 'remember selection' option, what I mean by that is once the user accepts the EULA don't make them tick the checkbox again, with the option to clear their selection if the EULA changes. I attempted to get this working by setting a cookie as per below:

`add rewrite action RW_ACT_EULAAcceptCookie insert_http_header Set-Cookie "\"EULAAccepted=true\" + \"; expires=\" + SYS.TIME.ADD(31536000).TYPECAST_TIME_AT + \"; path=/\""`

In the above action I'm setting a cookie for 365 days from the system time for the root of the sub domain.

`add rewrite policy RW_POL_EULAAcceptCookie "HTTP.REQ.URL.SET_TEXT_MODE(IGNORECASE).CONTAINS(\"/Citrix/\") || HTTP.REQ.URL.SET_TEXT_MODE(IGNORECASE).CONTAINS(\"/vpns/portal/\")" RW_ACT_EULAAcceptCookie`

The above policy is triggering the action if we hit the `/vpns/portal/` site (Unified Gateway) or the `/Citrix/` URL (StoreFront).

The above works well, and only creates the cookie once the users has logged in, however I didn't manage to modify my original rewrites to trigger on the `gateway_login_form_view.js` page when the cookie exists. For reference I used [CTX123676](http://support.citrix.com/article/CTX123676) as a starting point, if you have any ideas on how to get it going, please share in the comments below!

---

September, 2016 - Mike Roselli has managed to work out the cookie component, more info [here](https://discussions.citrix.com/topic/381209-automatic-eula-acceptance-by-cookie-rewrite-guide/), great work Mike!