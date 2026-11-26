---
layout: post
title:  'Dependency Injection - Resolving Collections of Dependencies'
date:   2016-02-21 07:20:58 -0400
category: blog
---
This is a useful tool for composing classes that use other classes while keeping their responsibilities separate. And it highlights some other helpful design principles that are assisted by dependency injection.

Suppose we have a class that’s going to validate an order for some products. It takes an instance of an `Order` class and returns a collection of `ValidationMessage` objects.

The list of validations it will perform (subject to change) is

*   Valid address
*   The quantity for each line is at least one
*   No items are shipped to states where they are illegal

Right away it’s obvious that there are way too many things going on to put in one class. Between all of these validations there could be a few different services to call and business rules to implement. But whatever class requires those validations shouldn’t actually know or care what they are.

This is bad:

``` csharp
    public class OrderProcessor
    {
        public void ProcessOrder(Order order)
        {
            var addressValidator = new AddressValidator();
            var orderLinesValidator = new OrderLinesValidator();
            var productStateRestrictionsValidator = new ProductStateRestrictionsValidator();
            var validationMessages = new List<ValidationMessage>();
            validationMessages.AddRange(addressValidator.GetValidationMessages(order));
            validationMessages.AddRange(orderLinesValidator.GetValidationMessages(order));
            validationMessages.AddRange(productStateRestrictionsValidator.GetValidationMessages(order));
            if (validationMessages.Any())
            {
                RejectOrder(order, validationMessages);
            }
        }
    }
```

`OrderProcessor` depends on each individual validator. That means that we can’t test `OrderProcessor` without testing all of the individual validators. Those classes might contain still other classes, so it’s impossible to test this class without also testing everything it depends on. (We can break it down into smaller methods, but that doesn’t even address the problem of its dependencies.)

Wouldn’t `OrderProcessor` be much simpler and easier to test if there was only one `IOrderValidator`?

``` csharp
    public class OrderProcessor
    {
        private readonly IOrderValidator _orderValidator;

        public OrderProcessor(IOrderValidator orderValidator)
        {
            _orderValidator = orderValidator;
        }

        public void ProcessOrder(Order order)
        {
            var validationMessages = _orderValidator.GetValidationMessages(order);
            if (validationMessages.Any())
            {
                RejectOrder(order, validationMessages);
            }
        }
    }
```

When we’re testing `OrderProcessor` we don’t really care about the specifics of the validations. The responsibility of this class is just to reject the order if there are validation messages.

So if there are multiple validations, how does this class depend on just one `IOrderValidator`? Like this:

``` csharp

    public class OrderValidator : IOrderValidator
    {
        private readonly IOrderValidator[] _subValidators;

        public OrderValidator(IOrderValidator[] subValidators)
        {
            _subValidators = subValidators;
        }

        public IEnumerable<ValidationMessage> GetValidationMessages(Order order)
        {
            return _subValidators.Select(
                validator => validator.GetValidationMessages(order).ToArray())
                .SelectMany(validationMessage => validationMessage);
        }
    }
```

`OrderValidator` contains an array of `IOrderValidator`. It passes the order to each one and returns all of the results. This class is also very easy to test because it doesn’t care about the implementations of any validations either. So we can write a unit test using a few mocked (fake) implementations of `IOrderValidator`.

But where does the array of `IOrderValidator` in the constructor come from? This is done through our dependency injection container, in this case from Castle Windsor. (It doesn’t have to be Windsor. If our classes depend on Windsor or another specific brand of DI that’s bad.)

``` csharp
    public class DependencyRegistration : IWindsorInstaller
    {
        public void Install(IWindsorContainer container, IConfigurationStore store)
        {
            container.Kernel.Resolver.AddSubResolver(new CollectionResolver(container.Kernel));
            container.Register(
                Component.For<IOrderValidator, OrderValidator>()
                    .Named("CompositeOrderValidator")
                    .IsDefault());

            container.Register(
                Component.For<IOrderValidator,AddressValidator>()
                    .Named("AddressValidator"),
                Component.For<IOrderValidator,OrderLinesValidator>()
                    .Named("OrderLinesValidator"),
                Component.For<IOrderValidator,ProductStateRestrictionsValidator>()
                    .Named("ProductStateRestrictionsValidator")
                );
        }
    }
```

## How Does This Work?

We add the `CollectionResolver` as a SubResolver. That tells Windsor that if we have a dependency on a collection of `IOrderValidator`, it can provide a collection of all of the registered implementations and inject that collection.

If we have a dependency on just one `IOrderValidator`, the container will supply `OrderValidator` (instead of `AddressValidator`, `OrderLinesValidator`, etc.) because we specified `.IsDefault` for that registration. (If we have multiple registrations for the same dependency then it’s vital to specify a default. Otherwise at runtime Windsor will pick one unpredictably. It might work as expected in testing and then fail in production. I learned that the hard way.)

Windsor is smart enough to avoid recursion by including another `OrderValidator` in the collection of `IOrderValidator`. (I wrote a unit test to be sure. Maybe I’ll post something else on that later.)

We don’t always add a name - `.Named(“Whatever”)` – when we register dependencies. That’s necessary if we’re going to have multiple registrations for the same dependency. If we forget then Windsor throws an exception when we try to resolve the dependency. (That’s good – better than working unpredictably.)

## What Does This Accomplish?

### Single Responsibility Classes are Easier to Test

`OrderProcessor` remains unaware of the specifics of any individual validations. We can test it without having to simultaneously test every one of those validations.

The individual validators don’t depend on `OrderProcessor` or each other. Writing unit tests for those individual classes is much easier. And if those validations are complex then we can follow the same pattern in those classes: They depend on other interfaces, and we can write single-responsibility classes for those, and they will be easier to unit test.

### Maintainability

Suppose tomorrow we need to add a new validation. We can do that without modifying `OrderProcessor` or `OrderValidator` at all. We just create a new `IOrderValidator` and register it.

We can also modify any of those individual validators in isolation. As a developer, it’s great to know that you can modify the behavior of the application in such a simple way without worrying about breaking unrelated code. What’s more, the code tells you how to modify it. Isn’t that much better than looking at one giant class and wondering how to make a change without breaking everything?

Any one of these validators can eventually encompass some complex behaviors. They might call services and have lots of their own dependencies, but `OrderProcessor` and other classes are shielded from that. When you’re working on `ProductStateRestrictionsValidator` you don’t have to think about `OrderProcessor` and vice versa. That’s the goal: to be able to change one part of the application without thinking too much about the rest. We’re smart, but we can’t expect ourselves or other developers to handle too much complexity at once. Our design protects us from that.

### Reusability

Reusability is a second-class concern. We hope for it, but it may or may not happen. But here’s how this approach helps:

If the order validation is built into a web page or is a method in another class, then what happens if we create a different way to place orders? If that validation is entangled with the page it lives on or with some other class then it’s harder to use somewhere else. Because it’s decoupled it’s easier to reuse.

The same is true of the address validator. What if we want to validate an address that isn’t part of an order? It’s easier when the validator is a separate class. That class will probably have a dependency on a separate `IPostalCodeValidator`, and that will also be reusable.

The same is true of the `ProductStateRestrictionsValidator`. At some point we’re going to want to validate whether we can sell a product to a certain state even though an order isn’t being placed. But there will probably be dependencies within that class that we can reuse. We don’t want to have to create a fake Order just to see if we can ship a product to a certain state.

## It All Works Together

Here’s the real beauty of it: What makes a class testable is also what makes it reusable and maintainable. (If you can write a unit test for it that proves that it’s not coupled to other classes.) What makes it maintainable is what makes it testable and reusable. That’s one reason why there’s emphasis on testable code. Even if we never write the actual unit tests we get the other benefits. But actually writing the unit tests ensures that our code is testable, and it’s the easiest way to find out immediately whether it works or has bugs.
