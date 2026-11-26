---
layout: post
title:  'Abstract Factory Pattern'
date:   2016-02-23 07:20:58 -0400
category: blog
description: abstract factory .net 
---
What is a scenario in which we might use an abstract factory, and how can we create one?

Let's start with a class that’s going to validate orders. The class is `OrderValidator`, and it depends on an `AddressValidator` to validate the orders’ shipping addresses.

I’m starting off correctly without making `OrderValidator` directly dependent on `AddressValidator`. Instead `OrderValidator` depends on an interface, `IAddressValidator`. That keeps the classes decoupled and makes them easier to test.

``` csharp
    public class OrderValidator
    {
        private readonly IAddressValidator _addressValidator;

        public OrderValidator(IAddressValidator addressValidator)
        {
            _addressValidator = addressValidator;
        }
        public IEnumerable<ValidationError> ValidateAddress(Order order)
        {
            return _addressValidator.Validate(order.ShippingAddress);
        }
    }
```

But suppose I ship to multiple countries, and the address validations are so different that I need different classes to do validations for different countries?

I could create a factory class that provides the needed validator depending on the ISO 3166 country code. But even with just a few countries it’s easy to see how this can get out of hand:

``` csharp
    public class AddressValidatorFactory
    {
        public IAddressValidator GetAddressValidator(Address address)
        {
            IAddressValidator validator;
            switch (address.CountryCode)
            {
                case "USA":
                    {
                        validator = new UnitedStatesAddressValidator();
                        break;
                    }
                case "FIN":
                    {
                        validator = new FinlandAddressValidator();
                        break;
                    }
                case "MWI":
                    {
                        validator = new MalawiAddressValidator();
                        break;
                    }
                default:
                    {
                        validator = new CommonCountryAddressValidator();
                        break;
                    }
            }
            return validator;
        }
    }
```

This could be problem. And what happens if `MalawiAddressValidator` has dependencies of its own? Is my factory class going to contain

``` csharp
        validator = new MalawiAddressValidator(new MalawiPostalCodeValidator());
```

That gets messy. Now my `AddressValidatorFactory` will end up with a long switch statement, and it will have direct dependencies on every individual validator and the dependencies of each of those validators. What I really want is to use dependency injection (constructor injection) in each one of these classes. That means that instead of calling `new FinlandAddressValidator()` and `new MalawiAddressValidator()` I have to allow my dependency injection container (like [Castle Windsor](https://github.com/castleproject/Windsor/blob/master/docs/README.md)) to create these classes. How do I do that?

The pieces might not make sense by themselves, but they will when we put them together.

First we need an interface representing a factory class to create instances of `IAddressValidator`. (It’s an abstract factory because it’s not a concrete class - it’s an abstraction of the factory.)

``` csharp
    public interface IAddressValidatorFactory
    {
        IAddressValidator GetAddressValidator(Address address);
    }
```

Next, since I’m using Castle Windsor, I’m going to declare a class that will get the name of the specific `IAddressValidator` to use based on the country code.

``` csharp
    public class AddressValidatorSelector : DefaultTypedFactoryComponentSelector
    {
        public AddressValidatorSelector() 
            : base(fallbackToResolveByTypeIfNameNotFound: true) { }

        protected override string GetComponentName(MethodInfo method, object[] arguments)
        {
            return "AddressValidatorFor_" + ((Address)arguments[0]).CountryCode;
        }
    }
```

Why is this class converting an `Address` into a string, like “AddressValidatorFor_MWI”? Because now we can register different implementations of `IAddressValidator` with the Windsor container by name.

Here’s what the component registration will look like:

``` csharp
    public class WindsorInstaller : IWindsorInstaller
    {
        public void Install(IWindsorContainer container, IConfigurationStore store)
        {
            container.AddFacility<TypedFactoryFacility>();
            container.Register(
                Component.For<IAddressValidator,UnitedStatesAddressValidator>()
                    .Named("AddressValidatorFor_USA"),
                Component.For<IAddressValidator, FinlandAddressValidator>()
                    .Named("AddressValidatorFor_FIN"),
                Component.For<IAddressValidator, MalawiAddressValidator>()
                    .Named("AddressValidatorFor_MWI"),
                Component.For<IAddressValidator, CommonCountryAddressValidator>()
                    .Named("FallbackCountryAddressValidator")
                    .IsDefault()            
                );
            container.Register(
                Component.For<IAddressValidatorFactory>()
                    .AsFactory(new AddressValidatorSelector())
                );
        }
    }
```

*   First we add the `TypedFactoryFacility`. That enables Windsor to work with abstract factories.
*   Then we register the individual validators by name.
*   We tell Windsor that when there’s a dependency on `IAddressValidatorFactory`, it should provide a factory that uses our `AddressValidatorSelector`. We don’t create a class to implement the factory. Windsor creates that. What we provide is the selector that tells the factory how to take the argument (an `Address`) and determine the name of the validator to create.

What happens if the country code is “HRV” and we don’t have an address validator registered with the name “AddressValidatorFor_HRV?” Going back to the `AddressValidatorSelector`, we passed in parameter to the base class constructor, `fallbackToResolveByTypeIfNameNotFound: true`. That tells the selector that if it can’t find anything matching the name to return the default implementation, `CommonCountryAddressValidator`. So if we register a validator for a country, we get that validator. If we haven’t registered one then we get a backup.

(What if multiple countries can use one validator? We don’t have to create a class for every single country. We could register the same class for multiple countries or perhaps put that logic in `AddressValidatorSelector`.)

## Testing the Abstract Factory

How could we possibly know if all this works? We don’t want to write a big chunk of the application, run it, and only then find out if this blows up. So we write a few unit tests. We want to test that the factory returns the `IAddressValidator` we expect for different addresses.

``` csharp
    [TestClass]
    public class AddressValidatorAbstractFactoryTests
    {
        [TestMethod]
        public void AddressValidatorAbstractFactory_ReturnsCorrectValidatorForCountry()
        {
            var container = new WindsorContainer();
            container.Install(new WindsorInstaller());
            var factory = container.Resolve<IAddressValidatorFactory>();
            var address = new Address { CountryCode = "FIN" };
            var validator = factory.CreateAddressValidator(address);
            Assert.IsInstanceOfType(validator, typeof(FinlandAddressValidator));
            factory.Release(validator);
        }

        [TestMethod]
        public void AddressValidatorAbstractFactory_ReturnsDefaultValidatorForUnknownCountry()
        {
            var container = new WindsorContainer();
            container.Install(new WindsorInstaller());
            var factory = container.Resolve<IAddressValidatorFactory>();
            var address = new Address { CountryCode = "XYZ" };
            var validator = factory.CreateAddressValidator(address);
            Assert.IsInstanceOfType(validator, typeof(CommonCountryAddressValidator));
            factory.Release(validator);
        }
    } 
```

The unit tests

*   Create a container
*   Install the dependencies using the `WindsorInstaller` we created
*   Use the container to resolve an `IAddressValidatorFactory`
*   Create an `Address`
*   Tell the factory to create an `IAddressValidator` for the `Address`
*   Verify that the `IAddressValidator` is of the expected type.

I expected at least a little bit of debugging, but both tests passed on the first try. (If I did need to fix an error, it would be much easier while writing the unit tests than later.)

The result?

*   We have an abstract factory that knows how to provide the correct address validator depending on the address’s country.
*   We keep all of that logic out of the classes that require address validation. They don’t need to know about that. Now those classes remain simpler and easier to test.
*   We can modify the address validators and how we select them without changing surrounding classes.
*   We can write individual address validator classes that are also easier to test in isolation.

## Last Thoughts

Hypothetically, if we were to support one hundred countries and between them they used a few dozen validators (seems unlikely) then we’re still going to need some code or other mechanism that matches the countries with the validators. That’s okay because we can now keep it far away from our other application code. We could put it in its own separate Windsor installer class. The purpose of that class is to match dependencies with implementations so all that code belongs there.

Here’s what `OrderValidator` looks like now. We’ve added some complexity behind the scenes but this class is unaware of it.

``` csharp
    public class OrderValidator
    {
        private readonly IAddressValidatorFactory _addressValidatorFactory;

        public OrderValidator(IAddressValidatorFactory addressValidatorFactory)
        {
            _addressValidatorFactory = addressValidatorFactory;
        }
        public IEnumerable<ValidationError> ValidateAddress(Order order)
        {
            var addressValidator = _addressValidatorFactory.CreateAddressValidator(order.ShippingAddress);
            return addressValidator.Validate(order.ShippingAddress);
        }
    }
```

The logic around how to choose a validator could change. Maybe we don’t just select validators by country. Maybe addresses get validated by more than one concrete class. We can change all of that without changing `OrderValidator`.

Here’s some [official documentation](https://github.com/castleproject/Windsor/blob/master/docs/typed-factory-facility-interface-based.md) on Castle Windsor’s Typed Factory Facility.