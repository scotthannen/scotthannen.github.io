---
layout: post
title:  "Our Dependencies Point the Wrong Way - The Dependency Rule Points Them The Right Way"
date:   2020-07-10 -0400
category: blog
description: Everything is better when our dependencies point the right way.
---

This post describes application of the Dependency Rule as described in Robert Martin's blog post, 
[The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html). 

From the article:

> This rule says that source code dependencies can only point inwards. Nothing in an inner circle can know anything at all about something in an outer circle. In particular, the name of something declared in an outer circle must not be mentioned by the code in the an inner circle. That includes, functions, classes. variables, or any other named software entity.

> By the same token, data formats used in an outer circle should not be used by an inner circle, especially if those formats are generate by a framework in an outer circle. We don't want anything in an outer circle to impact the inner circles.

Don't be put off by the diagram at the top of that blog post. For the purpose of this discussion, please put it
out of your mind. (Not forever, just right now.) Think smaller, simpler. This rule applies whenever we have classes in one assembly or project
that depend on classes or interfaces in another assembly or project. That's it. It applies if we have domain entities,
or if we have a very simple application composed of core functionality and some implementation details.

For simplicity, let's call that inner part - the core functionality - the *domain*. Picture those parts of our code that should
change less as inner circles, and implementation details as outer circles.

With that, let's look at a simple example: Code that depends on data from an API. Let's look at two ways of writing this code, one which follows
the Dependency Rule and one which violates it. I'll start with a violation of the Dependency Rule, because I think it's familiar to most of us. We tend to do the opposite
of what the rule recommends.

## The Violation

Suppose we have a process in our domain that needs to obtain data from an HTTP API call. We might create a
solution containing one project containing the process, and another containing our API client classes. I'll
refer a lot to an API client project, but it's really an example to stand in for all sorts of implementation 
details our domain depends on.

Perhaps the domain class looks like this:

```csharp
public class SomeDomainClass
{
    private readonly IFooApiClient fooApiClient;

    public SomeApplicationClass(IFooApiClient fooApiClient)
    {
        this.fooApiClient = fooApiClient;
    }    
}
```

...and in our API client project we have this:


```csharp
public interface IFooApiClient
{
    Task<FooDto> GetFooData(Guid id);
}

public class FooDto { }

public class FooApiClient : IFooApiClient
{
    private readonly HttpClient httpClient;

    public FooApiClient(HttpClient httpClient)
    {
        this.httpClient = httpClient;
    }

    public async Task<FooDto> GetFooData(Guid id)
    {
        // invokes HttpClient to get some data
    }
}
```

Does that look familiar? We define an API client interface alongside our implementation. The domain depends on
the API client project and injects the API client into the class that needs to make an API call. 

## Following the Dependency Rule

The difference is that the abstraction is in our domain. Instead of the domain depending on the API client and using its interface, we're doing the
opposite: The API client depends on the domain and implements its interface:

In the domain:

```csharp
public class Foo
{ }

public interface IFooDataProvider
{
    Task<Foo> GetFoo(Guid id);
}

public class Foo
{ }

public class SomeDomainClass
{
    private readonly IFooDataProvider fooDataProvider;

    public SomeApplicationClass(IFooDataProvider fooDataProvider)
    {
        this.fooDataProvider = fooDataProvider;
    }
}
```

And then, in our API client project, the API client might implement `IFooData`, or we might have an API client class and then adapt it to our interface 
like this:

```csharp
public class FooApiFooDataProvider : IFooDataProvider
{
    private readonly FooApiClient fooClient;

    public FooApiFooData(FooApiClient fooClient)
    {
        this.fooClient = fooClient;
    }

    public async Task<Foo> GetFoo(Guid id)
    {
        var fooDto = await this.fooClient.GetFooData(id);
        // catches 404 and returns null
        var foo = // map the FooDto to a Foo
        return foo;
    }
}
```

## Which Is Preferable, and Why?

At first glance they might look almost interchangeable. That might be because either way we're doing the same work:

- We define an abstraction (interface)
- We implement it
- We inject it
- We're likely going to map from `FooDto` to our own `Foo` model.
- If the API changes, we're going to have to change `FooDto` and its mapping to our model.

So what's the difference, and why does it matter?

I got hung up a few times writing this blog post because I tried to describe the pros and cons of each approach. Why didn't that work? Because it's 
so imbalanced. Violating the dependency rule is all cons. It's all negative. The benefits of following the Dependency Rule are that all those undesirable
things don't happen, plus some other benefits. So let's skip the pros and cons and talk about what goes wrong when we violate the Dependency Rule. I bet you're going to 
recognize some or all of these issues.

## What Goes Wrong When We Violate the Dependency Rule

### We're Not Depending On Abstractions

The Dependency Inversion Principle tells us to depend on abstractions. An interface named `IFooApiClient` is not an abstraction. Yes, we can mock it
in unit tests, but that's not the same thing. It represents an API client that makes HTTP requests. 

One of the reasons why we depend on abstractions is that we can, in theory, replace one implementation with another. What other implementation is going
to fulfill the `IFooApiClient` interface and return the same DTOs? We may imagine that we're not coupled to an implementation detail because we 
defined an interface, but we're about as tightly coupled as we could possibly be.

What should `IFooDataProvider` return if we request data for a Foo that doesn't exist? Perhaps it returns null. I don't like
nulls but at least it makes some sense. Or we could use a class like `Result<Foo>` or `Maybe<Foo>` which expresses clearly
that we may or may not get a result. If we adapt an API client to that interface it's much clearer how the implementation
should behave.

What does `IFooApiClient` return if we request something that doesn't exist? It might return an `HttpRequestException`. Then
when someone realizes that, they might put code in the domain class that catches `HttpRequestException` and checks to see
if the HTTP response code is 404, and that's how it figures out that there's no data available. Once that happens we
are tightly coupled to an implementation.

### Models From Outside the Domain Are Inside Our Domain
Just for emphasis: I'm using the term *domain* for convenience, but this is not specifically about Domain Driven Design. This is about any solution 
in which one project or assembly depends on implementation details provided in another.

We're likely going to create our own `Foo` model with a constructor to make sure it's instantiated correctly and useful methods that modify its state.
We might even make it immutable so that once it's constructed we always trust that it's in valid state.

But what does our Visual Studio intellisense show us when we type `Foo`? It's likely going to show us both `Foo` *and* `FooDto`. If our domain
references a project or assembly that contains `FooDto`, that class is in our domain. We will have two classes that represent the same thing. And 
quite likely out of convenience someone is going to use the wrong class. The world will not end when that happens, but now our code is going to be
slightly more confusing. Our new developer will see both and question which one to use and why. 

And then it gets worse. In addition to our API, we reference another project that retrieves Foo data from a database. What does that mean? Another Foo
model in our domain. 

It's only a matter of time before someone wants to create a model that's used by both the API and the database. They can't be in the domain because 
API project and database project (are we still calling it a "data layer?") don't reference the domain - it's the other way around. So we create a new
project that's referenced by all three and we put our new models there. That lasts a week or so until we realize that our API and database need different
models after all. (Or worse, we start passing around models where some properties are populated and others aren't, depending on whether they came from
the API or the database.) Then we start separating them again as needed, so that there are different version of different models in some cases
but not others.

If we've learned from those mistakes we might keep all of these models separate, but they're still all referenced by our domain. We create namespaces 
and naming conventions that make sense to us but to no one else and change from one project to the next.

> `FooModel` is our domain entity. `FooDto` is what we get from the API. `FooEntity` is what we store in the database. Never mind that our 
> domain entities are called "model", and technically the entity we store in the database is also a DTO (data transfer object.) You'll get used to it.

Can we keep track of all that? Yes. We can and do get by with it. It just slows us down. It creates confusion. It encourages defects. 

Here's how simple it can be: *If our domain doesn't reference the projects or assemblies those models are in, 
we will never use them in our domain.* Inside the domain is `Foo`. That's it.

### Changes Outside Create Changes Inside
All code is important and needs to work, but generally the logic in the inner circle or circles of our architecture should
be the most stable and well-tested. If our inner circle depends on code in an outer circle, changes to the outer code are more
likely to force changes to the inner code. We're going to have to make those changes either way, and sometimes changing the
inner code is unavoidable. But given a choice, shouldn't we always prefer to insulate the inner circle from change?

### Interface Segregation and Single Responsibility Violations

If we bring in outside interfaces they're likely to have methods we don't need. We might have a harder time understanding
which methods to use and when. If a class depends on an interface with lots of methods we might find ourselves adding more
and more methods to that class. We might end up with a nebulous `FooManager` or `FooService` class that encompasses anything
and everything vaguely foo related. It has its own giant interface which mirrors the API, and before we know it everything
that uses it is also coupled to the API.

The Dependency Rule doesn't prevent giant classes and interfaces. But if we define an interface that represents just what we
need and nothing more, it gently guides us toward smaller, more focused classes and away from giant, vague interfaces that 
include everything and exclude nothing.

(See [The Interface Segregation Principle Applied in C#/.NET](/blog/2019/01/01/interface-segregation-principle-applied.html).)

Following the Dependency Rule leads to some other benefits besides all those other awful things not happening as much, as soon, or ever.

## Benefits of the Dependency Rule

### Deferred Development

We're writing a domain class that's going to need some Foo data. If we violate the Dependency Rule and explicitly depend 
on an API client, we're stuck unless we have a class or an interface which represents the API. It may not be too bad if we're
requesting a Foo by it's ID, but it might be something more complex like posting some data using particular models.

If we're following the Dependency Rule it's easier to create our own abstraction. If we know that somewhere out there is Foo
data and we get it with an ID, we can define our `IFooDataProvider` interface, inject it into a class, finish it, test it, and move 
on without determining a concrete implementation.

Or we might create some data and know we'll have to post it to some API that hasn't even been created yet. We have some details
because we're communicating with other developers, so we create an abstraction like:

```csharo
public interface IResultsUpdate
{
    Task PostUpdate(Update update);
}
```

and then we just move on, knowing we can come back to the implementation later. (Side note: If we find ourselves creating
a lot of tiny interfaces with one method like this, we might consider [functions or delegates](/blog/2018/03/18/depending-on-functions-instead-of-interfaces.html)
instead of interfaces.

It gets better! We can come back to implementing the abstraction later, but we can also let another developer implement it.
This is a great way to collaborate. 

> I've finished this task. Do you need any help?
> *Yes. I've defined an interface or two that I'm depending on. It's going to be an API call. Can you create the client and adapt it to this interface? Thanks!*

Even before I heard of the Dependency Rule, grasping the benefit of this sort of deferred development was a big "lights on" moment for me. 
It's easier to maintain momentum if I work on one class, define abstractions as needed, test, and then implement the abstractions,
which in turn may have more dependencies, and so one. The Dependency Rule just makes it easier to continue that habit when dealing
with external dependencies. (Admittedly I'm not rigourous about testing each class as I write it,
but it's a lot easier to plan and write tests for code written this way.)

### Easier Decisions
The Dependency Rule is simple. "Inner" layers like the domain or supporting layers only depend or something within their circle - 
like the layer that implements domain intefaces depending on their domain. I don't recommend following rules just because they eliminate 
decision-making, but in this case it's a win-win. If we follow the rule it's going to make architectural decisions for us,
and they're good decisions.

There is no opposite to the Dependency Rule for us to choose. It's not like a current that can flow one way or the other.
We can violate it, but all dependencies can't point outward. Unless we're working on something really small, if we violate it
eventually our architecture will become somewhat more chaotic as we contort it to deal with the problems we gradually accumulate.
Once that happens it becomes less clear where and how to modify our application. (If I value a principle or rule I'll follow it
even on small projects just to reinforce it, and because small projects unexpectedly become big ones.)

## What If the API Client Has Multiple Consumers?
Suppose we have an API client library which doesn't exist soley to meet the needs of our domain. We might reference it
via a Nuget package. That library can't reference our domain, so how can it implement our domain's interface? 

Following the rule, our domain can't depend on anything outside of itself, so referencing the API client library isn't an option.
So what do we do? The same thing we would do if we *did* create our own API client project in our solution. We create a project
which references the API client library and the domain, and adapts the API client to our domain interface. 

The actual application that's running (Website, service, Azure Function, whatever) is the outer circle of our dependencies.
It references both the domain and the supporting code that implements the domain interfaces. The application's composition root
(think `Startup.cs`) is what brings it all together. If we're using Microsoft's dependency injection, that might look like

```charp
serviceCollection.AddTransient<IFooDataProvider, FooApiFooDataProvider>();
```

We might even decide to package that registration into its own project, which enables us to test and re-use some of it,
but that's another subject.

## Conclusion
We can write some great code while violating the Dependency Rule, but we're going to run into the consequences. It will slow us down.
It's not day and night, all or nothing. Adopting the rule results in improvement. Whether we're just getting by or doing really well, we can always improve. 
I don't follow principles or rules just because someone said to. (Although if you trust someone, that's not necessarily bad.) 
I can see the difference and hopefully describe the benefits clearly. I am not aware of the benefits of doing otherwise. Nothing
is carved in stone, but I'm convinced that it is a rule we should understand and adopt.
