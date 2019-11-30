---
date: 2017-09-10 11:00:00
title: Wiring data transformations for speed (part 2)
author: ["Yassin Mohamed"]

# featured: ["top01"]
tags : ["dataflow"]
series : ["Wiring data transformations for speed"]
---

If you remember our previous [article](/07/09/2017/wiring-data-transformations-for-speed-01) we highlighted some of the inherent complexities we introduce by writing code the way we do today. We also stopped at a very few high level because frankly speaking this was going to be a very long read if we did not stop ranting about the current state of affairs :((

In that article, we came to the conclusion that maybe we needed a way to remove those unnecessary complexities and deal with them outside the data flow graph to achieve maximum flexibility and speed. As the famous quote goes:

>Perfection is achieved, not when there is nothing more to add, but when there is nothing left to take away.  *Antoine de Saint-Exupéry*, **Airman's Odyssey**

In this article we will try to address `why` removing those *pesky details* from our data transformations can not only simplify our development life greatly but also open the door to more optimizations. We want to show here `what` can be achieved by deferring those concerns and `what` kind of performance optimizations can bolted on the system at a later stage because of the simplifications we introduced in our system.

## Show me the data (flow) !

Going back to our previous [article](/2017/wiring-data-transformations-for-speed-01) here is the graph we were dealing with:

![Fig1](/images/wiring-data-transformations-for-speed-001.png )

The `multiply` method takes two arguments here as input and produces one output. The `add` method also takes two arguments and produces one output.

The first important thing we are going to do is to start thinking about our arguments (be it for input or output) as data. The idea here is that we want to focus wiring data for transformations only and only deal with defining the flow graph of how data is converted. If we take the above graph and only concentrate on the data transformations paths we can , for example, rearrange and visualise it like this:

![Fig2](/images/wiring-data-transformations-for-speed-002.png )

This is not necessarily a major departure from what we had before . There is still a subtle difference that will allow us to surface the underlying data transformation stages in a more meaningful way.

We want to reduce data transformations to this absolute simple world of data converted to other data. Data is *the* central point of focus in our approach as opposed to transformations. Transformations are also central to this landscape, however, since they are not particularly *indulged* with state they can be *simplified* or just *ignored*. Well ... At least for now !

We are always *forced* to deal with state , in one form or another, via a very explicit and error prone fashions. Not only handling state but always *knowing* about and dealing with what happens to that state in the life cycle of the application. Meaning, all parts of our business logic has to know about and care about this state. Part of our transformation strategy above is the fact that we refuse to deal with what happens to that state outside our transformation. Our transformations all deal with a particular data as input and only care about generating the next output from that data. But above and beyond that it should not be concerned with what happens to that data afterwards !

If the above is starting to sound like the same ideas championed by the functional programming world as [lambda calculus](https://www.youtube.com/watch?v=eis11j_iGMs) it is actually intentional. The idea here to strip all our data transformations from unnecessary concerns. We want to get each function to its bare minimum so that we can think about them as [pure functions](https://en.wikipedia.org/wiki/Pure_function). 

The payoff of this approach is that we can introduce three things in our transformations. Namely:
* data ownership and control 
* data storage
* data transports 

## Injecting data ownership !

We decided that our transformations are not preoccupied with what happens to the data once it leaves a particular transformation. As a direct consequence we can *infer* data ownership based solely on that invariant. The run time is then free to decide and assign ownership to that data or manage it as it so pleased. It can even assign ownership in a temporal and short lived fashion. The sky is the limit here in what kind of games we can play for the sake of performance !

Just imagine a world where you do not have to think twice not only about data ownership but also dealing with the constant concerns relating to concurrency and multiple read/write controls. You heard that right ! So far no one owns that data for a reason ! The run time basically hands this data to a transformation and says here do what want with it and please do not be worried about anyone else touching this data while you are mocking with it. I will not only handle the concurrency issues but i (as the run time and chief scheduling executive) will also make sure that this data is freed when it is no longer needed ! If you are thinking garbage collection then you are on the right track again. But we are actually talking about more than simple garbage collection. We are talking about a world where the run time is free to decide which garbage collection strategy it wants to use for a set of transformations based on the input/output or churn rate it can detect at run time ! Any sort of reclamation scheme like [RCU](http://www.rdrop.com/users/paulmck/RCU/hart_ipdps06.pdf), [EBR](http://concurrencykit.org/presentations/ebr.pdf) or [anything else](https://www.youtube.com/watch?v=aV-RyMXXuks) we dream to fancy !

Moreover, the run time can not only decide the reclamation strategy but it can also decide the allocation and scaling strategy as well. If a particular transformation can be distributed it can decide to distribute that and scale it up or down based on a predetermined control strategy. Speaking of control theory we could get to a world where allocation and reallocation can be [balanced](https://www.youtube.com/watch?v=B6vr1x6KDaY) entirely on churn rate !

## Injecting data storage !

Moreover, at any point in time, the data between our transformations can be *frozen* before it gets to our next transformations in the pipeline. Since we clearly know all the details we need to know about a particular data we can decide to *inject* stages in our pipeline where data is *frozen* before it gets to the next transformation in the pipeline. In our experience we found that all the issues you encounter in production are always due to these factors. Its either transport , storage or synchronizations issues. Sometimes you even get *lucky* and get all in a combo[1]. Rarely do you get issues because something was done wrongly in your *transformation/computation*. If something is wrong with your transformation it would be very easy to detect and catch during your unit testing. The corollary to that is "If something fails in your transformation it basically means that you have a bad unit testing in place"!

Coming back to our storage story, data storage can also be done in memory. It does not have to only be in slow magnetic disks. By virtue of us knowing everything about our data schema we can store the output of our transformations in memory in a sequential log oriented fashion that we can scream through to achieve maximum throughput. To quote *Martin Thompson* in [Designing for Performance ](https://youtu.be/fDGWWpHlzvw?t=1234): **Memory systems are about 3 bets**

>* The temporal bet
* The spatial bet
* The striding bet

By rearranging or scheduling our transformations appropriately on the right cores and memory we can actually play to these bets !

## Injecting data transports !

In the same way that our data can be stored in a multitude of fashions we can also decide to transport it in between our transformations with a transport that fits the bill for our particular application needs. If rest/http is a better fit for a transformation we can decide to adopt that. On the contrary if plain TCP is better we can decide to inject that in between our transformations. Any kind of transport that fits the bill can be *injected* here. It goes without saying that also mixing transports and picking and choosing the appropriate transports in an "a la carte" fashion is obviously possible.

The major thing we are doing here is delaying decisions about what kind of transport we will chose until we are ready to test everything at the system level. Obviously we still have to deal with platform specific transports as well as all the errors that come with a distributed system but all we are saying here is lets decide to delay *when* we need to deal with those issues. Or more specifically lets turn them into something that the platform or run time or even the person deploying the system can decide for us and free us from those concerns altogether !

## Recap please ?

Initially what we wanted was to focus on the actual data because at the end of the day data is our real problem !

>If you don’t understand the data you don’t understand the problem. *Mike Acton* ["Data-Oriented Design and C++"](https://www.youtube.com/watch?v=rX0ItVEVjHc)

Unfortunately, we found that every time we want to wire data transformations we have to also think upfront about what we consider as *avoidable concerns*. Everyone will agree that once you remove these concerns life becomes much easier. We can all agree that even if it is not going to be easy to implement our systems based solely on the idea of data transformations, at the very least, we can all agree that it is not something impossible to do. Who said this was not worth a shot ?

Or to put it differently: Let us all agree here on the benefits of `what` we need to accomplish in terms of simplifying our data transformations before talking about `how` we are going to accomplish that goal. It might take us few iterations to get it right but we think that this is more a `when` question than an `if` question .. so stay tuned for more !


