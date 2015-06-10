---
layout: post
title:  "Sitecore 8 at a Glance"
date:   2014-10-1 10:18:00
categories: Sitecore Sitecore8
comments: true
---

With the release of the Sitecore 8 MVP Technical Preview, many of the features showcased during Sitecore Symposium 2014 were made available for review. The focus of this post is to detail high-level changes between Sitecore 7.5 and the technical preview of Sitecore 8.

Many new components were delivered in the new version of Sitecore, as well as, renaming of existing features and packaging of popular modules used in pre-Sitecore 8 versions.

### Renamed Components

* Page Editor = Experience Editor
* Marketing Center = Marketing Control Panel
* Email Campaign Manager = Email Experience Manager


### New Components

* Experience Analytics
* Experience Profile
* Experience Optimization
* List Manager
* Path Analyzer
* App Center
* Executive Dashboard

### New Features

* [Versioned Layouts](http://www.seanholmesby.com/presentation-details-changes-in-sitecore-8-how-renderings-are-stored/)
* Web API Services<em> (SPEAK components and building applications dependent upon Sitecore data)</em>

### Existing Features Packaged with Sitecore 8

* [Federated Experience Manager](https://www.connectivedx.com/thinking/posts/2014/09/orchestrating-connected-experiences-with-sitecore) <em>(available in pre Sitecore 8 versions)</em>
* Social <em>(previously Sitecore Social Connected)</em>

## General Look and Feel

The look and feel of the client is much improved. After the initial load, performance feels much quicker than Sitecore 7.5 and previous versions. The Launch Pad icon at the top of the page comes in very handy when wanting to switch between the new Sitecore 8 components and content editing, or experience editing; features that you're already used to using in pre-8 installations.

Moving through the various steps of editing content, publishing, workflow, etc. match that of previous versions -- just in a different layout. Everything is pretty much where you would expect it, but with a cleaner look and feel.

Obviously, with the addition of new Sitecore 8 features, you'll rely heavily on the Launch Pad to access these components.

## Sitecore Client Enhancements

### Experience Editor

The experience editor is accessible from the Sitecore Experience Platform, the start menu or directly within the content editor ribbon. Just as before, the only difference being the rename of <em>Page Editor</em> to <em>Experience Editor</em>.

### Experience Optimization

Testing is now exposed as <em>Optimization</em> in the experience editor section of the ribbon. Also accessible from the main Sitecore Experience Platform launch pad.

![sitecore8-1](/assets/images/sitecore8-1.png)

Content editors can establish tests, set goals and review performance reports.

![sitecore8-2](/assets/images/sitecore8-2.png)

Any tests created through Experience Optimization are saved under the Marketing Center Test Lab item buckets.

An example path for a home page test:

* Item: /sitecore/system/Marketing Center/Test Lab/2014/10/01/01/11/Home 20141001T011146Z
* Template: /sitecore/templates/System/Analytics/Testing/Test Definition

The Experience Optimization dashboard exposes reports enhancing new gamification concepts to the content editing and testing experience:

![sitecore8-3](/assets/images/sitecore8-3.png)

### Workflow

When moving content changes through Workflow, new "Approve with Test" and "Approve without Test" are available, consistently reminding content editors and decision makers to consider A/B testing at a content level.

![sitecore8-4](/assets/images/sitecore8-4.png)

### Email Experience Manager

The layout of the email campaign manager has changed, allowing for message creation and list importing available from the same menu

![sitecore8-5](/assets/images/sitecore8-5.png)

### List Manager

The List manager allows for creation of lists from files, or manual entry from an empty list

![sitecore8-6](/assets/images/sitecore8-6.png)

### Experience Analytics

New to Sitecore 8, the Experience Analytics feature brings together multivariate testing results, engagement scoring and overall site tracking statistics together in one location. The result is a powerful, new array of reports coupled with date range filtering and reporting facets.

![sitecore8-7](/assets/images/sitecore8-7.png)

### Path Analyzer

One of the new features I've had the most fun with so far is the Path Analyzer. Clicking on the map, zooming in and out, selecting successful and least effective paths is really quite fun. The example below is leveraging Analytics data collected from a Launch Sitecore instance:

![sitecore8-8](/assets/images/sitecore8-8.png)

Selecting a path by clicking on the map, leading from a mail campaign to a login page yields the results below. Clicking on an element within the funnel of a selected path visit shows the exit path from that particular page:

![sitecore8-9](/assets/images/sitecore8-9.png)

You can also select a path from the lists in the right navigation, narrowing down on a particular path of high importance. For instance, the selected path below is one of the most efficient full paths:

![sitecore8-10](/assets/images/sitecore8-10.png)

Showing the funnels of the select path:

![sitecore8-11](/assets/images/sitecore8-11.png)

### Social

Social is delivered out of the box with Sitecore 8. Previously, this required a separate installation of the Socially Connected module from SDN.

Available from the Page Editor, the "Messages" button under "Social" allowing content editors to create, edit and post a message on a target network. Take note of the new "Social" node in the content tree directly under the Sitecore root node.

![sitecore8-13](/assets/images/sitecore8-13.png)

### Pipeline Changes

While new pipeline processors have been added to accommodate the new Sitecore 8 features, others have been moved with Content Testing and Experience Editing in mind. Below are the main areas of change with regards to pipeline processing:

* Social related pipelines come configured out of the box.
* FXM related hooks
* Everything Analytics
* Moving of existing pipelines, such as:
  * SPEAK components
  * Experience Editor
  * Web API request handling (higher up in the httpRequestBegin pipeline)
  * RenderField (immediately after httpRequestBegin)

### Content Testing

Outside of Analytics, <em>Sitecore.ContentTesting.config</em> changes dominate the difference in Sitecore 8. Take a look at the config patch file. Content tests need to insert themselves in areas to override existing Sitecore rendering and processing handlers, such as:

* Insert Renderings
* RenderLayout
* GetChromeData
* Data Aggregation
* Database Agent (background processing to determine if a test has reached statistical relevancy)
* Content Testing provider (used by ItemManager)

The ItemManager's default item provider is now the content testing item provider, which is essentially a wrapper of the existing `Sitecore.Data.Managers.ItemProvider` overriding `GetItem()`. It is here where other versions/variations of items used via Content Testing are obtained for rendering.

For more information regarding ItemManager and providers, check out [this post](http://kamsar.net/index.php/2013/11/sitecore-data-architecture/) on overall Sitecore data architecture.

### Settings

One final interesting remark regarding configuration. You can now override the standard server time zone if needed via the <em>ServerTimeZone</em> setting.

### In Closing...

Sitecore 8 offers a much better editing experience. It's faster, cleaner and easier to work with. It'll be interesting to see the various modifications to content tests and how, as developers, we can leverage this data to build modules and further enhance the client.