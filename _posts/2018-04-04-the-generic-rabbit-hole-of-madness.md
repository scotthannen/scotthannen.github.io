---
layout: post
title:  "The Generic Rabbit Hole of Madness"
date:   2018-04-05 05:20:58 -0400
category: blog
description: Stop trying to solve your generic problem or you'll go insane.
---

Have you ever written a few generic classes and found yourself struggling to write yet another interface and class that interacts with those generic classes? Maybe something like this:

```
public interface IMultipleMappingCommandExecutor<TRepository, TData, TResult>
    where TRepository : IRepository<TData>
{
    IEnumerable<TResult> ExecuteCommands<ICommandInput, ICommandOutput>(
        IEnumerable<Command<ICommandInput, ICommandOutput>> commands)
        where TCommandInput : ICommandInput
        where TCommandOutput : ICommandOutput;
}
```
If you've gone down this path, you've likely experienced the frustration of cycling through these various states:  
- It won't compile.
- It compiles but our generic arguments require us to specify what the types in our `IEnumerable<TSomething<TSomethingElse<TWhat>>>` are, when we really wanted them to be anything and everything at the same time, because isn't that what generics are for?
- We create interfaces or abstract classes and constrain our generic types to implement them, like `where TCommandInput : ICommandInput`, and now we can use the type of collection we want and it compiles...
- ...But now our generic method doesn't know what the actual types in the collection are. It just knows that they implement `ICommandInput` or `ICommandOutput`. So our method has to check the actual type of the object and branch accordingly.
- We try to fix it by adding more generic arguments.
- We try to fix it by adding more base classes.
- We Google "covariant" because it's in a compiler error and then read the
  first three paragraphs of [Covariance and Contravariance in Generics](https://docs.microsoft.com/en-us/dotnet/standard/generics/covariance-and-contravariance).
- We go for a walk or play some ping pong to clear our head.
- We try replacing one or more of our types with `dynamic`.
- We post a question on Stack Overflow, replacing all of our class names with `Base`, `A`, `B`, and `C`, and our interface name with `IFruit<Base>`. The comments ask for clarification but we're having  a hard time remembering what we were trying to do.
- It works, somehow, but now we need it to do something slightly different, and our head explodes.

I've done all of the above (except the Stack Overflow question which is real and frequent) enough times that I gave it a name: *The Generic Rabbit Hole of Madness*. 

## Why Does This Happen?
As children, how often did we use our Legos to build lots of really small objects? If we had a few decks of cards, how often did we build fifty really small houses of cards? More likely we were focused on building some glorious, towering, complex architecture.

Perhaps that understandable mentality influences us later in life as software developers, so that we find ourselves trying to create larger, more complex classes rather than smaller, simpler ones. In addition, many introductory courses and books about object-oriented programming overemphasize the use of inheritance without warning of the dangers. 

When we first create a hierarchy of inherited classes, we may experience the illusion that we're building an impressive structure, just as we did with our Legos. In reality, we're just creating a lot of individual classes that are coupled to each other. An inherited class may seem like it builds on top of its base class, but it's really just a new class that depends on that base class. 

Generics are like some awesome new add-on set for our Legos with wheels, action figures, or lasers. Now we can build something way cooler with more moving parts and show it to our friends. It's only natural that when we first get them we'll go a little overboard.

## How to Step Back From The Edge
When I've gotten carried away building a bunch of interfaces like the ones in the example above, or maybe some base classes, most of the time I hadn't written any concrete, useful code to implement any of it. I was just enamored with the idea that my actual code would fit into this impressive framework I was creating and I spent way too much time on that. And almost without exception, the second I started coding something real (if I got that far) I realized that it wouldn't fit my generic monstrosity. 

The antidote to the Generic Rabbit Hole of Madness is to steer away from that distraction and just write the actual code that we need to do the actual thing that we need it to do right now. And write unit tests. If we begin writing more similar code and start to see a pattern develop, that's when we may see how to reuse common elements, which may include using generics. But whether or not that happens we're actually producing something that works and not just spinning our wheels.

## A Simple Example of a *Useful* Generic Class

A good illustration of why we use generics is `List<T>`. Before .NET 2.0, if we wanted a strongly-typed collection we had to write a lot of convoluted, repeated code. Many projects had classes for such collections like `SalesOrders`. `Contacts` , or `Products` which were all 100% identical, line-for-line, except for the type they collected. 

With .NET 2.0 and generics, all of that boilerplate code was perfected and moved into the framework itself so that we can reuse it and we only have to specify the type. We even have specialized collections like `HashSet<T>`, `Queue<T>`, and many more to address all sorts of scenarios, whereas writing and maintaining that code for numerous types would have been unrealistic. 

What does that show us? That we should use generics when we find ourselves writing or needing duplicate code that only varies by the type or types on which it operates. And more often than not, we're only going to identify such duplicate code *if we write some code in the first place.* Sometimes we can see a little bit into the future, or see around the corner. We usually can't. That's why we write the best code we can write today, and refactor as we go, including when we realize that we're about to introduce duplicate code.

I've lost way too many hours to the Generic Rabbit Hole of Madness, and I've been far more productive since recognizing and breaking the habit. Hopefully my experience will save someone else that time. If go down that hole and after an hour you haven't found the bottom, you likely never will. 
