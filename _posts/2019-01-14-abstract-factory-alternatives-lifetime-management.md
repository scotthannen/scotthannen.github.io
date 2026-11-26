---
layout: post
title:  "Alternatives To Abstract Factories Part I - Lifetime Management"
date:   2019-01-14 -0400
category: blog
description: Why we shouldn't use abstract factories to produce short-lived dependencies and what we can do instead.
---

Wow, that's got to be the most exciting title for a blog post, ever. It's all about the minutiae.

What do you see when you look at this class that depends on two interfaces which are injected into its constructor?

```csharp
public class DoesSomething
{
    private readonly IThingItNeeds _thingItNeeds;
    private readonly IOtherThingItNeedsFactory _otherThingFactory;

    public DoesSomething(
        IThingItNeeds thingItNeeds, 
        IOtherThingItNeedsFactory otherThingFactory)
    {
        _thingItNeeds = thingItNeeds;
        _otherThingFactory = otherThingFactory;
    }

    public void DoIt()
    {
        _thingItNeeds.CallSomeMethod();                                  // Uses one dependency...
        IOtherThing otherThing = _otherThingFactory.CreateOtherThing();  // Gets the other from the
        otherThing.CallOtherMethod();                                    // factory and uses it.
    }
}
```

Why is one dependency is "directly" injected while the other for the other we inject an abstract factory and request the dependency from the factory?

If you've seen classes like the above and created them, as I have, you might guess that the two dependencies have different lifetimes. For the first one a single instance is used and injected into other class instances over and over. But for the second one we need to ensure that a new instance is created each time `DoIt()` is called, so we use the factory to create a new instance.

### The Problems With Abstract Factories For Lifetime Management

What's wrong with this? The problem is that it causes the class to know way too much about its dependencies. And what it knows could be wrong anyway.

First, because from the viewpoint of `DoesSomething`, `IOtherThing` is an abstraction that it depends on. 
The only thing `DoesSomething` should know is `IOtherThing` has a method it needs to call. 
Now it knows more than that: It knows that `IOtherThing` must be transient - that is, it must have a shorter lifetime. 
The point of dependency injection is so that a class only sees the abstraction it depends on without knowing anything about the implementation. 
Why should it know or care whether the implementation is supposed to be transient?

Second, this doesn't even make sense because the abstract factory is also an abstraction. 
The class shouldn't know how that factory provides an instance of the dependency. 
How does the class know that the instance of the dependency returned by the factory is a new, transient instance?
It doesn't. Maybe the abstract factory is returning the same instance of the dependency over and over. There's a big assumption here about the implementation 
details of that abstract factory, and it could be wrong.

Third, what if the class didn't orginally depend on the factory but then I introduce it to manage the lifetime of the dependency? 
That means that I'm modifying a class because of details of the implementation of an abstraction it depends on. 
The abstraction was supposed to keep my class decoupled from the implementation, and now I've defeated that purpose. 

So in this case, unless we make assumptions about how the factory works, we've haven't change much by introducing the factory except adding some complication.
We were receiving the dependency by injection, and it may or may not be transient. Now we're getting it from the factory, and what we get from the factory may or may not be transient.

### Use Transient Dependencies

The place to manage the lifetime of dependencies isn't in the class that depends on them but in the composition root - likely an IoC container - which is where we specify how dependencies are created and how long they exist.

For instance, if I'm using an `ServiceCollection` - the default IoC container for ASP.NET Core applications, also available in the Microsoft.Extensions.DependencyInjection NuGet package, and I register the above dependencies like this:

```csharp
IServiceCollection serviceCollection = new ServiceCollection();
serviceCollection.AddSingleton<NeedsDependencies>();
serviceCollection.AddSingleton<IThingItNeeds, ThingItNeeds>();
serviceCollection.AddSingleton<IOtherThing, ShortLivedOtherThing>();
```
What happens? `AddSingleton` means that the first time `NeedsDependencies` is resolved, the other two dependencies are also resolved and injected into it, and then the container re-uses them forever, returning the same instances. If I need a new instance of `ShortLivedOtherThing`, too bad. I'm not going to get one.

If I change it to this:

```csharp
IServiceCollection serviceCollection = new ServiceCollection();
serviceCollection.AddTransient<NeedsDependencies>();
serviceCollection.AddSingleton<IThingItNeeds, ThingItNeeds>();
serviceCollection.AddTransient<IOtherThing, ShortLivedOtherThing>();
```

Now every time I resolve `NeedsDependencies`, the container will provide a new instance. 
It will reuse `ThingItNeeds` because we know it's okay to reuse that. But it will provide a new instance of `ShortLivedOtherThing` each time.
That's what *transient* means.

(What if I register a class as a singleton and its dependency as transient? That doesn't do any good, because if the class instance lives forever then its dependencies live forever, too.)

Should we worry excessively about making dependencies transient? Will our application suffer because we're creating class instance and over? *Probably* not.

Creating a new instance each time we need one isn't as efficient as reusing an instance, but more often than not the difference is minute. 
Consider a web application. During the processing of each and every request, how many class instances are created and then go out of scope in the framework code that we aren't even aware of? 
How much middleware code is invoked? How many objects have their properties read to render views as HTML? 
There's a lot going on. I'm not saying that it never matters, but odds are that creating a few more classes when they're needed will have no discernible impact whatsoever. 
What does have an impact? The bugs introduced by incorrectly using a singleton.

### Use Scoped Dependencies

In an ASP.NET Core application (and in some other scenarios) we might do this:

```csharp
IServiceCollection serviceCollection = new ServiceCollection();
serviceCollection.AddScoped<NeedsDependencies>();
serviceCollection.AddSingleton<IThingItNeeds, ThingItNeeds>();
serviceCollection.AddScoped<IOtherThing, ShortLivedOtherThing>();
```

In ASP.NET Core, `NeedsDependencies` is neither a singleton nor transient. 
If an instance is resolved during the processing of a web request, it will then be reused every time another instance is resolved during that same web request. 
When it is created, its dependency - `ShortLivedOtherThing` is created too. 
(It could be transient. Either way it will get created once every time `NeedsDependencies` is created, once per request.)
That's useful if a class retrieves data specific to the user making the request. We don't want to retrieve the data over and over, but we want to be sure the same data isn't used when handling another user's request.

(A few years ago before ASP.NET Core we had to add all sorts of weird boilerplate code to ASP.NET MVC to use a container with per-request scoped dependencies. This is way better.)

### Fix the Dependency So It Doesn't Need to Be Transient

Why does a particular class need to be transient? Perhaps because it's not thread safe or maintains some user or request-specific state. If so, can we fix it so that we *can* reuse it?

Much of the time we can create classes so that invoking their methods doesn't modify their state. 
A stateless class doesn't have private fields and properties that are modified when executing methods, so thread safety and reuse aren't an issue. 
As a habit I try to write all classes that way unless they exist specifically to contain data that changes. 

Recently I ran into a scenario where use of a third-party library kept a class from being thread safe. 
In this case I wasn't using an IoC container at all (and couldn't), so reworking it to make some dependencies transitive would have been a hassle.

In this case the issue was easily fixed by making one member static and using a [`ThreadStatic` attribute](https://docs.microsoft.com/en-us/dotnet/api/system.threadstaticattribute). 
That's something I rarely use, but it made more sense to modify the code where the problem was instead of changing everything else to work around it.

---
The preferred use of an abstract factory is when we need the implementation of a dependency to vary according to some runtime value. 
For example we might have a class representing a sales order, and we might need to select an implementation of sales tax logic according to the state or region where it will ship.

Even in a scenario where an abstract factory makes sense, what if we didn't initially use one and now we need one? 
Must we modify existing classes to depend on an abstract factory? Maybe, but there are alternatives. 
I was going to discuss them in this post, but I'll leave it for the next one. What comes to your mind? Maybe we'll converge on some of the same answers.