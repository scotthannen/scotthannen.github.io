---
layout: post
title:  'ASP.NET 5/Core Configuration Settings - Starting From Zero'
date:   2016-03-19 07:20:58 -0400
category: blog
---

This content is dated. What you're looking for is [Options Pattern in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/options).
We've come a long way from the painful awkwardness of [custom configuration sections](https://msdn.microsoft.com/en-us/library/2tw134k3.aspx).
Finding documentation was difficult, and creating complex objects was a chore. I'm still amazed at how much easier the options pattern is.

## The Old Way: web.config

`web.config` is familiar and comfortable, but to use it without coupling to it took a few extra steps. For example, we could create an interface bearing the properties we need, and inject that into a controller:

``` csharp
        public interface IPagingSettings
        {
            int PageSize { get; }
            int PrefetchPages { get; }
        }
```

and then create an implementation that reads from `<appSettings>`. (I’m glossing over some of the additional steps to ensure that the values can be parsed, aren’t missing, etc.)

``` csharp
        public class PageSettingsFromAppSettings : IPagingSettings
        {
            public int PageSize { get { return int.Parse(ConfigurationManager.AppSettings["pagingSettings:pageSize"]); }}
            public int PrefetchPages { get { return int.Parse(ConfigurationManager.AppSettings["pagingSettings:prefetchPages"]); }}
        }
```

It’s much easier to just call `<appSettings>` straight from a controller or other classes, even though they shouldn’t have a direct dependency on `ConfigurationManager`, and so more often than not that’s what happened.

## The New Way: Configuration Decoupled From a File

ASP.NET Core makes it much easier for us to define classes containing the settings we need and then populate those classes using a file or any other way we want to. But so far I’ve encountered code samples that show either too little (no context, what class is this code sample in?) or too much (big overview, thanks, now how do I store and retrieve a setting?) Maybe I’m just impatient. So I’m going to work throught and document how to perform the simple tasks I need from end to end. Those are:

*   Retrieve an application-wide setting
*   Retrieve a class-specific setting - more like the equivalent of a custom `ConfigurationSection`.  
    _From what I’ve read so far I suspect that the way the above two are done will be exactly the same._
*   Retrieve a connection string
*   Understand how to supply these settings without depending on any configuration file

### Retrieve an Application-Wide Setting

Without overthinking it, I want to have the equivalent of `<appSettings>`, and in it a setting called `SiteName` containing the display name of my site. I don’t think I’m ever going to want to do this. If I were injecting this into a class the interface would be `IAppSettings` or an instance of `AppSettings`, and I think that’s just way too general. It doesn’t follow the principle of _interface segregation_ (SOL**_I_**D.) But whatever, I’m doing it!

#### 1\. Create the Project

I started with an empty ASP.NET 5 preview template. It was missing the references for MVC controller. Add a NuGet package - no dice. I’m trying to learn how to configure settings, not how to create an empty project. (One thing at a time.) So I deleted it and started an ASP.NET preview Web Application project which contains all sorts of sample pages I don’t care about. But I don’t care right now. I’m back on track.

#### 2\. Add a Controller To Test the Setting I’m About to Create

I right-clicked my project, selected Add New Item -> MVC Controller Class -> named “TestMyAppSettingsController”

Here’s the controller:

``` csharp
        public class TestMyAppSettingsController : Controller
        {
            public IActionResult Index()
            {
                return Content("I will replace this with my setting.");
            }
        }
```

I tested it ([my localhost]/TestMyAppSettings) and it returns my content. Now I’m ready to create my `appSettings` type interface or class and inject it into this controller.

#### 3\. Create My “appSettings” equivalent

Again, I think this is too vague. But I’m going to learn this by repeating what I know and _then_ changing it. I opened `config.json` and found that Microsoft apparently read my mind from the past when creating their sample application.

``` json
        {
          "AppSettings": {
            "SiteTitle": "TestAspNetCoreSettings"
          },
          "Data": {
            "DefaultConnection": {
              "ConnectionString": "Server=(localdb)\\mssqllocaldb;Database=aspnet5-TestAspNetCoreSettings-69648091-510d-440f-bbd7-7157691e47f0;Trusted_Connection=True;MultipleActiveResultSets=true"
            }
          }
        }
```

Creepy, but it in a good way. But I want a `SiteName` setting, not `SiteTitle`. So I’ll edit the AppSettings section:

``` json
          "AppSettings": {
            "SiteTitle": "TestAspNetCoreSettings",
            "SiteName": "Scott's Awesome Site"
          }
```

So how will my value get into this controller?

There needs to be a class that maps to the AppSettings section in config.json. So I’ve added this class (I created a “Settings” folder and put it in there.)

``` csharp
        public class AppSettings
        {
            public string SiteTitle { get; set; }
            public string SiteName { get; set; }
        }
```

Now I need to populate my AppSettings class with the values from config.json. _This part is really nice_ because it’s understood that we’ll want to use dependency injection, in this case to insert an instance of `AppSettings` into a controller. So the place where I set this up is in the `ConfigureServices()` method in startup.cs, where we configure all of our dependency injection.

I’m adding just one line at the end of this method:

``` csharp
        services.AddTransient(settings => ConfigurationBinder.Bind<AppSettings>(Configuration.GetConfigurationSection("AppSettings")));
```

Translation: Dependency injection container, when I ask for an instance of `AppSettings`, resolve a new (transient) instance from the section in my configuration named “AppSettings.”

Next I’ll modify my controller so that it expects an instance of AppSettings to be injected. It just assumes that when the constructor is called that argument will be supplied.

``` csharp
        public class TestMyAppSettingsController : Controller
        {
            private readonly AppSettings _appSettings;

            public TestMyAppSettingsController(AppSettings appSettings)
            {
                _appSettings = appSettings;
            }

            public IActionResult Index()
            {
                return Content(_appSettings.SiteName);
            }
        }
```

#### 4\. It Works!

And then, like auto-magic, I test my controller again and this time it returns the value - “Scott’s Awesome Site” - from the instance of `AppSettings` which was populated from config.json. (I’m glossing over the seven or eight times it didn’t work and skipping to what actually worked.)

Two important details to note:

*   In the above line of code, `Configuration` is not just some container for config.json. It’s a `Microsoft.Framework.Configuration.ConfigurationSection` which contains configuration data from multiple sources _including_ config.json. (Look at the `Startup()` method and you’ll see how it’s specified which sources make up `Configuration`.) That’s going to be really useful, because it means that we can add into our `Configuration` other .json files, other types of files, or sources that don’t even come from files. In old ASP.NET world, “Configuration” meant your .config file. Now it can be anything.
*   Why transient? Why create a new instance each time? Because the properties of `AppSettings` aren’t read-only. If I’m just passing around one instance then something somewhere could change a property, affecting the entire site. If I make them read-only then they don’t get populated by the above line of code. I would prefer to make this a singleton but for the moment I’m happy just to have it working. I’ll come back to that.

I said this was going to be my equivalent of `AppSettings`, but in reality there’s nothing about it that’s different from any other class, including the one I’m going to create next.

### Retrieve a Class-Specific Setting

By “class-specific” I just mean something less vague and general than `AppSettings` - settings which may be required by a specific class or group of classes.

Again, this is just going to be the same as what we just did. But because it looks so easy, I’m going to put these settings in a separate file. Won’t that be nice - instead of having one ginormous configuration file (ok, there were ways to externalize sections) it’s going to be very easy to separate settings into their own files.

This time I’m going to start with the class, and I want to make a few things different:

*   I want the values to be read-only so I can create a singleton and not worry that anything will change its values.
*   Instead of just strings I want to use a few other types, including at least one collection.

Here’s the class and corresponding interfaces. (Don’t think about what the properties mean. They mean nothing.) It’s a little extra code but it’s so much easier than defining a custom `ConfigurationSection`. Maybe this isn’t worth the effort and this should also be transient. But if we can do it the hard way then we know we can do it the easy way.

``` csharp
        public interface IFooSettings
        {
            string Name { get; }
            IEnumerable Foos { get; }
        }
        public interface IFoo
        {
            string Color { get;  }
            double BarUnits { get;  }
        }

        public class FooSettings : IFooSettings
        {
            public string Name { get; set; }
            public List<Foo> FooList { get; set; }

            public IEnumerable Foos
            {
                get
                {
                    if (FooList == null) FooList = new List<Foo>();
                    return FooList.Cast<IFoo>();
                }
            }
        }

        public class Foo : IFoo
        {
            public string Color { get; set; }
            public double BarUnits { get; set; }
        }
```

Next, a fooSettings.json file:

``` json
        {
          "FooSettings": {
            "Name": "MyFooSettings",
            "FooList": [
              {
                "Color": "Red",
                "BarUnits": "1.5"
              },      {
                "Color": "Blue",
                "BarUnits": "3.14159'"
              },      {
                "Color": "Green",
                "BarUnits": "-0.99999"
              }
            ]
          }
        }
```

Then, in `Startup()` (in Startup.cs) where we specify what goes into our `Configuration`, I’ll add `fooSettings.json`:

``` csharp
        var builder = new ConfigurationBuilder(appEnv.ApplicationBasePath)
            .AddJsonFile("config.json")
            .AddJsonFile($"config.{env.EnvironmentName}.json", optional: true)
            .AddJsonFile("fooSettings.json");
```

Finally, in `ConfigureServices()` (also in Startup.cs) I’ll tell it to load an instance of `FooSettings`, cast it as `IFooSettings` (so the properties appear read-only) and supply that single instance for all dependencies on `IFooSettings`:

``` csharp
        var fooSettings = (IFooSettings)ConfigurationBinder.Bind<FooSettings>(Configuration.GetConfigurationSection("FooSettings"));
        services.AddInstance(typeof (IFooSettings), fooSettings);
```

I haven’t started doing unit tests yet. For the moment I just put a breakpoint on the second line to see if `fooSettings` had been created like I expected. No problem - it worked on the first try. (It even parsed my `double` values. If it can’t it just ignores them.) So now I can inject that into a controller:

``` csharp
        public class TestFooSettingsController : Controller
        {
            private readonly IFooSettings _fooSettings;

            public TestFooSettingsController(IFooSettings fooSettings)
            {
                _fooSettings = fooSettings;
            }

            public IActionResult Index()
            {
                var output = new StringBuilder();
                output.AppendLine($"Name: {_fooSettings.Name}");
                foreach (IFoo foo in _fooSettings.Foos)
                {
                    output.AppendLine($"Foo: {foo.Color},{foo.BarUnits}");
                }
                return Content(output.ToString());
            }
        }
```

Starting the site and browsing to [my localhost]/testfoosettings returns:

<pre>
    Name: MyFooSettings
    Foo: Red,1.5
    Foo: Blue,3.14159
    Foo: Green,-0.99999
</pre>

### Retrieve a Connection String

I’m not even going to code this because it will look just like the previous examples:

*   Add the settings into a section in either config.json or some other .json file
*   Create a class that maps to that section (or I can do these steps in the other order)
*   Register the class in `ConfigureServices()` so I can inject it when needed.

## Conclusion

When I typed the first paragraph I didn’t know how to define and use these configuration settings. That was a big obstacle to building a new ASP.NET Core application. There are still more best practices that I can discover and apply (like validation), but in order to do that I’d have to have it working first, which I now do.

What’s different from using `web.config`?

*   Our classes depend on strongly-typed classes and interfaces for settings, not files. We can still _use_ files, but we reference the classes, not the files.
*   We could do that before, but now it’s easier.
*   It’s also much easier to load more complex settings including nested values. We could do it before with a custom `ConfigurationSection` but good riddance.