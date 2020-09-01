---
layout: post
title:  "Modern Software"
date:   2019-09-16 14:21:47 +1000
categories: General 
tags:
- Attitudes
---
*Don’t use frequent software releases as a goal*

The development team wants to ship the features that create customer satisfaction, but lack the means to follow up on the success of that goal. This dilemma is a reflection of a long history of measuring work by output, not customer outcomes (ie, customer value).

 **Create a model for monitoring customer behaviors, thus providing feedback on what features are successful.**

Shift the focus away from velocity and the speed in which a feature is delivered, and focuses on the customer value a feature creates. Teams looking to embrace an outcome-driven practice should consider software tools that provide real-time build and deployment information, data and analytics around customer adoption, and built in feature flagging.  

I get really annoyed when discussing a change, someone argues that this can not be done as it will take longer than one iteration, so the work needs to be split up into smaller changes. This to me indicates that the real reason is not that we want smaller changes (which is normally a good practice) but the system architecture and processes does not lend itself to this type of work. 

## What Is a Modern System?

*Modern applications expect to have an undefined number of clients consuming the data and services it provides.*

A modern application provides an API for accessing that data and those services. The API is consistent, rather than bespoke to different clients accessing the application. 

Data is available in a generic, consumable format, such as JSON. APIs represent the objects and services in a clear, organized manner – RESTful APIs or GraphQL do a good job of providing the appropriate kind of interface.

*Select a technology stack that can support the developer to easily create functionality with an HTTP interface and clear API endpoints.*

Another principle is that it your software needs to be organised into smaller, functional modules. The smaller codebase will be easier to maintain. Enterprise‑grade application can have thousands of files and hundreds of thousands of lines of code. The interrelationships and interdependencies of the code and files may or may not be obvious. 

Building a working mental model of a larger application codebase can take a significant amount of time, and again puts a significant cognitive load on the developer.

Separating the application into smaller, functional modules, independent of other modules would reduce this burden as it limits the scope of impact to that module. 

*Need to separate the system into domains. How do you do that without creating more complexity.*

*The biggest bottleneck to rapid development is often not the architecture or your development process, but how much time your engineers spend focusing on the business logic of the feature they are working on.*

To get the best work out of your team, it is critical that your application ecosystem focuses on the following:

* Systems and processes that are easy to work with
* Architecture and code that are easy to understand
* DevOps support for managing tooling

An example of an **easy-to-work-with development environment:**

* A developer clones a GitHub repo
* He or she runs a couple of commands from a makefile
* Tests run
* The application comes up and is accessible

In contrast, development environments that require significant effort to get all the components up and running, including setting up systems like databases, support services, infrastructure components, and application engines, are significant barriers to productivity.