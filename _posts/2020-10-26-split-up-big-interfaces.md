---
layout: post
title:  "Splitting Large Interfaces Into Smaller Ones Is Easy"
date:   2020-10-26-0400
category: blog
description: Splitting up interfaces is a first step that makes other improvements easier.
---
There are so many problems with big interfaces, that is, interfaces with lots of methods.  
  
- A big interface has a big implementation, a giant class that does lots of stuff.  
- The Interface Segregation Principle tells us that if we have a big interface and different consumers all use different methods, using the same interface needlessly couples those consumers together. That effect isn't obvious, but it's real.  
- When we inject a big interface into another class, it's not always obvious what we're using it for. Which of its twenty methods do we call? It's not hard to find out, but not having to read more code to figure that out is better.  
- When we have a big interface there's often a big class that depends on it, calling lots of its methods. That class is also doing too many things, and now we've got two big classes (the one that depends on the big interface and the implementation of the big interface) that are tightly coupled.  
  
Those giant interfaces tend to have names that end in "Service" or "Manager." Sometimes it's "Repo," as if that's a special case where a single huge interface makes more sense.  
  
Cleaning that up is challenging but so rewarding. It's technical debt that slows down anyone who has to deal with it. It also breeds more technical debt because once it's out of control it becomes easier to pile onto it than sort it out. It feels great to grapple with that, own it, and come out on the other side with something better.  
  
Fortunately the first step is the easiest, and that's just splitting up the interfaces. Let's start with a simple example.  
  
### How to Painlessly Split Up an Interface  
  
Suppose we've got a big repo interface:  
  
```csharp  
public interface IFooRepo  
{  
    Foo GetFoo(Guid fooId);  
    IEnumerable<Foo> GetAllFoos();  
    void SaveFoo(Foo foo);  
    void DeleteFoo(Foo foo);  
    void UpdateFoo(Foo foo);  
    IEnumerable<Foo> GetFooByName(string name);  
    IEnumerable<Foo> GetFoosByPartialNameMatch(string name);  
    IEnumerable<Foo> GetFoosByDateRange(DateTime startDate, DateTime endDate);  
    void RenameFoo(Guid fooId, string name);  
    bool IsFooPurple(Guid fooId, int howPurpleIsIt);  
}  
```  
  
If you've never seen an interface like this, good for you. It happens all the time, though. We inject `IFooRepo` into any class that needs to touch Foo-related data. When we have some new interaction with Foo data we add it to the interface, and eventually it gets to be like this. 
  
In addition to all the other problems with big interfaces, another reason to split this up is CQRS (Command/Query Responsibility Segregation) which recommends that an operation should either modify data (command) or read data (query) but not both. If a class depends on `IFooRepo` we can't immediately tell whether it executes commands, queries, or both. Even if we're enforcing CQRS now, it's easy to violate if we're injecting a dependency that allows executing commands into a class that should only be executing queries.  
  
How do we split this up? It's easy. First we create two new interfaces, `IFooReadRepo` and `IFooWriteRepo`. Then we look at each method in `IFooRepo` and copy it into either one or the other depending on whether it reads data or modifies data. We end up with this:  
  
```csharp  
public interface IFooReadRepo  
{  
    Foo GetFoo(Guid fooId);  
    IEnumerable<Foo> GetAllFoos();  
    IEnumerable<Foo> GetFooByName(string name);  
    IEnumerable<Foo> GetFoosByPartialNameMatch(string name);  
    IEnumerable<Foo> GetFoosByDateRange(DateTime startDate, DateTime endDate);  
    bool IsFooPurple(Guid fooId, int howPurpleIsIt);  
}  
  
public interface IFooWriteRepo  
{  
    void SaveFoo(Foo foo);  
    void DeleteFoo(Foo foo);  
    void UpdateFoo(Foo foo);  
    void RenameFoo(Guid fooId, string name);  
}  
```  
  
Then we delete all of the methods from `IFooRepo` and change it to inherit from the read and write interfaces:  
  
```csharp  
public interface IFooRepo : IFooReadRepo, IFooWriteRepo  
{  
}  
```  
  
We can do this safely without breaking anything. Our concrete repo class still implements the same interface,  and that interface still has all the same methods. It just inherits them from the two new interfaces. Our code still compiles and our tests still pass.  
  
Next we look for any classes that depends on `IFooRepo`. We'll likely find classes that depend on only read methods or only write methods. When we do, we modify those classes to depend on the smaller interface. That is, we inject `IFooReadRepo` instead of `IFooRepo`.  
  
We'll need to update our dependency injection. If we're using Microsoft.Extensions.DependencyInjection we would look for  
  
```  
serviceCollection.AddSingleton<IFooRepo, SqlFooRepo>();  
```  
  
...and add, as needed:  
  
```  
serviceCollection.AddSingleton<IFooReadRepo, SqlFooRepo>();  
serviceCollection.AddSingleton<IFooWriteRepo, SqlFooRepo>();  
```  
  
We haven't split up the implementation, just the interfaces, so we can still use the same class as the implementation for both interfaces.  
  
If we end up without any classes that depend on `IFooRepo` we can delete that service registration. Why stop there? We can delete the interface, and change `SqlFooRepo` so that it implements the read and write interfaces instead of `IFooRepo`. And then we can delete `IFooRepo`.
  
These are all low-risk refactors. The compiler is checking most of it for us. If it's beneficial we can keep going and split the repo class itself into two classes, or we can leave it the way it is. Our options are open. We could decide to repeat the process and split some of the interfaces even further. (What are the odds that one class needs to both rename a Foo and delete a Foo? Do they need to depend on the same interface?)
  
### I Can See Clearly Now  
  
Splitting up an interface might be the end of our refactoring, or it may identify more opportunities for improvement that were previously too hard to see. What if a class depends on both the read and write methods of our repo? We might see something interesting if we replace the dependency on the big interface with dependencies the separate, smaller interfaces.  
  
Quite likely if a big class depended on that big repo interface and now it depends on both the read and write interfaces, we've identified the lines where we can break up that big class too. We'll likely find that in that big class, some methods depend on one interface and others depend on the other interface. If that's the case, do they really need to be in one big class? Maybe we can split that class up too. What if that class implements an interface that other classes depend on? We can split up that interface exactly the way we split up the repo interface. It's a safe refactor, and when that step is done we can split up that class, too.

Now we're turning the tide. Our big, out-of-control classes are getting smaller because the first refactor made it easier to see through the fog and figure out where to refactor next.  
  
Static code analyzers like [NDepend](https://www.ndepend.com/) can also spot some of these issues for us. If NDepend identifies classes where some methods interact with some fields and others with other fields. Those fields are likely dependencies we've injected, or they could be variables for some other use. NDepend will point out that this [lacks cohesion](https://blog.ndepend.com/lack-of-cohesion-methods/). If a class has some methods that use some fields and other methods that use other fields, does it need to be one class? Maybe not.  

### Why Do These Big Interfaces Happen in the First Place?
I think the underlying reason why big interfaces and their big implementations happen is that we don't design interfaces with a consumer in mind. That is, we don't say
> The class I'm writing needs to depend on something that does X. Therefore I'll define an abstraction (likely an interface) that represents that need. 

If we do that, our interfaces are small and have specific names. It's hard for them to become bloated because some new behavior doesn't fit the name of the existing interface. When we need something different we define a different abstraction. This tends to lead to small interfaces. Many might have only one method. We might realize that we can [use delegates or functions instead of interfaces](/blog/2018/03/18/depending-on-functions-instead-of-interfaces.html).

Instead, we tend to say,
> This class I'm writing needs to depend on something that does X. All of our X-related stuff goes in this interface, `IAllThingsXManager`, so I'll add a new method there. It already has twenty-two methods. What's one more?

Rather than creating interfaces as specific abstractions, we create them as groupings of methods that deal with some related data. We might have customer data and pile anything and everything vaguely related to interacting with it into an `ICustomerService` or `ICustomerManager` interface. Once that happens, it snowballs. Anything vaguely related to customer data gets added.

Do we perceive adding methods to classes and interfaces as preferable to creating new classes and new interfaces? I think sometimes we do. It's a habit to unlearn. It costs nothing to add new abstractions and new classes to fulfill new requirements. 

Ask yourself this: How many times have you had a giant class with 500 or 2,000 lines and wished it was smaller? How many times have you had a set of small, specifically-named classes and wished you could refactor them into a single giant one? I'd bet the first one happens all the time and the second one never. So why not write the code we want and stop writing the code we don't want?

### Conclusion  
  
Hopefully I've made a case for avoiding big interfaces in the first place. That's easier than splitting them up. Interfaces that can be split up but aren't create technical debt. Our code works and we can understand it, but that understanding takes just a little longer than it should. We end up with larger classes that do too many unrelated things. Those small increments of time that we spend reading and figuring out large interfaces and classes add up. Worse, one day we have to make a significant change and we realize that it's become very difficult to figure out how to do it.  
  
But too often we have no control over that. We might start a project long after it's gotten unwieldly, or maybe it's our own mess and we're just now realizing that it's hard to work with. Either way, we don't have to live with those past choices. Splitting up interfaces is one of the easiest ways to improve our code, and it often blows away some fog and makes it easier to see what else we can refactor.