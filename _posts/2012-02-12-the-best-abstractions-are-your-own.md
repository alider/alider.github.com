---
layout: post
title: "The best abstractions are your own"
description: ""
category: 
tags: []
---

Let me tell you the story of one particular project from early 00's. It's not a project I was involved in but i was told so many things about it that I feel like I was there. That time frameworks weren't very popular. There were no Spring, Hibernate, GWT, JSF, really nothing except EJB 2.x and plain JSP/Servlets. The team had to implement business application with the requirements similar for CMS. They started implementing it and somehow found out that they need kind of framework. So instead of doing what they ware paid for - implementing requirements - they focused on something which at the end was quite similar to Struts 1.x (released a few months later). After many months of hard work on this brilliant, abstract framework for the application they realized that it's just too late for the product. It was a big failure for the project and whole team.

What's the lesson? Don't build your own framework? It's much easier to say it now when we have a lot of frameworks. This lesson has been learned already and it's not a problem in these days - a least I haven't seen such case recently. But what I can see quite often is frameworks mis/overuse. Taking the application from the story I can easily think of at least 3 frameworks used in some typical implementation. Is it bad? Not necessarily but can be.

**Not only frameworks**

Generally abstractions are about hiding details of some complexity behind a interface which is simpler and closer to the problem we have. So that having OO model and the requirement to make it persistent in e.g. MySQL we aren't forced to solve ORM mismatch ourself and we can use some existing library like Hibernate. Abstractions are around us and we use them a lot e.g. TCP/IP for networking, streams for disk data. Why do we use them? Because they speed up our work making it easier and more standarized. Frameworks are just one type of abstractions.

**The problem**

So, where is the problem? First of all: <a href="http://www.joelonsoftware.com/articles/LeakyAbstractions.html"><i>All non-trivial abstractions, to some degree, are leaky.</i></a> - Joel Spolsky.

In many Java server-side applications we use Hibernate for the ORM problem. This is quite good abstraction but it's also typical for non-experienced team to get performance problems really soon. Why? Mostly because the tool was learned and used as black box only through its specification/api without monitoring what, when and how many quires are generated behind the scene. Spring Framework provides good mechanism for doing declarative transactions but here also knowing only the annotations which should be applied can cause problems with poor performance or even with lost data. It's not only related to frameworks and libs. Scala is getting very popular recently. There are many tutorials and talks showing how nice the operations on collections can be made. But at some time we can be punished by Garbage Collector if we overuse this style (a lot of temporal objects and memory allocations). 

**Summary**

We should use abstractions (frameworks, libraries, new languages) but to be really proficient and safe with them we need to know them almost like theirs creators. We should not use them as black boxes denying the details. We should know all side effects if they are. We should look into source code if we have. We should own them. If we are not able to own all abstractions used in our project then we should limit theirs number.
