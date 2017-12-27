---
title: Stackable Traits pattern - Part 1
layout: post
author: Dani Shemesh
permalink: /stackable-traits-pattern/
tags:
- scala
- stackable
- traits
source-id: 1Fx2TpNbL_lKIOBoy4Fni0NjlHa-vjb_J-r_v6gSMUck
published: true
date: 2017-12-08 14:40:45
---
*"Traits let you modify the methods of a class, and they do so in a way that allows you to stack those modifications with each other" (Programming in Scala by Martin Odersky)*

One of the reasons that Scala still feels to me as 'a better Java', is the power of its traits. Trait can be used for multiple inheritance as Java interface, as a rich interface with fields and a state, and to be mixed into a class.

An interesting behavior of traits, as opposed to classes, is the call for a super method. Call for a super method in classes is statically bound, means that the exact implementation that would be invoked is known upfront. In traits however it is dynamically bound. The term is usually refers to an invocation of a method in an object, where the implementation is decided on runtime, i.e. a polymorphic call. The super call is not defined when a trait defined, but only when it is mixed into a concrete class. This behavior allows us to 'stack' the traits, and used them the super call as ‘pipe’ and redirect output, quite similar to ‘pipe’ in linux. This is the basis for the use-cases we’ll review.

<img align="right" src="/img/stackable_narrative.png">
## **Error reporting design**

Consider the following:

An Ad-Server gets a request for an Advertisement from a mobile phone.

Here we'll discuss a solution for executing the actions that need to be taken when the Ad-Server encounter errors. There are 2 types of errors which should be handled appropriately as follows:

On **FatalError**:

1. The Logger *prints* to the Log

2. The Monitor *increments* the counter Metric.

3. Kafka-Producer *sends* a Json Event with a timestamp  to Kafka.

4. *when step 3 failed*, S3-Client *uploads* the Event as CSV with a timestamp to to S3 (backup)

On **InvalidRequestError**

1. The Logger *prints* to the Log

2. The Monitor *increments* the counter Metric.

3. Kafka-Producer *sends* a Json Event to Kafka.

An Error has a code, a description, and unique properties

<!-- HTML generated using hilite.me --><div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #007020; font-weight: bold">trait</span> <span style="color: #0e84b5; font-weight: bold">ServingError</span><span style="color: #666666">{</span>
 <span style="color: #007020; font-weight: bold">val</span> code<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">Int</span>
 <span style="color: #007020; font-weight: bold">val</span> description<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">String</span>
<span style="color: #666666">}</span>
<span style="color: #007020; font-weight: bold">case</span> <span style="color: #007020; font-weight: bold">class</span> <span style="color: #0e84b5; font-weight: bold">FatalError</span><span style="color: #666666">(</span><span style="color: #007020; font-weight: bold">override</span> <span style="color: #007020; font-weight: bold">val</span> code<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">Int</span> <span style="color: #666666">=</span> <span style="color: #40a070">1</span><span style="color: #666666">,</span> exceptionCause<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">String</span><span style="color: #666666">)</span> <span style="color: #007020; font-weight: bold">extends</span> <span style="color: #0e84b5; font-weight: bold">ServingError</span> <span style="color: #666666">{</span>
 <span style="color: #007020; font-weight: bold">override</span> <span style="color: #007020; font-weight: bold">val</span> description<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">String</span> <span style="color: #666666">=</span>  s<span style="color: #4070a0">&quot;Fatal Error code $code accrued</span>
<span style="color: #4070a0">}</span>
<span style="color: #4070a0">case class InvalidRequestError(paramName: String, paramValue: String, override val code: Int = 2) extends ServingError{</span>
<span style="color: #4070a0"> override val description: String = s&quot;</span><span style="color: #0e84b5; font-weight: bold">Invalid</span> <span style="color: #0e84b5; font-weight: bold">Request</span><span style="color: #666666">.</span> <span style="color: #0e84b5; font-weight: bold">Bad</span>  parameter<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">$paramName</span><span style="border: 1px solid #FF0000">&quot;</span> <span style="color: #902000">+</span> <span style="border: 1px solid #FF0000">&quot;</span> <span style="border: 1px solid #FF0000">&quot;</span> <span style="color: #902000">+</span> <span style="color: #902000">s</span><span style="border: 1px solid #FF0000">&quot;</span><span style="color: #902000">with</span> <span style="color: #902000">value:</span> <span style="color: #902000">$paramValue</span><span style="border: 1px solid #FF0000">&quot;</span>
<span style="color: #666666">}</span>
</pre></div>

And these are mocks for the Scala Clients

<!-- HTML generated using hilite.me --><div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #007020; font-weight: bold">object</span> <span style="color: #0e84b5; font-weight: bold">KafkaProducer</span><span style="color: #666666">{</span>
 <span style="color: #007020; font-weight: bold">def</span> send<span style="color: #666666">(</span>eventContent<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">String</span><span style="color: #666666">)</span><span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">Unit</span> <span style="color: #666666">=</span> println<span style="color: #666666">(</span>s<span style="color: #4070a0">&quot;sending to Kafka: $eventContent&quot;</span><span style="color: #666666">)</span>
<span style="color: #666666">}</span>
<span style="color: #007020; font-weight: bold">object</span> <span style="color: #0e84b5; font-weight: bold">Monitor</span><span style="color: #666666">{</span>
 <span style="color: #007020; font-weight: bold">def</span> incrementCounter<span style="color: #666666">(</span>counterName<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">String</span><span style="color: #666666">,</span> tags<span style="color: #007020; font-weight: bold">:</span> <span style="color: #666666">(</span><span style="color: #902000">String</span><span style="color: #666666">,</span> <span style="color: #902000">String</span><span style="color: #666666">))</span><span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">Unit</span> <span style="color: #666666">=</span> println<span style="color: #666666">(</span>s<span style="color: #4070a0">&quot;incrementing counter $counterName&quot;</span><span style="color: #666666">)</span>
<span style="color: #666666">}</span>
<span style="color: #007020; font-weight: bold">object</span> <span style="color: #0e84b5; font-weight: bold">Logger</span><span style="color: #666666">{</span>
 <span style="color: #007020; font-weight: bold">def</span> log<span style="color: #666666">(</span>event<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">String</span><span style="color: #666666">)</span><span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">Unit</span> <span style="color: #666666">=</span> println<span style="color: #666666">(</span>s<span style="color: #4070a0">&quot;error: $event&quot;</span><span style="color: #666666">)</span>
<span style="color: #666666">}</span>
<span style="color: #007020; font-weight: bold">object</span> <span style="color: #0e84b5; font-weight: bold">S3_Client</span> <span style="color: #666666">{</span>
 <span style="color: #007020; font-weight: bold">def</span> upload<span style="color: #666666">(</span>eventContent<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">String</span><span style="color: #666666">)</span><span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">Unit</span> <span style="color: #666666">=</span> println<span style="color: #666666">(</span>s<span style="color: #4070a0">&quot;uploading event $eventContent to S3&quot;</span><span style="color: #666666">)</span>
<span style="color: #666666">}</span>
</pre></div>


### **Step by step implementation, using stackable-traits**

##### Create traits that represent the subscribers and mix them.

We'll create the following traits, each represent a subscriber for an error event

<!-- HTML generated using hilite.me --><div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #007020; font-weight: bold">trait</span> <span style="color: #0e84b5; font-weight: bold">Log</span>
<span style="color: #007020; font-weight: bold">trait</span> <span style="color: #0e84b5; font-weight: bold">Metric</span>
<span style="color: #007020; font-weight: bold">trait</span> <span style="color: #0e84b5; font-weight: bold">S3_Backup</span>
<span style="color: #007020; font-weight: bold">trait</span> <span style="color: #0e84b5; font-weight: bold">Kafka</span>
</pre></div>

And mix them to the Error classes. Here, we use Scala type system to describe the Error classes, and the actions that need to be taken.

<!-- HTML generated using hilite.me --><div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #007020; font-weight: bold">trait</span> <span style="color: #0e84b5; font-weight: bold">ServingError</span><span style="color: #666666">{</span>
 <span style="color: #007020; font-weight: bold">val</span> code<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">Int</span>
 <span style="color: #007020; font-weight: bold">val</span> description<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">String</span>
<span style="color: #666666">}</span>
</pre></div>


##### Turn the traits into services that activates the Clients

Now it is clear which subscribers should receive a notification on error. We will enrich the traits so they would be able to activate the clients as well:

<!-- HTML generated using hilite.me --><div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #007020; font-weight: bold">abstract</span> <span style="color: #007020; font-weight: bold">class</span> <span style="color: #0e84b5; font-weight: bold">Sender</span> <span style="color: #666666">{</span>
 <span style="color: #007020; font-weight: bold">def</span> send<span style="color: #666666">(</span>event<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">ServingError</span><span style="color: #666666">)</span><span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">Unit</span>
<span style="color: #666666">}</span>
<span style="color: #007020; font-weight: bold">trait</span> <span style="color: #0e84b5; font-weight: bold">Log</span> <span style="color: #007020; font-weight: bold">extends</span> <span style="color: #0e84b5; font-weight: bold">Sender</span> <span style="color: #666666">{</span>
 <span style="color: #007020; font-weight: bold">override</span> <span style="color: #007020; font-weight: bold">def</span> send<span style="color: #666666">(</span>event<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">ServingError</span><span style="color: #666666">)</span><span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">Unit</span> <span style="color: #666666">=</span> <span style="color: #666666">{</span>
   <span style="color: #0e84b5; font-weight: bold">Logger</span><span style="color: #666666">.</span>log<span style="color: #666666">(</span>event<span style="color: #666666">.</span>description<span style="color: #666666">)</span>
 <span style="color: #666666">}</span>
<span style="color: #666666">}</span>
<span style="color: #007020; font-weight: bold">trait</span> <span style="color: #0e84b5; font-weight: bold">Metric</span> <span style="color: #007020; font-weight: bold">extends</span> <span style="color: #0e84b5; font-weight: bold">Sender</span> <span style="color: #666666">{</span>
 <span style="color: #007020; font-weight: bold">override</span> <span style="color: #007020; font-weight: bold">def</span> send<span style="color: #666666">(</span>event<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">ServingError</span><span style="color: #666666">)</span><span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">Unit</span> <span style="color: #666666">=</span> <span style="color: #666666">{</span>
   <span style="color: #0e84b5; font-weight: bold">Monitor</span><span style="color: #666666">.</span>incrementCounter<span style="color: #666666">(</span>event<span style="color: #666666">.</span>getClass<span style="color: #666666">.</span>getSimpleName<span style="color: #60a0b0; font-style: italic">/*this is not recommended...*/</span><span style="color: #666666">,</span> <span style="color: #4070a0">&quot;errorCode&quot;</span> <span style="color: #666666">-&gt;</span>
   event<span style="color: #666666">.</span>code<span style="color: #666666">.</span>toString<span style="color: #666666">)</span>
 <span style="color: #666666">}</span>
<span style="color: #666666">}</span>
<span style="color: #007020; font-weight: bold">trait</span> <span style="color: #0e84b5; font-weight: bold">S3_Backup</span> <span style="color: #007020; font-weight: bold">extends</span> <span style="color: #0e84b5; font-weight: bold">Sender</span> <span style="color: #666666">{</span>
 <span style="color: #007020; font-weight: bold">override</span> <span style="color: #007020; font-weight: bold">def</span> send<span style="color: #666666">(</span>event<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">ServingError</span><span style="color: #666666">)</span><span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">Unit</span> <span style="color: #666666">=</span> <span style="color: #666666">{</span>
   S3_Client<span style="color: #666666">.</span>upload<span style="color: #666666">(</span>event<span style="color: #666666">.</span>content<span style="color: #666666">)</span>
 <span style="color: #666666">}</span>
<span style="color: #666666">}</span>
<span style="color: #007020; font-weight: bold">trait</span> <span style="color: #0e84b5; font-weight: bold">Kafka</span> <span style="color: #007020; font-weight: bold">extends</span> <span style="color: #0e84b5; font-weight: bold">Sender</span> <span style="color: #666666">{</span>
 <span style="color: #007020; font-weight: bold">def</span> send<span style="color: #666666">(</span>event<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">ServingError</span><span style="color: #666666">)</span><span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">Unit</span> <span style="color: #666666">=</span> <span style="color: #666666">{</span>
   <span style="color: #0e84b5; font-weight: bold">KafkaProducer</span><span style="color: #666666">.</span>send<span style="color: #666666">(</span>event<span style="color: #666666">.</span>content<span style="color: #666666">)</span> <span style="color: #666666">}</span>
<span style="color: #666666">}</span>
</pre></div>



Alright, so now each trait have a "send" method that handles the event using the appropriate client.

But we still need to trigger it on error, and call them in the order described above

##### 'Stack' the services traits with each other

 I.e., Pipe of calls for the 'send' method in the services. As discussed, we’ll need to use super method calls for that.

Note that at this step we won't be stacking modifications, but the side-effects that triggered on error.

Alright so we need to stack the traits according to the order and logic defined in the narrative described above. The order of the invocation of the traits will be right to left.

Hence the order for FatalError: S3_Backup ← Kafka ← Monitor ← Log

Where S3_Backup should be invoked only when the Kafka-Producer failed to send the event to Kafka.

And for InvalidRequestError: Kafka ← Monitor ← Log

Let's re-arranged the mixing on the Error classes:

<!-- HTML generated using hilite.me --><div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #007020; font-weight: bold">case</span> <span style="color: #007020; font-weight: bold">class</span> <span style="color: #0e84b5; font-weight: bold">FatalError</span><span style="color: #666666">(</span><span style="color: #007020; font-weight: bold">override</span> <span style="color: #007020; font-weight: bold">val</span> code<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">Int</span> <span style="color: #666666">=</span> <span style="color: #40a070">1</span><span style="color: #666666">,</span> exceptionCause<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">String</span><span style="color: #666666">)</span> <span style="color: #007020; font-weight: bold">extends</span> <span style="color: #0e84b5; font-weight: bold">ServingError</span> <span style="color: #007020; font-weight: bold">with</span> S3_Backup <span style="color: #007020; font-weight: bold">with</span> <span style="color: #0e84b5; font-weight: bold">Kafka</span> <span style="color: #007020; font-weight: bold">with</span> <span style="color: #0e84b5; font-weight: bold">Monitor</span> <span style="color: #007020; font-weight: bold">with</span> <span style="color: #0e84b5; font-weight: bold">Log</span><span style="color: #666666">{</span>
 <span style="color: #007020; font-weight: bold">override</span> <span style="color: #007020; font-weight: bold">val</span> description<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">String</span> <span style="color: #666666">=</span> s<span style="color: #4070a0">&quot;Fatal Error code $code accrued&quot;</span>
<span style="color: #666666">}</span>
<span style="color: #007020; font-weight: bold">case</span> <span style="color: #007020; font-weight: bold">class</span> <span style="color: #0e84b5; font-weight: bold">InvalidRequestError</span><span style="color: #666666">(</span>paramName<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">String</span><span style="color: #666666">,</span> paramValue<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">String</span><span style="color: #666666">,</span> <span style="color: #007020; font-weight: bold">override</span> <span style="color: #007020; font-weight: bold">val</span> code<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">Int</span> <span style="color: #666666">=</span> <span style="color: #40a070">2</span><span style="color: #666666">)</span> <span style="color: #007020; font-weight: bold">extends</span> <span style="color: #0e84b5; font-weight: bold">ServingError</span> <span style="color: #007020; font-weight: bold">with</span> <span style="color: #0e84b5; font-weight: bold">Kafka</span> <span style="color: #007020; font-weight: bold">with</span> <span style="color: #0e84b5; font-weight: bold">Monitoring</span> <span style="color: #007020; font-weight: bold">with</span> <span style="color: #0e84b5; font-weight: bold">Log</span><span style="color: #666666">{</span>
 <span style="color: #007020; font-weight: bold">override</span> <span style="color: #007020; font-weight: bold">val</span> description<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">String</span> <span style="color: #666666">=</span> s<span style="color: #4070a0">&quot;Invalid Request. Bad  parameter: $paramName&quot;</span> <span style="color: #666666">+</span> <span style="color: #4070a0">&quot; &quot;</span> <span style="color: #666666">+</span> s<span style="color: #4070a0">&quot;with value: $paramValue&quot;</span>
<span style="color: #666666">}</span>
<span style="color: #007020; font-weight: bold">case</span> <span style="color: #007020; font-weight: bold">class</span> <span style="color: #0e84b5; font-weight: bold">FatalError</span><span style="color: #666666">(</span><span style="color: #007020; font-weight: bold">override</span> <span style="color: #007020; font-weight: bold">val</span> code<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">Int</span> <span style="color: #666666">=</span> <span style="color: #40a070">1</span><span style="color: #666666">,</span> exceptionCause<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">String</span><span style="color: #666666">)</span> <span style="color: #007020; font-weight: bold">extends</span> <span style="color: #0e84b5; font-weight: bold">ServingError</span> <span style="color: #007020; font-weight: bold">with</span> S3_Backup <span style="color: #007020; font-weight: bold">with</span> <span style="color: #0e84b5; font-weight: bold">Kafka</span> <span style="color: #007020; font-weight: bold">with</span> <span style="color: #0e84b5; font-weight: bold">Monitor</span> <span style="color: #007020; font-weight: bold">with</span> <span style="color: #0e84b5; font-weight: bold">Log</span><span style="color: #666666">{</span>
 <span style="color: #007020; font-weight: bold">override</span> <span style="color: #007020; font-weight: bold">val</span> description<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">String</span> <span style="color: #666666">=</span> s<span style="color: #4070a0">&quot;Fatal Error code $code accrued&quot;</span>
<span style="color: #666666">}</span>
<span style="color: #007020; font-weight: bold">case</span> <span style="color: #007020; font-weight: bold">class</span> <span style="color: #0e84b5; font-weight: bold">InvalidRequestError</span><span style="color: #666666">(</span>paramName<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">String</span><span style="color: #666666">,</span> paramValue<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">String</span><span style="color: #666666">,</span> <span style="color: #007020; font-weight: bold">override</span> <span style="color: #007020; font-weight: bold">val</span> code<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">Int</span> <span style="color: #666666">=</span> <span style="color: #40a070">2</span><span style="color: #666666">)</span> <span style="color: #007020; font-weight: bold">extends</span> <span style="color: #0e84b5; font-weight: bold">ServingError</span> <span style="color: #007020; font-weight: bold">with</span> <span style="color: #0e84b5; font-weight: bold">Kafka</span> <span style="color: #007020; font-weight: bold">with</span> <span style="color: #0e84b5; font-weight: bold">Monitoring</span> <span style="color: #007020; font-weight: bold">with</span> <span style="color: #0e84b5; font-weight: bold">Log</span><span style="color: #666666">{</span>
 <span style="color: #007020; font-weight: bold">override</span> <span style="color: #007020; font-weight: bold">val</span> description<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">String</span> <span style="color: #666666">=</span> s<span style="color: #4070a0">&quot;Invalid Request. Bad  parameter: $paramName&quot;</span> <span style="color: #666666">+</span> <span style="color: #4070a0">&quot; &quot;</span> <span style="color: #666666">+</span> s<span style="color: #4070a0">&quot;with value: $paramValue&quot;</span>
<span style="color: #666666">}</span>
</pre></div>

And stack the traits:

<!-- HTML generated using hilite.me --><div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #007020; font-weight: bold">trait</span> <span style="color: #0e84b5; font-weight: bold">Log</span> <span style="color: #007020; font-weight: bold">extends</span> <span style="color: #0e84b5; font-weight: bold">Sender</span> <span style="color: #666666">{</span>
 <span style="color: #007020; font-weight: bold">abstract</span> <span style="color: #007020; font-weight: bold">override</span> <span style="color: #007020; font-weight: bold">def</span> send<span style="color: #666666">(</span>event<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">ServingError</span><span style="color: #666666">)</span><span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">Unit</span> <span style="color: #666666">=</span> <span style="color: #666666">{</span>
   <span style="color: #0e84b5; font-weight: bold">Logger</span><span style="color: #666666">.</span>log<span style="color: #666666">(</span>event<span style="color: #666666">.</span>description<span style="color: #666666">)</span>
   <span style="color: #007020; font-weight: bold">super</span><span style="color: #666666">.</span>send<span style="color: #666666">(</span>event<span style="color: #666666">)</span>
 <span style="color: #666666">}</span>
<span style="color: #666666">}</span>
<span style="color: #007020; font-weight: bold">trait</span> <span style="color: #0e84b5; font-weight: bold">Metric</span> <span style="color: #007020; font-weight: bold">extends</span> <span style="color: #0e84b5; font-weight: bold">Sender</span> <span style="color: #666666">{</span>
 <span style="color: #007020; font-weight: bold">abstract</span> <span style="color: #007020; font-weight: bold">override</span> <span style="color: #007020; font-weight: bold">def</span> send<span style="color: #666666">(</span>event<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">ServingError</span><span style="color: #666666">)</span><span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">Unit</span> <span style="color: #666666">=</span> <span style="color: #666666">{</span>
   <span style="color: #0e84b5; font-weight: bold">Monitor</span><span style="color: #666666">.</span>incrementCounter<span style="color: #666666">(</span>event<span style="color: #666666">.</span>getClass<span style="color: #666666">.</span>getSimpleName<span style="color: #666666">,</span> <span style="color: #4070a0">&quot;errorCode&quot;</span> <span style="color: #666666">-&gt;</span> event<span style="color: #666666">.</span>code<span style="color: #666666">.</span>toString<span style="color: #666666">)</span>
   <span style="color: #007020; font-weight: bold">super</span><span style="color: #666666">.</span>send<span style="color: #666666">(</span>event<span style="color: #666666">)</span>
 <span style="color: #666666">}</span>
<span style="color: #666666">}</span>
<span style="color: #007020; font-weight: bold">trait</span> <span style="color: #0e84b5; font-weight: bold">S3_Backup</span> <span style="color: #007020; font-weight: bold">extends</span> <span style="color: #0e84b5; font-weight: bold">Sender</span> <span style="color: #666666">{</span>
 <span style="color: #007020; font-weight: bold">abstract</span> <span style="color: #007020; font-weight: bold">override</span> <span style="color: #007020; font-weight: bold">def</span> send<span style="color: #666666">(</span>event<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">ServingError</span><span style="color: #666666">)</span><span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">Unit</span> <span style="color: #666666">=</span> <span style="color: #666666">{</span>
   S3_Client<span style="color: #666666">.</span>upload<span style="color: #666666">(</span>event<span style="color: #666666">.</span>content<span style="color: #666666">)</span>
   <span style="color: #007020; font-weight: bold">super</span><span style="color: #666666">.</span>send<span style="color: #666666">(</span>event<span style="color: #666666">)</span>
 <span style="color: #666666">}</span>
<span style="color: #666666">}</span>
<span style="color: #007020; font-weight: bold">trait</span> <span style="color: #0e84b5; font-weight: bold">Kafka</span> <span style="color: #007020; font-weight: bold">extends</span> <span style="color: #0e84b5; font-weight: bold">Sender</span> <span style="color: #666666">{</span>
 <span style="color: #007020; font-weight: bold">abstract</span> <span style="color: #007020; font-weight: bold">override</span> <span style="color: #007020; font-weight: bold">def</span> send<span style="color: #666666">(</span>event<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">ServingError</span><span style="color: #666666">)</span><span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">Unit</span> <span style="color: #666666">=</span> <span style="color: #666666">{</span>
   <span style="color: #0e84b5; font-weight: bold">Try</span><span style="color: #666666">(</span><span style="color: #0e84b5; font-weight: bold">KafkaProducer</span><span style="color: #666666">.</span>send<span style="color: #666666">(</span>event<span style="color: #666666">.</span>content<span style="color: #666666">)).</span>getOrElse<span style="color: #666666">(</span><span style="color: #007020; font-weight: bold">super</span><span style="color: #666666">.</span>send<span style="color: #666666">(</span>event<span style="color: #666666">))</span>
 <span style="color: #666666">}</span>
<span style="color: #666666">}</span>
</pre></div>



You might notice that

1. The call 'super.send(event)' was added to ‘send’ method.

2. The 'override' modifier changed to *‘abstract override’*, because the send method now overrides the behavior but also calls for an abstract method with super.send. The modifier tells that this trait has to be mixed with a concrete class(later...).

    btw, If we'll omit the 'abstract’ from the modifier, we’ll get the following error:

   *"method send in class Sender is accessed from super. It may not be abstract unless it is overridden by a member declared 'abstract' and `override' super.send(event)"*

3. Kafka would call super, i.e. S3-Backup would be invoked only when KafkaProducer fails to deliver.

<img align="right" src="/img/stack-traits.jpg">

##### Create and 'Stack' modification traits:

The event's content that needs to be modified to be in the right format (json, csv) before sent as an input to KafkaProducer and S3. A Fatal error needs to be sent with a timestamp

Let's write the modification method and the stackable modification traits:

<!-- HTML generated using hilite.me --><div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #007020; font-weight: bold">object</span> <span style="color: #0e84b5; font-weight: bold">ServingError</span><span style="color: #666666">{</span>
<span style="color: #60a0b0; font-style: italic">// we are adding a method modification content method, to be used by the modification traits </span>
 <span style="color: #007020; font-weight: bold">def</span> modifyContent<span style="color: #666666">(</span>error<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">ServingError</span><span style="color: #666666">,</span> content<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">String</span><span style="color: #666666">)</span><span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">ServingError</span> <span style="color: #666666">=</span> <span style="color: #666666">{</span>
   error <span style="color: #007020; font-weight: bold">match</span> <span style="color: #666666">{</span>
     <span style="color: #007020; font-weight: bold">case</span> f<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">FatalError</span> <span style="color: #666666">=&gt;</span> <span style="color: #0e84b5; font-weight: bold">FatalError</span><span style="color: #666666">(</span>exceptionCause <span style="color: #007020; font-weight: bold">=</span> f<span style="color: #666666">.</span>exceptionCause<span style="color: #666666">,</span> content <span style="color: #007020; font-weight: bold">=</span> content<span style="color: #666666">)</span>
     <span style="color: #007020; font-weight: bold">case</span> i<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">InvalidRequestError</span> <span style="color: #666666">=&gt;</span> <span style="color: #0e84b5; font-weight: bold">InvalidRequestError</span><span style="color: #666666">(</span>i<span style="color: #666666">.</span>paramName<span style="color: #666666">,</span> i<span style="color: #666666">.</span>paramValue<span style="color: #666666">,</span> content <span style="color: #007020; font-weight: bold">=</span> content<span style="color: #666666">)</span>
   <span style="color: #666666">}</span>
 <span style="color: #666666">}</span>
<span style="color: #666666">}</span>

<span style="color: #007020; font-weight: bold">trait</span> <span style="color: #0e84b5; font-weight: bold">JsonTransformer</span> <span style="color: #007020; font-weight: bold">extends</span> <span style="color: #0e84b5; font-weight: bold">Sender</span> <span style="color: #666666">{</span>
 <span style="color: #007020; font-weight: bold">import</span> <span style="color: #0e84b5; font-weight: bold">ServingError._</span>
 <span style="color: #007020; font-weight: bold">abstract</span> <span style="color: #007020; font-weight: bold">override</span> <span style="color: #007020; font-weight: bold">def</span> send<span style="color: #666666">(</span>event<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">ServingError</span><span style="color: #666666">)</span><span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">Unit</span> <span style="color: #666666">=</span> <span style="color: #666666">{</span>
   <span style="color: #007020; font-weight: bold">super</span><span style="color: #666666">.</span>send<span style="color: #666666">(</span>modifyContent<span style="color: #666666">(</span>event<span style="color: #666666">,</span> <span style="color: #666666">{</span>
     <span style="color: #007020; font-weight: bold">val</span> date <span style="color: #007020; font-weight: bold">=</span> event <span style="color: #007020; font-weight: bold">match</span> <span style="color: #666666">{</span>
       <span style="color: #007020; font-weight: bold">case</span> d<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">DateTimestampedMessage</span> <span style="color: #666666">=&gt;</span> <span style="color: #4070a0">&quot;\&quot;date\&quot;:&quot;</span> <span style="color: #666666">+</span> d<span style="color: #666666">.</span>date
       <span style="color: #007020; font-weight: bold">case</span> <span style="color: #007020; font-weight: bold">_</span> <span style="color: #007020; font-weight: bold">=&gt;</span> <span style="color: #4070a0">&quot;&quot;</span>
     <span style="color: #666666">}</span>
     <span style="color: #007020; font-weight: bold">val</span> cause <span style="color: #007020; font-weight: bold">=</span> event <span style="color: #007020; font-weight: bold">match</span> <span style="color: #666666">{</span>
       <span style="color: #007020; font-weight: bold">case</span> f<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">FatalError</span> <span style="color: #666666">=&gt;</span> <span style="color: #4070a0">&quot;\&quot;cause\&quot;:&quot;</span> <span style="color: #666666">+</span> f<span style="color: #666666">.</span>exceptionCause
       <span style="color: #007020; font-weight: bold">case</span> i<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">InvalidRequestError</span> <span style="color: #666666">=&gt;</span> <span style="color: #4070a0">&quot;\&quot;cause\&quot;:&quot;</span> <span style="color: #666666">+</span> i<span style="color: #666666">.</span>paramName <span style="color: #666666">+</span><span style="color: #4070a0">&quot;:&quot;</span> <span style="color: #666666">+</span>i<span style="color: #666666">.</span>paramValue
     <span style="color: #666666">}</span>
     s<span style="color: #4070a0">&quot;&quot;&quot;</span>
<span style="color: #4070a0">             {</span>
<span style="color: #4070a0">                &quot;code:&quot;${event.code},</span>
<span style="color: #4070a0">                $date</span>
<span style="color: #4070a0">                $cause</span>
<span style="color: #4070a0">             }</span>
<span style="color: #4070a0">      &quot;&quot;&quot;</span>
   <span style="color: #666666">}))</span>
 <span style="color: #666666">}</span>
<span style="color: #666666">}</span>
<span style="color: #007020; font-weight: bold">trait</span> <span style="color: #0e84b5; font-weight: bold">CsVTransformer</span> <span style="color: #007020; font-weight: bold">extends</span> <span style="color: #0e84b5; font-weight: bold">Sender</span> <span style="color: #666666">{</span>
 <span style="color: #007020; font-weight: bold">import</span> <span style="color: #0e84b5; font-weight: bold">ServingError._</span>
 <span style="color: #007020; font-weight: bold">abstract</span> <span style="color: #007020; font-weight: bold">override</span> <span style="color: #007020; font-weight: bold">def</span> send<span style="color: #666666">(</span>event<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">ServingError</span><span style="color: #666666">)</span><span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">Unit</span> <span style="color: #666666">=</span> <span style="color: #666666">{</span>
   <span style="color: #007020; font-weight: bold">super</span><span style="color: #666666">.</span>send<span style="color: #666666">(</span>modifyContent<span style="color: #666666">(</span>event<span style="color: #666666">,</span> <span style="color: #666666">{</span>
     <span style="color: #007020; font-weight: bold">val</span> date <span style="color: #007020; font-weight: bold">=</span> event <span style="color: #007020; font-weight: bold">match</span> <span style="color: #666666">{</span>
       <span style="color: #007020; font-weight: bold">case</span> d<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">DateTimestampedMessage</span> <span style="color: #666666">=&gt;</span> d<span style="color: #666666">.</span>date
       <span style="color: #007020; font-weight: bold">case</span> <span style="color: #007020; font-weight: bold">_</span> <span style="color: #007020; font-weight: bold">=&gt;</span> <span style="color: #4070a0">&quot;&quot;</span>
     <span style="color: #666666">}</span>
     <span style="color: #007020; font-weight: bold">val</span> cause <span style="color: #007020; font-weight: bold">=</span> event <span style="color: #007020; font-weight: bold">match</span> <span style="color: #666666">{</span>
       <span style="color: #007020; font-weight: bold">case</span> f<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">FatalError</span> <span style="color: #666666">=&gt;</span> f<span style="color: #666666">.</span>exceptionCause
       <span style="color: #007020; font-weight: bold">case</span> i<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">InvalidRequestError</span> <span style="color: #666666">=&gt;</span> i<span style="color: #666666">.</span>paramName <span style="color: #666666">+</span><span style="color: #4070a0">&quot;,&quot;</span> <span style="color: #666666">+</span>i<span style="color: #666666">.</span>paramValue
     <span style="color: #666666">}</span>
     s<span style="color: #4070a0">&quot;&quot;&quot;${event.code},$date$cause&quot;&quot;&quot;</span>
   <span style="color: #666666">}))</span>
 <span style="color: #666666">}</span>
<span style="color: #666666">}</span>

<span style="color: #007020; font-weight: bold">trait</span> <span style="color: #0e84b5; font-weight: bold">Timestamp</span> <span style="color: #666666">{</span>
 <span style="color: #007020; font-weight: bold">val</span> date<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">Date</span> <span style="color: #666666">=</span> <span style="color: #007020; font-weight: bold">new</span> <span style="color: #0e84b5; font-weight: bold">Date</span><span style="color: #666666">()</span>
<span style="color: #666666">}</span>
</pre></div>

And mix the modification traits to the error classes

<!-- HTML generated using hilite.me --><div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #007020; font-weight: bold">case</span> <span style="color: #007020; font-weight: bold">class</span> <span style="color: #0e84b5; font-weight: bold">FatalError</span><span style="color: #666666">(</span><span style="color: #007020; font-weight: bold">override</span> <span style="color: #007020; font-weight: bold">val</span> code<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">Int</span> <span style="color: #666666">=</span> <span style="color: #40a070">1</span><span style="color: #666666">,</span> exceptionCause<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">String</span><span style="color: #666666">,</span> content<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">String</span> <span style="color: #666666">=</span> <span style="color: #4070a0">&quot;&quot;</span><span style="color: #666666">)</span> <span style="color: #007020; font-weight: bold">extends</span> <span style="color: #0e84b5; font-weight: bold">ServingError</span> <span style="color: #007020; font-weight: bold">with</span> S3_Backup <span style="color: #007020; font-weight: bold">with</span> <span style="color: #0e84b5; font-weight: bold">CsVTransformer</span> <span style="color: #007020; font-weight: bold">with</span> <span style="color: #0e84b5; font-weight: bold">Kafka</span> <span style="color: #007020; font-weight: bold">with</span> <span style="color: #0e84b5; font-weight: bold">JsonTransformer</span> <span style="color: #007020; font-weight: bold">with</span> <span style="color: #0e84b5; font-weight: bold">Timestamp</span> <span style="color: #007020; font-weight: bold">with</span> <span style="color: #0e84b5; font-weight: bold">Metric</span> <span style="color: #007020; font-weight: bold">with</span> <span style="color: #0e84b5; font-weight: bold">Log</span><span style="color: #666666">{</span>
 <span style="color: #007020; font-weight: bold">override</span> <span style="color: #007020; font-weight: bold">val</span> description<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">String</span> <span style="color: #666666">=</span> s<span style="color: #4070a0">&quot;Fatal Error code $code accrued&quot;</span>
<span style="color: #666666">}</span>
<span style="color: #007020; font-weight: bold">case</span> <span style="color: #007020; font-weight: bold">class</span> <span style="color: #0e84b5; font-weight: bold">InvalidRequestError</span><span style="color: #666666">(</span>paramName<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">String</span><span style="color: #666666">,</span> paramValue<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">String</span><span style="color: #666666">,</span> <span style="color: #007020; font-weight: bold">override</span> <span style="color: #007020; font-weight: bold">val</span> code<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">Int</span> <span style="color: #666666">=</span> <span style="color: #40a070">2</span><span style="color: #666666">,</span> content<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">String</span> <span style="color: #666666">=</span> <span style="color: #4070a0">&quot;&quot;</span><span style="color: #666666">)</span> <span style="color: #007020; font-weight: bold">extends</span> <span style="color: #0e84b5; font-weight: bold">ServingError</span> <span style="color: #007020; font-weight: bold">with</span> <span style="color: #0e84b5; font-weight: bold">Kafka</span> <span style="color: #007020; font-weight: bold">with</span> <span style="color: #0e84b5; font-weight: bold">JsonTransformer</span> <span style="color: #007020; font-weight: bold">with</span> <span style="color: #0e84b5; font-weight: bold">Log</span><span style="color: #666666">{</span>
 <span style="color: #007020; font-weight: bold">override</span> <span style="color: #007020; font-weight: bold">val</span> description<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">String</span> <span style="color: #666666">=</span> s<span style="color: #4070a0">&quot;Invalid Request. Bad  parameter: $paramName&quot;</span> <span style="color: #666666">+</span> <span style="color: #4070a0">&quot; &quot;</span> <span style="color: #666666">+</span> s<span style="color: #4070a0">&quot;with value: $paramValue&quot;</span>
<span style="color: #666666">}</span>
</pre></div>

##### Create and mix 'ServingErrorSender' :

Last, in order to trigger the error reports once an error object is created, we'll mix them with a sending trait.

<!-- HTML generated using hilite.me --><div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #007020; font-weight: bold">trait</span> <span style="color: #0e84b5; font-weight: bold">ServingErrorSender</span> <span style="color: #007020; font-weight: bold">extends</span> <span style="color: #0e84b5; font-weight: bold">Sender</span><span style="color: #666666">{</span>
 <span style="color: #007020; font-weight: bold">this:</span> <span style="color: #902000">ServingError</span> <span style="color: #666666">=&gt;</span>   <span style="color: #60a0b0; font-style: italic">// force to be mixed with a ServingError class, (look for 'see cake-pattern')</span>
 <span style="color: #007020; font-weight: bold">def</span> send<span style="color: #666666">()</span><span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">Unit</span> <span style="color: #666666">=</span> send<span style="color: #666666">(</span><span style="color: #007020; font-weight: bold">this</span><span style="color: #666666">)</span>
 <span style="color: #007020; font-weight: bold">override</span> <span style="color: #007020; font-weight: bold">def</span> send<span style="color: #666666">(</span>event<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">ServingError</span><span style="color: #666666">)</span><span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">Unit</span> <span style="color: #666666">=</span> <span style="color: #4070a0">&quot;&quot;</span>
<span style="color: #666666">}</span>
</pre></div>


<!-- HTML generated using hilite.me --><div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #007020; font-weight: bold">trait</span> <span style="color: #0e84b5; font-weight: bold">ServingError</span> <span style="color: #007020; font-weight: bold">extends</span> <span style="color: #0e84b5; font-weight: bold">ServingErrorSender</span><span style="color: #666666">{</span>
 <span style="color: #007020; font-weight: bold">val</span> code<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">Int</span>
 <span style="color: #007020; font-weight: bold">val</span> description<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">String</span>
 <span style="color: #007020; font-weight: bold">val</span> content<span style="color: #007020; font-weight: bold">:</span> <span style="color: #902000">String</span>
<span style="color: #666666">}</span>
</pre></div>

### **Try it out**

We have completed the task!

Now, when invoking a 'send'’ for an error, like the following:

<!-- HTML generated using hilite.me --><div style="background: #f0f0f0; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0e84b5; font-weight: bold">FatalError</span><span style="color: #666666">(</span>exceptionCause <span style="color: #007020; font-weight: bold">=</span> <span style="color: #007020; font-weight: bold">new</span> <span style="color: #0e84b5; font-weight: bold">IllegalArgumentException</span><span style="color: #666666">().</span>getClass<span style="color: #666666">.</span>getSimpleName<span style="color: #666666">).</span>send<span style="color: #666666">()</span>
</pre></div>

Results with the following printed to the log:

````
error: Fatal Error code 1 accrued //Log
incrementing counter FatalError // Metric
sending to Kafka:  // Kafka
{
  "code:"1,
  "date":Thu Nov 30 16:56:33 IST 2017
  "cause":IllegalArgumentException
}
````

### **Next**

In [part-2](https://fullgc.github.io/stackable-traits-pattern---part-2/) we'll use stackable-actor traits for gathering actor’s metrics.

### **References**

*[Programming in Scala(chapter 12.5) / Martin Odersky and co](https://www.artima.com/shop/programming_in_scala_3ed)*

------------------------------------------------------------------------------------------

*The complete source code and more running examples can be found in my [github](https://github.com/FullGC/stackable-traits)*.

<div id="disqus_thread"></div>
<script>

/**
*  RECOMMENDED CONFIGURATION VARIABLES: EDIT AND UNCOMMENT THE SECTION BELOW TO INSERT DYNAMIC VALUES FROM YOUR PLATFORM OR CMS.
*  LEARN WHY DEFINING THESE VARIABLES IS IMPORTANT: https://disqus.com/admin/universalcode/#configuration-variables*/
var disqus_config = function () {
this.page.url = "https://fullgc.github.io/stackable-traits-pattern/"
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
