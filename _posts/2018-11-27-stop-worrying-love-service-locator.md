---
layout: post
title:  "How I Learned to Stop Worrying and Love the Service Locator"
date:   2018-11-27 05:20:58 -0400
category: blog
description: dependency injection container service locator autofac antipattern anti-pattern
---

Service locators are an anti-pattern. They are what we used before we learned about dependency injection. If you already know what a service locator is and why it's harmful (it's unethical to explain one without the other) then skip past this first heading.

### Why Service Locators are Evil

To briefly illustrate what a service locator is and why it’s bad, here’s a “good” class with an injected abstract dependency:

```csharp
    public class ClassWithADependency
    {
        private readonly IDoesSomethingINeed _somethingINeed;

        public ClassWithADependency(IDoesSomethingINeed somethingINeed)
        {
            _somethingINeed = somethingINeed;
        }
    }
```

It’s easy to tell what the class depends on. It depends on `IDoesSomethingINeed`. The class is easy to test. We can use a mock or test double as the dependency. 

A service locator is a class that we ask to provide an instance of some dependency for us. It could be an instantiated class or a static class. We might call it like this:

```csharp
    public class ClassWithADependency
    {
        public void Blarg()
        {
            var somethingINeed = ServiceLocator.GetService<IDoesSomethingINeed>();
            somethingINeed.DoIt();
        }
    }
```

It seems convenient. Our logic for creating class instances is hidden away inside of `ServiceLocator`. But there are a few big problems:

First, it’s no longer easy to tell what any class depends on. If I’m injecting dependencies in the constructor it’s easy to tell if my class is getting out of control when it has five, six, or a dozen dependencies. The service locator allows us to hide a dependency on absolutely anything in any method. I can do this:

```csharp
    private SomeMethodDeepWithinMyClass()
    {
        var whatIsThis = ServiceLocator.GetService<SomethingTotallyUnrelatedToWhatThisClassShouldDo>(); 
        whatIsThis.DoWhoKnowsWhat();
    }
```

This makes it far more convenient to complicate and couple code by adding random dependencies all over the place. The service locator is like a [broken window](https://en.wikipedia.org/wiki/Broken_windows_theory). Once one developer does it, the rest join in. 

Second, a class littered with calls to a service locator is difficult to test. If the service locator is a static class then within a unit test you have to configure that static class to return mocks or test doubles, which is more complex than just injecting the mock where it’s needed. Or, if the service locator is itself injected as a dependency, then we have to create a mock that returns more mocks, like this:

```csharp
    var somethingINeedMock = new Mock<IDoesSomethingINeed>();
    somethingINeedMock.Setup(x => x.GetSomeValue()).Returns("theValue");
    var serviceLocatorMock = new Mock<IServiceLocator>();
    serviceLocatorMock.Setup(x => x.Resolve<IDoesSomethingINeed>()).Returns(somethingINeedMock.Object);
```

It’s not catastrophic at first but it gets out of control fast. I’ve seen cases where developers just give up because they can’t keep track of what dependencies a class gets from the service locator, so they use setup methods that create service locator mocks that return mocks of everything, whether an individual test needs it or not. At that point it’s nearly impossible to tell what you’re actually testing.

(You may have noticed that a call to a service locator looks a lot like a call to resolve something from a dependency injection container. In fact, if we were to call the container directly from most classes, then that would be using the container as a service locator.)

### When We Need a Service Locator

In order to properly use dependency injection, we need to control how classes are created at the composition root. ASP.NET Core controllers are an example. They are the classes created to handle incoming requests, and at application startup we can configure them to expect injection of certain dependencies. If those dependencies have their own dependencies, we inject those too. It’s depedency injection all the way down. There’s no need to for a controller to directly resolve something from `ServiceProvider` (.NET Core's dependency injection container), using it like a service locator. 

Sometimes properly configuring dependency injection just isn’t an option. What if we’re working in an old WebForms app? WebForms was recently updated to support dependency injection, but it may be too late if you’re working in an old app.

Or, what if you need to work on an existing class and the composition root is hopelessly mangled beyond comprehension? Suppose the class you need to modify looks like this:

```csharp
    public class ClassINeedToModify : ItInheritsFromThis
    {
        public ClassINeedToModify(
            DependsOnThis thing, 
            AndThis otherThing, 
            KitchenSink kitchenSink) 
            : base(kitchenSink)
        {
            thing.DoSomethingEvil(otherThing);
        }
    }
```

And, by the way, `ItInheritsFromThis` is a giant god class that you can’t modify because half the application depends on it. And it’s all created by calling constructors using values from even more static classes with no dependency injection container in sight.

Now you’re out of luck. You want to add new functionality using small, sanely designed, testable classes that employ dependency injection, but the sorry state of the application’s composition root makes it impossible to properly inject your new dependencies into the existing class.

What do we do? Give up and pile onto the mess? Here are a few options.

### Dependency Injection With Defaults

This might get us where we need to be:

```csharp
    public class ClassINeedToAddToTheMess
    {
        private readonly IDependsOnThis _dependency;

        public ClassINeedToAddToTheMess(IDependsOnThis dependency = null)
        {
            _dependency = dependency ?? new DependsOnThis();
        }
    }
```

Now we can instantiate a new instance of `ClassINeedToAddToTheMess` and it will create its own default dependenies, but we can test the class using mocks. 

This suffices in some cases where we don’t have a lot of nested dependencies and don’t need to do anything complicated.

But what if we need to construct more complex dependencies using named components and abstract factories - the sort of thing that a dependency injection container facilitates?

### Create Your Own Composition Root and Expose It With a Service Locator

Yes, i’m recommending that you expose yourself to scorn, ridicule, and self-loathing by deliberately and knowingly implementing a recognized anti-pattern. Maybe you’ll feel better if you add a comment like this:

```csharp
    // I’m so sorry that I had to implement a service locator. There was no other way. Forgive me!
```

You might feel even better if you write this:

```csharp
    /* Yes, this is a service locator. I’m not happy about it. But it wasn’t my idea to mangle 
     * the composition root so thoroughly that it’s impossible to properly inject anything into 
     * this third-level inherited god class, which by the way is also an anti-pattern. 
     */
```

A happier way to think about it is that you’re creating a new composition root from which to resolve your classes.

How do we do it? The same way we would if were going to correctly configure and use a dependency injection container. We can start by creating a class that configures the container. [This post](/blog/2018/04/03/tame-your-dependency-injection-registration.html) is largely devoted to that topic. 

For an example, using [Autofac](https://autofac.org/):

```csharp
    public class ClassINeedToAddAutofacModule : Module
    {
        protected override void Load(ContainerBuilder builder)
        {
            builder.RegisterType<ClassINeedToAddToTheMess>().As<IClassINeedToAddToTheMess>();
            builder.RegisterType<DependsOnThis>().As<IDependsOnThis>();
        }
    }
```

Notice that it’s specific. We’re creating a single dependency for our unfortunate god class, so we’re limiting this to configuring that one dependency and its dependencies. As a bonus, this is reusable if the application ever gets refactored so that we can use a dependency injection container the way we’d like to. 

What if we need to register a dependency on something created by the god class? For example, the god class writes log data using a [`TelemetryClient`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.applicationinsights.telemetryclient?view=azure-dotnet) it creates, but we don’t want our new classes coupled to `TelemetryClient`. We can work with that.

First, we create the abstraction that we do want to depend on:

```csharp
    public interface ILogger
    {
        void LogException(Exception ex);
    }
```

Then we create an implementation that depends on `TelemetryClient`:

```csharp
    public class TelemetryClientLogger : ILogger
    {
        private readonly TelemetryClient _telemetryClient;

        public TelemetryClientLogger(TelemetryClient telemetryClient)
        {
            _telemetryClient = telemetryClient;
        }

        public void LogException(Exception ex)
        {
            _telemetryClient.TrackException(ex);
        }
    }
```

Then we modify our module so that it depends on an `ILogger`. 

```csharp
    public class ClassINeedToAddAutofacModule : Module
    {
        private readonly ILogger _logger;

        public DoesSomethingINeedAutofacModule(ILogger logger)
        {
            _logger = logger;
        }

        protected override void Load(ContainerBuilder builder)
        {
            builder.RegisterType<ClassINeedToAddToTheMess>().As<IClassINeedToAddToTheMess>();
            builder.RegisterType<DependsOnThis>().As<IDependsOnThis>();
            builder.Register(c => _logger).As<ILogger>();
        }
    }
```

Now the classes created within our composition root can depend on the abstraction. 

### Inserting Our Service Locator

This is the slightly distasteful part. Our service locator can be an instantiated class or a static class. It doesn’t really matter because either way it won’t be testable within the context of our god class. Remember, we’re not creating that problem. The idea is to inject dependencies instead of instantiating them, but the fact that we can’t is the reason why we’re going this route. The existing code might not be testable. We’re just doing our best to ensure that the *new* classes we integrate with it are composed and testable.

Here is the service locator class. I considered calling it a "factory" because it sounds better, but let's just call it what it is.

```csharp
    public class DoesSomethingINeedLocator
    {
        private readonly IContainer _container;

        public DoesSomethingINeedLocator(ILogger logger)
        {
            var containerBuilder = new ContainerBuilder();
            containerBuilder.RegisterModule(new ClassINeedToAddAutofacModule(logger));
            _container = containerBuilder.Build();
        }

        public IDoesSomethingINeed Locate()
        {
            return _container.Resolve<IDoesSomethingINeed>();
        }
    }
```

To make this a little less icky, the service locator explicitly returns only one type of dependency. There is no `Get<TAnything>()` function. 

Now the god class, which was already creating and using its own `TelemetryClient`, can call this:

```csharp
    var thingINeedLocator = new DoesSomethingINeedLocator(_telemetryClientItAlreadyCreated);
    var thingINeed = thingINeedLocator.Locate();
```

Because we're creating a class instance - calling `new` to instantiate a new object - we can't mock this dependency, so the god class is less testable. The trade-off is that the classes it returns - all of the new stuff - use dependency injection, are testable, and don't have any reference to the container. 

We want our composition root with all its references to the container to be at application startup, far away from the business end of application. Since we can’t, we’ve planted a new composition root where we need it and done our best to keep it tidy.

### It's Testable

When we compose classes using a dependency injection container, we get a runtime error if we missed registering a dependency. We don’t need to run the application and wait for an exception to see if we’ve missed anything. We can write a unit test:

```csharp
    [TestMethod]
    public void AutofacModuleResolvesThingINeed()
    {
        var logger = new Mock<ILogger>();
        var containerBuilder = new ContainerBuilder();
        containerBuilder.RegisterModule(new ClassINeedToAddAutofacModule(logger.Object));
        using (var container = containerBuilder.Build())
        {
            container.Resolve<IClassINeedToAddToTheMess>();
        }
    }
```

### Is It Worth It?

If the new classes you need to create are complex and contain a lot of nested dependencies, then yes, it is. 

This approach is satisfying because it draws a line between the legacy code I'm working in and the new code I'm writing. The legacy classes don't allow me to inject dependencies, but my new ones do. The service locator is the bridge between them. That lets me work the way I'm used to, creating small, testable classes that I can compose using a dependency injection container. 


