---
layout: post
title: "C++11 and Boost - succinct like Python"
date: 2012-10-31 13:05
comments: true
categories: 
---
C++11 is the new standard of C++ that was released last year. Yes, I know that is now 2012, but compilers are just now starting to catch up and implement everything, though AFAIK there is not yet a fully compliant compiler. 

With a combination of C++11 and the [Boost](http://www.boost.org/) library, I think that it is possible to write code in a style that is almost as painless as in a modern dynamic language like Python. I also think that is not so well known how much C++ has changed for the better, outside the C++-community. Hence this post.

As an example, I have taken the first interesting excercise from [Dive into Python](http://www.diveintopython.net/toc/index.html), [fileinfo.py](http://www.diveintopython.net/object_oriented_framework/index.html), and converted it to C++, trying to remain as faithful as possible to the original code.

<!--more-->
`fileinfo.py` implements a simple MP3-metadata printer, which is built so that it will be easy to add other metadata printers in the future. For the Python code, see the previous link and for my commented C++ code see here:

{% gist 3871749 %}

If you remove my gratuitous commenting and the includes, you will notice that it is more or less as succinct as the Python code.

Perhaps more importantly, it is also as safe. No pointers to point at bad places and leak memory and no buffers to overflow. There is much more to why modern C++ is more safe and less memory leaky than it once was, besides the added emphasis on a more functional style, for example raw pointers are actively discouraged (use smart pointers instead) and arrays (not vectors!) are now finally in the standard library, so we use array<int, 10> instead of int array[10].

This program is actually the basis of a presentation I gave at work, which had more explanations on the features used. It only touches the surface on what you can do with C++11 You can read more about the new features in C++11 in [the biggest changes in C++11 and why you should care](http://blog.smartbear.com/software-quality/bid/167271/The-Biggest-Changes-in-C-11-and-Why-You-Should-Care) and of course on [Wikipedia](http://en.wikipedia.org/wiki/C%2B%2B11).

Herb Sutter, who is chair of the ISO C++ standards committee and thus perhaps not completely unbiased, had this to say on the relase of C++11:
> Now with C++11's improvements that incorporate many of the best features of managed languages, modern C++ code is as clean and safe as code written other modern languages, as well as fast, with performance by default and full access to the underlying system whenever you need it.

I think it is almost there. For example, the language is still too large, so in a new project you have to have a good code standard, that decides well on which parts should be used and how. The standard library, even including Boost, is still a bit weak in some areas, like downloading an URL. Also the lack of a proper package manager, like pip/PyPi, hurts.