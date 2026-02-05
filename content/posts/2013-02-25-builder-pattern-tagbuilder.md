---
title: "Applying the builder pattern to the .Net MVC TagBuilder"
date: 2013-02-25T20:51:07+00:00
categories: ["C#"]
tags: [".Net Mvc", "Builder Pattern", "c#", "Nitpicking", "Web Development"]
---

Sometimes I'm a nitpicker. Actually I'm a nitpicker 24/7, but I can't really say that on a blog, can I?

So today's nitpicking is about C#'s TagBuilder. This class allows you to build html without writing a single '&lt;' or '&gt;'.  Using this class usually boils down to something like this:

https://gist.github.com/amoerie/5033068

However, wouldn't it be nicer if we could say:

https://gist.github.com/amoerie/5033091

It's not a *huge* difference, but all the actions are chainable, provide a nice and consistent API and follow the Builder pattern.
Hey, if you're going to call your class the TagBUILDER it's the least you can do. As hard as I tried, I couldn't restrain myself and did the unthinkable: I wrote extension methods.
As it turns out, it's far from a hurdle to implement this kind of thing:

https://gist.github.com/amoerie/5033136

This is one of the things I really love about C#: you can roll your own API over existing classes as if they were there in the first place.
Using this builder pattern should make your HtmlHelpers much more readable. :-)

See you next time!
