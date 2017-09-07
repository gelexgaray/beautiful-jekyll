---
layout: post
title: "WCF, MSMQ and messages disordered on sending"
date: 2017-09-07 14:24:00 +0200
comments: true
tags: ["WCF", ".net"]
published: true
---
If you have worked with transactional MSMQ, there is something probably you have heard a lot of times: Messages sent from computer A to computer B are always delivered in the same order they were sent.

Well... this is not totally true: sometimes this messages can be unexpectedly disordered. Don´t loose your faith... it is totally normal, at least if WCF is playing its role in this scenario.

## Messages disordered on receiving
If you have played a little with WCF and MetMSMQ endpoints, this should be known for you: There is a behaviour called Throttling that controls how many parallel threads are going to process your messages. 

Messages are read in order, but whenever this throttling goes beyond 1, they are processed using lots of threads. In this parallel processing mode, threads can be disordered while running. Nothing new till here.

## Messages disordered on sending
The bogus thing comes when you use a throttling of 1, and produce and consume messages on the same machine. Both sending and receiving are performed on a single user thread. Probably you thought, like me, that messages should be processed in order... but that's what it really happens. 

Why?

### Asyncronous by default 
This is due to the asyncronous nature of WCF message sending. Let me explain: in WCF, messages are sent in an **asynchronous way by default**... and this is where the disorder comes into place.

There is a behaviour to change this:

* **transactedBatching**: This behaviour controls the asynchronous batch size used for sending operations. By using a batch size of 1, all messages will be sent synchronously

When transactedBatching is set to a batch size greater than 1, everything you enqueue will go to a buffer, and this buffer will be committed to a thread pool whenever it is full or there is a "silent time" of 20% of the total transaction duration in this sending process. Here is where the dissordering comes into place.

```
<system.serviceModel>
  <behaviors>
    <endpointBehaviors>
      <behavior name="Client Behavior">
        <synchronousReceive />
        <transactedBatching maxBatchSize="1" />
      </behavior>
    </endpointBehaviors>
  </behaviors>
  <client>
    <endpoint address="net.msmq://localhost/private/DestinationQueue" behaviorConfiguration="Client Behavior" binding="netMsmqBinding" bindingConfiguration="netMsmqBindingConfig" contract="ProxyClass" name="EndpointName" />
  </client>
</system.serviceModel>
```

### More on this topic
<https://docs.microsoft.com/en-us/dotnet/framework/wcf/feature-details/batching-messages-in-a-transaction>

## That's all, folks!
See you in the next post!
