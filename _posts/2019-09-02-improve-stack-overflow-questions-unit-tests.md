---
layout: post
title:  "Improve Your Stack Overflow Questions With Unit Tests"
date:   2019-09-02 -0400
category: blog
description: Use unit tests to help others to help you.
---

Suppose you're trying to write code to solve this problem:

> I've got a start date and end date. I want to count the number of times between
those dates where a given day (like 10) appears.
  
You try writing it, get stuck, and decide to try asking for help on [Stack Overflow](https://stackoverflow.com/). 
You post the problem above and include the code from your Winforms app, which looks something like this:

```csharp
public partial class Form1 : Form
{
    public Form1()
    {
        InitializeComponent();
    }

    private void btnCalculate_Click(object sender, EventArgs e)
    {
        try
        {
            var startDate = DateTime.Parse(txtStartDate.Text);
            var endDate = DateTime.Parse(txtEndDate.Text);
            var day = int.Parse(txtDay.Text);

            var numberOfTimesDayHappensInDateRange = 0;

             // Some code that tries to calculate this, but isn't getting
             // the results you want

            MessageBox.Show($"The answer is {numberOfTimesDayHappensInDateRange}!");
        }
        catch
        {
            MessageBox.Show("I think something is wrong with the values you entered!");
        }
    }
}
```

Some variation of the following will likely happen: 
- You'll get downvoted
- Your question will get closed
- You'll get no response
- You'll get answers that don't relate to your question
- Someone will leave a comment saying, "I have no idea what you're trying to do. Can you give an example showing what results you expect?"

We're sad when that happens. How can we improve our question?
One approach is to write a unit test that shows our inputs and the results
we expect. 

*Disclaimer: I asked some other folks on Stack Overflow what they thought of this.
The response amounted to, "Sure, but no one is going to do that." And they might be right.
But that's okay. I'm an optimist and I like to think this will help someone, even
if it has nothing to do with asking questions on Stack Overflow.*

What if we could make this easier to understand by isolating the code that contains the problem from everything else? 

We could start by putting just this part of the code in a separate class and method:

```csharp
public static class DateFunctions
{
    public static int NumberOfTimesDayOfMonthOccursInDateRange(
        DateTime startDate, DateTime endDate, int dayOfMonth)
    {
        // Some code that tries to calculate this, but isn't getting
        // the results you want
    }
}
```

*In case it's not obvious yet, this post doesn't include a working version of this function. This is about how to ask the question,
not the solution itself.*

That's a long function name, isn't it? But it's also a lot clearer what it's supposed to return. If we just called it `NumberOfTimesDayOccursInDateRange`
it might be less obvious that we're talking about a calendar day (1, 15, 27) and not a day of the week (Monday, Tuesday.)

All we've done so far is move the code from one place to another. That's a start. Now someone who wants to understand the method
and help us with it doesn't have to look at all of the extra code. What they're looking at is just the part we're trying to figure out.

It might seem obvious to us what results we expect to get for different inputs, but how can we communicate that clearly to someone else?

One way to accomplish that is with one or more unit tests. If we don't already have a unit test project in our solution, we can add one:

Right-click on the *solution* and select Add -> New Project... -> Visual C# -> Test -> Unit Test Project (.NET Framework).

There are other options such as NUnit and xUnit, but we'll just look at Microsoft's default version. The others do essentially the
same thing but with additional capabilities.

Our project will contain a single test class with a single unit test, like this:

```csharp
namespace UnitTestProject1
{
    [TestClass]
    public class UnitTest1
    {
        [TestMethod]
        public void TestMethod1()
        {
        }
    }
}
```

The `[TestClass]` attribute indicates that the class contains tests. The `[TestMethod]` attribute indicates that the method
*is* a test. This is how the test runner knows which parts of the code in this project are tests that we need to run.

For the sake of simplicity, a unit test is a method that *passes* if it runs without throwing any exceptions, and *fails* if 
it throws an exception. We write these methods so that if our code does what we expect it to do, it won't throw an exception. 
It will pass. If doesn't do what we expect, it will throw an exception and fail.

This will be clearer with an example. First, let's copy our `DateFunctions` class into the same file as our test. We don't have
to do that. Eventually that class will be in our "real" project - not the test project - and our test project will reference it. 
But at this step it's often convenient to have them both in one place. It's going to help with our Stack Overflow question. This
is also how I work if I'm answering someone's question and I want to be sure that my answer works. I just create a test project
and write both the code and the unit tests.

Here's what our file looks like with the `DateFunctions` class included and a simple unit test:

```csharp
[TestClass]
public class DateFunctionTests
{
    [TestMethod]
    public void NumberOfTimesDayOfMonthOccursInDateRange_returns_correct_result()
    {
        var startDate = DateTime.Parse("1/1/2019");
        var endDate = DateTime.Parse("2/15/2019");
        var dayOfMonth = 14;
        var expectedResult = 2;
        var actualResult = DateFunctions.NumberOfTimesDayOfMonthOccursInDateRange(startDate, endDate, dayOfMonth);
        Assert.AreEqual(expectedResult, actualResult);
    }
}

public static class DateFunctions
{
    public static int NumberOfTimesDayOfMonthOccursInDateRange(
        DateTime startDate, DateTime endDate, int dayOfMonth)
    {
        return 0; 
        /*
         * I just put this here so it compiles. This would be your
         * work in progress code. You've written something. It just
         * doesn't return the right results.
         */
    }
}
```

I made the test a little bit verbose, declaring everything as a variable, to make it easier to follow.
It shows that we're going to execute our function with specific inputs, and we know what result
we expect. 

The last statement - `Assert.AreEqual` - will check to see if our actual result is the same as our 
expected result. If it is not, it will throw an exception and the test will fail. 

To try it out, right-click on the test class and select __Run Test(s)__. The project will compile, Visual Studio
will run the test, and a "Test Explorer" window will open showing us the result. 

This test will fail, because our function returned *0*, not the expected result. If we drill down into that
test result by clicking on the red "X" icons, we'll eventually see this exception:

> Message: Assert.AreEqual failed. Expected:<2>. Actual:<0>. 

If we figure out how to fix our method so that it returns the correct result, this
test will no longer fail. Our test results will be green, not red.

If we want to step through the code, we can set breakpoints and then right-click
and select __Debug Test(s)__. This will enable us to step through the code.

### How This Is Better, So Far  

Let's say that you're trying over and over again to fix your function. 
Which is easier:
- Starting up your Windows Forms app, typing in some values, pressing the button, and looking at the result
- Running a unit test and seeing if it passes or fails

The unit test is much easier. Perhaps we've already caught on that creating a form
is a horrible way to test. Maybe we tend to use console apps instead. Even still, isn't running a unit test even easier than that?
A unit test is a lot like a console app, except that instead of us  *looking* at the result to see if it's correct,
the test already knows what the correct result is that it's looking for. It's built into the test.

Imagine if we pasted this code into our Stack Overflow question. It says, in effect,
> Here's the code I need help with. Here's a unit test that describes exactly what it should do.

If I were trying to help with this question, I would probably start by creating a unit test project,
pasting this code into it, and writing some unit tests. I would use the unit tests
to confirm that my solution works. (I want to give you a good answer, which means I'm not going to guess.)

But now, by providing both your code and the unit test, you've made it even easier
to help you. I can paste both the code and the unit test, and all I've got to do is
work with your code to make the unit test pass.

### It Gets Even Better

Do we really trust that our code works as expected because it returns the correct
result for just one set of inputs? Wouldn't we rather test it with multiple inputs? Maybe 
we have some weird edge cases. How do we test this with multiple inputs?

We could copy and paste the test method so that we have several test methods, 
each with different inputs and expected outputs, and then we could run all of those tests.
That's acceptable. That means that we can test our code with a range of inputs
all at the same time. Imagine trying that with a console app. It would be a pain. 
Unit tests make it easy.

But there's an easier approach than creating multiple tests. We can create a
single test that takes multiple inputs. This is called *a parameterized test*. 
It might look like this:

```csharp
[TestClass]
public class DateFunctionTests
{
    [DataTestMethod]
    [DataRow("1/1/2019", "2/15/2019", 14, 2)]
    [DataRow("1/1/2019", "1/13/2019", 14, 0)]
    [DataRow("1/1/2019", "12/31/2019", 31, 7)]
    public void NumberOfTimesDayOfMonthOccursInDateRange_returns_correct_result(
        string startDate, string endDate, int dayOfMonth, int expectedResult)
    {
        var actualResult = DateFunctions.NumberOfTimesDayOfMonthOccursInDateRange(
            DateTime.Parse(startDate), DateTime.Parse(endDate), dayOfMonth);
        Assert.AreEqual(expectedResult, actualResult);
    }
}
```

Each `DataRow` attribute contains a set of values - a start date, an end date, a day of the month, 
and an expected result. Instead of having to write more test methods to test a different
set of values, we can just add a new data row. The test runner will run the test once
for each set of values.

Imagine the frustration if we fix the function so that it works with one set of inputs,
but then returns the wrong results for different inputs. Imagine having to test over and over
with each set of inputs. Now it's easy to verify that the function returns the
correct result for all of the inputs at the same time.

### Our Improved Question
Now, instead of posting a bunch of irrelevant code that other people have to read through, 
we can post this:

---

I'm trying to write a function that returns the number of times a given day of the
month appears in a date range. Here's my code, which isn't working yet, and
some unit tests that show what the results should be. 

Can you help me to find the problem in my code?

```csharp
public static class DateFunctions
{
    public static int NumberOfTimesDayOfMonthOccursInDateRange(
        DateTime startDate, DateTime endDate, int dayOfMonth)
    {
       // Whatever code you have that isn't working right
    }
}

[TestClass]
public class DateFunctionTests
{
    [DataTestMethod]
    [DataRow("1/1/2019", "2/15/2019", 14, 2)]
    [DataRow("1/1/2019", "1/13/2019", 14, 0)] 
    [DataRow("1/1/2019", "12/31/2019", 31, 8)]
    public void NumberOfTimesDayOfMonthOccursInDateRange_returns_correct_result(
        string startDate, string endDate, int dayOfMonth, int expectedResult)
    {
        var actualResult = DateFunctions.NumberOfTimesDayOfMonthOccursInDateRange(
            DateTime.Parse(startDate), DateTime.Parse(endDate), dayOfMonth);
        Assert.AreEqual(expectedResult, actualResult);
    }
}
```
---

You may have seen people on Stack Overflow ask for an [MCVE](https://stackoverflow.com/help/minimal-reproducible-example) which
stands for "Minimal, Complete, Verifiable Example." In other words, they're
asking you to provide just the part of your code that you need help with, not
all sorts of extraneous details, make sure you don't leave out the important parts,
and show how you know whether or not it's working.

That's exactly what we've accomplished:

- __Minimal__ - we've stripped away all of the code that's *not* related to the
date function.
- __Complete__ - the whole date calculation is in that function
- __Verifiable__ - the unit test shows both what you expect and how you know if it's working.

Assuming that you've made some effort to write the code - you're showing your attempt at writing the function - 
you've written the perfect question. 

### The Best Part
The effort that you put into asking this question will make understanding and 
debugging your code so much easier that you're far more likely to solve your own
problem before you even post the question. 

These are roughly the same steps that most developers take to make their own problems
easier to understand and solve. We put code that performs specific functions into separate
classes and methods so that we can test it in isolation from other code. And then
we write unit tests to verify that our code does what we think it should do. 

We all make mistakes in our code, even in code that should seem trivial. This allows
us to catch our mistakes more quickly. We can often find our mistakes right after we make them.
It also means that if we must step through our code in the debugger, we won't have to step through
as much of it. Instead of setting breakpoints, starting up an app, and providing inputs just to test some
tiny piece of code, we can isolate just the part we're working on.

In the long run this means that we work faster and with greater confidence. If we haven't worked with unit tests 
we might expect that writing the extra test code will cause us to work more slowly, but it's actually the opposite.
When we subtract the time that we would have spent searching for defects that turn up later, re-reading code we
wrote days or weeks ago, and stepping through lots of code in the debugger, the net result is that we save lots of time.

It's not immediate. There is a little bit of a learning curve, and it might slow us down at first. 
But if you were ever going to make a leap of faith, this is the one to make. Once you're on the other side of it, you'll 
never want to go back. You might even find yourself wondering how you ever wrote code without unit tests. It's that good.

You may have read articles critical of unit testing, arguing that we can end up writing way too many useless tests and that 
it somehow damages the design of our code. There is usually an element of truth to such arguments. We may lose our balance
at first and veer from writing no tests to writing too many. The quality of our tests might be low. That's okay. It's part
of the learning experience. It helps if we realize that by not writing any unit tests, we're already at one extreme. We may
go to the opposite extreme before finding our balance.

As much as this practice has helped me, I still have a great deal to learn about unit testing.

Unit testing is also like a gateway drug that leads to all sorts of improved practices. In this example we had to move code
out of a Windows Form into its own class so that we could test it. That's a good habit. Making it possible to test the code 
improved the design of the code. That pattern repeats. Over time we're likely to find that arranging our code so that we
can write tests for it significantly improves the design of our code in other ways.

---

Was the premise of this whole post just a trick to introduce unit tests? Yes and no. I often read Stack Overflow questions
that would be much easier to understand if they included unit tests to describe the expected results. Sometimes I demonstrate
the idea by including unit tests in the answer, but explaining how to write tests would be way outside the scope of the
original question.

At the same time, I'm convinced that most developers who don't write unit tests would benefit from learning to write them.
As I hope I've demonstrated, they change the way we write our code, making it easier to isolate and solve our own problems.
We all get stuck and need help. I look things up on Stack Overflow all the time. But this change increases our ability to
push through problems on our own, increasing both our productivity and our confidence.
