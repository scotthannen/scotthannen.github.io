---
layout: post
title:  "Tame Your Giant, Beastly Dependency Injection Registration Class and Ride It Into Glorious Victory"
date:   2018-04-03 05:20:58 -0400
category: blog
description: Dependency injection configuration can be manageable and testable.
---

We use DI containers (dependency injection containers, also called *inversion of control containers* or *IoC containers*) to help us write cleaner, more manageable code. 
Isn't it ironic, then, that one of the most cluttered and chaotic areas of our code
is often the part where we register dependencies with the container?

(There are multiple terms to describe registering dependencies with a container, including *container configuration*, *component registration*, and probably more. For the sake of consistency I'll call it *registering dependencies* or *dependency registration*.)

For example, using Unity, the registration might look a little like this:

```
public void RegisterDependencies(IUnityContainer container)
{
    // +100 lines
    container.RegisterType<ISomeInterface, Something>("This");
    container.RegisterType<ISomeInterface, SomethingElse>("That", 
        new InjectionProperty("connectionString",
        ConfigurationManager.ConnectionStrings["thatConnectionString"].ConnectionString));
    container.RegisterType<IGoesNorth, Train>();
    container.RegisterType<Boat>();
    container.RegisterInstance<IGoesSouth>(container.Resolve<Boat>());
    // + 200 more lines
}
```

...except you can can scroll through five screens of this stuff, it's in no discernible order, and it registers dependencies for three or four different libraries used in your application, all mixed together.

It's not the end of the world because at least by cramming all of that in here we're keeping the rest of our code a little 
bit neater. But this still leads to a few problems:

- It's difficult to read and maintain.
- If we maintain other applications that depend on the same libraries, we'll duplicate the code. Those duplications 
  will look almost identical except for some variations in syntax (because there's always more than one way to do the same thing), 
  some different names, and with the registrations in a different order.
- There are no tests for this code, even though an error can cause fatal runtime exceptions. 
- It forces an application to know too much about the inner workings of classes it depends on. An application should deal with the interfaces of a class library without unnecessarily knowing its implementation details. That intent is defeated if our application must contain code to register all of the dependencies of a class library. 
- Finding and deleting unused registrations is harder, so we end up leaving in things we no longer need.

How do we fix this?

## Break Up the Big Method Into Small Methods
If we have a giant method with a hundred registrations and we're looking for an easy fix we can break that method into smaller methods. For a start we might separate them according to areas of functionality. Then our main method might look more like this:

```
RegisterUserComponents(container);
RegisterProductComponents(container);
RegisterLoggingComponents(container);
```

If that's as far as we get it's still more manageable than one big method.

## Separate Dependency Registration Into Modules

Most popular DI libraries provide interfaces or base classes we can implement that provide a familiar, consistent way to add registrations to a container.

```
container.AddFacility<YourFacility>();                // Castle Windsor Facility
containerBuilder.RegisterModule(new YourModule());    // Autofac Module
container.AddNewExtension<YourContainerExtension>();  // Unity Container Extension
```

Even within these three frameworks there are more options. Windsor also has *installers*, which are similar to facilities, but
there are also methods to discover and apply the installers within an assembly. The point is that you have options. (I use Windsor more than any other container, but I like to use other containers when writing or testing ideas to ensure that my designs aren't coupled to the use of a particular container.)

Suppose you have a Foo Calculator which is composed of several classes, and you want to cleanly separate registration of its dependencies into its own
class. You could create a class like this:

```
public class FooCalculatorContainerExtensions : UnityContainerExtension
{
    protected override void Initialize()
    {
        Container.RegisterType<IFooCalculator, FooCalculator>();
        Container.RegisterType<IFooRepository, SqlFooRepository>();
        Container.RegisterType<IFooValidator, FooInputValidator>("FooInputValidator");
        Container.RegisterType<IFooValidator, FooDataValidator>("FooDataValidator");
    }
}
```

and then, in the composition root of your application, where you register dependencies with your container, you would do this:

```
container.AddNewExtension<FooCalculatorContainerExtension>();
```

Isn't that easier to read than if those same component registrations were buried in several hundred lines of similar-looking code? 
Perhaps we could even prevent that giant registration class from growing so large in the first place.

## What Are the Benefits?
Does the title of this post oversell the rewards? Maybe it's not actually glorious... no, I'm going to stand by every word of it. We're not just going to eliminate the ugliness of one giant method by moving code around, although that's pretty awesome too. Once we've tamed this beast there are other ways it can serve us. 

### Reusability
This is particularly true if we're building a class library that we or others will use in different projects. 
The consumers of our library won't need to know how to regsiter its dependencies or add that code to their own container setup.
Quite possibly they can do it with a single line of code.

If we're concerned with making our library container-agnostic - that is, it isn't tied to one particular brand of DI container - 
we could put the Unity-specific or Windsor-specific code in a separate project. I do sometimes, but it depends on the circumstances
and I'm not rigid about it.

### Testability
Assuming that `FooCalculator` is the root object and it depends on the other classes, you could write a unit test like this:

```
[TestMethod]
public void VerifyFooCalculatorRegistration()
{
    using (var container = new Unity.UnityContainer())
    {
        container.AddNewExtension<FooCalculatorContainerExtension>();
        var resolved = container.Resolve<IFooCalculator>();
        Assert.IsNotNull(resolved);
    }
}
```

You could even call `container.Registrations` to get all of the registered types and verify that you can resolve all of them.

### Easier, More Complete Integration Tests
If you're writing integration tests that test a composition of several classes, you can even use a container to simplify your test setup.  
For example:

```
[TestClass]
public class FooCalculatorIntegrationTest
{
    private IUnityContainer _container;

    [TestInitialize]
    public void Setup()
    {
        _container = new Unity.UnityContainer();
        _container.AddNewExtension<FooCalculatorContainerExtension>();
    }

    [TestMethod]
    public void TestSomethingWithMockedRepository()
    {
        var repository = new Mock<IFooRepository>();
        repository.Setup(x => x.GetFoo(It.IsAny<Guid>())).Returns(new Foo(someParticularFooDetails));

        // Replace the previous registration of the repository with the mock.
        _container.RegisterInstance<IFooRepository>(repository.Object);

        var subject = _container.Resolve<IFooRepository>();
        // Do something with the subject. Act and assert.
    }

    [TestCleanup]
    public void Cleanup()
    {
        _container.Dispose();
    }        
}
```

This is especially helpful if we're testing classes with lots of nested dependencies. If we've written reusable code to set up a container
to create class instances, why not use it? And, since it's an integration test, doesn't it make sense that it would also test the code we'll use to register the dependencies?

## Injecting Dependencies Into Our Dependency Registration Classes

That might sound convoluted. But suppose an application depends on our Foo Calculator library, and registers its dependencies by conveniently
adding our `FooCalculatorContainerExtension` to its container. There's just one problem: `SqlFooRepository` needs a SQL connection string.
The library has no way of knowing what that connection string is. How do we provide that dependency to `SqlFooRepository`? (For the 
purpose of demonstration I'm just concerned with a single SQL connection string. In practice this could be a class containing several
settings or an instance of some other class.)

There are a few approaches. Regardless of which we use, what's critical is that the dependency is explicit. It's best if we can require that it 
be fulfilled in order for the code to compile. Second best would be a clear, helpful runtime exception.

One is constructor injection into our container extension/module/facility:

```
public class FooCalculatorContainerExtension : UnityContainerExtension
{
    private readonly string _fooRepositoryConnectionString;

    public FooCalculatorContainerExtension(string fooRepositoryConnectionString)
    {
        if(string.IsNullOrEmpty(fooRepositoryConnectionString))
            throw new ArgumentException($"{nameof(fooRepositoryConnectionString)} is required.");
        _fooRepositoryConnectionString = fooRepositoryConnectionString;
    }

    protected override void Initialize()
    {
        Container.RegisterType<IFooCalculator, FooCalculator>();
        Container.RegisterType<IFooRepository, SqlFooRepository>
            (new InjectionProperty("connectionString", _fooRepositoryConnectionString));
        Container.RegisterType<IFooValidator, FooInputValidator>("FooInputValidator");
        Container.RegisterType<IFooValidator, FooDataValidator>("FooDataValidator");
    }
}
```

Now the extension can't be created and added without supplying the connection string. Its usage would look like this:

```
var fooCalculatorExtension = new FooCalculatorContainerExtension(configuration["connectionStrings:fooCalculator"]);
container.AddExtension(fooCalculatorExtension);
```

Now the application doesn't know too much about the library, except that it needs a connection string. The library, on the other hand, 
doesn't know where or how an application consuming it stores its connection strings. (If you've ever used a library that assumes there's an
app.config/web.config, assumes a particular setting or connection string is there, and throws a useless exception when it's missing, you'll
understand how frustrating that is.)

If a more complex set of settings was involved, I'd likely create a class containing all of them. Using .NET Core you could 
[bind a settings object to your configuration](https://weblog.west-wind.com/posts/2017/Dec/12/Easy-Configuration-Binding-in-ASPNET-Core-revisited).
You could provide a helper method to load those values from app.config/web.config. In any of those cases there would be 
validation which 

- ensures that required values are supplied
- throws clear, helpful exceptions so that no one has to open up the code and figure out what it's looking for and where

## Don't Call Constructors Unless You Have No Choice, And You Always Have a Choice

One last thought: One of the benefits of a DI container is that it inspects a class's constructor, determines what its dependencies are,
and resolves the dependencies it needs to call the constructor. For the most part our registration should not use factory methods which 
resolve objects from the container and then pass them to the constructor of another class.  

What am I even talking about? If that made no sense or you've never heard of such a thing, stop reading now because you don't need to see this:

```
protected override void Initialize()
{
    Container.RegisterType<ISomething, Something>();
    Container.RegisterType<ISomeOtherThing, SomethingElse>();
    Container.RegisterType<IFooRepository, SqlFooRepository>
        (new InjectionProperty("connectionString", _fooRepositoryConnectionString));
    Container.RegisterInstance(typeof(IFooValidator), "FooDataValidator", 
        new FooDataValidator(Container.Resolve<ISomething>()));
    Container.RegisterInstance(typeof(IFooValidator), "FooInputValidator", 
        new FooInputValidator(Container.Resolve<ISomeOtherThing>()));
    Container.RegisterInstance(typeof(IFooCalculator), 
        new FooCalculator(Container.Resolve<IFooRepository>(),
            new []
            {
                Container.Resolve<IFooValidator>("FooDataValidator"),
                Container.Resolve<IFooValidator>("FooInputValidator")}));
}
```

Yes, that happens. I don't know why. If all of the dependencies are registered, there's no sane reason to call constructors,
register the instances with the container, resolve them, and pass them to more constructors. *That's what the container does for us.*
Part of what makes DI containers so awesome is that we don't have to deal with that complexity. 
When we're writing code, there are no levels of nested dependencies. Our classes depend on abstractions, so as we're working with a class
we don't have to think about what the implementations of those abstractions depend on. We eliminate that benefit if we take back upon ourselves the responsibility
of composing our classes "manually" by calling their constructors.

The above mess is functionally identical to

```
protected override void Initialize()
{
    Container.RegisterSingleton<ISomething, Something>();
    Container.RegisterSingleton<ISomeOtherThing, SomethingElse>();
    Container.RegisterSingleton<IFooRepository, SqlFooRepository>
        (new InjectionProperty("connectionString", _fooRepositoryConnectionString));
    Container.RegisterSingleton<IFooValidator, FooDataValidator>("FooDataValidator");
    Container.RegisterSingleton<IFooValidator, FooInputValidator>("FooInputValidator");
    Container.RegisterSingleton<IFooCalculator, FooCalculator>();
}
```

Hopefully this helps us to transform our dependency registration classes from giant, menacing beasts into a manageable herd of smaller, tamer beasts that we can harness to serve us or saddle and ride into code-refactoring victory. Or both!