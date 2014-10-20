---
layout: post
title:  "Working With The Sitecore Azure API: Settings and Content"
date:   2014-7-29 10:18:00
categories: Sitecore Azure
---

Details within this post outline how to obtain basic settings, global variables and Azure module items from Sitecore's Azure API. Included are also some pitfalls to be aware of when working with the library.


## Settings
The Settings object allows for access to the different settings defined within <em>Sitecore.Azure.config</em>.

{% highlight xml %}
<settings>
  <setting name="EnableEventQueues">
    <patch:attribute name="value">true</patch:attribute>
  </setting>

  <setting name="Media.DisableFileMedia">
    <patch:attribute name="value">true</patch:attribute>
  </setting>

  <setting name="Azure.EnvironmentsPath" value="/App_Data/AzureEnvironments" />
  <setting name="Azure.HostedServicePropertiesUpdateTime" value="00:00:20" />
  <setting name="Azure.DefaultUpdateCacheInterval" value="00:00:20" />
  <setting name="Azure.VendorsBlobContainer" value="http://cloudsettings.sitecore.net/vendors" />
  <setting name="Azure.VendorsStorage" value="/App_Config/AzureVendors" />
  <setting name="Azure.Package.NoEncryptPackage" value="false" />
  <setting name="Azure.RoleName" value="SitecoreWebRole" />
  <setting name="Azure.UiRefreshInterval" value="00:00:05" />
  <setting name="Azure.GetEnvironmentFileInfoBlobContainer" value="http://cloudsettings.sitecore.net/{version}-{locale}/GetEnvironmentFileInfo.html" />
  <setting name="Azure.HttpRequestRetries" value="3" />
  <setting name="Azure.TranslationsPath" value="/temp/AzureTranslations" />
  <setting name="Azure.ManagerDisabled" value="false" />
  <setting name="Azure.PublishTargetsContainer" value="publishtargets" />
  <setting name="Azure.TrafficManager.Enable" value="true" />

  <!-- Setup Logging level of Log trace of Sitecore Azure App. 
  Info  -  Show only common messages
  Debug -  Exception will be shown too. -->
  <setting name="Azure.LoggingSettings.LogLevel" value="Debug" />
</settings>
{% endhighlight %}

Obtaining `EnvironmentsPath` is exposed through the getter `Sitecore.Azure.Configuration.Settings.EnvironmentDefinitionsPath`.

## Global Variables

The `GlobalVariables` collection allows for retrieval of any `sc.variable` defined within the site's web.config.

{% highlight xml %}
<sitecore database="SqlServer">
  <sc.variable name="dataFolder" value="C:\inetpub\wwwroot_sandbox\Azure72\Data"/>
  <sc.variable name="mediaFolder" value="/upload"/>
  <sc.variable name="tempFolder" value="/temp"/>
{% endhighlight %}
  
Obtaining a global setting:

{% highlight c# %}
var dataFolder = Settings.GlobalVariables["dataFolder"];
{% endhighlight %}

Configuration settings is not the only data exposed through the Settings class, you can also obtain:

* Environment Definitions - A collection of all environments defined under the Azure module root item.
* Environments Root - The Azure module root item.
* Vendors - Collection of all vendors under the Azure module root item.

## Obtaining Azure Item Wrappers
Looking up specific Azure items using the standard <em>Sitecore.Data.Items.Item</em> object is fine, however, the Sitecore Azure API offers an ORM of sorts to access this information. You can instantiate these objects by passing in their related Sitecore item.

The example below shows how you can obtain an Azure deployment item and walk up the tree by accessing each item's parent until we reach the environment item root.

{% highlight c# %}
var db = Database.GetDatabase("master");

// Azure Deployment
var azureDeploymentDataItem = db.Items[deploymentId];
AzureDeploymentItem = new AzureDeploymentItem(azureDeploymentDataItem);

// Web Role
var roleDataItem = AzureDeploymentItem.InnerItem.Parent;
WebRoleItem = new WebRoleItem(roleDataItem);

// Farm
var farmDataItem = WebRoleItem.InnerItem.Parent;
FarmItem = new FarmItem(farmDataItem);

// Location
var locationDataItem = FarmItem.InnerItem.Parent;
LocationItem = new LocationItem(locationDataItem);

// Environment
var environmentDataItem = LocationItem.InnerItem.Parent;
EnvironmentItem = new EnvironmentItem(environmentDataItem);
{% endhighlight %}

## Obtaining Specific Azure Items

To obtain Azure specific objects, follow the patterns below to access. Take note of the <em>pitfalls</em> section when considering this approach.

### Environment

There are multiple ways to obtain an environment from the API. You can get all environment definitions:

{% highlight c# %}
var environments = Settings.EnvironmentDefinitions;
{% endhighlight %}

Explicitly target an environment type. Using a local emulator as an out-of-the-box example:

{% highlight c# %}
var environmentDefinition = Settings.EnvironmentDefinitions.GetEnvironment("Local Emulator");
var environment = Sitecore.Azure.Deployments.Environments.Environment.GetEnvironment(environmentDefinition);
{% endhighlight %}

<em>Local Emulator<em> is defined in the Azure Environment configuration file under <em>App_Data\AzureEnvironments\</em>

{% highlight c# %}
<?xml version="1.0" encoding="utf-16"?>
	<EnvironmentDataStorage xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
		<EMail>devfabric@sitecore.net</EMail>
		<EnvironmentId>fe6d565e-289b-4076-a089-3588321a836c</EnvironmentId>
		<EnvironmentType>Local Emulator</EnvironmentType>
		...
{% endhighlight %}
		
#### Location

Once an environment is obtained, getting an instance of a location is rather simple. You can Explicitly get a location:
{% highlight c# %}
var location = environment.GetLocation("localhost");
{% endhighlight %}

Iterate over a collection of locations:
{% highlight c# %}
var locations = environment.Locations;
{% endhighlight %}

<h4>Farm</h4>
Similar to the approaches above, a single farm can be obtained by deployment type:
{% highlight c# %}
var farm = location.GetFarm("Delivery01", DeploymentType.ContentDelivery);
{% endhighlight %}

as a collection:
{% highlight c# %}
var farms = location.Farms;
{% endhighlight %}

<h4>Web Role</h4>
Obtain a single instance by name:
{% highlight c# %}
var role = farm.GetWebRole("Role01");
{% endhighlight %}

or assuming <em>Role01</em> exists for a common Azure configuration:
{% highlight c# %}
var role = farm.WebRole01;
{% endhighlight %}

as a collection:
{% highlight c# %}
var roles = farm.WebRoles;
{% endhighlight %}

#### Azure Deployment
Obtain a single instance by deployment type:
{% highlight c# %}
var deployment = role.GetDeployment(DeploymentSlot.Production);
{% endhighlight %}

as a collection:
{% highlight c# %}
var deployments = role.Deployments;
{% endhighlight %}

### Deployment Settings
The deployment object exposes deployment settings from the API as well. In the example below, we leverage the `FilePathFilter` object to access deployment item settings:

{% highlight c# %}
public string GetExcludedDirectories()
{
    var environment = Sitecore.Azure.Deployments.Environments.Environment.GetEnvironment(Settings.EnvironmentDefinitions.GetEnvironment("Local Emulator"));
    var location = environment.GetLocation("localhost");
    var farm = location.GetFarm("Delivery01", DeploymentType.ContentDelivery);
    var role = farm.WebRole01;
    var deployment = role.GetDeployment(DeploymentSlot.Production);

    // get values from field "Exclude Directories"
    return deployment.FilePathFilter.ExcludeDirectories;
}
{% endhighlight %}

Exposing these fields:

![azure excludes](/assets/images/azure-excludes-ref.png)

## Pitfalls

### Item Creation on Object Instantiation

While the API itself is easy to use, it's essential to understand what's happening behind the scenes. Whenever instantiating an object deriving from the abstract class `AzureEntity`, if the requested item does not exist, the framework will automatically create the item for you. I've seen this happen specifically for location and farm items. Make sure your name matches the entry you're looking for, otherwise, you'll have extra items created under the Azure module branch in the content tree.

Consider a scenario of requesting a location of <em>MyLocation</em>. The Azure location item does not exist. I'll run the following code:

{% highlight c# %}
public Location GetNonExistingLocation()
{
    var environment = Sitecore.Azure.Deployments.Environments.Environment.GetEnvironment(Settings.EnvironmentDefinitions.GetEnvironment("Local Emulator"));
    return environment.GetLocation("MyLocation");
}
{% endhighlight %}

Suppose I then request the farm <em>MyFarm</em> which also does not exist in content:

{% highlight c# %}
public Farm GetNonExistingFarm()
{
    var location = GetNonExistingLocation();
    return location.GetFarm("MyFarm", DeploymentType.ContentDelivery);
}
{% endhighlight %}

This results in those items being created by the API:

![azure api ref](/assets/images/azure-api-ref.png)