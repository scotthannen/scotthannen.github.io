---
layout: post
title:  "String Interpolation Functions vs. string.Format Constants"
date:   2018-12-29 05:20:58 -0400
category: blog
tags: .net, string interpolation, string.format, constants
description: An alternative to string format constants for string interpolation
---

Have you ever done this, perhaps to construct a URL or something similar? I have:

```csharp
const string SomethingFormat = "I need to interpolate {0}, {1}, and {2}";

string GetInterpolatedString(string value1, DateTime value2, int value3)
{
    return string.Format(SomethingFormat, value1, value2, value3);
}
```

If we isolated the format string as a constant, that suggests that 
 - we used it in multiple places and didn't want to duplicate it
 - we isolated it for the sake of readability

How do we use string constants this way when using string interpolation instead of `string.Format`? 

The short answer is that we don't, because we can't. And maybe it wasn't the best approach anyway.

This won't compile because the value isn't a constant:

```csharp
const string SomethingFormat = $"I need to interpolate {value1}, {value2}, and {value3}";
```

This won't compile because, well, it just doesn't make any sense:

```csharp
static string SomethingFormat = $"I need to interpolate {value1}, {value2}, and {value3}";
```

The underlying problem is that what we're attempting to create is not a constant at all, but a function - in this case a function that takes three arguments and returns a `string`.

The "pattern" illustrated at the outset worked, but was seriously flawed. Why? Because we can do this:

```csharp
const string SomethingFormat = "I need to interpolate {0}, {1}, and {2} ... and {3}!";

string GetInterpolatedString(string value0, DateTime value1, int value2)
{
    return string.Format(SomethingFormat, value0, value1, value2);
}
```

...and it compiles but gives a runtime error because our constant expects four arguments but we're supplying three. We thought we were being clever (okay, I did) by storing the `string` as a constant, but its disconnection from the function that uses it makes it fragile. Sure, I could write a unit test, but string interpolation is better because in addition to being easier to read and write, it gives us compiler errors instead of runtime errors when it's wrong. 

It also becomes harder to understand if the constant is used in many places, which means that we won't see the constant when we're
modifying methods that use it and vice versa. 

Again, what we're really trying to accomplish is not a constant, but a function. 

One way to accomplish that is to replace our constant with a function, perhaps like this:

```csharp
static string FormatSomething(string foo, int bar, DateTime date)
{
    return $"I need to interpolate {foo}, {bar}, and {date}";
}
```

That's way better, but there's still a problem. What if this function isn't limited in use to one class or method, and I change it? 
Yes, I'll get a compiler error, and that's good, but the compiler error will arguably be in the *wrong place*. 
This function is a bit like a class that implements an interface. It exists to serve a particular purpose for its consumers. 
If I change it, perhaps by adding or removing arguments, the compiler error will indicate that its consumers are calling it with the wrong arguments. 
But that's not really the case. I've actually broken the function so that it no longer fulfills the requirements of its consumers. 

What I really want is a compiler error on the function, indicating that its signature is wrong, just like the one I get when
a class implements an interface but doesn't have the correct methods or signatures.

How can we specify that a method, not a class, implements an expected signature? With a delegate:

```csharp
delegate string FormatSomethingFunction(string foo, int bar, DateTime date);
```

The implementation would look like this:

```csharp
static FormatSomethingFunction FormatSomething { get; } = (foo, bar, date) => 
    $"I need to interpolate {foo}, {bar}, and {date}";
```

Just as a class implements an interface, the method is explicitly required to conform to the delegate's signature.

As I side note, I recently realized that for years, when I need to create an abstraction to supply a method to a class, 
I almost always created an interface and a class implementation, even though all I needed was a single method. To be honest,
I never even considered using delegates, even though they exist for exactly that purpose - creating abstractions for methods. 
And, to my surprise, I found that many times using a more specific tool [leads to unexpected benefits](http://scotthannen.org/blog/2018/03/18/depending-on-functions-instead-of-interfaces.html).

Why not this?

```csharp
static Func<string, int, DateTime, string> FormatSomething { get; } = (foo, bar, date) => 
    $"I need to interpolate {foo}, {bar}, and {date}";
```

If we just use a method signature instead of a delegate then the function's intended use is not as clear. 
We could have functions for entirely unrelated purposes that have the exact same signatures. 
Using a delegate and naming it (clearly) enables us to communicate not just its signature but its purpose.

The initial problem almost seems too small to be called a problem. But what we've accomplished is:

1. We've leveraged strong typing so that runtime errors are replaced with compiler errors. There's no compile-time relationship between a format string and its arguments. String interpolation improves on that, so we're taking advantage of it.
2. We're declaring the intent of our code as explicitly as possible so that future developers can easily understand what they're about to modify. (I'd say, "modify or debug", but the previous point makes debugging less likely.)

It's only a tiny bit more readable and easier to understand than a `string.Format` constant, so readability isn't the big benefit. 
The more likely savings are the time someone doesn't spend figuring out why the change they made broke something. 