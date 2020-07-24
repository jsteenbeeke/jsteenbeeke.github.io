---
layout: post
title:  "Mobile app synchronization using event sourcing"
author: "Jeroen Steenbeeke"
date:   2019-04-16 11:28:00 +0100
---
A couple of years ago I followed a course on the [Getting Things Done](https://en.wikipedia.org/wiki/Getting_Things_Done)
 methodology, after having struggled with prioritizing and even remembering stuff that needed doing.
 
As a result, I now have daily task lists of important stuff, and it's given me a lot of peace of mind, but the
methodology was about more than just day-to-day tasks, and a significant portion of the course was about long-term
perspective, and things you wanted to do and reach.

One of things I wanted to do was to learn how to make Android applications, and figuring I'd kill two birds
with one stone, I created my own task list app (which also has a web interface) called Coppermind (a reference
to the Feruchemists of the [Mistborn](https://en.wikipedia.org/wiki/Mistborn) series, who use "copperminds" to store knowledge).

This app has been working admirably for quite some time now, though last year I did a complete rebuild
to switch to a more flexible method of defining repeating tasks.

<!--more-->

## The problem

The old application synchronized with the backend by performing a simple query: give me all currently
relevant data. This included all active tasks, all projects, all rules, etcetera. This query was repeated
every few minutes. While the complete set of data transfered in this manner was small, it added up over a
span of 24 hours, easily eclipsing most other apps in data usage (the only exception was Youtube I believe).

What I wanted to do, instead of just ask my server to "give me everything", I instead created a mechanism
that provided me answer to the question "what has changed since my last synchronization?".

## Event sourcing

Event sourcing was a technique used at my previous employer to rebuild the data model of an application based on the state
of another application. One instance of the source application was queried, and then used to launch (I believe) 5 instances
of the target application. Any future change to the data model only needed to communicate what had changed to the target application
 (a problem I believe they eventually solved by hooking up both applications to the same event bus). The technique is described in more detail in [Martin Fowler's article on the subject](https://www.martinfowler.com/eaaDev/EventSourcing.html).
 
I expanded my application to keep a log of all changes in much the same manner, and added a REST service
that allowed the Android app to retrieve all changes since the last query, which it used to update its
local datamodel.

## Does this work?

This technique works quite well when it comes to reducing bandwidth usage both for my server and my mobile phone. Only tasks
that are not yet known on my phone get transfered, drastically cutting down the bandwidth usage.

### Except there is a bug

Last Tuesday I found that my phone would no longer sync with the backend. I first thought this was a simple deployment issue,
but the problem persisted, apparently caused by a discrepancy between the tasks known by the phone and those known by the server.

The solution in such a case is simple: wipe the phone data and rebuild the database from scratch using event playback. But this
took considerably longer than I expected considering the roughly 40 tasks that are currently active.

As of right now, the database contains 730 tasks (689 of which are marked as completed). The number of events logged, however,
is 229112, roughly 300 events for each task. This is divided as follows:

```
       type        | count 
-------------------+-------
 Complete task     |   689
 Create project    |    15
 Create repetition | 75667
 Create scope      |     5
 Create scope rule |     4
 Create task       |   743
 Delete project    |     0
 Delete repetition | 75121
 Delete scope      |     0
 Delete scope rule |     0
 Delete task       |    13
 Rename project    |     0
 Rename scope      |     0
 Update task       | 76855

```

~~I have not yet determined the cause of this, but considering the fact that most of the events logged
 are updates (a repetition is updated by deleting the old one and creating a new one), I suspect the
 app may be registering its updates more than once (perhaps a concurrency issue?).~~
 
~~For now I can probably mitigate this duplicate data by creating a deduplication background job. Once
 I have solved this mystery I will create a follow-up post.~~
 
 **Update, 21st of May 20202**: I have since managed to solve this issue. The background sync job had multiple instances
  running concurrently, as each new app start registered a new instance of the sync job. [Here's how I solved it.](/2020/05/21/background-sync-in-android.html).
