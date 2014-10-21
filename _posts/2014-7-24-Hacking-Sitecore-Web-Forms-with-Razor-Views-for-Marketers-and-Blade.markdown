---
layout: post
title:  "Hacking Sitecore Web Forms with Razor Views for Marketers and Blade"
date:   2014-7-24 10:18:00
categories: Sitecore WFFM
comments: true
---

[Web Forms for Marketers](http://sdn.sitecore.net/Products/Web%20Forms%20for%20Marketers.aspx) provides flexibility on part of content editors to create and manipulate simple forms for collecting user data. While there's a great deal of flexibility on the back-end, customizing the format and layout of the form can sometimes be a more difficult task.

Knowing what some of the limitations are regarding the rendered markup of Web Forms for Marketers, I wondered how difficult it would be to override the rendering of the form to allow drastic changes to the standard look-and-feel of the out-of-the-box implementation. Sure, content edits can add CSS classes to content, but I wanted to push the envelope and see how far we could go.

## Disclaimer

Now, I'll be the first to admit when you're looking to use a feature like Web Forms for Marketers, it's absolutely necessary to know the strengths and limitations prior to recommending a specific implementation plan. Working closely with clients and designers is critical to a successful implementation. One that can be managed well over time by content editors, but remains on the upgrade path for future updates by Sitecore.

## Introducing Razor Views for Marketers
The goal of this project is simple. Override the rendering of Web Forms for Marketers forms to take full control of the rendered markup.

Purely out of research, [Razor Views for Marketers](https://github.com/PetersonDave/RazorViewsForMarketers) was built to extend the rendering of the form. Content remains the same, as well as the structure and templates within Sitecore. Razor Views for Marketers simply replaces how forms are rendered and how validation is conducted against view models. Page editor also works.

## Why Blade?

Razor Views for Marketers uses [Blade](https://github.com/kamsar/Blade) to take advantage of MVC-style razor view templating, allowing us to leverage MVC editor templates and dynamic model binding. The razor views give complete control over how fields are rendered, as well as razor views for field sections and the form itself.

## Example Form
As a proof of concept, I set out to replace the "Leave a Message" form. Out-of-the-box implementation renders the form as:

![leave a message](/assets/images/leave-a-message.png)

With a Single-Line Text field rendering as:

{% highlight html %}
<div id="main_0_centercolumn_0_form_EC97FD637B414CA48CEF55F2D4EA1916_field_1C1014623802498484904C0DACF7D6FB_scope" class="scfSingleLineTextBorder fieldid.%7b1C101462-3802-4984-8490-4C0DACF7D6FB%7d name.Your+name">
	<label for="main_0_centercolumn_0_form_EC97FD637B414CA48CEF55F2D4EA1916_field_1C1014623802498484904C0DACF7D6FB" id="main_0_centercolumn_0_form_EC97FD637B414CA48CEF55F2D4EA1916_field_1C1014623802498484904C0DACF7D6FB_text" class="scfSingleLineTextLabel">Your name</label>
	<div class="scfSingleLineGeneralPanel">
		<input name="main_0$centercolumn_0$form_EC97FD637B414CA48CEF55F2D4EA1916$field_1C1014623802498484904C0DACF7D6FB" type="text" maxlength="256" id="main_0_centercolumn_0_form_EC97FD637B414CA48CEF55F2D4EA1916_field_1C1014623802498484904C0DACF7D6FB" class="scfSingleLineTextBox">
		<span class="scfSingleLineTextUsefulInfo" style="display:none;"></span>
		<span id="main_0_centercolumn_0_form_EC97FD637B414CA48CEF55F2D4EA1916_field_1C1014623802498484904C0DACF7D6FB6ADFFAE3DADB451AB530D89A2FD0307B_validator" class="scfValidator trackevent.%7bF3D7B20C-675C-4707-84CC-5E5B4481B0EE%7d fieldid.%7b1C101462-3802-4984-8490-4C0DACF7D6FB%7d inner.1" style="color:Red;display:none;">Your name must have at least 0 and no more than 256 characters.</span><span id="main_0_centercolumn_0_form_EC97FD637B414CA48CEF55F2D4EA1916_field_1C1014623802498484904C0DACF7D6FB070FCA141E9A45D78611EA650F20FE77_validator" class="scfValidator trackevent.%7b844BBD40-91F6-42CE-8823-5EA4D089ECA2%7d fieldid.%7b1C101462-3802-4984-8490-4C0DACF7D6FB%7d inner.1" style="color:Red;display:none;">The value of the Your name field is not valid.</span>
	</div>
	<span class="scfRequired">*</span>
</div>
{% endhighlight %}

<strong>Replacement Form</strong>
The Razor Views for Marketers Implementation, using the following set of Razor views:

{% highlight html %}
@using System.Web.Mvc
@using System.Web.Mvc.Html
@using RazorViewsForMarketers.Helpers
@inherits BladeRazorRendering<RazorViewsForMarketers.Models.RazorViewForMarketersFormModel>

<h1>@Model.Form.Title</h1>

@if (Model.Form.ShowIntroduction)
{
    <p>@Model.Form.Introduction</p>
}

@using (Html.BeginForm(null, null, FormMethod.Post, new { id = "razorViewForMarketers" }))
{
    @Html.DisplayTextFor(model => model.SubmitMessage)

    <div class="form">
        @Html.EditorFor(model => model.Form.Sections)
    </div>

    <button type="submit" value="Submit" name="Command">Submit Form</button>
}

<script src="~/Scripts/jquery-1.10.2.js"></script>
<script src="~/Scripts/jquery.unobtrusive-ajax.js"></script>
<script src="~/Scripts/jquery.validate.js"></script>
<script src="~/Scripts/jquery.validate.unobtrusive.js"></script>
{% endhighlight %}

Sections, implemented with editor templates:

{% highlight html %}
@using System.Web.Mvc.Html
@model RazorViewsForMarketers.Models.WffmSection

<div class="section">    
    @Html.EditorFor(model => model.Fields)
</div>
{% endhighlight %}

and fields also have editor templates. Example of a Single-Line Text:

{% highlight html %}
@using System.Web.Mvc.Html
@using RazorViewsForMarketers.Helpers
@inherits BladeRazorRenderingEditorTemplate<RazorViewsForMarketers.Models.Fields.SingleLineTextField>

<div class="field">
    <!-- model binding fields -->
    @Html.HiddenFor(model => model.Id)
    @Html.HiddenFor(model => model.IsRequired)
    @Html.Hidden("ModelType", Model.ModelType)
    <!-- model binding fields -->

    @BladeHtmlHelper.SitecoreLabel(Html, model => model)

    @Html.TextBoxFor(model => model.Response)
    @BladeHtmlHelper.RequiredIndicator(model => model)

    <p>@Model.Information</p>

    @Html.ValidationMessageFor(model => model.Response)
</div>
{% endhighlight %}

Through helper methods, we're able to maintain Page Editor functionality while rendering Web Forms for Marketers fields. Hidden fields are necessary for dyanmic model binding on postback for the form.

Updated rendered output of the Single-Line Text field from the razor view above:

{% highlight html %}
<div class="field">
    <!-- model binding fields -->
    <input id="Form_Sections_0__Fields_0__Id" name="Form.Sections[0].Fields[0].Id" type="hidden" value="1c101462-3802-4984-8490-4c0dacf7d6fb">
    <input id="Form_Sections_0__Fields_0__IsRequired" name="Form.Sections[0].Fields[0].IsRequired" type="hidden" value="True">
    <input id="Form_Sections_0__Fields_0__ModelType" name="Form.Sections[0].Fields[0].ModelType" type="hidden" value="RazorViewsForMarketers.Models.Fields.SingleLineTextField">
    <!-- model binding fields -->

    <label for="Form_Sections_0__Fields_0__field_Response">Your name</label>
    <input id="Form_Sections_0__Fields_0__Response" name="Form.Sections[0].Fields[0].Response" type="text" value=""> *
</div>
{% endhighlight %}

## Implementation Notes
The Razor Views for Marketers core logic [wraps the existing SimpleForm](https://github.com/PetersonDave/RazorViewsForMarketers/blob/master/Source/RazorViewsForMarketers/Core/ModifiedSimpleForm.cs) to perform operations such as:

* Submitting the form
* Collecting WFFM save actions
* Collecting form field control results
* Expose the form's FormItem

The wrapper is essentially an empty shell of the form, allowing us to leverage the existing WFFM framework for submitting forms through the standard Sitecore WFFM pipelines.

A [field decorator](https://github.com/PetersonDave/RazorViewsForMarketers/blob/master/Source/RazorViewsForMarketers/Core/DecoratedControlResult.cs) encapsulates existing WFFM field control results for form processing.

## Want More?
If you're interested in this approach, [check out the repo](https://github.com/PetersonDave/RazorViewsForMarketers) <a href="" target="_blank">.

Recommendations on how to contribute are included. The framework supports model binding of all existing Web Forms for Marketers fields, however, views and validators have been created to accommodate only those fields within "Leave a Message" form.