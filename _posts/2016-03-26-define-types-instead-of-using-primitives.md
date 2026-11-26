---
layout: post
title:  'Define Types Instead of Using Primitives'
date:   2016-03-26 07:20:58 -0400
category: blog
---
If we’re going to use certain abstractions throughout our code to represent fundamental concepts in our domain, it makes sense to do some extra work when deciding what abstractions to use and creating those abstractions. It’s really easy to quickly decide, for example, that a product ID should be a `string` or a price should be a `decimal`. But once we begin using it we’re going to have to live with it.

_If we create a custom type we have much more flexibility to change details later. If we use a primitive type it’s much harder to change our mind later._

## Examples

### Product ID

#### Product ID as a `string`

``` csharp
        public class OrderLine
        {
            public string ProductId { get; set; }
            public int Quantity { get; set; }
            public decimal Price { get; set; }
        }
```

#### Potential problems

This is not overthinking. This happens:  
Tomorrow we acquire another company or begin doing business in another country, and one string value is no longer enough to uniquely identify a product. Now it’s an ID and a company code or a country code or both.

If we’ve used a `string` throughout our application to represent a product ID, this is going to be a tougher change. Do we update `OrderLine` and _every other class_ that contains `ProductId` like this:

``` csharp
        public class OrderLine
        {
            public string ProductId { get; set; }
            public string CountryCode {get ;set; }
            public int Quantity { get; set; }
            public decimal Price { get; set; }
        }
```

Then in every single place where we compare products we have to remember to compare both properties. If we had a null check for `ProductId` we need a null check for `CountryCode` too.

Hopefully we don’t do something even worse like concatenate the values together and parse the string in other places. Then we’re not only coupled to a `string`, we’re coupled to a `string` that’s expected to contain certain values.

Also, what if there are rules that determine what makes a valid value for `ProductId`. Perhaps it always needs to be eight characters long and alphanumeric. How many places in our code will we have checks to ensure that the value of `ProductId` on various classes is valid? How many `.Trim()` and `.ToUpper()` calls will be all over our application because we’re not sure who or what is going to set those properties?

#### Product ID as a Class

“Product ID” is an important concept in our application and we’re going to use it everywhere, so it deserves it’s own abstraction, it’s own class:

``` csharp
        public class ProductId
        {
            private readonly string _id;
            private readonly string _companyCode;

            public ProductId(string id, string companyCode)
            {
                if (string.IsNullOrWhiteSpace(id) || !char.IsLetter(id[0]))
                    throw new Exception("Bad id!");
                if (string.IsNullOrWhiteSpace(companyCode))
                    throw new Exception("Bad company code!");
                _id = id.Trim().ToUpper();
                _companyCode = companyCode.Trim().ToUpper();
            }

            public override bool Equals(object obj)
            {
                var other = obj as ProductId;
                return other != null
                     && (ReferenceEquals(this, other) || 
                     (string.Equals(_companyCode, other._companyCode) && string.Equals(_id, other._id)));
            }

            protected bool Equals(ProductId other)
            {
                return string.Equals(_id, other._id) && string.Equals(_companyCode, other._companyCode);
            }

            public override int GetHashCode()
            {
                unchecked
                {
                    return ((_id?.GetHashCode() ?? 0) * 397) ^ (_companyCode?.GetHashCode() ?? 0);
                }
            }
        }
```

Does this seem like a little more work? It is, but it’s worth it. (And Resharper did half of it.)

*   We can add other properties to this, modify the equality checks, and a lot of our code that references `ProductId` won’t need to change at all. We’ll still need to change the parts that communicate with databases or other services, but it’s still easier than modifying every single class that references a product.
*   We can check for equality and distinction just like with a `string`.
*   The logic to validate a `ProductId` is in one place. If it changes, we change it in one place.
*   While it’s easy to create an invalid string, it’s impossible to create an invalid `ProductId`.

We’d want to avoid adding anything to this class that isn’t required to uniquely identify a product. Because we’re passing this object around sooner or later someone may find it convenient to add some other detail, like a product description or manufacturer name. But this class exists strictly to identify a product. It can be a property of another class that contains those details, and when needed we can use it to retrieve those other details from other services. But we don’t want to start with what used to be a string and end up passing around a bunch of other values we may or may not need.

### Price

Update - I'm working on a Money implementation which is [available on GitHub](https://github.com/scotthannen/Plasticfloor.Money). `Money` isn't the same thing as `Price`. `Money` is just the amount and currency. A `Price` will include `Money` amounts but will likely have other properties. In time I'll update this post or add a new one. 

#### Price as a `decimal`

A `decimal` is a perfect abstraction of a price, right up until it isn’t.

*   What currency is the price? US dollars, Canadian dollars, or something else. You can’t tell. (Again - add another property everywhere?)
*   A `decimal` can be negative. A price probably shouldn’t be negative.
*   A `decimal` can equal 1.001\. An item costs $1.00, but 1,000 of that item costs $1,001.
*   A price might have other properties that shouldn’t be separated from it. Maybe a price expires after a certain date.
*   A given customer might not be authorized to see a price. How do you represent that? A zero? A nullable `decimal`? How do you distinguish that from not having a price at all?

It’s even clearer in this case that a price is more than just a `decimal`.

#### Price as a Class

``` csharp
        public class Price
        {
            private readonly decimal _amount;
            private readonly string _currencyCode;

            public Price(decimal amount, string currencyCode)
            {
                var rounded = Math.Round(amount, 2);
                if (rounded < .01M) throw new ArgumentOutOfRangeException(nameof(amount), "Amount must be at least .01");
                if (currencyCode == null
                    || currencyCode.Trim().Length != 3
                    || currencyCode.Trim().Any(codeChar => !char.IsLetter(codeChar)))
                    throw new ArgumentException("CurrencyCode must be three alpha characters.", nameof(currencyCode));
                _amount = rounded;
                _currencyCode = currencyCode.Trim().ToUpper();
            }

            public decimal Amount => _amount;
            public string CurrencyCode => _currencyCode;

            public override bool Equals(object obj)
            {
                var other = obj as Price;
                return other != null 
                    && (ReferenceEquals(this, other) || (other._amount == _amount && string.Equals(other._currencyCode, _currencyCode)));
            }

            protected bool Equals(Price other)
            {
                return _amount == other._amount && string.Equals(_currencyCode, other._currencyCode);
            }

            public override int GetHashCode()
            {
                unchecked
                {
                    return (_amount.GetHashCode()*397) ^ (_currencyCode?.GetHashCode() ?? 0);
                }
            }

            public static bool operator ==(Price left, Price right)
            {
                return Equals(left, right);
            }

            public static bool operator !=(Price left, Price right)
            {
                return !Equals(left, right);
            }
        }
```

Whatever additional properties come up that are intrinsically part of a price, we can add them to this one class. We can add properties to represent not having a price or choosing not to a show a price. We can create methods that convert prices between currencies. And we can add this class into compositions of classes that involve multiple prices.

It’s also possible that `CurrencyCode` shouldn’t be a `string` either. We could create a class for that as well that contains an inner `string`. Then we can validate to ensure that currency code values are correct. We might decide later that an enumeration is the way to go, and we can change that within the `CurrencyCode` class.