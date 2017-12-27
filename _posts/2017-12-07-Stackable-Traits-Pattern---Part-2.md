---
title: Stackable Traits pattern - Part 2
layout: post
author: dani.shemesh
permalink: /stackable-traits-pattern---part-2/
tags:
- scala
- stackable
- traits
- akka
source-id: 1m6eIR7wGA1qDRmceIp1S3If9B9k-lpDksi4aRdQ_d1Y
published: true
date: 2017-12-07 14:40:45
header-img: "img/burger-stack.jpg"
---

[Stackable traits](https://fullgc.github.io/stackable-traits-pattern/) can be applied to actors as well.
Specifically, we can use stackable actor traits to modify the behavior of the 'receive' method.


## **Gathering Metrics**

We like to gather the following metrics

* time-in-mailbox: The time from the moment a message was enqueued into an actor's mailbox until the moment it was dequeued for processing.

* processing-time: How long did it take for the actor to process a message.

And to log when an actor starts to handle a message and before it finishes.

### **Stackable Actors based implementation**

Say we have the following actor that we like to monitor:

<!-- HTML generated using hilite.me --><div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #007020; font-weight: bold">abstract</span> <span style="color: #007020; font-weight: bold">class</span> <span style="color: #0e84b5; font-weight: bold">MyActor</span> <span style="color: #007020; font-weight: bold">extends</span> <span style="color: #0e84b5; font-weight: bold">Actor</span> <span style="color: #007020; font-weight: bold">with</span> <span style="color: #0e84b5; font-weight: bold">StrictLogging</span> <span style="color: #666666">{</span>
 <span style="color: #007020; font-weight: bold">override</span> <span style="color: #007020; font-weight: bold">def</span> receive<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">Receive</span> <span style="color: #666666">=</span> <span style="color: #666666">{</span>
   <span style="color: #007020; font-weight: bold">case</span> message <span style="color: #007020; font-weight: bold">=&gt;</span> logger<span style="color: #666666">.</span>info<span style="color: #666666">(</span><span style="color: #4070a0">&quot;performing some work...&quot;</span><span style="color: #666666">)</span>
 <span style="color: #666666">}</span>
<span style="color: #666666">}</span>
</pre></div>

Let's start with the 'time-in-mailbox’ metric. The simple way to implement it is to take time before the message is sent, and calculate the time in mailbox when the actor is starting pressing it. For the sake of the example we’ll assume that a message is created just before it being sent.

The message class that should be monitored be:

<!-- HTML generated using hilite.me --><div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #007020; font-weight: bold">trait</span> <span style="color: #0e84b5; font-weight: bold">RecordableMessage</span> <span style="color: #007020; font-weight: bold">extends</span> <span style="color: #0e84b5; font-weight: bold">RichMessage</span> <span style="color: #666666">{</span>
  <span style="color: #007020; font-weight: bold">val</span> dispatchTime<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">Long</span> <span style="color: #666666">=</span>
  <span style="color: #0e84b5; font-weight: bold">System</span><span style="color: #666666">.</span>currentTimeMillis<span style="color: #666666">()</span>
<span style="color: #666666">}</span>
<span style="color: #007020; font-weight: bold">trait</span> <span style="color: #0e84b5; font-weight: bold">RichMessage</span> <span style="color: #666666">{</span>
  <span style="color: #007020; font-weight: bold">val</span> messageName<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">String</span>
<span style="color: #666666">}</span>
</pre></div>

We initializing the time before the message is being sent, and give it a name, to be use as a tag for the metric.

Next, create the stackable trait for monitoring the actor on RecordableMessage

<!-- HTML generated using hilite.me --><div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #007020; font-weight: bold">trait</span> <span style="color: #0e84b5; font-weight: bold">LatencyRecorderActor</span> <span style="color: #007020; font-weight: bold">extends</span> <span style="color: #0e84b5; font-weight: bold">Actor</span> <span style="color: #007020; font-weight: bold">with</span> <span style="color: #0e84b5; font-weight: bold">StrictLogging</span> <span style="color: #666666">{</span>
 <span style="color: #007020; font-weight: bold">val</span> actorName<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">String</span> <span style="color: #666666">=</span> <span style="color: #007020; font-weight: bold">this</span><span style="color: #666666">.</span>getClass<span style="color: #666666">.</span>getSimpleName

 <span style="color: #007020; font-weight: bold">abstract</span> <span style="color: #007020; font-weight: bold">override</span> <span style="color: #007020; font-weight: bold">def</span> receive<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">Receive</span> <span style="color: #666666">=</span> <span style="color: #666666">{</span>
   <span style="color: #007020; font-weight: bold">case</span> recordableMessage<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">RecordableMessage</span> <span style="color: #666666">=&gt;</span>
     <span style="color: #0e84b5; font-weight: bold">Monitor</span><span style="color: #666666">.</span>record<span style="color: #666666">(</span><span style="color: #4070a0">&quot;time-in-mailbox&quot;</span><span style="color: #666666">,</span> actorName<span style="color: #666666">,</span> recordableMessage<span style="color: #666666">.</span>messageName<span style="color: #666666">,</span>
        recordableMessage<span style="color: #666666">.</span>dispatchTime<span style="color: #666666">)</span>
     <span style="color: #007020; font-weight: bold">val</span> start <span style="color: #007020; font-weight: bold">=</span> <span style="color: #0e84b5; font-weight: bold">System</span><span style="color: #666666">.</span>currentTimeMillis<span style="color: #666666">()</span>
     <span style="color: #007020; font-weight: bold">super</span><span style="color: #666666">.</span>receive<span style="color: #666666">(</span>recordableMessage<span style="color: #666666">)</span>
     <span style="color: #0e84b5; font-weight: bold">Monitor</span><span style="color: #666666">.</span>record<span style="color: #666666">(</span><span style="color: #4070a0">&quot;processing-time&quot;</span><span style="color: #666666">,</span> actorName<span style="color: #666666">,</span> recordableMessage<span style="color: #666666">.</span>messageName<span style="color: #666666">,</span> start<span style="color: #666666">)</span>
   <span style="color: #007020; font-weight: bold">case</span> message <span style="color: #007020; font-weight: bold">=&gt;</span> <span style="color: #007020; font-weight: bold">super</span><span style="color: #666666">.</span>receive<span style="color: #666666">(</span>message<span style="color: #666666">)</span>
 <span style="color: #666666">}</span>
<span style="color: #666666">}</span>
</pre></div>

You might notice that

1. As discussed in [part-1](https://fullgc.github.io/stackable-traits-pattern/), the modifier of the 'receive' method should be "abstract override"

2. We gather the metrics only on RecordableMessage message

3. For calculating 'time-in-mailbox', 'dispatchTime’ is used

4. For calculating 'processing-time', we take take time before invoking the action, then invoking the action, and record the ‘processing-time’ when it finished.

The LoggerActor is the following 

<!-- HTML generated using hilite.me --><div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #007020; font-weight: bold">trait</span> <span style="color: #0e84b5; font-weight: bold">LoggerActor</span> <span style="color: #007020; font-weight: bold">extends</span> <span style="color: #0e84b5; font-weight: bold">Actor</span> <span style="color: #007020; font-weight: bold">with</span> <span style="color: #0e84b5; font-weight: bold">StrictLogging</span> <span style="color: #666666">{</span>
 <span style="color: #007020; font-weight: bold">abstract</span> <span style="color: #007020; font-weight: bold">override</span> <span style="color: #007020; font-weight: bold">def</span> receive<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">Receive</span> <span style="color: #666666">=</span> <span style="color: #666666">{</span>
   <span style="color: #007020; font-weight: bold">case</span> recordableMessage<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">RecordableMessage</span> <span style="color: #666666">=&gt;</span>
     logger<span style="color: #666666">.</span>info<span style="color: #666666">(</span>s<span style="color: #4070a0">&quot;handling message: ${recordableMessage.messageName}&quot;</span><span style="color: #666666">)</span>
     <span style="color: #007020; font-weight: bold">super</span><span style="color: #666666">.</span>receive<span style="color: #666666">(</span>recordableMessage<span style="color: #666666">)</span>
     logger<span style="color: #666666">.</span>info<span style="color: #666666">(</span>s<span style="color: #4070a0">&quot;done handling message: ${recordableMessage.messageName}&quot;</span><span style="color: #666666">)</span>
   <span style="color: #007020; font-weight: bold">case</span> message <span style="color: #007020; font-weight: bold">=&gt;</span> <span style="color: #007020; font-weight: bold">super</span><span style="color: #666666">.</span>receive<span style="color: #666666">(</span>message<span style="color: #666666">)</span>
 <span style="color: #666666">}</span>
<span style="color: #666666">}</span>
</pre></div>

Lastly, mix these traits to a concrete MyActor class

 <div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #007020; font-weight: bold">class</span> <span style="color: #0e84b5; font-weight: bold">MyMonitoredActor</span> <span style="color: #007020; font-weight: bold">extends</span> <span style="color: #0e84b5; font-weight: bold">MyActor</span> <span style="color: #007020; font-weight: bold">with</span> <span style="color: #0e84b5; font-weight: bold">LatencyRecorderActor</span> <span style="color: #007020; font-weight: bold">with</span> <span style="color: #0e84b5; font-weight: bold">LoggerActor</span>
 </pre></div>


### **Try it out**

Create a concrete RecordableMessage:

<div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #007020; font-weight: bold">case</span> <span style="color: #007020; font-weight: bold">class</span> <span style="color: #0e84b5; font-weight: bold">SomeRecordableMessage</span><span style="color: #666666">()</span> <span style="color: #007020; font-weight: bold">extends</span> <span style="color: #0e84b5; font-weight: bold">RecordableMessage</span> <span style="color: #666666">{</span>
   <span style="color: #007020; font-weight: bold">override</span> <span style="color: #007020; font-weight: bold">val</span> messageName<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">String</span> <span style="color: #666666">=</span>
   <span style="color: #007020; font-weight: bold">this</span><span style="color: #666666">.</span>getClass<span style="color: #666666">.</span>getSimpleName
<span style="color: #666666">}</span>
</pre></div>

And send it to a MonitoredActor instance

<!-- HTML generated using hilite.me --><div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #007020; font-weight: bold">val</span> actorSystem <span style="color: #007020; font-weight: bold">=</span> <span style="color: #0e84b5; font-weight: bold">ActorSystem</span><span style="color: #666666">(</span><span style="color: #4070a0">&quot;system&quot;</span><span style="color: #666666">)</span>
<span style="color: #007020; font-weight: bold">val</span> myMonitoredActor <span style="color: #007020; font-weight: bold">=</span> actorSystem<span style="color: #666666">.</span>actorOf<span style="color: #666666">(</span><span style="color: #0e84b5; font-weight: bold">Props</span><span style="color: #666666">[</span><span style="color: #902000">MyMonitoredActor</span><span style="color: #666666">])</span>
myMonitoredActor <span style="color: #666666">!</span> <span style="color: #0e84b5; font-weight: bold">SomeRecordableMessage</span><span style="color: #666666">()</span>
</pre></div>

Results with the following printed to the log:

````
handling message: SomeTriggerMessage
time-in-mailbox latency for message SomeTriggerMessage in actor MyMonitoredActor is 102
performing some work...
processing-time latency for message SomeTriggerMessage in actor MyMonitoredActor is 212
done handling message: SomeTriggerMessage
````

### **Wrapping up**

Stackable traits pattern is a good choice when you need to 'pipe' actions or modify and redirect data for an action. Mix and stack traits to describe the state of the class and execute the actions are clean and flexible, and generally the scala-functional way to do it.

<img src="/img/scala_devs.png">

------------------------------------------------------------------------------------------

*The complete source code can be found in my [github](https://github.com/FullGC/stackable-traits)*.

<div id="disqus_thread"></div>
<script>

/**
*  RECOMMENDED CONFIGURATION VARIABLES: EDIT AND UNCOMMENT THE SECTION BELOW TO INSERT DYNAMIC VALUES FROM YOUR PLATFORM OR CMS.
*  LEARN WHY DEFINING THESE VARIABLES IS IMPORTANT: https://disqus.com/admin/universalcode/#configuration-variables*/
var disqus_config = function () {
this.page.url = "https://fullgc.github.io/stackable-traits-pattern---part-2/"
this.page.identifier = stackable-1
};
(function() { // DON'T EDIT BELOW THIS LINE
var d = document, s = d.createElement('script');
s.src = 'https://FullGC.disqus.com/embed.js';
s.setAttribute('data-timestamp', +new Date());
(d.head || d.body).appendChild(s);
})();
</script>
<noscript>Please enable JavaScript to view the <a href="https://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>
