---
layout: post
title:  "Adventures in Item Buckets"
date:   2016-11-25 20:00:00
categories: Sitecore Unicorn
comments: true
---
Recently, we can into multiple issues regarding the Multilist with Search field and <a href="https://doc.sitecore.net/sitecore_experience_platform/content_authoring/managing_items/item_buckets/item_buckets" target="_blank">Sitecore Item Buckets.</a>. 

This field has known <a href="https://kb.sitecore.net/articles/372032" target="_blank">documented issues</a>. Given our dependency on this field type throughout our solution, resolving the issues we encountered was absolutely critical for a fully functional CMS.

# Hanging Search Results

Editors were getting inconsistent behavior in how results were displayed when searching using the Multilist with Search field. More often than not, the search field looked as if it was searching for results, but none were actually displayed; erroring out in the background. Similar to other <a href="http://stackoverflow.com/questions/31458086/how-do-i-control-the-priority-of-nested-queries-in-sitecore-contentsearch-with-t" target="_blank">reported issues</a>, the provider was presumably erroring out on some of our more complicated queries.

Sitecore support issued a patch referencing issue #398622, in which the Solr provider was overridden using a custom assembly. The following references were replaced:

1. Search Provider
```Sitecore.ContentSearch.SolrProvider.SolrSearchIndex, Sitecore.ContentSearch.SolrProvider```
with
```Sitecore.Support.ContentSearch.SolrProvider.SolrSearchIndex, Sitecore.Support.398622```

2. Switch on Rebuild Provider
```Sitecore.ContentSearch.SolrProvider.SwitchOnRebuildSolrSearchIndex, Sitecore.ContentSearch.SolrProvider```
with
```Sitecore.Support.ContentSearch.SolrProvider.SwitchOnRebuildSolrSearchIndex, Sitecore.Support.398622```

# Item Buckets Settings

Altering the behavior of item bucket settings (/sitecore/system/Settings/Buckets/Item Buckets Settings) for the field _Show Search Results In_ to _New Content Editor_ produces the issue of not being able to search for results within the Multilist with Search field. 

To reproduce the issue, we first searched for a result within a bucketable item. Opening one of the items returned in the search results to a new content editor window creates a scenario where any subsequent searches (via the Multilist with Search field, for example) are conducted under the context of an item not included within a registered search index. This then executes against the core database instead of the intended target.

Applying <a href = "https://kb.sitecore.net/articles/755636" target="_blank">a patch</a> produced by Sitecore to introduce an index fallback resolves the issue. In our case, the item's parent Bucket template was not included in the templates for the crawler configuration of our master index. 

# Unable to Create Bucketable Items

Creation of bucketable items where more than one item is created in a given item bucket folder will produce unexpected results. Editors will be presented with a random "Save Changes" dialogue on item creation, in which clicking "yes" forces the item to be saved as a non-bucketable item in an item bucket. 

Given an OOTB configuration, this particular scenario might be hard to produce consistently as the folder structure is determined by the date and time of item creation down to the minute. In other words, you'd have to create multiple items in the same minute to produce the issue. For our scenario, we are overriding the default folder structure where multiple items can be created within the same bucket folder rather easily.

Getting around this issue involves disabling of the pipeline processor that selects the item after item creation:

{% highlight xml %}
<processors>
  <uiAddFromTemplate>
    <processor patch:instead="*[@type='Sitecore.Buckets.Pipelines.UI.AddFromTemplate.SelectItem']" mode="off" type="Sitecore.Buckets.Pipelines.UI.AddFromTemplate.SelectItem" />
  </uiAddFromTemplate>
</processors>
{% endhighlight %}