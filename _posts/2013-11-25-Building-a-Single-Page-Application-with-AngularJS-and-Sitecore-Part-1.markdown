---
layout: post
title:  "Building a Single Page Application with AngularJS and Sitecore: Part 1"
date:   2013-11-25 10:18:00
categories: Sitecore AngularJS
comments: true
---

During the November meetup of the Philadelphia Area Sitecore User Group, we explored the possibilities of using Angular with Sitecore. Two options were explored. The first, building a SPA with Angular and Sitecore Item Web API and finally, Integrating Angular into a Sitecore instance. This post, the first in a series of two related posts, discusses the Sitecore Item Web API implementation.

## SPA with Sitecore Item Web API

Building a Single Page Application where we need content from Sitecore, the Sitecore Item Web API is a perfect option. We're able to access our Sitecore instance via GET requests, obtaining Sitecore items serialized as JSON. While adding a dependency on our SPA, we can contain the dependency as single point of entry and distribute the content across view models.

![angular web api](/assets/images/angular-web-api.jpg)

In our [SPA demo site](https://github.com/PetersonDave/SinglePageAppDemo), we use an Angular factory as the entry point into our Sitecore instance by way of the Sitecore item Web API. Angular factories allow us to obtain data and reuse it across multiple controllers and routes. The data is saved in an array through `get()` and made available via `getProfiles()`:

{% highlight javascript %}
    app.factory('profileservice', function ($http) {
        var items = {};
        var myProfileService = {};

        myProfileService.addItem = function (item) {
            items.push(item);
        };
        myProfileService.removeItem = function (item) {
            var index = items.indexOf(item);
            items.splice(index, 1);
        };
        myProfileService.getProfiles = function () {
            return items;
        };

        myProfileService.update = function (itemid, params) {
            console.log(params);
            
            var url = '-/item/v1/?sc_itemid=' + itemid;
            $http.put(url, params);
        };

        myProfileService.get = function (callback) {
            var url = '-/item/v1/?scope=s&query=/sitecore/content/Home/Repository/*';
            $http.get(url)
                .then(function (res) {
                    items = res.data.result.items;
                    callback();
                });
        };
        
        return myProfileService;
    });
{% endhighlight %}

<em>Note: For this demo, the get request is calling the current site, as the current site is running as the Sitecore instance. This can easily be changed to a non-relative URL path.</em>

The factory `profileService` then serves content to our controller `allprofiles`. There are two methods to take note of here:

* `vm.load()` - Makes the GET request to our Sitecore instance, populating the profiles array within our profileService.
* `populateRepository()` - Our callback method which profileService calls upon successfully loading profiles from the Sitecore Item Web API. A repository property of our model is populated here.

{% highlight javascript %}
(function () {
    'use strict';

    // Controller name is handy for logging
    var controllerId = 'allprofiles';

    // Define the controller on the module.
    // Inject the dependencies. 
    // Point to the controller definition function.
    angular.module('app').controller(controllerId,
        ['$scope', '$http', 'profileservice', allprofiles]);

    function allprofiles($scope, $http, profileservice) {
        var vm = this;

        vm.newprofile = {};
        vm.profileService = profileservice;
        vm.load = function () {
            profileservice.get(populateRepository);
        };

        function populateRepository() {
            vm.repository = profileservice.getProfiles();
        }
    }
})();
{% endhighlight %}

Finally, within our view, we access the data and use Angular's awesome databinding to populate our view from our view model:

{% highlight html %}
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

The "Load Data" link invokes our `vm.load()` method, while div class "row" is repeated for each item returned in our view model's repository property.

## Technical Considerations

### Security

When making Sitecore Item Web API calls, if you're planning on making cross-domain requests, you may run into some issues. While I have seen some recommended solutions, I have not tried implementing any.

Accessing Sitecore content adds additional configuration to ensure content can be accessed by your application. In our example, we're saving our credentials within $httpprovider's default header. Any GET request to our Sitecore instance are accessed as our demo site's Admin user:

{% highlight javascript %}
    app.config(
    ...
    '$httpProvider', function($httpProvider) {
        $httpProvider.defaults.headers.put = { 'X-Scitemwebapi-Username': 'admin' };
        $httpProvider.defaults.headers.put = { 'X-Scitemwebapi-Password': 'b' };        
    });
{% endhighlight %}
	
<em>Note: for obvious reasons, do not use your admin account. This is for demo purposes only.</em>

### Performance

While it might seem difficult to overload your Sitecore instance from requests coming from a SPA, be aware of how your SPA accesses your Sitecore instance. You would never want to have your method which invokes GET requests within any repeated elements, such as our ng-repeat element.  

## Conclusion

Accessing Sitecore content using the Sitecore Item Web API is very easy and works exceptionally well within a SPA or application using the AngularJS framework.

For the complete solution, including routing, Sitecore field mappings and all views and controllers, go here:

[https://github.com/PetersonDave/SinglePageAppDemo/tree/with-sitecore-webapi](https://github.com/PetersonDave/SinglePageAppDemo/tree/with-sitecore-webapi).

[jekyll-gh]: https://github.com/mojombo/jekyll
[jekyll]:    http://jekyllrb.com