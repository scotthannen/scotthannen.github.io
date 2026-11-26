---
layout: post
title:  "Hexagonal/Ports and Adapters Architecture Part I"
date:   2024-07-29 -0400
category: blog
description: Part one of an explanation of Hexagonal architecture. This is geared toward people who've seen all the diagrams but are still confused.
---
Hexagonal architecture is not a complex concept to understand, but that's only true once we understand it. I find it difficult to explain. You can draw a diagram, but what does the code look like in a language you're familiar with? You can illustrate it using code, but the architecture isn't about how we write the code. Someone might look at the code and think that the architecture means writing code that looks like the example.  

It also has terminology which is very useful when discussing pieces within the architecture, but might not initially help us understand the concepts. Someone might see a bunch of words used in unfamiliar words and say, "That's it, I'm out."  

This post is a stab at making it easier to understand. We all learn differently. Maybe this will help someone who didn't get it before.  

I'm going to leave out the diagram. I'll also try to lay off the terminology. After explaining concepts I'll introduce the names we call them.

### Brief Overview

Hexagonal architecture was <a href="https://alistair.cockburn.us/hexagonal-architecture/" target="_blank">described by Alistair Cockburn in 2005</a>. Later he used the name "Ports and Adapters" because it's more descriptive. There is no hexagon. There's not six of anything. It was the shape of the diagram. "Hexagonal architecture" is still a valid name, and I'll use that throughout because it's shorter.  

Some benefits:

-   We can design and test an application's core behavior without having to go through the UI, setting up data in a database, or connecting one service to other services. Tests that do those things have value, but we shouldn't need them just to test logic.    
-   We can replace one technology with another more easily. Some might ask whether that actually happens. It certainly does. During initial development we might use a document database because it's fast and easy to change, and later we might see the need for SQL. Today we do something when we receive an HTTP request. Tomorrow we might want to do the same thing in response to a message. That's easier if the behavior isn't tightly coupled to an HTTP endpoint.    
-   The architecture helps us to avoid database-centric design and thinking. It shifts focus to the application's uses and behaviors. Storing data in a database may be a necessity, but it's not the reason why the application exists. The database is just something the application needs.    

### Inside and Outside

We can think of Hexagonal architecture as defining two areas - the inside and the outside. Picture a hexagon, a circle, whatever works for you.  

Inside is code that describes what the application does and how it behaves. Inside is logic and behavior. There is nothing technology-specific on the inside.

-   No databases    
-   No messages    
-   No inputs and outputs defined by other systems, like the DTOs they use for HTTP requests and responses    
-   No UI  

None of those things are mentioned on the inside. All of those technology-specific details are on the outside.  

An important principle is that the parts on the outside know a little bit about the inside. They exist to make the parts on the inside work. But the inside knows nothing about the outside.  

### What Goes Inside?

The inside might not know about the outside, but in order for the application to do anything, all the pieces need to work together. 
That means that the outside must know about the inside. The inside must make certain parts visible so
that the outside can interact with them.

To that end, the inside contains interfaces that describe its interaction with the outside world. There are two types of interfaces:

-   Things that you can tell the application to do. Think of these as inputs.    
-   Things that the application needs. It might need a repository that provides data it needs, or where it can store data. It might need to send out notifications when things happen. In both cases it knows that those things exist, but nothing about how they are implemented. Those interfaces are abstract.  

Don't be thrown off by the word "interface." It does not refer to the literal interface keyword in some languages, although it might include such interfaces. They are ultimately methods, or things with methods.

-   Things on the outside call methods to convey input to the application, telling it to do something. Often an application has a set of commands, and we provide input by passing a command to a method that handles it.    
-   Things on the inside call interface methods to interact with the outside.  
    -   Give me this data    
    -   Save this data    
    -   Send a notification    

At this point a concrete example might help. Suppose your application is a blog website, and one of the operations is to add a blog post. When a blog post is added, you want to
-   Verify that there's not already a blog post with the same title    
-   Save the new blog post    
-   Raise a notification or event indicating that a new post has been added.  

What might that look like? 

First, there's going to be some sort of interface that allows callers to tell the application. "I want to add a blog post."  

That could look like  

```csharp
public interface IBlogService
{
    void AddBlogPost(BlogPost post);
}
```
 ...or

  
```csharp
public class AddBlogPostCommand : ICommand
{
    public string Title { get; set; }
    public string Content { get; set; }
    public Guid AuthorId { get; set; }
    public string[] Tags { get; set; }
}
```
...and something that accepts the command, like  

```csharp
public interface ICommandHandler
{
    void Handle(ICommand command);
}
```
Remember, don't take these as concrete examples. Hexagonal architecture does not tell us how to write this code. The point is that the application provides some programmatic way to tell it to do something.  

Then, in order to do its work, the application needs more interfaces describing the things it must interact with. For example:  

```csharp
public interface IBlogRepository
{
    BlogPost[] GetAllBlogs();
    void SaveBlogPost(BlogPost blogPost);
}

public interface INotificationServive
{
    void SendNotification(INotification notification);
}
```
Again, these are examples, not specifications for how we should write the code.  

This allows us to write something like:  

```csharp
public void Handle(AddBlogPost command)
{
    var blogsWithSameTitle =
         _blogRepository.GetBlogs().Where(blog => blog.Title == command.Title);

    if(blogsWithSameTitle.Length > 0)
    {
        throw new Exception("There is already a blog with that title!");
    }
    var newPost = new BlogPost(
        Command.Title, command.Content, command.AuthorId, command.Tags);
    _blogRepository.SaveBlogPost(newPost);
    var notification = new BlogPostAddedNotification(newPost.Id);
    _notificationService.SendNotification(notification);
}  
```
The behavior of the application is described apart from any technology.

-   Where does the command come from? A UI? An HTTP request? It's not specified. This code doesn't know and doesn't care.
-   Where is the data? In a database? A file? Something else? Not specified. 
-   Where does the notification go? An email? A message? Both? Not specified.

It's easy to write tests for this because we don't have to send HTTP requests or set up a database with test data. We don't have to check a message queue to make sure a notification was sent.  

What's more, suppose we're working on the logic of the application but we haven't even decided what sort of database we want to use? That's okay. We don't need to answer that right now. It's helpful to be able to delay those decisions or to change our minds as we learn more about the needs of the software.  

If that makes sense then you understand most of Hexagonal/Ports and Adapters architecture.  

This is a good time to introduce some terminology.  

#### Application
The application is everything on the inside, everything that isn't technology-specific. That's confusing because really the application is everything from the UI to the database. But it helps to know what someone means when they use the term in the context of the architecture.  

#### Ports
Ports are the interfaces that the application exposes to the outside world. Ports are part of the inside of the application.

There are two kinds of ports:
-   *Driving ports* are the interfaces used to tell the application what to do.  
    They are also called *provided interfaces*. The application provides them to consumers.    
-   *Driven ports* are the interfaces the application uses to do its job.  
    They are also called *required interfaces*. The application requires them to do its job.  

Now that I've explained ports in a simple way, I'm going to change that definition to be more accurate.

Ports are not the individual methods. Driving ports, in particular, are groups of methods provided for related use cases. For example, one port might be the methods used by people who create and modify blogs. Another port might be the methods used by automated services to perform some sort of maintenance.  

That's not too confusing, but maybe it's one piece of information too many right now. It's okay to back up and leave that for later. The distinction matters, but if everything else is mostly clear then our understanding is good enough for now.  

One detail I want to call attention to: When we look at an application's ports, we see all of the operations it is possible for someone to perform with that application. That's helpful for two reasons:

-   If we want to understand or change what happens when we add a blog post, we know where to start looking.    
-   This helps us to encapsulate our application's data. The only way to interact with the application is through its ports. Unless we deliberately expose a port that says, "Update my data however you want," then we have control over how our data is modified.   

----------

 So far we've talked about the inside of the application. What comes next? The adapters - the parts on the outside which enable our application to connect to the outside world - to receive requests from actual users, save data, etc.  That's what the next post will cover.

 For this to really make sense you might need to see a code example. If you want to see that right now, Google "hexagonal architecture examples in code" and specify your preferred language. Remember to look for the parts that correspond to what you've learned so far, but don't get hung up on the specifics. That's where confusion can set in. You can't show someone code without writing code, and that inevitably means the code will be written in some specific manner. There's no getting around that. But remember that those details are examples and do not define the architecture.  

Eventually I'll write a code sample myself in C#.

<a href="/blog/2024/08/06/hexagonal-ports-and-adapters-architecture-ii.html">Here's part two</a>.