---
layout: post
title:  "Depending on Functions Instead of Interfaces - The Navigation Problem"
date:   2018-04-09 07:20:58 -0400
category: blog
description: Can we depend on delegates instead of interfaces and preserve our ability to navigate to delegate implemenations?
---

A reader pointed out a significant unaddressed gap in a previous post, [Depending on Functions Instead of Interfaces - Why and How](/blog/2018/03/18/depending-on-functions-instead-of-interfaces.html). 
The code as shown in the post would be more difficult to navigate than code using interfaces. If a class depends on an interface you can navigate to implementations in Visual Studio
(assuming that they're in the same solution) using CTRL-F12 or right-clicking on the interface and selecting "Go to implementation."

But suppose we have this class which depends on a delegate:

```
public class ClassThatDependsOnMath
{
    private readonly DoMath _doMath; // <-- The delegate

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

and the implementation of the delegate is this method:

```
public class Calculator 
{
    public Single AddNumbers(Single value1, Single value2)
    {
        return value1 + value2;
    }
}
```

If we're looking at `ClassThatDependsOnMath`, we see that it depends on `DoMath`, and we want to navigate to implementations of `DoMath`, how do we do that? 
We can get there, but the path is a little bit longer. We'd have to find uses of the delegate which will lead
us to the code where it's registered with the dependency injection container, and read that to see what the registered implementation is. 

This issue is somewhat mitigated by two factors:

- Even though we can navigate to the implementation(s) of an interface, we may still need to visit the dependency injection
  code to determine *which* implementation is used by the class we're looking at. However, many times there are just a few
  implementations and we already know which one we're looking for, so the ability to navigate to the implementation from
  the interface is still a convenience.
- Whether we depend on an interface or a delegate, we can't navigate to its implementation if it's not available in the
  source code of our application. 

So it's not the end of the world, but there's room for improvement. How can we make it better? By making our method that implements the delegate implement it *explicitly*. 
It's not that different from implementing an interface, except that it's the method, not the class.

```
public class Calculator 
{
    public readonly DoMath AddNumbers = (Single value1, Single value2) => value1 + value2;
}
```

Technically we could even omit the types from the function declaration since they're already specified by the delegate. 
I prefer to leave them in so that I can see the types when looking at the method.

```
public class Calculator 
{
    public readonly DoMath AddNumbers = (value1, value2) => value1 + value2;
}
```

Now we can find usages of the delegate (Shift+F12) which will lead us to both its registrations with the DI container *and*
its implementations. Resharper provides "Find Usages Advanced" (Ctrl+Shift+Alt+F12) which includes the option to find delegate targets. 

It's different, and different is a concern because it means developers will see something that doesn't adhere to a familiar convention. 
How big that concern is depends on you and your team. 

If you read the previous post you'll know that I haven't bought stock in depending on delegates vs. interfaces. It simply has benefits in some cases, such as when you want to
prevent the uncontrolled addition of methods to existing interfaces, the corresponding growth of their implementations, and codependent manner in which it enables classes consuming
the interface to add responsibilities without explicitly adding dependencies. Growth of one leads to growth of the other. It's not a problem with interfaces per se, just the enablement
of seemingly small shortcuts that accumulate into large classes with tangled dependencies. I use the words *codependent* and *enable* together because it reminds me of 
that sort of unhealthy relationship. The visible addition of dependencies to a class should help to prevent an accumulation of responsibilities in that class. But too often our interface says, "That's okay, just hide your extra 
responsibilities inside me. It's okay to violate the Single Responsibility Principle. We'll do it together."

A delegate puts its foot in the ground and says, "You can be as careless as you want, but I won't help you hide what you're doing." Unless, that is, we start adding more arguments to the
delegate's signature or adding properties to its inputs or output just because one implementation needs them. That leads to another antipattern, the Inconsistently Populated Object.
More on that later. 