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

```csharp
public class Person
{
    public int Id { get; set; }
    public string Name { get; set; }
    public DateTime Birthday { get; set; }
    public int AddressId { get; set; }
    public virtual Address Address { get; set; }
}

public class Address
{
    public int Id { get; set; }
    public string Street { get; set; }
    public string HouseNumber { get; set; }
    public string PostalCode { get; set; }
    public string City { get; set; }
}
```

However, I don't want to expose my domain objects to my views, so I'll use DTO's: data transfer objects.

I've created a simple, empty interface called IDto that I'll use as a marker interface for my DTO's: (I'll explain in a second why I'm doing this)

```csharp
public interface IDto<TModel> where TModel : class, new()
{

}
```

Here are the DTO implementations for the Person and Address classes above:

```csharp
public class PersonFormDto: IDto<Person>
{
    public int Id { get; set; }

    [Required(ErrorMessage = "Hey why don't you tell me who you are?!")]
    public string Name { get; set; }

    [DisplayFormat(DataFormatString = "{0:dd/MM/yyyy}", ApplyFormatInEditMode = true)]
    [Required(ErrorMessage = "Come on now, don't be shy.")]
    public DateTime Birthday { get; set; }

    public AddressFormDto Address { get; set; }
}

public class AddressFormDto: IDto<Address>
{
    public int Id { get; set; }

    [Display(Name = "Street")]
    [Required(ErrorMessage = "You don't have a street?")]
    public string Street { get; set; }

    [Display(Name = "House Nr.")]
    [Required(ErrorMessage = "What's your house number?")]
    public string HouseNumber { get; set; }

    [Display(Name = "Postal code")]
    [Required(ErrorMessage = "Ahum")]
    public string PostalCode { get; set; }

    [Display(Name = "City")]
    [Required(ErrorMessage = "Hey don't forget to fill in your city")]
    public string City { get; set; }
}
```

As you can see, I've applied some display and validation logic to these DTO's. In the future, I could add more DTO's to add different validation and display

Now, one of the headaches of using DTOs is mapping. These DTOs contain the same properties (but could also have only a subset!) as the model classes, and before we can send these DTOs off to the view, we'll need to mirror all properties from the model to the DTO. Instead of doing this manually, I'll use the excellent [Value Injecter](http://valueinjecter.codeplex.com/) library. What Value Injecter does really well is looking at the properties of a DTO and a model and matching them by type and name, which allows for automatic value injection. It even supports flattening and unflattening and you can customize the injection process in various ways.

However, one of the things it doesn't support out of the box is nested DTOs. As you can see, the PersonFormDto class contains an Address property of type AddressFormDto. If we try to inject an instance of Person into a PersonFormDto object, ValueInjecter will complain that the type 'AddressFormDto' doesn't match the type 'Address'.

For this reason, I've written a custom class that extends ValueInjecters' LoopValueInjection, with some IDto specific logic. The nice thing about my custom class is that it has no effect whatsoever on other types of properties.
Its custom behavior will only trigger if a class is found which implements IDto.

```csharp
/// <summary>
/// Custom injection to support nested DTO properties
/// </summary>
public class DtoInjection: LoopValueInjection
{
    /// <summary>
    /// If one of the types implements IDto, check if its generic type parameter
    /// matches the other type
    /// Otherwise, just use the default typematcher.
    /// </summary>
    /// <param name="sourceType"></param>
    /// <param name="targetType"></param>
    /// <returns></returns>
    protected override bool TypesMatch(Type sourceType, Type targetType)
    {
        if (IsDtoAndModelType(sourceType, targetType) || IsDtoAndModelType(targetType, sourceType))
            return true;
        return base.TypesMatch(sourceType, targetType);
    }

    /// <summary>
    /// If the target property or source property is a DTO, we'll need to recursively use 
    /// this class to set the value
    /// </summary>
    /// <param name="v"></param>
    /// <returns></returns>
    protected override object SetValue(object v)
    {
        if (IsDtoAndModelType(SourcePropType, TargetPropType) || IsDtoAndModelType(TargetPropType, SourcePropType))
        {
            var target = Activator.CreateInstance(TargetPropType);
            if(v != null)
                target.InjectFrom<DtoInjection>(v);
            return target;
        }
        return base.SetValue(v);
    }

    /// <summary>
    /// Returns true if the dtoType implements IDto with generic type modelType
    /// </summary>
    /// <param name="dtoType"></param>
    /// <param name="modelType"></param>
    /// <returns></returns>
    private bool IsDtoAndModelType(Type dtoType, Type modelType)
    {
        // get DTO interface from dtoType
        var dtoInterface = dtoType.GetInterfaces()
            .FirstOrDefault(i => i.IsGenericType && i.GetGenericTypeDefinition() == typeof(IDto<>));

        // if dtoType doesn't implement IDto, return false
        if (dtoInterface == null)
            return false;

        // Get the generic type argument and check if it matches the modeltype
        var dtoModelType = dtoInterface.GetGenericArguments().First();
        return dtoModelType == modelType;
    }
}
```

All this class really does is allow mathing types of Address - AddressFormDto and Person - PersonFormDto, and any future Dtos we might make. I also validate if AddressFormDto implements IDto&lt;Address&gt;, if not the injection will fail.

Now there's only one more step between this and painless automagical mapping between models and dtos: adding a few extensions methods. I'm quite fond of C#'s ability to add methods to classes you haven't written yourself, as they allow for a nice API. These are my extension methods for IDto:

```csharp
public static class DtoExtensions
{
    /// <summary>
    /// Make a new model and inject dto values
    /// </summary>
    /// <typeparam name="TModel">The type of the model</typeparam>
    /// <param name="dto">The dto object that contains the property values</param>
    /// <returns>A new instance of TModel with the same property values of this dto</returns>
    public static TModel ToModel<TModel>(this IDto<TModel> dto)
        where TModel : class, new()
    {
        var model = new TModel();
        dto.UpdateModel(model);
        return model;
    }

    /// <summary>
    /// Updates an existing model with the property values of the dto
    /// </summary>
    /// <typeparam name="TModel">The type of the model</typeparam>
    /// <param name="dto">The dto object that will provide the property values</param>
    /// <param name="model">The model object that will take the property values</param>
    public static void UpdateModel<TModel>(this IDto<TModel> dto, TModel model)
        where TModel : class, new()
    {
        model.InjectFrom<DtoInjection>(dto);
    }

    /// <summary>
    /// Injects the properties of the given model into this dto
    /// </summary>
    /// <typeparam name="TDto">The type of the DTO, which should implement IDto of type TModel</typeparam>
    /// <typeparam name="TModel">The type of the model</typeparam>
    /// <param name="dto">The dto that will take the property values</param>
    /// <param name="model">The model that provides the property values</param>
    /// <returns>This dto with its properties injected by the model</returns>
    public static TDto FromModel<TDto, TModel>(this TDto dto, TModel model)
        where TDto : IDto<TModel>
        where TModel : class, new()
    {
        dto.InjectFrom<DtoInjection>(model);
        return dto;
    } 
}
```

Why go through the trouble of writing these extension methods you ask? Well, first of all, like I've said, they allow for a very nice API which I'll show soon. Secondly, having my injection logic in one place allows me to refactor it more easily. It would even allow me to replace the ValueInjecter with a completely different library altogether! Controllers don't need to know how dto's and models are mapped, so these extensions methods are a nice way to abstract that away.

So now, for the finale, this is some sample code of how I would write my Controller methods using the classes I've described above:

```csharp
public class PeopleController: Controller
{
    private IRepository<Person> _people;

    public PeopleController(IRepository<Person> people)
    {
        _people = people;
    }

    public ActionResult Index()
    {
        return View();
    }

    public ActionResult PersonForm(int? id)
    {
        Person person = id.HasValue ? _people.Find(id.Value) : new Person();
        var personFormDto = new PersonFormDto().FromModel(person);
        return View(personFormDto);
    }

    [HttpPost]
    public ActionResult PersonForm(int id, PersonFormDto personFormDto)
    {
        if (ModelState.IsValid)
        {
            bool isNew = id == 0;
            if (isNew)
            {
                // make model object from the dto
                var person = personFormDto.ToModel();

                // add to db
                _people.Add(person);
            }
            else
            {
                // fetch from db
                var person = _people.Find(id);

                // update with dto values
                personFormDto.UpdateModel(person);

                 _people.Update(person);
            }
               
            _people.SaveChanges();
            return RedirectToAction("Index");
        }
        return View(personFormDto);
    }
}
```

So there you have it. Seriously simplified controller code. There are some reservations to be made though: I've read that ValueInjecter doesn't support IEnumerable properties out of the box. I'm still looking for a way to update my custom DtoInjection class to support this in the SetValue method. I know it can be done manually, but I do prefer automagical stuff. :-)

Happy holidays!
