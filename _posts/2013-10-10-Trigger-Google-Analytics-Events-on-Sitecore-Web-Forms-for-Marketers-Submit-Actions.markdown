---
layout: post
title:  "Trigger Google Analytics Events on Sitecore Web Forms for Marketers Submit Actions"
date:   2013-10-10 10:18:00
categories: Sitecore WFFM
comments: true
---

Suppose we wan to record an event in Google Analytics when successfully submitting forms. Before triggering our Google Analytics event, we first need to understand the different success modes:

* Redirect - After successfully passing form validation, the request is redirected to a specified success page.
* Success Message - After successfully passing form validation, a success message is rendered on the page after the post back, keeping the user on the same page.

Whichever success mode we choose, we need a client-side event to be triggered via our Google Analytics API. One commonality between the different success modes is the rendering logic for obtaining the success message in content and rendering on our WFFM form. We're going to take advantage of this logic to inject our client-side event at the end of the success message.

The solution involves the following steps:

* Defining our Google Analytics Tracking Save Action
* Adding the Google Analytics Tracking Save Action to a form
* Modifying the forms success action pipeline to inject the tracking script
* Let Sitecore WFFM handle the rest

## Success Action
We'll use  a custom Save Action to hold our client-side script and format it to include any appropriate attributes for our tracking code.

Save actions are saved under: <em>/sitecore/System/Modules/Web Forms for Marketers/Settings/Actions/Save Actions/</em>

![save action](/assets/images/save-action.png)

Since this is all client-side, the goal of this event is to reuse and attach the script to appropriate forms by way of a Save Action. The save action will no be doing any server-side actions. As you can see, the code for this save action is empty:

{% highlight c# %}
public class RecordGoogleAnalyticsSubmitAction : ISaveAction, ISubmit
{
    public void Execute(ID formid, AdaptedResultList fields, params object[] data)
    {
        return;
    }

    public void Submit(ID formid, AdaptedResultList fields)
    {
        return;
    }
}
{% endhighlight %}

Once created, add the save action to any forms which require tracking events on submit within Google Analytics.

## Pipeline Processor
In the WFFM config include, there are two pipeline processors for each Success Mode. We're going to replace FormatSuccessMessage with our own to append the script to the end of the success message. The success message will be formatted for both success modes (redirect and show message).

{% highlight xml %}
<successAction>
	<processor type="Sitecore.Form.Core.Pipelines.SuccessRedirect, Sitecore.Forms.Core"/>
	<processor type="Custom.Pipelines.FormatSuccessMessage, Custom"/>
</successAction>
{% endhighlight %}

The FormatSuccessMessage implementation is simple, we append our script to the success message:

{% highlight c# %}
public class FormatSuccessMessage
{
    public void Process(SubmitSuccessArgs args)
    {
        var script = FormSaveActionUtilities.GetGoogleAnalyticsScriptFromSaveAction(args);

        Assert.IsNotNull((object)args, "args");
        if (args.Form == null)
            return;
        args.Result = args.Form.SuccessMessage + script;
    }
}
{% endhighlight %}

The Google Analytics script is obtained by looking up the Save Action, if it exists on the form. Save actions are available in the SubmitSuccessArgs as XML. We'll take that XML, parse it into a ListDefinition. From there, we can convert to items, commands and finally ActionItems. Once we find our action item in the list of ActionItems, we pull our script and format it properly.

{% highlight c# %}
public class FormSaveActionUtilities
{
    public static string GetGoogleAnalyticsScriptFromSaveAction(SubmitSuccessArgs args)
    {
        var googleAnalyticsSubmitActionId = new ID("{617DD7D9-950B-4767-B40A-CEFA848B64C6}");
        var script = string.Empty;

        var lid = ListDefinition.Parse(args.Form.SaveActions).Groups[0].ListItems;
        if (lid == null) return script;

        var items = lid.Select(item => args.Form.Database.GetItem(item.ItemID));
        var actions = items.Where(command => command != null)
                            .Select(command => new ActionItem(command))
                            .ToList();

        var googleAnalyticsAction = actions.FirstOrDefault(action => action.ID == googleAnalyticsSubmitActionId);
        if (googleAnalyticsAction != null)
        {
            script = string.Format(googleAnalyticsAction.Parameters, args.Form.Name);
        }

        return script;
    }
}
{% endhighlight %}

On submit, the pipeline processor finds the script, formats it and appends it to the success message. Within the WFFM Simple Form, the form programmatically pulls our modified success message and renders within a literal, triggering our client-side tracking code.