---
layout: post
title:  "Complex Cross-Field Validation in Sitecore Web Forms for Marketers"
date:   2013-10-3 10:18:00
categories: Sitecore WFFM
comments: true
---

Web Forms for Marketers (WFFM) offers multiple ways to validate form data prior to executing the Form Submit Actions. Outlined below are various methods to validating form data given the level of complexity required in your validation rules.

## Standard Validation

If you've worked with WFFM in the past, you're familiar with the standard field validators. The scope of validation is limited to the value of the field and nothing more. Using the complex field type <em>Password-Confirmation</em> as an example, one of the validators selected by default is the <em>Compare Password - Confirmation</em> validator.

![standard](/assets/images/standard1.png)

Looking at the code, notice how scope is limited to the field value:

{% highlight c# %}
using Sitecore.Form.Core.Validators;
using System.Web.UI;
using System.Web.UI.WebControls;

namespace Sitecore.Form.Core.Client.Validators
{
  public class PasswordConfimationValidator : FormCustomValidator
  {
    protected override bool OnServerValidate(string value)
    {
      Control control = this.FindControl(this.classAtributes["confirmationControlId"]);
      if (control != null)
        return ((TextBox) control).Text == value;
      else
        return false;
    }
  }
}
{% endhighlight %}

## Cross-Field Validation

Suppose we have a rule which requires data in one field to be a certain value given data in a completely different field. We have a couple of options:

* Standard Validation - paired with helper methods to traverse controls on the page to find the control we're looking for
* Form Verification Actions - access to all form fields

### Option 1: Standard Validation

Given the explanation above, the goal of standard form field validators is clear -- validate the value for the associated field and nothing more. With the help of Sitecore Forms Core library utility methods, we can search for specific fields in our form. Finding fields outside the context of our field validator.

Please note that while this approach allows for cross-field validation, our validator is now bound to a field in code limiting use of the validator for a very specific field set. Removing our code-referenced field from our form would require altering content validation settings.

{% highlight c# %}
protected override bool OnServerValidate(string value)
{
  // get the form
  var form = WebUtil.GetParent<SimpleForm>(this);
  
  // find another field within our form
  var field = (SingleLineText)WebUtil.FindFirstOrDefault(form, c => c is SingleLineText && (c as SingleLineText).Result.FieldName == "City");

  // compare values between this field (State) and the value of a different field (City)
  bool isBoston = field != null && field.Result.Value.ToString() == "Boston";
  bool isMassachusetts = !string.IsNullOrEmpty(value) && value == "MA";
  return isBoston && isMassachusetts;
}
{% endhighlight %}

* `Webutil.GetParent()` - we're able to locate the WFFM form user control
* `Webutil.FindFirstOrDefault()` - by passing in our Func to locate the field City of type SingleLineText, we're able to locate a field outside of our validator's context field.

### Option 2: Form Verification Actions

If option 1 above does not suit your needs, we must turn our attention to Form Verification Actions. With Verification Actions, we have access to all form field values and can implement validation rules for any level of validation complexity.

Continuing with our Password-Confirmation example above, the <em>Check User and Password</em> Form Verification provides a more complex level of validation:

![verification](/assets/images/verification.png)

The `CheckUserPassword` class overrides Execute, providing access to all fields within the form -- notice the `IEnumerable<ControlResult>` fields:

{% highlight c# %}
namespace Sitecore.Form.Submit
{
	public class CheckUserPassword : CheckUserAction
	{
	...
		public override void Execute(ID formid, IEnumerable<ControlResult> fields)
		{
			string empty;
			string failedMessage = base.FailedMessage;
			string str = failedMessage;
			if (failedMessage == null)
			{
				str = ResourceManager.GetString("USER_NAME_OR_PASSWORD_IS_INCORRECT");
			}
			string str1 = str;
			
			ControlResult controlResult = fields.FirstOrDefault<ControlResult>((ControlResult f) => f.FieldID == base.UserNameField);
			ControlResult controlResult1 = fields.FirstOrDefault<ControlResult>((ControlResult f) => f.FieldID == this.PasswordField);
			if (controlResult != null)
			{
      			...
{% endhighlight %}

## Complex Cross-Field Validation

Now that we've got the basics out of the way, lets get to the good stuff.

When validating from within a Form Verification action, suppose we need to access a drop-down field's data source to determine if another field value is a value within the drop-down list. To achieve this, we can take advantage of some helper classes throughout the `Sitecore.Forms.Core` library.

The data source for the drop-down is actually defined within the field `Parameters` of the field item in the form. As an example, your parameters field value might resemble the following:

![parameters](/assets/images/parmeters.png)

To validate based on the values with a paramters field, we can use code similar to the following:

{% highlight c# %}
private static bool IsValueInFieldDatasource(IEnumerable<ControlResult> fields, string fieldName, string value)
{
	Sitecore.Diagnostics.Assert.IsNotNullOrEmpty(fieldName, "fieldName cannot be null");
	if (string.IsNullOrEmpty(value)) return false;

	bool isValidSelection = false;

	var field = fields.FirstOrDefault(x => string.Compare(x.FieldName, fieldName, StringComparison.OrdinalIgnoreCase) == 0);
  	if (field != null)
	{
		var parameters = Sitecore.Context.Database.Items[field.FieldID].Fields["Parameters"];
		if (parameters.HasValue)
		{
            		IEnumerable<Pair<string, string>> p = ParametersUtil.XmlToPairArray(parameters.Value, true);
			var query = p.FirstOrDefault();
				    
            		bool isQueryValid = query != null && !string.IsNullOrEmpty(query.Part2);
            		if (isQueryValid)
            		{
                		var querySettings = QuerySettings.ParseRange(query.Part2);
                		var dataSource = QueryManager.Select(querySettings);

                		isValidSelection = dataSource.Get(value) != null;
            		}
		}
	}

	return isValidSelection;
}
{% endhighlight %}

We can't work directly with the raw parameters value. There are a few things we have to first accomplish:

* URL decode the Parameters field value. Without it, we can't transform this into actual data.
* Parse the value above into an instance of QuerySettings.
* Execute the query defined within QuerySettings via QueryManager to get the actual results.

QuerySettings has all sorts of helper properties and methods. By properly parsing the XML field value into an instance of QuerySettings, we automatically get the query and all associate data/text fields.

Some notes on the code snippet:

* `ParametersUtil.XmlToPairArray()` - Parses the XML values within the parameters field into a key/value pair of settings defined within the field. We use this to obtain the actual query. The second parameter tells ParametersUtil to URL decode the string value.
* `QuerySettings.ParseRange()` - Parses the XML into an instance of QuerySettings which we can use to run a query and get the results of the data source.
* `QueryManager.Select()` - Executes a Sitecore query given a QuerySettings instance.

[jekyll-gh]: https://github.com/mojombo/jekyll
[jekyll]:    http://jekyllrb.com
