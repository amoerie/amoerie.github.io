---
title: "A generic approach to data tables with server-side processing"
date: 2013-03-08T21:22:07+00:00
categories: ["C#"]
tags: ["jQuery Datatables", "ASP.NET MVC", "Server-side processing"]
---

A little library that provides a generic way to use jQuery Datatables with ASP.Net MVC 4

**Update to future readers: this code is no longer maintained! Use at your own risk, the Github repository should be seen as a pointer on how this can be done, not as a drop-in library to consume.**

This is going to be a long post. Don't say I didn't warn you!

If you're not one for long texts (but why are you reading blogs then?), today's post in the 'who-needs-hobbies-anyway' series is all about https://github.com/amoerie/generic-datatables

Now, for the matter at hand: data tables. One of the recurring themes  (of which there are many, actually) in web application development is producing tables for some class. You probably recognize this scenario: 'We need to manage instances of x, so the website should display them in a table that supports pagination, ordering by columns and searching by columns or a global search. And that's just displaying. Ideally we want to add new records and edit or delete existing ones. And if it's at all possible, we don't even want to leave the page for any of these things. Oh, and if it works slow, we'll complain. A lot. You have 1 day to make this.

Next week you get a call that they need to manage instances of 'y' too, with all the same requirements. Basically a rinse and repeat for a different object with different properties, but in a repetitive fashion nonetheless.

I believe developers across the globe have shed tears in despair over this, because it's actually rather hard to 'template' this or provide a quick way to build these kind of things, unless you've written some meta-module that produces code for you. (In which case : tsk tsk tsk, code generators, really? Where's your pride?)

Well, after this needlessly long introduction, I can come to the point of this blog post: I've tried building a generic module that provides a solid structure to build datatables upon. I wish I could say you can just drag and drop the library in your project and it Santa Claus and his reindeers will make your code automagically work with it, but there is some work to be done.

Let me just explain what my goals were when I started developing this:

1. I only want to write code that is specific to the instance type 'x' or 'y' or whatever
2. Pagination, sorting and filtering should all be pushed down the pipe to generated SQL statements, for maximum performance. Minimize in-memory processing.
3. I still want full control over the html and javascript, without any javascript generation

So what are the results?

Here's a visual first:

![People table](http://i.imgur.com/u3dbkyA.png)

This table fetches data from two different database tables: 'Person' and 'Address'. Please ignore the fact that 'Time' does not make sense as a property of Person, I just wanted a Timespan property to fiddle with. :-)

Now, to explain how you make this work (in a .Net MVC 4 application using Entity Framework 5 with SQL Server 2012), these are the steps:

## How does it work?

### Step 1: The html

The idea is simple: create a datatable, give it a unique name (There will be a session variable called 'datatables' that contains a dictionary of datatable configurations. In this dictionary, configurations will be saved with this names as the key)
and configure the properties that should be shown. Currently supported datatypes include int?, decimal?, double?, DateTime?, Timespan? and of course their non-nullable equivalents.
You need to give the properties names which should correspond with the javascript mDataProps (see Step 4). Don't worry, I don't use these for anything else so you can actually name them whatever you want.

Lastly, you can add a last column with custom html. The best way I've found to pass this in is through the use of a razor helper, but you could technically also use a partial.

The signature of the LastColumn() function is:

https://gist.github.com/amoerie/5119920

Anyway, this is how the HtmlHelper works:

https://gist.github.com/amoerie/5119865

### Step 2: The model binder

Just like most other attempts at making a generic server-side wrapper for datatables, I've provided a C# class to map the incoming ajax requests and a model binder that does the heavy lifting for you.
All you need to do is register the model binder, put the following somewhere in Global.asax (or better, in a separate ModelBindersConfig in the App Start folder):

https://gist.github.com/amoerie/5119912

### Step 3: The Controller

The idea of server-side processing with a datatable is that your MVC Controller accepts the incoming ajax requests, parses it, fetches the data from somewhere and applies the filtering, sorting, etc. specified in the request object.
However, because the library does most of the heavy work, you only have to glue some specific stuff together. In the PeopleController:

https://gist.github.com/amoerie/5119894

### Step 4: The Javascript

Somewhere along the way I considered generating the necessary javascript with the HtmlHelper as well. I wanted to keep full control of the javascript though, so in the end I decided against it.
This means you need to write the following javascript to get the datatable to actually do something:

https://gist.github.com/amoerie/5119908

### That's it!

If the stars align, it should work. Be sure to check out the demo page in the source (Views/People/Index).
There I've added a search filter per property, applied the Twitter Bootstrap theme and added add/edit/delete functionalities.
I also smacked the show/hide columns plugin on it, and also the infinite scrolling plugin (although it's not enabled by default)
I should make a simple, basic demo page someday. Shoot me a mail and maybe I will. :-)

That's it for today's post. Making this was definitely a fun exercise in Expression trees, and I like to think I've grown again as a developer. It's times like this where I whisper to myself that one day I'll be one of the greats, while I stare in a challenging manner at the big John Skeet poster on my wall. I don't really have a big John Skeet poster though. (...do they make those?)
