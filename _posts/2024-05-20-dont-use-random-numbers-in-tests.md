---
layout: post
title:  "Don't Use Random Numbers in Tests"
date:   2024-05-20 -0400
category: blog
description: No, using random values does not ensure that our code works with a range of values.
---
Have you ever seen unit tests that generate random input values to ensure that the code works with a wide range of input values? This article explains why that makes no sense and we shouldn't do it.

Here is a simple, contrived example. This function takes an `int` argument and returns `true` if the number is in the range from 5,000 to 10,000.

```csharp
public static bool IsNumberBetween5000And10000(int number)
{
    if (number > 4999 && number <= 10001) return true;
    return false;
}
```
Did you spot the bug? The function will return `true` if the number we're checking is 10,001. But that's okay - I'm going to write unit tests, so I'll find my mistake. (You've also caught me not doing TDD.)

But I'm worried that if I hard-code numbers in the tests then the test will only prove that the code works with those exact numbers. So instead of testing with hard-coded numbers I'll use random numbers. That way I'm not just testing with one number. I'll know that it passes for all the numbers in that range. (Yes, that's exactly the reasoning.)

Here's a test to verify that if the number is greater than 10,000 the method returns `false`. 

```csharp
[TestMethod]
public void RangeCheckerReturnsFalseIfNumberIsGreaterThan10000()
{
    Random random = new Random();
    int input = 10000 + random.Next(10000, int.MaxValue);
    int result = NumberChecker.IsNumberBetween5000And10000(input);
    Assert.IsFalse(result);
}
```
Great, the test passes!  Do you see the problem? That's a trick question. There are three problems.

First, the test shouldn't pass because the code is wrong. I thought I was being clever by testing with every possible number greater than 10,000, so why doesn't my test catch the bug? Because each time the test runs I'm not testing with *all* the numbers, I'm testing with *one* number. The test will catch the bug but only if the randomly-generated input is exactly 10,001. I might have to run it *billions of times* before it fails. Not a great strategy, is it? Hopefully this alone demonstrates that using random numbers to test a "range" makes no sense. The only real way to test a range is to run the test for every number in that range. (No, I am not recommending that.)

Next, my random number generation is wrong. I'm generating a random number that could be as large as `int.MaxValue - 1` and then I'm adding 10,000 to it. In C# that could add up to a number greater than `int.MaxValue`, and the result is a negative number. In this case my mistake means that the test will pass, but it's not testing what it should. In other cases this sort of mistake causes a test to sporadically fail even though the code it tests is correct.

These are the two biggest problems with using random inputs in tests.
- Even if the test can catch a bug, you might have to run it hundreds or thousands of times before that happens. The bug the test is supposed to prevent might happen in production first. What was the point of the test?
- Tests with random inputs often fail not because the code is wrong, but because we didn't get the random value generation right.

What happens in real life when a test full of random inputs fails? The output doesn't tell us what value was used. We don't spot the bug. We run the test again and it passes. Great, never mind. If our test beats the odds and catches a bug, we will probably ignore it. If we actually try to figure out why the test failed that one time we will almost certainly spend more time trying to understand the test than trying to figure out if anything is really wrong with the code.

That brings us to a third problem: If we do this in real-world scenarios (not like my contrived example) we're likely to have tests with lots of random values. It's not just numbers - we might use random numbers to create random dates.  (Lots of things go wrong with random dates.) This results in tests that are very hard to understand. We have to carefully read the range of every random value to understand what the inputs are and then remember them. That's hard. If we don't know what the expected inputs are we can't tell what the test is testing or why it should pass or fail. It's exasperating to look at a failing test, find that it contains a dozen random numbers or dates, perhaps some of them are added to each other, and have to figure out what values they're supposed to contain so we can understand why the test failed.

Please don't do it. It's all risk and waste with no reward. Instead, know what the edge cases are and test them.

In this case:

```csharp
[TestMethod]
public void RangeCheckerReturnsFalseIfNumberIsGreaterThan10000()
{
    var result = NumberChecker.IsNumberBetween5000And10000(10001);
    Assert.IsFalse(result);
}

// or a parameterized test

[DataTestMethod]
[DataRow(4999, false)]
[DataRow(5000, true)]
[DataRow(10000, true)]
[DataRow(10001, false)]
public void RangeCheckerReturnsCorrectResult(int input, bool expected)
{
    var result = NumberChecker.IsNumberBetween5000And10000(input);
    Assert.AreEqual(expected, result);
}
```

Both of these tests are easy to read and catch the bug. 

---
I do sometimes use random values when they aren't relevant to the outcome, like `Guid.NewGuid().ToString()`. Perhaps I shouldn't, but it's arguably a different issue.