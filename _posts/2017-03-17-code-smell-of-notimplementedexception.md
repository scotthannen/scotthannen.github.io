---
layout: post
title:  'The Code Smell of NotImplementedException'
date:   2027-03-17 07:20:58 -0400
category: blog
---
I like the term "code smell" because it acknowledges the subjective nature of the quality of application code. It's like when you open your refrigerator and get a whiff of something that's just _off_. It doesn't mean that everything in there has gone bad, but it's a reliable indication that there's something unhealthy in there.

A `NotImplementedException` in production code is just such a smell. It doesn't mean that the code won't work, but it indicates either a design flaw or a shortcut.

For example, let's say we've created an interface:


``` csharp
    interface IShoppingCartItem;
    {
        ProductId Product { get; set; }
        int Quantity { get; set; }
        Money Price { get; set;}
        IEnumerable<ProductId> GetRecommendedAddOnProducts();
    }
```

Then we create all sorts of classes that implement `IShoppingCartItem`, each with their own functionality for looking up recommended add-on products. Maybe we have some base classes that others inherit from.

Over time we start selling warranties with our products, so we add a new method to our interface:

``` csharp
    interface IShoppingCartItem
    {
        ProductId Product { get; set; }
        int Quantity { get; set; }
        Money Price { get; set;}
        IEnumerable<ProductId> GetRecommendedAddOnProducts();
        void AddSuggestedWarranty();
    }
```

We then go about adding the new method to all of our classes, until we realize we've forgotten one thing - we have a class called `SuggestedWarrantyShoppingCartItem` which implements `IShoppingCartItem`. But you can't add a warranty to a warranty. So we do this:

``` csharp
    public class WarrantyShoppingCartItem : IShoppingCartItem
    {
        public ProductId Product { get; set; }
        public int Quantity { get; set; }
        public Money Price { get; set; }
        public IEnumerable<ProductId> GetRecommendedAddOnProducts()
        {
            return new ProductId[] {};
        }

        public void AddSuggestedWarranty()
        {
            // We're so helpful that we include a message with our exception!
            throw new NotImplementedException("A warranty can't have a warranty.");
        }
    }
```

#### Why Shouldn't We Do This?

Why is this undesirable? (I read that people react negatively to the term "bad code", so I won't ask, "Why is this bad code?")

First, it partially defeats the entire reason for having strongly-typed objects. Strong typing prevents us from trying to call methods or properties that don't exist. 

For example, if `IShoppingCartItem` didn't have the `AddSuggestedWarranty` method and we typed it our IDE (like Visual Studio) we're going to get an error: 'IShoppingCartItem' does not contain a definition for 'AddSuggestedWarranty'. That's good. We want that error to stop us from trying to call a method that doesn't exist. This error can't affect our users because the code won't even compile until we fix it.

But by implementing the method we've lost that huge benefit. Now our application can throw an exception at runtime just because we call a method on an object. 

Can't we still work around this? Sure. We can do this:

``` csharp
    void AddWarrantyToItem(IShoppingCartItem shoppingCartItem)
    {
        if (!(shoppingCartItem is WarrantyShoppingCartItem))
        {
            shoppingCartItem.AddSuggestedWarranty();
        }
    }
```

or this:

``` csharp
    void AddWarrantyToItem(IShoppingCartItem shoppingCartItem)
    {
        try
        {
            shoppingCartItem.AddSuggestedWarranty();
        }
        catch (NotImplementedException ex)
        {
            // ??? Do nothing?
        }
    }
```

But again, we're working against the language. What's the point in knowing that an object implements an interface if we have to inspect the underlying type to see if it _really_ implements the interface? The whole point to implementing an interface is that we don't know or care what the underlying type is. We only know what we need to know to be able to access certain methods and properties. But in the examples above we obviously don't "trust" the interface, so it's not doing us any good.

The [Liskov Substitution Principle](http://www.c-sharpcorner.com/uploadfile/b1df45/liskov-substitution-principle-in-c-sharp/), or the "L" in SOLID says essentially that. If we have an object of type `IShoppingCartItem` or type `Foo` or any other type, that should tell us all that we need to know about the methods and properties of that object. We should be able to _substitute_ one implementation for another. (Most explanations of the Liskov Substitution Principle involve classes that inherit from other classes, but it applies equally to interface implementation.) 

If we ever add another implemention of `IShoppingCartItem` that doesn't implement `AddSuggestedWarranty` then we'll have to look for anything that calls that method and check for that type also so that we don't throw an exception. What if the class is used by another library? Chances are that we're going to forget to fix that library, or maybe we don't even know about it. Maybe it's not even our library and we can't fix it. At best we've caused ourselves or others extra maintenance. At worst we've broken something. 

### What Is the Smell?

Mentioned at the outset, a "code smell" is likely a symptom of other design problems. The `NotImplementedException` thrown in the above code isn't a critical problem if it's accounted for everwhere. But once you see it you're on alert that whenever you work with an interface in that codebase you may want to check all of the classes that implement it to see if any of them throw exceptions.

But that exception really highlights another underlying design problem. Why are `GetRecommendedAddOnProducts` and `AddSuggestedWarranty` even methods of `IShoppingCartItem`? A class should have a single responsibility (the "S" in SOLID.) The responsibility of a shopping cart item is probably just to indicate which product is in the cart and how many of it. Should a class that implements `IShoppingCartItem` also have responsibility for recommending other items or adding them to the cart? Probably not. 

This pattern, which painted a developer into the `NotImplementedException` corner, likely began with thinking that inheritance was the best way to make the functionality of those methods available where needed. They could put a default method in a base class and override it in more specific classes. They didn't think of the Single Responsibility Principle or of creating separate classes to provide those methods. Then someone else came along and added theirs in. And it's a safe bet that you won't find any unit tests for these classes, either.

### It's Constructive

I feel compulsively apologetic for even using the term "code smell." But it serves a purpose. It's easy to say, "Don't throw `NotImplementedException`." But rather than just being negative, I'd rather try to understand what might have led to it. Why would I do differently? This is especially true when revisiting code that I've written. What should _I_ have done differently? Or is it maybe not really so bad in this case? If I can ask those questions and put an answer into words that I'm satisfied with then chances are I've actually learned something. (I call that the "Doogie Howser ending.")
 