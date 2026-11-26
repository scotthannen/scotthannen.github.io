---
layout: post
title:  "The Liberation of Pure DI (plus a bonus rant about MEF, which is not an IoC container)"
date:   2018-12-16 05:20:58 -0400
category: blog
description: MEF is not an IoC/dependency injection container.
tags: dependency injection, mef, ioc, pure di
---

It's important to know the distinction between dependency injection and using a dependency injection/IoC container. 
Dependency injection is writing classes so that they *receive* dependencies instead of creating them or referencing static classes. 
A container helps you to create instances of those dependencies.

To illustrate (forgive me, I really need to learn how to be brief and just mention a concept without explaining it):

```csharp
    public class MyClassThatNeedsDependency
    {
        private readonly IThingINeed _thing;

        public MyClassThatNeedsDependency(IThingINeed thing)
        {
            _thing = thing; // dependency injection - good!
        }

        public void DoSomething()
        {
            _thing.MakeMagicHappen();
            var someOtherThing = new SomeOtherThingINeed(); // creates an instance of a concrete 
            someOtherThing.MakeOtherMagicHappen();          // implementation - bad
        }
    }
```

`IThingINeed` is injected. The class expects an instance supplied to its constructor. Because the class *receives* an `IThingINeed` instead of creating an implementation, it is completely decoupled from any implementation of `IThingINeed`. That's good. When we write unit tests for this class we can mock `IThingINeed`.

Then it creates an instance of `SomeOtherThingINeed`. That's potentially bad. Because it does that, this class is tightly coupled to that class. We can't use or test `MyClassThatNeedsDependency` without also using or testing `SomeOtherThingINeed`. It's also harder to change `SomeOtherThingINeed`. What if we want to add constructor arguments? That means every class that creates an instance of `SomeOtherThingINeed` must be changed to supply those arguments. Those classes will need to know where to get the values for those arguments. The tight coupling becomes even tighter and our code slowly becomes harder to maintain.

That's dependency injection. We receive dependencies instead of creating them, which frees each class to do its own thing without having to worry about how its dependencies do their things.

An IoC container is a tool that helps us to create instances of classes that employ dependency injection. For example, suppose one class depends on `ISomething`, but the implementation of `ISomething` is a class that has three more dependencies. And some of those dependencies have dependencies, and so on. The result is that each individual class is simpler in isolation (exactly what we want) but creating an instance of a class is now more complex. A constructor call might look something like this:


```csharp
    IHouse house = new FloorPlan23House(
		new ElectricalSystem(
			new GeModel3000WiringPanel(), 
			new SolarPowerSource(new acmeSolarPanelArray(New Sun()), new MassiveBattery())), 
		new Kitchen(new MarbleCounterTop(new HomeDepotSink()....
```

This could become incomprehensible if you're working with a large number of classes. An IoC container like Windsor, Autofac, Unity, or Ninject makes it more manageable. You can write code that looks like this:


```csharp
    var container = new WindsorContainer();
    container.Register( 
        Component.For<IHouse, Floorplan23House>(), 
        Component.For<IPowerSupply, SolarArray>(), 
        Component.For<ISolarPanel, Model3000SolarPanel>(), 
        Component.For<IBattery, MassiveBattery>(),
        // lots more
    );
    IHouse house = container.Resolve<IHouse>();
```

As long as you've specified which concrete class or instance to use for each type required in a constructor, it calls all the constructors to create new objects, passes them to other constructors, and so on until it has created the "top-level" class you need. If there's a type you haven't "registered" it throws a helpful exception that says, "I'm trying to create a class that needs `ISomething`, but you haven't told me what the implementation is." What's nice is that you can work with each class within its own context without having to think about details of other classes you don't care about in that moment. (You can also organize all of this setup code so that it's easier to manage, and write tests so you don't get runtime errors. It's nice if exceptions are helpful, but they're 100 times more helpful if you catch them in unit tests.)

### Enter MEF, Which Is Not an IoC Container
This brings me to Microsoft's Managed Extensibility Framework, or MEF. I survived many happy years without it before I found myself working in an environment where it was the only dependency injection container allowed. (If you are familiar with MEF and that statement makes no sense, don't worry - I've caught up with you by the end of this story.) I found this out after writing all of my configuration using Windsor. Fortunately none of my code was coupled in any way to Windsor, so I only needed to change my DI configuration, not my "real" code that did real stuff.

I'm going to be honest and up front about this: I didn't try extremely hard to figure out how to use MEF like a "normal" dependency injection container so I could write code like in the example above. I gave myself 30 minutes to find it. I've worked with a number of IoC containers and although I lean toward Windsor, I like to try others when writing blog posts or answering StackOverflow questions. They're like Chinese buffets - each is different, but if you've been to one you'll recognize what you see at another. (That's one of my favorite things about Chinese buffets.) Trust me, 30 minutes is reasonable. Try it. Give yourself 30 minutes to find instructions for using Autofac as an IoC container. You will only need about 30 seconds (because Autofac is an IoC container.)

In 30 minutes I found tons of pages about how to use MEF, but none explaining how to use it as a simple IoC container. I tried the [official documentation](https://docs.microsoft.com/en-us/dotnet/framework/mef/) where I found stuff that looked like this:


```csharp
    [Import(typeof(ICalculator))]
    public ICalculator calculator;

    [Export(typeof(ICalculator))]
    class MySimpleCalculator : ICalculator
    {
    }
```

What... the... actual... Never, ever will I put these attributes on anything. Perhaps there is a parallel universe in which an evil version of myself with a beard and/or eyepatch would put an attribute on a class that implements `ICalculator` specifying that, yes, it really does implement `ICalculator`, so that some weird voodoo elsewhere can assign a class instance to a public *field*... wait, this isn't even constructor injection. It's *property* injection! Here are my internal members, publicly exposed and blowing in the wind so that MEF *or anything else* can set them, and I'll write my code to behave as if this has happened at some undetermined point in the lifetime of the object! I understand that there are scenarios for this, but given that constructor injection is never even *mentioned* in this documentation, why would anyone consider MEF as their first choice for implementing it? I'm starting to think that MEF isn't the problem. 

Further down:


```csharp
    [Export(typeof(IOperation))]
    [ExportMetadata("Symbol", '+')]
    class Add: IOperation
    {
        public int Operate(int left, int right)
        {
            return left + right;
        }
    }
```

Stop, I can't take it anymore. I'm prepared to accept that this is profoundly ignorant on my part, but I don't even want to know what `[ExportMetadata("Symbol", '+')]` means or what I might have to do elsewhere for it to make sense. I certainly wouldn't inflict it on the developer who comes after me. 

By the way, guess what the compiler does if you place `[Export(typeof(IOperation))]` on a class that doesn't implement `IOperation`? If you guessed nothing, that it's just a way to say that a class implements an interface when it doesn't so that you can get a runtime error instead of a compiler error, you are a winner. 

I'm being a little hard on MEF right now, but here are a few points in its defense: 

- The [documentation I referenced](https://docs.microsoft.com/en-us/dotnet/framework/mef/) does not include the words "dependency injection" or "inversion of control." I don't think that's a mistake.
- As mentioned, the code samples do not resemble anything one would do with any popular dependency injection container (all of which resemble each other.) 
- Microsoft provides a "normal" container for implementing constructor injection. It's called [Unity](https://github.com/unitycontainer/unity). And [another one](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-2.2) which is the default for ASP.NET core, but you can use it in any type of project. Both of them are explicitly described as tools for implementing dependency injection. I don't think Microsoft is confused. 

Could those clues perhaps indicate that MEF makes a lousy IoC container because *it isn't one* and that the real problem is someone thinking not only that it is an IoC container, but that it should be preferred above all others? 

I tried to shed some light on this confusion by looking at how MEF is currently used in the code, and found stuff like this:


```csharp
    RegistrationBuilder registrations = new RegistrationBuilder();
    registrations.ForTypesMatching(r => r.Name.EndsWith("Foo"))
        .SetCreationPolicy(CreationPolicy.NonShared)
        .ExportInterfaces()
        .Export();
    registrations.ForTypesMatching(r => r.Name.EndsWith("Blarg"))
        .SetCreationPolicy(CreationPolicy.NonShared)
        .ExportInterfaces()
        .Export();
    catalog.Catalogs.Add(new AssemblyCatalog(typeof(FooBlarg).Assembly, registrations));
```

Let's break down some of the intriguing questions this raises:

- If you're wondering whether we're looking for classes with names ending in `Foo` to implement interfaces which also have names ending in `Foo`, so am I. I'm about 53% certain that's correct. 
- How many class implementations in the assembly we're searching end with `Foo` and `Blarg`? The answer is two and one, respectively. 
- How do you find out which interfaces and classes this configures? You *search the text of your code* for `Foo` and `Blarg`.
- We already know which interfaces and implementations we want to use (because they are in a class in the same project), so why don't we at least use their full names instead making developers search for classes that end with a suffix and upon finding them, hope that they have correctly guessed what this code is doing? Perhaps because we saw it in a tutorial and it looked cool?
- Can you right-click on a class definition to see where it's used? No, because the class is only instantiated after discovering the type by part of its name. 
- Did I mention that these types are not discovered in a dynamically loaded assembly (the sort of scenario for which MEF exists), but rather exist in the exact same project as the code that "discovers" them (which contains fewer than ten classes, including the one that searches for the other ones?) I didn't need to, because you probably observed that the above code contains explicit instructions to search for types with names ending in `Blarg` *only in the assembly containing type `FooBlarg`*, the very type this code is meant to "discover." 
- Is this like writing code to search for your coffee mug by starting from the current location of your coffee mug and then searching for all coffee mugs that occupy the same physical space? You're on fire.

Searching an assembly for interfaces and implementations is a perfectly sane approach if you haven't explicitly referenced the assembly (which assumes that the types aren't in the same project as the DI setup code) and don't know what types you want to use. (Even in that scenario I wouldn't search for them by name, because magic strings.) MEF exists for that scenario. It's called a *plugin architecture*. Using a plugin architecture to compose several known classes in a single project is not bad code. It is evil code.

So despite my initial reaction, MEF is not the problem. It's just not meant to be an IoC container. Perhaps [this blog post by a Microsoft MVP](https://blogs.msmvps.com/peterritchie/2010/02/24/mef-is-not-an-ioc-container-but-mef-uses-ioc/) entitled "MEF is not An IoC container; but MEF uses IoC" is also a hint. Another is that Googling ".net dependency injection containers" returns numerous lists containing popular libraries like Autofac, Windsor, Ninject, SimpleInjector, and Unity, followed by several others I've never heard of before, but not one of which mentions MEF, because *MEF is not an IoC container.* 

If you need to select an IoC container as a standard for everyone to use and are considering MEF, Newtonsoft.JSON, EPPlus, or anything else which is not a dependency injection container, please use [this tool I have configured](https://www.randomlists.com/list-randomizer?items=Spring.NET%0AStructureMap%0ACastle%20Windsor%0AAutofac%0AUnity%0ANinject%0ADryLoc%0ALinFu%0APicoContainer.NET%0AS2Container.Net&qty=1&dup=false) which will help you select something more suitable than MEF.

And if I'm dead wrong and MEF is meant for general use as an IoC container in common scenarios where we don't need to discover types and don't want to plaster our code with redundant attributes, please email me as quickly as possible before something happens to you and that knowledge disappears forever.

### Saved By Pure DI
How would I replace Windsor with MEF (which is not an IoC container?) How would I configure named dependencies and abstract factories?

I realized a few things:
 - I was only working with about a dozen types, and of those, the consuming application only needed to interact with a single interface.
 - My classes were stateless, so I didn't need to worry about using a container to manage object lifetimes.
 - The implementations of my abstract factories looked more complicated than what they needed to be. They weren't bad, but they would take about three minutes to explain to someone unfamiliar with Windsor, as opposed to the rest of the configuration which someone who had never seen Windsor could understand the moment they saw it, at which point they would immediately understand 80% of how to use Windsor or any other container. I like to make things easy for people, but I also start off assuming that they are as smart and capable as I am. If I know something they don't, maybe they know three other things I don't. But I've digressed. My abstract factories were unnecessarily complicated.

I tend to use an IoC container out of habit because even with a smaller number of classes I prefer it to calling nested constructors. I also thought that in this case using Windsor for a simple configuration would provide a useful example. But if the only IoC container I can use isn't an IoC container at all, then perhaps the better choice is no container at all, A.K.A. Pure DI. (It's also called "Poor Man's DI," which implies that it's always undesirable, that IoC containers aren't free open-source software, and that all developers are men.)

Remember, dependency injection isn't using a container. It's how we write our classes. The classes themselves don't know whether they're being created using a container or by calling their constructors the "normal" way. If they do need the container to function then we've done something wrong. Poor Man's DI means that having created classes that rely on dependency injection, we create them the old-fashioned way, without a container.

The result was different from what I'm used to, but all things considered it wasn't end-of-the-world horrible either. To briefly summarize (and obsfucate) the application receives binary data in a byte array along with a string that indicates what sort of data it has received. It uses the string to determine which implementation of `IFoo` should be created to process the data. (That's where the abstract factory came in.) Here's the factory implementation which replaces what I had done with Windsor:


```csharp
    public class FooFactory
    {
        private readonly Dictionary<string, IFoo> _foos
            = new Dictionary<string, IFoo>(StringComparer.OrdinalIgnoreCase);

        public FooFactory()
        {
            // shared
            var fooWidget = new FooWidget(
                new WidgetProvider(
                    new WidgetBuilder(),
                    new[]
                    {
                        TypeConversions.AdjustIncompatibleTypes, //injecting delegates
                        TypeConversions.AdjustInvalidStrings     //instead of interfaces
                    }));
            var fooMapper = new FooMapper(new BlargFormatter(), new PaulaBeanFormatter());
            var dataValidator = new SpecializedFormatDataValidator();

            // Red (the same Foo is used for two data types)
            var blueSpecializedFormatReader = new BlueSpecializedFormatReader(dataValidator);
            var redFoo = new RedFoo(
                blueSpecializedFormatReader,
                fooWidget, 
                fooMapper,
                FooDirectories.GetDirectory);
            _foos.Add(FooDataTypes.RedFooOne, redFoo);
            _foos.Add(FooDataTypes.RedFooTwo, redFoo);

            // Blue
            var blueFooReader = new SpecializedFormatReader(dataValidator);
            var blueFoo = new BlueFoo(
                new FooTypeSelector(),
                FooDirectories.GetDirectory,
                blueFooReader,
                fooWidget,
                fooMapper);
            _foos.Add(FooDataTypes.BlueFoo, blueFoo);

            // Green
            var greenFoo = new GreenFoo(fooWidget, fooMapper);
            _foos.Add(FooDataTypes.GreenFoo,
                greenFoo);
        }

        public IFoo GetFoo(string dataType)
        {
            if (_foos.ContainsKey(dataType))
                return _foos[dataType];
            throw new ArgumentException($"No foo is specified for data type {dataType}");
        }
    }
```

As you can see, I'm creating the various implementations of `IFoo` and putting them in a `Dictionary<string, IFoo>` so I can call `GetFoo` to retrieve the implementation I need at runtime.

To emphasize, this isn't my first choice. These nested constructor calls look like an example to demonstrate why we use an IoC container instead of doing this. Someone could reasonably be puzzled and wonder why I didn't. On the other hand:
 - While Windsor or Autofac aren't hard to understand, reading this doesn't require anyone to learn anything new. (Composing this with an IoC container so that people could see it and learn something new would be arguably better, but you can't win them all.)
 - The longest method isn't that long. If it gets any longer I can easily break it up into smaller methods.
 - I ended up eliminating some slightly overcomplicated abstract factory code and combining my class composition with the factory that produces instances of `IFoo`, which meant less code overall. 
 - It's easy to catch missing components in a DI registrations with tests, but now I get instant compiler errors if I miss a constructor argument.
 - If I need to include some other dependency that isn't created within this factory, I can inject it into the factory, and then into the classes that need it.
 - Both my production code and my integration tests used my Windsor factory to create test subjects. Now they use this factory. I didn't have to change a thing. 
 - I didn't violate all that is natural and decent by writing evil, fragile code that searches my interfaces and classes to "find" types I already know about.

It wouldn't take much for this approach to become problematic. For example, what if I needed scopes or to create more than just implementations of `IFoo`? I wouldn't want to make the above code bigger and more complex or have several such "factory" classes. What if I needed to create unrelated class instances that shared certain dependencies? An IoC container makes that easier by creating a dependency once and reusing it wherever it's needed. But now I've proven that my code is decoupled from whatever method I use to compose class instances, so I can change it back again as needed. Someone could even bend time and space to use MEF if they wanted to.

Still, I had fallen into a comfortable rut of always doing things the same way, so it was oddly liberating to realize that my pure DI version was in some ways simpler and better than my Windsor version. It's nice to take something away and see that you can live without it.