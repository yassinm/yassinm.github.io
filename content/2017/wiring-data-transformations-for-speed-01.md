---
date: 2017-09-09 11:00:00
title: Wiring data transformations for speed (part 1)
author: ["Yassin Mohamed"]

# featured: ["top01"]
tags : ["dataflow"]
series : ["Wiring data transformations for speed"]
---

If you think about our software industry today all we do , day in and day out, is "transform" data from one state to another. The systems we have in place today can be seen , at a very high level, as black boxes that take a variable number of arguments as inputs and produce a well defined output

>The purpose of all programs, and all parts of those programs, is to transform data from one form to another. *Mike Acton* ["Data-Oriented Design and C++"](https://youtu.be/rX0ItVEVjHc?t=757)

It is becoming insanely hard to keep focusing on that sentence when we are writing code these days. Instead, we fiddle and play with very complicated constructs that hide and obfuscate the true nature of what a program is supposed to do. It has become so revoltingly bad that not only can we not see the big picture from a high level view of the system we also cant even understand what the program is doing without a hefty amount of documentation and/or debugging !

In this article we will try to highlight some of the accidental complexities we introduce (knowingly or unknowingly) by writing code the way we do today.

## Going back to compilers

```java
public static void main(final String[] args) {
  long aXb = multiply(2, 3);
  long sum = add(1, aXb);
  System.out.println("sum:" +sum);
}
```

If you look at the above code you will see that all we are actually doing is moving data from one form to another. The `multiply` method takes a `{2,3}` tuple as input and outputs `{6}`. And the same is for the `add` Method. In fact if we feed this code to a compiler we will most likely end up with this data dependency graph

![Fig1](/images/wiring-data-transformations-for-speed-001.png )

Any compiler/jit worth its salt will actually collapse the above via DCE (dead code elimination) and turn it into a single literal inside the main function. But that is actually beside the point for now. All we want to show is the data dependency graph between the different code blocks.

The point we are trying to drive home is that the compiler is basically deconstructing what we gave to it into a data flow graph. Which begs the question: "Why did we push the compiler to make this translation?". Unfortunately the answer lies in years of history. Wish explains why we keep writing code that represents a data flow graph but we keep expressing that graph in a very imperative (not to say cumbersome) way and expect the compiler to go and figure what we actually mean. 

Before we jump to the conclusion that maybe there is a better way of coding the above let us first understand how many decisions we made by just writing the above code. Yes it is indeed very easy to write but once we understand the cost of the implicit decisions we made by just writing the above code we can step back and ask ourselves why did we force ourselves into this ?

## Mutability/concurrency !

The first implicit decision we made was a major one and it was about data ownership. Imagine if the data we were passing to the multiply method was not a simple literal but a large object or even to make things worse an array of objects ? Everything is passed by reference in java. As such you cannot attach arbitrary read/write capabilities to your objects at the time you are calling a method. If you want immutability you either have to use something like a guava library(with its Immutable.of saga) or make sure that the objects you pass is actually an interface. Or even worst convert your primitives into their java objects counterparts which are guaranteed to be immutable!That is a very hefty price to pay in terms of memory layout and even memory overhead if you ask me. Basically your objects have to be immutable at the time you are creating them ! But we want to stress that you have to know the trade offs of being immutable at a very early stage and stick to it. All developers are expected to know these common pitfalls and are supposed to deal with it ...  ¯\_(ツ)_/¯

As an alternative approach, actor oriented languages like pony's [Reference capabilities](https://www.ponylang.org/learn/#reference-capabilities) have a way of cleanly solving these hard concurrency issues by defining what it calls a *path* towards a reference. Having a reference to an object is not enough. You can also specify not only what you are capable of doing with that reference but what you are not. Which i found very refreshing to say the least ! By virtue of being newer, languages like [ponylang](https://www.ponylang.org) do not adhere to the idiosyncrasies we learned to live with after years of development history. It remains that, for the majority of popular languages out there, the run time nor the type system is going to help unless one is ready to use tricks we mentioned above.

## Application binary interface !

Its worth noting that the compiler took a look at the above code and decided (or was hard coded to decide ) what the ABI was between the function calls. The compiler just "assumed" that we actually wanted to pass a couple of variables to a local function. That is why the compiler was able to optimize it and reason about it. Now this makes a lot of sense in the majority of cases where you just want the compiler to make something that can be easily inlined.

Unfortunately, there is no way to tell a compiler please turn this function call into a distributed function call and please make sure you take care about the endian-ness and/or other ugly (but still important) issues associated with that. Another example would be trying to call the same function but written in a different language via a different FFI (foreign function interface). Java FFI is called JNI and is a nightmare to work with in practice. Plus the fact that it only supports "C" like any other languages out there. Cross language FFI is not something you can do easily across all languages for now without being forced to go through our dear mighty "C".

I know this is not necessarily what compilers are meant to support today. Most compiler writers will be happy to say "Thank you but no thank you" if you asked them to add features like the above. And they are probably right. A compiler is probably not the right place to do this. The point is that a compiler and the type system already know what we are passing as arguments between functions. Then why do we need a different way to again specify these same data structures so that they can be passed between languages and systems ? 

If a you as a programmer wants to see if you can call a rest API just to make the multiply method work in a distributed fashion you have to actually call a distributed system expert and make sure you deal with all the error handling as well as which transport is the most adequate one to use. Not to mention the threading model Just for that simple question ! It would be nice if in 2017 we had a way to tap into a different way of designing programming languages so that cross language boundaries or even  system boundaries was more seamless!

## Threading model !

Which brings us to our next issue. The dreaded threading model ! By definition the compiler assumes that if you call a method you are most certainly trying to call it within the current threading model .. obviously !!! If you want to work around that perception then you have to explicitly code around it !

Now, we know from experience that writing multi threaded code is not for the faint of heart. Unfortunately everyone and their uncle wants to write multi threaded/concurrent code because simply there is no way around it. Neither the run time nor the compiler make it easy to construct something that will say please run this particular code in a different thread and please make that as seamless as possible. I am not saying its impossible. You can still achieve something that works. But it still takes a serious head spin every time you want to embark in that .. or you need to call an expert again!  

Someone will mention the ideas of go subroutines and how easy it is to work with them. Which can be applauded since this is a step in the right direction. It does however attest to the simple fact that we need something better than what we currently have. Meaning, life could be much easier than this way of doing things. It is just error prone not to say completely brainless !

Somewhat related to the threading model is the notion 

of when a particular call to a function is going to be made. By definition a compiler will assume that what you want is to call that method immediately and you have to wait for the response. Meaning you have a blocking call automatically. It would be nice if we could execute that call asynchronously. Or to be even fancier switch to a blocking or non blocking call via a configuration flag and not decide that upfront imperatively.

## Conclusion

We could go on an on about what is wrong with the current tooling and programming paradigms we have today. Unfortunately we would only be stating the problems ! What we wanted to highlight here was the inherent problems associated with the way we do things today before moving forward. And to quote our favourite speaker again:

>Solving problems you probably don’t have creates more problems you definitely do. *Mike Acton* ["Data-Oriented Design and C++"](https://youtu.be/rX0ItVEVjHc?t=846)

To be clear the point of this article was to show that at the very beginning we all start with a simple flow graph. The fact that we are coding that flow graph in an imperative way is not the root cause of all our issues. However, as soon as we are done coding that basic flow graph we are immediately faced , more like *forced*, to deal with the issues we highlighted above. And since we are so used to and prefer to solve everything with code we decided , maybe wrongly, that we should deal with those issues by using the exact same means we used to express our data flow graph ! And therein maybe lies our mistake!

Now here is an idea: Would it be nice if we could code our basic flow graph in the way we are used to today but take out all the accidental/surrounding complexities and deal with them outside our simple flow graph ? Do you think that would simplify things and maybe move it a ton faster ? And when we say faster we do not only mean faster in terms of code speed but we actually mean coding and delivery time as well !

If we have your attention then stay tuned for another article this is going to be a (somewhat) long ride !


