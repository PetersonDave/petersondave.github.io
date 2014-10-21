---
layout: post
title:  "Sitecore Web Forms for Marketers Password Confirmation Field"
date:   2013-9-21 10:18:00
categories: Sitecore WFFM
comments: true
---

Sitecore's Web Forms for Marketers contains a <em>Password Confirmation</em> complex field for accepting and validating both a password and password confirmation. The out-of-the-box configuration orders the required validators as:

* Confirmation must be filled in
* Password must be filled in

even though the fields are rendered on the form as:

* Password
* Confirmation

While this isn't a show-stopping issue for most people, you may want control over the display order of these validators.

There's currently no way that I've found to change the ordering via content or configuration. If you're looking to take control of the ordering, you'll have to programmatically alter the ordering within the complex field code-behind. Looking at the library which provides the implementation to this field, it's clear the order is reversed when setting the validation properties.

This code addresses the issue of required field validation ordering.

* *Field Location:* /sitecore/system/Modules/Web Forms for Marketers/Settings/Field Types/Complex/Password-Confirmation
* *Library:* Sitecore.Forms.Core.dll
* *Namespace:* Sitecore.Form.UI.UserControls

{% highlight c# %}
// maintaining format of original source.
public override bool SetValidatorProperties(BaseValidator validator)
{
    base.SetValidatorProperties(validator);
    object obj = null;
    if (validator as ICloneable != null)
    {
        obj = ((ICloneable)validator).Clone();
    }

    // apply the confirmation validator
    object[] confirmationTitle = new object[4];
    confirmationTitle[0] = this.ConfirmationTitle;
    confirmationTitle[1] = this.MinLength;
    confirmationTitle[2] = this.MaxLength;
    confirmationTitle[3] = this.PasswordTitle;
    validator.ErrorMessage = validator.ErrorMessage.FormatWith(confirmationTitle);
    object[] objArray = new object[4];
    objArray[0] = this.ConfirmationTitle;
    objArray[1] = this.MinLength;
    objArray[2] = this.MaxLength;
    objArray[3] = this.PasswordTitle;
    validator.Text = validator.Text.FormatWith(objArray);
    object[] confirmationTitle1 = new object[4];
    confirmationTitle1[0] = this.ConfirmationTitle;
    confirmationTitle1[1] = this.MinLength;
    confirmationTitle1[2] = this.MaxLength;
    confirmationTitle1[3] = this.PasswordTitle;
    validator.ToolTip = validator.ToolTip.FormatWith(confirmationTitle1);
            
    BaseValidator baseValidator = validator;
    string[] d = new string[2];
    d[0] = " confirmationControlId.";
    d[1] = this.confirmation.ID;
    baseValidator.CssClass = string.Concat(baseValidator.CssClass, string.Join(string.Empty, d));
    if (obj != null && obj as BaseValidator != null)
    {
        // build and insert the password validator in the first validation slot
        BaseValidator d1 = (BaseValidator)obj;

        object[] passwordTitle = new object[4];
        passwordTitle[0] = this.PasswordTitle;
        passwordTitle[1] = this.MinLength;
        passwordTitle[2] = this.MaxLength;
        passwordTitle[3] = this.ConfirmationTitle;
        d1.ErrorMessage = d1.ErrorMessage.FormatWith(passwordTitle);
        object[] minLength = new object[4];
        minLength[0] = this.PasswordTitle;
        minLength[1] = this.MinLength;
        minLength[2] = this.MaxLength;
        minLength[3] = this.ConfirmationTitle;
        d1.Text = d1.Text.FormatWith(minLength);
        object[] maxLength = new object[4];
        maxLength[0] = this.PasswordTitle;
        maxLength[1] = this.MinLength;
        maxLength[2] = this.MaxLength;
        maxLength[3] = this.ConfirmationTitle;
        d1.ToolTip = d1.ToolTip.FormatWith(maxLength);

        d1.ID = string.Concat(d1.ID, "confirmation");
        d1.ControlToValidate = this.confirmation.ID;
        if (validator.Attributes["inner"] != "1")
        {
            this.confirmationBorder.Controls.Add(d1);
        }
        else
        {
            this.confirmationPanel.Controls.Add(d1);
        }
    }
    return true;
}
{% endhighlight %}

[jekyll-gh]: https://github.com/mojombo/jekyll
[jekyll]:    http://jekyllrb.com
