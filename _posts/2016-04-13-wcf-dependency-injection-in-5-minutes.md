---
layout: post
title:  "Dependency Injection in WCF Services in Five Minutes"
date:   2016-04-13 07:20:58 -0400
category: blog
---

Dependency injection is neither too complex nor effortlessly intuitive. The benefits and problems it solves may not become apparent until you've used it or see it in action. I often recommend it when answering questions on [stackoverflow](http://stackoverflow.com/), not because it's the direct answer to the question, but because it enables a solution or changes the way we design classes and applications so that some problems aren't problems any more.

But if you've only heard of it and are trying to learn more, many explanations are bit frustrating, using (IMHO) a bit too much jargon and too many unhelpful diagrams. (That "jargon" is often useful vocabulary but most of us don't have the luxury of learning in a linear manner that proceeds from terminology to advanced concepts.)

Therefore in this post I'm only going to show *how* to use it with minimal explanation of *why*. Here's a good *[why](http://www.jamesshore.com/Blog/Dependency-Injection-Demystified.html)* article. This example will use [Castle Windsor](https://github.com/castleproject/Windsor/blob/master/docs/README.md) to enable dependency injection, but that's just one option.

It's going to look a little like extra code that doesn't do anything. I'm going to ask you to trust that it's worth it many times over. (If you decide that it isn't then you lose little or you've already stopped reading.) I'm going to keep this code bare-bones and minimal.

### 1. Create a WCF Service Application and Add Castle Windsor

Create a new WCF service application or open an existing one.  
Add the [Castle Windsor WCF integration facility nuget package](https://www.nuget.org/packages/Castle.WcfIntegrationFacility/) to the project. 

### 2. Add Some Boilerplate Code

Do this a few times and you'll never forget it. (Plus I use [ReSharper](https://www.jetbrains.com/resharper/) which helps with `using` namespaces.)

Add a class named `WindsorInstaller` to the project:

``` csharp
    using Castle.MicroKernel.Registration;
    using Castle.MicroKernel.SubSystems.Configuration;
    using Castle.Windsor;

    namespace YourNamespace
    {
        public class WindsorInstaller : IWindsorInstaller
        {
            public void Install(IWindsorContainer container, IConfigurationStore store)
            {
                //Nothing here yet! We'll come back to it.
            }
        }
    }
```

If there's not already a `Global.asax`, add one. (Add -> New Item -> Web -> Global Application Class.)
In `Global.asax.cs` add the following:

``` csharp
    public class Global : System.Web.HttpApplication
    {
        static IWindsorContainer container;
        
        protected void Application_Start(Object sender, EventArgs e) 
        {
            container = new WindsorContainer();
            container.AddFacility<WcfFacility>();
            container.Install(new WindsorInstaller()); //The class you just created.
        }
    }
```
__What We've Done So Far__

- We've created a single, global instance of `WindsorContainer`. But aren't global variables bad? It's okay - except for the `WindsorInstaller` class, nothing is every going to use that `WindsorContainer`. The point is that there's only one and it lives as long as the application does.
- We've added the `WcfFacility`. That just means we've told the `WindsorContainer` that we'll need it to help with WCF services. 
- We've told the `WindsorContainer` to use the new class we created, `WindsorInstaller`, to get any additional configuration instructions that it's going to need. (We haven't put anything there. We'll come back to it.)

### 3. Create a WCF Service and Add Some Code That Does Stuff

Add a new WCF service called `GreetingService` to the project. Its job is to return a greeting. (It's useless, but a more detailed example takes focus from the main point.)
Put this in `IGreetingService`:

``` csharp
    [ServiceContract]
    public interface IGreetingService
    {
        [OperationContract]
        string GetGreeting();
    }
```
Put this in `GreetingService.svc.cs` (for convenience you can paste it all into one file):

``` csharp
    public class GreetingService : IGreetingService
    {
        private readonly IGreetingProvider _greetingProvider;

        public GreetingService(IGreetingProvider greetingProvider)
        {
            _greetingProvider = greetingProvider;
        }

        public string GetGreeting()
        {
            return _greetingProvider.GetGreeting();
        }
    }

    public interface IGreetingProvider
    {
        string GetGreeting();
    }

    public interface IComputerNameProvider
    {
        string GetComputerName();
    }

    public class ComputerNameGreetingProvider : IGreetingProvider
    {
        private readonly IComputerNameProvider _computerNameProvider;

        public ComputerNameGreetingProvider(IComputerNameProvider computerNameProvider)
        {
            _computerNameProvider = computerNameProvider;
        }

        public string GetGreeting()
        {
            return string.Concat("Hello from ", _computerNameProvider.GetComputerName());
        }
    }

    public class EnvironmentComputerNameProvider : IComputerNameProvider
    {
        public string GetComputerName()
        {
            return System.Environment.MachineName;
        }
    }
```

__What Was All That?__

The purpose of this is to create classes with dependencies. 

- `GreetingService` depends on `IGreetingProvider`.
- `IGreetingProvider` is implemented by `ComputerNameGreetingProvider` which depends on `IComputerNameProvider`.

### 4. Configure Windsor to Make All This Work

I'd call this the fun part but that would imply that the rest wasn't fun.

How are these classes going to get created? How are the right arguments going to get passed into the constructors? WCF service classes don't even *have* constructors, do they? How is that constructor ever going to get called?

This is where dependency injection comes in. We're going to *inject* the *dependencies* into each class. The dependencies are specified in the constructors, so this is also termed *constructor injection*.

First, we need to specify that when an instance of `GreetingService` is created to handle a request, Windsor will be responsible for creating it. To do that, edit the markup of the service and add this:

    Factory="Castle.Facilities.WcfIntegration.DefaultServiceHostFactory, Castle.Facilities.WcfIntegration"

So the markup will look like this:

``` html
    <%@ ServiceHost Language="C#"Debug="true"
        Service="WindsorWalkthrough.GreetingService"
        CodeBehind="GreetingService.svc.cs"
        Factory="Castle.Facilities.WcfIntegration.DefaultServiceHostFactory, Castle.Facilities.WcfIntegration"
    %>
```

Then, let's go back to the `WindsorInstaller` class and fill in the details that the `WindsorContainer` will need to do its work. We also call this *registering dependencies.*

```csharp
    public class WindsorInstaller : IWindsorInstaller
    {
        public void Install(IWindsorContainer container, IConfigurationStore store)
        {
            container.Register(
                Component.For<IGreetingService, GreetingService>(),
                Component.For<IGreetingProvider, ComputerNameGreetingProvider>(),
                Component.For<IComputerNameProvider, EnvironmentComputerNameProvider>());
        }
    }
```
What we're telling Windsor what arguments to pass to the constructor of each class when it creates them.

- When a constructor requires an `IGreetingService`, create a `GreetingService`.
- When a constructor requires an `IGreetingProvider`, create a `ComputerNameGreetingProvider`.
- For `IComputerNameProvider`, create a `EnvironmentComputerNameProvider>`.

Think of it as each class just saying what it needs (what it depends on) and Windsor's job is to create those classes with whatever dependencies they need.

A technical term for this configuration that occurs when the application starts is *[composition root](http://blog.ploeh.dk/2011/07/28/CompositionRoot/)*. We're specifying all at once how all of these classes and their dependencies are *composed* instead of having it scattered through individual classes each creating other classes.

Now you can test the service, invoke the `GetGreeting()` method, and observe that it returns  
`Hello from [your computer name]`.

### What Did All of This Accomplish?

- We were able to write each class so that it depends on abstractions (interfaces.) That's [Dependency Inversion](http://martinfowler.com/articles/dipInTheWild.html), the D in [SOLID](https://scotch.io/bar-talk/s-o-l-i-d-the-first-five-principles-of-object-oriented-design). Each class isn't responsible for creating other classes and knowing what they depend on. They know as little as possible, just what is defined in the interface.
- Because the classes depend on interfaces, it's possible to replace one implementation of an interface with another without modifying the class that depends on it. 
- Because the classes depend on interfaces, they're easier to unit test. We can create each class with "mocked" (dummy) versions of the interfaces it depends on. 
- When a class’s dependencies are all specified in its constructor, it’s much easier to look at a class and see what it depends on. It’s also easier to tell if a class has too many dependencies which means it’s doing too many things and may need to be refactored.

We can write classes that way without Castle Winsdor or dependency injection, but how would we create them? How would we create a class that depends on a few other classes that each depend on more classes, and so on? It would be too hard, so we might just end up creating great big classes that do too many things. Now, as a side effect of using dependency injection, it's easier to write smaller classes that each have a [Single Responsibility](http://www.oodesign.com/single-responsibility-principle.html) (the S in [SOLID](https://scotch.io/bar-talk/s-o-l-i-d-the-first-five-principles-of-object-oriented-design).)

### Final Thoughts  
- Ignore terminology that confuses. Dependency injection isn't dependency inversion or inversion of control, but DI containers are also called *inversion of control (IoC) containers* because DI enables IoC. (I typed that and my own brain went numb.) Why is it even called a *container*?
- Dependency injection does not require a container like Windsor, Autofac, Simple Injector, Unity, etc. But in common use dependency injection is synonymous with use of one of these containers because that's how dependency injection is most often implemented.
- Some developers feel that DI is unnecessary for smaller projects. They may be right, but I recommend using it anyway, especially when learning it. It's good practice and it encourages good habits.
- DI doesn't just inject classes that implement interfaces. Maybe your class depends on a SQL connection string. You can inject values, including from `app.config`/`web.config`.
- By default Windsor and other containers create objects as singletons unless you tell them not to. Sometimes that's not desirable. They all have ways to [control that behavior](https://github.com/castleproject/Windsor/blob/master/docs/lifestyles.md).

I'll add more later on how to register some other types of dependencies and more details. 