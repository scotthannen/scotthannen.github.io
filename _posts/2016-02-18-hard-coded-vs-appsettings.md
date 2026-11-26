---
layout: post
title:  "Hard-Coded Settings vs. appSettings"
date:   2036-02-18 07:20:58 -0400
category: blog
---

Often, when including constants in our code, we’re haunted by the possibility that one day we’ll want to change them, and so we place them in `<appSettings>` keys instead.

In some cases that’s absolutely necessary. Take for example, a URL that links to another site in our own domain. That URL is different in across test environments and production, so `<appSettings>` (or something else in app/web.config) makes sense because we can use config transformations to change the values. Or we might know from experience that this particular value is going to change.

What if a page on our site displays the top three of our best-selling products. Should we hard code the number three? What if one day we decide there should be four?

In practice that almost never happens. More often than not a large application ends up with many – perhaps dozens – of `<appSettings>` keys that never, ever change. Maybe our application hasn’t even used them in years, but they remain because we don’t know whether we can remove them, and they seem harmless except for cluttering our app/web.config. (Congratulations if that never happens in your applications.)

And when those values do change, it’s just as likely to occur when we’re updating the whole application, so it still wouldn’t have mattered whether the value was hard-coded. How likely are we to determine that the page should show four best-selling products instead of three, and it has to happen right now, so aren’t we grateful that it’s in web.config? In practice almost never.

## Constants Aren’t Evil

Part of the solution is to decide that constants aren’t evil, and that we shouldn’t fear them or code-shame ourselves for hard coding them.

That’s not to say that we should just throw them around haphazardly. It’s a good practice to explicitly declare constants with meaningful names. That way we don’t end up having to find and change the same values in multiple places and risk a defect if we miss one. But even if we only use a constant once, it makes our code more readable because someone else can quickly tell what that value is used for. If someone sees `NumberOfBestSellersDisplayed` in our code instead of “3” then it’s apparent what that value is used for.

## Compromise

If it seems reasonably likely that a value might change (even though it probably won’t) then perhaps we can live with this compromise:

``` csharp
numberOfBestSellersDisplayed = ConfigurationManager.AppSettings.ValueOrDefault<int>("numberOfBestSellersDisplayed", 3);
```

`ValueOrDefault` is part of a small extensions library I use, but it’s pretty easy to code. The value is hard coded as “3” and there’s nothing in `<appSettings>`. But if we ever need to, we can add an `<appSettings>` value that overrides the constant. That’s probably never, ever going to happen. But if we’re going to stay up at night worrying that our constant isn’t configurable, this is a compromise that keeps us from cluttering our `<appSettings>`.

(I like using the extensions library because every single method is unit tested.)

### Side note

If I’m going to refer to `<appSettings>` then I won’t do that inside the method that uses the value – that makes the method difficult to unit test. I’ll either inject the value by itself, or if there are multiple values define some sort of`ISettings` interface that has the various properties and have an implementation that calls `<appSettings>`.

There are some other ways to avoid `<appSettings>` clutter and confusion but this isn’t about that so I’ll save that for later.

