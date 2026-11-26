---
layout: post
title:  '.NET Event Bus - Dependency Injection Friendly and Agnostic'
date:   2016-04-01 07:20:58 -0400
category: blog
description: event bus .net dependency injection
---
Update - The source code is [available on GitHub](https://github.com/scotthannen/PlasticFloor.EventBus).

An event bus is a useful way to decouple events from handlers. Instead of handlers subscribing to numerous publishers they subscribe to a _type_ of event. The subscriber may not need to know anything about source of the event. The event itself may contain all the subscriber needs.

For example, suppose you have an application that accepts orders for products. Every time an order is placed you must

*   Log a record of the order details
*   Send notifications when certain types of products are ordered
*   Decrement inventory
*   Send a confirmation email

The class that submits the order doesn’t need to know about any of those actions. And the classes that perform those actions are interested in the order itself, not in the class that submitted it.

The domain event pattern allows any class that submits an order to call

``` csharp
        _eventBus.Raise(new OrderSubmittedEvent(order));
```

and then any number of classes that implement `IEventHander<OrderSubmittedEvent>` can receive and handle the event.

There are plenty of event bus implementations. I wanted mine to work well with a dependency injection container so that I can resolve event handlers from that container. But I also didn’t want it to be coupled to any one DI framework. Here’s my implementation. (Perhaps I should stick this on GitHub.)

## Event Bus Interfaces and Basic Implementation

First are the interfaces and a few common classes. I like these to be in a separate assembly from any DI-specific code. Then I can adapt to the DI container I’m going to use. (I prefer Windsor.)

``` csharp
        // Event bus interface for injecting into classes
        public interface IEventBus
        {
            void Raise<TEvent>(TEvent @event) where TEvent : IEvent;
            void RaiseSafely<TEvent>(TEvent @event) where TEvent : IEvent;
        }

        // Marker interface for events. Technically this isn't absolutely
        // necessary. I'm on the fence. But I like indicating that a class
        // will be used for this purpose.
        public interface IEvent
        {}

        public interface IEventHandler<in TEvent> where TEvent : IEvent
        {
            void HandleEvent(TEvent e);
        }

        // This is what will enable integration of DI containers. I can write 
        // implementations that resolve event handlers from a container.
        public interface IHandlerProvider
        {
            IEnumerable<IEventHandler<TEvent>> GetHandlers<TEvent>() where TEvent : IEvent;
            void ReleaseHandler(object handler );
        }

        public delegate void EventHandlerDelegate<in TEvent>(TEvent e) where TEvent : IEvent;

        // The only predefined event I use
        public class ExceptionEvent : IEvent
        {
            private readonly Exception _exception;

            public Exception Exception { get { return _exception; } }

            public ExceptionEvent(Exception exception, string message = null)
            {
                if(string.IsNullOrEmpty(message))
                    _exception = exception;
                else
                    _exception = new Exception(message, exception);
            }
        }

        // The actual event bus
        public class EventBus : IEventBus
        {
            private readonly Dictionary<Type, List<Delegate>> _handlers = new Dictionary<Type, List<Delegate>>();
            private readonly List<IHandlerProvider> _handlerProviders = new List<IHandlerProvider>();

            public void RegisterProvider(IHandlerProvider handlerProvider)
            {
                _handlerProviders.Add(handlerProvider);
            }

            public void Register<TEvent>(params EventHandlerDelegate<TEvent>[] handlers) where TEvent : IEvent
            {
                if (!_handlers.ContainsKey(typeof(TEvent))) _handlers.Add(typeof(TEvent), new List<Delegate>());
                foreach (var handler in handlers)
                    _handlers[typeof(TEvent)].Add(handler);
            }

            public void Handle<TEvent>(TEvent e) where TEvent : IEvent
            {
                Contract.Requires(e != null);
                HandleWithRegisteredDelegates(e);
                HandleWithHandlerProviders(e);
            }

            private void HandleWithRegisteredDelegates<TEvent>(TEvent e, bool safe=false) where TEvent : IEvent
            {
                if (_handlers.ContainsKey(typeof (TEvent)))
                    _handlers[typeof (TEvent)].ForEach(h => ExecuteHandler(((EventHandlerDelegate<TEvent>) h), e, safe));
            }

            private void HandleWithHandlerProviders<TEvent>(TEvent e, bool safe = false) where TEvent : IEvent
            {
                _handlerProviders.ForEach(handlerProvider =>
              {
                  var providedHandlers = handlerProvider.GetHandlers<TEvent>();
                  foreach (var providedHandler in providedHandlers)
                  {
                      ExecuteHandler(providedHandler.HandleEvent, e, safe);
                      handlerProvider.ReleaseHandler(providedHandler);
                  }
              });
            }

            private void ExecuteHandler<TEvent>(EventHandlerDelegate<TEvent> handler, TEvent e, bool safe) where TEvent : IEvent
            {
                if (safe)
                    try
                    {
                        handler.Invoke(e);
                    }
                    catch (Exception ex)
                    {
                        try
                        {
                            Handle(new ExceptionEvent(ex));
                        }
                        catch { }
                    }
                else
                    handler.Invoke(e);
            }

            private void Handle<TEvent>(TEvent e, bool safe) where TEvent : IEvent
            {
                Contract.Requires(e != null);
                HandleWithRegisteredDelegates(e, safe);
                HandleWithHandlerProviders(e, safe);
            }

            public void Raise<TEvent>(TEvent e) where TEvent : IEvent
            {
                Contract.Requires(e != null);
                Handle(e, false);
            }

            public void RaiseSafely<TEvent>(TEvent e) where TEvent : IEvent
            {
                Contract.Requires(e != null);
                Handle(e, true);
            }

            public bool HasHandler<TEvent>() where TEvent : IEvent
            {
                return _handlers.ContainsKey(typeof(TEvent))
                       || _handlerProviders.Any(h => h.GetHandlers<TEvent>().Any());
            }
        }
```

The `Register` method allows registering a delegate to handle an event. It’s there but I never use it. Instead I use `RegisterProvider` which allows me to add a DI-specific `IHandlerProvider`.

When an event is raised by calling `Raise` or `RaiseSafely` then the event is passed to each delegate and to each `IEventHandler` returned by the `IHandlerProvider`.

`RaiseSafely` ensures that the event bus will not raise any errors thrown by event handlers. Instead it will raise an `ExceptionEvent`. (If an exception handler throws an exception then it eats the exception. Stop the madness.)

## Windsor Facility

The Winsdsor facility resides in a separate assembly. That way you aren’t forced to reference Windsor just to use the event bus. The facility creates a single instance of `EventBus` and registers it to fulfill all requests for `IEventBus`.

It also registers its own `HandlerProvider` for the `EventBus` to use. That `HandlerProvider` uses the container to resolve any registered instances of `IEventHander<TEvent>`. This is an acceptable use of the container because it’s all done at the composition root and keeps the container hidden from classes outside the assembly.

``` csharp
        public class DomainEventFacility : AbstractFacility
        {
            private HandlerProvider _handlerProvider;
            protected override void Init()
            {
                var eventBus = new EventBus();
                Kernel.Register(
                    Component.For<IEventBus>()
                        .Instance(eventBus)
                        .LifestyleSingleton()
                    );
                _handlerProvider = new HandlerProvider(Kernel);
                eventBus.RegisterProvider(_handlerProvider);
            }
        }    

        internal class HandlerProvider : IHandlerProvider
        {
            private readonly IKernel _kernel;
            public HandlerProvider(IKernel kernel)
            {
                _kernel = kernel;
            }

            public IEnumerable<IEventHandler<TEvent>> GetHandlers<TEvent>() where TEvent : IEvent
            {
                return _kernel.ResolveAll<IEventHandler<TEvent>>();
            }

            public void ReleaseHandler(object handler)
            {
                _kernel.ReleaseComponent(handler);
            }
        }
```

This makes using `IEventBus` with the container very easy. You just add the facility as you would any other facility:

``` csharp
            _container = new WindsorContainer();
            _container.AddFacility<DomainEventFacility>();
```

and then register handlers:

``` csharp
            _container.Register(
                Component.For<IEventHandler<OrderSubmitEvent,OrderSubmitLog>(),
                Component.For<IEventHandler<OrderSubmitEvent,OrderSubmitNotifier>()
            );
```

and they are all triggered with each call to

``` csharp
        _eventBus.Raise(new OrderSubmitEvent(order));
```