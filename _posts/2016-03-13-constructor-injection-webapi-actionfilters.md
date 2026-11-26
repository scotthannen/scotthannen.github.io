---
layout: post
title:  'Dependency (Constructor) Injection With Web API Action Filters'
date:   2016-03-13 07:20:58 -0400
category: blog
---
There’s nothing original here. I wanted to be able to inject dependencies into Web API action filters and couldn’t see how until I read [this post](http://blog.ploeh.dk/2014/06/13/passive-attributes/) by Mark Seemann, author of _[Dependency Injection in .NET](https://www.manning.com/books/dependency-injection-in-dot-net)_.

Although a few other pages cited that post as containing the solution to my problem, it didn’t jump right out at me. Maybe it didn’t for you either. Or maybe you’re just smarter than me. Lots of people are.

But perhaps this simplified explanation of what Seemann is doing will make it more accessible. (Call this _Reading Other People’s Posts For Dummies, by One_.)

### Why Dependency Injection Into Action Filters Seems Harder Then Dependency Injection Into Controllers

We often create action filters by inheriting from `ActionFilterAttribute`, like this:

``` csharp
    public class MyActionFilterAttribute : ActionFilterAttribute
    {
        public override void OnActionExecuting(HttpActionContext actionContext)
        {
            //Action filter does something
        }
    }
```

It’s an `Attribute`, so if we want to apply the filter to a controller or action we just add the attribute:

``` csharp
    [MyActionFilter]
    public class MyController : ApiController
```

But what if `MyActionFilterAttribute` has a constructor with dependencies? Now we can’t just add the attribute because we call the constructor wherever we use the attribute. This is not what we want:

``` csharp
    [MyActionFilter(new Something(), "someValue")]
    public class MyController : ApiController { }
```

### The Solution

The solution is this:

*   Create an empty attribute. (In his post Seemann calls it a “passive attribute.”) It doesn’t actually do anything. We just put it on controllers and actions.
*   Create an action filter which doesn’t inherit from `ActionFilterAttribute`. It’s not an attribute. This class can use constructor injection.
*   Register the action filter as a global filter. It’s going to get called for every request.
*   When the action filter is executing it looks at the controller and/or action to see if the attribute we defined is present. If the attribute isn’t there then the filter doesn’t do anything.
*   Even though the filter is global, it’s really only executing on methods where the attribute is present. So it works just like an `ActionFilterAttribute`. It only really applies when the attribute is present.

### Where Does the Dependency Injection Container Fit In?

The above only works if our action filter is resolved by our dependency injection container so that the container can inject dependencies into the constructor. How do we do that? By resolving a _single instance_ of the action filter from our container into the `GlobalFilterCollection`. I’m not going to gloss over that. First the code for the action filter, and then the configuration.

### What Does the Action Filter Look Like?

Seemann shows an implementation in [his post](http://blog.ploeh.dk/2014/06/13/passive-attributes/). I’m just making two changes:

*   I moved the more confusing parts into a generic base class so that I never have to <strike>write</strike> copy/paste the same code again.
*   I’m checking for the attribute on the action and on the controller.

Here’s the base class:

``` csharp
    public abstract class ActionFilterBehaviorBase<TAttribute> : IActionFilter where TAttribute : Attribute
    {
        public Task<HttpResponseMessage> ExecuteActionFilterAsync(HttpActionContext actionContext, CancellationToken cancellationToken, Func<Task<HttpResponseMessage>> continuation)
        {
            if (ControllerHasAttribute(actionContext) || ActionHasAttribute(actionContext))
            {
                return ExecuteFilterBehavior(actionContext, cancellationToken, continuation);
            }
            return continuation();
        }

        protected abstract Task<HttpResponseMessage>  ExecuteFilterBehavior(HttpActionContext actionContext, CancellationToken cancellationToken,
            Func<Task<HttpResponseMessage>> continuation);

        public virtual bool AllowMultiple { get; } = false;

        private bool ControllerHasAttribute(HttpActionContext actionContext)
        {
            return actionContext
                    .ControllerContext.Controller.GetType()
                    .GetCustomAttributes(false)
                    .Any(attribute => attribute.GetType().IsAssignableFrom(typeof(TAttribute)));
        }

        private bool ActionHasAttribute(HttpActionContext actionContext)
        {
            return actionContext
                .ActionDescriptor
                .GetCustomAttributes<TAttribute>()
                .Any();
        }
    }
```

It overrides `ExecuteActionFilterAsync` and does two things:

*   It checks for the attribute (`TAttribute`)
*   It executes its own custom behavior only if the attribute is found on the controller or action.

Here’s how it’s used:

First, an empty (passive) attribute:

``` csharp
    [AttributeUsage(AttributeTargets.Class | AttributeTargets.Method)]
    public class MyCustomFilterAttribute : Attribute { }
```

We can put that on a controller or action, but by itself it won’t do anything.

Next, the action filter:

``` csharp
    public class MyCustomFilterBehavior : ActionFilterBehaviorBase<MyCustomFilterAttribute>
    {
        protected override Task<HttpResponseMessage> ExecuteFilterBehavior(HttpActionContext actionContext, CancellationToken cancellationToken, Func<Task<HttpResponseMessage>> continuation)
        {
            //The custom behavior of the filter goes here. This is only executed if
            //MyCustomFilterAttribute is found.
        }
    }
```

It’s as easy to write as an “ordinary” filter that inherits from `ActionFilterAttribute`.

### How Does the Action Filter Get Resolved by the Container?

There are different ways to arrange this. To be honest, I didn’t overthink it. What matters most is that it’s taking place in the composition root (at application startup.) We’re configuring up front and from then on nothing in the code is referencing the container. Here’s what I did.

I added another method to `FilterConfig` for registering global filters that were resolved from the container:

``` csharp
    public class FilterConfig
    {
        public static void RegisterGlobalFilters(GlobalFilterCollection filters) //Existing method
        {
            filters.Add(new HandleErrorAttribute());
        }

        public static void RegisterGlobalFiltersWithDependencies(HttpFilterCollection filters,
            IUnityContainer container)
        {
            filters.Add(container.Resolve<MyCustomFilterBehavior>());
        }
    }
```

Then I call that method from Global.asax in `Application_Start()`:

``` csharp
        protected void Application_Start()
        {
            var container = new UnityContainer();

            UnityConfig.RegisterComponents(container);
            GlobalConfiguration.Configure(WebApiConfig.Register);

            FilterConfig.RegisterGlobalFiltersWithDependencies(GlobalConfiguration.Configuration.Filters, container);
        }
```

What if `MyCustomFilterBehavior` has dependencies that need to be transient? Then it would still be a singleton, but it would depend on an abstract factory so that those dependencies would be resolved each time the behavior executed.

### Did I Miss Anything?

A pet peeve of mine is that technical explanations on any subject often gloss over some important detail, assuming that you already know it, or perhaps just forgetting it. As in the case of how to resolve the filter from the container, I’d rather show a less-than-perfect example than none at all, because then you have something to start from and you can change it.

But if I missed anything, send me an email, or perhaps send me an email and post a question on [stackoverflow](http://stackoverflow.com/) so that I can answer it there and other people can offer ideas.
