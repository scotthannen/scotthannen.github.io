---
layout: post
title:  "Invalid Reasons for Ignoring Code Metrics and Analytics"
date:   2018-04-18 05:20:58 -0400
category: blog
description: Our reasons for ignoring code metrics and static analysis are excuses.
---

Here's an embarrassing but true story: For the first several years that I developed using .NET I wrote code exclusively in VB.NET. I can't tell you how many things I googled twice.

> `httprequest add authentication headers .NET`

...read a few results...

> `vb.net httprequest add authentication headers`

One day I checked out a new app someone was working on and saw that he had written it in C#, with braces and semicolons all over the place. My initial response was reactionary. *We've always used VB.NET! It can do whatever C# can! The change will slow us down! We should all discuss this first!* 

At first I was ready to go to war, but I quickly realized that win or lose, I was on the wrong side. If I somehow won that battle, the prize would be *not* learning C#. How is that going to look on my resume? So I flipped and immediately started struggling to write new code in C#. Within a week I was over the hump, and within two I never wanted to go back. 

This illustrates that when we don't want to do something new or different we can often find reasons not to, even if they are weak or factually inaccurate.

That brings us to code metrics and static analysis. Just like I didn't want to switch from VB.NET to C#, we may have lame reasons why we shouldn't generate or care about metrics that measure aspects of our code, like lines per method, methods per interface, test coverage, or cyclomatic complexity.

Here are a few:

### 1. You shouldn't use code metrics to manage developers
Correct. Code metrics and analysis are developer tools. As developers we use countless tools on our own initiative. These tools are no different.

### 2. I've never heard of them
I hadn't heard of them for a long time either. But now you've read this far and you've heard of them. You can't unknow it. Boom shakalaka!

### 3. The only metric that matters is the value you deliver
Forgive me for being blunt, but this reasoning is based on profound and perhaps willing ignorance of factors that impact the delivery of software. It's like saying that the only automobile "metrics" that matter are miles traveled and resale value, so we shouldn't pay attention to numbers like tire pressure, engine temperature, and miles since the last oil change. Is that sound thinking? If we don't check our tire pressure or keep track of our oil changes, aren't we putting the 'only metrics that matter' at risk?

This relates to the value of our code in two ways. First, if the value that matters is provided by delivering new features with minimal defects, don't we want to know about problems in our code that might cause us to deliver that value at an exponentially slowing velocity and with more defects? Analyzing metrics isn't to detract from the significance of delivered value - it's to ensure that we keep delivering more of it. Achieving a business goal at the cost of increasingly impeding each subsequent achievement is not delivering value.

Second, software must be maintained, likely by someone who comes after us. Isn't the maintainability of our software a large part of its value? I hope so. 

I'm not saying that metrics and analysis are the only way to ensure that we provide value by delivering maintainable code. Rather, the argument that code metrics are somehow at odds with delivering value is ignorant and fallacious. It is the [language of the Expert Beginner](https://daedtech.com/how-stagnation-is-justified-language-of-the-expert-beginner/).

### 4. Code metrics are not predictive
The purpose of metrics and analysis is not to predict defects. They measure factors which indicate the maintainability of code. Large, incohesive classes and long methods with high cyclomatic complexity are inherently harder to read and understand. They are harder to unit test which means that more time is spent on manual end-to-end tests. Modifications take longer, and, in my experience, are more likely to produce new defects.

There's an easy way to test, though. Run the Visual Studio metrics or use a static analyzer. Look at whichever classes or methods it indicates are excessively complex. Do you want to make significant changes to them, or even small ones? If not then the analysis has successfully identifed an area for potential improvement.


### 5. Each component is a snowflake
It's been said that each unit of software is unique, and it follows that no one metric can perfectly measure them all. That's true. There's no one blood pressure measurement that's perfect for every person but we still get it checked. There's obviously no perfect number of lines of code for every method or number of methods for a class, but if our snowflake has twenty public methods and two thousand lines of code it's a hailstone.

### 6. Different tools measure cyclomatic complexity differently
True, but the difference is usually insignificant. The speedometers on our cars are only 95-99% accurate, but if they say we're going 120mph we usually slow down. 

High cyclomatic complexity doesn't automatically mean that a method is a problem. It just gives us a reason to inspect that method and see if it is testable and adequately tested. Maybe there's a giant `switch` statement that isn't worth refactoring and we can decide to ignore it. In each case we're using our judgment. A metric does not compel us to change anthing.

I've only hinted at why metrics and analysis are useful. Discussing potential objections - some understandable, some less so - took up the whole post. To read more on the subject I recommend [this series of posts](https://blog.ndepend.com/category/code-metrics/). 