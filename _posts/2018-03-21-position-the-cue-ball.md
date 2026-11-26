---
layout: post
title:  "Position the Cue Ball"
date:   2018-03-21 07:20:58 -0400
category: blog
description: Shooting pool as an analogy for writing maintainable code
---

I think most people are familiar with billiards, also called "pool." It's the game with balls on a table where you hit one ball - the white one, called the "cue", with a cue stick, causing it to hit other balls and hopefully knock them into one of the pockets at the edges of the table. If a ball is sitting right next to a pocket it might be really easy to knock it in. But if the cue ball is at one end of the table and the ball you want to knock in is way at the other end or nowhere near a pocket then it takes more skill.

As a teenager I used to play billiards. While I became marginally better at knocking the balls into the pockets, I quickly plateaued. Almost every game followed the same pattern:
 
- First was the "break", which is where the balls are arranged in triangle, you hit them with the cue ball, and they roll off in different directions. There are fifteen balls so chances are something will roll into a pocket even though it wasn't aimed.
- Next I would start aiming to sink individual balls. This started off easy too, because there were lots of balls and at least a few were already near pockets.
- At about the halfway point the game would get really hard because I had made the easier shots and the remaining ones were increasingly difficult. The last few shots were usually all but impossible.

One day a man walked into the condo clubhouse where I was playing, saw me struggling, and told me what I needed to do differently if I wanted to improve.

He said that anyone can make an easy shot and sink a ball. But the real skill is the ability to control where the cue ball rolls after it knocks another ball in so that the next shot and the shot after that will be easy too. You don't win just by being able to make really difficult shots, although that helps. You win by not setting yourself up to make difficult shots in the first place.

This was revolutionary (to me.) I was so focused on whether or not each ball went into the pocket that I never thought about where the cue ball would go. I thought that it might end up well-positioned by dumb luck, and when it didn't I tried to compensate by making a difficult shot. I didn't even know that positioning the cue ball was a thing. (I just learned there's even a word for it, *leave*, as in where you leave the ball. That might be the most logical name I've ever heard for anything in any sport.)

### What Does This Have to Do With Software Development?

That sounded hard, though, so I stopped shooting pool and became a software developer instead. Sure enough, a similar pattern emerged:

- When I started writing a new application it was easy to imagine that everything might fall into place. You can't paint yourself into a corner with a single brush stroke, so it's easy to start off optimistic.
- As I continued, there were plenty of easy tasks to keep me moving along. I often started out by designing database tables and stored procedures. If I got stuck on one thing I'd leave it and work on something else.
- Toward the end, the testing process would lead to the discovery of new requirements. As I adjusted my code accordingly, it quickly became tangled and unmaintainable. Before the application was even finished, each change I had to make was like attempting a tricky bank shot, and left the code in an even worse state for the next change that would come along.

Of course, the comparison of software development to billiards is just an analogy and all analogies break down.

- In software development you can only plan so far in advance because you don't start out with all of the balls on the table. Your customer will add, move, and remove balls over time. They will move the pockets. They will take a ball out of a pocket and ask you to sink it in a different pocket. That's not a bad customer. That's just how we iteratively develop software. It's way better than when they used to put all the balls on the table up front, tell you to document each shot all the way through the end of the game, and hold you to it. (And then move things around anyway.) 
- If unpredictability or even bad choices get you into a tough spot, you can rearrange the table a little bit too. That's refactoring. It's not cheating - in fact, we should do it from time to time. 
- The good news: Ideally no one is playing against us, trying to position the cue ball so that our next shot is harder. Our team is working together, everyone is skilled, and everyone wants everyone else to win.

Here's the point: Each shot can potentially set up the next one to be easy, reasonably difficult, or nearly impossible. We apply the analogy to writing code by taking reasonable steps to ensure that the code we write today doesn't make the code we or someone else must write tomorrow unnecessarily difficult. But how can we do that when we don't even know what tomorrow's requirements will be?

### Nothing Is Trivial

We're familiar with design patterns, anti-patterns, SOLID principles, and a few more:

- Write short methods (15 lines of real code or less) with meaningful names
- Use constants and avoid "magic strings"
- Don't repeat yourself
- Prefer composition over inheritance
- Don't do evil things with reflection and runtime type-checking unless you have no choice, and you usually do

Many times we overlook these and other principles when making seemingly small decisions, perhaps individual lines of code, and then we wonder why our codebase ends up gnarlier than we'd hoped. Why do we do this? Here are a few reasons. (I'm not saying they're good reasons. They're just reasons.)

- We're rushed.
- Maintainable code matters, but for some reason not in the class or method we're working on right now. (This comes in many forms, including pragmatism, as in "I understand why unit tests are so important but I don't bother with them because I'm pragmatic," or, "My finely-tuned pragmatism tells me not to create a new class when there's no defined limit on the number of methods I can add to an existing one.")
- We're maintaining such fragile, incomprehensible code that the best we can hope for is to place our Jenga block on top and escape with our lives.
- We ignore minor details when reviewing someone else's code because we hate to nitpick. 

The problem is that it's difficult to associate cause (small decisions) with effect (unmaintainable code.) That's because the effect is the accumulation of many small causes, and it might take weeks, months, or longer before we feel enough pain to realize that our code has gotten to a bad place. We don't like where we end up, but it's hard to look back and see the steps that got us there. 

The remedy is to *never think that any decision is trivial* and do our best within reason to apply whatever we know to every line of code that we write. If we knew exactly which decisions would or would not give us pain later, and why, then we would be able to predict the future. But we can't know that. Our inability to see the future is the very reason why we write maintainable code. The most accurate prediction we can make is general: Today's "trivial" decisions will haunt us or someone else tomorrow.

Even if we apply everything we know we'll still tie some knots we'll wish we hadn't. But it won't happen as much or as quickly. Our applications won't live forever, but they'll be more Clint Eastwood and less Lassie. (Because Lassie died and got replaced over and over. It was either that or callously compare someone who died young to bad software or give up on the Clint Eastwood comparison.)

### Think A Step Ahead

A more specific way that we can keep our cue ball well-positioned is by thinking through possible or even hypothetical future changes to our code. This doesn't mean actually writing code to account for those scenarios. But if our code is maintainable then we should be able to at least imagine how we might modify it. If I think a future requirement is likely, I might go so far as to document or comment what the modifications might look like. 

Depending on abstractions plays a big role in this. If a class has to validate shipping addresses and it depends on an abstraction called `IAddressValidator` that I defined for this class to validate addresses, then even if my current implementation is really simple I've left the door open to provide a more complex implementation or even a facade for multiple implementations. Even if that never happens, I've still made both the address validator and the class that depends on it simpler and more testable by keeping them separate. If the behavior changes more significantly - perhaps my validator might return suggested corrections - then at least I know where that interaction is defined, and I can change the interface if I need to because it was defined for this purpose. It's not some giant all-purpose interface that's used everywhere.

I'm not trying to be prescient, and this certainly doesn't mean that all future changes will be easy or that I won't have to change any existing interfaces or classes. Hopefully it does mean that by separating behaviors into dependencies that I can picture changing, I'll have written something that won't make the next person curse me too much. I'd like them to feel that I played as their partner, not as their opponent.

Billiards is a game for fun, so the decision to either improve or just knock the balls around should depend entirely on what's fun for us. Software development should be fun too, but it's obviously more consequential. We have years of other people's accumulated wisdom at our fingertips, and often our own experience that we forget to listen to. That experience helps us to plan a step ahead, prepare for the unforeseen future, and not end up behind the eight ball.


