---
draft: true
date: 2014-10-06
title: Wiring data transformations for speed (part 3)
author: ["Yassin Mohamed"]
---

## Let there be lambdas .. and lambdas there was !


Not only design an API but also deal with idiosyncrasies of golang 

## State of the union !

when i want to think about data processing platforms/run-times

Take some time and let the above quote sink in a bit. 

If the above sounds similar to the notion of ! 

Suffice to say that we wanted to draw your attention to one thing and one thing only. 

In this article we will try to cover how we can strip some of our data transformations from unnecessary concerns. The more a data streaming platforms is close to the original ideas in lambda calculus the more i find it friendlier and approachable! The more our data transformation is close to the original ideas in lambda calculus the more it becomes friendlier and approachable by even novice programmers !


We are going to introduce an approach based on this simple principle. We will lay the ground work first of why we think it is important to remove unnecessary concerns from our data transformations and bring them to their very basics. This will inherently allow us to wire data transformations and achieve maximum speed. The point of this article is to make you, as the reader, think *Yes we can* by the end your read !


Unfortunately we all had to suffer the terrible mistakes that plagued our industry with things like OO programming that blurred these pure notions of functional programming and mixed them with impure and filthy state! And yes i mean it when i say "impure" ! We can say that we are slowly recovering from these mistakes in today's large streaming platforms but we are still a long way from the promised land of functional programming. 

As a guiding principle of what i consider the most basic way to think about a data processing platform i always keep this great quote from [Mike Acton "Data-Oriented Design and C++"](https://www.youtube.com/watch?v=rX0ItVEVjHc) in my mind:

Usually when we think about high speed serialization we are thinking about the fastest way to move data between two separate systems. Unfortunately, serialization does not happen between two systems only. It is also important to think about how the same data can travel freely not only between threads in the same process but also across process boundaries. If you read our previous articles you will know that we always have a secret wish of being able to move data across langugages boundaries as well but still at the same very high speed. 


Lately hoewever, i have been exposed to data streaming platforms like [spark](https://spark.apache.org/) , [Apache Flink](https://flink.apache.org/) and [Kafka streams](https://www.confluent.io/blog/introducing-kafka-streams-stream-processing-made-simple/). I have to admit that even if the approach is still bringing value to the table it is still (in my humble opinion) not simplified enough that a simple person can grasp what is going on. I also feel that there is so much boiler plate involved that once you decide to go one way you are basically stuck with that decision and you spend your life trying to prove to yourself (and your boss) that whatever platform you picked is actually the best thing since sliced bread :) This is why google felt compelled to invent [another layer](https://beam.apache.org/) on top of these platforms so that they can move people away from a particular platform if they so wish to do so.



It goes without saying that at some point we need to deal with that state and decide how we are going to store it eventually! 

Now, obviously when we are talking about data transformations we need to keep that data around and store it somewhere. This indeed is a very interesting subject, specially with all the research going about databases systems,  but i will not talk about it for now. Maybe one day i will gather the time and energy to talk about my experiences in this domain but for now i happen to have a more interesting subject to tinker with. Speaking of storage systems i recently came to appreciate the advantages of storing your complete state in a distributed log system like [kafka](http://kafka.apache.org/) which is becoming more prevalent these days. Specially in the big data industry (my current employer is in this arena) where you cannot expect normal off the shelf yesterday's era storage systems (read the Oracle,DB2 of the world) to deal with that type of load and scale in terms of data. Assuming you are not a government agency with loads of wad to spare ... of course !

This data can come from anywhere in our system. It could be the output of a previous transformation or it could very well be something that was stored in our multitude of storage systems. We actually do not care ! All we care about is the actual data ! To introduce this world of pure lambdas mixed with data we are going to start with what the system looks like from a fifty thousand feet view.

What if we could draw this graph via flow based tool and 



We also claim that this flow graph based approach is actually better for performance in the long run.

There is not much we could do to tell it we actually wanted it to do something else. For example we could not switch to an FFI (Foreign function interface) via a configuration flag. 

These questions remain: "What is the cost of solving problems the way we do today?" Are we actually solving things or are we just introducing more problems ? What can we improve ?



We also want to postpone (purposefully) few things to a the runtime that will take care of these concerns. Namely:

* data/state ownership and control 
* data/state storage
* data/state transports 

This fictional runtime can be as simple or as complicated as we want. Suffice to say that by not dealing with these concerns in our transformations we are actually free from them ! We still need to show that this fictional runtime is still capable to introduce (more like inject) these concerns at a later stage !

 This is usually the time the dream of just interacting with your underlying transformations from a high level view with absolutely no clutter starts to show up in the radar! If all you had to deal with was this high level view of the system do you think you would still need a ton of explanation and documentations to go with this ? Our answer is ... not Really !


What this basically means is that getting to know our data actually helps us understand our problem. Our transformations are all dealing with data and moving it from one state to another. 

[1] *insert "fun" story about a fiber nick card failing in an Oracle system that was  connected to a storage system *

  ## Existing lambda platforms !

  We all had to suffer the terrible mistakes that plagued our industry with things like OO programming that blurred the notions of functional programming and mixed them with impure and filthy state! And yes i mean it when i say "impure" ! Luckily all hope is not lost and we are slowly recovering from these mistakes in today's run time platforms. But we are still a long way from the promised land of pure functional programming.

  As an example, recent hype in functional programming has slowly been rediscovered in programming paradigms like [reactive Streams](http://www.reactive-streams.org/) or even the new [JDK 9 Flow API](https://community.oracle.com/docs/DOC-1006738). I have also been exposed lately to data streaming platforms like [spark](https://spark.apache.org/) , [Apache Flink](https://flink.apache.org/) and [Kafka streams](https://www.confluent.io/blog/introducing-kafka-streams-stream-processing-made-simple/).

  You cannot miss but notice a recurring theme in all the system above. First of all, the run time is expected to deal with a lot of details like the scheduling and threading control we mentioned in our previous article. This removes a lot concerns already from the person writing those functions.

  However, there is still a huge amount of boiler code involved that once you decide to go one way you are basically stuck with that decision and you spend your life trying to prove to yourself (and your boss) that whatever platform you picked was actually the best thing since sliced bread :) If you look under the covers in all these platforms you will still find APIs that are trying to express functional programming paradigms but with a language medium that clearly has idioms and idiosyncrasies from the object oriented world. I am looking at you Java !!! Not to mention, it is always easy to add some extra state in the mix ... because why not!

  Suffice to say that from a portability point of view we still have  a long way to go !
