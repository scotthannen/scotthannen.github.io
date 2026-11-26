---
layout: post
title:  "Extension Methods in .NET for Beginners"
date:   2016-06-25 07:21:58 -0400
category: blog
description: extensions extension methods .net beginners
---

Extension methods are a convenience - a big convenience - that effectively allow you to add methods to an existing class without actually modifying that class. 

## Syntax

Extensions are static methods in static classes, like this:

``` csharp
public static class IntExtensions
{
    public static bool IsInRange(this int input, int minimum, int maximum)
    {
        if(minimum > maximum) throw new ArgumentException("Minimum cannot be greater than maximum.");
        return input >= minimum && input <= maximum;
    }
}
```

This enables us to call `IsInRange` as if it was a method belonging to `int`.

``` csharp
var x = 10;
var isValid = x.IsInRange(1, 100);
```

What causes the static method `IsInRange` to behave as an extension is the keyword `this`.  
Notice that even though the `IsInRange` method has a parameter (`string int`) we're not explicitly passing it.  
Instead, `x` is passed as the `input` parameter. Only one parameter per method can be prefaced with `this`.

Functionally this is exactly the same as calling

``` csharp
isValid = IntExtensions.IsInRange(x, 1, 100);
```

That's why I call it a "convenience." It's just allowing me to pretend that `IsInRange` is a method of `int`.
If `x` is an `int` and we type `x.` or even if we type `5.` then Visual Studio's intellisense will show us the extension method.

They are useful in two overlapping scenarios:

## You can't add new methods to an existing class

Obviously we can't modify the `int` class, as it's part of the .NET framework. Extensions allow us a way around that.

## You shouldn't add new methods to an existing class.

This is even more important. If we *could* modify the `int` class and add an `IsInRange` method to it, *should* we?  
We probably shouldn't. Why not?

### The Open/Closed Principle

The [Open/Closed Principle](http://joelabrahamsson.com/a-simple-example-of-the-openclosed-principle) states:  

> Software entities (classes, modules, functions, etc.) should be open for extension, but closed for modification.  

In other words, the `int` class is done. It does what it needs to do and now it's finished. If we modify it then perhaps we'll break it. Do we want to test every single method and behavior of the `int` class, or risk having other classes that use `int` break? No, we want to leave it alone. Instead, just as the principle suggests, we're *extending* it. The same is true if it's our own class. 

### The Single Responsibility Principle

We may want to be able to call a method on a class, but does that mean that the class should have that method?  
Consider, for example, `List<T>`. If we have a list of strings (`List<string>`) and some other collection of strings - another list, or an array - it could be useful to know which elements the two lists have in common. But the Single Responsibilty Principle indicates that a class shouldn't take on too many responsibilities. The responsibility of a `List` is to contain the items that we add to it, not to inspect another `List` or an array and see what matches. So even if we could modify that class, it wouldn't make sense to add that method to `List<T>`.  
(Side note: Neither would we want to add that functionality by creating a new class that inherits from `List<T>`. That's evil, but another story: [Prefer composition over inheritance](http://stackoverflow.com/questions/49002/prefer-composition-over-inheritance).)

In this case we don't need to write an extension. It's already provided for us in the Linq namespace, which includes the [`Intersect`](https://msdn.microsoft.com/en-us/library/bb460136(v=vs.100).aspx) extension. 

That enables us to use the `Intersect` method on `List<T>` or any other class that implements `IEnumerable<T>`.

``` csharp
var list = new List<string>(new[] {"A", "B", "C", "D"});
var array = new[] {"B", "C", "G"};
var intersection = list.Intersect(array); // B, C
```

This is not to say that we should not add methods to existing classes. But we should ask whether it really makes sense to do so if we can just provide an extension method. Are we considering adding the method because we need it in just one scenario? If so, the existing class is  likely fine just as it is. Both the Open/Closed Principle and the Single Responsibility Principle indicate that we shouldn't necessarily modify a class just because we need to do something different with it.

## Context-specific methods

A class might only need a method in certain contexts. For example, suppose you have a class that represents a triangle, and you want a method to draw it on a Windows form. You might use the same `Triangle` class in multiple applications - some are Windows forms, some are websites, etc. It wouldn't make sense for the `Triangle` class itself to have a method like `Draw(Form form, Point location)`.  
But we could create that method as an extension in our Windows Form application or in a library used by that application. That way if we're using the `Triangle` class in that context we'll have access to that method, but in a different context that method won't exist.

## Caveats/other thoughts

### Don't use them for evil

With great extensibility comes great responsibility. When we create extensions we're really creating static methods in static helper classes. That can be an anti-pattern, allowing us to put functionality where it doesn't belong. Static classes can also make testing more difficult (or even impossible.)

For those reasons extension classes should be small and stateless. They're not a place to put any sort of application logic. This, for example, is bad:

``` csharp
public static class UserExtensions
{
    public static void SaveToDatabase(this User user)
    {
        var connectionString = ConfigurationManager.ConnectionStrings["db"].ConnectionString;
        // Open a database connection and save the User to the database
    }
    
    public static void ValidateNewUser(this User user)
    {
        // Not a good place to put the rules for validating a user.
    }
}
```

If we do that then any class that calls `user.SaveToDatabase()` will be difficult to test. The unit test will depend on the connection string being in app.config, availability of the database, etc. If a class needs to save a `User` then we'd rather inject an interface like `IUserRepository` into that class. (See [Dependency Injection](http://scotthannen.org/blog/2016/04/13/wcf-dependency-injection-in-5-minutes.html). That's a different subject.) 

### Unit test

Because extension classes are really "helper" classes it follows that we'll try to write helpful methods that other developers can reuse. They should usually be small enough to unit test. (If they're not, maybe they shouldn't be static extension methods.) That way we can feel comfortable that other developers can rely on our helpful extension methods. For example:

``` csharp
[TestMethod]
public void IsInRange_ReturnsCorrectResult()
{
    Assert.IsTrue(1.IsInRange(1,100));
    Assert.IsTrue(100.IsInRange(1, 100));
    Assert.IsTrue(5.IsInRange(5,5));
    Assert.IsFalse(0.IsInRange(1, 100));
    Assert.IsFalse(101.IsInRange(1, 100));
}

[TestMethod]
[ExpectedException(typeof (ArgumentException))]
public void IsInRange_ThrowsExceptionForInvalidRange()
{
    1.IsInRange(100, 1);
}
```

## Summary

Extensions make sense when we want reusable code that applies to a particular class but can't or shouldn't be a part of that class. 

#### Unrelated

This has no direct relation to writing extension classes. But if we were going to write an extension that checks for values in a range, why write it specifically for `int`? This does the exact same thing, but it works for `int`, `decimal`, `DateTime`, and anything else that implements `IComparable<T>`. The unit tests above still pass, and for safety I might add a few more checks.

``` csharp
public static bool IsInRange<TComparable>(this TComparable input, TComparable minimum, TComparable maximum) 
    where TComparable : IComparable<TComparable>
{
    if(minimum.CompareTo(maximum) > 0 ) throw new ArgumentException("Minimum cannot be greater than maximum.");
    return input.CompareTo(minimum) > -1 && input.CompareTo(maximum) < 1;
}
```
