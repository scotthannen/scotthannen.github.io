---
layout: post
title:  "The Multidimensional Developer"
date:   2024-01-02 -0400
category: blog
description: There are more ways to grow than learning another language or JavaScript framework.
---
Hopefully we're past talking about the 10x developer. No one knows what it means except that if there's a 10x there must be a 1x. It's about comparisons between developers. I'm not interested in that.

Instead I'll describe five dimensions along which developers can grow. Why five? Because that's how many I think of right now. If I'd written this post ten years ago it would be called *The 2D Developer.* My purpose is not to compare anyone to anyone. (Such a comparison wouldn't be stacked in my favor.) Nor is it to say that everyone should care about all of these things right now or how far anyone should pursue them. And although some things might naturally come before others, these are not sequential. 

## 1. Languages, Frameworks, and Tools
This is the only dimension that is even remotely quantifiable, but it's still vague. It describes the number of things with which we have worked or can work. It's also where we start. We go from working with zero programming languages to one. Within that programming language we use additional tools or different frameworks. Being able to use one or two enables us to create software. The ability to use more than one increases our options.

We shouldn't over- or under-emphasize this. Instead we should put it in perspective. Knowing how to use more things is good. It looks better on our resume. It keeps us from getting bored. It keeps our skills from becoming outdated. We have more ways to solve problems. We prove to ourselves and others that we can learn and adapt.

If we focus all of our energy on this there's a risk that we'll ignore other areas where we can grow, and the returns diminish. We might know how to write code in ten languages and use a dozen JavaScript frameworks but how much does that help us when we're working on a single project? 

When I talk about this dimension I'm referring to the ability to make something work. It might not be pretty, but we can get something to render on a screen, move data in and out of a database, or send a message from A to B using some particular technology. If someone else doesn't know how to make it work and we do, we can help them. We can answer questions on Stack Overflow. It's valuable all by itself.

## 2. Code Quality
At some point we'll likely care about more than getting stuff to work. We'll want to write code that's easier to read, change, and test. It's not the same as knowing how to write code in a given language. We can be fluent in English but have room to improve our communication skills.

There are plenty of divergent opinions on what "code quality" means. What matters is that there is such a thing, it matters, and we can improve. It might include language-agnostic details like using descriptive names. If we work in OOP languages we might learn principles such as those included in the SOLID acronym. (I hesitate to mention SOLID because we often place great emphasis on those five principles and leave out many others. Or we might not agree with all of those principles. But do we know what they are and why we agree or disagree with them?)

In my opinion code quality includes automated testing. There are differing opinions on what sort of tests we should write, how many, and what they are called. (You may have encountered *testsplaining*. "Actually, that's not a unit test. It's an integration test.") But most agree that we should learn to write some sort of automated tests so that we don't depend on manually testing everything to know whether it works, which eventually borders on impossible. We can and should form our own opinions. Many developers swear by Test-Driven-Development. You may agree. but realize that writing tests and TDD are not the same.

As with learning new technologies, we should put code quality in perspective. If we're struggling to learn how to make Angular display a component within a component then we probably have to focus on that and not get hung up on other details. There are subjective opinions about the importance of code quality. I will concede that it isn't everything (only because nothing is everything.) At some point we must decide what's good enough and deliver code. Its subjective quality only matters to the extent that it helps us to deliver valuable software. 

Having described two of these dimensions, it's also easier to see how they complement each other even if we grow more along one and not the other. Consider two developers within a team. One knows more about what makes code maintainable, but they're learning React and they can't figure out why their component is stuck in a loop or how to manage state. The other developer hasn't thought much about code quality but they understand React hooks. What each knows is valuable despite what they don't know. It's pointless to argue over which is more important.

## 3. Business Value and Process
We can make a computer do what we want using various languages and frameworks. We can make it easy for developers to read and maintain. Those qualities matter. But regardless of that, is the software valuable? Does it software do anything that people need or want? To the extent possible, does it work the way they want it to? Does our end-to-end process streamline the creation and delivery of value or hinder them?

At some point we'll likely experience frustration because although we can competently build what someone tells us to build, we find that it doesn't get used. Maybe no one wanted it, or when they got what they wanted they didn't like it. It might get thrown away, or we might spend time re-working it and wonder if there could have been a way to prevent so much waste.

Another dimension along which we can grow is learning how to find out what is valuable and deliver it with less waste.

A number of principles to help us accomplish this are in the Agile Manifesto. Unfortunately this has also become controversial. The word "Agile" has been abused over the years and when people say it they often mean Scrum, backlogs, story points, standups, and more that has nothing to do with Agile. It can help but more often it adds bureaucracy and distracts from Agile.
Others may disagree with the Agile Manifesto itself.

I'll sidestep most of that and cite the first value:
> Our highest priority is to satisfy the customer through early and continuous delivery of valuable software.

Hopefully we can all agree on that. If we don't deliver software then what are we doing? If it's not valuable then why are we doing it? Read the other values and the twelve principles. They represent lifetimes of experience and are in my opinion a solid foundation, but my intent is not to dogmatize them. My point is not that we should grow by adhering to Agile but that we should learn to pay attention to the value of software and reduce waste in the process of delivering it.

This area of growth can be harder because unless you work alone you can't improve alone. For example, producing valuable software with less waste requires increased collaboration with users and business experts. It could be that your organization doesn't work that way, doesn't realize how much better the results would be, and doesn't want to change. They might be stuck in a version of "Agile" which is actually a rigid process that hinders collaboration and innovation.

If that happens you'll have to decide whether to limit your own growth or to move on to another organization or even become self-employed. I'm not recommending any over the others. Sometimes it makes more sense to just get paid. I hope that your organization wants you to grow even if it must grow with you. A developer who collaborates and solve problems is far more valuable than one who simply follows instructions, but your organization must recognize, allow, and encourage it.

Like everything else I'm writing about here, this is not all-or-nothing. We can start small by questioning requirements and suggesting alternatives. One wasteful thing we don't do can save many hours and lots of money while enabling us to do something more valuable instead.

## 4. Architecture
In some respects this is just an extension of code quality. Just as we decide how to make code readable and maintainable when writing classes or methods, we do so when deciding how to arrange and structure code. Good architecture is always deliberate. If we don't decide how to architect our code then we'll make each change according to whatever seems easiest to the person making it at that time. We will not accidentally achieve a maintainable design. We will create a big ball of mud over and over. 

Architecture involves more than just our own code. Eventually we'll work on larger systems that are composed of smaller applications. (Microservics are common. Not necessarily better, but common.) If a mass of spaghetti code can happen with one application, it can be so much worse with numerous such applications talking to each other. It can easily reach an unrecoverable state in which the larger system no longer works and it is impossible for anyone to understand why or fix it.

As applications work together we must decide how they should interact and why. Should they send messages to each other, make HTTP requests, both, or something else? What happens if one application becomes unavailable? Can we stop the entire system from failing? 

Many companies have separate teams to handle architecture across systems. Whether you make decisions, they do, or you collaborate will vary. That may depend on how much you know. It helps if we're in a better position to make decisions instead of depending one someone else.

## 5. User Experience (UX)
This is closely related to business value but it's worth calling out separately. 

UX involves seeing software from the perspective of its users. This overlaps with something I mentioned earlier, which is creating software that users want instead of assuming that they'll use whatever we give them. It also includes making software less confusing and easier to understand. 

Software developers have a mixed reputation in this area. Some say that developers are incapable of making good UX decisions. This has led to the unfortunate proliferation of design-first development processes in which a UX designer decides what is best for users and then hands their design off to developers to implement.

We can prove the naysayers wrong by learning for ourselves what enhances or detracts from UX. This may include more communication with our users so that we understand their perspective more. A lot of UX can be generalized. For example, if we realize that a button's text doesn't clearly describe what it does, it's better to change the text than to expect users to learn what it means.

There are degrees of UX expertise. A skilled UX designer will likely have insights that developers won't. But I've never once seen a case where a designer made all the right choices. We arrive at the best design by collaborating, and learning about UX allows us to contribute meaningfully to that collaboration. Regardless, we should speak up when something in a design seems unclear or counterintuitive and listen to anyone else who does so.

## The Circles of Zorro
I hope no one reads this and thinks that they or someone else must learn about Agile or architecture in order to contribute value to a software development team. That is absolutely not the case. What I'm describing are opportunities for growth. If we're fortunate enough to work within a collaborative team and organization then we can complement each others' skill and knowledge.

If I omitted somethng it's not because I don't think it's important. Communication is important. It just isn't specific to software development. 

This brings me to the Circles of Zorro, which is a made-up term for the way old Zorro (Anthony Hopkins) trains new Zorro (Antonio Banderas) in the film *The Mask of Zorro.*

![Old Zorro teaching new Zorro](/images/circle_of_zorro.png)

Old Zorro instructs new Zorro to stand inside a small circle and tells him that his entire world is within that circle. He cannot think in terms of larger strategies until he learns to fight directly against opponents within that circle. He must start with a narrow focus and then expand his reach.

I like this illustration not because it describes how software developers should learn (it doesn't) but rather how our sphere of decision-making and influence grows. For better or for worse, our early work experience will place us in a small circle. We'll be told to make something work and we won't think about more than how to do that. That's okay. At that point we're under a heavy cognitive load trying to figure out one thing and we're almost certainly not thinking about architecture.

But that does not mean that we must be like Zorro-in-training and master one circle before moving on to larger outer circles. The areas of growth I've described are not circles with in circles. They are different ways to grow. As we get out of our initial survival mode we can explore any or all of them in any order. (This isn't exhaustive. These are the ones that come to mind now categorized according to how I think of them.)

The point is that we should not measure our growth by the number of languages or frameworks we learn, or limit our growth to that. There's more to learn and more ways to grow.


