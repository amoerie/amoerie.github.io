---
title: "Multilanguage support in C# with code first and the Entity Framework"
date: 2013-01-11T20:43:27+00:00
categories: ["C#"]
tags: ["Entity Framework", "Code First", "Multilanguage", "Localization"]
---

One of the downsides of living in Belgium is that everything needs to be available in multiple languages, given that we're sporting no less than 3 official languages.

This has consequences for developers too, and today I would like to expand a little on how to deal with these consequences.

There is already decent language support in the most popular programming languages, such as resource bundles in Java and resources in .Net. However, a question that often goes unanswered is how to to translate dynamic resources. What if you want to maintain a list of Products where one product can have separate names in English, French, Dutch, etc.
Furthermore, what if you want do the same thing for the product description? What if you want to dynamically add other languages later on?

Most people seem to resort to separating out the properties that need to be translatable, and putting these in a different entity. So you could have a Product class with just the Id and some other properties, while the Name property resides in a 'ProductTranslated' class with a property 'Language' and properties for every translatable Product property. (I'll stop saying the word property now, I promise)

This approach, while logical and simple, has its downsides though. You'll find your number of tables are going to expand rapidly to the point where you might start asking questions about performance and maintainability.

I've been digging into the Entity Framework as of late, and I believe I have found *a* (not the!) solution for this issue: Dictionaries and xml. It makes sense to use dictionaries here, since you want to look up unique values for a product name by language. However, the Entity Framework does not support mapping properties of type IDictionary&lt;TKey, TValue&gt; to a database. (At least, today - 11 January 2013 - it doesn't)
There's a 'workaround' though. Since a dictionary is essentially no more than just an IEnumerable of KeyValuePair&lt;TKey, TValue&gt; we can map this to xml. And xml is something databases *can* handle.

Lets have a look at what we're trying to achieve!

In a dreamworld, I would like to declare my Product class like this:

https://gist.github.com/4513146

Furthermore, I would like to use this class just like I can use a dictionary:

https://gist.github.com/4513169

It should come as no surprise that this is actually possible in C#. Here's the full source code of MultilingualString.cs:

https://gist.github.com/4513218

If you browse through the code a bit, you might notice I've actually implemented IDictionary&lt;string, string&gt; but without specifically saying that where my class declaration is. This is because the Entity Framework doesn't seem to play nice when I do that. (Such a mean bully)
Anyway, stay with me for just another second. As is, this class does little more than decorating a Dictionary&lt;string, string&gt;. The real value however is in the Value property. (1. See what I did there? 2. I said property again. )

You see, whenever you update MultilingualString in any way (remove keys, add or change values, etc.) it will internally update the Value property, which is an xml representation of the dictionary.
The description property of product1 above would translate roughly to this (I've added whitespace for clarity, but usually the Value property will contain no whitespace characters) :

https://gist.github.com/4513266

Sweet! The only thing left to do now is add some mappings so the Entity Framework knows what's up, and you're good to go with Multilingual String properties!

This is how you can trick the Entity Framework into mapping your MultilingualStrings to one column:

https://gist.github.com/4513800

Yay we did it! One reservation to make is that Linq to Entities will NOT like your queries against the database, and it will throw an UnsupportedOperationException when you try to, for example, find all products with a certain English name. You can however solve this easily by using the excellent [LinqKit](http://www.albahari.com/nutshell/linqkit.aspx) library. Just call AsExpandable() on your DbSet&lt;TEntity&gt; and execute Compile() on your expressions. (See LinqKit doc for examples)

If you want to see the full source (including a Test class!) go to https://bitbucket.org/moerie/multilanguagestring

Happy coding!
