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

```csharp
public class Product
{
    public int Id { get; set; }
    public MultilingualString Name { get; set; }
    public MultilingualString Description { get; set; }
}
```

Furthermore, I would like to use this class just like I can use a dictionary:

```csharp
var product1 = new Product();
product1.Name["en"] = "Product 1";
product1.Description["fr"] = "Description 1";
product1.Name["fr"] = "Produit 1";
product1.Description["fr"] = "Déscription 1";
product1.Name["nl"] = "Produkt 1";
product1.Description["nl"] = "Omschrijving 1";
```

It should come as no surprise that this is actually possible in C#. Here's the full source code of MultilingualString.cs:

```csharp
/// <summary>
/// Represents a string that can be translated into several languages.
/// </summary>
public class MultilingualString
{
    private const string TranslationsNode = "translations";
    private const string TranslationNode = "translation";
    private const string LanguageNode = "language";
    private const string ValueNode = "value";

    private IDictionary<string, string> _translations;

    #region Properties

    /// <summary>
    /// Gets or sets the serialized XML of all <see cref="Translations"/>
    /// </summary>
    public string Value { get; set; }

    private IDictionary<string, string> Translations
    {
        get { return _translations ?? (_translations = FromXml(Value)); }
    }

    /// <summary>
    /// Gets or sets the element with the specified key.
    /// </summary>
    /// <returns>
    /// The element with the specified key or an empty string if the key was not found
    /// </returns>
    /// <param name="key">The key of the element to get or set.</param>
    /// <exception cref="T:System.ArgumentNullException"><paramref name="key"/> is null.</exception>
    /// <exception cref="T:System.Collections.Generic.KeyNotFoundException">The property is retrieved and <paramref name="key"/> is not found.</exception>
    /// <exception cref="T:System.NotSupportedException">The property is set and the <see cref="T:System.Collections.Generic.IDictionary`2"/> is read-only.</exception>
    public string this[string key]
    {
        get
        {
            return ContainsKey(key) ? Translations[key] : string.Empty;
        }
        set
        {
            Translations[key] = value;
            UpdateValue();
        }
    }

    /// <summary>
    /// Gets an <see cref="T:System.Collections.Generic.ICollection`1"/> containing the keys of the <see cref="T:System.Collections.Generic.IDictionary`2"/>.
    /// </summary>
    /// <returns>
    /// An <see cref="T:System.Collections.Generic.ICollection`1"/> containing the keys of the object that implements <see cref="T:System.Collections.Generic.IDictionary`2"/>.
    /// </returns>
    public ICollection<string> Keys
    {
        get { return Translations.Keys; }
    }

    /// <summary>
    /// Gets an <see cref="T:System.Collections.Generic.ICollection`1"/> containing the values in the <see cref="T:System.Collections.Generic.IDictionary`2"/>.
    /// </summary>
    /// <returns>
    /// An <see cref="T:System.Collections.Generic.ICollection`1"/> containing the values in the object that implements <see cref="T:System.Collections.Generic.IDictionary`2"/>.
    /// </returns>
    public ICollection<string> Values
    {
        get { return Translations.Values; }
    }

    /// <summary>
    /// Gets the number of elements contained in the <see cref="T:System.Collections.Generic.ICollection`1"/>.
    /// </summary>
    /// <returns>
    /// The number of elements contained in the <see cref="T:System.Collections.Generic.ICollection`1"/>.
    /// </returns>
    public int Count
    {
        get { return Translations.Count; }
    }

    /// <summary>
    /// Gets a value indicating whether the <see cref="T:System.Collections.Generic.ICollection`1"/> is read-only.
    /// </summary>
    /// <returns>
    /// true if the <see cref="T:System.Collections.Generic.ICollection`1"/> is read-only; otherwise, false.
    /// </returns>
    public bool IsReadOnly
    {
        get { return Translations.IsReadOnly; }
    }

    #endregion

    /// <summary>
    /// Returns an enumerator that iterates through the collection.
    /// </summary>
    /// <returns>
    /// A <see cref="T:System.Collections.Generic.IEnumerator`1"/> that can be used to iterate through the collection.
    /// </returns>
    public IEnumerator<KeyValuePair<string, string>> GetEnumerator()
    {
        return _translations.GetEnumerator();
    }

    /// <summary>
    /// Adds an item to the <see cref="T:System.Collections.Generic.ICollection`1"/>.
    /// </summary>
    /// <param name="item">The object to add to the <see cref="T:System.Collections.Generic.ICollection`1"/>.</param>
    /// <exception cref="T:System.NotSupportedException">The <see cref="T:System.Collections.Generic.ICollection`1"/> is read-only.</exception>
    public void Add(KeyValuePair<string, string> item)
    {
        Translations.Add(item);
        UpdateValue();
    }

    /// <summary>
    /// Removes all items from the <see cref="T:System.Collections.Generic.ICollection`1"/>.
    /// </summary>
    /// <exception cref="T:System.NotSupportedException">The <see cref="T:System.Collections.Generic.ICollection`1"/> is read-only. </exception>
    public void Clear()
    {
        Translations.Clear();
        UpdateValue();
    }

    /// <summary>
    /// Determines whether the <see cref="T:System.Collections.Generic.ICollection`1"/> contains a specific value.
    /// </summary>
    /// <returns>
    /// true if <paramref name="item"/> is found in the <see cref="T:System.Collections.Generic.ICollection`1"/>; otherwise, false.
    /// </returns>
    /// <param name="item">The object to locate in the <see cref="T:System.Collections.Generic.ICollection`1"/>.</param>
    public bool Contains(KeyValuePair<string, string> item)
    {
        return Translations.Contains(item);
    }

    /// <summary>
    /// Determines whether the <see cref="T:System.Collections.Generic.ICollection`1"/> contains a specific language.
    /// </summary>
    /// <returns>
    /// true if <paramref name="language"/> is found in the <see cref="T:System.Collections.Generic.ICollection`1"/>; otherwise, false.
    /// </returns>
    /// <param name="language">The object to locate in the <see cref="T:System.Collections.Generic.ICollection`1"/>.</param>
    public bool ContainsLanguage(string language)
    {
        return ContainsKey(language);
    }

    /// <summary>
    /// Copies the elements of the <see cref="T:System.Collections.Generic.ICollection`1"/> to an <see cref="T:System.Array"/>, starting at a particular <see cref="T:System.Array"/> index.
    /// </summary>
    /// <param name="array">The one-dimensional <see cref="T:System.Array"/> that is the destination of the elements copied from <see cref="T:System.Collections.Generic.ICollection`1"/>. The <see cref="T:System.Array"/> must have zero-based indexing.</param>
    /// <param name="arrayIndex">The zero-based index in <paramref name="array"/> at which copying begins.</param>
    /// <exception cref="T:System.ArgumentNullException"><paramref name="array"/> is null.</exception>
    /// <exception cref="T:System.ArgumentOutOfRangeException"><paramref name="arrayIndex"/> is less than 0.</exception>
    /// <exception cref="T:System.ArgumentException">The number of elements in the source <see cref="T:System.Collections.Generic.ICollection`1"/> is greater than the available space from <paramref name="arrayIndex"/> to the end of the destination <paramref name="array"/>.</exception>
    public void CopyTo(KeyValuePair<string, string>[] array, int arrayIndex)
    {
        Translations.CopyTo(array, arrayIndex);
    }

    /// <summary>
    /// Removes the first occurrence of a specific object from the <see cref="T:System.Collections.Generic.ICollection`1"/>.
    /// </summary>
    /// <returns>
    /// true if <paramref name="item"/> was successfully removed from the <see cref="T:System.Collections.Generic.ICollection`1"/>; otherwise, false. This method also returns false if <paramref name="item"/> is not found in the original <see cref="T:System.Collections.Generic.ICollection`1"/>.
    /// </returns>
    /// <param name="item">The object to remove from the <see cref="T:System.Collections.Generic.ICollection`1"/>.</param>
    /// <exception cref="T:System.NotSupportedException">The <see cref="T:System.Collections.Generic.ICollection`1"/> is read-only.</exception>
    public bool Remove(KeyValuePair<string, string> item)
    {
        bool removed = Translations.Remove(item);
        UpdateValue();
        return removed;
    }

    /// <summary>
    /// Determines whether the <see cref="T:System.Collections.Generic.IDictionary`2"/> contains an element with the specified key.
    /// </summary>
    /// <returns>
    /// true if the <see cref="T:System.Collections.Generic.IDictionary`2"/> contains an element with the key; otherwise, false.
    /// </returns>
    /// <param name="key">The key to locate in the <see cref="T:System.Collections.Generic.IDictionary`2"/>.</param>
    /// <exception cref="T:System.ArgumentNullException"><paramref name="key"/> is null.</exception>
    public bool ContainsKey(string key)
    {
        return Translations.ContainsKey(key);
    }

    /// <summary>
    /// Adds an element with the provided key and value to the <see cref="T:System.Collections.Generic.IDictionary`2"/>.
    /// </summary>
    /// <param name="key">The object to use as the key of the element to add.</param>
    /// <param name="value">The object to use as the value of the element to add.</param>
    /// <exception cref="T:System.ArgumentNullException"><paramref name="key"/> is null.</exception>
    /// <exception cref="T:System.ArgumentException">An element with the same key already exists in the <see cref="T:System.Collections.Generic.IDictionary`2"/>.</exception>
    /// <exception cref="T:System.NotSupportedException">The <see cref="T:System.Collections.Generic.IDictionary`2"/> is read-only.</exception>
    public void Add(string key, string value)
    {
        Translations.Add(key, value);
        UpdateValue();
    }

    /// <summary>
    /// Removes the element with the specified key from the <see cref="T:System.Collections.Generic.IDictionary`2"/>.
    /// </summary>
    /// <returns>
    /// true if the element is successfully removed; otherwise, false.  This method also returns false if <paramref name="key"/> was not found in the original <see cref="T:System.Collections.Generic.IDictionary`2"/>.
    /// </returns>
    /// <param name="key">The key of the element to remove.</param>
    /// <exception cref="T:System.ArgumentNullException"><paramref name="key"/> is null.</exception>
    /// <exception cref="T:System.NotSupportedException">The <see cref="T:System.Collections.Generic.IDictionary`2"/> is read-only.</exception>
    public bool Remove(string key)
    {
        bool removed = Translations.Remove(key);
        UpdateValue();
        return removed;
    }

    /// <summary>
    /// Gets the value associated with the specified key.
    /// </summary>
    /// <returns>
    /// true if the object that implements <see cref="T:System.Collections.Generic.IDictionary`2"/> contains an element with the specified key; otherwise, false.
    /// </returns>
    /// <param name="key">The key whose value to get.</param>
    /// <param name="value">When this method returns, the value associated with the specified key, if the key is found; otherwise, the default value for the type of the <paramref name="value"/> parameter. This parameter is passed uninitialized.</param>
    /// <exception cref="T:System.ArgumentNullException"><paramref name="key"/> is null.</exception>
    public bool TryGetValue(string key, out string value)
    {
        return Translations.TryGetValue(key, out value);
    }

    private void UpdateValue()
    {
        Value = ToXml(_translations);
    }

    private IDictionary<string, string> FromXml(string value)
    {
        if (value == null)
            return new Dictionary<string, string>();
        return
            XElement.Parse(value)
                    .Elements()
                    .ToDictionary(t =>
                        {
                            var languageContent = t.Element(LanguageNode);
                            Debug.Assert(languageContent != null, "Could not load language in " + this);
                            return languageContent.Value;
                        },
                        t =>
                        {
                            var valueContent = t.Element(ValueNode);
                            Debug.Assert(valueContent != null, "Could not load value in " + this);
                            return valueContent.Value;
                        });
    }

    private string ToXml(IEnumerable<KeyValuePair<string, string>> translations)
    {
        var translationsXml = new XElement(TranslationsNode);
        foreach (var translationXml in from keyValuePair in translations let keyXml = new XElement(LanguageNode, keyValuePair.Key) let valueXml = new XElement(ValueNode, keyValuePair.Value) select new XElement(TranslationNode, keyXml, valueXml))
        {
            translationsXml.Add(translationXml);
        }
        return translationsXml.ToString(SaveOptions.DisableFormatting);
    }

    /// <summary>
    /// Returns a string that represents the current object.
    /// </summary>
    /// <returns>
    /// A string that represents the current object.
    /// </returns>
    public override string ToString()
    {
        return string.Format("Translations: {0}, Value: {1}", _translations, Value);
    }
}
```

If you browse through the code a bit, you might notice I've actually implemented IDictionary&lt;string, string&gt; but without specifically saying that where my class declaration is. This is because the Entity Framework doesn't seem to play nice when I do that. (Such a mean bully)
Anyway, stay with me for just another second. As is, this class does little more than decorating a Dictionary&lt;string, string&gt;. The real value however is in the Value property. (1. See what I did there? 2. I said property again. )

You see, whenever you update MultilingualString in any way (remove keys, add or change values, etc.) it will internally update the Value property, which is an xml representation of the dictionary.
The description property of product1 above would translate roughly to this (I've added whitespace for clarity, but usually the Value property will contain no whitespace characters) :

```xml
<translations>

  <translation>
    <language>en</language>
    <value>Description 1</value>
  </translation>

  <translation>
    <language>fr</language>
    <value>Déscription 1</value>
  </translation>
  
  <translation>
    <language>nl</language>
    <value>Omschrijving 1</value>
  </translation>
    
</translations>
```

Sweet! The only thing left to do now is add some mappings so the Entity Framework knows what's up, and you're good to go with Multilingual String properties!

This is how you can trick the Entity Framework into mapping your MultilingualStrings to one column:

```csharp
public class Product
{
    private MultilingualString _description;
    private MultilingualString _name;

    public int Id { get; set; }

    public virtual string DescriptionXml
    {
        get { return Description.Value; }
        set { Description.Value = value; }
    }

    public virtual string NameXml
    {
        get { return Name.Value; }
        set { Name.Value = value; }
    }

    [NotMapped] // don't map MultilingualStrings. EF will eat your guts.
    public virtual MultilingualString Description // added backing property to avoid null
    {
        get { return _description ?? (_description = new MultilingualString()); }
        set { _description = value; }
    }

    [NotMapped]
    public virtual MultilingualString Name
    {
        get { return _name ?? (_name = new MultilingualString()); }
        set { _name = value; }
    }
}
```

Yay we did it! One reservation to make is that Linq to Entities will NOT like your queries against the database, and it will throw an UnsupportedOperationException when you try to, for example, find all products with a certain English name. You can however solve this easily by using the excellent [LinqKit](http://www.albahari.com/nutshell/linqkit.aspx) library. Just call AsExpandable() on your DbSet&lt;TEntity&gt; and execute Compile() on your expressions. (See LinqKit doc for examples)

If you want to see the full source (including a Test class!) go to https://bitbucket.org/moerie/multilanguagestring

Happy coding!
