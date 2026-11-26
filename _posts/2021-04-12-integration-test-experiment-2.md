---
layout: post
title:  "An Experiment With Making Integration Tests Easier to Write - Part Two"
date:   2021-04-12 -0400
category: blog
description: Integration tests are easier to write if we can re-use our Ioc setup.
---

In the <a href="/blog/2021/04/07/integration-test-experiment-1.html" target="_blank">previous post</a>
I arranged the dependency composition code for a small application so that the dependencies I would mock when integration
testing are separated from the classes I want to test. (As noted in this post, I'm referring to
<a href="https://martinfowler.com/bliki/IntegrationTest.html" target="_blank">“narrow integration tests“</a> 
which test the successful collaboration of code while mocking external systems like HTTP endpoints and databases.)

The plan is to create integration tests which re-use the production code used to compose the classes I want to test
(I'm calling these "internal dependencies") so that the tests don't duplicate it or compose them differently. If that
happened the tests would be less reliable. Hopefully this will also make the tests less brittle. Instead of calling
constructors to create the classes I'm going to test I can just resolve them from the IoC container (in this case
an `IServiceProvider`.) If the composition changes I won't have to modify the tests as much.

As I mentioned in the last post, this is confusing when I described it to others and I wasn't even sure if it would make
any sense until I wrote some code. So I'll get straight into demonstrating it. This is all weird, fake code that doesn't
do anything meaningful but it reflects real code that resulted in real integration tests. 

## The Scenario

The fake code I'm testing does a few things:

- There's a `FooMessageListener` which receives a Service Bus message. It deserializes the message to get some
  Foo data, creates a `FooCommand`, and passes it to an `IFooCommandHandler`.
 - The implementation - `FooCommandHandler` - invokes an `IFooValidator`. The implementation of that, in turn, calls
  an HTTP endpoint to determine if the Foo is valid.
 - If the Foo is valid, the command handler saves it using an `IFooRepository`. If not it throws an exception.

The point of all this is just to have a class - `FooMessageListener` - with a few dependencies, 
some of which are mocked and some aren't.

In the production code we have these two classes which add the internal and external dependencies to an `IServiceCollection`:

```csharp
public class FooApplicationInternalDependencies : IDependencyComposition
{
    public void ComposeDependencies(IServiceCollection serviceCollection)
    {
        serviceCollection.AddSingleton<IFooValidator, FooApiFooValidator>();
        serviceCollection.AddSingleton<IFooCommandHandler, FooCommandHandler>();
        serviceCollection.AddSingleton<FooMessageListener>();
    }
}

public class FooApplicationExternalDependencies : IDependencyComposition
{
    private readonly IConfiguration configuration;

    public FooApplicationExternalDependencies(IConfiguration configuration)
    {
        this.configuration = configuration;
    }

    public void ComposeDependencies(IServiceCollection serviceCollection)
    {
        serviceCollection.AddLogging();
        serviceCollection.AddHttpClient<FooApiClient>();

        string connectionString = this.configuration["sqlConnectionString"];
        serviceCollection.AddScoped<IFooRepository, SqlFooRepository>(provider =>
            new SqlFooRepository(connectionString));
    }
}
```

## Mocking Dependencies

The integration tests will re-use `FooApplicationInternalDependencies`. There's no need to duplicate it. 
But we'll create another class to replace the external dependencies with mocks:

```csharp
public class MockedFooDependencies : IDependencyComposition
{
    public MockedFooDependencies()
    {
        HttpMessageHandler = new MockHttpMessageHandler();
        FooRepository = new Mock<IFooRepository>();
    }

    public MockHttpMessageHandler HttpMessageHandler { get; }
    public Mock<IFooRepository> FooRepository { get; }

    public void ComposeDependencies(IServiceCollection serviceCollection)
    {
        serviceCollection.AddHttpClient<FooApiClient>()
            .ConfigurePrimaryHttpMessageHandler(() => HttpMessageHandler);
        serviceCollection.AddSingleton(FooRepository.Object);
        serviceCollection.AddSingleton(typeof(ILogger<>), typeof(NullLogger<>));
    }
}
```

This class is similar to `FooApplicationExternalDependencies` except that it uses mocks (Moq) for the
API client, the repository, and logging. Those mocks are exposed as properties so that our tests can interact with 
them. 

In this example all of the mocks are for use by `FooMessageListener` and its dependencies. That class will be the
entry point for my tests. The tests will interact with all the classes and the mocks, but I'll run the tests by
sending messages to that listener. But I wouldn't create one of these classes for each entry point. Rather I'd start
with one for the composition root of the application. Many classes will use the same mocks. Hopefully this will keep
the test code less brittle. In this class I'll create mocks for external dependencies required by the application,
not for individual entry points. 

What's this `MockHttpMessageHander`? I could have just mocked `FooApiClient` entirely, and that would be okay. 
But as I separate classes into those I test and those I mock, I want to test more and mock less. So instead of mocking
the whole class I'm mocking the `HttpMessageHandler` which gets injected into the `HttpClient` which in turn gets injected 
into `FooApiClient`. `MockHttpMessageHander` is crude. It allows me to specify what the response will be for a request, 
regardless of any details of that request such as the URL, content, or headers. 
<a href="https://github.com/WireMock-Net/WireMock.Net" target="_blank">WireMock</a> or something else more thorough is probably
a better real-world choice. I'm using this just to illustrate that I'd rather test more and mock less.

```csharp
public class MockHttpMessageHandler : HttpMessageHandler
{
    private HttpResponseMessage responseMessage;

    public void SetResponse<TResponseContent>(TResponseContent content, HttpStatusCode statusCode = HttpStatusCode.OK)
    {
        this.responseMessage = new HttpResponseMessage(statusCode)
        {
            Content = new StringContent(JsonConvert.SerializeObject(content))
        };
    }

    protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
    {
        if (this.responseMessage == null)
        {
            throw new InvalidOperationException("You must specify a response message by calling SetResponse.");
        }
        return Task.FromResult(this.responseMessage);
    }
}
```
Because this is a "narrow" integration test it doesn't verify that `FooApiClient` correctly integrates with a remote API.
For example, the client might call the wrong endpoint or fail to provide the correct headers. Ideally there would
be a separate integration test for that. (More on that later.)

## A Composition Root for Integration Tests

Next I'll add a class that composes all of the dependencies - internal and external - for my integration tests:

```csharp
public class FooTestComposition
{
    private readonly List<Action<IServiceCollection>> serviceConfigurations;
    private IServiceProvider serviceProvider;

    public FooTestComposition()
    {
        MockedFooDependencies = new MockedFooDependencies();
        this.serviceConfigurations = new List<Action<IServiceCollection>>();
    }

    public MockedFooDependencies MockedFooDependencies { get; }

    public void AddServiceConfiguration(Action<IServiceCollection> serviceConfiguration)
    {
        this.serviceConfigurations.Add(serviceConfiguration);
    }

    public void Build()
    {
        var serviceCollection = new ServiceCollection();
        var applicationComposition = new ApplicationComposition(
            serviceCollection,
            new FooApplicationInternalDependencies(),
            MockedFooDependencies);
        applicationComposition.ComposeDependencies();
        this.serviceConfigurations.ForEach(serviceOverride => serviceOverride.Invoke(serviceCollection));
        this.serviceProvider = serviceCollection.BuildServiceProvider();
    }

    public T ResolveService<T>()
    {
        if (this.serviceProvider == null)
        {
            throw new InvalidOperationException("Call Build() before attempting to resolve services.");
        }

        return this.serviceProvider.GetRequiredService<T>();
    }
}
```

Just like the `ApplicationComposition` class from the previous post this configures an `IServiceCollection` with 
both the internal and external (mocked) dependencies. But it does a few things differently:

- It exposes `MockedFooDependencies` so that we can interact with the mocks. We can set up their behaviors or
 verify calls made to them.
- The `AddServiceConfiguration` method allows us to register services directly with the `IServiceCollection`. 
I'll come back to this, but the point is that not every test is required to use the mocks defined in `MockedFooDependencies`.
  We can override those registrations to use other mocks or implementations.

This might seem like a lot of code, but I'll be able to re-use it to keep lots of integration tests short and simple.
It could use some refactoring. As-is it's coupled to both `FooApplicationInternalDependencies` and 
`MockedFooDependencies`. But it's not that long so I'll leave it alone for now. First I want to get to the end and
see what sort of integration tests this enables me to write. And here they are:

## What Do the Integration Tests Look Like?

Because these are integration tests I'm only verifying the final outcomes. If the remote API says that the `Foo` data
is valid it gets saved to the repo. If not an exception is thrown and it's not saved to the repo. 

```csharp
[TestClass]
public class FooMessageReceivedTests
{
    private MockedFooDependencies mocks;
    private FooMessageListener messageListener;

    [TestInitialize]
    public void Setup()
    {
        var testComposition = new FooTestComposition();
        testComposition.Build();
        this.messageListener = testComposition.ResolveService<FooMessageListener>();
        this.mocks = testComposition.MockedFooDependencies;
    }

    [TestMethod]
    public async Task Message_Received_Happy_Path()
    {
        var fooId = Guid.NewGuid();
        var fooName = Guid.NewGuid().ToString();
        var messageContent = new FooMessage {FooId = fooId, Name = fooName};
        Message message = CreateMessage(messageContent);
        this.mocks.HttpMessageHandler.SetResponse(new FooApiResponse{ IsValid = true});

        await this.messageListener.Receive(message);

        this.mocks.FooRepository.Verify(x => x.Save(It.Is<Foo>(foo => 
            foo.Id == fooId && foo.Name == fooName)));
    }

    [TestMethod]
    public async Task Invalid_Foo_Is_Not_Saved()
    {
        var messageContent = new FooMessage { FooId = Guid.NewGuid(), Name = Guid.NewGuid().ToString() };
        Message message = CreateMessage(messageContent);
        this.mocks.HttpMessageHandler.SetResponse(new FooApiResponse { IsValid = false });

        await Assert.ThrowsExceptionAsync<Exception>(async () => await this.messageListener.Receive(message));

        this.mocks.FooRepository.Verify(x => x.Save(It.IsAny<Foo>()), Times.Never());
    }

    private Message CreateMessage(object body)
    {
        var bytes = Encoding.UTF8.GetBytes(JsonConvert.SerializeObject(body));
        var message = new Message(bytes);
        return message;
    }
}
```

This is where the work seems to pay off. They're less brittle because they're not coupled to the constructors of all the classes involved. The test subject - 
an instance of `FooMessageListener` - is created by resolving it from the IoC container:

```csharp
this.messageListener = testComposition.ResolveService<FooMessageListener>();
```

If I add more classes that don't need mocking to the application's composition those changes are already reflected in
the tests because that part of the composition root is shared between production and the tests.

The tests are shorter because they don't have to repeatedly define all of the mocks. They just perform whatever setup
and verification is required. If I add classes to the composition root that I will need to mock I'll have to update 
`MockedFooDependencies` so that the tests can resolve them. That's why I'd prefer to have that class include
all the mocked dependencies for the entire application instead of just the ones for individual entry points. That means
one change instead of multiple changes and prevents me from having to keep track of which mocks are used by which entry
points. (It also creates a slight risk that I'll neglect removing mocks if in the future they're not used by any tests.)

## What's Next?

So far the emphasis has been on creating narrow integration tests that mock external
dependencies. But having gone down this path where we're using our IoC container to set
up tests, we're not limited to narrow integration tests. We can modify this approach to
integrate our tests with real or test versions of HTTP endpoints, databases, etc.

There are a few ways to do that. `FooTestComposition` has the `AddServiceConfiguration` method
which allows individual tests to override the mocks and use different implemntations. 
We could write tests that replace the repository mock with a concrete repository class
so that our tests interact with an database. The `[TestInitialize]` method might look like this:

```csharp
[TestInitialize]
public void Setup()
{
    var testComposition = new FooTestComposition();
    testComposition.AddServiceConfiguration(
        serviceCollection => 
            serviceCollection.AddSingleton<IFooRepository>(new SqlFooRepository("someConnectionString")));
    testComposition.Build();
    this.messageListener = testComposition.ResolveService<FooMessageListener>();
    this.mocks = testComposition.MockedFooDependencies;
}
```

Now the test would still use the mocked HTTP response but an actual database. 
We could similarly set up `FooApiClient` to use a real endpoint. 

What if we want to use an actual HTTP endpoint *and* database without mocking anything? In that case
our integration tests could use both `FooApplicationInternalDependencies` and 
`FooApplicationExternalDependencies`, sharing them with the production code. We would simply provide different
configuration values. We'd have no mocks. 

## Conclusion

I've tried this approach on a small scale with real-world code. I liked the results but it's still experimental. 
I mentioned at the outset of the previous post that I don't see many of this type of narrow integration test. 
That means that just as I have limited experience setting them up, I also don't have much experience actually
using them. I can't speak as much as I'd like to how useful they are and what the pitfalls are.

My motivation is twofold.
- I see lots of end-to-end tests that run all of the application components and interact with them. They
might actually send Service Bus messages or begin by sending requests to a locally run HTTP endpoint.
  Those types of tests are valuable but far slower and more work to create and maintain. I'd like to have
  enough of those to ensure that the applications are configured correctly and interact with their environment. 
  (For example, does the message listener receive a real message from a real Service Bus queue?) But I
  don't want every integration test to be that complex and slow.
- If I'm going to write integration tests that don't run the whole application, I didn't know a good way to set
those up. Creating compositions of production code and mocks by calling constructors from within test classes
  seemed too complicated and brittle and would deviate too much from production code. I didn't feel like I could
  even begin writing these sorts of tests without addressing that first.
  
Does it make any sense? I'll say it's good enough that I'll apply it again with the next opportunity that I 
have, although I'm likely to refactor or make even larger changes. I welcome any feedback. Maybe there's a wheel I don't need to reinvent. Or maybe this will help someone. 
I can't tell without trying and sharing. Let me know.

