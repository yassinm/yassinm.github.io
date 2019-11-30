---
date: 2014-09-25
author: ["Yassin Mohamed"]
title: Walking down memory lane (part 2)

featured: ["top00"]
tags : ["memory"]
series : ["Walking down memory lane"]
---

As per our previous [article](/2014/walking-down-memory-lane-01) we identified that we need a way to serialize objects into memory in a very efficient manner. To recap here is a picture showing the steps we need to remove when sending/receiving objects over the network.

![Fig1](/images/walking-down-memory-lane-02-000.png )

FTR we are still not interested in solving how these troublesome bytes are transferred over the network from one process to another once the data is ready to be transferred. For this article we are leaving this aspect aside and will only concentrate on efficiently storing those objects in memory and manipulating them correctly.

## Unsafe relationships with data

Like we said earlier java has a dark side if you want to really start fiddling with bytes in an unsafe and unprotected manner. This is called (quite rightly) sun.misc.Unsafe. For the gory details i will refer you to this [article](https://mechanical-sympathy.blogspot.ca/2012/07/native-cc-like-performance-for-java.html). What we gain immediately by using this unsafe method are :

We finally have a way of accessing bytes for read and writes purposes directly just like we used to in c/c++. We can also do so very efficiently and in a very high speed manner.
The code using these objects is not aware of what is happening behind the scene.
For the sake of cache coherency, it is much better to access the memory in a single stride fashion so that we are not jumping back and forth in memory.
Hiding behind an interface

## Hiding behind an interface

What we achieved, instantly, by using the “unsafe” method is a read/write through interface. The block of memory behind the object is effectively acting as a cache that we use to read/write behind the scene. The fact that we are dealing with unsafe is only an implementation specific issue. All the code using this interface is unaware of what is happening.

![Fig2](/images/walking-down-memory-lane-02-001.png )


This effectively means that the memory backing these objects could come from anywhere in the system ! We are not limited by the limitations of heap memory and its cumbersome garbage collection schemes. The gates are open for an infinite amount of unmanaged off-heap memory ! In fact, we are not even limited by the memory available on the process altogether. We could use a shared memory scheme where all the data could be read/written by a separate locally running process. Or for that matter anything that was published by the kernel in a memory location somewhere on the system !!! As long as we can address the memory in a progressive single stride fashion we can effectively address anything located locally or somewhere else on the universe. More on this later …

## Sharing is messaging !

As we said above the memory that is living behind the proxy object could also be a shared memory location. As long as we take care of how we are accessing these memory locations concurrently we can effectively send messages between local threads running within the same process !

![Fig3](/images/walking-down-memory-lane-02-002.png )

In the same vein it is not a big stretch to see how this technique could be applied to threads running across a multitude of processes running within the same machine or running on different geographically located data centres as part of a big cluster. It is only a matter of making sure the objects see the exact same data copied verbatim across those process boundaries. The biggest technical hurdle that is left to overcome is the synchronization mechanism we will use to make sure we are accessing these objects safely. Of course it will also have to be a very fast synchronization mechanism or we will lose the benefit of doing all these magical memory fiddling contraptions!

Furthermore, if we keep accessing memory with the Single Writer Principle in mind we might be able to share this memory location with more than one reader in a non contention based fashion. To quote that page ‘On x86/x64 loads can be re-ordered with older stores according to the memory model so memory barriers are required when multiple threads mutate the same data across cores. The single writer principle avoids this issue because it never has to deal with writing the latest version of a data item that may have been written by another thread and currently in the store buffer of another core”. But let us not go ahead of ourself here, we are not there yet !

One thing that you (maybe) missed to notice is that i completely ignored talking about how we were going to turn our objects to memory bytes via the serialization step which we set out to fix in the beginning of this article. As you can see that step is unnecessary since the code using our fine grained fly weight proxy object is directly reading and writing to the bytes locations. As such, when the user code calls the set functions on the proxy object we are technically performing the serialization to memory right away. Vice versa, when the user calls the get functions available on the proxy object they are effectively de-serializing the object from memory. All of the above is done efficiently and with minimal locking involved … Et voila !

## Do you know the memory man ? the memory man ? who lives on Drury Lane ?

Now one more little titbit remaining is that the process or thread we are sharing our data with does not have to be written in java at all. It could be written in any language binding that can use our proxy object. It can be in Java,c#,c/c++,erlang, ruby,nodejs, python , php … you name it. It just does not matter at all. Like we said before ,for all intents and purposes , it could be coming from the kernel itself. As long as the memory representation and location is followed we can both communicate over this high speed channel. Imagine a world where you could write your database access in Java and your number crunching in c/c++. And they could be both living within the same process or across from two separate data centers !

## Conclusion

We started with solving a simple serialization/de-serialization problem and we are already talking about fixing the inherent communication problems when people want to mix code written for different platforms and with different languages without sacrificing performance. I would say that this side effect is a very beautiful addition to solving our original problem(s). Assuming it works out … obviously !

Until we get there i can always say that “I have a dream … I have a dream that one day down in a process — with its vicious language base discrimination , with its OS having its lips dripping with the words of interposition and nullification — one day right there in a process, little java boys and c# girls will be able to join hands with little c/c++ boys and ruby girls as sisters and brothers. I have a dream that one day every valley shall be exalted, and every hill and mountain shall be made low. The rough places will be plain and the crooked places will be made straight, and the glory of all languages shall be revealed”

So stay tuned for that dream to come true !