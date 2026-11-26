---
layout: post
title:  "Consistency - Will New Developers Really Be Confused?"
date:   2020-10-14-0400
category: blog
description: Consistency is a principle, not a magic word.
---
Have we heard this argument when discussing disagreements involving code?

> We need to make new code consistent with existing code or new developers will be confused.

The purpose of this blog post is not to dispute the consistency argument. Just the opposite - first I'll argue for it. Rather my intent is to drain
some power from it, to put some speed bumps in front of it. When the consistency angel lands on one shoulder and tells us that this code must be the same as that
code, let this other voice ask, "The same how? Why? Have we thought this through?" It's not even a devil; It's another
angel with a different perspective, and often agrees with the other one.

First let's look at why consistency is beneficial when applied thoughtfully. This will set up a contrast. 
Consistency is good when it provides these benefits. It's questionable if it does not provide these 
benefits and harmful if it works against them.

### Good Consistency

#### Consistency Reduces Cognitive Load

Looking at unfamiliar code is inherently confusing. We're simultaneously trying to grasp what code does, why, in what context, and often whether
it works as expected. Needless inconsistency adds to that cognitive load. If we take that initial confusion and add to it variables that are prefixed (or not) 
inconsistently or the indentation seems random, it becomes harder for our minds to process the more important details.

We avoid that by establishing conventions. In C# some code bases prefix field names with an underscore. Others don't. Which is better? It matters less
than whether we pick one and apply it everywhere.

#### Consistency Helps Someone Who Understands One Part of Our Code to Understand Others

Suppose we choose a certain library to fulfill a need: Let's say we choose Autofac as our IoC container. If we use it consistently throughout our 
code base then once a developer understands how it works they'll understand it more easily throughout the code. Sure, they can understand more than one, but
why not make it easier for them? We wouldn't introduce a different one without compelling reason.

If we see a design pattern employed across multiple similar, related classes then once we understand it in one place we'll understand more quickly what's
going on in another. We should not, however, assume that developers are so limited that similarity between one area and another is the only way they can
understand how code works. 

#### Consistency Narrows Our Choices and the Remaining Ones Are Good

What if a design pattern is employed across similar, related classes and now we need to create a new one? We could approach it as if the other classes
don't exist, and we'd have lots of choices in front of us. The decision might take longer and we'd make it with less certainty. If we aim for consistency
we can often eliminate most choices and follow a pattern found in existing code.

If we need to choose a library to perform some purpose, that decision is easier if we're already using a library for that. In that case we just keep using
the same one. 

In both cases consistency helps us to make choices more quickly by removing choices or strongly recommending others. Fewer choices might sound bad,
but especially when starting out in unfamiliar code it's reassuring to know that we're doing something the expected way. Decision paralysis can waste
time. The code that results from following a pattern will likely be good enough. (*likely*, not *certainly.*)

Let's contrast these benefits with happens when the drive toward consistency goes wrong.

### Harmful Consistency

#### Will New Developers Really Be Confused?

We've probably worked in code bases where something weird is done over and over, or something that clearly violates the principles we follow to write the
best code we can. I don't want to use a real example, so I'm going to make one up. It's no less strange than the real examples I'm not using.

Imagine we have an area of our application in which each class contains an identical set of unused private fields and private methods. It's obvious that
each new class is created by copying and pasting an existing one. The majority of the code in each class is not used. The methods that
are used have weird names like `Invert` and `Enrich`, and you realize that those names have no connection with what those methods do.

Figuring out what's going on in these classes wastes some of your time. So when you create your own, you leave out the parts that aren't used,
and you help out the next person by giving the methods meaningful, descriptive names.

But then comes the objection: "Consistency is very important. All of the other classes have these methods and variables, and the names are all the same. If you
make this one different... (here it comes!) ...  *new developers will be confused.*"

Quite often when I hear that argument I *am* the most recent developer to join the team. No one ever asks me what confuses me. They tell me 
what confuses me, and they're usually wrong.

Yes, it is possible to confuse new developers. It's certainly possible to confuse me. But I've never looked at code and thought, 
*Hmm. They did this odd thing for no apparent reason in ten other places but didn't do it here. Therefore this code is confusing. 
If they did the same odd thing here also I would not be confused.* Is that plausible?

If a developer found that confusing, how would they cope if we modified one class to accommodate new requirements? 
Would we add it to all the others just for that one person? Would it actually help them? How? 

Unused code is just an example, but the same reasoning applies to other scenarios, whether it's naming, applying
design patterns, or something else. We'll be consistent when we can because it does make code easier to understand.
But any developer understands that two classes or modules that exist for different reasons will differ from
one another. 

Don't just say, "New developers will be confused." Reason on it. Sometimes we'll find that it makes no sense.

#### Sometimes There Is No Pattern

Sometimes we see an coincidental pattern in the code and then expect new code to follow it. For example, we might have several classes that perform similar
functions. At some point the requirements for one class change, one of its private methods becomes more complex, and we extract it into a separate service. 
We didn't do the same with the other classes, so now in that respect this one is different from the others.

We could reason that we shouldn't refactor the class because then it won't be like the others. But that doesn't hold up under scrutiny. The two
classes were *already* different. They had different names, and their methods did different things. Otherwise why are they separate classes? Don't
enforce imaginary consistency.

One or two of anything is not a pattern. Don't be that person who writes twenty lines of code or a few classes
and then tells everyone else to write theirs the same way because it's "the pattern." Please don't.

#### Don't Confuse Patterns With Design Patterns

Design patterns are beneficial when used correctly. Sometimes developers justify their peculiar consistency by calling it a "pattern,"
perhaps hoping that the goodness of properly applied design patterns will rub off on whatever they're doing
(and want others to do.) It won't.

This confusion is compounded when the pattern consists of misapplying a design pattern, often using it
for no apparent reason. (Think singletons everywhere or "adapters" that adapt interfaces to deliberately
identical interfaces.) Now we've got double pattern! Or maybe even pattern squared!
Pattern is a neutral word. If something is a pattern or we call it one, that doesn't make it good or bad.
Knitters follow patterns. So do serial killers.

![Doge says Wow! Such pattern! many design!](/images/doge-likes-consistency.png)

#### Consistency Is Coupling

Coupling occurs when changes to one part of our code affect other parts of our code, often forcing us to change one part because another part changed.
A good deal of what we learn as developers is how to avoid coupling. Why? Because it makes code harder to modify. Small changes ripple across coupled code
and become larger changes. As the size of the change increases so does the risk of introducing defects.

Now think about what happens when we have multiple classes and determine that they should have characteristics in common for reasons that have nothing
to do with the purpose or behavior of any of them. We've artificially coupled them together. We may have carefully designed each to minimize coupling,
but now we've done the opposite. The effect could be minor and harmless, or we might have to choose between 
writing "bad" code to satisfy that coupling or modifying unrelated code so that our new code is easier to write.

We may weigh the cost of that coupling and decide that it's worth it for the sake of consistency. It often is. 
It's less harmful than other forms of coupling because it exists only until we decide it doesn't. 
But the cost is real. Consistency is coupling. 
I've said it twice in the hope that when someone hears the one word in this context they'll think
the other.

### How to Detect Good Consistency and Avoid Harmful Consistency

How do we test whether consistency makes sense in a particular case? Here's are a few suggestions:

Whatever we think should be consistent, *try to express it in words.* For example,

- Do not prefix field names with underscores.
- We use ABC library to do XYZ. Let's keep using it to do that unless we need something it can't handle instead of using two or more libraries.
- All classes in this namespace which implement `ISomeInterface` should have private methods named `Invert` and `Enrich`.

This encourages us to reason on 
- whether or not a pattern exists
- exactly what that pattern is
- whether it's intentional or coincidental
- whether it makes sense to perpetuate it

If needed, discuss other factors:
- Is the alternative to consistency based on concerns about an existing pattern or is it matter of personal preference? If there's not a good reason to deviate
then we likely shouldn't.
- Has the pattern created technical debt, and does repeating it add more technical debt? 
- Would an alternative be easier to write, understand, test, and maintain? Why? How?

The benefits of consistency might outweigh other factors. Sometimes it makes sense to repeat something 
sub-optimal to avoid inconsistency. But the point is that we're *deciding* to be consistent or inconsistent.
We should not be subservient to our past decisions or those of others. 

If we choose consistency then perhaps we can document it along with our reasoning. (After all, we've expressed it in words.) 
Perhaps place a ReadMe file in the source code or note it in a Wiki. Even then it should remain open
to change. The point is that if we've reasoned on it and made a decision, we don't have to have the same
discussion often. And we'll always remember why we're doing what we're doing.

Expressing the reasons for our decisions is powerful. We can document and discuss it, accept it or 
challenge it. Sometimes we reason about consistency, but sometimes we use it as magic word that ends
reason and discussion. Let us view consistency as a principle on which to reason and not as a mere invocation.

## Conclusion

If I've convinced anyone to be thoughtful about how and why they prioritize code consistency then my work here is done. Consistency is beneficial as 
a principle, but harmful when we use it as a reason to disregard other principles.

Agile methodology and common sense tell us that we should look at what we've done in the past, do more of what works, and stop doing what doesn't. Apply
that to consistency. Where does it help? Ask your new team members what helps them and what slows them down. If something helps, keep doing it. 
If what hinders them or you is something done consistently, *stop doing it.*


