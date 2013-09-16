---
layout: post
title: "Big programming, small programming"
date: 2013-09-03 17:39
comments: true
categories: 
---
Programming is complicated. Different programs have different abstraction levels, domains, platforms, longevity, team sizes, etc ad infinitum. There is something fundamentally different between the detailed instructions that goes into, say, computing a checksum and the abstractions when defining the flow of data in any medium-sized system.

I think that the divide between *coding the details* and *describing the flow* of a program is so large that a programming language could benefit immensely from keeping them conceptually separate. This belief has led me to design a new programming language - *Glow* - that has this separation at its core.

<!-- more -->

## Contrasts

_In the big picture_, we want good documentation.

_In the details_, documentation is often outdated and redundant.


In the big picture, uncontrolled side effects and mutable global variables lead to an unmaintainable mess where hidden assumptions are everywhere.

In the details, say within one or a handful of functions, mutation can be efficient, idiomatic and easy to understand.


In the big picture, we often want to annotate functions with types, even if the language can infer them automatically or does not need them.

In the details, we don't want to litter the code with explicit types, making it harder to read and write and harder to change implementation details.


In the big picture exceptions are dangerous beasts. They are essentially gotos. Non-local behaviour that can make a system hard to reason about and that can give suprising failure modes.

In the details, exceptions are a convenient way of signalling that the input to a non-total function did not live up to its contract or that the surrounding system (hardware, DB, etc) are not in the assumed state. If we can signal these breaches of contract compile-time, using the type system, this is great. Unfortunately not everything can be statically proven.


In the big picture, visual programming is very natural. It is, in fact, so natural that we often make flowcharts of how systems work. It would be better if these flowcharts would actually be the system, rather than an incomplete, possibly outdated description.

In the details, visual programming is just an inconvenient gimmick. IMHO. It may change one day.

## The feature checklist

- Pythonesque syntax
- Statically typed
- Automatically type inferred
- Strict evaluation
- Garbage collected
- Reference implementation compiled through LLVM
- Separate functions (small) and components (big)
- Components are similar to coroutines and have named input and output pipes (like channels in Go, pipes in Haskell, more general than generators in e.g Python), which are part of their type. Pipes are synchronous/blocking.
- Even though pipes are synchronous, the standard libray has queue components that can be mounted on a pipe to make it asynchronous/non-blocking with an explicit queue size.
- An input pipe can be looped over or read "manually" one value at a time. You may not peek.
- If a component loops over an input pipe, it may be inlined into the outputting component.
- Pipes can be attached to components with different connections, for example making the components run in parallel, in different processes, with target component on GPU, etc.
- A kind of linear typing for mutable data
- Mutable data cannot be shared between components. If sent from one component to another, it becomes unreadable in the first.
- Constant data can be shared between components.
- Algebraic datatypes, structs and vectors
- Pattern matching
- Components and functions are first class and can be curried.
- A function cannot have side effects. Effects are triggered through pipes to components.
- The C FFI has to be wrapped as components and is the only way to get effects. None are builtin.
- Contracts for assumptions which do not fit the type system.
- Preprocessing/macro system with the full power of the language, like Lisp or Template Haskell.
- Simple textual metadata can be attached to components, pipes and connections, which help when laying out the connections visually in a flowchart-type interface.

### Components
A component is a function that may have input arguments, but not return arguments. It also has zero or more input pipes, output pipes or bidirectional pipes. A component can send data to an output pipe and it can receive data on an input pipe, either by "forall" to loop until the pipe closes or "get" to block until it gets the next value. Pipes can also be bidirectional, which means that you can both send and receive. This is convenient, for example when you want to make database queries to a component.

A pipe can be closed both upstream (sender has run out of data) and downstream (receiver has found what it is looking for). Since a pipe can be closed downstream, you can compose strict pipelines that behave lazily by terminating early when they are done, yet are nicely decoupled.

If a component has only one input pipe, it is deterministic in the sense that the behaviour is identical to a pure function with a lazy list or generator as input argument. This means that it is easier to test. If we had a "peek" function that could check if an input pipe had anything new yet, the receiving component could loop and do different stuff depending on when a new value is ready, begging for race conditions. If a component has more than one input pipe it is not deterministic in general, since "forall" can take several pipes of the same type and emit the values in the order they are produced.

Components may clean up after a pipe closes, which I will describe more in a later post in conjunction with exceptions.

A component with one input pipe and one output sounds very similar to a function. You are supposed to use a function if there is a one-to-one mapping between input and output and you do not keep state. isPrime (:: Int -> Bool) would most naturally be a function. Run length encoding, for example, would more naturally be a component.

    #no arguments, one input pipe named values and one output pipe named encoding.
    runLengthEncode :: Eq a => | values a || encoding (a, Int)
    runLengthEncode():
      first = True
      count = 0
      oldval...
      foreach value in values:
        if first:
          oldval = value
          count = 1
        elif value == oldval:
          count += 1
        else:
          put encoding (oldval, count)
          oldval = value
          count = 1
      if count:
          put encoding (oldval, count)

Ignore syntax, the use of Haskell-style type classes and the detail of what default value oldval will get at the start, when "values" just have to be a type that is comparable. When we have just one input and one output, this is equivalent to something like a generator using "yield" in Python.

I don't think pipes need to come with any performance overhead compared to a normal function call, so they are supposed to be used a lot.

### Concurrency

Components can be run concurrently and it should "just work", since components cannot share mutable data. If you need shared mutable data, a component that keeps state and has more than one input pipe will no longer have any guarantees of determinism and could be used as (or wrap), for example, a database.

The same component could also be run concurrently in parallel against the same input pipe, as long as the input and output order does not matter. This problem is inherent in parallelism. In MapReduce, the problem of the unordered output pipe is solved by attaching a key to each indata value, which can also be attached to the output.

When connecting two components, the type of connection indicates how the output component is run. As mentioned in the checklist, above, you can also imagine GPU connections. Since all effects except local mutation are handled as pipe communication, a Glow component should not need many constraints to target a GPU.

### Effects and FFI

No side-effects are built in to the language. Even stuff like current time and print are handled by a C FFI. A FFI must be wrapped as a component, so all effects are clearly visible as pipe communication. Since you might want the same output component to handle, for example, logging or file handling in many components, unattached output pipes with the same name and type are merged to one output pipe when two components with dangling outputs are connected.

### Testing and dependency injection

Since side effects are often slightly troublesome when testing, it is my intention that a component connection can later be overriden (or monkey-patched, if you will) by a mock-component with the correct type. This means that the file system, database or clock for an entire application can be overriden for testing purposes or when you want to simply switch out a component across an application. I have not yet fully worked out how namespaces, etc, should work in this context.


## Visual programming
We use two types of files to program Glow. The detail files, which can contain anything, and the overview files, which may only contain constants and how components are connected to each other. An overview file may not use macros.

Both are editable with any text editor, but the overview file can also be edited visually as a vectorized flowchart (vectorized to zoom in on details). This should have a number of benefits.
- The overview chart is good documentation that is never out of date. Not for a library API, but for a system or application.
- You can design programs on your tablet or phone, by touch. Fill in the details when you get to a keyboard.
- You get a visual REPL for debugging and testing. Change constants (read more below) and watch changes, live.
- The flowchart may also serve as a simple control panel for the application in production.

It is important that you should never need a special application to view the overview file. It must be perfectly legible as text, even when generated by a flowchart-editor.

Note that what you would work with in this interface is not types and class hierarchies as you might in a modelling tool, but the actual component instances. The static data. Component instances and other data that is generated dynamically, either compile-time or runtime, would not show up. Only the concrete, static data, components and connections can be in the overview file. This is also why no macros are allowed in the overview file - how would an interface edit component connections that are generated compile time and not written explicitly in the original source?

During runtime, the same flowchart-like interface could be used to add visualisers of data flowing through different pipes. Graphs, counters, latest value, etc, could be monitored. Perhaps color to indicate relative traffic.Just as objects in Python have a __str__ or __repl__ method, when they need to be printed, standard Glow data types could have html-representations rendered in the interface.

I believe that the component+pipes concept lends itself very well to Functional Reactive Programming. A side-effect of this is that statically defined data could have controllers that dynamically changes the program behaviour during runtime. I'll try to flesh out my ideas for that later.

## In closing
This was just a first overview of Glow. I have not yet written about the type system, the module system, exceptions and contracts (yes, they are tightly coupled), metaprogramming and "sublanguages", how functional reactive programming fits with components or, you know, the actual syntax. All those things, except syntax (meh) and module system (probably something close to MixML) are already designed.

Also, if it was not obvious, there is no compiler or anything else yet. Not a shred of code. I know that a compiler (and visual editor!) is a serious undertaking. I think that this particular programming language does not rely on any magic that makes it harder to implement than, say, Rust. I also think it would be different enough to stand out. Perhaps others are intrigued and willing to help out?

This was inspired by Bret Victor, the Haskell pipes library, Go and many other things.


