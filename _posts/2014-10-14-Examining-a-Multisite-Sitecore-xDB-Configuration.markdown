---
layout: post
title:  "Examining a Multisite Sitecore xDB Configuration"
date:   2014-10-14 10:18:00
categories: Sitecore xDB
---

With the introduction of The Experience Database (xDB) in Sitecore 7.5, MongoDB hosts the primary repository of web activity across Sitecore backed websites. Web visitors, now known as contacts, are captured along with each page view (Interactions in xDB) generated in a given browsing session. Much like its predecessor, DMS, the new xDB separates web activity by site.

When building upon xDB in a multi-site implementation, being aware of how Sitecore captures and processes this information is essential for a successful multisite configuration.

## A Quick Look at Sitecore 7.5 with xDB

### Contact Creation

Contacts are identified just as they were in Sitecore DMS.

A cookie, <em>SC_ANALYTICS_GLOBAL_COOKIE</em>, is created with a Guid uniquely identifying the contact. From there, a contact record is created. This contact will be referenced for the lifetime of the cookie as interactions are recorded against the contact.

### Interactions

Site activity is captured as documents within Interactions. High level data points of Interactions include:

* Site Name
* Pages viewed with URL, item Guid
* Visit page count total
* Browser type
* Screen resolution
* Geo location data

While the structure of the data differs from DMS, as we're now storing data as documents, commonality exists between the data points captured in DMS vs the new xDB structure.

### Contact Merging

The main takeaway with xDB is Sitecore's ability to find matching contacts and merge those contacts given a predefined value uniquely identifying visitors. In previous versions, visitors identified by their Global Session cookie were maintained as unique visitor records with DMS. Upon processing of analytics data during the `SessionEnd` event in xDB, contacts are merged, creating a single consolidated view of the customer.

Contact merging is useful in two specific areas:

* Multiple browsing sessions for a single site - Such as sharing a shopping cart between a session on a PC and transferring that session to a mobile device. For more information on this approach, See [Nick Wesselman's series of in-depth Sitecore 7.5 posts](http://www.techphoria414.com/Blog/2014/June/One_Month_with_Sitecore_7-5_Part_1).
* Multisite Sitecore Implementation - Two sites sharing the same membership model, both uniquely identifying contacts in the same way.

## A Multisite Example

Suppose we have a multisite implementation, Launch Sitecore and Launch SkyNet (our fictitious Launch Sitecore evil twin). Both sites follow Sitecore's best practice recommendations for configuration in IIS, while also sharing MongoDB, Collection and Reporting databases.

For the purpose of this example, while the two sites share membership, Single Sign-On is not implemented, requiring the user to identify themselves on both sites. Having such a setup will show how the xDB implementation handles contact merging and the importance of a common contact identification strategy shared across all sites.

![multisite](/assets/images/multisite.png)

### Browsing Session #1: Launch Sitecore

Browsing the site for the first time results in the creation of a Global Analytics cookie. If you're familiar with DMS, this works in the same way as previous versions. The cookie is what xDB will use to tie contacts together for unique browsing sessions.

![session 1](/assets/images/session1.png)

While browsing Launch Sitecore, suppose we login, recognizing the current browsing session as a single customer within our membership model. At the point in which the user is identified, the contact, who previously was anonymous is now labelled using the unique identifier. In this example, we're using the username from the extranet domain.

![session 1 details](/assets/images/session1-details.png)

Notice how the previous page views (xDB interactions) are now tagged with the contact id of the logged in user. Launch Sitecore programmatically identifying the contact is below. The line of code we're most interested in is Tracker.Current.Session.Identify(domainUser).

{% highlight c# %}
string name = Sitecore.Context.User.Profile.FullName;
if (name == String.Empty) name = Sitecore.Context.User.LocalName;
Tracker.Current.Contact.Tags.Add("Username", domainUser);
Tracker.Current.Contact.Tags.Add("Full name", name);</code>

Tracker.Current.Contact.Identifiers.AuthenticationLevel = AuthenticationLevel.PasswordValidated;
Tracker.Current.Session.Identify(domainUser);
{% endhighlight %}

### Browsing Session #2: Launch SkyNet

Upon browsing Launch SkyNet, we have a completely different Global Session Guid in our cookie.

![session 2](/assets/images/session2.png)

To Launch SkyNet, we're anonymous and in no way connected to the user identified in Launch Sitecore. As soon as we login on Launch SkyNet, using the same logic to uniquely identify the contact (extranet domain username), Sitecore will flush the contact to xDB, updating the interactions with the contact id of the recognized user.

Any updates to facets, tags, counters, automation states, and contact attributes will be auto-merged within the MergeContacts pipeline processors.

**Key takeaway:** Contact consolidation occurs at the point in which the current tracking session is identified via `Tracker.Current.Session.Identify()`.

![session 2 details](/assets/images/session2-details.png)

## Summary

Regardless of how many sites you have running through a single instance of Sitecore, xDB processing and contact merging will consolidate contacts while maintaining the page interactions of each site. It is through this process we're able to maintain a single view of the customer and maintain the customer lifetime value as seen by the Sitecore experience database.

Risks of splitting separate Sitecore instances to separate instances of xDB processing will result in a partial view of the customer and their relative engagement value for each site instance.