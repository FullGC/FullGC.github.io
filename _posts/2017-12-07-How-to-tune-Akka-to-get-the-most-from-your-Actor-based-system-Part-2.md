---
title: How to tune Akka to get the most from your Actor-based system - Part 2
layout: post
author: dani.shemesh
permalink: /how-to-tune-akka-to-get-the-most-from-your-actor-based-system-part-2/
tags:
- scala
- akka
- kamon
source-id: 1F1uJy7CsdjhzUlCOz7EQXgHJJWlO_Yr0bI56PpvPpvI
published: true
date: 2017-12-09 14:40:45
header-img: "img/tune-Akka.jpg"
---

<i>This post is the second of a two parts series of articles on how to tune Akka configurations</i>

* [Part-1: Initial Akka Configurations](https://fullgc.github.io/how-to-tune-akka-to-get-the-most-from-your-actor-based-system-part-1)
* [Part-2: Gather and analyze Akka metrics with Kamon and stackable traits](https://fullgc.github.io/how-to-tune-akka-to-get-the-most-from-your-actor-based-system-part-2)

[Previously](https://fullgc.github.io/how-to-tune-akka-to-get-the-most-from-your-actor-based-system-part-1/), we’ve tried to adjust Akka configurations for some possible use cases. After we set up a configuration and have a system up and running, we like to know how well we did and "re-tune" the configuration where needed.

## Part-2: Gather and analyze Akka metrics with Kamon and stackable traits

This part focuses on Akka metrics, means a high level data on the Akka Objects we've configured( i.e. Dispatchers, Routes, Actors(Routees), and Messages), how we gather them at Inneractive and some useful tips.

### **Monitoring Tools for Akka**

Akka library does not include a native monitoring tool, however there are a few tools that provide additional metrics for Akka-based application(i.e. memory usage, trace information), profiling and more capabilities that help identify performance issues or to put the finger on a bottleneck.  

To name a few:

* ['Lightbend Monitoring' ](https://developer.lightbend.com/docs/monitoring/2.0.x/home.html)

    Provides about all necessary features, including key Akka metrics and [span traces](https://developer.lightbend.com/docs/monitoring/2.0.x/instrumentations/akka/akka.html#span-tracer).

    Takipi plugin provides actor events that can trigger debug snapshots of the stack trace, i.e. the state at the time of the error. Btw It's not free...

* [Newrelic](https://docs.newrelic.com/)

    A powerful performance monitoring and management framework, has Spray and

    Akka-Http instrumented features. However it's most important feature, at least regarding our purpose, the [X-ray](https://docs.newrelic.com/docs/apm/transactions/x-ray-sessions/introduction-x-ray-sessions), which gives deeper insight into a key transaction's, is available only with a 'Pro’ subscription.

* Flame Graphs / VirtualVm or any other JVM profiling tool

* Kamon


**Kamon**

#### Overview

Kamon is an open source tool for monitoring applications running on the JVM. It supports various backends and has [modules](http://kamon.io/documentation/get-started) that integrate and gather metrics for Akka, Play, Spray/Akka-Http and more.

Kamon uses [Aspectj](http://www.eclipse.org/aspectj/) to add a layer of code before and after a method is executed, and in this [aspect- oriented](https://en.wikipedia.org/wiki/Aspect-oriented_programming) fashion records metrics data(e.g.  Records time before and after processing of messages/futures/routing (See [Appendix](#appendix)).

Here at Inneractive we use Kamon to gather all metrics, including application custom metrics, even in applications that do not usu Akka, because it proves to perform better (CPU wise) than other alternatives. 

It contains metric types functionalities like .histogram(..), .counter(..), .gauge(..) and .minMaxCounter(..)

#### Configuration for Kamon Akka metrics

[Akka integration ](http://kamon.io/documentation/kamon-akka/0.6.6/overview/)has a [collection of metrics](http://kamon.io/documentation/kamon-akka/0.6.6/actor-router-and-dispatcher-metrics/) for actor, router and dispatcher objects. At first, In our Exchange server, we collected them all. Soon enough our monitoring tools, [Prometheus](https://prometheus.io/) and [Datadog](https://www.datadoghq.com/dg/slpg/?utm_source=Advertisement&utm_medium=GoogleAdsNon1stTierBrand&utm_campaign=GoogleAdsNon1stTierBrand-Non1st&utm_content=Datadog&utm_keyword=%7Bkeyword%7D&utm_matchtype=%7Bmatchtype%7D&gclid=EAIaIQobChMI19CG9MjA1wIVEpMbCh1t0Q4dEAAYASAAEgLd4vD_BwE) went down / froze because of a *crazy* load of metrics. The reason is that we have 1000-1500 Actor instances. The metrics's names include the instance name (it is not a Tag as you might expect). We do have our own tags like 'environment’ and ‘host’, and about ~600 Exchange instances, and there you get a huge number of metrics that our monitors could not handle.

Also the 'aspect' way of gathering metric’s data is quite expensive, performance wise.

Our metric configuration looks as follows:

<!-- HTML generated using hilite.me --><div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #4070a0">&quot;metric&quot;</span><span style="color: #666666">:</span> <span style="color: #666666">{</span>
  <span style="color: #4070a0">&quot;filters&quot;</span><span style="color: #666666">:</span> <span style="color: #666666">{</span>
   <span style="color: #4070a0">&quot;akka-actor&quot;</span><span style="color: #666666">:</span> <span style="color: #666666">{</span>
     <span style="color: #4070a0">&quot;excludes&quot;</span><span style="color: #666666">:</span> <span style="color: #666666">[</span><span style="color: #4070a0">&quot;**&quot;</span><span style="color: #666666">],</span>
     <span style="color: #4070a0">&quot;includes&quot;</span><span style="color: #666666">:</span> <span style="color: #666666">[]</span>
   <span style="color: #666666">},</span>
   <span style="color: #4070a0">&quot;akka-dispatcher&quot;</span><span style="color: #666666">:</span> <span style="color: #666666">{</span>
     <span style="color: #4070a0">&quot;excludes&quot;</span><span style="color: #666666">:</span> <span style="color: #666666">[],</span>
     <span style="color: #4070a0">&quot;includes&quot;</span><span style="color: #666666">:</span> <span style="color: #666666">[</span><span style="color: #4070a0">&quot;**&quot;</span><span style="color: #666666">]</span>
   <span style="color: #666666">},</span>
   <span style="color: #4070a0">&quot;akka-router&quot;</span><span style="color: #666666">:</span> <span style="color: #666666">{</span>
     <span style="color: #4070a0">&quot;excludes&quot;</span><span style="color: #666666">:</span> <span style="color: #666666">[],</span>
     <span style="color: #4070a0">&quot;includes&quot;</span><span style="color: #666666">:</span> <span style="color: #666666">[</span><span style="color: #4070a0">&quot;**&quot;</span><span style="color: #666666">]</span>
   <span style="color: #666666">},</span>
   <span style="color: #4070a0">&quot;trace&quot;</span><span style="color: #666666">:</span> <span style="color: #666666">{</span>
     <span style="color: #4070a0">&quot;excludes&quot;</span><span style="color: #666666">:</span> <span style="color: #666666">[],</span>
     <span style="color: #4070a0">&quot;includes&quot;</span><span style="color: #666666">:</span> <span style="color: #666666">[</span> <span style="color: #4070a0">&quot;**&quot;</span><span style="color: #666666">]</span>
   <span style="color: #666666">}</span>
 <span style="color: #666666">}</span>
<span style="color: #666666">...</span>
<span style="color: #666666">}</span>
</pre></div>


As you may noticed,  we exclude akka-actor, which means all routees metrics(time-in-mailbox, processing-time, mailbox-size), and gather metrics for actors ourselves. This is because: 

* Routees metrics outnumbered all the others combined, since we have lots of actor instances, and we desperately needed our monitors to be working...

* We don't really mind for a single routee in pretty much all cases, so gathered information like average, sum, 95 can do the trick

* The metrics's names are somewhat awkward(partially because the name of the routees is like $a $b… and naming them means to create them explicitly in the code)

* We know the names of our actors, and have a foothold in our own actor's code. So we could gather metrics ourselves easily enough, and with a more concise information.   

In addition, we wanted metrics about the messages, processing time and the time it took the  

message to arrive.

#### Tips

1. When you route manually, means when you don't have a router, or whenever you have create the routees explicitly, you can give them names yourself(see [actorOf](https://doc.akka.io/api/akka/2.0/akka/actor/ActorContext.html) parameters).   

2. Akka does not provide an Api to know the size of the mailbox. you can monitor  by 

1. Have a messages counterlike Kamon does)

2. Override the MessageQueue. Google it. 

#### Configuration for Kamon Akka ActorSystem

As stated, the aspectj operating by Kamon is quite expensive, even when excluding of all routees metrics. By default, Kamon uses the default-dispatcher. If you don't set a default-dispatcher yourself, the threadpool size of the default-dispatcher will be the number of cores. In practice Kamon told us that about 75% of the running threads were Kamon’s.. You can set a dispatcher for Kamon as follows 
<img align="right" src="/img/kamon_the_rock.png">

<!-- HTML generated using hilite.me --><div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"> <span style="color: #4070a0">&quot;akka&quot;</span><span style="color: #666666">:</span> <span style="color: #666666">{</span>
   <span style="color: #4070a0">&quot;actor&quot;</span><span style="color: #666666">:</span> <span style="color: #666666">{</span>
     <span style="color: #4070a0">&quot;default-dispatcher&quot;</span><span style="color: #666666">:</span> <span style="color: #666666">{</span>
       <span style="color: #4070a0">&quot;fork-join-executor&quot;</span><span style="color: #666666">:</span> <span style="color: #666666">{</span>
         <span style="color: #4070a0">&quot;parallelism-factor&quot;</span><span style="color: #666666">:</span> <span style="color: #40a070">0.25</span><span style="color: #666666">,</span>
         <span style="color: #4070a0">&quot;parallelism-max&quot;</span><span style="color: #666666">:</span> <span style="color: #40a070">2</span><span style="color: #666666">,</span>
         <span style="color: #4070a0">&quot;parallelism-min&quot;</span><span style="color: #666666">:</span> <span style="color: #40a070">1</span>
       <span style="color: #666666">}</span>
     <span style="color: #666666">}</span>
   <span style="color: #666666">}</span>
 <span style="color: #666666">}</span>
</pre></div>


Here we allow up to 2 threads(but even 1 would probably be enough), with a parallelism-factor of 0.25, means up to 2 threads when the machine has at least 8 cores. 
This does the job.

#### Appendix

class [ActorMonitor.scala](https://github.com/kamon-io/kamon-akka/blob/master/kamon-akka-2.4.x/src/main/scala/kamon/akka/instrumentation/ActorMonitor.scala) from the module kamon-akka

<!-- HTML generated using hilite.me --><div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><table><tr><td><pre style="margin: 0; line-height: 125%">83
84
85
86
87
88
89
90
91
92
93
94
95
96
97
98
99
100
101
102
103
104
109
110</pre></td><td><pre style="margin: 0; line-height: 125%"> <span style="color: #007020; font-weight: bold">def</span> processMessage<span style="color: #666666">(</span>pjp<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">ProceedingJoinPoint</span><span style="color: #666666">,</span> envelopeContext<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">EnvelopeContext</span><span style="color: #666666">)</span><span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">AnyRef</span> <span style="color: #666666">=</span> <span style="color: #666666">{</span>
       <span style="color: #007020; font-weight: bold">val</span> timestampBeforeProcessing <span style="color: #007020; font-weight: bold">=</span> <span style="color: #0e84b5; font-weight: bold">RelativeNanoTimestamp</span><span style="color: #666666">.</span>now

      <span style="color: #007020; font-weight: bold">try</span> <span style="color: #666666">{</span>
        <span style="color: #0e84b5; font-weight: bold">Tracer</span><span style="color: #666666">.</span>withContext<span style="color: #666666">(</span>envelopeContext<span style="color: #666666">.</span>context<span style="color: #666666">)</span> <span style="color: #666666">{</span>
          pjp<span style="color: #666666">.</span>proceed<span style="color: #666666">()</span>
        <span style="color: #666666">}</span>

      <span style="color: #666666">}</span> <span style="color: #007020; font-weight: bold">finally</span> <span style="color: #666666">{</span>
        <span style="color: #007020; font-weight: bold">val</span> timestampAfterProcessing <span style="color: #007020; font-weight: bold">=</span> <span style="color: #0e84b5; font-weight: bold">RelativeNanoTimestamp</span><span style="color: #666666">.</span>now
        <span style="color: #007020; font-weight: bold">val</span> timeInMailbox <span style="color: #007020; font-weight: bold">=</span> timestampBeforeProcessing <span style="color: #666666">-</span> envelopeContext<span style="color: #666666">.</span>nanoTime
        <span style="color: #007020; font-weight: bold">val</span> processingTime <span style="color: #007020; font-weight: bold">=</span> timestampAfterProcessing <span style="color: #666666">-</span> timestampBeforeProcessing

        actorMetrics<span style="color: #666666">.</span>foreach <span style="color: #666666">{</span> am <span style="color: #007020; font-weight: bold">=&gt;</span>
          am<span style="color: #666666">.</span>processingTime<span style="color: #666666">.</span>record<span style="color: #666666">(</span>processingTime<span style="color: #666666">.</span>nanos<span style="color: #666666">)</span>
          am<span style="color: #666666">.</span>timeInMailbox<span style="color: #666666">.</span>record<span style="color: #666666">(</span>timeInMailbox<span style="color: #666666">.</span>nanos<span style="color: #666666">)</span>
          am<span style="color: #666666">.</span>mailboxSize<span style="color: #666666">.</span>decrement<span style="color: #666666">()</span>
        <span style="color: #666666">}</span>
       <span style="color: #666666">...</span>
      <span style="color: #666666">}</span>
    <span style="color: #666666">}</span>

<span>…</span>
  <span style="color: #666666">}</span>
</pre></td></tr></table></div>


<!-- HTML generated using hilite.me --><div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0e84b5; font-weight: bold">Line</span> <span style="color: #40a070">83</span> <span style="color: #666666">-</span> 'processMessage' executes 'around'<span style="color: #666666">(</span>before and after<span style="color: #666666">)</span> message processing by the actor<span style="color: #666666">(</span>jpg<span style="color: #666666">,</span> line <span style="color: #40a070">88</span><span style="color: #666666">)</span>
<span style="color: #0e84b5; font-weight: bold">Line</span> <span style="color: #40a070">84</span> <span style="color: #666666">-</span> <span style="color: #0e84b5; font-weight: bold">Takes</span> start time before the message being processed
<span style="color: #0e84b5; font-weight: bold">Line</span> <span style="color: #40a070">88</span> <span style="color: #666666">-</span> <span style="color: #0e84b5; font-weight: bold">Invokes</span> the method that handles message is processing
<span style="color: #0e84b5; font-weight: bold">Lines</span> <span style="color: #40a070">92</span><span style="color: #666666">-</span><span style="color: #40a070">94</span> <span style="color: #666666">-</span> <span style="color: #0e84b5; font-weight: bold">Latency</span> calculations
<span style="color: #0e84b5; font-weight: bold">Lines</span> <span style="color: #40a070">96</span><span style="color: #666666">-</span><span style="color: #40a070">99</span> <span style="color: #0e84b5; font-weight: bold">Updates</span> metrics cache
</pre></div>


**Monitor with Stackable Actor Traits**

We like to collect aggregated data for the routees and messages then.

In order to monitor time-in-mailbox, processing-time, mailbox-size that we excluded from Kamon metric configuration, we need to monitor 'around' the message processing, i.e. around the ‘receive’ method

It can be achieved by imitating Kamon's usage of [Aspectj](http://www.eclipse.org/aspectj/) as we just saw.

We however, use a 'stackable-traits‘ mixed to the Actors, for a monitoring layer around ‘receive'. 

[I wrote about stackable traits pattern here](https://fullgc.github.io/stackable-traits-pattern/). in [part-2](https://fullgc.github.io/how-to-tune-akka-to-get-the-most-from-your-actor-based-system-part-2/) **I describe(a simplified version of) how we use stackable actor traits in Inneractive, with code samples.**

### **Dashboard Overview**

Our [dashboard](#bookmark=id.gdmyvra3gln) consists of Kamon's Akka metrics and custom metrics.

Akka metrics collection explained in details in the [docs](http://kamon.io/documentation/kamon-akka/0.6.6/actor-router-and-dispatcher-metrics/) 

Among all metrics, put special attention to:

* 'active-threads' and ‘running-threads’, these give a very good view of your threads distribution so you would tune the dispatcher's configuration if needed

* 'pool-size', when you set upper, lower and increment factor and not a fixed size. This tells you how many threads are allocated in the pool in practice.

* 'processed-tasks' - How busy is the executor, maybe more threads are needed

* 'routing-time' - tells if the routing strategy fits.

* Custom metrics - 'time-in-mailbox by actor/message' for routees. Maybe a specific message handling does the trouble.

* Custom metrics 'message-start-process' for routees, when the value is high it can indicate that the receiver needs more resources or there are too many instances, lead to lots of context-switches. Increase of the "thoughput" parameter may help.

![image alt text]({{ site.url }}/public/asNMIb0wNDGjYnrdr4zCuQ_img_0.png)

### Wrapping up

Monitoring of the Akka objects plays an important and integral part of the tuning step (and generally important for keep track of the system behavior).
There are various tools and ways to do so, Kamon is friendly and recommended, and you can gather metric yourself.


### References

*[Kamon Documentation](http://kamon.io/documentation/get-started/)*

*[Kamon-Akka repository](https://github.com/kamon-io/kamon-akka)*

*[Stackable Actor Traits](https://fullgc.github.io/stackable-traits-pattern---part-2/)*

<div id="disqus_thread"></div>
<script>

/**
*  RECOMMENDED CONFIGURATION VARIABLES: EDIT AND UNCOMMENT THE SECTION BELOW TO INSERT DYNAMIC VALUES FROM YOUR PLATFORM OR CMS.
*  LEARN WHY DEFINING THESE VARIABLES IS IMPORTANT: https://disqus.com/admin/universalcode/#configuration-variables*/
var disqus_config = function () {
this.page.url = "https://fullgc.github.io/how-to-tune-akka-to-get-the-most-from-your-actor-based-system-part-2/"
this.page.identifier = Akka-2
};
(function() { // DON'T EDIT BELOW THIS LINE
var d = document, s = d.createElement('script');
s.src = 'https://FullGC.disqus.com/embed.js';
s.setAttribute('data-timestamp', +new Date());
(d.head || d.body).appendChild(s);
})();
</script>
<noscript>Please enable JavaScript to view the <a href="https://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>
