---
title: "Automatic injection of models to nested Data Transfer Objects (DTOs) and vice versa with ValueInjecter"
date: 2012-12-22T17:49:27+00:00
categories: ["C#"]
tags: ["ValueInjecter", "DTO", "ASP.NET MVC"]
---

The [DTO pattern](http://en.wikipedia.org/wiki/Data_transfer_object) is often used in multi-layered applications where data is transfered from one layer to another, where -preferably- we keep the data being transfered to a minimum.

One of the issues I've ran into while applying this pattern is the use of nested DTO's and automatic mapping from DTOs to models and vice versa.

Allow me to clarify with an example:

Suppose I have a .NET MVC application where I want do perform some simple CRUD operations on the following domain classes, Person and Address:

https://gist.github.com/4359150

However, I don't want to expose my domain objects to my views, so I'll use DTO's: data transfer objects.

I've created a simple, empty interface called IDto that I'll use as a marker interface for my DTO's: (I'll explain in a second why I'm doing this)

https://gist.github.com/4359214

Here are the DTO implementations for the Person and Address classes above:

https://gist.github.com/4359219

As you can see, I've applied some display and validation logic to these DTO's. In the future, I could add more DTO's to add different validation and display

Now, one of the headaches of using DTOs is mapping. These DTOs contain the same properties (but could also have only a subset!) as the model classes, and before we can send these DTOs off to the view, we'll need to mirror all properties from the model to the DTO. Instead of doing this manually, I'll use the excellent [Value Injecter](http://valueinjecter.codeplex.com/) library. What Value Injecter does really well is looking at the properties of a DTO and a model and matching them by type and name, which allows for automatic value injection. It even supports flattening and unflattening and you can customize the injection process in various ways.

However, one of the things it doesn't support out of the box is nested DTOs. As you can see, the PersonFormDto class contains an Address property of type AddressFormDto. If we try to inject an instance of Person into a PersonFormDto object, ValueInjecter will complain that the type 'AddressFormDto' doesn't match the type 'Address'.

For this reason, I've written a custom class that extends ValueInjecters' LoopValueInjection, with some IDto specific logic. The nice thing about my custom class is that it has no effect whatsoever on other types of properties.
Its custom behavior will only trigger if a class is found which implements IDto.

https://gist.github.com/4360044

All this class really does is allow mathing types of Address - AddressFormDto and Person - PersonFormDto, and any future Dtos we might make. I also validate if AddressFormDto implements IDto&lt;Address&gt;, if not the injection will fail.

Now there's only one more step between this and painless automagical mapping between models and dtos: adding a few extensions methods. I'm quite fond of C#'s ability to add methods to classes you haven't written yourself, as they allow for a nice API. These are my extension methods for IDto:

https://gist.github.com/4360096

Why go through the trouble of writing these extension methods you ask? Well, first of all, like I've said, they allow for a very nice API which I'll show soon. Secondly, having my injection logic in one place allows me to refactor it more easily. It would even allow me to replace the ValueInjecter with a completely different library altogether! Controllers don't need to know how dto's and models are mapped, so these extensions methods are a nice way to abstract that away.

So now, for the finale, this is some sample code of how I would write my Controller methods using the classes I've described above:

https://gist.github.com/4360130

So there you have it. Seriously simplified controller code. There are some reservations to be made though: I've read that ValueInjecter doesn't support IEnumerable properties out of the box. I'm still looking for a way to update my custom DtoInjection class to support this in the SetValue method. I know it can be done manually, but I do prefer automagical stuff. :-)

Happy holidays!
