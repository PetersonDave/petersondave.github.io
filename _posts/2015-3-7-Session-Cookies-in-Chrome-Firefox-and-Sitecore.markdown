---
layout: post
title:  "Session Cookies in Chrome, Firefox, and Sitecore"
date:   2015-03-07 00:20:00
categories: Cookies
comments: true
---

Handling of session cookies differs between browsers. The focus of this post details common misconceptions of session cookie management and its impact to how your web application operates for any given browser.

##Types of Cookies

There are two main categories of cookie types:

* *Persistent Cookies* - Cookies which are carried or persisted across multiple browsing sessions. These cookies will expire on a given date and time.
* *Session Cookies* - Also known as a _transient cookie_ or _in-memory cookie_. The lifetime of session cookies remain for the length of the browsing session. Once you close your browser, session cookies are cleared. 
	
Session cookies aim to solve the problem of a temporary data store for a given browsing session, which are automatically cleaned once that browsing session has ended. Examples of where session cookies are most likely used include storing of shopping cart items, form data or theme selections, temporary tracking data, etc. While session cookies seem to be a safe solution, it's important to understand how handling of session cookies differs between browsers. 

##Chrome

###Background Apps

![background-apps](/assets/images/chrome-background.png)

Since version 19, Chrome has altered how it runs in the background which has an immediate impact on how you expect Chrome to handle session cookies when you close your browser. Under _advanced settings > System_, the option "Continue running background apps when Google Chrome is closed" is checked by default. In other words, if you close your browser, it will continue to run in the background (to support Chrome applications and extensions). Allowing Google Chrome to run in the background keeps the Chrome application session alive and prevents session cookies from being cleared. 

The issue has been entered and marked as "won't fix", recognized as <a href="https://code.google.com/p/chromium/issues/detail?id=128513" target="_blank">expected behavior by the development team</a>.

###Start-Up

![start-up](/assets/images/chrome-startup.png)

On start-up, Chrome offers the ability to "Continue where you left off". Checking this setting persists session cookies from one browsing session to the next. This makes sense. With this setting, we're explicitly telling the browser to remember exactly what we were doing at the time the browser was closed. By default, this is turned off. 

##Firefox 

Similar to Chrome's start-up feature, Firefox Session cookies are also saved to allow for Firefox's session restore feature. If the browser is forcibly closed or crashes, session cookies are not deleted and the session remains. It's worth noting, this does not happen on sites backed by https. While this is default behavior, unlike Chrome, closing the browser will clear any session cookies present.

This feature has also created a <a href="https://bugzilla.mozilla.org/buglist.cgi?bug_id=337551,345830,358042,362212,369289,375182,376605,377233,381940,395749,398827,399748,417711,431547,437911,441544,576845" target="_blank">lively discussion on Mozilla's bug tracker</a>.

##Sitecore xDb

How does Sitecore handle session cookies?

Looking at Sitecore DMS, you'll notice the cookie representing the current browsing session for Analytics is a session cookie (SC_ANALYTICS_SESSION). For xDB, the dependency on this cookie has been removed. Session management through both the ```SessionEnd``` and ```VisitEnd``` pipelines invoke ```Sitecore.Analytics.Tracker.EndVisit``` flushing the current session and queueing writes to the Analytics database, removing any potential for browser-specific issues around handling of session cookies.

##Final Thoughts

Don't rely solely on browser behavior for proper clean-up of session cookies during a given browsing session. If you must rely on session cookies, look to server-side logic for help in determining when a given session ends. It's also probably safer to explicitly define an expiration date for your cookie than to rely on consistent behavior across all browsers and versions.