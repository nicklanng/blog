+++
author = "Nick Lanng"
categories = ["architecture"]
date = 2014-02-11T23:27:22Z
description = ""
draft = false
slug = "service-oriented-hackathon"
tags = ["architecture"]
title = "Service-oriented Hackathon"

+++

<p class="intro">Last week my company held its quarterly hackathon event. A hackathon is a time for developers to try out some of those things they think would be a great idea but which may be difficult to get on to the official roadmap. As all of our products live in the cloud, my team's project was to spike a distributed service-oriented architecture (or SOA), similar to the way Netflix operates.</p>

## Why use a service-oriented architecture?
This past year, my team have been working on a greenfield system which represents a brand new product line for my company. We've built a nicely structured ASP.Net MVC application, but as it all gets compiled into the same distributable we've found a tendency for the separate sections of the site to become coupled - this has led to some pretty nasty refactor stories that always seem to take longer than we estimate.

A small section of our application is asynchronous processing, we made sure to separate that out from the rest of the application and we coordinate the whole thing with a small bespoke queue system. We are keen to move to a more proven queue system and that got us thinking about how we might have designed the application if we had a decent queue system from the beginning. Would we have built such a coupled stack or could we instead have lots of services all with a single responsibility and some common interface between them. Refactoring would certainly be a lot easier if the heap of code was smaller and any potential system breaks due to massive overhauls could be restricted to a small subsystem of the application.

I think that's really the thing that I find interesting about architecture. Principles that are normally used to inform code can often be used to inform system architectures. In this case, the Single Responsibility Principle. Separate components can be scaled, refactored and can break all independently from the rest of the system.

My understanding of the history and arguments for service-oriented architectures is very limited but this is a subject I will endeavour to learn more about.

## How did we do it?
We set ourselves a few constraints before we started our work.

* Everything must run on Linux. We're primarily a C# house but running Windows servers in the cloud can tend to limit tool support. We decided to try using [Mono](http://www.mono-project.com/), we had heard great things about how far the language had come and wanted to see for ourselves what it could do.

* The project must be built as if it were a real live system. We wanted to prove how quickly we could get a functional system up and running while still considering factors such as scalability and redundancy. We chose to use [Amazon Web Services](http://aws.amazon.com/) to host the solution, from the team's small amount of experience with AWS it seemed to be a really powerful toolbox for creating cloud systems.

* Although we are all highly experienced in ASP.Net MVC, we decided to try a more lightweight framework to learn which features of MVC we actually need and use. Recently, we've really started to question all of our framework choices; do we use them because they really are the best tool for the job or just because we are used to working with them? [Nancy](http://nancyfx.org/) was the framework we chose to use because we liked it's simplicity and some of us have had good times with the project it was based on, [Sinatra](http://www.sinatrarb.com/).

* We wanted real-time features in our website so this led us to using websockets. We discussed a few websockets frameworks and the two final contenders were [SignalR](http://signalr.net/) and [Socket.io](http://socket.io/). Both of them are really powerful frameworks but we decided to choose Socket.io, primarily because it was a simpler offering with less of a pre-built architecture. This would mean that a subsystem of our site would need to be built with Node.js but, since everything is distributed, this subsystem should be relatively standalone.

* The service should be pulled together with an event system, an example of this would be: a new telephony event is fired into the message queue, the event is then picked up by a service to store the persistent record and another service informs the user of the call. Using AWS would naturally lead us to use two of their offered services, Simple Queue Service and Simple Notification Service, but, wary of tying a solution too closely to AWS and wanting to dig a bit deeper into how a message queue fits together, we decided to go with [RabbitMQ](https://www.rabbitmq.com/), a very popular and widely used messaging system.

## What did we learn?

We built the solution in Visual Studio 2013 with all the tooling support (Resharper, NCrunch etc...) and made sure it ran on our local development machines. When we came to compile the system on Linux we expected to have to deal with a multitude of errors but there was nothing.

It just worked.

In Mono, *msbuild* is replaced with *xbuild*, all of your msbuild scripts will be supported by xbuild. You can also install Mono for windows which gives you the same tools you would use on Linux. You can even compile using Mono for Windows and the same executable will work on Linux. I am so impressed with how simple the whole thing was. We weren't using terribly complex language features and we built everything against .Net 4 client profile to be extra sure of compatibility, but I am certainly confident enough to suggest using Mono in our live systems.

A message queue seems to be a great way to decouple your system's subsystems. After we had decided on the next event to support we found it very easy to split off into small teams to support the event in existing and new services. A lot of care and consideration has to be taken when defining your messaging language, as it has the ability to cause rewrites throughout your system (*or at least those subsystems which care about a particular message type*).

Due to our subsystems being so small and self-contained they ended up only having a few small classes. The main benefits from this are ease of understanding the codebase for new developers and that you are only very thinly bound to any technology choices for that subsystem. We also had one subsystem fail during our demo but, as nothing else was relying on that subsystem and only a very small section of the site needed it, the rest of the site carried on functioning perfectly.

Another thing we noticed was the ability to deploy subsystems individually. You have to be careful about running different versions of the site, specifically subsystems built against different version of the messaging contracts, but it does help when deploying manually and patching small parts of the site quickly.

We found Nancy to be very easy to work with and fast to get up and running. The documentation on hosting Nancy with Nginx on Linux was perfect and I managed to set up the hosting in about 30 minutes. We didn't try to implement authentication or any other feature in our small demo so we still have a lot to learn about how feasible Nancy is to use in a live environment. One thing for sure is how quickly we'll be able to spike these concerns.

Hosting the system on AWS was a great idea. I was blown away at how quickly we had a scalable, load-balanced system up and running and completely secure, too! The only two routes in to the entire system was through the HTTP load balancer and the websocket server. I am told that Amazon are currently working on load balancing websocket traffic so that should soon make the whole thing a lot neater. Configuring new instances and setting up security groups on AWS is so simple, I would highly recommend you to have a play with it. They offer a pretty decent trial, too!

Unfortunately, due to time constraints, we didn't have any time to automate our deployments. This is something I'd like to go back and work out, as they say that if you ever need to SSH in to one of your instances then you are doing it wrong.

We didn't have time to configure RabbitMQ to be highly available, which had a knock-on effect of making the other subsystems that rely on it a single point of failure. In fact, we only managed to make the web servers highly available and that was due to using AWS's elastic load balancers. Obviously, we could have made the rest of the system highly available if we had the time to learn how, but if we had used AWS's message queue offerings, which are highly available with zero configuration, then we could have made the entire system much more robust with much less time spent. I think this is an important consideration for a live system, if getting to market as soon as possible is the primary concern then consider using AWS's services while building the system, but be sure to put a strong interface between the services and your application so that you don't couple your system with AWS. One day you may decide to move hosts or build your own data centre.

<hr/>

Using the right tools for the job can make development a lot faster than you may be used to. New technology and architectures are not always the silver bullet you are looking for but they can still be effective at killing a few old dragons. (*I believe mixed metaphors are often the most effective.*)
