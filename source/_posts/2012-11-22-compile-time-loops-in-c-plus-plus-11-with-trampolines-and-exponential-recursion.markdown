---
layout: post
title: "Compile time loops in C++11 with trampolines and exponential recursion"
date: 2012-11-22 15:05
comments: true
categories: 
---

Nowadays with _constexpr_, we can make pure functions that are
calculated compile time. The restriction on these functions is that
they may only consist of one statement (though additional lines with
typedefs, static_assert, etc are OK), and this statement may only call
other constexpr functions.

If-statements are easy - just use the ternary ?: operator or, if you
want to show off, template specialization. But the only way to loop
with these restrictions is by using recursion. Furthermore, neither
clang 3.1 nor GCC 4.7 (nor the current build of GCC 4.8) support tail
call elimination in these constexprs, so normal linear loops will
still eat stack space if we loop for a while. Also, the standard
recommends that the default maximum recursion depth should be 512,
which means that if we want to do something silly/fun/interesting
compile time, we have to mess with proprietary compiler switches to
get the program to compile. No fun.

In this post, I show one way of working around those limitations.

<!--more-->

But first - why would one want to do constexpr meta programming, when
we already have template meta programming? Constexprs and templates
are almost exactly as powerful, except that template meta programming
is lazy, which some might think fits better into a functional style. A
more important difference is that template meta programming can work
with types, which means that we can use it for cool type-level
correctness stuff. So, once again, why would we use constexprs?
1) Constexprs are evaluated several times faster in the compilers I know
of, which can be important. 2) The syntax is much nicer.

## Simple recursion

Simple recursion is not a problem:

``` c++ Simple constexpr recursion - check primality
constexpr bool isprime(unsigned long n){
  return n == 2 or (n % 2 and rec_isprime(n, 3));
}

constexpr bool rec_isprime(unsigned long n, unsigned long div){
  return n % div == 0 ? 
           false
         : (div * div > n ? true : rec_isprime(n, div + 2));
}

#include<iostream>

int main(){
  //constexpr in declaration guarantees that it is evaluated compile time
  constexpr bool p1 = isprime(997);
  constexpr bool p2 = isprime((1 << 20) - 1);
  std::cout << p1 << " " << p2 << std::endl;
}
```

This program will reach a depth of at the most sqrt(2^20), which is
2^10 = 1024, and since we check at most slightly less than half of
those, we will be fine. Since 2^20 - 1 is not a prime number, we will
be even more fine. However if we try check if 2^31 - 1 is prime, the
compiler will refuse to do so. In this case we can fix it by
increasing the maximum constexpr evaluation depth for our compiler,
but in general that might take too much RAM, so lets do our own tail
call elimination with [continuation passing style](http://en.wikipedia.org/wiki/Continuation-passing_style) and a
<a href="http://en.wikipedia.org/wiki/Trampoline_(computing)">trampoline</a>
instead.

This way, we have a trampoline function that takes a CPS function as
an argument and loops, continously calling the continuation returned
from the previous call.

## Exponential recursion

In order to use a trampoline, we still need an infinite or near
infinite loop for the basic trampoline function. This is where
exponential recursion comes in. The recursion in `rec_isprime` behaves
sort of like a linked list - each invokation links to the next. If
each `rec_isprime` statement instead called two `rec_isprime`, which in
turn called two other, etc, we would get a call graph that was more
like a binary tree than a linked list. This way we would not reach our
maximum recursion depth until we visited 2^512 nodes and we would
never blow the stack!

Here is such a function:
``` c++ Exponential constexpr recursion - count invocations
const int MAXD = 24;

constexpr int count(int n, int depth=1){
  return depth == MAXD ? 
           n + 1
         : count(n, depth + 1) + count(n, depth + 1) + 1;
}

#include<iostream>

int main(){
  constexpr int i = count(0);
  std::cout << i << std::endl;
}
```

This program will visit 2^24 - 1 = 16777215 nodes. On my normal
desktop computer with clang 3.1 this program took 27 seconds to
compile and the process uses no almost memory at all. With GCC 4.7.2
it compiles in 0.2 seconds! This is because GCC uses memoization for
both templates and constexprs, so it will instantly see that the other
count call is unneccessary and transform our 16 million node tree to a
linked list of depth 24 again.

That is sort of cheating, so let's change the function slightly:
```c++ Exponential constexpr recursion - count without memoization
const int MAXD = 24;

constexpr int count(int n, int depth=1){
  return depth == MAXD ? n + 1: count(count(n, depth + 1), depth + 1) + 1;
}

#include<iostream>

int main(){
  constexpr int i = count(0);
  std::cout << i << std::endl;
}
```

Now memoization will not work. Clang takes 26 seconds and still uses
no memory. GCC surprises us again, now taking 44 seconds and using
over 3 gigs of RAM! If we increase MAXD to 25, GCC will again use
twice as much RAM. I don't know if this is because of memoization or
something else. I have reported it as a
[bug](http://gcc.gnu.org/bugzilla/show_bug.cgi?id=55442) to GCC.

Still though, it works!

## The trampoline

``` c++ Checking primality with a trampoline
const int MAXD = 100;

template<typename Fun>
constexpr Fun trampoline(const Fun f, int depth=0){
  return 
    f.done_ ? 
      f
    : (depth == MAXD ?
         f()
       // If we used mutual recursion here, Fun would not have to check for done_
       // unfortunately clang segfaults on mutual recursion in a constexpr..
       : trampoline(trampoline(f(), depth + 1)(), depth + 1));
}

struct Prime{
  explicit constexpr Prime(unsigned long n, unsigned long div)
  : n_(n)
  , div_(div)
  , done_(n % 2 == 0)
  , retval_(n == 2)
  {}

  explicit constexpr Prime(bool retval)
  : n_(0)
  , div_(0)
  , done_(true)
  , retval_(retval)
  {}

  //CPS! Prime does not recurse, but instead returns a continuation
  constexpr Prime operator()(){
    return (done_ ? *this 
            :(n_ % div_ == 0 ?
              Prime(false)
              : (div_ * div_ > n_ ? 
                 Prime(true)
                 : Prime(n_, div_ + 2))));
  }

  unsigned long n_;
  unsigned long div_;
  bool done_;
  bool retval_;
};

#include<iostream>

int main(){
  constexpr bool p1 = trampoline(Prime((1L << 31) - 1, 3)).retval_;
  constexpr bool p2 = trampoline(Prime((1L << 32) - 1, 3)).retval_;
  std::cout << p1 << " " << p2 << std::endl;
}
```

As you can see, we are allowed to create objects with constexpr
constructors in a constexpr, so we can create containers like linked
lists and trees. You can create maps, (simulated) vectors and sets on
top of trees, which means that anything is possible. In truth, prime
number calculation is not very exciting, but this post is running long
as it is, so a compile-time CPS implementation of Game of Life or
something will have to wait for next time.

Now, imagine the cheating possibilities in [The Computer Language Benchmarks Game](http://shootout.alioth.debian.org/) ;). Yes, yes, I
know it can be done in some other compiled languages as well. Template
Haskell and the Lisps come to mind.
