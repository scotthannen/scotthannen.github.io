---
layout: post
title:  "Simplify .NET Development With Fewer Projects and Solutions"
date:   2023-12-26 -0400
category: blog
description: Splitting everything into more projects and solutions adds complexity and overhead
---

*I'm trying to write shorter blog posts so that perhaps I'll produce more.*

Two ways in which we unnecessarily complicate .NET applications are by
- creating more projects than we need within a solution 
- creating more solutions than we need within an application

### Too Many Projects
Projects are a unit of modularization. In other words, they are a way to keep some parts of our code separate from other parts. They aren't the only way to modularize. We also do this by putting code in different folders, which usually means putting it in different namespaces.

#### Why Separate Code Into Projects at All?
If we can modularize using namespaces, what is the benefit of projects? A big one is that projects allow us more control over how classes are referenced by and visible to other classes.

If we're employing something like Clean Architecture or Hexagonal Ports-and-Adapters architecture (or we're just trying to stay well-organized) then we might want to keep higher-level logic like business rules separate from implementation code like database repositories.

An example is that our logic might refer to an `ISomethingRepository` interface. That's all we want it to know about. We don't want details about SQL or some other database leaking into our logic. We can enforce this by placing our logic in one project which contains the interface. Then another project contains implementation details. It references the project containing the logic and it implements the interface. Two projects cannot reference each other, so this relationship enforces our wish that the logic will not reference implementation details.

A second, more obvious reason to have separate projects is that they contain applications which can or must be deployed separately. For example you might have a Web API project and a message listener that processes dequeued messages. These are separate components and we wouldn't normally try to put them both in one project.

#### When Not to Separate Projects
Sometimes we have SQL code, ServiceBus code, HTTP client code, and some other implementation code, and for some reason we feel that because they are different they belong in separate projects. Why? I don't know. Unless we're aware of a reason why they must be separate, it probably doesn't do anything for us except increase the number of projects which makes the solution harder to navigate.

If we put every different type of code in a different project, we're likely to also create separate unit test projects for each of them, doubling the number of projects.

Finally, we'll inevitably discover something that all of those projects need to share, so we'll create yet another project to put that code in. At this point our dependencies will be unnecessarily confusing. I prefer to avoid the overused "new developers will be confused" argument, but in this case it might actually apply. It can be hard to look at all of these projects and keep track of what references what and why. 

### Too Many Solutions
Excess solutions can be even more problematic than excess projects. An example is when we have a single domain or bounded context. That might require a Web API, a message listener, a scheduled service, and perhaps other related components. Our reflex may be to put each of them in a separate solution. Perhaps we reason that if we put them in one, the solution will have too many projects.

(When I refer to multiple solutions, I don't mean those that exist within the same directory and perhaps share projects, but rather having multiple directories each containing one solution and projects which belong to that solution.)

There's obviously no rule that says we can't put components in separate solutions that way. There may be valid reasons. But we shouldn't do this by default because it creates complications that can increase the complexity of development.

#### We Lose the Benefits of Project References
This is a concern if we're trying to keep our application logic separate from our implementation details. If we have a web API and a message listener service in two solutions, where is the higher-level application logic? 

There are a few ways this can work. And they do all work, but they're all sub-optimal.

One possibility is that we put some application logic in the Web API solution and some in the message listener solution depending on where we need it right now. We might put it in a separate project within that solution so we can pretend that we're keeping our logic and implementation details separate, That's self-deception. Wherever we put it that logic, that's what it's coupled to. What happens when there's some operation performed in the web API but now we want to do it in response to a message? We're either going to have to duplicate that logic or we're going to have to find some other way to share it between solutions.

That need to share code between solutions is almost inevitable. It frequently leads us to place shared code in yet another solution and build a NuGet package which is consumed by the others. This slows development because we must first change code in one place, build a package, and then reference it in another place. But there are other projects that don't need the latest version of our NuGet package. It's easy to introduce a breaking change and not notice until later when we update those other projects to a newer version.

These problems go away when projects are located in the same solution. As we make changes to shared code we know right away if we've introduced breaking changes. Either the solution won't build or unit tests fail. The feedback is immediate. And we don't spend time building a package from one solution and updating references in another.

#### A Gift That Keeps on Giving

The problems created by sharing code between solutions tend to self-perpetuate. 
- We might create a package for sharing code between solutions, but it's for a specific purpose. When we need to share more code that does something else, we create more packages.
- We might not move shared logic into shared code until we need to share it. This leads us to having application logic scattered in various places which makes it harder to understand.
This often results from a misunderstanding of DRY and a lack of architecture. We may have learned (correctly) that we shouldn't over-aggressively remove code duplication. Only de-duplicate when similar or identical code exists in perhaps three or more places.
But when it comes to business logic we don't want to scatter it about and then gather it again as needed to avoid duplication. Our architecture should keep business logic in one place right from the start. This avoids the costly and risky work of pulling it out of an application (to which it has likely become coupled) and moving it into some shared repository.

#### Code Duplication
If we don't have lots of packages then we're likely to have lots of duplicate code. Our different application components might use the same database (which is okay if they're part of the same domain or context) and end up duplicating code to interact with it. Or perhaps they communicate with the same external Web APIs and we duplicate HTTP client code.

#### Everything Goes Through the Web API
This is quite possibly the most common solution, but also one of the worst. Our application likely has not only business logic, but code to ensure that when data is modified, a message is sent out on a message bus, or some other behavior occurs. To ensure that this always takes place, we put all of that behavior in a Web API application, and other components in the same domain interact with domain entities and data by sending HTTP requests to that web API.

This adds a level of unnecessary complication. If these projects were in the same solution they could all reference the same project containing business logic, command and query handlers (often called the "application layer.") Instead each operation becomes distributed. One component handles part of an operation and then makes an HTTP request to a web API to complete another part. 

This makes understanding the application at runtime more difficult. An operation might begin within one component but fail because of an exception thrown in another component, and now we must correlate the two to understand the problem.

Unless we've been careful with our architecture there's a good chance that the operation consists of business logic that exists at both ends of that HTTP request. That also makes it harder to understand.

It also makes testing more difficult. We can write a test for the message listener that receives a message, performs some logic, and sends off an HTTP request. We can write another test for the part that's handled by the web API. But it's much more difficult to write a test for the entire end-to-end operation.

This is inevitable if our application makes a necessary HTTP request to a web API which is part of another application. It is needless complication if our application could invoke code via a project reference but instead makes an HTTP request to another part of itself.

Ironically, while we split code into solutions to avoid having solutions with more projects, we'll likely find that when we keep it together we'll have fewer total projects and less code because of reduced duplication and infrastructure.

### Conclusion
The complexity we add by unnecessary splitting code into projects and solutions doesn't stop us in our tracks. Otherwise we wouldn't do it. Rather it slows us down, compounding the effects of other unrelated decisions that slow us down. No number of straws break the camel's back. Each added straw just makes the camel's job harder. 

We can also compare excess splitting to when we were children and we spread the food we didn't want to eat around on the plate to make it look like there was less of it. But there wasn't really less of it. In this case by spreading it around we might even make more of it, and someone has to eat it.

I'm not proposing a rule. Rules rarely work. But if our rule has been that different types of code go in different projects and different components go in different solutions, we don't have to keep following that rule either. It probably hasn't helped us.