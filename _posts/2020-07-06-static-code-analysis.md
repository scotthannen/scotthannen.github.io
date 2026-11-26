---
layout: post
title:  "Pragmatism, Perfectionism, and Static Code Analysis"
date:   2020-07-06 -0400
category: blog
description: No one enjoys nitpicking. Automate it.
---


There are two extremes when it comes to the cleanliness and maintainability of code:

1. I'm pragmatic. The most important thing is that it works. The other details don't matter.
2. We should endlessly chase unattainable perfection.

## "Pragmatism"
Pragmatism is good, right? It means to value what is practical over what is theoretical. Unfortunately 
that word is often used to mean, "I do what makes sense to me and don't waste time learning to improve."
It amounts to taking whatever we don't want to learn or do and labeling it "theoretical" so we 
can ignore it.

That's harsh, but I'll go a step further. I'm going to guess that most developers who call themselves
"pragmatic" couldn't define the word without looking it up. Be wary of anyone who justifies 
making messes for others to clean up by using a word they don't understand.

## Perfection
The other end of the spectrum recognizes that technical debt and defects don't always result from large
decisions made badly. They're caused by an accumulation of tiny choices that gradually make code harder
to understand, making changes difficult and defects more likely. Those developers scour over their own
code and that of others looking for variables with the wrong scope and interface segregation violations.

While I'd rather lean toward the second group than the first, that pursuit of perfection is still
problematic. Why?

Reviewing code to comment on every minute detail is time-consuming and introduces a long feedback cycle. 
It's also subjective. One developer says a class is large and confusing. But how large is too large? Confusing
  to whom? Why? 

It's also a miserable experience for everyone involved. Even if I agree in principle with adhering to 
exacting standards, I wouldn't want to be on either side of that code review. I don't want someone pointing 
out 20 minor issues to me, and then five more that I introduce by fixing those 20. 

The only thing worse is being the reviewer who finds and highlights those issues. I don't know about
you, but I don't have it in me to point out every single potential flaw I see, no matter how small.
Not even if I believe they matter.
I can't sustain that for any period of time. Between the effort and my empathy for a fellow human developer
I'm going to relent, ignoring the small things and focusing on the big ones. And those small things
will accumulate. We may have empathy but our code doesn't.

That brings us back to pragmatism. Seeking code utopia by carefully reviewing to enforce standards may sound
like a good idea in theory, but it's not practical. It's going to take too long, we'll start to hate ourselves
and each other, and we'll end up cutting corners anyway.

## A Pragmatic Solution - Static Code Analysis
How do we make consistent adherence to standards less theoretical and more practical, and therefore pragmatic?

What we need is feedback similar to what we get from the compiler when we make a type, omit a semicolon, 
don't close our braces, etc. The compiler is ultra nitpicky. Very little gets past it. Such constant
negative feedback coming from a person would be unbearable, but coming from the compiler we don't take it
personally, because it isn't. Feedback comes quickly, not in a wave that hits us a day later. It helps us.
We rely on it and value it.

*Static code analysis* provides that feedback. After the code has compiled, a code analyzer applies another
set of rules to identify potential issues within our code, much like a human code reviewer might but with less pain.

- An analyzer can evaluate in a minute what would take a reviewer hours to read and document. That means less
expenditure of time and a rapid feedback cycle.
- Its evaluation is not subjective. Teams can tweak the rules as they see fit, but an analyzer applies those rules consistently.
- It reduces the unpleasant human experience of correcting and being corrected.

Static code analysis doesn't (or shouldn't) eliminate code review. It just does a lot of the heavy lifting, pointing
out the small things so that we can focus on the architecture and behavior of code. And as it does so it finds issues 
we might not recognize otherwise. They are not robot overlords. If we understand what they are telling us and disagree. we
can and should tweak the rules or mute them.

## NDepend

At this point I'll start referring to a particular product, [NDepend](https://www.ndepend.com/). Full disclosure: I was given
a license to work with and evaluate, although not with any requirement that I advertise it or promote it. I'm not paid to
write this. I'm writing (a year later) because I'm convinced of the value of products like it and this one in particular.

### What Issues Does It Identify?
There are [many](https://www.ndepend.com/docs/code-metrics). I'll highlight a few. 
They're significant because any one of them can make code difficult to maintain
and increase the risk of defects. They're also not easy for a person to spot when reviewing code:

#### Long Methods
Limiting the lines of code per method is like setting a speed limit. Is 70mph really the perfect limit, and is 71mph too fast?
There's no perfect one-size-fits-all number. But the idea isn't that any number is perfect. Rather, if we set the limit at 70mph
then we won't drive 120mph.

The same goes for lines of code. My arbitary limit is fifteen per method. It's not perfect, but if I stick to it and only rarely
violate it, it's impossible to write a method with 100 lines. NDepend warns that 20 is difficult to understand and maintain
while 40 or more makes a method extremely complex. You can change those numbers. The tool doesn't carve the rules in stone -
it helps to enforce what you and your team decide so you don't have to count lines of code.

#### Classes Lacking Cohesion
Massive "god classes" create all sorts of pain and are difficult to break up. What if we could spot the first step toward
creating such a class and prevent it before it even begins to grow?

NDepend and other methods look for cohesion in classes. Suppose a class has a method that depends on certain private fields,
and then we add another method that depends on different fields. Should that be one class? Why not put the one method with its
fields in one class and the other method with its fields in a different one? The more we add, the harder it becomes to see, 
and then it snowballs. Static analysis can catch that when it starts.

#### Cyclomatic Complexity
While not a perfect measurement, it helps to identify classes and methods in which too many decisions are made. If a method
contains so many decisions that there are 20, 30, or more paths within its execution then it may be impossible for a developer
to read it and comprehend what it's going to do in all of those scenarios or to write tests for them. 
Consider what that potentially means: We could write or modify code that we literally don't understand. 

I've heard the objection that calculations of cyclomatic complexity vary. It's true. Compare it to thermometer that might
be off by a degree. And while we're at it, 98.6&deg;F isn't everyone's perfect temperature. But if the thermometer says your
temperature is 107&deg;F (and you're still conscious) you don't worry about that one degree of accuracy.

That's how I view all code metrics. They're not magic numbers that tell us whether or not our code is perfect. They're 
guidelines that tell us when we're at or over the limit of what's okay.

Wouldn't we want a code reviewer to say why the issues they point out matter? NDepend (and other analyzers) explain the signifance
of the rules. NDepend links to articles for further reading so that we can understand and evaluate for ourselves the principles that
underlie its suggestions. 

## Conclusion
Static code analysis helps us to strike the balance between fake pragmatism and unattainable perfection. It doesn't ensure 
that we follow all of our standards and adhere to all of our principles, but it guides us in that direction. It's harder to 
violate the Single Responsibility principle if we're not creating giant, incohesive classes with a dozen public methods. It's
harder to violate the Interface Segregation Principle if we limit the number of methods per interface. And then the chaos and
messiness that we prevent will make it easier for human reviewers to see what remains to improve.

Does this sound like a sales pitch? I suppose it is. I want to work in code developed by people who use NDepend or something
like it because it's going to be easier to understand and modify. I hate the feeling when I've introduced a defect and I
realize that in part it's because the code was difficult to read, change, and test, but it was my change and I own it. It's 
going to happen. I want less of it.

What does it cost? Depending on the number of developers, $300-$400, more for a build server. That's the cost of a few hours
of our time. That's the cost of a single in-depth code review and a fraction of the cost of fixing a single defect (not counting)
the cost of the defect itself). It's the cost of an extra few hours spent trying to understand code before we
change it. 

Reason on it this way: It's paid software to do what a person does except faster and more reliably, which in turn saves
money and pays for itself. That's not a sales pitch for NDepend - it's a sales pitch for software. It's the underlying reason
why computers and software exist. It's beneficial for the same reason why whatever software we're paid to write is beneficial.

It's genuine  pragmatism, and I believe that it's an edge for developers and teams who use it.

