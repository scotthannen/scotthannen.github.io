---
layout: post
title:  "Don't POOP - The Partial/Optional Object Population Anti-Pattern"
date:   2021-04-19 05:20:58 -0400
category: blog
description: If we don't need all the properties of a class we should create a new class. Here's why.
---

The Partial/Optional Object Population (POOP) anti-pattern occurs when have a class with multiple properties,
we re-use it in various parts of our application, and in different places we populate some properties and ignore others,
leaving some with default values. We do this because it seems easier than creating a new class with only
the properties we need. It appears to save a few seconds but leads to maintainability issues and defects.

Suppose we have these classes containing information about a product:

```csharp
public class Product
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public string Description { get; set; }
    public ProductPrice Price { get; set; } 
}

public class ProductPrice
{
    public decimal Amount { get; set; }
    public string Currency { get; set; }
}
```
(You may see issues with this code that have nothing to do with whether objects are partially populated, 
but that's realistic. Wherever we find one anti-pattern we're likely to find others.)

We start with methods that return a `Product` or a collection:

```csharp
Task<Product> FindProduct(Guid id);
Task<IEnumerable<Product>> FindProducts(SearchCriteria criteria);
```

Each `Product` returned from these methods has an ID, name, description, and a `ProductPrice` object representing
the price of the product. So far, so good.

Then we encounter another scenario: Another part of our application needs the ID, name, and description,
but not the price. Perhaps the price lookup consumes more resources and that method doesn't use it, so why retrieve it?

We create a new method:

```csharp
Task<Product> GetProductDisplayDetails(Guid productId);
```

This method returns a `Product` with a null `Price` property. What's the harm? The method that calls
`GetProductDisplayDetail` doesn't use the `Price` property, and We know that `Price` isn't populated when we get 
a `Product` from this method.

Then we have a requirement to provide promotional discounts for some products, so we modify our 
`ProductPrice` with some new properties:

```csharp
public class ProductPrice
{
    public decimal Amount { get; set; }
    public string Currency { get; set; }
    public decimal DiscountPrice { get; set; }
    public string DiscountCode { get; set; }
}
```

...and we create a new method that returns discounted products for specific customers:

```csharp
Task<Product> GetDiscountedProduct(Guid productId);
```

When we call this method we get a `Product` with a `ProductPrice`, and the `DiscountPrice` and `DiscountCode` 
properties are populated. 

Now we've got three ways to get a `Product`, and which properties are or aren't populated depends on which
method we got it from.

## The Problem

When we start out we know that some `Product` or `ProductPrice` properties are populated depending on which
method returned them. We know that because we wrote the code five minutes ago.

Over time as our code becomes more complex we may pass these `Product` objects to more methods, and as we do
so it becomes harder to keep track of where they came from. Developers working in other parts of the code
may be surprised to find that when they perform an operation on one `Product` it works, but in another
case they get a `NullReferenceException` because they expected a `Price` and there was none. Or there was 
a price and they expected a `DiscountCode` but there was none. Or, worse, they expected the `DiscountPrice` to 
be populated and introduced another defect because sometimes the discounted price is $0.

Chances are that if we "solve" a few problems by adding a few properties to these classes we won't stop. The more
properties the class has the more likely it that some new consumer will have a use for it, and re-using it
will seem more expedient than creating a new one. That consumer may have a need to add its own properties, and the 
problem grows.
We may end up with dozens of properties, combinations of which are populated by some code paths and not others.
I've seen this become so confusing that developers begin to duplicate properties. Clusters of properties appear on
a class and on an object contained within that class. Imagine seeing multiple `DiscountPrice` properties when
reading code and having to figure out which one has a value, or realizing that the answer is "It depends."

POOP also violates the Interface Segregation Principle. The varied, unrelated consumers of the class
become effectively coupled to each other because a change to the class made to meet the need of one consumer potentially impacts
the other consumers. That wouldn't be a concern if they all used different classes instead of using the same one
for unrelated purposes.

This is the sort of problem we learn to work around, but doing so has consequences. Developers lose
the ability to look at code in a smaller context. If a method does something with a `Product`, we can't 
understand that method in isolation. We find ourselves tracing all the code paths leading to that method to
figure out where that `Product` came from. We can figure it out if we're careful, but having to be careful
slows us down. When we carefully add new code we add to the complexity for the next developer. They must be
as careful as we were and understand the new code we've added as well. 

This cycle *may* be sustainable, but it costs us. Changes take longer and longer and the chances of introducing defects increases. 
Instead of modifying the code to add useful functionality we're doing so to fix defects. Fixing them takes
longer and is more likely to introduce even more defects. This churn can go on for months or years

Good thing we saved a few *seconds* by re-using existing classes instead of creating new ones!

## Solution: Invariants

The easiest solution is to define `Product` and `ProductPrice` to enforce that properties which should be populated
are always populated. This is an *invariant* - something that is always true. An example of this would
be an immutable class. Its properties are validated and set when an instance is created and never change.

It might look like this:

```csharp
public class Product
{
    public Product(Guid id, string name, string description, ProductPrice price)
    {
        if (id == Guid.Empty) 
            throw new ArgumentException($"'{nameof(id)}' cannot be an empty Guid.");
        if (string.IsNullOrEmpty(name)) 
            throw new ArgumentException($"'{nameof(name)}' cannot be null or empty.");
        if (string.IsNullOrEmpty(description)) 
            throw new ArgumentException($"'{nameof(description)}' cannot be null or empty.");
        Id = id;
        Name = name;
        Description = description;
        Price = price ?? throw new ArgumentNullException(nameof(price));
    }
    public Guid Id { get; }
    public string Name { get; }
    public string Description { get; }
    public ProductPrice Price { get; }
}
```
The object can't be created without supplying all of its properties, and once it's created those properties
can't be changed (assuming that we also make `ProductPrice` immutable.) The properties aren't optional. I had
to work that extra "O" in there to make the "POOP" acronym.

It's more code and takes longer to write, but that effort may pay for itself many times over. Why?
Because whenever a future developer encounters `Product` anywhere they don't need to consider where it came
from. The class definition itself tells them everything they need to know. What they think they know about
it will never be wrong. The extra minute or two it took to create an immutable type may in the long run
save dozens of hours or more that might have been spent reading code over and over or fixing defects.

If we have lots of properties then applying the builder pattern may be easier than having a constructor
with many arguments. C# 9 also introduces <a href="https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-9#record-types"
target = "_blank">record types</a> which make defining immutable classes easier.

I use immutability as an example of invariance. What ultimately matters is not that an object is immutable
but that it's always in a valid state.

## Solution: Group Sets of Properties Into Classes

One of the reasons why this problem might occur in the first place is that `Product` has dozens of properties.
If we need a class that has many of those properties and a few new ones, that's when it seems easier to
add to existing classes instead of creating new ones. 

We can mitigate this by grouping properties into classes as we did with the `ProductPrice` class. This makes
it easier to create new classes with combinations of the properties we need. Whenever we find ourselves duplicating
sets of related properties across multiple classes we should consider encapsulating them within a single class.

## Solution: Inheritance

I'm just kidding. Inheritance might seem like an easy solution. If we need a class with all 
the properties of `Product` and a few more, we can inherit from `Product` and the new class has more properties.
But unless we're careful it leads us back to the same place. We'll need a class with some of the properties
from `Product`, some of the properties from the new inherited class, and a few new ones. The we end up with
partially populated objects plus the confusion of inheritance. The previous solution is better. 
Prefer composition over inheritance.

## Conclusion

We've all found ourselves working in code that's difficult to understand and safely modify. 
When that happens, can we identify individual decisions that led to that difficulty? If we can identify them
then we can avoid repeating them and even begin to fix them.

Big complexity is usually the accumulation of small decisions, such as

1. We decided to create a class with multiple properties but didn't think making it immutable was worth the effort.
2. We needed a new property some of the time, and adding it to an existing class and populating it only when
needed would take less time than creating a new class.
3. Repeat step 2. We just need one more property.

We make these choices because individually they are small, having no apparent consequences. The effects
come later. This, in my opinion, is one of the greatest challenges of software development. 
There are causes and effects, but we don't connect them because the effects are deferred, and by the time
we encounter them there are lots of causes mixed together. This is why our code gets out of control, we hate it,
and yet given the chance to start over we make the same decisions with the same eventual results.

Once we begin to connect cause and effect we see that few decisions are truly insignificant. Realizing this could
overwhelm us. How do we look into the future and see the outcome of each choice? We can't. 
We *can* guide our decisions by applying principles and avoiding anti-patterns so that we make fewer decisions we 
regret and the rest are easier to change as the need arises.


