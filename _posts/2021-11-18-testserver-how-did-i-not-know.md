---
layout: post
title:  "ASP.NET Core Integration Tests With TestServer - How Did I Not Know About This?"
date:   2021-11-18 01:20:58 -0400
category: blog
description: TestServer makes writing integration tests for ASP.NET Core application easy.
---

I've been looking for ways to write better integration tests. My [previous set of experiments](/blog/2021/04/07/integration-test-experiment-1.html) proved useful in practice, but the approach could use maybe one level of de-complication.

In the meantime I discovered something out-of-the-box tool that provides a simple way to test ASP.NET Core applications: [ASP.NET Core `TestServer`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.testhost.testserver). It doesn't replace what I worked on previously. It's something separate. I use both approaches. But this is easier and doesn't require changing the way an application is configured.

In this post I'll start with a simple scenario in case that's all you need and then get into some of the more fun stuff we can do with `TestServer`. Once you see the potential you'll likely get more ideas about how to write the sort of tests you want to write.

### The Shortest, Simplest Version
I'm starting with Visual Studio's included Web API application which includes a `WeatherForecastController` which returns a random weather forecast from this GET endpoint:

```csharp
[HttpGet]
public IEnumerable<WeatherForecast> Get()
{
    var rng = new Random();
    return Enumerable.Range(1, 5).Select(index => new WeatherForecast
    {
        Date = DateTime.Now.AddDays(index),
        TemperatureC = rng.Next(-20, 55),
        Summary = Summaries[rng.Next(Summaries.Length)]
    })
    .ToArray();
}
```

Next I'm adding a test project using the "NUnit Test Project (.NET Core)" template. You can use MsTest. It doesn't matter. The test project references the Web API project. And I'm adding a few NuGet packages:

- Microsoft.AspNetCore.TestHost v3.1.21 
- Newtonsoft.Json (latest)
- Moq (latest)

Here's the test file in its entirety. I'll explain what each part does, but it's probably obvious just from looking it.

```csharp
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.TestHost;
using Newtonsoft.Json;
using NUnit.Framework;
using System.Net.Http;
using System.Threading.Tasks;

namespace TestServerBlogPost.Tests
{
    [TestFixture]
    public class WeatherForecastIntegrationTests
    {
        private TestServer _testServer;
        private HttpClient _testClient;

        [SetUp]
        public void Setup()
        {
            _testServer = new TestServer(
                new WebHostBuilder()
                    .UseStartup<Startup>());
            _testClient = _testServer.CreateClient();
        }

        [TearDown]
        public void TearDown()
        {
            _testClient?.Dispose();
            _testServer?.Dispose();
        }

        [Test]
        public async Task Controller_Returns_Forecast()
        {
            var request = new HttpRequestMessage(HttpMethod.Get, "/weatherforecast");
            HttpResponseMessage response = await _testClient.SendAsync(request);
            string responseContent = await response.Content.ReadAsStringAsync();
            WeatherForecast[] forecasts = JsonConvert.DeserializeObject<WeatherForecast[]>(responseContent);
            Assert.AreEqual(5, forecasts.Length);
        }
    }
}
```

Here's what's happening:
- We're creating a `TestServer` using a `WebHostBuilder` that in turn uses the `Startup` from the Web API application. 
- `TestServer` exposes an `HttpClient` which allows us to send HTTP requests to the application. Both are created before each test runs and disposed after each test runs.
- The test sends an HTTP request to the "weatherforecast" endpoint, gets a response, and asserts something about that response. 

We can already see the possibilities. We can send different types of requests. We can add headers to requests to verify that our authorization works. We can inspect response headers if that's what we want to verify. Our tests include middleware.

Some other benefits:
- Unlike tests that require starting the whole application, these are fast. The first test might take 300ms, but after that they're less than 50ms. 
- We don't have to duplicate models used within the application. The test references the application and uses its models.
- We're not duplicating what's in `Startup` so we can run tests. The more we do that, the less we're testing our actual runtime code, defeating the purpose of the test. We're doing the opposite - we're testing `Startup`. If there's some dependency we haven't configured in `Startup` we'll catch that. 

I wouldn't write a lot of these. We can cover most application logic with unit tests. These integration tests are only to cover what unit tests don't cover: Making sure that the pieces are all configured correctly and they all come together as a whole.

Having covered the happy path, let's look at some other scenarios.

### Configuration

What if our `Startup` depends on some configuration value? I'm going to change startup so that it depends on a configuration value by adding this to `ConfigureServices`:

```csharp
var awesomeness = Configuration.GetValue<int>("awesomeness");
if (awesomeness < 1) throw new Exception("Not awesome enough!");
```
Now the test that used to pass fails instead because that setting is missing. So how do we supply it?

Here's one way. It feels like cheating but I can live with it. In the web application's `Program` class I have this:

```csharp
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureHostConfiguration(configurationBuilder =>
        {
            configurationBuilder.AddJsonFile("appsetting.json");
        })
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.UseStartup<Startup>();
        });
```
It loads a JSON settings file, and I've put the setting in appsetting.json so the app can run:

```json
"awesomeness": "1"
```
Now how do we supply this setting to our test also? One solution is just to copy the settings file into the test project. Maybe add a comment noting that it's a copy of the "real" file. Make sure the file's Properties indicate that it's copied to the output directory. 

Then, in the test, make sure that settings file is included in the configuration:

```csharp
[SetUp]
public void Setup()
{
    _testServer = new TestServer(
        new WebHostBuilder()
            .ConfigureAppConfiguration(configurationBuilder =>
            {
                configurationBuilder.AddJsonFile("appsetting.json");
            })
            .UseStartup<Startup>());
    _testClient = _testServer.CreateClient();
}
```
(Note that in `Program` it's `ConfigureHostConfiguration` but here it's`ConfigureAppConfiguration`.)

Now the test passes. That might be good enough. But what if you want your tests to use different values? We don't want to have to create separate versions of our settings files. In that case we can supply or override those settings in our test using `AddInMemoryCollection` like this:

```csharp
[SetUp]
public void Setup()
{
    _testServer = new TestServer(
        new WebHostBuilder()
            .ConfigureAppConfiguration(configurationBuilder =>
            {
                configurationBuilder.AddJsonFile("appsetting.json");
                configurationBuilder.AddInMemoryCollection(CreateInMemoryConfiguration());
            })
            .UseStartup<Startup>());
    _testClient = _testServer.CreateClient();
}

private IDictionary<string, string> CreateInMemoryConfiguration()
{
    var configuration = new Dictionary<string, string> { {"awesomeness", "2"} };
    return configuration;
}
```
Because we call `AddInMemoryCollection` after `AddJsonFile` the new value we supply overrides what's in the file. We could decide that we don't want the file at all or we could keep it for convenience and override whatever we want to change. It might make sense to have a second file that overrides settings in the first file. Our options are open.

### Adding Missing Dependencies
Sometimes the application depends on components that are registered in `Program` instead of in `Startup`, and those can cause our tests to fail. If we need to we can add them to our `TestServer` using `ConfigureServices` like this:

```csharp
[SetUp]
public void Setup()
{
    _testServer = new TestServer(
        new WebHostBuilder()
            .ConfigureAppConfiguration(configurationBuilder =>
            {
                configurationBuilder.AddJsonFile("appsetting.json");
                configurationBuilder.AddInMemoryCollection(CreateInMemoryConfiguration());
            })
            .ConfigureServices(services =>
            {
                services.AddTransient(typeof(ILogger<>), typeof(NullLogger<>));
            })
            .UseStartup<Startup>());
    _testClient = _testServer.CreateClient();
}
```

### Replacing Services With Fakes or Mocks
This is where `TestServer` provides some great flexibility. Since this is an integration test we might want it to interact with a database instance. In that case the test will get the connection string from the settings file or we can supply a different one somehow. 

What if for whatever reason we don't want our test communicating with a real database? What if want to replace it with some in-memory database? 

Let's say our application depends on this interface, and the implementation interacts with the database:

```csharp
public interface IFooRepository
{
    Task<Foo> GetFoo(Guid id);
    Task SaveFoo(Foo foo);
}
```
`Startup` may register a "real" implementation, but `TestServer` allows us to override it. Moq isn't my first choice, but it's brief and familiar, so let's replace the runtime implementation with a mock. 

We'll add a field for the mock, and then `Setup` will instantiate it and use the `ConfigureTestServices` method to override whatever implementation `Startup` specifies for `IFooRepository`.

```csharp
private Mock<IFooRepository> _fooRepositoryMock;

[SetUp]
public void Setup()
{
    _fooRepositoryMock = new Mock<IFooRepository>();
    _testServer = new TestServer(
        new WebHostBuilder()
            .ConfigureAppConfiguration(configurationBuilder =>
            {
                configurationBuilder.AddJsonFile("appsetting.json");
                configurationBuilder.AddInMemoryCollection(CreateInMemoryConfiguration());
            })
            .ConfigureServices(services =>
            {
                services.AddTransient(typeof(ILogger<>), typeof(NullLogger<>));
            })
            .ConfigureTestServices(services =>
            {
                services.AddSingleton<IFooRepository>(_fooRepositoryMock.Object);
            })
            .UseStartup<Startup>());
    _testClient = _testServer.CreateClient();
}
```
Now we can write tests that set up the mock to return certain values or we can use it to verify that the application saved the expected `Foo`.

### Resolving Services
In the above example we're using a mock instead of a real database. What if we wanted to use a real database but we needed to set up test data (inserting a `Foo`) or see what was inserted into it (reading a `Foo`)?

`TestServer` exposes the application's `IServiceProvider` via its `Services` property. We can use that to resolve `IFooRepository` and interact with it. Instead of writing more database code to insert and inspect records, we can use the same classes that we use in our application.

In our test method we can set up test data like this:

```csharp
var fooRepository = this._testServer.Services.GetService<IFooRepository>();
await fooRepository.SaveFoo(new Foo {Id = Guid.NewGuid(), Name = "Foo!"});
```
### A Base Class to Make Writing Tests Easier
I rarely write base classes for tests but in this case it helps by reducing duplicate code. This base class 
- encapsulates creation and disposal of `TestServer`
- exposes the `HttpClient` needed to send requests and the `IServiceProvider` for resolving services
- calls virtual methods for adding configuration settings or replacing services with fakes, mocks, or something else

```csharp
[TestFixture]
public class TestServerTestBase // worst name ever
{
    private TestServer _testServer;
    protected HttpClient TestClient { get; private set; }
    protected IServiceProvider Services => _testServer.Services;

    public virtual void Setup()
    {
        _testServer = new TestServer(
            new WebHostBuilder()
                .ConfigureAppConfiguration(configurationBuilder =>
                {
                    configurationBuilder.AddJsonFile("appsetting.json");
                    configurationBuilder.AddInMemoryCollection(CreateInMemoryConfiguration());
                })
                .ConfigureServices(services =>
                {
                    services.AddTransient(typeof(ILogger<>), typeof(NullLogger<>));
                })
                .ConfigureTestServices(OverrideServices)
                .UseStartup<Startup>());
        TestClient = _testServer.CreateClient();
    }

    [TearDown]
    public void TearDown()
    {
        TestClient?.Dispose();
        _testServer?.Dispose();
    }

    protected virtual void OverrideServices(IServiceCollection services)
    {
    }

    protected virtual Dictionary<string, string> CreateInMemoryConfiguration()
    {
        return new Dictionary<string, string>();
    }
}
```

How is it possible that I wrote slower, more complicated, more brittle tests for years and didn't know about this? It's a mystery. I hope you find it useful.