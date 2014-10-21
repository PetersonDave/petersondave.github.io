---
layout: post
title:  "Building a Single Page Application with AngularJS and Sitecore: Part 2"
date:   2013-11-25 10:18:00
categories: Sitecore AngularJS
comments: true
---

In the second and final in a series of posts related to AngularJS and Sitecore, we'll explore Angular in a Sitecore context. By running Angular within a Sitecore instance, we're pairing the speed of Angular with dynamic rendering of Sitecore, opening our discussion to multiple interesting scenarios.

## SPA Integration with Sitecore

Suppose we want more than just Sitecore content in our SPA, we also want Sitecore renderings. Our previous example ran as a SPA with a direct line to Sitecore by way of the Sitecore Item Web API. Building on the previous example, lets now have the start page of our SPA as the home item of a new Sitecore site within our instance. We'll then define content which lets us:

* Use renderings for content outside of our views (static content for all pages across our SPA).
* Use Sitecore items as views in our Angular routing.

![angular integrated](/assets/images/angular-integrated.jpg)

<em>Note: Since we're building off the previous example, we're still using the Sitecore Item Web API. This could easily be changed to other methods, such as directly serializing objects to JSON.</em>

Looking at Sitecore, this is our configuration:

![tree](/assets/images/tree.jpg)

* Callouts - Ads displayed below our Angular views (labeled as "Heading Callout" above).
* Profiles - Data returned by our Web API calls.
* Views - It's here where we'll be swapping out our Angular views for Sitecore items.

Looking at the presentation details of our views, you'll notice they don't have the default layout used by our SPA, as this would cause a nested default layout effect within our application. Instead, we define a "PartialView". 

![partial view](/assets/images/partial-view.jpg)

The ParialView layout paired with an allprofiles rendering essentially makes this the equivalent of the allprofiles view in our previous example. The difference, however, is now our views are powered by Sitecore. We can insert other renderings, content, etc.

**PartialView Layout:**

{% highlight html %}
<%@ Page language="c#" Codepage="65001" AutoEventWireup="true" %>
<%@ OutputCache Location="None" VaryByParam="none" %>

<sc:FieldRenderer ID="FieldRenderer1" runat="server" FieldName="details" />
<sc:placeholder key="main" runat="server" /> 
{% endhighlight %}

**allprofiles rendering:**

{% highlight html %}
<%@ Control Language="c#" AutoEventWireup="true" TargetSchema="http://schemas.microsoft.com/intellisense/ie5" %>
<div data-ng-controller="allprofiles as vm">
    <p><a class="btn btn-primary btn-lg" ng-click="vm.load();">Load Data</a></p>

    <div class="row" ng-repeat="profile in vm.repository">
        <div class=" col-md-1">
            <a class="btn btn-danger btn-small" href="#profiles/{{profile.ID}}">Edit</a>
        </div>
        <div class="col-md-1">{{profile.DisplayName}}</div>
        <div class="col-md-6">{{profile.ID}}</div>
    </div>
</div>
{% endhighlight %}

The only change required is to wire everything up. We can do this rather easily by altering our routes to now point at our Sitecore views instead of html files on the file system. Since we're in the context of a Sitecore instance, Sitecore properly handles the request serving only the content for the views requested.

{% highlight javascript %}
    function getRoutes() {
        return [
        {
            url: '/profiles',
            config: {
                templateUrl: 'Views/allprofiles',
                controller: 'allprofiles'
            }
        }, {
            url: '/profiles/:profileId',
            config: {
                templateUrl: 'Views/modifyprofile',
                controller: 'modifyprofile'
            }
        }, {
            url: '/',
            config: {
                templateUrl: 'Views/main',
                controller: 'main'
            }
        }];
    }
{% endhighlight %}
	
## Technical Considerations

### Caching

Angular is fast because only specific areas of the DOM are manipulated, but we have to be aware of caching issues. Any personalized areas of content or special rendering rules may very well be ignored. 

When publishing updates to our views, we want to be sure we're getting the latest and greatest version. We can force the application to reload views by appending the item version number as a query string parameter to our view. This will force a refresh through our routing.

{% highlight javascript %}
    function getRoutes() {
        return [
        {
            url: '/profiles',
            config: {
                templateUrl: 'Views/allprofiles?v=4',
                controller: 'allprofiles'
            }
        }, {
            url: '/profiles/:profileId',
            config: {
                templateUrl: 'Views/modifyprofile?v=6',
                controller: 'modifyprofile'
            }
        }, {
            url: '/',
            config: {
                templateUrl: 'Views/main?v=2',
                controller: 'main'
            }
        }];
    }
{% endhighlight %}
	
### DMS/Tracking

Out of the box configuration, DMS will track with every page request. Since our SPA remains on the same page for each request, we're not going to be actively tracking each action by the user. Of course, we could implement a solution where we're firing client-side events to our DMS solution.

### Page Editor

Simply won't work. Manipulating content or rendering placement within the SPA is not possible through the page editor.

### Workflows

Tests using basic workflows failed on HTML validation. The extra Angular directives caused failures upon attempting to submit for approval.

### Views in Sitecore Content

Our views are publicly accessible with a PartialView layout. We want to be sure these don't make their way out to be indexed, as they don't have a full page layout associated with them. We want to be sure these are not crawled and no pages link back to these outside the context of our SPA.

## Just Because You Can, Doesn't Mean You Should

While we all love to push the boundaries of what Sitecore can offer and  building solutions while incorporating other technologies, it's important to be aware of the pros versus the cons. Building a SPA within the context of a Sitecore instance, while an interesting discussion, may bring about more issues than problems its solving. 

For me at least, if someone were to suggest a similar approach, it would have to be quite convincing. In almost all cases, all we're going to need is access to Sitecore content. If all you need is content, you may want to take a different route (no pun intended). 

## Alternate Approach: Angular Views in Sitecore

If all we're after is Sitecore renderings used in conjunction with Angular, why not simply use Angular views? Why add the complexity of a SPA, if we can achieve our goal by building dynamic forms with Angular. Using Angular views, without Angular routing, we now regain control over the issues listed above under Technical Considerations.

## Conclusion

Walking through each of these posts and scenarios using Angular in a Sitecore context was fun. I encourage anyone interested in using Angular to take into account all points covered in both posts. Hopefully these experiments assist the community and other developers looking to leverage Angular in future projects.

For the complete solution, including routing, Sitecore field mappings and all views and controllers, go here:

[https://github.com/PetersonDave/SinglePageAppDemo/tree/with-sitecore-integrated](https://github.com/PetersonDave/SinglePageAppDemo/tree/with-sitecore-integrated)