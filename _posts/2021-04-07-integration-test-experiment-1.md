---
layout: post
title:  "An Experiment With Making Integration Tests Easier to Write - Part One"
date:   2021-04-07 -0400
category: blog
description: Integration tests are easier to write if we can re-use our Ioc setup.
---

In most of the code I work in there are lots of focused unit tests and occasionally some end-to-end tests, but few or no integration tests which verify the successful collaboration of classes working together.

When I say "integration tests" I mean <a href="https://martinfowler.com/bliki/IntegrationTest.html" target="_blank">what Martin Fowler calls "narrow integration tests:"</a>

>-   exercise only that portion of the code in my service that talks to a separate service
>-   uses test doubles of those services, either in process or remote
>-   thus consist of many narrowly scoped tests, often no larger in scope than a unit test (and usually run with the same test framework that's used for unit tests)

Here I'm documenting an approach I've experimented with successfully. Is it new? I have no idea. Is there room for improvement? Certainly, but I have to start somewhere. Perhaps others will find it helpful and have some suggestions.

## Two Birds With One Stone
In a typical .NET application (and no doubt in other types) there's a section of code in which we configure dependencies with an IoC container. In .NET Core that's the `IServiceCollection` which we configure in or around `Startup.cs`.

This code tends to become a little unwieldly, as I noted in <a href="http://scotthannen.org/blog/2018/04/03/tame-your-dependency-injection-registration.html" target="_blank">another blog post</a>. Every time we have another implementation to register we throw it into some poorly organized method which eventually spans a few screens.

This creates both a challenge for integration testing and an opportunity to improve our IoC setup. The challenge is that when we compose the classes for our integration tests, we can't be sure that the test setup matches our production runtime setup. And if they don't match, how valid are the tests? In other words, suppose we have this in our `Startup.cs`:

```csharp
serviceCollection.AddSingleton<FooMessageListener>();
serviceCollection.AddSingleton<IFooValidator, FooApiFooValidator>();
serviceCollection.AddSingleton<IFooCommandHandler, FooCommandHandler>();
serviceCollection.AddLogging();
serviceCollection.AddHttpClient<FooApiClient>();
serviceCollection.AddSingleton<IFooRepository, SqlFooRepository>();
```
...and we want to test `FooMessageListener` which depends on all of the other registered types. How are we going to create an instance of `FooMessageListener`? We could create instances of each class and perhaps use mocks for `FooApiClient` and `SqlFooRepository`, but that creates two problems:

- The tests are brittle. Every time we change the runtime composition we'll also have to change the tests.
- We'll have doubt. Does the composition in my tests match the runtime composition in `Startup.cs`? If they're not the same, how valid are the tests?

A solution is to use the same code both when we configure our dependencies for runtime and for our narrow integration tests. This kills two birds with one stone:

- Our integration tests are less brittle because we resolve test subjects from the IoC container instead of creating them by "manually" calling constructors
- Setting up our code this way helps us to organize our IoC setup code, even making it testable.

That sounded confusing to everyone I explained it to, and to myself
as I described it, so I'll go straight to an example.

## The Scenario
Let's start with a scenario that requires several classes that we want to test 
in collaboration. 
Coming up with fake code is harder than anything else, so this barely coherent. 
Please bear with me and keep in mind that this is about how we test the classes,
not the classes themselves.

The subject that I'm going to test is an Azure Service Bus message listener 
which responds when it receives a message. 
It in turn depends on a number of other classes:

- When it receives a message it creates a command and invokes a command handler.
- The command handler calls a validator which checks that the command contains valid data.
- The implementation of the validator makes an HTTP request to an API endpoint and receives a response indicating that the data is valid or invalid.
- If the data is valid, it's saved to a repository. If it's not, an exception is thrown.

Note that I'm not even including the message listener class here because its contents
aren't relevant. What matters in this scenario is that our application consists 
of classes we wish to test:

- message listener
- command handler
- validator

...and those we wish to mock:

- external API
- repository

For the purposes of this article I'll call those that we test "internal dependencies" 
and those that we mock "external dependencies." 
The plan is to separate the two so that in our integration tests we can easily 
replace the external dependencies with mocks while using real production code to 
configure all of the other dependencies.

## Separating Internal and External Dependencies

Just to clarify my ad-hoc terminology, by *internal dependencies* I mean those
which do not interact with external systems. These are the ones that are tested, not 
mocked. By *external dependencies* I mean external systems such as HTTP endpoints and
databases, the ones we'll mock so that we can test the internal dependencies.

Here's a class and interface used to configure an `IServiceCollection`:

```csharp
public interface IDependencyComposition
{
    void ComposeDependencies(IServiceCollection serviceCollection);
}

public class ApplicationComposition
{
    private readonly IServiceCollection serviceCollection;
    private readonly IDependencyComposition internalDependencyComposition;
    private readonly IDependencyComposition externalDependencyComposition;

    public ApplicationComposition(
        IServiceCollection serviceCollection,
        IDependencyComposition internalDependencyComposition,
        IDependencyComposition externalDependencyComposition)
    {
        this.serviceCollection = serviceCollection;
        this.internalDependencyComposition = internalDependencyComposition;
        this.externalDependencyComposition = externalDependencyComposition;
    }

    public void ComposeDependencies()
    {
        this.internalDependencyComposition.ComposeDependencies(this.serviceCollection);
        this.externalDependencyComposition.ComposeDependencies(this.serviceCollection);
    }

    public void VerifyDependencies(params Type[] dependencyTypes)
    {
        var unresolvedDependencies = new Dictionary<Type, Exception>();
        ServiceProvider serviceProvider = this.serviceCollection.BuildServiceProvider();
        foreach (Type dependencyType in dependencyTypes)
        {
            try
            {
                serviceProvider.GetRequiredService(dependencyType);
            }
            catch (Exception ex)
            {
                unresolvedDependencies.Add(dependencyType, ex);
            }
        }

        if (!unresolvedDependencies.Any()) return;

        var dependencyExceptions = unresolvedDependencies
            .Select(unresolved =>
                new InvalidOperationException($"Resolving type {unresolved.Key} failed.", unresolved.Value));
        throw new AggregateException(
            "Some dependencies could not be resolved. See inner exceptions.",
            dependencyExceptions);
    }
}
```

This leads us to create two implementations of `IDependencyComposition`: 
One for internal dependencies and one for external dependencies:

```csharp
public class FooApplicationInternalDependencies : IDependencyComposition
{
    public void ComposeDependencies(IServiceCollection serviceCollection)
    {
        serviceCollection.AddSingleton<IFooValidator, FooApiFooValidator>();
        serviceCollection.AddSingleton<IFooCommandHandler, FooCommandHandler>();
        serviceCollection.AddSingleton<FooMessageListener>();
    }
}

public class FooApplicationExternalDependencies : IDependencyComposition
{
    private readonly IConfiguration configuration;

    public FooApplicationExternalDependencies(IConfiguration configuration)
    {
        this.configuration = configuration;
    }

    public void ComposeDependencies(IServiceCollection serviceCollection)
    {
        serviceCollection.AddLogging();
        serviceCollection.AddHttpClient<FooApiClient>();

        string connectionString = this.configuration["sqlConnectionString"];
        serviceCollection.AddScoped<IFooRepository, SqlFooRepository>(provider =>
            new SqlFooRepository(connectionString));
    }
}
```
`FooApplicationExternalDependencies` demonstrates that if we depend on configuration 
values we can inject `IConfiguration`. We could also inject classes which contain the
values we need after they've already been read from our configuration. 
Either way it should be possible to write unit tests for both of these classes. 
We shouldn't pollute them with calls to static methods.

At runtime in our `Startup.cs` class we would create an instance of 
`FooApplicationInternalDependencies`, `FooApplicationExternalDependencies`, 
and `ApplicationComposition` and use them to configure our `IServiceCollection`:

```csharp
var internalDependencies = new FooApplicationInternalDependencies();
var externalDependencies = new FooApplicationExternalDependencies(configuration);
var applicationComposition = new ApplicationComposition(
    serviceCollection,
    internalDependencies,
    externalDependencies);
applicationComposition.ComposeDependencies();
```

## What Has This Accomplished So Far?
So far it looks a little bit like moving around stuff we were already doing anyway. 
But there are benefits:

First we've got a little bit of organization around this dependency setup instead of just 
dumping a wall of code into a method in `Startup.cs`. 
Granted we've only got a few dependencies so far, but as we add more we can continue to 
break up that setup into smaller methods or even classes. 
This can help us to make code reusable. 
If we find that several projects in a solution require some of the same 
components we don't need to duplicate the dependency setup in each one. 
To me that duplicate code is a sign that if the setup hasn't gotten messy 
and hard to follow, it's about to.

I found that the lack of such organization was the initial obstacle to using this 
approach to create integration tests. If our startup contains a big "blob" of container 
setup in no particular sequence it's much harder to sort through and figure out which 
parts we want to test and which parts we want to mock.

Second, we've gained the ability to test the composition of our application. 
We can write unit tests which verify that we'll be able to resolve expected types from 
the `ServiceProvider` without runtime errors. 
That's much better than finding out at runtime that we missed registering some dependency. 
We can also perform a verification at startup, like 

```csharp
applicationComposition.VerifyDependencies(typeof(FooMessageListener));
```
...so that if we missed some dependency we'll fail fast.

The third benefit comes in the next blog post, because this one already has enough code 
in it. 
Now that we've separated the dependencies that get tested from those that get mocked, 
we can re-use part of our code for setting up integration tests. 
We'll be able to

- use `FooApplicationInternalDependencies` to configure an `IServiceCollection`,
- create a class that sets up mocks for the "external" dependencies and exposes them so that we can configure their behavior and avoid creating the same mocks over and over in multiple tests,
- resolve test subjects from the IoC container so that we don't have to create them by calling lots of constructors and our tests are less brittle.

I wasn't sure if this would make any sense until I worked all the way through applying it
in a small service with a few dependencies and then using it to write a few integration tests. 

It's still experimental and could use some polish. I like that it's a low-risk experiment. 
All I'm doing is organizing some setup code I'd have to write anyway. If it doesn't work
out I can just dump all this back into `Startup.cs` where it was. 

In the <a href="/blog/2021/04/12/integration-test-experiment-2.html">next post</a> I'll share what those integration tests look like.
