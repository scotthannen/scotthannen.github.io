---
layout: post
title:  'Inject Settings Rather Than Reading from AppSettings'
date:   2016-04-18 07:20:58 -0400
category: blog
---

We often see code that reads from `AppSettings` or app.config when it needs some settings value or connection string.

``` csharp
public class ClassThatNeedsSettings
{
    public void DoSomething()
    {
        var numberOfTimesToDoSomething = ConfigurationManager.AppSettings["numberOfTimesToDoSomething"];	
        var importantStringValue = ConfigurationManager.AppSettings["importantStringValue"];
        var connectionString = ConfigurationManager.AppSettings["connectionStringName"].ConnectionString;
        // Do something with those values
    }
}
```

Why is this not ideal?

- You have to convert, parse, or otherwise manipulate values which could be incorrectly formatted or even missing.
- If a value is missing or malformed, the exception thrown won't be helpful without some additional handling. A `NullReferenceException` or `FormatException` at runtime won't indicate that some `AppSettings` key is null or that it's supposed to contain a date, not an integer. 
- Those `AppSettings` references could be sprinkled throughout the class making it difficult to read the class and easily tell what it depends on. 
- It's difficult or even impossible to unit test. First, all of those values will need to be in the app.config of the unit test project. But what then if we want to test the behavior of the application both if the value of `numberOfTimesToDoSomething` is zero and if it's 10? `AppSettings` can only contain one value.
- What if the class that needs the settings is in a library and doesn't have it's own app.config? How will the consuming application "know" that it must contain certain values? What if it already uses keys with the same names? What if it's an ASP.NET Core application and doesn't even have a web.config?
- One could argue that it violates the Single Responsibility Principle. The class should have one responsibility and only be changed to alter the way it fulfills that responsibility. Where the values come from has nothing to do with the class, so we shouldn't have to change the class because of the environment in which it runs.

These concerns don't mean that app.config, web.config, or `AppSettings` are bad. They serve a purpose. What this demonstrates is that a class should not *depend* on that file. 

In the above example, does `ClassThatNeedsSettings` really need an app.config file? No. It needs an integer, a string, and a connection string. So it's better to *provide those values* than to have the class read them out of a file itself. 

### How To Design the Class Differently Using Constructor Injection

For the sake of discussion we'll say that these are values that should not be passed as arguments to any methods. Why not? Because those arguments would reveal details of the class that aren't relevant to the consumer. For example the consumer shouldn't know that `ClassThatNeedsSettings` communicates with a SQL database, so it wouldn't make sense for a consumer to pass a connection string as a method argument.

A better alternative is to pass the arguments to the class's constructor. The class requires these parameters, so we provide them when we create the class.

We could do it like this:

``` csharp
public class ClassThatNeedsSettings
{
    private readonly int _numberOfTimesToDoSomething;
    private readonly string _importantStringValue;
    private readobly string _connectionString;
    
    public ClassThatNeedsSettings(
        int numberOfTimesToDoSomething,
        string importantStringValue,
        string connectionString)
        {
            _numberOfTimesToDoSomething = numberOfTimesToDoSomething;
            _importantStringValue = importantStringValue;
            _connectionString = connectionString;
        }

    // etc.
}
```

We create `readonly` fields to store the values passed in the constructor because the class simply receives them and operates with them. It doesn't change them.