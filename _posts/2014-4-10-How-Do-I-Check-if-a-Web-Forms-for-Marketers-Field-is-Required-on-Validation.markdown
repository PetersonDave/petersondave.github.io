---
layout: post
title:  "How Do I Check if a Web Forms for Marketers Field is Required on Validation?"
date:   2014-4-10 10:18:00
categories: Sitecore WFFM
---

When within the context of a WFFM field custom validator, determining whether or not a field is required may not be a straight-forward task. Understand that within the context of the validator, there are two ways of obtaining an instance of a form:

* Obtain an instance of the current SimpleForm control, find your field and check if any MarkerLabels are present
* Obtain an instance of the WFFM form itself, find your field item and check if its Required field is checked

## Approach 1: Obtain an Instance of the current SimpleForm

{% highlight c# %}
protected override bool OnServerValidate(string value)
{
    var validatorSettings = GetValidatorSettingsItem();
    Sitecore.Diagnostics.Assert.IsNotNull(validatorSettings, ValidationSettingsItemMissingMessage + ValidationSettingsItem);

    // obtain an instance of the SimpleForm
    var form = WebUtil.GetParent<SimpleForm>(this);

    // obtain an instance of our field
    var countryField = (CountryDropDownList)WebUtil.FindFirstOrDefault(form, c => c is CountryDropDownList && string.Compare((c as CountryDropDownList).Result.FieldName, validatorSettings.CountryFieldName) == 0);
    if (countryField != null)
    {
        // is the field required?
        bool isRequired = countryField.Requred.Any();
                
        // continue handling further processing here...
    }

    return base.OnServerValidate(value);
}
{% endhighlight %}

Here we're getting an instance of the SimpleForm using `WebUtil.GetParent<SimpleForm>()`. From there, we're searching for our field, also via WebUtil. One very important thing to keep in mind is that the instance of the form we're getting is the rendered output of the form itself and not the actual form item in Sitecore content. This makes it quite difficult to determine the checked state of the required checkbox on our form field. Instead, we're dependent upon doing an `.Any()` on the "Requred" property of our field to see if any `RequiredWithMarkerValidator` items exist.

<em>Shout out to [Mike Reynolds](http://sitecorejunkie.com/) for the idea of running `.Any()` on the Requred property.</em>

## Approach 2: Obtain an Instance of the current WFFM Item

{% highlight c# %}
protected override bool OnServerValidate(string value)
{
    var validatorSettings = GetValidatorSettingsItem();
    Sitecore.Diagnostics.Assert.IsNotNull(validatorSettings, ValidationSettingsItemMissingMessage + ValidationSettingsItem);

    // obtain an instance of the SimpleForm
    var form = WebUtil.GetParent<SimpleForm>(this);

    // lookup the actual form item
    var formItem = Sitecore.Context.Database.Items[form.FormID];

    // obtain the WFFM field
    var countryField = formItem.Children[validatorSettings.CountryFieldId];
            
    // get its required field value from the Checkbox field
    Sitecore.Data.Fields.CheckboxField checkbox = countryField.Fields["Required"];
    if (checkbox != null)
    {
        bool isRequired = checkbox.Checked;
        // continue processing...
    }

    return base.OnServerValidate(value);
}
{% endhighlight %}

Here we're still getting an instance of the `SimpleForm` using `WebUtilGetParent<SimpleForm>()`, but now we're using the FormId property on that control to get the Sitecore ID of the form. We can then use this to get the item in Sitecore content to read the "Required" field value and determine its checked state. 

Obviously, you'll want to do additional null checking and most likely use an ORM like [Synthesis](https://github.com/kamsar/Synthesis) to get strongly type objects mapping back to our WFFM forms and field items.