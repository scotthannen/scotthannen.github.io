---
layout: post
title:  'Adapter Pattern'
date:   2016-02-25 07:20:58 -0400
category: blog
---
There are all sorts of frameworks for logging exceptions and other data. I even created one of my own several years ago. It buffers messages, sends batches via MSMQ that are read by a Windows service, and then writes them to SQL tables.

It works great, but it’s also the very definition of “legacy code.” It uses all sorts of static objects and singletons and methods with strange, badly-named parameters. It’s got so much coupling that I can’t use the back end without also using the same client code.

I don’t want anything I develop to have a direct dependency on this legacy code.* But I really don’t want it to depend on any other framework either.

This isn’t suitable:

``` csharp
    public class FooProvider
    {
        public IEnumerable<Foo> GetFoos()
        {
            try
            {
                //Create the Foo list
            }
            catch (Exception ex)
            {
                ScottsFunkyLogger.LogLoggableObject(new ApplicationEvent(
                    ex.ToString(), Guid.NewGuid(), EventLogEntryType.Error));
                return new Foo[] {};
            }
        }
    }
```

Neither is this, because this means my class can never be used without Castle Windsor.

``` csharp
    public class FooProvider
    {
        private readonly Castle.Core.Logging.ILogger _logger;

        public FooProvider(Castle.Core.Logging.ILogger logger)
        {
            _logger = logger;
        }

        public IEnumerable<Foo> GetFoos()
        {
            try
            {
                //Create the Foo list
            }
            catch (Exception ex)
            {
                _logger.Error(ex.ToString());
                return new Foo[] {};
            }
        }
    }
```

Side note: This is just for illustration. In most cases I would use an interceptor and wouldn’t have exception logging in my class at all. But let’s say in this case I want to handle the exception gracefully so I need to log it. And even if I log from an interceptor I have to log to something.

The Adapter pattern allows me to use either of these without creating an unwanted dependency or putting embarrassing old legacy code in my new class. I can create the interface I want and then adapt other classes to it.

First, I’ll create a new interface in a separate library:

``` csharp
    public interface ILogger
    {
        void LogException(Exception ex);
        void LogMessage(string message);
    }
```

That library should be minimal - just a few interfaces, no implementations. That’s because everything else I develop is going to reference that interface. I want someone to be able to reference my class library and the interface library without having to add all of the baggage of other packages and libraries that I don’t want or need. Then they require specific versions of other packages that conflict with what I have, and it ties a knot.

Then I can create class libraries that adapt these logging frameworks (or others) to my interface. I still have the baggage of my old logging framework, but it’s in that library. Within that library you might find this adapter class that hides my legacy code behind the new interface.

``` csharp
    public class ScottsFunkyLegacyLogger : ILogger
    {
        public void LogException(Exception ex)
        {
            ScottsFunkyLogger.LogLoggableObject(new ApplicationEvent(
                ex.ToString(), Guid.NewGuid(), EventLogEntryType.Error));
        }

        public void LogMessage(string message)
        {
            ScottsFunkyLogger.LogLoggableObject(new ApplicationEvent(
                message, Guid.NewGuid(), EventLogEntryType.Information));
        }
    }
```

Here’s Castle’s `ILogger` adapted to our interface:

``` csharp
    public class CastleLogger : ILogger
    {
        private readonly Castle.Core.Logging.ILogger _innerCastleLogger;

        public CastleLogger(Castle.Core.Logging.ILogger innerCastleLogger)
        {
            _innerCastleLogger = innerCastleLogger;
        }
        public void LogException(Exception ex)
        {
            _innerCastleLogger.Error(ex.ToString());
        }

        public void LogMessage(string message)
        {
            _innerCastleLogger.Info(message);
        }
    }
```

Now my `FooProvider` looks like this:

``` csharp
    public class FooProvider
    {
        private readonly MyInterfaceLibrary.ILogger _logger;

        public MyClass(MyInterfaceLibrary.ILogger logger)
        {
            _logger = logger;
        }

        public IEnumerable<Foo> GetFoos()
        {
            try
            {
                //Create the Foo list
            }
            catch (Exception ex)
            {
                _logger.Error(ex.ToString());
                return new Foo[] {};
            }
        }
    }
```

Now I can use my ugly legacy code while effectively keeping it out of my new code. I also have the freedom to change my mind, and it’s very easy to adapt any logging framework to that interface. There could be two different applications using the same `FooProvider` with completely different logging frameworks.

*Legacy Code

When I have ungainly legacy code like my old logging framework I think of it as a crazy, embarrassing uncle. You don’t want him interacting with anyone but you can’t get rid of him. So you make a nice space for him in the attic, lock him in, and put a slot in the door so you can pass in food and he can pass notes for other people that you can read first and redact. That way you completely control his interaction with the outside world, and you never even open the door unless he requires urgent medical assistance.