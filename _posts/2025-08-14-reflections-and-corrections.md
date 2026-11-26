---
layout: post
title:  "Reflections and Corrections"
date:   2025-08-14 -0400
category: blog
description: Thoughts on what I've learned since writing previous blog posts
---
If you're reading this, perhaps you've found something on this blog interesting or helpful. Odds are that you've never read the <a target="_blank" href="https://scotthannen.org/about/">About Me</a> page. And why would you?

This post is about my other blog posts. Like the About Me page, I'm writing it only because to some extent my blog represents me. Over time, however, the individual posts may not. I feel compelled to address this somehow, even though it's unlikely that anyone cares except me.

Some of this reflection is a little bit funny. This blog contains bits and pieces of my learning journey, but it's weird to read myself <a target="_blank" href="https://scotthannen.org/blog/2016/02/19/unit-tests-hello-world.html">describing unit tests</a> like I've just discovered fire. In other cases the information is obsolete. And sometimes I no longer feel the same way about what I wrote or I was just wrong.

With that in mind, here are some notes on a few older posts:

### <a target="_blank" href="https://scotthannen.org/blog/2016/02/23/abstract-factory-pattern.html">Abstract Factory Pattern</a><br/><a target="_blank" href="https://scotthannen.org/blog/2019/01/14/abstract-factory-alternatives-lifetime-management.html">Alternatives To Abstract Factories Part I - Lifetime Management</a>
I used abstract factories a lot in 2016. Since then it's completely fallen off my radar. I didn't use it in any domain modeling sense because I had no idea what that was. 

In the example I created an `IAddressValidatorFactory`. Code that needed an `IAddressValidator` would pass it some argument (like an address). The implementation would decide what sort of `IAddressValidator` to create, create it, and return it.

In retrospect the way I used these factories made little sense. Even if it's abstract, why would a consumer need to know that there's a factory creating implementations of `IAddressValidator`? 

There may be different address validation strategies. For example, the way we validate postal codes or provinces could vary depending on the country. But I'd  encapsulate all of that within an implementation of `IAddressValidator`. Pass the address to it, and internally it decides which validation strategies to implement. Within that encapsulation there could still be a factory that returns a different `IAddressValidator`. But I wouldn't expose any of that complexity to any consumer that just wants to validate an address.

### <a target="_blank" href="https://scotthannen.org/blog/2016/03/13/constructor-injection-webapi-actionfilters.html">Dependency (Constructor) Injection With Web API Action Filters<a/>

Once upon a time we needed this. Now we don't. <a target="_blank" href="https://learn.microsoft.com/en-us/aspnet/core/mvc/controllers/filters?view=aspnetcore-8.0#dependency-injection">There are attributes</a> like `ServiceFilterAttribute` and `TypeFilterAttribute` that simplify injecting dependencies into filters.

### <a target="_blank" href="https://scotthannen.org/blog/2016/03/26/define-types-instead-of-using-primitives.html">Define Types Instead of Using Primitives</a>
This post could be replaced with
> Google "value objects." They're great. Use them.
> 
If we're using C# records then we don't need to implement equality checks.

### <a target="_blank" href="https://scotthannen.org/blog/2016/04/01/event-bus-implementation.html">.NET Event Bus - Dependency Injection Friendly and Agnostic</a>

This was fun to play with back in 2016. I think I used it once. In real life I think I used it once. I was picking up little pieces of domain modelling concepts but never got to use them. Eight years later I rarely get the opportunity to do anything that resembles domain modelling. Unfortunately the pull of anemic, database-centric code is very strong in many .NET environments. 

Given the opportunity to use domain events I don't know whether I'd bother with writing custom code when options like NServiceBus are available.

### <a target="_blank" href="https://scotthannen.org/blog/2017/04/26/dependency-inversion-principle-for-beginners.html">The Dependency Inversion Principle For Beginners</a>

Dependency inversion is a valuable concept. These days it's so baked into .NET applications that I'm not sure if the explanation helps. I don't think about it when writing code. It's just what we do. 
At the same time I didn't understand the concept as well and my explanation was probably all over the place.

### <a href target="_blank" href="https://scotthannen.org/blog/2018/03/18/depending-on-functions-instead-of-interfaces.html">Depending on Functions Instead of Interfaces - Why and How</a><br/><a href target="_blank" href="https://scotthannen.org/blog/2018/04/09/depending-on-functions-navigation.html">Depending on Functions Instead of Interfaces - The Navigation Problem</a>

I still like the idea of it and I use it once in a while, but not as much as I hoped. It's one more slightly different thing introduced into a code base and never catches on. 

### <a target="_blank" href="https://scotthannen.org/blog/2018/12/29/string-interpolation-functions-vs-format-constants.html">String Interpolation Functions vs. string.Format Constants</a>

This is a solution without a problem. 

### <a target="_blank" href="https://scotthannen.org/blog/2019/01/01/interface-segregation-principle-applied.html">The Interface Segregation Principle Applied in C#/.NET</a>

I mostly stand by this. I argued that the principle is about dependencies and coupling, not about avoiding large interfaces. I suppose it's about both, although we generally overlook the coupling issues and focus on the size of the interface.
I'll re-iterate that articles focusing on not throwing `NotImplementedException` completely miss the point. We shouldn't throw that, but that's not what the principle is about.

### <a target="_blank" href="https://scotthannen.org/blog/2020/06/20/mediatr-didnt-run-over-dog.html">No, MediatR Didn't Run Over My Dog</a>
This is the one that haunts me. I was in an odd purist phase in which I ranted about MediatR left and right. Yes, it was misused in places. No, I still don't see how it's connected to the mediator design pattern. And yes, I still cringe when I see "CQRS with MediatR" articles because the two have nothing to do with each other. 

That has nothing to do with the tool. I've used it many times since then. It's familiar. It does something we need. My reasons why we shouldn't use it didn't make sense. I recant. (In this case I felt compelled to add comments to that effect all over the original post.)

### <a href target="_blank" href="https://scotthannen.org/blog/2021/04/07/integration-test-experiment-1.html">An Experiment With Making Integration Tests Easier to Write - Part One</a><br/><a href target="_blank" href="https://scotthannen.org/blog/2021/04/12/integration-test-experiment-2.html">An Experiment With Making Integration Tests Easier to Write - Part Two</a>

It was an experiment. It led me in the right direction but I wouldn't do what I described here. It's overcomplicated. Over time I've settled on something more like what I described in <a target="_blank" href="https://scotthannen.org/blog/2021/11/18/testserver-how-did-i-not-know.html">this post<a/>.

- Cleanly separate dependency injection code so that I can re-use the production code in a test.
- Create a `TestServer` or `IHost` and register dependencies with it using the same code as I use in production, including code that reads from configuration settings.
- If needed, override some dependencies with mocks.
- Either send HTTP requests using the `HttpClient` provided by `TestServer`, or resolve dependencies from the `ServiceProvider` and interact with them.

---
That's it. I'm the only one who cares, but now I feel better.