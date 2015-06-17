---
layout: post
title:  "Extending Sitecore.Services.Client"
date:   2015-06-16 20:35:00
categories: Experiment Sitecore
comments: true
---
With the release of Sitecore 7.5, Sitecore's core product ships with the SPEAK framework. A dependency for SPEAK components is Sitecore.Services.Client, a REST API exposing Sitecore content for consuming applications. While the main consumer of this service are applications built with the SPEAK framework, developers can leverage this service for other non back-end applications.

While you can use Sitecore.Services.Client for public facing applications, there are certain risks that should be considered (which could be a post by itself). The goal of this post is to document an experiment in extending the existing functionality of the service to expose related content. 

The approach detailed within the post is purely experimental. Similar approaches should only be considered if validated against all other avenues for using Sitecore as a service. Consider ASP.NET MVC controller actions through attribute routing or implementing a custom ASP.NET Web API solution for CRUD operations.

#Obtaining Item References
The default implementation exposes content as JSON through a REST API where you can obtain content by:

	1. Item ID
	2. Item Path
	3. Query
	
The resulting JSON returns a set of data inclusive of standard Sitecore system related fields and custom template fields. For item referencing fields, a pipe-delimited list of ids is returned by the service rather than the related item's content. Having to obtain all content for a particular item and all of its related content requires multiple calls to the API.

For example, consider the following API call and its response:

##Request
_http://sitecore8/sitecore/api/ssc/item/?path=/sitecore/content/home/sample-item_

##Response
{% highlight json %}
{  
   "ItemID":"51ece5bb-1da7-4b7a-a168-cf90cd05f692",
   "ItemName":"sample-item",
   "ItemPath":"/sitecore/content/Home/sample-item",
   "ParentID":"110d559f-dea5-42ea-9c1c-8a5df7e70ef9",
   "TemplateID":"76036f5e-cbce-46d1-af0a-4143f9b557aa",
   "TemplateName":"Sample Item",
   "CloneSource":null,
   "ItemLanguage":"en",
   "ItemVersion":"1",
   "DisplayName":"sample-item",
   "HasChildren":"True",
   "ItemIcon":"/temp/IconCache/Applications/16x16/document.png",
   "ItemMedialUrl":"/temp/IconCache/Applications/48x48/document.png",
   "ItemUrl":"~/link.aspx?_id=51ECE5BB1DA74B7AA168CF90CD05F692&amp;_z=z",
   "Relationship":"{728D489D-4A4C-419A-A0E3-AE1D611A4BC1}|{91D41B8E-3921-4E40-A8B8-3E3A9AB1EE8B}",
   "Text":"",
   "Title":"Sample Item"
}
{% endhighlight %}

Notice how the multilist field "Relationship" outputs the item reference ids and not the actual content. To obtain the related item content, we would have to invoke the service for each item reference in the API response.

#Extending Sitecore.Services.Client
In order to include related content in our response, we'll have to override Sitecore's default implementation. 

We can do this by taking the following actions:

	1. Define our own Model Factory for the response
	2. Create our own Handler Provider to resolve the model factory
	3. Wire it all up through dependency injection

Required References:

	1. Sitecore.Services.Core
	2. Sitecore.Services.Infrastructure.Sitecore
	3. Sitecore.ContentSearch.Linq
	4. Ninject.Web.WebApi (DI for this example)

##Model Factory
We'll need a model factory to read in the related item content and append a new field to the ```ItemModel``` object. The ```ItemModel``` itself is a dictionary of key/value pairs where the key is the field name and the value is the field value. Adding a new field to our model is simple enough as we don't have to extend the model itself. It already supports dynamically adding fields and their values. 

The ```create``` method below iterates over each field and performs a lookup for related content items via ```ExpandReferencedItems```. The resulting array of items are then added to a new field ```RelationshipItems``` within our ```ItemModel```.

It's important we add a new field to the model as opposed to editing the existing field. We can't assume any SPEAK application dependencies can properly consume and parse an array of item content instead of the default pipe-delimited format.

{% highlight c# %}
public class CustomModelFactory : IModelFactory
{
	private readonly IModelFactory _factory;

	public CustomModelFactory()
	{
		_factory = new ModelFactory();
	}

	public ItemModel Create(Item item, GetRequestOptions requestOptions)
	{
		var model = _factory.Create(item, requestOptions);

		foreach (Field field in item.Fields)
		{
			// skip system-level fields
			if (field.Name.StartsWith("__")) continue;

			// skip non item referencing fields (accounted for in model above)
			var referencedIds = ID.ParseArray(field.Value);
			if (!referencedIds.Any()) continue;
			
			var referencedModels = ExpandReferencedItems(referencedIds, requestOptions);
			if (referencedModels.Any())
			{
				model[field.Name + "Items"] = referencedModels;
			}
		}

		return model;
	}

	private IEnumerable<ItemModel> ExpandReferencedItems(IEnumerable<ID> ids, GetRequestOptions requestOptions)
	{
		var referencedModels = new List<ItemModel>();
		foreach (var id in ids)
		{
			var item = Sitecore.Context.Database.Items[id];
			if (item == null) continue;

			var childItem = Create(item, requestOptions);
			referencedModels.Add(childItem);
		}

		return referencedModels;
	}

	public ItemModel[] Create(Item[] items, GetRequestOptions requestOptions)
	{
		return items.Select(item => Create(item, requestOptions)).ToArray();
	}

	public LinkModel CreateLink(string href, string rel, string method = "GET")
	{
		return _factory.CreateLink(href, rel, method);
	}

	public LinkModel CreateLink(HttpRequestMessage requestMessage, string controller, IDictionary<string, object> routeValues, string rel)
	{
		return _factory.CreateLink(requestMessage, controller, routeValues, rel);
	}

	public FacetCategoriesModel CreateFacets(FacetResults facets)
	{
		return _factory.CreateFacets(facets);
	}
}
{% endhighlight %}

##Handler Provider
With the Model Factory in place, we'll next need a Handler Provider to explicitly use the factory when requests are received for item lookup by id, path or by child item.

{% highlight c# %}
public class CustomHandlerProvider : IHandlerProvider
{
	private readonly IHandlerProvider _baseHandlerProvider;

	public CustomHandlerProvider()
	{
		_baseHandlerProvider = new HandlerProvider();
	}

	public IItemRequestHandler GetHandler<T>() where T : class
	{
		Type type = typeof(T);
		if (type == typeof(GetItemByContentPathHandler))
		{
			return new GetItemByContentPathHandler(ResolveItemRepository(), 
				ResolveModelFactory());
		}

		if (type == typeof(GetItemByIdHandler))
		{
			return new GetItemByIdHandler(ResolveItemRepository(),
				ResolveModelFactory());
		}

		if (type == typeof(GetItemChildrenHandler))
		{
			return new GetItemChildrenHandler(ResolveItemRepository(), 
				ResolveModelFactory());
		}

		return _baseHandlerProvider.GetHandler<T>();
	}

	public virtual IItemRepository ResolveItemRepository()
	{
		return new ItemRepository(new SitecoreLogger());
	}

	public virtual IItemSearch ResolveItemSearch()
	{
		return new ItemSearch();
	}

	public virtual IModelFactory ResolveModelFactory()
	{
		return new CustomModelFactory();
	}
}
{% endhighlight %}

##Dependency Injection
We then bind our new handler provider to Sitecore's ```IHandlerProvider```.

{% highlight c# %}
private static IKernel CreateKernel()
{
	var kernel = new StandardKernel();
	kernel.Bind<IHandlerProvider>().To<CustomHandlerProvider>();
	kernel.Bind<ILogger>().To<SitecoreLogger>();

	RegisterServices(kernel);
	return kernel;
}
{% endhighlight %}

#Final Result
The ```RelationshipItems``` field now displays all related content, eliminating the need for multiple Web API calls.

{% highlight json %}
{  
   "ItemID":"51ece5bb-1da7-4b7a-a168-cf90cd05f692",
   "ItemName":"sample-item",
   "ItemPath":"/sitecore/content/Home/sample-item",
   "ParentID":"110d559f-dea5-42ea-9c1c-8a5df7e70ef9",
   "TemplateID":"76036f5e-cbce-46d1-af0a-4143f9b557aa",
   "TemplateName":"Sample Item",
   "CloneSource":null,
   "ItemLanguage":"en",
   "ItemVersion":"1",
   "DisplayName":"sample-item",
   "HasChildren":"True",
   "ItemIcon":"/temp/IconCache/Applications/16x16/document.png",
   "ItemMedialUrl":"/temp/IconCache/Applications/48x48/document.png",
   "ItemUrl":"~/link.aspx?_id=51ECE5BB1DA74B7AA168CF90CD05F692&amp;_z=z",
   "Relationship":"{91D41B8E-3921-4E40-A8B8-3E3A9AB1EE8B}|{728D489D-4A4C-419A-A0E3-AE1D611A4BC1}",
   "Text":"",
   "Title":"Sample Item",
   "RelationshipItems":[  
      {  
         "ItemID":"91d41b8e-3921-4e40-a8b8-3e3a9ab1ee8b",
         "ItemName":"item reference 2",
         "ItemPath":"/sitecore/content/External/item reference 2",
         "ParentID":"6e85c0ee-50d6-4b73-8f59-ff5925849ec5",
         "TemplateID":"1930bbeb-7805-471a-a3be-4858ac7cf696",
         "TemplateName":"Standard template",
         "CloneSource":null,
         "ItemLanguage":"en",
         "ItemVersion":"1",
         "DisplayName":"item reference 2",
         "HasChildren":"False",
         "ItemIcon":"/temp/IconCache/Applications/32x32/Document.png",
         "ItemMedialUrl":"/temp/IconCache/Applications/48x48/Document.png",
         "ItemUrl":"~/link.aspx?_id=91D41B8E39214E40A8B83E3A9AB1EE8B&amp;_z=z"
      },
      {  
         "ItemID":"728d489d-4a4c-419a-a0e3-ae1d611a4bc1",
         "ItemName":"item reference 1",
         "ItemPath":"/sitecore/content/External/item reference 1",
         "ParentID":"6e85c0ee-50d6-4b73-8f59-ff5925849ec5",
         "TemplateID":"1930bbeb-7805-471a-a3be-4858ac7cf696",
         "TemplateName":"Standard template",
         "CloneSource":null,
         "ItemLanguage":"en",
         "ItemVersion":"1",
         "DisplayName":"item reference 1",
         "HasChildren":"False",
         "ItemIcon":"/temp/IconCache/Applications/32x32/Document.png",
         "ItemMedialUrl":"/temp/IconCache/Applications/48x48/Document.png",
         "ItemUrl":"~/link.aspx?_id=728D489D4A4C419AA0E3AE1D611A4BC1&amp;_z=z"
      }
   ]
}
{% endhighlight %}