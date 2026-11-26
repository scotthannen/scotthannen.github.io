---
layout: post
title:  "Untestable Code Knows No Mercy"
date:   2019-01-22 -0400
category: blog
description: Your code doesn't care why it's untestable.
---

What is untestable code?
 - It has no unit tests.
 - Writing unit tests for it would be difficult or impossible without modifying the code.
 - Common attributes include global state, tight coupling, and long methods that do lots of things. 

Testable code, on the other hand, tends to consist of smaller classes and methods with well-defined purposes. It avoids use of global state and uses a reasonable amount of abstraction to avoid tight coupling between components.

Neither is meant to be a comprehensive definition. They're just enough so that I can throw around the terms *testable code* and *untestable code* for the rest of this post. 

Untestable code is a cause that produces known effects. Those effects include:
- More defects
- Modifications are harder and riskier
- End users wait longer for new features and bug fixes

Conversely, testable code results in the reduction of all three.

I hope that this is exciting, good news for at least a few people. The problems I've just summarized aren't inevitable. They are effects that come from causes, and we can control those causes. 

Why the emphasis on cause and effect? Because sometimes we understand them but somehow think that they won't apply in certain cases. We know what results from untestable code but act as if we don't.

I've addressed one such case [in this blog post](http://scotthannen.org/blog/2018/04/03/tame-your-dependency-injection-registration.html). Sometimes we use IoC containers to make our code more testable, and then we write hundreds or more lines of untestable code to configure the containers. Eventually we find it difficult to understand the code that composes the rest of our application. How does this happen? Because somehow we imagine that our dependency registration code is the exception to all the rules we normally set for ourselves, so we put it in a few massive methods and we don't write tests for it. Then it gives us trouble because that's what large methods without tests do. 

SQL stored procedures are another problem area for untestable code. Have you ever observed that teams can be careful and meticulous in C# but then go crazy in SQL? It doesn't help that the language inherently favors optimization over all other concerns. We think in terms of what we want to do with individual records but then we have to code it using set-based operations. We express behaviors in inner and outer joins and make critical decisions with inline functions that appear in unexpected places. This makes it much harder to read complex code and understand what it does and why.

We can't avoid global state in SQL because tables *are* global state. It's hard to avoid tight coupling when we're forced to repeat lists of columns so that adding a column or changing a data type in one table can lead to cascading changes in any statement that touches it. 

There are [testing frameworks](https://tsqlt.org/) that address some of these concerns, and it's possible to break large procedures into smaller ones that pass data using table variables. I've yet to see a team use either approach. I'm convinced that testable SQL is possible. We just don't think about it that way. 

I'm highlighting SQL stored procedures as an example because of the rampant abuse, but I'm thinking of any case in which we feel that for some exceptional reason we must or should write large amounts of complex, untestable, untested code. JavaScript is another. I wrote a ton of it before learning how to write unit tests for it, and I've seen massive amounts of application-critical JavaScript without a single test. (Doesn't a language without type safety need *more* tests, not fewer?)

Given that cause, there is only one reason to imagine that we will escape the harmful effects: Perhaps our code will show us mercy. 

We don't say it or think of it that way, but isn't that what we're counting on? Perhaps our 600 line stored procedure will exhibit compassion, realizing that we had no choice but to write it that way, and kindly exempt us from the consequences. The thoughtful inner monologue of our code could go something like this: 

> It is not your fault. You could not have made me smaller, more readable, or given me unit tests, because I am SQL. It would be unfair for me to give you all the same grief and pain that comparable code would give you under any other circumstances. Therefore I shall show you mercy. I shall have no defects. I am incomprehensible, but I shall be the one piece of software that  no one will ever need to understand or modify. 

It sounds irrational, but doesn't it sum up exactly what we're hoping will happen? Aren't we just pretending that cause and effect will give us a pass?

The consequences for writing monolithic "units" of untestable code are exactly the same whether it's SQL, C#, JavaScript, or any other language. To believe otherwise is like raising a grizzly bear cub and thinking that it won't maul us to death just like a wild one. The bear doesn't know whether it's wild or not. It just acts according to its nature. Our code doesn't know why it's hundreds or thousands of lines long, untested and untestable, and it doesn't care. It will grieve us and vex us six ways from Sunday. Everything we fear most will happen, and then some. 

We can either find a way to make it manageable or make an informed choice to accept the risks, but we can't expect our code to show us mercy. 
