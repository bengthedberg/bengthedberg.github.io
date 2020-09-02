---
layout: post
title:  "Requirements and Use Cases"
date:   2019-12-02 10:11:22 +1000
categories: Design 
tags:
- Requirement
- Analysis
---
An example of how you take a set of requirements and create use cases from them, list dependencies, order them and pull out the MVP.

Here is an example of a set of requirements, this would normally be the result from interviewing subject matter experts and core stake holders:

> *TechMeet is a social network that makes it easy for people to track events covering their favourite tech stack.* 
>
> *Organiser can sign up and list their events. When adding an event, they should specify the date and time, location, and type of technology.*
>
> *An organiser should have a page called My Upcoming Events. From there, they should be able to edit or remove an existing event or add another one to the list.* 
>
> *Users should be able to view all upcoming events or search them by organiser, type, or location. They should be able to view the details of an event and add it to their calendar.* 
>
> *Additionally, users should be able to follow their favourite organiser. When they follow an organiser, they should see the upcoming events of their favourite organisers in the Event Feed.* 

These requirements are simple enough not to get distracted by the domain, but deep enough to provide a good foundation for this example.

**Tip**: The importance at this stage is to get a big picture of what the system should implement. Do not overthink the implementation by going into too many details. These details can come later as they would most likely change when the development progress.

### Create Use Cases

Next, we should create the uses cases based on the high-level features in the previous requirement set. 

Tip: Express the use cases **using only a few words** that are **copied from the requirement itself**. For example, in the second paragraph we have two use cases, sign up and add an event.

Use cases with less words, using the terminology in the requirements, makes it easier to communicate with other team members or the stake holders. We do not have to go through pages and pages of detailed requirements document or try and interpret meaning.  **The idea is to form a basic understanding of high-level features**. Details can come later when we get closer to implementation. 

Lets first describe what the system is to deliver by defining its goal:

**System Description: ** *"TechMeet is a social network that makes it easy for people to track events covering their favourite tech stack.”*

This may look a bit contrived as its just a copy from the first paragraph, though the point here is to have a statement that describes the purpose of this system. It will be used when prioritising the use cases based on the systems core functionality, also referred to as the MVP (Minimal Value Product).

Extracting the requirements into use cases can be done in different ways. I prefer to start with the actors, meaning the people or other systems that interact with this system, and what way they use the system. 

In our case we have *organisers* and *users* as actors, so we underline them to highlight these. Then we mark any actions these actors perform as well as screens, reports or anything else that is specifically mention in the requirement. The following is my attempt:  

> <u>Organiser</u> can **sign up** and **list their events**. When adding an event, they should specify the date and time, location, and type of technology.
>
> An <u>organiser</u> should have a page called **My Upcoming Events**. From there, they should be able to **edit** or **remove** an existing event or **add** another one to the list.
>
> <u>Users</u> should be able to **view all upcoming events** or **search** them by organiser, type, or location. They should be able to **view the details** of an event and **add it to their calendar.**
>
> Additionally, <u>users</u> should be able to **follow their favourite organiser**. When they follow an organiser, they should **see the upcoming events** of their favourite organisers in the **Event Feed**.

Once we dissect the requirement in this way, we can hopefully find some common system entities. These are the ones I came up with: 

- Events
  - Add an Event
  - My Upcoming Events (for organiser)
  - Edit an Event
  - Remove an Event
  - List All Upcoming Events
  - Search
  - View Details
- Event Calendar
  - Add Event to Calendar
  - Remove Event From Calendar
  - View Events I’m Attending
- Following
  - Follow an organiser
  - Unfollow an organiser
  - Who I Follow
  - Event Feed

The **sign-up** action indicates that we will need authentication with organiser registering their details. This is usually common use cases that we group together under authentication:

- Authentication 
  - Sign Up
  - Login
  - Logout
  - Change Password
  - Edit Profile


### Dependency Between Use Cases

So, we have extracted the use cases and entered them into our backlog. 

In order to identify the core use cases that affect the architecture of our application, we need to work out their dependency first. 

For example, we cannot unfollow an organiser without following them first. 

Also, in order to follow an organiser, we need to view the list of events or the details of an event, because that is where we are going to display the name of an organiser. So next to each organiser, we can have a button to follow them. 

Based on these dependencies, the order in which we need to implement these use cases is like this: 

Implement view events first, then follow an organiser, and finally unfollow an organiser. 

Don't worry about the use cases related to authentication. Just focus on the core of the application. To keep things simple, assume that each use case is dependent only on, maximum, one other use case, not many use cases. 

Once you work out the dependency between all use cases, start with the ones that have no dependency to any other use cases. Put a number next to them. I call this number, Order. Once you determine the order for each use case, organize them in a layout like this. 

![image-20200902133107025](/assets/req1)

### Order the Use Cases on Dependencies

My first column has the use case **Add an Event** as this is the most fundamental use case. If our system doesn't have the ability to add an event, then there is nothing a user can do. They cannot browse or search them, neither can they add them to their calendar. All other use cases are dependent on this one.

Once we implement add an event, we can implement view **My Upcoming Events**. Add it to the second column. This is going to be the page where an organiser can see all the events they have added. 

And from there they can **edit or remove an event** so we will add that into the third column. 

Also, once add an event is implemented, we can **display all upcoming events** on the home page. Add that to the second column as well. On this page, we can give the user the ability to **add an event to their calendar** so add that to the third column.

Then we can create a page where they can **view events they are attending** and **remove them from their calendar**. 

In the list of upcoming events, next to each organiser you are going to put a button **to follow that organiser.** 

Subsequently, we will create a page for the user to see **who they are following,** and on that page, they will be able to **unfollow an organiser**.

Also, when they follow an organiser, they should be able to **view their events** in the **event feed**. 

And finally, when we implement view all upcoming events, we can give the user the ability to **search for events** or click on an **event** to view its **details**.

**Use Cases Ordered by Dependencies**

![image-20200902133211315](/assets/req2)

### Extracting the Core Use Cases

Now it's time to identify the core use cases. These are the **use cases that shape the domain of our application**. They often involve **capturing data and change the state of the application**. So, from this list, which use cases do you think are the core use cases? 

Adding an event changes the state of our application. That is a core use case. 

In the second column, we don't have any use cases that change the state of the application. They all are *reporting use cases*. 

What about the third column? Here we have a few use cases that change the state of the application, such as edit an event, remove an event, add an event to calendar, and follow an organiser. 

But I do not see edit an event as a primary or core use case because it is like add an event. 

Once we build a domain model that allows us to capture n event, editing or removing it should not have a major impact in our domain. So, they are out. 

Adding an event to calendar, however, requires extending our domain model. For each user, we need to keep track of the events they are attending. So, this is another core use case. And the same applies to follow an organiser. For each user, we need to keep track of the organisers they follow. 

Now look at the fourth column, we don't have any primary use cases because all these are for reporting. 

What about the fifth column? Removing an event from calendar is kind of like adding an event to calendar. Same goes for unfollowing an organiser. Both these use cases are extensions of the core use cases in the third column.

**Core Use Cases**

![image-20200902133244915](/assets/req3)

### Identify the Supporting Use Cases

These are the most important use cases that we need to address first. But there is a tiny problem here. If we implement only these three use cases in our first iteration and showcase the system, there is no way they can see if an event is added to the database or not. So, we need to pick one or more use cases as **supporting use cases** to report on the state of the application. 

We can pick either of the use cases in the second column. But its preferred to pick **All Upcoming Events** as on this page, we are going to allow the user to add an event to their calendar or follow an organiser. So, this page, or this use case, *bridges the gap between the core use cases in the first and third columns*. 

We're going to use this technique one more time. Once we implement the use cases in the third column, there is no way for the user to see if the application is working. So, I am going to pick two use cases from the fourth column. **View Events I'm Attending** and **Who I'm Following**.

Again, these use cases are not really core use cases, but I use them as supporting use cases to build a piece of functionality end-to-end. This will help us build the skeleton of our application and gradually add new features to it. Once the skeleton is ready, we can also write automated tests to test the application functionality end-to-end.

![image-20200902133311928](/assets/req4)

### Result

At this point we have created a list of use cases, ordered them based on core functionality, added supported cases. This will enable us to demonstrate the system at a showcase, at the end of the first iteration.