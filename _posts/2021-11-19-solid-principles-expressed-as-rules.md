---
layout: post
title:  "SOLID Principles Expressed as Rules"
date:   2021-11-19 01:20:58 -0400
category: blog
description: These are what SOLID principles look like for me when I apply them.
---

Rules are specific applications of principles. We can and must apply principles without rules, but any application of a principle is an action or inaction. Rules make it easier to remember which actions or inactions apply a principle.

For example, "Be safe" is a principle to follow when driving. But what if every time we drove we had to consider every action and decide what was or wasn't safe? That's a lot to think about. Rules help us with that. Drive on the right side of the road. Stay within the speed limit. Stop at red lights. We still have to make decisions, but many decisions are already made because they uphold the principle, "Be safe."

Rules aren't perfect. They can't cover everything. We still need to use judgment. Sometimes they're arbitrary. Why is 70 miles per hour the limit instead of 75? Someone picked it. Do we have to drive on the right side of the road to be safe? No, people in other parts of the world drive on the wrong side of the road and they're okay.

We can make up our own rules. I don't make left turns across busy roads unless there's no traffic. I can if I want to but I don't. 

I'm saying all of this to put rules in perspective. Principles are more valuable than rules, but rules help us to follow principles. 

With that in mind, I've put together some rules that I use to help me follow SOLID principles. They convert valuable but sometimes vague principles into specific behaviors. I do not assert that everyone else should follow them any more than I would tell people in the UK to drive on the right side of the road. They are for me. And I suggest them when I'm trying to help others apply the same principles. 

I don't regard SOLID principles themselves as if they were carved in stone and brought down from the mountain. I've just found that they help me to write better code. Others found that out first which is why they described them and made a nifty acronym out of them.

So here are my rules. Maybe you'll find them helpful. Feedback is welcome because I like to know if this was helpful, but also because you might have an perspective I haven't considered.

### Single Responsibility

- The name of a class should be specific enough to both indicate what it does and exclude what it does not do. If a class is named `OrderService` there is no limit to what we might add to it. If a class is named `PostalCodeValidator` then its responsibility is obvious and anything we add to it beyond that responsibility will stick out.
- A class should have no more than *n* injected dependencies. My magic number is five. New classes rarely have more than three, but if I'm about to exceed I refactor.

### Open/Closed
If we have a class that does *X* but we also need *Y*, don't modify the existing class so that it does both. Just make a new class. 

If we have a method that does *X* but we also need *Y*, don't add optional parameters or figure out how to make one method do different things. We don't get extra points for making one thing do two things. Make another method.

To be honest I have the hardest time with this principle. I almost never consciously apply it. Maybe I should. But principles exist to benefit us, not the other way around. It doesn't need me to apply it. So I'll leave this as a question mark. Maybe I'll understand better one day, or I won't.

### Liskov Substitution
- Do not throw `NotImplementedException`. The exception is if we're forced to implement an interface we didn't create. It should be rare. It might never happen.
- If a method has a variable or argument of type *X*, it should not need to know about anything else about the type of that object. It should not need to use reflection to see if the object is of some other type so that it knows which methods it can call or what to do with it.

This second rule is riddled with exceptions. We may deal with framework types or libraries that give us no other choice. Sometimes infrastructure code uses reflection to inspect types and create generic types. 

If we must have code that inspects the types of objects, we should ask, is it deliberate, or is it a one-off because of poor interface or class design? If it's the latter then we should either try to fix it or encapsulate it inside some class that "hides" that weird, one-off difference from the rest of our code.

### Interface Segregation
This one is confusing because .NET has an `interface` keyword. The Interface Segregation applies to those interfaces, but it's not limited to them. The "interface" is any contract that a type exposes. For simplicity I'll refer to `interface` interfaces, but we can apply it to classes as well.

- Define interfaces and abstractions from the viewpoint of the class that  depends on them. In other words, if you realize that a class which submits orders needs to validate orders, create an `IOrderValidator` interface with only the methods the consumer needs. Don't find or create a general-purpose `IOrderAllTheStuff` interface, add another method there, and depend on that.
- Just like with the Single Responsibility Principle, give interfaces names that describe what they do and exclude the rest. Single-responsibility interfaces tend to have single-responsibility implementations.
- If the functionality we need to depend on is in a giant class or interface, consider defining the smaller interface you need and adapting the existing type to your narrower interface. In other words, the implementation of your small interface "wraps" or hides the larger type.
- Avoid mutually exclusive use of interface members. That is, one class uses some members and another class uses other members. In that case why are those members in the same interface instead of two or more different interfaces?

There are exceptions. `IRepository` classes with all sorts of create, update, get, and delete methods are common. It's familiar to developers across codebases, so I question how helpful it is to insist on interface segregation. I like defining a separate `IReadRepository` interface with read methods and then having the write methods in an `IRepository` interface which inherits from `IReadRepository`. That way if a class should only do queries it can depend on the smaller interface, even if the implementation is all one class.

I'm not rigid with interface segregation. Maybe one consumer depends on all the members of an interface but another depends on only one. That's okay. What matters is that we aim for that separation instead of throwing everything into a few big interfaces willy-nilly.

### Dependency Inversion
Create a "high-level" project which contains important application logic. Don't put any implementation details in that project - no API clients, no database code, no messaging, and no controllers. Define interfaces to represent whatever that high-level code needs to do that might involve those implementation details.

(I work mostly in .NET which is why I refer to "projects." It's a way that we organize parts of our code that stay together because they're related but stay separate so that they aren't coupled or we can decide *how* they are coupled.)

Those interfaces should describe uses or behaviors. They should not have names like `IFooApiClient` or `IFooSqlRepository` that describe implementations. And those interfaces should be in the same project as the code that depends on them.

The implementation details are in in other projects. Those implementation details are "low-level" code. They depend on (reference) the high-level project and implement its interfaces.

Whatever models or entities that high-level code uses should also be defined within the high-level project. If it's convenient low-level code can re-use them. But high-level code should never know about contracts or models used by low-level code.

For example, our high-level code might have an `Order` class. If we get order data from an API in low-level code, whatever model the API returns should be mapped to the high-level `Order` class within the low-level code.

Code that allows users or external actors to interact with our high-level code is also an implementation detail. It is low-level code. A Web API project or Azure Function can re-use models from our high-level code, but models from low-level code should not appear in our high-level code.

We should not create "shared" projects with models used by low-level and high-level code. Once that happens the sharing can go two ways when it should only go one way. 

An exception is when the whole application is low-level code. Maybe it's infrastructure and only exists to interact with databases or messaging. In that case the difference between high-level and low-level doesn't exist.

But when in doubt it's easy to start with separate projects. We can always combine them later.

### Conclusion
If all of this sounds a bit opinionated it's because interjecting "maybe" and "sometimes" is a poor writing practice that I give into a lot anyway. 

These are my rules (in the sense that I follow them, not that they all originate with me.) I chose them, which means I'm also free to revise or break them as I see fit.  They serve me by acting as shortcuts that help me apply what I've already learned. Good things happen when I follow them. 

I am not asserting that my rules should be your rules. Use them, some or all, or something like them, if it helps.



