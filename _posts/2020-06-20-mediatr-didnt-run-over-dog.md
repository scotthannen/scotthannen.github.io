---
layout: post
title:  "No, MediatR Didn't Run Over My Dog"
date:   2020-06-20 -0400
category: blog
description: We're doing something wrong with MediatR.
---

__Update 12/31/2021:__
__I've received a lot of positive feedback on this post. But eighteen months later I no longer agree with everything I wrote. I'm preserving the original post but adding some notes. I still have concerns, but those that remain aren't a big enough deal that I would object to using this in most of the scenarios where I objected before.__

I'm slightly concerned that this could be taken as some animus-driven screed against MediatR. 
But we're using it wrong, and we're using it wrong for the wrong reasons. 
Here's the really short TL;DR version:

## TL;DR
Injecting command handler abstractions into classes that depend on command handlers is good. 
Replacing that by injecting an `IMediator` and sending commands to that is probably bad. 
The reasons given for using it are often meaningless.
We often use it to implement the service locator anti-pattern, not the mediator design pattern. 


## Illustrated

This code depends on two abstractions, a command handler and a query handler. They are injected into the constructor. 

```csharp
public class DoesThings
{
    private readonly ICommandHandler<DoThisCommand> doThisHandler;
    private readonly IQueryHandler<SomeQuery, Data> someQueryHandler

    public DoesThings(
        ICommandHandler<DoThis> doThisHandler,
        ICommandHandler<DoThat> doThatHandler)
    {
	this.doThisHandler = doThisHandler;
        this.someQueryHandler = IQueryHandler<SomeQuery, Data> someQueryHandler)
    }

    public async Task DoSomething(Input input)
    {
       /// ...create a DoThisCommand
       await doThisHandler.Handle(doThisCommand)
    }

    public Data GetData(Input input)
    {
       /// ...create a SomeQuery
       var result = await someQueryHandler.Handle(query);
    }
}
```
Replacing it with this code is detrimental, not beneficial:

```csharp
public class DoesThings
{
    private readonly IMediator;

    public DoesThings(IMediator mediator)
    {
	this.mediator = mediator;
    }

    public void DoSomething(Input input)
    {
       /// ...create a DoThisCommand
       mediator.Send(doThisCommand)
    }

    public Data GetData(Input input)
    {
       /// ...create a SomeQuery
       var result = await mediator.Handle(query);
    }
}
```
For the rest of this post I'll refer to "MediatR", but to be clear I'm not referring to any and all use of MediatR. I'm referring specifically to replacing abstracted query and command handlers (the first example) with sending commands and queries through `IMediator` (the second example).

## Does This Implement the Mediator Pattern?
Articles describing the mediator pattern describe it as:
- Simplifying complex interactions between objects
- Reducing coupling by preventing objects from interacting directly with each other

When do we use this pattern? When interactions between objects have become too complex. For example:
- Performing some logical behavior requires setting a number of properties on another object 
- Groups of objects must interact with each other. A common illustration is taxi drivers communicating with each
other. Instead of each taxi driver knowing about every other taxi driver and sending messages to all of them, they
send messages to a dispatcher. Now each taxi driver has a simple relationship with the dispatcher instead of a
complex relationship with lots of taxi drivers.

How does replacing command and query handlers with MediatR fit this pattern? It doesn't. 

Injecting `IQueryHandler<FooQuery>` into a class and calling its `Handle` method is as simple as it gets. It is
absurd to replace

```csharp
var result = await someQueryHandler.Handle(query);
```

with 

```csharp
var result = await mediator.Handle(query);
```

...and try to convince ourself or others that we have replaced a complex interaction with a simpler one.

Does it reduce coupling? Not at all. Our class already depended on an abstraction. It was not coupled to any
implementation of the command or query handler. We can add decorators to the implementations if we need to. 
We can mock them in unit tests. Once we've accomplished that decoupling, adding another level of abstraction doesn't make our code super-extra
200% decoupled. It's just pointlessly "clever." 

If injecting an abstraction and calling a single method - simple dependency injection - creates complex interactions
and coupling and MediatR solves it, then shouldn't every dependency injected into every class be replaced with MediatR?
Unless we've got something far more complex going on, it's a solution we don't need to a problem we don't have.

Another way to tell whether we're actually implementing the mediator pattern correctly: The mediator will be
an abstraction we define, and its name will indicate what it mediates, like `IChatRoomMessageMediator`. If we
have two different classes doing different things and they both depend on the same `IMediator` because that's
what mediates all the things everywhere, we should revisit that.

## Does MediatR Help Us Implement CQRS?
Command/Query Responsibility Segregation (CQRS) means (in brief) that we avoid combining commands that change something with queries that return data. An operation is either a command or a query, but not both.

Look again at the two examples. Which implements CQRS? Both. Neither. Let me explain:

In both examples we have a command and a query, and they are separate. Was it a challenge to implement CQRS without MediatR? 
Not at all. MediatR did not sprinkle magic CQRS dust on our code. 
Both code samples do the same thing, the second one with a pointless extra layer of abstraction.

The answer is also that neither necessarily implements CQRS. 
What if our query handler, which is supposed to read and not write, violates that segregation by doing something 
command-like anyway? Which example prevents it? Neither. 
The only way to keep our commands and queries segregated is to segregate them. 
Nothing will enforce that for us.

I could be way off, but I suspect that some developers learn about CQRS and MediatR at the same time 
and see them as more closely related than they are. 
Or perhaps they get the impression that CQRS is inherently intertwined with the mediator pattern.
It's true that MediatR provides a handler interface, encouraging us to think in terms of handlers. 
But how long does it take to create those generic interfaces? 
Thirty seconds? About the same time it takes to add a NuGet package?

And, peculiarly, MediatR names all of its interfaces `IRequestHandler`. 
Why not `ICommandHandler` and `IQueryHandler`? 
Names matter. They don't enforce anything, but they communicate intent. 
Having commands and queries both implement `IRequest` and their handlers implement `IRequestHandler` does not express the intent to keep commands separate from queries.

## What About Reducing the Number of Injected Dependencies?
The first example above injects two dependencies, a command handler and a query handler. 
In the second example, MediatR reduces that to one dependency, `IMediator`. 
Is fewer better? It depends. In this case, no.

One of the many benefits of dependency injection is that we can easily look at a class and tell how many dependencies 
it has and what they are. 
If a class has five or more dependencies (that's an arbitrary number) then it's almost certainty violating the Single Responsibility Principle. 
If it has ten dependencies it's certainly violating the SRP. 

If performing a single logical operation involves invoking multiple dependencies, creating a single abstraction
to represent that behavior (like a facade) is beneficial. How can we tell if it is? Because instead of invoking
three methods on three dependencies, we're invoking one method on one dependency. That's a good way to 
reduce the number of dependencies. 

But if we have three or five or ten calls to different dependencies, we merge all of those dependencies into one, 
and then make the same number of calls to one dependency (MediatR), we're sweeping the problem under the rug.

It's like adding more and more unrelated methods to a single interface where they don't belong because we don't 
want to inject more dependencies. It's like packing `HandlePlaceSalesOrderCommand`, 
`HandleProductInventoryQuery`, and `HandleDeleteAccountCommand` all into one interface
and leaving the door open to add keep adding more methods. If our code allows us to do all of those unrelated things
from one class without adding more dependencies, that's a smell.

Injecting too many dependencies is undesirable, but it's a problem that reveals itself as we add the *n*th dependency. 
The ability to hide or obscure them is harmful because it makes the problem harder to see. 

Is there perhaps a misunderstanding that this is where MediatR helps us with complex interactions? It doesn't.
If we have five interactions (complex or not) with five dependencies and we replace it with the same five interactions
with one dependency, we have not reduced the complexity of interactions.

Finally, if we've got a simple class with a handful of dependencies, why are we even talking about reducing them.
A class having more than one dependency is not a problem. Unless our class has too many dependencies, reducing
the number solves a non-existent problem.

__Update 12/31/2021__
__I still think this way in most scenarios. Web API controllers are little bit different. In the type of application where we'd use MediatR they don't contain logic. They're nothing but classes that receive inputs and route them to higher-level code in the form of requests or commands. In that case, if we're going to directly inject command and request handlers we'd have to either keep controllers small to limit the number of injected dependencies or have a ton of injected dependencies. In practice neither choice feels right.__

__If it was higher level code I'd still argue that a class should have a limited number of explicit dependencies. But in well-written API controllers that constraint likely isn't useful.__

## It's a Service Locator
The service locator is a [known anti-pattern](https://blog.ploeh.dk/2010/02/03/ServiceLocatorisanAnti-Pattern/).
An example of that anti-pattern might be injecting an IoC container (like `IServiceProvider`) into a 
class so that class can resolve dependencies from it. 
A service locator version of the first two examples might look like this:

```
public class DoesThings
{
    private readonly IServiceProvider serviceProvider;

    public DoesThings(IServiceProvider serviceProvider
    {
        this.serviceProvider = serviceProvider;
    }

    public void DoSomething(Input input)
    {
       /// ...create a DoThisCommand
       var commandHandler = serviceProvider.GetService<ICommandHandler<DoThisCommand>>();
       await commandHandler.Handle(doThisCommand)
    }

    public Data GetData(Input input)
    {
       /// ...create a SomeQuery
       var queryHandler = serviceProvider.GetService<IQueryHandler<SomeQuery, Data>>();
       var result = await queryHandler.Handle(query);
    }
}
```
Just as the last section decribed, the service locator is an unhealthy way to combine multiple unrelated dependencies.

Do you see what the MediatR example has in common with the service locator example? 
With the service locator example, the only way to know what the class depends on is to look for every 
single reference to `serviceProvider` *and* to see what's resolved from it. 
It's also easy to add behaviors to a class that don't belong in it just by requesting difference services 
that are registered with `serviceProvider`. What can we request? Anything that's registered.

Code that depends on MediatR is similar. We must look at every use of `mediator` *and* see what commands or queries are sent to it. 
There could be dozens of command and query handlers registered with MediatR. 
What restricts a class from sending a different command or query, something unrelated to the purpose of that class? 
Nothing. Those new dependencies are hidden, obscured behind the `IMediator`. 
MediatR opens the door to the same code smells as a service locator if we use it as a service locator.

__Update 12/31/2021__
__This is wrong. We can resolve any service from a service locator. We can't resolve *any* service from `IMediator`. We can only send it requests which it routes to handlers.__

__This is harmless or even desirable in Web API controllers. I still wouldn't want to replace explicit dependencies with an abstraction like this elsewhere in code.__

## Other Thoughts and Concerns
MediatR scans our assemblies to find classes that implement its handler interface. 
Do we really have so many commands and handlers that registering them with our IoC container is a problem? 
Likely not. If we do, there are NuGet packages like [Scrutor](https://www.nuget.org/packages/Scrutor/) 
that do the same thing without asserting control over how we inject those dependencies. Or we can write our own. 

We should define the abstractions on which our classes depend, not import them from a NuGet package.
Try writing interfaces (or delegates) to represent your command and query handlers. 
How long does it take to create an abstraction like `interface ICommandHandler<TCommand>` or `interface IOrderStatusQueryHandler`?
It's easy.

## The Underlying Principle
Wow, that sounded dogmatic and harsh. I hope I don't sound like MediatR murdered
someone I care about in a deal gone wrong and now I'm lurking in dark alleys waiting for the chance to get even.

There is an underlying principle to what may sound like dogmatism, and it has nothing specifically to do with MediatR:

__All code must justify itself. When we write code we should be able to describe in words what problems it solves. 
This is also true of implementing design patterns or importing libraries.__

We don't need to say what our reasoning is or write it. But can we think it? 
Can we explain it if asked?

Those descriptions might sound like

>I shouldn't modify this class and everything that depends on it so that implements a different interface. 
I'll apply the adapter pattern so that I can implement this interface without breaking changes to other code.

>I need to serialize and deserialize JSON, and this well-tested library already does that.

>This code works if I write it all in one line, but it might take someone longer to figure out what it does, so 
if I break it up into intermediate steps the next person will be able to understand it quickly.

If our reason is a principle that we understand then citing the principle may be sufficient explanation.

>I understand how violating the Single Responsibility Principle creates problems later that might not seem
obvious today, so I'm going to keep these unrelated functions in separate classes.

These are not good reasons to add code or apply a pattern:

>It's the "XYZ" pattern

(without explaining why "XYZ" pattern is applicable or helpful in this case.)

>Here's an article about it

(without explaining in our own words why it's beneficial.)

>We did it here for a reason that made sense, so now we have to do it everywhere whether we need it or not
>or future developers will be confused.

(Give future developers a little credit. They'll be confused because we *did* something that made no sense,
not because we *didn't*.)

If we use MediatR it should pass that test. We should be able to explain in our own words what complex interactions it simplified.
Does it reduce coupling? Was adding MediatR really the only way to prevent that coupling? 

If we can answer some of those questions we should use it. It we can't I'm on the side of leaving it out.

## Updated Thoughts 12/31/2021
Even in Web API controllers I'd lean toward writing my own command handler abstraction for a few reasons:
- I find it confusing when that abstraction is called `IMediator` when it's not a mediator.
- It causes me to describe my commands as queries and my command handlers as query handlers.
- I'd rather not use abstractions from an external library.

I did write my own `ICommandHandler<TCommand>` implementation which I can use with any IoC container. It's not as robust as MediatR and I'd have to update it to handle many scenarios.

But none of these concerns are such a big deal that I'd object to using MediatR in Web API controllers. I've seen cases where developers start using it to replace other interactions within higher-level code. Instead of handlers having explicit dependencies it becomes `IMediator` all the way down. That's not a MediatR problem, just something weird that people did.

