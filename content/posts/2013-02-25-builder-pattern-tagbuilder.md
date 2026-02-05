---
title: "Applying the builder pattern to the .Net MVC TagBuilder"
date: 2013-02-25T20:51:07+00:00
categories: ["C#"]
tags: [".Net Mvc", "Builder Pattern", "c#", "Nitpicking", "Web Development"]
---

Sometimes I'm a nitpicker. Actually I'm a nitpicker 24/7, but I can't really say that on a blog, can I?

So today's nitpicking is about C#'s TagBuilder. This class allows you to build html without writing a single '&lt;' or '&gt;'.  Using this class usually boils down to something like this:

```csharp
// this will create <button>
var input = new TagBuilder("button"); 

// Set some attributes
input.Attributes.Add("name", "property.name");
input.Attributes.Add("id", "property_id");

// some classes for markup
input.AddCssClass("btn btn-primary");

// equivalent to jQuery's $("property_id").text("Click me!");
input.InnerHtml = "Click me!"; 
```

However, wouldn't it be nicer if we could say:

```csharp
// this will create <button>
var input = new TagBuilder("button")
  // Set some attributes
  .Attribute("name", "property.name")
  .Attribute("id", "property_id")
  // some classes for markup
  .Class("btn btn-primary")
  // equivalent to jQuery's $("property_id").text("Click me!");
  .Html("Click me!");
```

It's not a *huge* difference, but all the actions are chainable, provide a nice and consistent API and follow the Builder pattern.
Hey, if you're going to call your class the TagBUILDER it's the least you can do. As hard as I tried, I couldn't restrain myself and did the unthinkable: I wrote extension methods.
As it turns out, it's far from a hurdle to implement this kind of thing:

```csharp
/// <summary>
/// Helper methods that allow builder-style construction of tags
/// </summary>
public static class ExtensionsForTagBuilder
{
    #region Attribute

    /// <summary>
    /// Sets an attribute on this tag
    /// </summary>
    /// <param name="this">This tagbuilder</param>
    /// <param name="attribute">The attribute to set</param>
    /// <param name="value">The value to set</param>
    /// <param name="replaceExisting">A value indicating whether the value should override the existing value should there be one.</param>
    /// <returns>This tagbuilder</returns>
    public static TagBuilder Attribute(this TagBuilder @this, string attribute, string value, bool replaceExisting = false)
    {
        @this.MergeAttribute(attribute, value, replaceExisting);
        return @this;
    }

    /// <summary>
    /// Sets an attribute on this tag
    /// </summary>
    /// <param name="this">This tagbuilder</param>
    /// <param name="attribute"></param>
    /// <param name="html"></param>
    /// <param name="replaceExisting">A value indicating whether the value should override the existing value should there be one.</param>
    /// <returns>This tagbuilder</returns>
    public static TagBuilder Attribute(this TagBuilder @this, string attribute, IHtmlString html, bool replaceExisting = false)
    {
        return Attribute(@this, attribute, html.ToString(), replaceExisting);
    }

    #endregion

    #region Class

    /// <summary>
    /// Adds a class to this tag.
    /// </summary>
    /// <param name="this">This tagbuilder</param>
    /// <param name="class">The class(es) to add</param>
    /// <returns>This tagbuilder</returns>
    public static TagBuilder Class(this TagBuilder @this, string @class)
    {
        @this.AddCssClass(@class);
        return @this;
    }

    #endregion

    #region Html

    /// <summary>
    /// Sets the InnerHtml property of this tag
    /// </summary>
    /// <param name="this">This tagbuilder</param>
    /// <param name="html">The html to set as the html for this tagbuilder</param>
    /// <returns>This tagbuilder</returns>
    public static TagBuilder Html(this TagBuilder @this, string html)
    {
        @this.InnerHtml = html;
        return @this;
    }

    /// <summary>
    /// Sets the InnerHtml property of this tag
    /// </summary>
    /// <param name="this">This tagbuilder</param>
    /// <param name="tag">The tag to set as the html for this tagbuilder</param>
    /// <returns>This tagbuilder</returns>
    public static TagBuilder Html(this TagBuilder @this, TagBuilder tag)
    {
        return Html(@this, tag.ToString());
    }

    /// <summary>
    /// Sets the InnerHtml property of this tag
    /// </summary>
    /// <param name="this">This tagbuilder</param>
    /// <param name="html">The html to set as the html for this tagbuilder</param>
    /// <returns>This tagbuilder</returns>
    public static TagBuilder Html(this TagBuilder @this, IHtmlString html)
    {
        return Html(@this, html.ToString());
    }

    #endregion

    #region AppendHtml

    /// <summary>
    /// Appends to the InnerHtml property of this tag
    /// </summary>
    /// <param name="this">This tagbuilder</param>
    /// <param name="html"></param>
    /// <returns>This tagbuilder</returns>
    public static TagBuilder AppendHtml(this TagBuilder @this, string html)
    {
        if (@this.InnerHtml == null)
            @this.InnerHtml = string.Empty;
        @this.InnerHtml += html;
        return @this;
    }

    /// <summary>
    /// Appends to the InnerHtml property of this tag
    /// </summary>
    /// <param name="this">This tagbuilder</param>
    /// <param name="tag">The tag to append to the html for this tagbuilder</param>
    /// <returns>This tagbuilder</returns>
    public static TagBuilder AppendHtml(this TagBuilder @this, TagBuilder tag)
    {
        return AppendHtml(@this, tag.ToString());
    }

    /// <summary>
    /// Appends to the InnerHtml property of this tag
    /// </summary>
    /// <param name="this">This tagbuilder</param>
    /// <param name="html">The html to append to the html for this tagbuilder</param>
    /// <returns>This tagbuilder</returns>
    public static TagBuilder AppendHtml(this TagBuilder @this, IHtmlString html)
    {
        return AppendHtml(@this, html.ToString());
    }

    #endregion

    #region PrependHtml

    /// <summary>
    /// Prepends to the InnerHtml property of this tag
    /// </summary>
    /// <param name="this">This tagbuilder</param>
    /// <param name="html">The html to prepend to the html for this tagbuilder</param>
    /// <returns>This tagbuilder</returns>
    public static TagBuilder PrependHtml(this TagBuilder @this, string html)
    {
        if (@this.InnerHtml == null)
            @this.InnerHtml = string.Empty;
        @this.InnerHtml = html + @this.InnerHtml;
        return @this;
    }

    /// <summary>
    /// Prepends to the InnerHtml property of this tag
    /// </summary>
    /// <param name="this">This tagbuilder</param>
    /// <param name="tag">The tag to prepend to the html for this tagbuilder</param>
    /// <returns>This tagbuilder</returns>
    public static TagBuilder PrependHtml(this TagBuilder @this, TagBuilder tag)
    {
        return PrependHtml(@this, tag.ToString());
    }

    /// <summary>
    /// Prepends to the InnerHtml property of this tag
    /// </summary>
    /// <param name="this">This tagbuilder</param>
    /// <param name="html">The html to prepend to the html for this tagbuilder</param>
    /// <returns>This tagbuilder</returns>
    public static TagBuilder PrependHtml(this TagBuilder @this, IHtmlString html)
    {
        return PrependHtml(@this, html.ToString());
    }

    #endregion

    /// <summary>
    /// Renders and returns the element as a <see cref="F:System.Web.Mvc.TagRenderMode.Normal"/> element.
    /// </summary>
    /// <param name="this">This tag</param>
    /// <returns>The rendered element as a <see cref="F:System.Web.Mvc.TagRenderMode.Normal"/> element</returns>
    public static MvcHtmlString ToHtml(this TagBuilder @this)
    {
        return MvcHtmlString.Create(@this.ToString());
    }

    /// <summary>
    /// Renders and returns the HTML tag by using the specified render mode.
    /// </summary>
    /// <param name="this">This tag</param>
    /// <param name="renderMode">The render mode.</param>
    /// <returns>The rendered HTML tag by using the specified render mode</returns>
    public static MvcHtmlString ToHtml(this TagBuilder @this, TagRenderMode renderMode)
    {
        return MvcHtmlString.Create(@this.ToString(renderMode));
    }
}
```

This is one of the things I really love about C#: you can roll your own API over existing classes as if they were there in the first place.
Using this builder pattern should make your HtmlHelpers much more readable. :-)

See you next time!
