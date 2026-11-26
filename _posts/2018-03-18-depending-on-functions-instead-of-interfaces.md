---
layout: post
title:  "Depending on Functions Instead of Interfaces - Why and How"
date:   2018-03-18 07:20:58 -0400
category: blog
description: Using delegates instead of interfaces can save us more than just a few lines of code.
---

Here's a simple, contrived example of a class that depends on an interface, `IDoesMath`, followed by the same class modified to depend on a function instead. As you'll quickly see, there is no _immediate_ benefit. The change doesn't make this a better class. So are there any benefits to depending on functions rather than interfaces? I think that at least in some cases, depending on functions has potential to improve the long-term maintainability of our code. (Can I be any more careful to qualify that statement?)

#### Depends on an interface
``` 
public class ClassThatDependsOnMath  
{ 
    private readonly IDoesMath _math; // interface

    public ClassThatDependsOnMath(IDoesMath math) 
    {
        _math = math; 
    }

    public Single Calculate(Single value1, Single value2)
    {
        return _math.DoMath(value1, value2);
    }
}
```

#### Depends on a function  
``` 
public class ClassThatDependsOnMath
{
    private readonly Func<Single, Single, Single> _doMath; // function

    public ClassThatDependsOnMath(Func<Single, Single, Single> doMath)
    {
        _doMath = doMath;
    }

    public int Calculate(Single value1, Single value2)
    {
        return _doMath(value1, value2);
    }
}
```

Again, I don't see a compelling difference. The interface actually makes it a little bit easier to see what the dependency does. (More on that later.) So why would we ever consider the second approach?

### How Depending on Functions Can Make a Difference

Depending on functions has at least two potential benefits:  
- It enforces the Interface Segregation Principle. You can use some methods of an interface but not others. A function is all-or-nothing. You use it or you don't.
- It encourages following the Single Responsibility Principle. That doesn't exactly translate to smaller classes and smaller methods, but in practice that's often the outcome.

I've seen too many cases where a new method gets added to an existing interface and its implementation(s) because it's momentarily convenient. Then another gets added, and another, until the interface is full of non-cohesive methods that don't make sense together, and the classes that implement them are huge. Eventually lots of classes depend on that interface, each using just one or two methods. If we've already gone down that dark path then for each class that depends on the interface there's the risk that someone modifying that class will find a reason to add more methods to the interface. (One window is broken, anarchy ensues.)

That violates the Interface Segregation Principle because now classes depend on the interface but ignore most of its methods. Eventually we'll need to create a new implementation of the interface for one specific scenario, and we'll have to implement all of the other methods even though we don't need them. Or we leave the methods we don't need unimplemented, but that's just evil. 

Another side effect is that when we write unit tests for classes that depend on the interface we have to figure out which methods it uses so we know which ones to mock. Eventually we end up with mocks for methods we don't even use. Now our unit tests are hard to read because they're polluted with code they don't even need, and we have to maintain them because of changes that don't actually affect them. (Have you ever changed an interface and found yourself fixing compile errors in two dozen mocks that you may or may not even use because it's faster than figuring out if you need them? I have.)

Meanwhile, if the interface has grown out of control, its implementation(s) almost certainly violate the Single Responsibility Principle. So do the classes that depend on them. If we're using constructor injection we can look at a class's constructor to see what it depends on, and we can tell when we're adding responsibility because we add more dependencies. Adding new methods to existing interfaces makes that less visible.

Depending on functions doesn't necessarily prevent all of that, and it's not the only solution. (Maybe we just need a verb in our interface name so that it more clearly reflects what it should and shouldn't contain.) But it is a defense that can guide us away from adding bloat to our interfaces and reveal when we're adding responsibilities to classes. 

But how does it impact readability and the way we configure our dependency injection container?

### Delegate vs. Func or Action

In the example above I showed a class that depends on a math function: `Func<Single, Single, Single>`. If we're doing constructor injection then we're likely using a dependency injection container. In that case we're going to run into a few problems:
What if different classes depend on functions that serve different purposes but have the same signatures?
Also, our container setup code would be confusing. If we register an implementation of `Func<Single, Single, Single>` then nothing tells us what that function does or why something depends on it.

We can solve both problems by declaring delegates and depending on them instead of just function signatures. Now our math example might look like this:

``` 
public delegate int DoMath(Single value1, Single value2);

public class ClassThatDependsOnMath
{
    private readonly DoMath _doMath;

    public ClassThatDependsOnMath(DoMath doMath)
    {
        _doMath = doMath;
    }

    public int Calculate(Single value1, Single value2)
    {
        return _doMath(value1, value2);
    }
}
```

Now when we register implementations with our containers, we're registering implementations for `DoMath`. It's explicit, just like when we register interface implementations.

But what does that container registration code look like? I had to know how easy or ugly it might be, so I tested with two popular containers, [Autofac](https://autofac.org/) and [Castle Windsor](http://www.castleproject.org/).

In both cases I want to register this delegate

``` 
public delegate Single DoMath(Single value1, Single value2);
```

with this implementation:

``` 
public class AddsNumbers 
{
    public Single Add(Single value1, Single value2)
    {
        return value1 + value2;
    }
}
```

### Registering Delegates With Autofac
Assuming that I want to resolve everything from the container (because a class implementation might have its own dependencies to resolve) I need to
Register the class implementation (just once, not once for each delegate)
For the delegate, register a function that resolves the class implementation and then returns the desired method.

It looks like this:

``` 
builder.RegisterType<AddsNumbers>();
builder.Register<DoMath>(c =>
{
    var componentContext = c.Resolve<IComponentContext>();
    var addsNumbers = componentContext.Resolve<AddsNumbers>();
    return addsNumbers.Add;
});
```

Yes, you have to resolve `IComponentContext` _from_ `IComponentContext`. That's not pretty, but we can write an extension that makes it better:

``` 
public static class AutofacBuilderExtensions 
{ 
    public static IRegistrationBuilder<TDelegate, SimpleActivatorData, SingleRegistrationStyle> RegisterDelegate<TDelegate, TSource>( 
        this ContainerBuilder builder,  
        Func<TSource, TDelegate> extractDelegate,  
        string sourceComponentName = null,  
        string registeredComponentName = null)  
        where TDelegate : class
    {
        var registrationFunction = new Func<IComponentContext, TDelegate>(context => 
        { 
            var c = context.Resolve<IComponentContext>(); 
            var source = sourceComponentName == null 
                ? c.Resolve<TSource>() 
                : c.ResolveNamed<TSource>(sourceComponentName); 
            return extractDelegate(source); 
        }); 

        return registeredComponentName == null ? 
            builder.Register(registrationFunction) : 
            builder.Register(registrationFunction) 
                .Named<TDelegate>(registeredComponentName); 
    } 
}
```

This includes a few extra parameters in case we need to specify a named instance of a dependency to resolve, and/or we want to specify a name for our new component registration. Now the component registration looks like this:

``` 
builder.RegisterType<AddsNumbers>();
builder.RegisterDelegate<DoMath, AddsNumbers>(addsNumbers => addsNumbers.Add);
```

I can live with that, especially if it helps prevent other code maintenance issues.

### Registering Delegates With Windsor

The process is identical, so I'll skip straight to the extension methods.

``` 
public static class WindsorRegistrationExtensions
{
    public static ComponentRegistration<TDelegate> RegisterDelegate<TDelegate, TSource>(
        this IKernel kernel, 
        Func<TSource, TDelegate> extractDelegate, 
        string sourceComponentName = null, 
        string registeredComponentName = null) 
        where TDelegate : class
    {
        var component = Component.For<TDelegate>().UsingFactoryMethod((kernel1, context) =>
        {
            var source = sourceComponentName == null ? kernel1.Resolve<TSource>() : kernel1.Resolve<TSource>(sourceComponentName);
            return extractDelegate(source);
        });
        kernel.Register(registeredComponentName == null ? component : component.Named(registeredComponentName));
        return component;
    }

    public static ComponentRegistration<TDelegate> RegisterDelegate<TDelegate, TSource>(
        this IWindsorContainer container, 
        Func<TSource, TDelegate> extractDelegate, 
        string sourceComponentName = null, 
        string registeredComponentName = null) where TDelegate : class
    {
        return container.Kernel.RegisterDelegate(extractDelegate, sourceComponentName,
            registeredComponentName);
    }
}
```

And the usage (in this case including a component name):

``` 
Kernel.Register(Component.For<AddsNumbers>());
Kernel.RegisterDelegate<DoMath, AddsNumbers>(addsNumbers => addsNumbers.Add, registeredComponentName:"Adds");
```

### Mocking a Delegate

Stupid delegate! You're just one method and nobody likes you. (That never gets old.)

What about mocking as it relates to unit tests? Suppose we have this class which depends on our `DoMath` delegate, and we want to mock that delegate so that always returns 99:

``` 
public class DependsOnMathFunction
{
    private readonly DoMath _doMath;

    public DependsOnMathFunction(DoMath doMath)
    {
        _doMath = doMath;
    }

    public Single ThisIsRedundantButDoMyMath(Single number1, Single number2)
    {
        return _doMath(number1, number2);
    }
}
```

We could use an anonymous function:

``` 
[TestMethod]
public void TestWithMockedDelegate()
{
    DoMath doMathMock = (value1, value2) => 99; // That was easy!
    var subject = new DependsOnMathFunction(doMathMock);
    var output = subject.ThisIsRedundantButDoMyMath(1, 2);
    Assert.AreEqual(99, output);
}
```

Or if we need to we could use Moq:

``` 
var doMathMock = new Mock<DoMath>();
doMathMock.Setup(x => x(It.IsAny<Single>(), It.IsAny<Single>())).Returns(99);
```

We could also use a method declared in our test class or in a static class.

### How Does This Affect the Way We Implement Dependencies?

This primarily affects how classes "see" the methods they depend on, not how we write the classes those methods are in. Still, we might see two differences:  

First, the classes may not need to implement interfaces. I'm in the habit of writing lots and lots of interfaces. Some people think that too many interfaces are a form of clutter. If we're not going to use the interface because we're referencing a method via a delegate instead then we don't need the interface, so why create it?

Second, the function to which a delegate points could be static. 
I used to think that static methods were bad because if a class calls a static method, that method can't be mocked or replaced, making the class harder to unit test. 
I was partially right and partially very wrong. That's true if a class calls a static method _directly_. 
But a delegate is an abstraction which can be mocked. 
Static functions are fine as long as they don't depend on or modify any global state. 

That makes our container setup a whole lot simpler because there are no classes to register or instances to resolve.

```
// Autofac
builder.Register<DoMath>(context => MyStaticMathClass.Add);

// Windsor
Kernel.Register(Component.For<DoMath>().Instance(MyStaticMathClass.Add));
```

For some developers the ability to depend on static functions is a better reason to depend on delegates or functions than anything else I've mentioned. It's a step toward functional programming. It challenges my ingrained habit of writing and implementing interfaces, which feels like having a tiny part of my brain rewired. I like that.

## Navigating to Delegate Implementations
A reader helpfully pointed out that in Visual Studio it's easy to navigate to the implementation(s) of an interface.
How do we navigate to implementations of a delegate? Read [Depending on Functions Instead of Interfaces - The Navigation Problem](/blog/2018/04/09/depending-on-functions-navigation.html).
