---
layout: post
title:  'Unit Testing "Hello World" Examples'
date:   2016-02-19 07:20:58 -0400
category: blog
description: An entry-level example of unit testing in .NET/C#
---

I’m borrowing the example of a palindrome detector to illustrate how unit tests work. (I forget where I saw it used.) Given how simple the class is some of the tests may be overkill. But even when the code looks simple a few unit tests never hurt. This isn’t intended to explain good unit tests vs. bad or how to write code that’s easier to unit test. This is more like a “Hello, world” for unit testing.

## The Palindrome Detector

``` csharp
    public class PalindromeDetector
    {
        public bool IsPalindrome(string inputText)
        {
            if (inputText == null) throw new ArgumentNullException("inputText");
            var inputChars = inputText.ToLower().Where(char.IsLetterOrDigit).ToArray();
            if (inputChars.Length == 0) return false;
            return inputChars.SequenceEqual(inputChars.Reverse());
        }
    }
```

How to add unit tests

To write unit tests we would usually create a new project of type “Unit Test Project” in our solution. That creates a project with the references needed to run unit tests. Then we would reference the project containing the classes we want to test.

A class containing unit tests is marked with the `[TestClass]` attribute. The individual tests are methods marked with the `[TestMethod]` attribute.

## Basic Tests

For our `PalindromeDetector` class we want to verify that it returns true if the input is a palindrome and false if the input is not a palindrome.

The first test is a little more detailed to illustrate the **AAA (Arrange, Act, Assert)** pattern.

*   **Arrange**  
    Prepare what we want to test. In this case that means creating an instance of `PalindromeDetector` and creating an input.
*   **Act**  
    Execute whatever the behavior is that we want to test. In this case that means calling the `IsPalindrome` function and passing in our input.
*   **Assert**  
    State what we expect the outcome to be. In this case we’re asserting that output is true, since the input was a palindrome. We don’t really have to be that explicit. The second test (return false for a non-palindrome) does the same thing in fewer lines. There’s nothing wrong with that.

``` csharp
    [TestClass]
    public class PalindromeDetectorTests
    {
        [TestMethod]
        public void PalindromeDetector_DetectsPalindrome()
        {
            //Arrange
            var detector = new PalindromeDetector();
            var input = "madamimadam";

            //Act
            var output = detector.IsPalindrome(input);

            //Assert
            Assert.IsTrue(output);
        }

        [TestMethod]
        public void PalindromeDetector_DetectsNonPalindrome()
        {
            var detector = new PalindromeDetector();
            Assert.IsFalse(detector.IsPalindrome("not a palindrome"));
        }
    }
```

A test passes if it runs without throwing any exceptions. If the condition specified by the `Assert` statement is not true then it throws an exception, causing the test to fail. For example,

``` csharp
        Assert.IsTrue(output);
```

throws an exception if output = false. Or if we expect a list to have 5 items we could use

``` csharp
        Assert.AreEqual(5, myList.Count);
```

## Testing Other Scenarios

This makes unit testing especially useful. Within reason we want to be sure that regardless of what the input is, the result is something that we expect. For example, do we expect the comparison to be case-insensitive? Is an empty string a palindrome or not? We can write tests for those.

``` csharp
        [TestMethod]
        public void PalindromeDetector_IsCaseInsensitive()
        {
            var detector = new PalindromeDetector();
            Assert.IsTrue(detector.IsPalindrome("MadamImAdam"));
        }

        [TestMethod]
        public void PalindromeDetector_EmptyStringIsNotPalindrome()
        {
            var detector = new PalindromeDetector();
            Assert.IsFalse(detector.IsPalindrome(string.Empty));
        }
```

Does the method throw an exception when it should? We can test for that also.

``` csharp
        [TestMethod]
        [ExpectedException(typeof(ArgumentNullException))]
        public void PalindromeDetector_ThrowsErrorForNullInput()
        {
            var detector = new PalindromeDetector();
            detector.IsPalindrome(null);
            Assert.Fail("Should have thrown an exception.");
        }   
```

If `IsPalindrome` doesn’t throw an exception then execution continues with the next line, `Assert.Fail()`, which just causes the test to fail. If it throws an exception but it isn’t an `ArgumentNullException`, the exception isn’t caught, so the test fails.

These tests may be overkill because `PalindromeDetector` is simple and easy to read. But in practice here’s what they accomplish:

*   We don’t need to make a mental note to test what happens if the input is empty or null, etc. from the UI or within the application. We know that it’s tested, and we can easily repeat the test.
*   Running 10, 20, or 50 tests that execute in a total of 5 seconds is a lot faster than testing those same behaviors from within the application, especially if we’re going to do it over and over.
*   Even when the class is simple, we make mistakes. Ideally we write the test right before or after we write the class and run it right away. If we make a mistake we find it immediately and fix it. Which is faster, finding an error in one method or finding an error that could be in one of several classes? Or worse, finding an error when we’re testing the whole application end-to-end?
*   We might not expect to write code that’s more reliable and write it faster, but that’s exactly what can happen.
*   When code has no unit tests, it’s harder for someone making changes to tell if they’ve broken something. Imagining every possible effect of a change is difficult or impossible. Running a unit test is easy or even automated.
*   The behavior of the class is documented. Someone else could edit the class and not realize that the comparison should be case-insensitive. But if they change that behavior the test will fail. As soon as that happens they’ll realize how the class is expected to behave and fix it.
*   If we try to write unit tests, it forces us to write our code in smaller units that can be tested (“testable code.”) That results in simpler, more maintainable code whether or not we even write the tests.

## How to Run Tests

*   In the Visual Studio menu, open Test -> Windows -> Test Explorer.
*   After you build the project all of your tests will appear in the window.
*   You can select “Run all” or select individual tests, right-click, and “Run selected tests.”
*   Right-click within the class or an individual test and select “Run Tests” to run all the tests in the class or just one test.

You can also automate this. We have unit tests run when we check in and the commit is rejected if unit tests fail.

Select “Debug” instead of “Run” and you can set breakpoints or break on exceptions. That’s useful when a test fails and we need to find out why.

In the Test Explorer window, tests that pass turn green. Tests that fail turn red. Click on the failed test to see the stacktrace. If the test failed because of a failed Assert then you’ll see that. Or if it failed because of an exception in the code being tested you’ll see that.

## Last Thing

Just before posting I realized that my palindrome detector was all wrong. A string can be a palindrome as long as the sequence of letters and digits are the same when reversed. Spaces and punctuation don’t count. So I had to modify the method to account for that.

The existing scenarios I tested for were still valid, so I could run those tests again, and they all passed. (How convenient.) But then I needed to add two more tests to account for strings with spaces and punctuation.

Again, this is perhaps overkill since what the method does is so simple. But when testing the same method with different parameters, adding another test takes literally a few seconds. You just copy and paste the previous method, change the name, and change the parameters. It’s really no work at all.

Here are the remaining unit tests.

``` csharp
    [TestMethod]
    public void PalindromeDetector_DetectsWithNonAlphaCharacters()
    {
        var detector = new PalindromeDetector();
        Assert.IsTrue(detector.IsPalindrome("123Madam, I'm Adam!321"));
    }

    [TestMethod]
    public void PalindromeDetector_TreatsNonAlphanumericAsEmpty()
    {
        var detector = new PalindromeDetector();
        Assert.IsFalse(detector.IsPalindrome("!@# #@!"));
    }
```