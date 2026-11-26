---
layout: post
title:  "Hexagonal/Ports and Adapters Architecture Part II"
date:   2024-08-06 -0400
category: blog
description: Part two of an explanation of Hexagonal architecture. This is geared toward people who've seen all the diagrams but are still confused.
---
### Summary of Part I
The <a target="_blank" href="https://scotthannen.org/blog/2024/07/29/hexagonal-ports-and-adapters-architecture-i.html">previous post</a> introduced some basics of Ports and Adapters architecture.
- There's an *inside* which contains application logic independent of technology like UI frameworks, databases, external HTTP endpoints, or messaging systems. The inside contains no references of any kind to any of those details, which are on the outside. We call the inside the *application*.
- The inside exposes some sort of methods or interfaces used to interact with it. Those are called *provided interfaces*. (*Interface* does not mean the `interface` keyword in many languages, although we might use that.)
- The inside also exposes interfaces that describe what it requires in order to function, like repositories. These are abstract and don't describe any particular technology. They are called *required interfaces*.
- Together, we call these external-facing interfaces *ports*. More specifically, a port is a group of related interfaces. That distinction matters, but if it's unclear then set it aside for now and come back to it.
- The ports used to make application do things are *driving ports*. The ports the application requires in order to function are *driven ports*. 

I prefer not to overemphasize the terminology. We all learn differently. It helped me to understand concepts first. Then putting names with them "anchors" them. 

### How Do We Talk to the Application?
Everything on the inside of the application is abstract. There's no UI and no database. It wouldn't be useful or usable if we left it that way. How do we add those technology-specific details so we can actually use the application?

The missing pieces are adapters. These include things that interact with the application's ports, telling it to do things. For example:

- A UI where a user submits input or presses buttons
- An HTTP endpoint that receives requests
- A listener that receives messages
- A timer that acts at intervals or scheduled times

Each of these starts with something that the inner application doesn't know about - button presses, HTTP request models, events, and so on. Then they adapt those by calling methods exposed through the application's ports. That might mean calling a method on an interface, sending a command to a command handler, or raising an event. 

As mentioned in the previous post, the outside knows about the inside, because it must adapt the input it receives to something that the inner application can understand. The inside remains unaware of the outside.

Here's an example in C#. Our application - the "inner" application - exposes an interface that accepts certain commands. That interface is a port. An HTTP endpoint accepts a request and adapts it to that port.

```csharp
[HttpPost]
public async Task<IActionResult> CancelOrder(CancelOrderRequest request)
{
    // The inner application doesn't know what CancelOrderRequest is.
    // Map it to a command defined within the inner application.
    CancelOrderCommand command = request.ToCommand();
    
    // The application exposes some command handler interface.
    await _commandHandler.Handle(command);
    return Ok(); 
}
```
This bears repeating because it's where the name of the architecture - "Ports and Adapters" comes in. 

- The application exposes a *port*, which is some technology-agnostic method consumers can call.
- In the real world we need technology-specific ways to call those methods. *Adapters* are technology-specific. They provide a way to adapt an external input to a port.

That's how we tell the application what to do. But what about the things the application needs in order do it, like a database, or somewhere to send notifications?

### How Does the Application Talk to Its Database and Other External Resources

The answer is that we use adapters for that too. Remember, the application exposes *required interfaces*, which describe external things it expects to interact with. It knows nothing about the implementation. We meet that requirement by writing code which implements those interfaces, adapting them to some concrete implementation.

As an example, suppose our blog application requires a repository interface. It exposes this:

```csharp
public interface IBlogRepository
{
    BlogPost[] GetAllBlogPosts();
    void SaveBlogPost(BlogPost blogPost);
}
```

We might look at that and assume that it represents a database, but notice that nothing in the interface specifies that. The implementation could also be something like this:

```csharp
public class FileSystemBlogRepository : IBlogRepository
{
    public BlogPost[] GetAllBlogPosts()
    {
       // Code that reads files and maps their contents
       // to a collection of BlogPost objects       
    }
    
    public void SaveBlogPost(BlogPost blogPost)
    {
        // Code that converts the BlogPost to some file format
        // and saves it
    }
}
```

Remember the HTTP endpoint? It accepts some request object that the application doesn't know about, and maps it to a command the application understands. The application doesn't know about those details.

The same is true here. The application only knows about the `IBlogRepository` interface and its own objects like `BlogPost`. If that must be mapped to and from something else like a relational SQL tables or files, that happens in the adapter. The outside knows about the inside. The inside doesn't know about the implementation details on the outside.

That keeps a lot of clutter out of the inside. Have you worked in an application that has numerous models for the same thing, like `BlogPost`, `BlogPostModel`, `BlogPostDto`, `BlogPostData`? At first you can't tell what they're used for, and eventually you memorize what each one means in the context of this one application. (The naming conventions are different in other applications because they're meaningless.) That happens because either there is no inside and outside, or we bring all the classes from the outside to the inside, where they all mix together. Hexagonal architecture prevents that by keeping models used by the adapters on the outside.

### Abstractions, Not Interfaces
A common mistake is for the inside to define interfaces that aren't abstract. For example, if we need data that comes from an external Web API, we might define an interface like

```csharp
public interface IAuthorApi
{
    AuthorResponse GetAuthor(Guid authorId);
}
```
Can you see how this goes against the flow of the architecture?

 1. The interface isn't abstract. It describes a specific implementation.
 2. It returns the model used by that Web API.
 3. If it fails we're likely to get an exception or response with an HTTP status code. If the inner application must check for that, it's no longer technology-agnostic. 

Instead, we should define something truly abstract, like

```csharp
public interface IAuthorRepository
// or IAuthorData - just not IAuthorApi
{
    Author GetAuthor(Guid authorId);
}
```

This assumes that we're using an `interface` with a class implementation. That's typical, but it's a feature of certain languages. It's not a requirement of Hexagonal architecture. The point is that we should avoid accidentally bringing technology-specific details into the inner application. The required interfaces on the inside should be abstract. Technology-specific details belong in the adapters.

---

### Some More Terminology
Like before, I'm saving terminology for last. Why? Because understanding the concepts is more important than knowing names for them. If we can apply the concepts then it doesn't matter as much whether we remember the terms. The terms are helpful, though, because they cement the concepts in our minds and enable us to communicate. There's plenty of content surrounding this architecture, and it's easier to understand if we speak the language.

With that, here are some more terms. Not all are specific to Ports and Adapters architecture.

- A *primary actor* is whatever initiates action. That could be the UI, message listener, timer, etc. The adapters they interact with are called *driving adapters.* They drive. They're active. They make things happen.
- A *secondary actor* is something that acts because the application tells it to. That could be the database or outgoing message system. The adapters for those are called *driven* adapters. They don't initiate anything. The inner application tells them what to do - give me this data, save this data, send this message.

### That's It?
At this point you might wonder if that's all there is to it. We haven't gotten into code yet, but I hope you'll agree that the concepts are not rocket science. Then why so many words to explain it? Because while it's easy to understand, it's oddly challenging to communicate. Try it and you'll find out. 

It's not difficult to explain because people are incapable of understanding it. It's because we all have our own mental models, and we're thinking in terms of the code we're used to seeing. Once we understand the concepts, it might take more effort to map them to our preferred technology stack and whatever type of software we write.

If you've followed along and got the concepts and even a few of the terms, now is a good time to Google and find other material. My hope is that if none of it quite made sense to you before, maybe it will now. Or if you understand it and you've struggled to explain it to someone else, maybe this post will help. We all learn differently, so there's no telling which explanation will help any given person the most.

Also, Google "hexagonal architecture example in (whatever language you prefer.)" There are numerous concrete examples on GitHub. As you look at them, see if you can recognize how they implement the architecture. They will all be very different. Look for the patterns. See if you can spot things that don't make sense, and think about what you might do differently and why. Remember that implementing the architecture does not mean that you must write code as it appears in any example.

### What's Next?
One aspect of the architecture I haven't touched on is: How do we combine the inner application with the adapters to create fully functional software with a UI, database, etc. that we can deploy? I haven't gone into that because the answer is specific to the tech stack we use, and even then there are numerous ways to go about it. Some of the existing code samples on GitHub will help. I will eventually follow up with a .NET-specific example.

The architecture and its terminology describe how it fits together. That part isn't left out. I find it confusing to describe in abstract terms. If you're interested, check out the section entitled The Configurable Dependency Pattern in <a target="_blank" href="https://jmgarridopaz.github.io/content/hexagonalarchitecture.html#tc3">this article</a> by the late Juan Manuel Garrido de Paz who sadly passed away shortly after co-authoring the book <a target="_blank" href="https://www.amazon.com/Hexagonal-Architecture-Explained-Alistair-Cockburn-ebook">Hexagonal Architecture Explained</a> with Alistair Cockburn. Actually, read the whole article. Read the book.

Also, when should we use this architecture? When should we not? 

I welcome any feedback on what helped or what didn't. <a href="mailto:scotthannen@gmail.com">scotthannen@gmail.com</a>.