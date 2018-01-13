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
header-img: "img/tune-Akka.jpg"
---

<i>This post is the first of a two parts series of articles on how to tune Akka configurations</i>

* [Part-1: Initial Akka Configurations](https://fullgc.github.io/how-to-tune-akka-to-get-the-most-from-your-actor-based-system-part-1)
* [Part-2: Gather and analyze Akka metrics with Kamon and stackable traits](https://fullgc.github.io/how-to-tune-akka-to-get-the-most-from-your-actor-based-system-part-2)

At some point, whether it during your new actor-based system planning, or after you have some prototype working, you'll probably find yourself dig into the Akka Docs to find the right combinations of the various possibilities of routing, dispatchers, number of actors instances and so forth..
Depending on the complexity of your system and performance requirements, this could get tedious.

## Part-1: Initial Akka Configurations

Let's start with Akka configuration, specifically the configuration of [actor-instances](#heading=h.hhztx0701fu1), [routing strategy](#heading=h.cuvgdmxiz64e) and [dispatchers & executors](#heading=h.no1l9o35uyp0). Below is the relevant section of the application.conf
````
 {
  akka {
    actor {
      akka.actor.deployment {
        /my-service {
           nr-of-instances = ???
           router = ???
           dispatcher = "my-dispatcher"
    }
  my-dispatcher {
    executor = ???
    type = ???
}
````

Let’s review some scenarios in which you may want to scale your routees:

<br><br>
### **Number of Actor Instances**

I like to start by thinking about how many instances of an actor should I have?
````
 akka.actor.deployment {
    /my-service {
      nr-of-instances = ???
    }
}
````

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
````
akka.actor.deployment {
  /parent/router {
    resizer {
      lower-bound = ???
      upper-bound = ???
      pressure-threshold = ???
      messages-per-resize = ???
      ...
    }
  }
}
````

It is possible to configure resizable routees(actors instances managed by a router).

routees can be added or removed dynamically, based on performance. You can configure specifically how much to scale up and down in case of unusual behavior.

##### Scale /  [Back-Pressure](https://www.reactivemanifesto.org/glossary) DIY!

When one component is struggling to keep-up, the system as a whole needs to respond in a sensible way.

You're somewhat familiar with [Akka-Streams](https://doc.akka.io/docs/akka/2.5/scala/stream/index.html), which widely known as a framework that manages back-pressure for you. You can imitate the general behavior by yourself.

Let's review some scenarios in which you may want to scale your routees: 

###### *The producer(In our use case, one of your actors), can produce faster than the received consumer(actor or any other source) can handle.*
<img align="right" src="/img/loaded.png" height="300" width="300">
 In this case you may:

* Back-pressure the producer, i.e. reduce the number of producer's routees.

* Add more consumers(routees...)!

* Leave it. You don't necessary need to back-pressure, Note it may lead to a loss of messages(Bounded mailbox)/ running out of memory...

<img align="right" src="/img/easy.png" height="300" width="300">
###### *The consumer is faster than the producer.*

Here the consumer will block waiting for the next item.

* Remove some consumers(routees...)! 

* Add more producers if your system can theoretically produce faster.  

* Leave it. Then you may not get the most from your machine.

##### *Actor per-request*
<img align="right" src="/img/meeseeks.png" height="100" width="100">
<span style="font-weight: 400;">“</span><i><span style="font-weight: 400;">You press, you make a request, the </span></i><a href="https://en.wikipedia.org/wiki/Meeseeks_and_Destroy"><i><span style="font-weight: 400;">Meeseeks</span></i></a><i><span style="font-weight: 400;"> fulfills the request, and then it stops existing”(</span></i><a href="https://en.wikipedia.org/wiki/Rick_Sanchez_(Rick_and_Morty)"><i><span style="font-weight: 400;">Rick Sanchez</span></i></a><i><span style="font-weight: 400;">)</span></i>

<span style="font-weight: 400;">Actor per request works very similarly. An instance is created for every request, process it and then will be destroyed.</span>

You can configure Spray/Akka-HTTP to work in actor-per-request mode, or do it yourself, however it is not part of the Akka configuration so I won't get deep into details. In a nutshell-

* Easy to manage state in the actor, because the context is always of a specific request, hence you don't have to maintain any mapping of State => Request

* [And here are some more reasons](http://techblog.net-a-porter.com/2013/12/ask-tell-and-per-request-actors/)

Note that there is a context-switches overhead and could theoretically lead to memory issues

<br><br>
### **Routing**

Akka provides "strategies" for the Akka router to define workload distribution among actors. 
````
akka.actor.deployment {
    /my-service {
      router = ???
    }
}
````

#### Strategies Overview

Let's quickly review the the routing strategies
* **Random** - Distributes messages randomly

* **Round-Robin** - Distributes messages sequencely

* **Smallest-Mailbox** - Sends the message to the smallest mailbox

* **Broadcast** - Distributes every message to all routees.

* **Scatter-Gather-First** - Distributes every message to all routees. *Only* the first to respond execute the task.

* **Tail-Chopping** - Sends the message to one, randomly picked, Routee and then after a small delay to a second Routee.

* **Consistent-hashing** - uses consistent hashing to select a Routee based on the sent message

* **In-Code** - Custom your own routing by route it yourself


#### *Strategies Cheatsheet*
<img align="right" src="/img/strategies.jpg">

*Can be solved by increasing the number of routees (which may cost in context-switches overhead)

**As a replacement for 'smallest mailbox'. 2. Latency differences could be high among connections to remote actors

***the overhead depends on the task, whether it on the same machine or not


<br><br>
### **Dispatchers and Executors**
````
akka.actor.deployment {
    /my-service {
      dispatcher = ???
      type = ???
    }
}
````

Dispatchers are [what makes Akka actors "tick"](https://doc.akka.io/docs/akka/2.5/java/dispatchers.html), means put messages in mailboxes and route them. In addition, they are also an implementation of ExecutionContext, means they can execute Runnables and so a Scala Future.
````
my-dispatcher {
  executor = ???
  throughput = ???
....
}
````

#### Fork-Join-executor

````
my-dispatcher {
  executor = "fork-join-executor"
....
}
````

Java 7 introduced the Fork-Join executor.

As the name suggests, it *forks* a task into subtasks, each executed by a different thread, and *joined* the results.

There are 2 main characters that worth mentioning here. According to [Oracle docs](https://docs.oracle.com/javase/tutorial/essential/concurrency/forkjoin.html) - 

1. *"It is designed for work that can be broken into smaller pieces recursively".*

    It hence best for recursive problems - where a task can be broken into sub-tasks such that they would be executed in parallel and their results would be collected.

2. *"The fork/join framework is distinct because it uses a work-stealing algorithm. Worker threads that run out of things to do can steal tasks from other threads that are still busy"*

    **Fork-Join shows better performance in most cases, compare to old Thread-Pool-Executor**, as it makes a better uses of the resources, as the idle threads can steal tasks from busier threads.

    However there is a build-in danger here.

From the first statement, when a 'Fork' performed, we have multiple threads and each of them is responsible to run a task. 

From the second, when a thread is done it can take some other task. But what if he got stuck on this task? The other threads will wait on the 'Join' at some point, which is a threads starvation.

````
my-dispatcher {
  executor = "fork-join-executor"
  fork-join-executor {
    # Min number of threads to cap factor-based parallelism number to
    # Note that these threads will be created anyway on fork'
    # so try to avoid an unnecessary overhead.
    parallelism-min = ???
    # Parallelism (threads) ... ceil(available processors * factor)
    parallelism-factor = ???
    # This is NOT an upper bound on the total number of threads!
    # Max number of threads to cap factor-based parallelism number to
    parallelism-max = ???
....
  }
}
````

A common case is to use Fork-Join executor for futures inside an actor. Here, the dispatcher's configuration of the actor should be considered as well. For example, the more threads you have for the actor, the more 'future’ tasks would be performed, and you’ll may want more threads for them.

#### Thread-pool-executor

The old Java 5 executor for asynchronous task execution can still fit in some cases, and without the Fork-Join overhead.

While Fork-Join breaks the task for you, if you know how to break the task yourself, then your code built already as minimal task that should be executed by a single thread, which fit thread-pool-executor.

Thread-pool executor is used by akka Dispatcher and PinnedDispatcher. 

**Dispatcher** let you define min, max and increase factor / fixed size for your threadpool.

````
my-dispatcher {
  type = Dispatcher
  executor = "thread-pool-executor"
  thread-pool-executor {
    # Min number of threads to cap factor-based parallelism number to
    parallelism-min = ???
    # Parallelism (threads) ... ceil(available processors * factor)
    parallelism-factor = ???
    # Max number of threads to cap factor-based parallelism number to
    parallelism-max = ???
  }
....
}
````

The key is to find the right balance for an actor instances to work in parallel and use the threads as much as they need so other actors and processes would be able to work as well. Its also true for the Fork-Join executor and needs to be quite accurate. In Part 2 //TBA (link to part 2)

We'll talk about how to measure it.

**PinnedDispatcher** dedicates a unique thread for each actor. This is usually not the pattern you want for the machine's resources are limited. Hence it makes sense for the actor to share a pool of threads. However, if your actor performs a prefered task and you don’t want its instances to share its pool. 

Do not use it if you have more instances than the number of cores in the machine. 

It is also not recommended for Futures, because  you'll probably need more than 1 thread... 

#### Affinity-pool-executor

````
my-dispatcher {
  executor = "affinity-pool-executor"
....
}
````

This executor tries its best to have your actor instance always schedule with the same thread, which should increase throughput.

It is recommended for small number of actor instances, for if you have much more instances than threads, it is just not possible.

#### Tips
<img align="right" src="/img/dispatcher.jpg" height="250" width="250">
* Don't use the [Akka default dispatcher](https://doc.akka.io/docs/akka/2.5/scala/dispatchers.html) for your actorSystem nor for the actors themselves. Note that external Akka based frameworks use it as default, and you should configure a dedicated dispatcher for them as well.

* Have a different Dispatcher for each actor, and for Futures inside an actor.

* Dispatchers has 'throughput' parameter, which "*defines the maximum number of messages to be processed per actor before the thread jumps to the next actor"* Setting It to higher value than the default, 1, is likely to improve performance if it is not part of Affinity-pool dispatcher, and if your actors generally not very busy (otherwise the lack of fairness can cause a high load in some mailboxes).

* Read [this terrific post in ScalaC blog](https://blog.scalac.io/improving-akka-dispatcher.html). It explains Dispatcher's internals in details.

<br><br>
### **Next**

In [Part-2](https://fullgc.github.io/how-to-tune-akka-to-get-the-most-from-your-actor-based-system-part-2) I will show how we monitor and analyze our actor-based system.

```
Inneractive maintains an Exchange server, which, simply put,
gets an advertisement for a mobile application from Ad-Networks.
In fact there are ~500 server instances at a given moment,
dealing with ~10,000,000 Ad requests per a minute.
During the process the Exchange server performs a real-time auction by going out (with scala.Future)s
to multiple Ad-Networks(consumers).
This translates to ~150,000,000 transactions per a minute.
The Exchange server is akka-based, it uses Spray as a server side-http,
and the entire flow is actor-based.
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
