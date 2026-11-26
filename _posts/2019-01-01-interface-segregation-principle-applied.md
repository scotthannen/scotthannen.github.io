---
layout: post
title:  "The Interface Segregation Principle Applied in C#/.NET"
date:   2019-01-01 -0400
category: blog
description: Practical application of the Interface Segregation Principle in C#/.NET
---
TL;DR:

- The Interface Segregation Principle says that a client class should not depend on *part* of an interface. 
- Giant interfaces with lots of methods are undesirable, but that's not the point of the ISP.
- If a class implements an interface and some of its methods throw a `NotImplementedException`, that's bad, but has nothing to do with the ISP.
- One way to apply the ISP is to write interfaces for dependencies from the perspective of the clients that depend on them so that there are no extra methods.

---

The common definition of the Interface Segregation Principle (ISP) is:

> No client should be forced to depend on methods it does not use.

This predates the .NET Framework, so how can we apply it to .NET code?

 - The "client" is a class that depends on another class or interface.
 - The word "interface" doesn't refer specifically to the `interface` keyword. A class could depend on the public members of another class. 
 But we often write our "client" classes to depend on interfaces defined with the `interface` keyword. 

So, while this post (and many others) apply the principle to interfaces declared with the `interface` keyword, the principle is not limited to that.

The word "depend" can be a little confusing in this context. How can we depend on a method but not use it? What this actually means is that we shouldn't depend 
on an interface which contains methods we don't use. 

### A Very Simple Example

```csharp
public interface IDoesStuff
{
    void DoSomething();
    void DoSomethingElse();
    void DoesEvenMore();
}

public class ClientThatDependsOnInterface
{
    private readonly IDoesStuff _dependsOnThis;
 
    public ClientThatDependsOnInterface(IDoesStuff dependsOnThis)
    {
        _dependsOnThis = dependsOnThis;     
    }

    public void ClientMethod()
    {
        _dependsOnThis.DoSomethingElse();
    }
}
```
In this example `ClientThatDependsOnInterface` depends on `IDoesStuff` but does not use two of its three methods. That doesn't mean that we 
should change `ClientThatDependsOnInterface` so that it uses all three methods. Rather, it should just depend on a different interface that only 
has the method it needs. 

### What It Isn't

I've read a few posts about the Interface Segregation Principle that seem to be more about Single Responsibility and Liskov Substitution. 

Some point out that our interfaces should be small, having fewer methods. That's true, but that relates more to the Single Responsibility Principle. 
If our interface has 23 methods and a class depends on all 23 of them then we have some serious code smells, but the client is *not* forced to depend on methods 
it does not use. Both the interface implementation and the class that depend on it probably have too many responsibilities.

Others point out that if our interfaces have too many methods we might find ourselves creating classes that implement the interface but throw a `NotImplementedException` 
for certain methods that we haven't implemented. There's an ISP violation because a class that depends on the interface *can't* depend on all of its methods. 
If it did, it would throw a `NotImplementedException`. But it's still possible (and far more common) to violate the ISP *without* throwing a `NotImplementedException`, 
so it's still not quite on target for illustrating the ISP. Throwing that exception is more of a Liskov Substitution problem. The ISP is about depending on an interface with methods we don't need, 
not creating classes that fail to fully implement an interface.

As a side note, I sometimes feel a little pretentious throwing about the names of these principles, like it's supposed to sound smart or put me in league with the big 
minds who formulated them. That's not my intent. If it is, I don't think it's working. I refer to them because, in my opinion, they're just really good principles. 
They weren't invented out of thin air. They were the results of many years of experience and analysis. Then, in my own experience, I find that many issues in the code 
I work with are traceable to violations of these principles. That makes them good guidelines to follow. No one says that we must rigidly obey them without exception, 
but we should understand that the accumulation of such exceptions is what makes our code a mess. 

### What It Is

To quote [Uncle Bob's article in which he defines the principle](https://drive.google.com/file/d/0BwhCYaYDn8EgOTViYjJhYzMtMzYxMC00MzFjLWJjMzYtOGJiMDc5N2JkYmJi/view):

> ...when a client depends upon a class that contains interfaces that the client does not use, but that other clients do use, then that client will be
> affected by the changes that those other clients force upon the class.

Suppose we have an interface with three methods. Then we have three classes that depend on the interface, each depending on a different method. The three classes, 
while unrelated to each other, are in a way coupled to each other. If we change one method to account for the needs of the one class that depends on it, we're still 
changing the interface used by two other classes that don't need to change. Then changes to the implementation of the interface are more likely to impact classes that 
depend on the other two methods. 

In effect, all classes that depend on the interface become coupled to each other: 
 - One class requires a change to the interface to support its own needs. 
 - The change to the interface forces a change to the class which implements it.
 - The change to the class potentially impacts other clients of the interface.

The article does refer to "fat" interfaces, but it's not about the number of methods:

> Classes that have "fat" interfaces are classes whose interfaces are not cohesive. In other words, the interfaces of the class can be broken up into groups 
> of member functions. Each group serves a different set of clients. Thus some clients use one group of member functions, and other clients use the other groups.

As mentioned in the outset, this isn't just about interfaces. It applies when different clients use different methods of a single class. We could apply the principle to our interfaces 
and disregard it in our classes if a dozen classes depend on a dozen interfaces but then one class implements all twelve interfaces. The reason why I apply this more to interfaces is 
because the Single Responsibility Principle independently guards against classes that do too many unrelated things.

Chances are that if we violate it in our interfaces we'll violate in our classes too. I ran an NDepend analysis of one codebase and found a large class implemented by a large interface. 
Two sets of methods were used by two sets of clients. NDepend flagged it as not *cohesive* and recommended breaking it up. How could it tell? Because 
because two sets of methods used two distinct sets of dependencies and member variables with no overlap. Just as described above, 
the class was actually two unrelated classes that ended up as one when it could be cleanly separated into two classes implementing two interfaces. That's
what I had to do when we needed a completely different implementation for roughly half the methods in the class. It required changes to lots of classes 
I wouldn't have had to touch if they didn't all depend on one giant interface.

Why do we do that? At some point someone needed new functionality not related to any existing class, and chose to add it into an existing class anyway
instead of creating a new one. It's not like we pay some extra licensing fee every time we use the `class` and `interface` keywords.

### How To Created Segregated Interfaces

Even though big interfaces are a potential problem, the ISP isn't about the size of interfaces. It's about whether classes use the members of 
the interfaces on which they depend. Remember the example of the violation? The problem wasn't that the class needed to use the whole interface. It was that 
the interface needed to be limited to what the class needed. Except for a few broad, general-purpose interfaces like loggers, that's only possible if 
the interface was written from the perspective of the class that needs to depend on it.

To summarize, this is how we can apply the Interface Segregation Principle in C#:

**Define interfaces from the perspective of the classes that depend on them.**

Don't just take an existing interface that looks similar to the one your class needs and modify it or add methods to it. Determine 
what your class needs to depend on, and describe it with an interface. 

Suppose your class needs to validate a sales order to make sure it doesn't contain errors. There might be some giant, vaguely-named interface like 
`IOrderService` with three dozen methods. Don't add another. Maybe that interface already contains the method you need. Consider not using that interface anyway.

Instead, just define the interface your class needs:

```csharp
public interface IOrderValidator
{
    void ValidateOrder(Order order);
}
```

If there's no implementation then you can create a new single-responsibility class to implement it. If there is then you can adapt the existing 
implementation to your interface or refactor part of it into a separate class. This way you either prevent unnecessary coupling or at least avoid making it worse. 

I've found that this approach helps me to work on one class at a time, not getting wrapped up in how the dependencies will be implemented. 
This class needs its orders validated, so it depends on `IOrderValidator`. I don't care about the implementation right now. I can work on that  
next, later, or never. By "never" I mean that another developer can implement that interface. Working on one class at a time keeps me moving along.

Sometimes I also see StackOverflow questions where developers are trying to figure out complex generic types and inheritance that don't make any 
sense and waste a lot of time. I've [done it myself over and over](http://scotthannen.org/blog/2018/04/05/the-generic-rabbit-hole-of-madness.html). 
This is a little off-topic from the ISP, but working this way can also steer us out of that rut. Instead of creating complex classes and then figuring 
out how we're going to use them, we work the other way - starting with the interface we need to implement and implementing it. That moves us 
toward writing the code we need instead of solving weird, pointless problems we thought up for no reason.

### Delegates Are Segregated Interfaces
This has [captured my interest lately](http://scotthannen.org/blog/2018/03/18/depending-on-functions-instead-of-interfaces.html). You'd think I bought stock in delegates and want the price to go up. But in a nutshell, delegates are 
interfaces for a single method, which makes them about as segregated as possible. 

If I define and depend on this delegate:

```csharp
public delegate OrderValidationFunction(Order order);
```

...then I'm not going to depend on a method I don't need.

Another nice side effect is not having single-method interfaces with weird, redundant names like `IOrderValidator` and `ValidateOrder`. 
`IOrderValidator` just means "interface with a `ValidateOrder` method." It gets worse if it's a method to retrieve some data. If the method 
is `GetOrderValidationRules` then what do you call the interface? `OrderValidationRulesProvider`? If I'm saving and retrieving data then 
perhaps that's a repository. But if I'm just "getting" something then I probably don't need a whole new interface.

### Segregated Interfaces Make Unit Testing Easier
Have you ever had to unit test a class that depends on an interface with 40 methods but uses six of them, perhaps one in this method, two in 
this method, and so on? You have to read through various methods to figure out which interface members to mock for which tests. In some cases 
developers just give up and create a giant test setup that mocks everything. It works, but now the tests are incomprehensible and unmaintainable.

Mocking an interface with a handful of methods is easy. Mocking a delegate is even easier.

All of this is somewhat related to the Single Responsibility Principle. To me, SRP violations are where the headaches really pile up. The ISP helps 
us avoid the backwards pressure to add responsibilities to existing classes. Like other SOLID principles, it doesn't prevent us from writing bad code. 
It influences us toward writing better code. 

I haven't enabled comments on this site, but you can comment [here](https://dev.to/scotthannen/-the-interface-segregation-principle-applied-in-net-2edd).

