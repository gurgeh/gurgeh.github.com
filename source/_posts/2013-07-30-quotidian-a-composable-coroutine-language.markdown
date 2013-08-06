---
layout: post
title: "Quotidian - a composable coroutine language"
date: 2013-07-30 19:29
comments: true
categories: 
---
I admit it. I have a language addiction. I crave new programming languages. But while I savour the flavour of any freshly brewed manual, there is an even greater flaw in my character - I like to design new lanuages myself.

Quotidian is a language based on the idea that imperative algorithms often have a purpose to fill locally, but globally, effects have to be contained, beaten into submission and eliminated.

<!--more-->

Imagine a basic language that looks like ML, with functions, strict call-by-value semantics, algebraic data types, pattern matching, type inference, garbage collection and so on. Now we add the defining feature of Quotidian - routines.

A routine is something like pipes/conduits in Haskell, goroutines + channels in Go, yielding generators in Python or coroutines in general. It has parameters but no return value. Instead it communicates through sending and receiving values through named channels. These named channels are part of the routine's type signature. Values are sent with "send <outchannel> <value>". Values are received with "let x = recv <inchannel>" to wait for one value, or "for x in recv <inchannel>" to loop over all values until the channel is closed.

A program is constructed by connecting the channels of routines. Routines can be "curried", meaning that you can connect one channel to another channel in another routine and now get a new routine with the unconnected channels as type.

You can connect several outchannels to one inchannel. Doing a recv on such an inchannel will wait for any of the outchannels to have data.

[Parallel connection]

[Two-way communication]

[Mutable values and linear typing]

[Program example]

[Visual programming]

