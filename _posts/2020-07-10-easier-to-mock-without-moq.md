---
layout: post
title:  "Sometimes It's Easier to Mock Without Moq"
date:   2020-07-10 -0400
category: blog
description: I like Moq and use it, but sometimes there's a better way.
---

I like [Moq](https://github.com/Moq/moq4/wiki/Quickstart). It's easy to use and it's used frequently enough
that developers are familiar with it. But sometimes there's an even easier way to write tests, and that
involves creating classes that mock interfaces.

As an example (I'll try not to convolute this) suppose we have a class that
- listens to messages from a queue
- reads the contents of the message and converts it into a command
- invokes a command handler

I want to write a test to verify that given a message with specific contents,
the command sent to the command handler has the expected properties.

Here's what that test might look like using Moq:
)

```csharp
[TestMethod]
public async Task CommandHandlerInvokedWithCorrectCommand()
{
    var commandHandlerMock = new Mock<ICommandHandler<DoSomethingCommand>>();
    DoSomethingCommand handledCommand = null;
    commandHandlerMock.Setup(x =>
            x.HandleCommandAsync(It.IsAny<DoSomethingCommand>()))
        .Callback<DoSomethingCommand>(command => handledCommand = command);
    var subject = new MessageListener(commandHandlerMock.Object);
    var message = CreateMessage("X", 2);
    await subject.ReceiveMessage(message);

    Assert.IsTrue(handledCommand.SomeProperty == "x"
                  && handledCommand.SomeOtherProperty == 2);
}
```

In case you haven't used this, here's what we're doing:
- Using Moq to create a `Mock<ICommandHandler<DoSomethingCommand>>`
- Setting up the mock so that when `HandleCommandAsync` is called,
it executes a callback that assigns the `DoSomethingCommand` parameter
passed to the command handler to the `handledCommand` variable so that
we can inspect the command that was sent to the handler.

This isn't bad. I've written it a bunch of times, and developers who've
used Moq know what it means. But can we make it a little simpler?
Judge for yourself.

In the time it takes me to set up that mock, I can create this class:

```csharp
public class MockCommandHandler<T> : ICommandHandler<T>
{
    public T HandledCommand { get; private set; }

    public async Task HandleCommandAsync(T command)
    {
        HandledCommand = command;
    }
}
```

That took seconds. What does it do for the unit test?
I think it's an improvement.

```csharp
[TestMethod]
public async Task CommandHandlerInvokedWithCorrectCommand()
{
    var commandHandler = new MockCommandHandler<DoSomethingCommand>();
    var subject = new MessageListener(commandHandler);
    var message = CreateMessage("X", 2);
    await subject.ReceiveMessage(message);
    var handledCommand = commandHandler.HandledCommand;
    Assert.IsTrue(handledCommand.SomeProperty == "x"
                  && handledCommand.SomeOtherProperty == 2);
}
```

Isn't that easier to read? It's undeniably shorter. And I can reuse that
mock command handler in other tests. It also makes a difference if a
test necessarily sets up more dependencies. We want to keep our tests as
short and easy to read as possible.

### The Wall of Moq

As a side note, sometimes if classes have too many dependencies we declare
a bunch of Moq mocks as private variables in our tests followed by lots of initialization code that
instantiates them and sets them up, and still individual tests have to do different setup for some of them.
All we see when we open the file is Moq.

That's not a Moq problem - it's a problem with the classes we're testing. They
have too many dependencies. Anyway, I don't like it. I call it "the Wall of Moq."
It makes tests difficult to read. We'll see mocks set up with behaviors they don't
need because no one can keep track of which ones are needed for any one test.
Once that happens we can no longer understand either the tests or the code
we're testing.

### It's Really Easy With Delegates

What if instead of an `ICommandHandler<T>` interface we have a delegate,
and we inject that into our class? (More on that [here](/blog/2018/03/18/depending-on-functions-instead-of-interfaces.html)).

If we inject this delegate into our message listener instead of an interface:

```csharp
public delegate Task CommandHandler<T>(T command);
```

...then we can use a local method as our mock:

```csharp
[TestMethod]
public async Task CommandHandlerInvokedWithCorrectCommand()
{
    DoSomethingCommand handledCommand = null;
    async Task CommandHandlerMock(DoSomethingCommand command) => handledCommand = command;
    var subject = new MessageListener(CommandHandlerMock);
    var message = CreateMessage("X", 2);
    await subject.ReceiveMessage(message);
    Assert.IsTrue(handledCommand.SomeProperty == "x"
                  && handledCommand.SomeOtherProperty == 2);
}
```

Now we don't have to declare a separate `MockCommandHandler`, and
we can still declare and set up the mock in a single succint line.

It's not a huge difference, but these alternatives will save us some typing,
make it easier to reuse code in our tests, and keep our tests a little bit
easier to read. And if I'm just mocking a method that returns something
I'll still use Moq.

