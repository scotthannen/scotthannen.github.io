---
layout: post
title:  "Dependency Injection In a Console Application"
date:   2036-06-29 07:21:58 -0400
category: blog
---

James Shore [wrote](http://www.jamesshore.com/Blog/Dependency-Injection-Demystified.html) that 

> "Dependency Injection" is a 25-dollar term for a 5-cent concept.

Dependency injection using frameworks like [Castle Windsor](http://www.castleproject.org/projects/windsor/),[Autofac](https://autofac.org/), [Simple Injector](https://simpleinjector.org/index.html), or others is immensely useful and not inherently difficult. The challenge is that when someone says, "You can solve problem *xyz* by using a dependency injection container," they're potentially asking you to learn a few new concepts and at least one new tool *simultaneously.* Anything is more challenging to learn when you're trying to learn a few things at once.

In particular it requires understanding

* What is dependency injection?
* How does dependency injection encourage me to write classes and interfaces differently?
* How do I configure and use a dependency injection container?

I'm not going to try to cover all of that right here. I recommend James Shore's post linked above, [Dependency Injection Demystified](http://www.jamesshore.com/Blog/Dependency-Injection-Demystified.html). In my experience sometimes the idea becomes simpler if you just see it in action, so in this post I'd like to demonstrate using dependency injection in the simplest context possible (in .NET world), a console application.

(Why would anyone use DI in a console app? Isn't that the service locator anti-pattern? See the end of this post.)



#### Why use dependency injection in a console application? Isn't that the service locator anti-pattern?

At first I thought it was rather silly to introduce a DI container to a console app. It feels a little bit like trying to use DI just for the sake of using DI, not because it's the right tool for the job. But now I see it differently:

* If you're going to learn to use a dependency injection container, a console app is actually the easiest place to do it. That's because unlike with a WCF service or MVC application, you don't have the extra overhead of configuring the application to use the container. You can just create it and start using it.
* It's not a bad way to write a console app. It encourages writing all of the functionality in other classes rather than putting everything in the `Main` method of the `Program` class. Now those classes are potentially reusable and testable. I'm not saying it's *the way* to write a console app, just that it's not bad, and may be an improvement if the app isn't really small.

Using a container in a console app means explicitly resolving at least one dependency from the container. Is that a misuse? No. In this case the `Main` method becomes our composition root. It's the entry point where we configure our dependencies, resolve an instance of the class that's going to do the actual work, and then execute one or more methods allowing that class to do its work. If that class referenced the container then that would be service location. But it doesn't, so it's not.