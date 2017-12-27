---
title: How to tune Akka to get the most from your Actor-based system - Part 1
layout: post
author: "Dani Shemesh"
permalink: /how-to-tune-akka-to-get-the-most-from-your-actor-based-system-part-1/
comments: true
tags:
- scala
- akka
source-id: 1iQAan2An0EVwukuK4r_L1rS6pCgLCtIVCp0qVxOZ-dk
published: true
date: 2017-12-10 14:40:45
header-img: "img/post-bg-02.jpg"
---

At some point, whether it during your new actor-based system planning, or after you have some prototype working, you'll probably find yourself dig into the Akka Docs to find the right combinations of the various possibilities of routing, dispatchers, number of actors instances and so forth..
Depending on the complexity of your system and performance requirements, this could get tedious.

<i>This post is the first of a two series of articles on how to tune Akka configurations</i>

* [Part-1: Initial Akka Configurations](https://fullgc.github.io/how-to-tune-akka-to-get-the-most-from-your-actor-based-system-part-1)
* [Part-2: Gather and analyze Akka metrics with Kamon and stackable traits](https://fullgc.github.io/how-to-tune-akka-to-get-the-most-from-your-actor-based-system-part-2)

## Initial Akka Configurations

Let's start with Akka configuration, specifically the configuration of [actor-instances](#heading=h.hhztx0701fu1), [routing strategy](#heading=h.cuvgdmxiz64e) and [dispatchers & executors](#heading=h.no1l9o35uyp0). Below is the relevant section of the application.conf

<!-- HTML generated using hilite.me --><div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"> <span style="color: #666666">{</span>
  akka <span style="color: #666666">{</span>
    actor <span style="color: #666666">{</span>
      akka<span style="color: #666666">.</span>actor<span style="color: #666666">.</span>deployment <span style="color: #666666">{</span>
        <span style="color: #666666">/</span>my<span style="color: #666666">-</span>service <span style="color: #666666">{</span>
           nr<span style="color: #666666">-</span>of<span style="color: #666666">-</span>instances <span style="color: #007020; font-weight: bold">=</span> <span style="color: #666666">???</span>
           router <span style="color: #007020; font-weight: bold">=</span> <span style="color: #666666">???</span>
           dispatcher <span style="color: #007020; font-weight: bold">=</span> <span style="color: #4070a0">&quot;my-dispatcher&quot;</span>
    <span style="color: #666666">}</span>
  my<span style="color: #666666">-</span>dispatcher <span style="color: #666666">{</span>
    executor <span style="color: #007020; font-weight: bold">=</span> <span style="color: #666666">???</span>
    <span style="color: #007020; font-weight: bold">type</span> <span style="color: #666666">=</span> <span style="color: #666666">???</span>
<span style="color: #666666">}</span>
</pre></div>


Let’s review some scenarios in which you may want to scale your routees:

### **Number of Actor Instances**

I like to start by thinking about how many instances of an actor should I have?

<!-- HTML generated using hilite.me --><div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"> akka<span style="color: #666666">.</span>actor<span style="color: #666666">.</span>deployment <span style="color: #666666">{</span>
    <span style="color: #666666">/</span>my<span style="color: #666666">-</span>service <span style="color: #666666">{</span>
      nr<span style="color: #666666">-</span>of<span style="color: #666666">-</span>instances <span style="color: #007020; font-weight: bold">=</span> <span style="color: #666666">???</span>
    <span style="color: #666666">}</span>
<span style="color: #666666">}</span>
</pre></div>

The size may depend on other configurations like routing strategy, dispatcher, threadpool size and more. Nevertheless, the nr-of-actor ‘strategy’ can be decided at this point.
Let’s review our options and use cases:

#### Single instance (or- Domain actor)

* A dedicated actor for low-priority side-effects like sending metrics, write to a log or to a cache and so forth.
* A mutable, single-source that needs to be handled(Cache)
* When you just have to work sequentially for whatever reason 

#### Fixed number of instances

* Instance per-a copy of a resource, or per a mutable resource
* For sharding, i.e. when you manage a distributed key-value cache, and want to shard the inputs, then you might want to have an actor to manage each shard.
* To execute tasks in parallel, and don't think you’ll need to manage [Back-Pressure](https://www.reactivemanifesto.org/glossary) nor to scale up	

#### Resizeable number of instances(when using a router)

<!-- HTML generated using hilite.me --><div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%">akka<span style="color: #666666">.</span>actor<span style="color: #666666">.</span>deployment <span style="color: #666666">{</span>
  <span style="color: #666666">/</span>parent<span style="color: #666666">/</span>router <span style="color: #666666">{</span>
    resizer <span style="color: #666666">{</span>
      lower<span style="color: #666666">-</span>bound <span style="color: #007020; font-weight: bold">=</span> <span style="color: #666666">???</span>
      upper<span style="color: #666666">-</span>bound <span style="color: #007020; font-weight: bold">=</span> <span style="color: #666666">???</span>
      pressure<span style="color: #666666">-</span>threshold <span style="color: #007020; font-weight: bold">=</span> <span style="color: #666666">???</span>
      messages<span style="color: #666666">-</span>per<span style="color: #666666">-</span>resize <span style="color: #007020; font-weight: bold">=</span> <span style="color: #666666">???</span>
      <span style="color: #666666">...</span>
    <span style="color: #666666">}</span>
  <span style="color: #666666">}</span>
<span style="color: #666666">}</span>
</pre></div>

It is possible to configure resizable routees(actors instances managed by a router).

routees can be added or removed dynamically, based on performance. You can configure specifically how much to scale up and down in case of unusual behavior.

##### Scale /  [Back-Pressure](https://www.reactivemanifesto.org/glossary) DIY!

When one component is struggling to keep-up, the system as a whole needs to 

respond in a sensible way.

You're somewhat familiar with [Akka-Streams](https://doc.akka.io/docs/akka/2.5/scala/stream/index.html), which widely known as a framework that manages back-pressure for you. You can imitate the general behavior by yourself.

Let's review some scenarios in which you may want to scale your routees: 

<img align="right" src="/img/loaded.png">
###### *The producer(In our use case, one of your actors), can produce faster than the received consumer(actor or any other source) can handle.*

 In this case you may:

* Back-pressure the producer, i.e. reduce the number of producer's routees.
* Add more consumers(routees...)!
* Leave it. You don't necessary need to back-pressure, Note it may lead to a loss of messages(Bounded mailbox)/ running out of memory...

<img align="left" src="/img/easy.png">
###### *The consumer is faster than the producer.*

Here the consumer will block waiting for the next item.

* Remove some consumers(routees...)! 

* Add more producers if your system can theoretically produce faster.  

* Leave it. Then you may not get the most from your machine.

<img align="right" src="/img/meeseeks.png">
##### *Actor** per-request*

<span style="font-weight: 400;">“</span><i><span style="font-weight: 400;">You press, you make a request, the </span></i><a href="https://en.wikipedia.org/wiki/Meeseeks_and_Destroy"><i><span style="font-weight: 400;">Meeseeks</span></i></a><i><span style="font-weight: 400;"> fulfills the request, and then it stops existing”(</span></i><a href="https://en.wikipedia.org/wiki/Rick_Sanchez_(Rick_and_Morty)"><i><span style="font-weight: 400;">Rick Sanchez</span></i></a><i><span style="font-weight: 400;">)</span></i>

<span style="font-weight: 400;">Actor per request works very similarly. An instance is created for every request, process it and then will be destroyed.</span>

<span style="font-weight: 400;">You can configure Spray/Akka-HTTP to work in actor-per-r</span>

You can configure Spray/Akka-HTTP to work in actor-per-request mode, or do it yourself, however it is not part of the Akka configuration so I won't get deep into details. In a nutshell-

* Easy to manage state in the actor, because the context is always of a specific request, hence you don't have to maintain any mapping of State => Request

* [And here are some more reasons](http://techblog.net-a-porter.com/2013/12/ask-tell-and-per-request-actors/)

Note that there is a context-switches overhead and could theoretically lead to memory issues

### **Routing**

Akka provides "strategies" for the Akka router to define workload distribution among actors. 

<!-- HTML generated using hilite.me --><div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%">akka<span style="color: #666666">.</span>actor<span style="color: #666666">.</span>deployment <span style="color: #666666">{</span>
    <span style="color: #666666">/</span>my<span style="color: #666666">-</span>service <span style="color: #666666">{</span>
      router <span style="color: #007020; font-weight: bold">=</span> <span style="color: #666666">???</span>
    <span style="color: #666666">}</span>
<span style="color: #666666">}</span>
</pre></div>


#### Strategies Overview

Let's quickly review the the routing strategies
* **Random** - Distributes messages *randomly
* **Round-Robin** - Distributes messages sequencely*
* **Smallest-Mailbox** - Sends the message to the smallest mailbox
* **Broadcast** - Distributes every message to all routees.
* **Scatter-Gather-First** - Distributes every message to all routees. *Only* the first to respond execute the task.
* **Tail-Chopping** - Sends the message to one, randomly picked, Routee and then after a small delay to a second Routee.
* **Consistent-hashing** - uses consistent hashing to select a Routee based on the sent message
* **In-Code** - Custom your own routing by route it yourself

#### *Strategies Cheatsheet*

<table>
<tbody>
<tr>
<td><b>Strategy</b><span style="font-weight: 400;"></span></td>
<td><b>When?</b><span style="font-weight: 400;"></span></td>
<td><b>Pros (when used as recommended</b><span style="font-weight: 400;">)</span></td>
<td><b>Cons (and misused implications)</b></td>
</tr>
<tr>
<td><b>Random</b></td>
<td>
<ul>
	<li style="font-weight: 400;"><span style="font-weight: 400;">No more than a few routees  or</span></li>
	<li style="font-weight: 400;"><span style="font-weight: 400;">Lightweight tasks or low throughput</span></li>
</ul>
</td>
<td>
<ul>
	<li style="font-weight: 400;"><span style="font-weight: 400;">~Evenly distributed</span></li>
	<li style="font-weight: 400;"><span style="font-weight: 400;">No overhead</span></li>
</ul>
</td>
<td>
<ul>
	<li style="font-weight: 400;"><span style="font-weight: 400;">Some mailboxes significantly more loaded than others*</span></li>
</ul>
</td>
</tr>
<tr>
<td><b><i>Round-Robin</i></b></td>
<td>
<ul>
	<li style="font-weight: 400;"><span style="font-weight: 400;">router handles similar tasks</span></li>
	<li style="font-weight: 400;"><span style="font-weight: 400;">Lightweight tasks or low throughput</span></li>
</ul>
</td>
<td>
<ul>
	<li style="font-weight: 400;"><span style="font-weight: 400;">~Evenly distributed</span></li>
	<li style="font-weight: 400;"><span style="font-weight: 400;">Minimal overhead</span></li>
</ul>
</td>
<td>
<ul>
	<li style="font-weight: 400;"><span style="font-weight: 400;">Some mailboxes more loaded than others*</span></li>
</ul>
</td>
</tr>
<tr>
<td><b>Smallest-Mailbox</b></td>
<td>
<ul>
	<li style="font-weight: 400;"><span style="font-weight: 400;">router handles similar tasks</span></li>
	<li style="font-weight: 400;"><span style="font-weight: 400;">No more than hundreds of routees**</span></li>
</ul>
</td>
<td>
<ul>
</ul>
</td>
<td>
<ul>
	<li style="font-weight: 400;"><span style="font-weight: 400;">Most evenly load of messages in mailboxes</span></li>
	<li style="font-weight: 400;"><span style="font-weight: 400;">Significant overhead for a high number of routees</span></li>
	<li style="font-weight: 400;"><span style="font-weight: 400;">Doesn’t hold with remote routees</span></li>
</ul>
</td>
</tr>
<tr>
<td><b>Broadcast</b></td>
<td>
<ul>
	<li style="font-weight: 400;"><span style="font-weight: 400;">Task must be completed at least ones </span></li>
	<li style="font-weight: 400;"><span style="font-weight: 400;">Limited number of routees on the same machine or lightweight tasks</span></li>
</ul>
</td>
<td>
<ul>
	<li style="font-weight: 400;"><span style="font-weight: 400;">Fault tolerance</span></li>
</ul>
</td>
<td>
<ul>
	<li style="font-weight: 400;"><span style="font-weight: 400;">Significant Machine’s cores overhead***</span></li>
</ul>
</td>
</tr>
<tr>
<td><b>Scatter-Gather-First</b></td>
<td>
<ul>
	<li style="font-weight: 400;"><span style="font-weight: 400;">Latency is very important</span></li>
	<li style="font-weight: 400;"><span style="font-weight: 400;">Using remote actors**</span></li>
</ul>
</td>
<td>
<ul>
	<li style="font-weight: 400;"><span style="font-weight: 400;">Low latency</span></li>
</ul>
</td>
<td>
<ul>
	<li style="font-weight: 400;"><span style="font-weight: 400;">Machine’s cores overhead</span></li>
</ul>
</td>
</tr>
<tr>
<td><b>Tail-Chopping</b></td>
<td>
<ul>
	<li style="font-weight: 400;"><span style="font-weight: 400;">Tasks must be completed</span></li>
	<li style="font-weight: 400;"><span style="font-weight: 400;">Latency is not very important</span></li>
	<li style="font-weight: 400;"><span style="font-weight: 400;">Using remote actors</span></li>
</ul>
</td>
<td>
<ul>
	<li style="font-weight: 400;"><span style="font-weight: 400;">Fault tolerance</span></li>
</ul>
</td>
<td>
<ul>
	<li style="font-weight: 400;"><span style="font-weight: 400;">Machine’s cores overhead</span></li>
	<li style="font-weight: 400;"><span style="font-weight: 400;">Fault recovery is slow</span></li>
</ul>
</td>
</tr>
<tr>
<td><b>Consistent-hashing</b></td>
<td>
<ul>
	<li style="font-weight: 400;"><span style="font-weight: 400;">Each routee is responsible for a specific part of a resource</span></li>
</ul>
</td>
<td></td>
<td></td>
</tr>
<tr>
<td><b>In-Code</b></td>
<td>
<ul>
	<li style="font-weight: 400;"><span style="font-weight: 400;">When nothing else suites</span></li>
</ul>
</td>
<td></td>
<td></td>
</tr>
</tbody>
</table>


*Can be solved by increasing the number of routees (which may cost in context-switches overhead)

**As a replacement for 'smallest mailbox'. 2. Latency differences could be high among connections to remote actors )

***the overhead depends on the task, whether it on the same machine or not


### **Dispatchers and Executors**

<!-- HTML generated using hilite.me --><div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%">akka<span style="color: #666666">.</span>actor<span style="color: #666666">.</span>deployment <span style="color: #666666">{</span>
    <span style="color: #666666">/</span>my<span style="color: #666666">-</span>service <span style="color: #666666">{</span>
      dispatcher <span style="color: #007020; font-weight: bold">=</span> <span style="color: #666666">???</span>
      <span style="color: #007020; font-weight: bold">type</span> <span style="color: #666666">=</span> <span style="color: #666666">???</span>
    <span style="color: #666666">}</span>
<span style="color: #666666">}</span>
</pre></div>

Dispatchers are [what makes Akka actors "tick"](https://doc.akka.io/docs/akka/2.5/java/dispatchers.html), means put messages in mailboxes and route them. In addition, they are also an implementation of ExecutionContext, means they can execute Runnables and so a Scala Future.

<!-- HTML generated using hilite.me --><div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%">my<span style="color: #666666">-</span>dispatcher <span style="color: #666666">{</span>
  executor <span style="color: #007020; font-weight: bold">=</span> <span style="color: #666666">???</span>
  throughput <span style="color: #007020; font-weight: bold">=</span> <span style="color: #666666">???</span>
<span style="color: #666666">....</span>
<span style="color: #666666">}</span>
</pre></div>


#### Fork-Join-executor

<!-- HTML generated using hilite.me --><div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%">my<span style="color: #666666">-</span>dispatcher <span style="color: #666666">{</span>
  executor <span style="color: #007020; font-weight: bold">=</span> <span style="color: #4070a0">&quot;fork-join-executor&quot;</span>
<span style="color: #666666">....</span>
<span style="color: #666666">}</span>
</pre></div>

Java 7 introduced the Fork-Join executor.

As the name suggests, it *forks* a task into subtasks, each executed by a different thread, and *joined the *results.

There are 2 main characters that worth mentioning here. According to [Oracle docs](https://docs.oracle.com/javase/tutorial/essential/concurrency/forkjoin.html) - 

1. *"It is designed for work that can be broken into smaller pieces recursively".*

It hence best for recursive problems - where a task can be broken into sub-tasks such that they would be executed in parallel and their results would be collected.

2. *"The fork/join framework is distinct because it uses a work-stealing algorithm. Worker threads that run out of things to do can steal tasks from other threads that are still busy"*

**Fork-Join shows better performance in most cases, compare to old Thread-Pool-Executor**, as it makes a better uses of the resources, as the idle threads can steal tasks from busier threads. 

However there is a build-in danger here. 

From the first statement, when a 'Fork' performed, we have multiple threads and each of them is responsible to run a task. 

From the second, when a thread is done it can take some other task. But what if he got stuck on this task? The other threads will wait on the 'Join' at some point, which is a threads starvation.

<img align="right" src="/img/fork.jpg">

<!-- HTML generated using hilite.me --><div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%">my<span style="color: #666666">-</span>dispatcher <span style="color: #666666">{</span>
  executor <span style="color: #007020; font-weight: bold">=</span> <span style="color: #4070a0">&quot;fork-join-executor&quot;</span>
  fork<span style="color: #666666">-</span>join<span style="color: #666666">-</span>executor <span style="color: #666666">{</span>
    <span style="color: #007020; font-weight: bold">#</span> <span style="color: #0e84b5; font-weight: bold">Min</span> number of threads to cap factor<span style="color: #666666">-</span>based parallelism number to
    <span style="color: #007020; font-weight: bold">#</span> <span style="color: #0e84b5; font-weight: bold">Note</span> that these threads will be created anyway on fork<span style="border: 1px solid #FF0000">&#39;</span><span style="color: #666666"></span>
    <span style="color: #007020; font-weight: bold">#</span> so <span style="color: #007020; font-weight: bold">try</span> to avoid an unnecessary overhead<span style="color: #666666">.</span>
    parallelism<span style="color: #666666">-</span>min <span style="color: #007020; font-weight: bold">=</span> <span style="color: #666666">???</span>
    <span style="color: #007020; font-weight: bold">#</span> <span style="color: #0e84b5; font-weight: bold">Parallelism</span> <span style="color: #666666">(</span>threads<span style="color: #666666">)</span> <span style="color: #666666">...</span> ceil<span style="color: #666666">(</span>available processors <span style="color: #666666">*</span> factor<span style="color: #666666">)</span>
    parallelism<span style="color: #666666">-</span>factor <span style="color: #007020; font-weight: bold">=</span> <span style="color: #666666">???</span>
    <span style="color: #007020; font-weight: bold">#</span> <span style="color: #0e84b5; font-weight: bold">This</span> is <span style="color: #0e84b5; font-weight: bold">NOT</span> an upper bound on the total number of threads<span style="color: #666666">!</span>
    <span style="color: #007020; font-weight: bold">#</span> <span style="color: #0e84b5; font-weight: bold">Max</span> number of threads to cap factor<span style="color: #666666">-</span>based parallelism number to
    parallelism<span style="color: #666666">-</span>max <span style="color: #007020; font-weight: bold">=</span> <span style="color: #666666">???</span>
<span style="color: #666666">....</span>
  <span style="color: #666666">}</span>
<span style="color: #666666">}</span>
</pre></div>

A common case is to use Fork-Join executor for futures inside an actor. Here, the dispatcher's configuration of the actor should be considered as well. For example, the more threads you have for the actor, the more 'future’ tasks would be performed, and you’ll may want more threads for them.

#### Thread-pool-executor

The old Java 5 executor for asynchronous task execution can still fit in some cases, and without the Fork-Join overhead.

While Fork-Join breaks the task for you, if you know how to break the task yourself, then your code built already as minimal task that should be executed by a single thread, which fit thread-pool-executor.

Thread-pool executor is used by akka Dispatcher and PinnedDispatcher. 

**Dispatcher** let you define min, max and increase factor / fixed size for your threadpool.

<!-- HTML generated using hilite.me --><div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%">my<span style="color: #666666">-</span>dispatcher <span style="color: #666666">{</span>
  <span style="color: #007020; font-weight: bold">type</span> <span style="color: #666666">=</span> <span style="color: #0e84b5; font-weight: bold">Dispatcher</span>
  executor <span style="color: #007020; font-weight: bold">=</span> <span style="color: #4070a0">&quot;thread-pool-executor&quot;</span>
  thread<span style="color: #666666">-</span>pool<span style="color: #666666">-</span>executor <span style="color: #666666">{</span>
    <span style="color: #007020; font-weight: bold">#</span> <span style="color: #0e84b5; font-weight: bold">Min</span> number of threads to cap factor<span style="color: #666666">-</span>based parallelism number to
    parallelism<span style="color: #666666">-</span>min <span style="color: #007020; font-weight: bold">=</span> <span style="color: #666666">???</span>
    <span style="color: #007020; font-weight: bold">#</span> <span style="color: #0e84b5; font-weight: bold">Parallelism</span> <span style="color: #666666">(</span>threads<span style="color: #666666">)</span> <span style="color: #666666">...</span> ceil<span style="color: #666666">(</span>available processors <span style="color: #666666">*</span> factor<span style="color: #666666">)</span>
    parallelism<span style="color: #666666">-</span>factor <span style="color: #007020; font-weight: bold">=</span> <span style="color: #666666">???</span>
    <span style="color: #007020; font-weight: bold">#</span> <span style="color: #0e84b5; font-weight: bold">Max</span> number of threads to cap factor<span style="color: #666666">-</span>based parallelism number to
    parallelism<span style="color: #666666">-</span>max <span style="color: #007020; font-weight: bold">=</span> <span style="color: #666666">???</span>
  <span style="color: #666666">}</span>
<span style="color: #666666">....</span>
<span style="color: #666666">}</span>
</pre></div>

The key is to find the right balance for an actor instances to work in parallel and use the threads as much as they need so other actors and processes would be able to work as well. Its also true for the Fork-Join executor and needs to be quite accurate. In Part 2 //TBA (link to part 2)

We'll talk about how to measure it.

**PinnedDispatcher** dedicates a unique thread for each actor. This is usually not the pattern you want for the machine's resources are limited. Hence it makes sense for the actor to share a pool of threads. However, if your actor performs a prefered task and you don’t want its instances to share its pool. 

Do not use it if you have more instances than the number of cores in the machine. 

It is also not recommended for Futures, because  you'll probably need more than 1 thread... 

#### Affinity-pool-executor

<!-- HTML generated using hilite.me --><div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%">my<span style="color: #666666">-</span>dispatcher <span style="color: #666666">{</span>
  executor <span style="color: #007020; font-weight: bold">=</span> <span style="color: #4070a0">&quot;affinity-pool-executor&quot;</span>
<span style="color: #666666">....</span>
<span style="color: #666666">}</span>
</pre></div>


This executor tries its best to have your actor instance always schedule with the same thread, which should increase throughput.

It is recommended for small number of actor instances, for if you have much more instances than threads, it is just not possible.

#### Tips
<img align="right" src="/img/dispatcher.jpg">
* Don't use the [Akka default dispatcher](https://doc.akka.io/docs/akka/2.5/scala/dispatchers.html) for your actorSystem nor for the actors themselves. Note that external Akka based frameworks use it as default, and you should configure a dedicated dispatcher for them as well.

* Have a different Dispatcher for each actor, and for Futures inside an actor.

* Dispatchers has 'throughput' parameter, which "*defines the maximum number of messages to be processed per actor before the thread jumps to the next actor"* Setting It to higher value than the default, 1, is likely to improve performance if it is not part of Affinity-pool dispatcher, and if your actors generally not very busy (otherwise the lack of fairness can cause a high load in some mailboxes).

* Read [this terrific post in ScalaC blog](https://blog.scalac.io/improving-akka-dispatcher.html). It explains Dispatcher's internals in details.

### **Next**

In [Part-2](https://fullgc.github.io/how-to-tune-akka-to-get-the-most-from-your-actor-based-system-part-2) I will show how we monitor and analyze our actor-based system.

```
Inneractive maintains an Exchange server, which, simply put, gets an advertisement for a mobile application from Ad-Networks.
In fact there are ~500 server instances at a given moment, dealing with ~10,000,000 Ad requests per a minute.
During the process the Exchange server performs a real-time auction by going out (with scala.Future)s to multiple Ad-Networks(consumers).
This translates to ~150,000,000 transactions per a minute.
The Exchange server is akka-based, it uses Spray as a server side-http, and the entire flow is actor-based.
We use other Akka frameworks in other modules like Akka-Http and Akka Streams.
```

<div id="disqus_thread"></div>
<script>

/**
*  RECOMMENDED CONFIGURATION VARIABLES: EDIT AND UNCOMMENT THE SECTION BELOW TO INSERT DYNAMIC VALUES FROM YOUR PLATFORM OR CMS.
*  LEARN WHY DEFINING THESE VARIABLES IS IMPORTANT: https://disqus.com/admin/universalcode/#configuration-variables*/
var disqus_config = function () {
this.page.url = "https://fullgc.github.io/how-to-tune-akka-to-get-the-most-from-your-actor-based-system-part-1/"
this.page.identifier = Akka-1
};
(function() { // DON'T EDIT BELOW THIS LINE
var d = document, s = d.createElement('script');
s.src = 'https://FullGC.disqus.com/embed.js';
s.setAttribute('data-timestamp', +new Date());
(d.head || d.body).appendChild(s);
})();
</script>
<noscript>Please enable JavaScript to view the <a href="https://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>