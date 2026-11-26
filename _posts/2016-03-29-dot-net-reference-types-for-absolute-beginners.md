---
layout: post
title:  '.NET Reference Types for Absolute Beginners'
date:   2016-03-29 07:20:58 -0400
category: blog
description: .NET reference types for beginners
---
I was recently trying to answer a question on [stackoverflow.com](http://stackoverflow.com) for someone just attempting to understand the basics of how reference types behave. I looked for a blog post to point them to, but many start off with the heap and stacks and boxing, and they always try to explain value types in the same post. Those are all important things to understand, but I don’t think it’s necessary to throw the kitchen sink at someone trying to grasp a basic concept. Would you like a marble? Here, it’s in this kitchen sink. Catch!

## What Are Reference Types?

For the purposes of this post, reference types are every class except for these:

`bool, byte, char, decimal, double, enum, float, int, long, sbyte, short, struct, uint, ulong, ushort`

Reference types are every other class in the .NET Framework and any class anyone creates. (But not `struct`s.)

## Examples - They’re Better Than Explanations

Here’s a simple class:

``` csharp
    public class Foo
    {
        public int FooValue { get; set;}
    }
```

When we do this:

``` csharp
    var myFoo = new Foo();
```

We are creating a new object, an _instance_ of `Foo`. The variable `myFoo` refers to that object, that `Foo`.

This creates a new variable:

``` csharp
    var myOtherFoo = myFoo;
```

but it doesn’t create a new `Foo`. The new variable refers to the same object, the same `Foo`, as `myFoo`. That’s why if we do this:

``` csharp
    myFoo.FooValue = 5
```

then this is also true:

``` csharp
    myOtherFoo.FooValue == 5
```

Now if we do this:

``` csharp
    myFoo = new Foo();
```

or this:

``` csharp
    myFoo = null
```

what happens to `myOtherFoo`? Nothing at all. It still equals the same `Foo`. All we’ve done is change what object `myFoo` points to. It points to a new `Foo`, or it points to nothing at all (`null`.)

That’s why it’s called a “reference” type. The object exists in memory, and variables and properties _refer_ to it.

To illustrate this another way:

``` csharp
    var fooList = new List<Foo>();
    var myFoo = new Foo();
    fooList.Add(myFoo);
    fooList.Add(myFoo);
    fooList.Add(myFoo);
```

How many items are in `FooList`? Three. How many instances of `Foo` are there? One. The list contains three _references_ to the same object.

The list is also an object. So if we do this:

``` csharp
    var otherFooList = fooList;
    otherFooList.Clear();
```

Then both `fooList` and `otherFooList` both refer to the same list. There’s only one list, and we’ve just cleared it.

## A Few More Concepts

### Variables That Reference Objects Can Be Null

Here are a few things that will cause exceptions.

``` csharp
    var foo1 = new Foo();
    foo1 = null;
    var value = foo1.FooValue; //Exception!

    var fooList = new List<Foo>();
    var foo2 = new Foo();
    fooList.Add(foo1);
    fooList.Add(foo2);
    var value2 = fooList[0].FooValue; //Exception!

    Foo foo3; //It's null because we haven't set a value.
    var value3 = foo3.FooValue; //Exception!    
```

Those three statements will each throw a `NullReferenceException`. Why? Because we can’t check the `.FooValue` property of `null`. A `NullReferenceException` happens when we expect a variable to refer to something when it’s actually `null`.

### This Also Applies to Properties and Values Returned From Functions

``` csharp
    public class Bar
    {
        public Foo BarFoo { get; set;}
    }

    var myBar = new Bar();
    myBar.BarFoo.FooValue = 10; //Exception!

    myBar.BarFoo = new Foo();
    myBar.BarFoo.FooValue = 10; //OK
```

Also, if a function returns a reference type, what it returns can either be an object or it can be `null`.

``` csharp
    class FooProvider
    {
        Foo GetFoo(bool returnNull)
        {
            if(returnNull) return null;
            return new Foo();
        }
    }

    var provider = new FooProvider();
    var foo1 = provider.GetFoo(false);
    var foo2 = provider.GetFoo(true);
    foo1.FooValue = 1; //OK
    foo2.FooValue = 2; //Exception!
```

In other words, _anything_ that can refer to a reference type can potentially be `null`. When we write classes and functions it’s often helpful to be sure that properties and values returned from functions are never `null`, or we know exactly when they might be `null`. That way we don’t have to constantly check to see if values are `null`. How do we check that?

``` csharp
    if(foo1==null){
        //It's null.
    }

    if(foo1!=null){
        //It's not null.
    }
```

### Functions Can’t Change The Values of Our Variables (Unless We Let Them)

``` csharp
    public class FooUpdater
    {
        public void UpdateFoo(Foo fooToUpdate)
        {
            fooToUpdate = new Foo();
            fooToUpdate.FooValue = 0;
        }
    }

    var foo = new Foo();
    foo.FooValue = 3;
    var updater = new FooUpdater;
    updater.UpdateFoo(foo);
```

What is the value of `foo.FooValue`? It’s still `3`. Why? Because when we pass `foo` as a parameter to `UpdateFoo` it actually creates a new variable, `fooToUpdate`, which refers to the same `Foo`. But as soon as we said `fooToUpdate = new Foo();`, `fooToUpdate` no longer referred to the same `Foo`. It referred to a new one. If we changed it to this:

``` csharp
        public void UpdateFoo(Foo fooToUpdate)
        {
            fooToUpdate.FooValue = 0;
        }
```

then `UpdateFoo` isn’t creating a new `Foo`. It’s setting the property on the existing one.

If we add the `ref` keyword to the function then the function can modify the variable that was passed into it.

``` csharp
    public class FooUpdater
    {
        public void UpdateFoo(ref Foo fooToUpdate)
        {
            fooToUpdate = null;
        }
    }

    var foo = new Foo();
    var updater = new FooUpdater;
    updater.UpdateFoo(ref foo); //Now foo is null.
```

That’s not something we do too often. But in this case, because the function uses the `ref` keyword, it can change what `foo` references, in this case setting it to `null`.

### It’s All Different With Value Types

`bool, byte, char, decimal, double, enum, float, int, long, sbyte, short, struct, uint, ulong, ushort`

These are the value types, and they don’t behave the same way. But hopefully leaving them out of this has made this explanation a little easier to absorb.