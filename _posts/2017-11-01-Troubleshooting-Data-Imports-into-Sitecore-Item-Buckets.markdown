---
layout: post
title:  "Troubleshooting Data Imports into Sitecore Item Buckets"
date:   2017-11-01 20:00:00
categories: Sitecore
comments: true
---

We recently ran into a scenario where items imported from an external source weren't making their way into Sitecore. A failure was recorded during item creation of the bucketable item. The items destination is a bucket, which organizes the bucket folders using the default bucket folder path based on date and time of the item's creation.

The following is the error which was recorded in the logs: 

An item name must satisfy the pattern: ^[\\w\\*\\$][\\w\\s\\-\\$]*(\\(\\d{1,}\\)){0,1}

A look inside the buckets provider provided direct insight to the root cause of the issue. The default provider (Sitecore 8.1u3), is defined as:
 
{% highlight xml %}
<bucketManager defaultProvider="default" enabled="true" patch:source="Sitecore.Buckets.config">
	<providers>
		<clear/>
		<add name="default" type="Sitecore.Buckets.Managers.PipelineBasedBucketProvider, Sitecore.Buckets"/>
	</providers>
</bucketManager>
{% endhighlight %}

Overriding the default bucket folder naming convention using the item name created an opportunity for error where the item name failed validation for creation of its parent bucket folder items.

By default, Sitecore uses date/time to determine bucket folders

{% highlight xml %}
<setting name="BucketConfiguration.BucketFolderPath" value="yyyy\/MM\/dd\/HH\/mm" patch:source="Sitecore.Buckets.config"/>
{% endhighlight %}

Using Sitecore's item bucket settings (/sitecore/system/Settings/Buckets/Item Buckets Settings), we were overriding the folder naming convention using the first two characters of the item's name. 

![buckets rules](/assets/images/buckets-rules.png)

Conversion of the item name to replace spaces with hypens during the import process ensured proper item naming standards, however, failed to create buckets folders in which an item contained a space in the second position. This resulted in an attempt to create a bucket parent folder of "-", which is invalid and generates an exception during item creation.