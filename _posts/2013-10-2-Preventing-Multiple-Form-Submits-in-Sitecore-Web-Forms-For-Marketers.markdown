---
layout: post
title:  "Preventing Multiple Form Submits in Sitecore Web Forms for Marketers"
date:   2013-10-2 10:18:00
categories: Sitecore WFFM
---

When building forms using Web Forms for Marketers (WFFM), custom save actions can be added to save form data to a database or email a confirmation message. This post details an approach to disable the submit button to prevent multiple save actions from being triggered, such as sending multiple confirmation emails.

Similar to the approach outlined <a href="http://sitecorejunkie.com/2013/07/28/add-javascript-to-the-client-onclick-event-of-the-sitecore-wffm-submit-button/" target="_blank">here for attaching an alert on the submit button</a>, we're going to extend the class responsible for rendering WFFM forms and attach javascript to the submit button.

{% highlight c# %}
using System;
using Sitecore.Form.Core.Renderings;
using Sitecore.Form.Web.UI.Controls;

namespace Custom.Form.Web.UI.Controls
{
    public class PreventDoubleClickFormRender : FormRender
    {
        // disables submit button if group validator for submit is valid
        private const string PreventDoubleSubmitJs = @"function disableSubmitButton(groupValidator, submitButton) {{ $(submitButton).disabled = Page_ClientValidate(groupValidator);}}";

        protected override void OnLoad(EventArgs e)
        {
            base.OnLoad(e);

            Page.ClientScript.RegisterClientScriptBlock(GetType(), ID, PreventDoubleSubmitJs);

            var formSubmitId = FormInstance.ID + SitecoreSimpleForm.prefixSubmitID;
            var submit = FormInstance.FindControl(formSubmitId) as SubmitButton;
            if (submit != null)
            {
                submit.Attributes["onclick"] += string.Format("disableSubmitButton('{0}', '{1}');", submit.ValidationGroup, submit.ClientID);
            }
        }
    }
}
{% endhighlight %}

<strong>Validating Input</strong>

An important note about this code is that we do not disable the button if the form fails validation. On submit, ASP.NET's `Page_ClientValidate` is called to evaluate all validators defined within the group validator for the submit button `submit.ValidationGroup`.

**A Few Notes on Implementing**

* Notice the naming convention for the submit control. `Sitecore.Form.Web.UI.Controls.SitecoreSimpleForm` defines the naming convention within the control's `OnInit()` method.
* When appending the script to the `onclick` attribute of the submit button, do not use `OnClientClick()` as the submit control will replace your code within `OnLoad()` (reference: `Sitecore.Form.Web.UI.Controls.FormSubmit`).

**Wiring Up The Rendering**

The last step in using our new code is to reference the class within the defined Form rendering. Updating the namespace, tag and assembly bring it all together:

![wffm settings](/assets/images/wffm-settings.png)

[jekyll-gh]: https://github.com/mojombo/jekyll
[jekyll]:    http://jekyllrb.com
