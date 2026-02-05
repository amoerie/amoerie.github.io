---
title: "Replacing method calls with expressions at runtime with Linqkit and the Entity Framework"
date: 2013-03-09T14:06:08+00:00
categories: ["C#"]
tags: ["c#", "Dark magic", "Entity Framework", "Expressions", "Lambda Expressions", "Linq", "Linq to Entities", "LinqKit"]
---

> LINQ to Entities does not recognize the method '...' method, and this method cannot be translated into a store expression.

Does this sound familiar to you? It's a common issue that many developers face, as evidenced by the astronomical amount of questions about it on [stackoverflow.com](http://stackoverflow.com/search?q=LINQ+to+Entities+does+not+recognize+the+method)

## The problem

To understand why this happens, you must first realize that Linq To Entities - the thing that produces the SQL statements for you - never really executes your C# code. So when you write:

https://gist.github.com/amoerie/5124034

there is never a point in time where the entire 'People' table is loaded from your database into memory, and then parsed in-memory. Rather, Linq to Entities will parse your expression by 'reading it' (and not executing it!) and produce equivalent SQL statements. The above C# code would roughly produce:

https://gist.github.com/amoerie/5124038

This is a rather brilliant system, and a very performant one too, but as with any system it has its limitations. And that's what today's post is all about: you can't call methods in your LINQ statements if they need to be translated to valid SQL statements. One solution that you'll see come up often on stackoverflow is to just call 'ToList()' or 'AsEnumerable()', which loads the entities in memory and makes your SQL translation woes go away.

While this is indeed a failsafe solution, sometimes it's not an option to load en entire table in memory. Sometimes you just want some reusable statements to be a part of your LINQ statements. And you know that if you wrote them manually each time, Linq to Entities would have no issue translating it to SQL, but because you've chosen to encapsulate the logic in methods (which is good! [DRY](http://en.wikipedia.org/wiki/Don't_repeat_yourself)!) they are rendered untranslatable.

Don't blame Linq to Entities though, right now there is no way to get the contents of a method as 'readable expressions', so it's only normal it throws the towel in the ring when it catches you calling methods it doesn't recognize. However, there *is* a way to make this work. I'm not saying anyone should do this, but there is a way and I have a working implementation of it.

I'll walk you through it, and at the end you can *decide for yourself* wether this is something you want to do or not. First, let's define our goal.

## The goal

We want to be able to call a method inside a [Lambda Expression](http://msdn.microsoft.com/en-us/library/bb397687.aspx) and have it produce valid SQL statements. I've setup a little sample application where we want to manage our Products, where each Product has a Name. An important functional requirement is also that our application will support many languages, and we want our products to have different names per language. I've setup the models like this:

https://gist.github.com/amoerie/5124060

Simply put, we have a Product class with many ProductTranslations. ITranslatable and ITranslation are interfaces that allow me to write extension methods for additional functionality should I need it later on, without imposing any inheritance constraints. For example, I have the following extension method for ITranslatable, which returns the appropriate ITranslation instance of an ITranslatable instance depending on the current default CultureInfo:

https://gist.github.com/amoerie/5124085

Now, I'd very much like to write C# code like this, which should produce valid SQL statements:

https://gist.github.com/amoerie/5124090

Unfortunately, Linq to Entities does not recognize the 'Current()' method and will fail to provide any meaningful SQL statements.

## A solution

Let's start with the essence of the problem: Linq to Entities is great at reading [Expressions](http://msdn.microsoft.com/en-us/library/bb397951.aspx) and converting that to SQL. The only two limitations are that only expressions are supported that can be translated to TSQL and that it can only translate what you pass it. It can't fetch Expression trees from method calls, which would require decompiling CLR code... and let's not go there.

You can't even store your expression in a temporary variable

### Step 1: LinqKit and expression expanding

The first step to reaching our goal is using [LinqKit](http://www.albahari.com/nutshell/linqkit.aspx). This neat little Library allows you declare expressions in variables and use them in subqueries them in your Linq statements. It uses a really neat trick to do this: it looks for references to expressions in your expression tree and automatically 'expands' this, as if you had written the full expression in the first place. This is actually very similar to the substitution model that programming languages use to run your code. [Martin Odersky](http://en.wikipedia.org/wiki/Martin_Odersky), founder of the excellent [Scala](http://www.scala-lang.org/) language, has an excellent and very eloquent [explanation on this subject in his Coursera course here](https://class.coursera.org/progfun-2012-001/lecture/3). The general idea is that you replace calls to methods, variables, etc. with their actual content at runtime.

This is actually very similar to what we're trying to achieve: replacing the call to 'Current()' with a valid Expression tree at runtime. Let's write a preliminary version of what we want by using Linqkit:

https://gist.github.com/amoerie/5124148

As you can see, we no longer need to write the code for 'current' each time anymore, which is great. However, in any real-world application you're probably composing LINQ queries all over the place and it would quickly become a pain to pass this variable around. What we really want is that a call to Current() would be replaced at runtime with the expression from the previous code snippet...

### Step 2: Extending LinqKit

Extending a rather complex library? Yes we can! *ducks under flying tomato*

The idea is that we want to do what LinqKit is doing, dynamically expanding/inserting expressions where appropriate, but with some custom logic. So download that LinqKit source and off we go. :-)

The first thing we'll do is define an interface for a class that can replace expressions at runtime:

https://gist.github.com/amoerie/5124165

Any class implementing this interface should:

- tell whether this class has any replacements to make for a given expression
- return a new expression where any matched patterns were replaced with the appropriate expressions

Let's create one for our 'Current()' method!

https://gist.github.com/amoerie/5124184

Difficult? Yes. Maybe.
We're essentially extracting some type information from the MethodCallExpression and using that to create a dynamic call to the MakeCurrentExpression method, which needs a TEntity and TTranslation type to function.

We can now extend LinqKit's ExpressionExpander to inject our replacing rules:

https://gist.github.com/amoerie/5124191

Note that I've only added method overrides for VisitMethodCall. There are plenty of other overriding possibilities if you want to dynamically replace property accessors, etc. However, I found this blog post to be complex enough as it is already. :-)

### Step 3: Roll your own ExpressionExpander

The last thing to do now is to change the LinqKit AsExpandable methods so they return our custom expander instead of the default one:

https://gist.github.com/amoerie/5124195

### Step 4: Bask in glory

Now, when we open the index page for products, for a culture "nl-BE" and filter by name contains "a", like so:

https://gist.github.com/amoerie/5124090

This produces the following SQL statements:

https://gist.github.com/amoerie/5124210

### Step 5: Use the source

Now, I know this was all boring, long and still too fast to decently explain everything that's going on. That's why you can go to [github](https://github.com/amoerie/Translations.Web/) and see all the code in one place.

## Things to consider

As a developer, sometimes you can feel it when something just doesn't sit right when you write code. I couldn't shake that feeling while I was writing this code.
I have some issues:

- You are doing dark magic behind the scenes, replacing method calls with arbitrary expressions. Note that these expressions don't have to match the implementation of that method you called. The actual implementation of the method could even throw NotImplementException and it would still work.

- I haven't experimented enough to safely say this has no side effects. However, it shouldn't interfere with anything else as long as the implementations of your 'ShouldReplace' methods are strict enough.

- Future updates of LinqKit (although it hasn't been updated in a while) could break your code.

- Future updates of Entity Framework could break your code. I don't immediately see how this would happen, but never say never.

## It's still pretty cool though

In my humble opinion, this is a solid argument for the power of C#. Today we replaced a method call in a LINQ statement with another arbitrary expression, which is then translated into valid SQL statements. This all happens behind the scenes, without the need for anyone knowing about it. Whether this is a good or bad thing is debatable (and honestly it's definitely at least a bit bad) but it's still pretty cool that it can be done though.

That's it for today, if you've read it all through the end, congratulations!
