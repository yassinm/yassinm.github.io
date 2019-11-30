---
date: 2014-09-21
author: ["Yassin Mohamed"]
title: Walking down memory lane (part 1)

featured: ["top00"]
tags : ["memory"]
series : ["Walking down memory lane"]
---

One of the topics that you often have to deal with in java, especially in high performance and low latency distributed code, is how you send a set of objects to a worker thread in the most efficient manner. This thread could be running in the same process or on a completely separate machine located somewhere else in your network. This has always been the case for people coming from a c/c++ environment even before the advent of Java. However, Java has steadily gained so much in performance and JIT improvements over the years that nowadays it is on par , in some areas, with c/c++. This of course does not come free and requires a very careful design and a very good understanding of current IPC mechanisms. Moreover, some of the new techniques involving Off-heap memory allocations and lock free queues can even go further. As always, the devil is in the details. In this post i will try to first look at how you can pass data efficiently between different threads/processes and the different issues we have to solve when dealing with the java language.

## Serializing/de-serializing

If you want to pass around a block of memory in c/c++ all you need to do on the receiving side is cast the actual memory location to a particular structure and you immediately get an object you can play with. At your own risk of course! This scenario is similar to having all your belts off while driving a formula one vehicle and going the wrong way on a high speed lane … Other cars coming at you being the data that was sent to you. This is why you can refer to this practice as having “unsafe” relations with your data! On the flip side, this does not mean the code receiving the data has to be running within the same process. It could be running somewhere else in the universe as long as the actual data was transported successfully! You still have to take care of the normal Endian-ness of the platform, which is obviously also needed in Java, but that is all that’s required. Technically speaking, there is no need for serialization/de-serialization required since the actual objects are already in bytes format. Unfortunately, we do not have the luxury of these dangerous “Shoot my own feet features” practices in Java. That is, unless you turn yourself into the dark side by using the famously unsafe using sun.misc.Unsafe ! More on this later…

In java , when you pass objects around between threads within the same process you can get away by only passing the reference to the actual object. Obviously, you will have to take great care when the passed object is accessed concurrently across the multiple threads in your process. You can deal with issues like this by implementing one of the concurrent access mechanisms available in the JVM. The problem however comes when your target thread is running in a separate process. In that scenario you need a way of serializing/de-serializing the objects before and after they hit the wire. This ,dealing with objects back and forth, becomes very cumbersome suddenly.

The above additional complexity stems from the fact that we do not have (yet) a way to deal with arrays of structures in java as we do in c/c++. We basically can’t have “unsafe” relations with our data. All objects , including arrays of all types, are passed by reference. As such these objects must all live in the heap and are all susceptible for garbage collection. This basically means that anything you get from the wire has to be “transformed” into actual objects before they are ready for consumption. Lately, the growing majority experience this aspect in Java when dealing with Restful services. An object is first serialized into JSON , sent over the wire via a transport protocol, and then de-serialized when it arrives at its destination before it is passed to the upper application specific code. The simpler the object the easier it is to perform those steps and the faster the overall code becomes. I will not bore you with the multiple technologies available in that arena but i will point you to this comparison.

## Memory control

Unfortunately, there is no particular way one can instruct the JVM on how objects are stored and how they are placed in memory in contiguous or non contiguous fashion. As far as the general programmer is concerned these pesky details are what make the platform “safe”. As such, these details are considered irrelevant for the majority of use cases. This is exactly why it is not needed for a lot of users. The exception is when your code needs to deal with performance critical access to very large amount of objects and you need to have a good control on how these particular objects are stored in memory efficiently. .

Furthermore, the access pattern across different threads and cores accessing the actual fields in your objects can also be problematic. False sharing explained here and here is a very good case on why it is important to know your memory layout in a performance critical code. Again this is not for everyone! But you can still see why it is imperative to understand the concepts involved when you need to write performance critical code.

To make things even more complicated there is no “traditional” way in Java on how you access bytes directly as soon as they are off the network in a zero copy fashion. Once you get data in a NIC buffer the application has to go through a context switch before data is available in user land. The same thing happens when the bytes are on their way out. There are commercial libraries/hardware to do these but so far this not part of the things available out of the box and definitely not for “free” !

## Conclusion

You would think that someone has already worked out the details for most of these issues and came up with a clean and simplistic way , ala Corba style, for you to send data across different threads/processes and with high speed. Fortunately there are some outstanding libraries out there we will explore in our next articles, but you still have to understand these complexities yourself and you still need to be in the know. Ergo, the need for this first part covering the basics!

To conclude i will say that we need to figure a way to:

Turn objects to bytes (and vice versa) in a very high speed fashion
Make sure we have a good control on how the memory is been used even in java.
The above outlines only some of the issues i can think of right now. To me these issues constitute a list of requirements i would want my high speed serializing/de-serializing code to deal with. Do i hear a future framework in java in the horizon that would solve (most of) these issues … that’s a “definite maybe” so stay tuned!