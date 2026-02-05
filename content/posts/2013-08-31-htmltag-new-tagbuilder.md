---
title: "HtmlTag, it's the new TagBuilder"
date: 2013-08-31T22:37:17+00:00
categories: ["C#"]
tags: ["HtmlBuilders", "ASP.NET MVC", "TagBuilder", "NuGet"]
---

Hello again! Long time no see.

Today's post will be nothing but advertising for my latest and greatest open source library called [HtmlBuilders](https://github.com/amoerie/htmlbuilders). It's even conveniently available as a Nuget package, so what are you waiting for? (*All the cool kids are using it, so why not you?*)

Okay maybe I should elaborate a bit and tell you why you would want to use this. To do that, we must pay a small visit to the past: do you remember [my earlier post about the TagBuilder in C#](/posts/2013-02-25-builder-pattern-tagbuilder/)? You can go check it out, I'll wait.

...

Anyway, as I churn out industrial amounts of HTML on a daily basis, I'm always looking for ways to ~~type less~~ be more productive. Of course, when I say that I churn out HTML, I actually mean I'm writing HTML helpers to somewhat organize my large projects and keep my views readable and clean to other developers.
In my earlier post, I 'enhanced' the existing API of the TagBuilder by using C#'s [extension methods](http://msdn.microsoft.com/en-us/library/vstudio/bb383977.aspx), but extension methods will only take you a certain distance.

In the end, I was met with the following limitations:

- No support for parsing an HTML string to a TagBuilder
- No LINQ-style approach to manipulating the children/parents of a tag, as seen in jQuery
- No fluent syntax available (this was partially solvable with extension methods)
- No easy access to the style attribute

So, in my youthful and naive enthusiasm I set out to write my own TagBuilder called it HtmlTag.
Let's take a crash course, shall we?

## Part 1: Creating HTML

If you fancy [Twitter Bootstrap](http://getbootstrap.com/) as much as I do, this little snippet of HTML will be no stranger to you:

```html
<div class="control-group">
    <div class="controls">
        <label class="checkbox">
            <input type="checkbox"> Remember me
        </label>
        <button type="submit" class="btn">Sign in</button>
    </div>
</div>
```

Now, if you were to write an HTML helper for this piece of HTML, it would probably look a bit like this: (assuming you use TagBuilder)

```csharp
var controlGroup = new TagBuilder("div");
controlGroup.AddCssClass("control-group");
var controls = new TagBuilder("div");
controls.AddCssClass("controls");
var label = new TagBuilder("label");
label.AddCssClass("checkbox");
var input = new TagBuilder("input");
input.MergeAttribute("input", "checkbox");
label.InnerHtml = input + " Remember me";
var button = new TagBuilder("button");
button.MergeAttribute("type", "submit");
button.AddCssClass("btn");
button.InnerHtml = "Sign in";
controls.InnerHtml = label.ToString() + button;
controlGroup.InnerHtml = controls.ToString();
```

With the extension methods from my earlier post you could even do something like this:

```csharp
var fluent = new TagBuilder("div").Class("control-group")
    .AppendHtml(new TagBuilder("div").Class("controls")
        .AppendHtml(new TagBuilder("label").Class("checkbox")
            .AppendHtml(new TagBuilder("input").Attribute("type", "checkbox"))
            .AppendHtml(" Remember me")))
    .Append(new TagBuilder("button").Attribute("type", "submit").Class("btn").AppendHtml("Sign in"));
```

Sweet. Or not sweet enough? What about the blazing new HtmlTag API:

```csharp
var fluent = new HtmlTag("div").Class("control-group")
    .Append(new HtmlTag("div").Class("controls")
        .Append(new HtmlTag("label").Class("checkbox")
            .Append(new HtmlTag("input").Type("checkbox"))
            .Append(" Remember me")))
    .Append(new HtmlTag("button").Type("submit").Class("btn").Append("Sign in"));
```

Cool! There's still some boilerplate left though. The HtmlBuilders package contains a class called "HtmlTags" (see the trailing 's'?) that contains convenience properties
for just about every non-deprecated HTML element:

```csharp
var fluent = HtmlTags.Div.Class("control-group")
    .Append(HtmlTags.Div.Class("controls")
        .Append(HtmlTags.Label.Class("checkbox")
            .Append(HtmlTags.Input.Checkbox)
            .Append(" Remember me")))
    .Append(HtmlTags.Button.Submit.Class("btn").Append("Sign in"));
```

Now that's neat. Of course, some people still prefer to write the HTML. This is entirely possible, and HtmlBuilders will depend on the [HTML agility pack](http://htmlagilitypack.codeplex.com/) to parse and validate any html as a string

```csharp
var parsed =
    HtmlTag.Parse("<div class='control-group'>" +
                  "<div class='controls'>" +
                  "<label class='checkbox'><input type='checkbox'> Remember me</label>" +
                  "<button type='submit' class='btn'>Sign in</button>" +
                  "</div>" +
                  "</div>");
```

## Part 2: Manipulating the Html

What if you have this:

```html
<label class="checkbox">
    <input type="checkbox" style="width:5px"> Remember me
</label>
```

And you want to make this:

```html
<label class="checkbox" for="remember">
    <input id="remember" style="width:5px;height:10px" type="checkbox"> Remember me
</label>
```

*But here's the catch: you get the object as an incoming parameter in a method!*

Before: TagBuilder madness

```csharp
public TagBuilder AddSomeAttributes(TagBuilder label)
{
    label.MergeAttribute("for", "remember");
    // is there even a better way?
    label.InnerHtml = label.InnerHtml.Replace("input", "input id='remember'");

    // if there is already a style attribute
    if (label.InnerHtml.IndexOf("style", StringComparison.Ordinal) != -1)
    {
        // meh, I give up
    }
    else
    {
        label.InnerHtml = label.InnerHtml.Replace("input", "input style='height:10px'");
    }
    return label;
}
```

Now: HtmlTag glory

```csharp
public HtmlTag AddSomeAttributes(HtmlTag label)
{
    label.Attribute("for", "remember");
    // LINQ search
    var input = label.Children.Single(c => c.TagName.Equals("input") && c.HasAttribute("type") && c["type"].Equals("checkbox"));
    // Immediate access to styles without having to worry about existing styles, correct formatting, etc.
    input.Id("remember").Style("height", "10px");
    // We don't need to update label, any changes made to the input will automatically affect the HTML rendered by label
    return label;
}
```

## Conclusion

The HtmlBuilders package should enable you to create and manipulate HTML in C# easier than before. I especially love the easy access to the children/parents of a tag and the powerful parsing features.
If you decide to try it out or just skim through the code a bit, please don't hesitate to leave a comment or feedback either here or on Github!

Thanks for reading and see you next time!
