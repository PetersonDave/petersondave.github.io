---
layout: post
title:  "Stripping Unwanted HTML From Sitecore Rich Text Fields"
date:   2018-11-27 20:00:00
categories: Sitecore
comments: true
---
A recent project requirement involved sharing of news article content between our public websites and a Xamarin app (which consumes Sitecore content through an API). Our article content, stored as Rich Text, can contain markup which does not translate well to a mobile app. To resolve, we introduced a new field to store the sanitized markup, updated during the item saving event handler. Below is the implementation. Fortis is used as the item wrappring library.

{% highlight c# %}
public class SanitizeArticleForAppHandler
{
    private readonly ISpawnProvider _spawnProvider;

    public bool Enabled { get; set; }
    
    public SanitizeArticleForAppHandler(ISpawnProvider spawnProvider)
    {
        this._spawnProvider = spawnProvider;
    }

    /// <summary>
    /// Called when [item saving].
    /// </summary>
    /// <param name="sender">The sender.</param>
    /// <param name="args">The <see cref="EventArgs"/> instance containing the event data.</param>
    public void OnItemSaving(object sender, EventArgs args)
    {
        try
        {
            if (!this.Enabled) return;

            var sitecoreItem = Event.ExtractParameter<Item>(args, 0);
            if (sitecoreItem == null) return;

            bool isContextMasterDatabase = SitecoreDatabaseHelper.MasterDatabase.Equals(sitecoreItem.Database.Name);
            if (!isContextMasterDatabase) return;

            var article = this._spawnProvider.FromItem<IArticlePage>(sitecoreItem) as IArticlePage;
            if (article == null) return;

            this.SanitizeArticleContentBody(article);
        }
        catch (Exception ex)
        {
            Sitecore.Diagnostics.Log.Error("Error while sanitizing article content", ex, this);
        }
    }

    /// <summary>
    /// Strip unwanted markup from content body for app consumption, save to 'sanitized content body' field
    /// </summary>
    /// <param name="article"></param>
    private void SanitizeArticleContentBody(IArticlePage article)
    {
        var doc = new HtmlAgilityPack.HtmlDocument();
        doc.LoadHtml(article.ContentBody.Value);

        doc.DocumentNode.Descendants()
            .Where(n => n.Name.Equals("script", StringComparison.OrdinalIgnoreCase) || n.Name.Equals("style", StringComparison.OrdinalIgnoreCase) || n.Name.Equals("iframe", StringComparison.OrdinalIgnoreCase))
            .ToList()
            .ForEach(n => n.Remove());

        using (new EventDisabler())
        {
            using (new SecurityDisabler())
            {
                article.SitecoreItem.Editing.BeginEdit();
                article.SitecoreItem.Fields["Sanitized Content Body"].Value = doc.DocumentNode.OuterHtml.Trim();
                article.SitecoreItem.Editing.EndEdit();
            }
        }
    }
}
{% endhighlight %}

Wiring up via patch:

{% highlight xml %}
<events>
     <event name="item:saving" role:require="!ContentDelivery">
       <handler type="Documents.Core.Events.SanitizeArticleForAppHandler, Documents.Core" method="OnItemSaving" resolve="true">
         <enabled>true</enabled>
       </handler>
     </event>
</events>
{% endhighlight %}

Note the event disabler to prevent the item saving event handler from being called repeatedly during the end edit of the item. Other approaches to consier involve updating content on the save event in the Rich Text field itself.