---
layout: post
title:  'The Dependency Inversion Principle For Beginners'
date:   2017-04-26 07:20:58 -0400
category: blog
description: The Dependency Inversion Principle applied in .NET/C# with no complicated diagrams
---

The definition of the principle, from Robert Martin's 1996 paper [The Dependency Inversion Principle](https://web.archive.org/web/20110714224327/http://www.objectmentor.com/resources/articles/dip.pdf) is this:

>A. HIGH LEVEL MODULES SHOULD NOT DEPEND UPON LOW LEVEL MODULES. BOTH SHOULD DEPEND UPON ABSTRACTIONS.  
>B. ABSTRACTIONS SHOULD NOT DEPEND UPON DETAILS. DETAILS SHOULD DEPEND UPON ABSTRACTIONS.

What are high level modules? What are low level modules? What is a dependency and how do I invert one? Some of these questions will be answered.

Dependency Inversion is not a language-specific concept and obviously predates the .NET Framework. This explanation, however, is .NET-centric.

#### What Is a Dependency?

A *dependency* is something - another class, or a value - that your class depends on, or requires. For example:

- If a class requires some setting, it might call `ConfigurationManager.AppSettings["someSetting"]`. `ConfigurationManager` is a *dependency*. The class has a dependency on `ConfigurationManager`, which is the same thing in more words and the passive voice. But people say that.
- If you write two classes - let's say `Log` and `SqlLogWriter` - and `Log` creates or requires an instance of `SqlLogWriter`, then `Log` depends on `SqlLogWriter`. `SqlLogWriter` is a dependency.

Did you know there was a word for that? For a while I didn't. And without that word, I didn't realize that those two scenarios have something in common, that both describe classes with dependencies. It's eye-opening for me to reflect on how just acquiring a word influenced my perception and thinking.

Until I saw that fundamental commonality - *I write classes that **depend** on other classes* - there was no way to grasp the common problems stemming from *how* I depended on other classes. I saw the results: My code was difficult to maintain. Unit testing was foreign because my dependencies made my code untestable, so I relied primarily on unreliable manual testing. The term *dependency* revealed the commonality, and the Dependency Inversion principle revealed a better way to manage dependencies.

#### What Are High Level Modules and Low Level Modules?

*Updated on 10/20/2021. I originally glossed over this subject because it meant nothing to me, but I've come to appreciate this distinction much more since then.*

The first part of the definition mentions "high level modules" and "low level modules." What are "modules?" It's just a difference in terminology between C++ and C#. In C# we write classes. C++ also uses classes, but that's beside the point. Forget modules and think about classes and projects.

What are "high level" and "low level?" high level code is the part of our code that contains our application logic, like domain code or application services. It's abstract. It's the part a non-developer is most likely to understand. It shouldn't change much or at all because of how our application receives input or how it stores data.

Low level code provides concrete implementation details. It includes, for example, the Web API controllers or message listeners that receive input, or the code that interacts with a database. 

Both high level and low level code should depend on abstractions rather than being tightly coupled to each other. But the distinction between the two still matters. In particular when it comes to the projects/assemblies within our application, we can't have circular references with two projects referencing each other. We must decide which one will reference the other. That's another way of saying we must decide which project will depend on the other. To reference a project, assembly, or NuGet package is to depend on it. That's why in a Visual Studio project the other projects and packages it references are called "dependencies."

Following the Dependency Inversion Principle, our low level implementation code should reference our high level, abstract code. If that seems confusing it may be because so many solutions do the opposite. Have you ever seen an application with some sort of Service project that references a Data project full of SQL code? In that case the high level code depends on the low level code. 

How do we turn this around so that low level code depends on high level code? A simplified answer is 

- We define abstractions (like interfaces) in our high level project
- The low level project references the high level project
- The implementations of those interfaces are in the low level project

Going back to the common example of a Service project and a Data project, that means our Service project might contain an interface like this:

```csharp
public interface IFooRepository
{
    Foo GetFoo(Guid id);
}
```

The Data project references the Service project and implements the interface:

```csharp
public class SqlFooRepository : IFooRepository
{
    public Foo GetFoo(Guid id)
    {
        // Some code to get Foo from a SQL database
    }
}
```

I wrote more about this in [another post](/blog/2020/07/10/dependency-rule.html) that explains in more detail why it's important that high level code doesn't depend on low level code. 


#### What Are Abstractions?

The principle states that we should "depend on abstractions." Abstractions are generally interfaces, abstract classes, and delegates (in .NET terms.) We call it an "abstraction" because it's an indirect representation of a concrete class or method. Interfaces are used most commonly. So for the sake of discussion, applying dependency inversion means that we *depend on interfaces.* ([This article](http://blog.ploeh.dk/2010/12/02/Interfacesarenotabstractions) clarifies that interfaces are *not* abstractions, but abstractions are usually interfaces or sometimes base classes. I mention it to acknoweldge that I'm generalizing. Pretend that *interface* and *abstraction* are synonymous even though they're not.)

3/22/2018 - [This post](/blog/2018/03/18/depending-on-functions-instead-of-interfaces.html) discusses the use of functions or delegates as abstractions. 


#### How Do We Apply Dependency Inversion?

Take the example of a class that depends on `AppSettings`:

```C#
public class ClassThatDependsOnSettings
{
    public void DoSomethingThatNeedsSettings()
    {
        var someSetting = ConfigurationManager.AppSettings["someSetting"];
        var otherSetting = ConfigurationManager.AppSettings["otherSetting"];
        CallSomeOtherMethod(someSetting, otherSetting);
    }
}
```

This class depends on `ConfigurationManager`. There must be a web.config or app.config or this class can't work. Here's what the same class might look like if it depends on abstraction - in other words, it depends on an interface instead of depending on `ConfigurationManager`.

First, the interface:

```C#
public interface ISettings
{
    string SomeSetting { get; }
    int OtherSetting { get; }
}
```


Then the class:

```C#
/public class ClassThatDependsOnSettings
{
    private readonly ISettings _settings;

    public ClassThatDependsOnSettings(ISettings settings)
    {
        _settings = settings;
    }

    public void DoSomethingThatNeedsSettings()
    {
        CallSomeOtherMethod(_settings.SomeSetting, _settings.OtherSetting);
    }
}
```

Our initial requirements haven't changed - we still want to get our values from `AppSettings`, so we write an implementation of `ISettings` that does just that:  

```C#
public class AppSettings : ISettings
{
    public string SomeSetting { get { return ConfigurationManager.AppSettings["someSetting"]; } }
    public int OtherSetting { get { return int.Parse(ConfigurationManager.AppSettings["otherSetting"]); } }
}
```

This second version applies the principle of Dependency Inversion because it depends on an abstraction - the `ISettings` interface. It's a humble example and just a beginning. 

#### Why Does It Matter?

Now we can easily replace one implementation of `ISettings` with another. Our class doesn't know or care because as far as it is concerned, the implementation of the interface doesn't matter.  

Sooner or later we're going to find that we want a class to use one dependency in one scenario and a different one in another scenario. Perhaps in some cases we want those settings to come from user input instead of `AppSettings`. So we might do something funky we regret later like putting weird arguments in our constructor or other methods like `useAppSettings` and branching within the code, or we create a duplicate version of the same class, or we try to solve the problem with inheritance. I've done all three and more.

Some might call [YAGNI](https://martinfowler.com/bliki/Yagni.html) on that. We're planning for extensibility that we will likely never need. But if we intend to write unit tests for our classes then we already require that flexibility - sooner is now. We need to be able to test a class in isolation, not testing all of its dependencies at the same time. The example above is simple because the class only has one dependency. But what if our class has a few dependencies, and those have dependencies, and so on? If we haven't applied Dependency Inversion then it's impossible to test one class without also testing every class it depends on. That's not really unit testing, and it defeats one of the benefits of unit testing, which is finding bugs quickly by testing *small units of code.* If we haven't applied this principle then we likely haven't written unit tests, and we may never have experienced the awesomeness of effortlessly finding and fixing a bug in a method that we wrote three minutes ago as opposed to building the whole car, turning the key, and debugging the whole thing from end to end. The time taken to write unit tests quickly pays for itself.

Even in the simple example above, if we depend on `AppSettings`, how do we write unit tests to test our class with different settings? It's impossible because we can't have more than one version of an `AppSettings` key (unless we contort our code further to account for that.) But if we depend on an abstraction (interface) then we can use an implementation of our interface created just for testing purposes. For example:

```C#
public class TestScenarioOneSettings : ISettings
{
    public string SomeSetting { get { return "A"; } }
    public int OtherSetting { get { return 3; } }
}
```

There are different ways to create these "fake" dependencies. We can write simple classes like the one above, sometimes called *test doubles.* Tools like [Moq](https://github.com/moq/moq4) provide easy ways to create implementations of interfaces with methods and properties that return predetermined results. That multiple versions of such frameworks are created and maintained for various languages underscores that many developers value unit testing with test doubles or mocks. 

Finally, Dependency Inversion is the D in [SOLID](http://deviq.com/solid), a set of established, tested principles that improve the quality and maintainability of our code. If this is the only one of those principles we apply we'll still get some benefit. But applying one leads to applying the others. We see how much one helped so we examine another more closely. It also happens because these principles complement each other. For example, the Interface Segregation principle improves how we design the interfaces on which other classes depend. The Liskov Substitution principle ensures that our class can depend on an interface without "knowing" what the implementation is. 

#### What Gets Inverted?
You may find it useful or interesting trying to visualize what is getting inverted. But that's just the name of the principle. The principle tells us to depend on abstractions, and that's how we apply it. It doesn't tell us to invert anything.

In his paper, immediately after defining the Dependency Inversion principle, Robert Martin states,

>One might question why I use the word "inversion". Frankly, it is because more traditional software development methods, such as Structured Analysis and Design, tend to create software structures in which high level modules depend upon low level modules, and in which abstractions depend upon details.

He was recommending that developers look at the way they traditionally structured their software and managed their dependencies, and invert it. To me this makes the name of the principle (not the principle itself) self-referential. Dependency Inversion is an inversion of not applying Dependency Inversion. (He's Robert Martin, he wrote the awesome paper, and he gets to call it whatever the heck he wants whyever he wants. Plus if he called it something else it might not fit the SOLID acronym.) 

#### Other Resources
It's my hope that you can Google "Dependency Inversion" and that, as a result of reading this, you'll find more in-depth articles more accessible. You'll also see how oversimplified my examples are.

Some topics that follow Dependency Inversion are [Dependency Injection](http://stackoverflow.com/questions/130794/what-is-dependency-injection) and [Dependency Injection containers, also called IoC (Inversion of Control) containers](http://stackoverflow.com/documentation/.net/5085/dependency-injection/17949/why-we-use-dependency-injection-containers-ioc-containers#t=201704280224154117753). It's not the same as Dependency Inversion, but a way of implementing and managing it. (Both have the initials "DI" which breeds confusion.) My experience is that it's difficult to learn the *how* and the *why* simultaneously. I recommend taking a leap of faith and starting with *how* to use an IoC container, and then the *why* will make sense faster. 

Dependency Injection is even built into ASP.NET Core. Prior to that if you wanted to use an IoC container in a web application you'd have to add a 3rd-party library like Castle Windsor, Unity, or others. Clearly many developers consider it an essential pattern.

I also recommend reading all of the posts in [Robert Martin's blog](http://blog.cleancoder.com/). This blog inspires me and makes we want to be a better developer. I haven't yet adopted TDD (Test-Driven Development) as he advocates but I'm certain that I'm going to try it with serious intent.

#### Diagrams?
This is one case where I'll assert that diagrams don't help. If someone's mind is working to comprehend a new principle *and* unfamiliar terminology at once, a diagram may be just the straw that cognitively overloads the camel. Not only can they not mentally process the diagram, they may become *less* able to process what they've read than if they hadn't seen the diagram. Maybe that's just me. Maybe I'm lazy and I should try harder to follow the diagrams. Google "Dependency Inversion", check out the diagrams found alongside the explanations, and judge for yourself. 




 
