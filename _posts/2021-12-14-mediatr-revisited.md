---
layout: post
title:  "MediatR Isn't All That Bad"
date:   2041-12-14 01:20:58 -0400
category: blog
description: I said some stuff about MediatR before. I've changed my mind.
---

I've never gotten more feedback from a blog post than from one I wrote about MediatR last year entitled [No, MediatR Didn't Run Over My Dog](/blog/2020/06/20/mediatr-didnt-run-over-dog.html). By that I mean maybe ten people have contacted me. Most of them agreed. MediatR's creator, Jimmy Bogard disagreed on several points.

This blog post is a partial retraction. Over a year later I don't feel the same way about it and I don't want my previous post on the subject to be my last word. So here's a brief summary of how I see this differently.

### It's Not a Service Locator
A [service locator](https://blog.ploeh.dk/2010/02/03/ServiceLocatorisanAnti-Pattern/) lets you obtain (or attempt to obtain) just about any service. An example is injecting `IServiceProvider` into a class instead of injecting individual services. If we inject `IServiceProvider` then our code can call

```csharp
var someService = _serviceProvider.GetService<Anything>();
```
Our class can request any service, which means it can depend on anything and everything. 

When we use MediatR or something similar, we're not able to resolve any service. We're able to create any sort of request (anything that implements `IRequest`) and have it routed to the appropriate `IRequestHandler`. 

It's not as explicit as actually injecting `IRequestHandler<SomeSpecificRequest>`, but it's not as open-ended as a service locator that allows us to resolve any service.

As a side note, it's been pointed out that service locators aren't always bad. I agree, and wrote about it [a few years before](/blog/2018/11/27/stop-worrying-love-service-locator.html).  There's a place for almost anything. I think we can agree, though, that in most cases a pattern other than a service locator is preferable.

### There Are Valid Reasons to Use MediatR in Controllers
The use I see most commonly for MediatR is in HTTP controller classes. The alternative is to inject every handler into the controller. In practice this can yield some odd results.
- We might inject a lot of command handlers into a single controller
- In order to avoid injecting too many dependencies we might arbitrarily break up controllers into smaller classes so that each one has fewer injected dependencies.

But aren't smaller classes with fewer dependencies better? In most contexts I would answer yes. And even in this context I wouldn't find that objectionable.

So why would I make an exception for controllers? Because when used correctly they serve a limited purpose. They don't (or shouldn't) contain application logic. They exist only to convert incoming requests to invocations of higher-level code. 

When I look at it that way it makes less sense to rigidly apply the same "rules" that I might apply in higher-level code. A controller might have many endpoints, but that doesn't mean it's taking on multiple unrelated responsibilities or piling unrelated logic into one class. 

### Where I Wouldn't Use MediatR
Some classes exist only to invoke a single command handler. For example, most message trigger classes I've worked with, like Azure Functions, only listen to one queue. In that case, why inject `IMediator` instead of injecting the required command handler? I don't see a benefit.

Sometimes developers do weird things with MediatR. They see how it's used in controllers and start to think it's the normal way to inject *all* dependencies throughout their code. They might reason that if we use it in one place but don't use it everywhere that's inconsistent and therefore bad. 

We won't have this problem if we understand *why* we use it in certain places. We should strive to always understand why we do whatever we do, wherever we do it. Forced consistency is easier because it requires us to understand less, but it's not a substitute for thoughtful decisions.

### What Remains?
I still have some concerns, but they're not as big as the ones I've reconsidered. They're small enough that where I previously objected to any and all use of MediatR, I won't anymore. I just don't think it's a big deal. 

Here are those concerns and thoughts:

#### The Mediator Pattern
We call it a "mediator" and it uses an interface called `IMedatior`, but it has nothing to do with the mediator pattern. The mediator pattern replaces a complex operation with a simple one. 

For example, when we start a car's engine a number of operations take place, but we likely don't know what they are. We replace it with a simple operation - turning a key or pressing a button. In terms of code that's like calling one method which calls several others so that the consumer doesn't need to know about them.

When we use MediatR we're replacing

```csharp
var result = _startCarHandler.Handle(startCarRequest);
```

with 

```csharp
var result = _mediator.Handle(startCarRequest);
```

As I said, there might be valid reasons to do that, but it's not a mediator.

#### Command/Query Segregation
There are a ton of blog posts about 'implementing CQRS with MediatR'. One by one they describe CQS ([usually not CQRS](https://stackoverflow.com/a/34280756/5101046)) and then how to use MediatR, but they never connect the dots and explain what one has to do with the other. Why? Because the connection doesn't exist. MediatR is neutral in this regard. No query interface can enforce CQS, ensuring that a query handler doesn't invoke command-like behavior.

As I said before, it seems odd to promote CQS by specifically referring to command as "requests" and command handlers as `IRequestHandler`. 

### Conclusion
The point of the previous post wasn't to find fault with MediatR, and neither is this one. This is a correction, because I no longer agree with much of what I previously said. If I hadn't written that post, the minor concerns I still have wouldn't warrant writing this one.

One might ask - if you don't like the names, why not just write your own implementation and call everything whatever you want? I did. It was interesting to write and involves a degree of fun with reflection and open generic types. It's not as robust as MediatR. If I'm on a project and someone would rather just use MediatR instead I'll throw in my $.02 but I won't object.

