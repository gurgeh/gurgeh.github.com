---
layout: post
title: "A better algorithm for backups and rolling logs"
date: 2013-02-22 17:46
comments: true
categories: 
---

In your `/var/log/` you will most probably have logs that have grown too large and rolled over. Per default your system logger gzips and stores a few of the older ones and finally when they get too numerous, it just deletes them. Same thing for log handlers in most languages, for example Python's RotatingFileHandler. Backups are usually also handled the same way, when you don't want to store every backup.

That is almost never what I want. If I store only N backups, I don't want them to be only the very latest. If they are close in time they will be more similar and in some sense contain less information than backups that are spread out over time. Of course the newer ones are probably on average more interesting to me, so I don't want them evenly spread over time. The following simple algorithm is my suggestion for a better way to handle rotating logs and backups.

<!--more-->

## The algorithm

If we call the day the very first backup is made "day 0", the current state can be defined as a list of integers. If we currently have backups from day 0, 15, 20, 25 and 28, this is represented with the list *[0, 15, 20, 25, 28]*. Let us say that we want a maximum of 5 backups, then when we add the backup for day 29, we have to decide which backup to discard. We do this by rating each of the 6 possible (since we have 6 different numbers to potentially discard) configurations with a fitness function and chosing the best one. A lower score is better.

The optimal fitness function will be different from case to case. In essence it should be a balance, set by a parameter, between how valuable it is to have recent data and how interesting it is to have the data well spread out. As an example of a parameterless logarithmic fitness function, consider the following function, which rates one configuration by punishing each point by how far it is from it's ideal logarithmic position:

{% codeblock lang:python %}
import math

def expfit(backups, now):
    exponent = math.log(now) / math.log(len(backups))
    
    def exp_deviation(backup, backupnr):
        return abs(now - backup - backupnr ** exponent)
    return sum(exp_deviation(b, bnr)
               for bnr, b in enumerate(reversed(backups)))
{% endcodeblock %}

I have stored the code in an online interpreter called [repl.it](http://repl.it/ICA/3). You can try it out by clicking the Play button and then typing `expfit([0, 15, 20, 25, 28], 29)` and for example compare with replacing your most recent backup with todays, `expfit([0, 15, 20, 25, 29], 29)`.

To simulate how it would look after a certain time, I use the following code (also loaded in the session above):

{% codeblock lang:python %}
def tryall(backups):
    remove = min((expfit(backups[:i] + backups[i + 1:], backups[-1]), i)
                  for i in xrange(len(backups)))[1]
    return backups[:remove] + backups[remove + 1:]

def simulate(nrbackups, targetnow):
    backups = range(nrbackups)
    for now in xrange(nrbackups, targetnow + 1):
        backups = tryall(backups + [now])
    return backups
{% endcodeblock %}

Run it by `simulate(10, 100)` which should yield *[18, 30, 44, 59, 70, 84, 92, 96, 99, 100]*. That list looks like rather a nice set of backups after 100 days, to me.

## A functional reactive programming simulation in Elm

Visualising this seemed like a perfect excuse to try out [Elm](http://elm-lang.org/) which is a purely functional programming language which can handle input and output with the cool [Functional Reactive Programming](http://elm-lang.org/learn/What-is-FRP.elm) technology. I have put the resulting simple simulation [here](/assets/simulation.html).

## Possible improvements

+ The fitness function can potentially be made better, for example by incorporating a parameter as I described.
+ The algorithm is currently greedy, just using the best configuration for each step. It could make Monte Carlo simulations to see if a currently worse configuration will be better in the long run.
+ For some applications, early versions of a file are smaller than late versions. The fitness function could incorporate that you want to use a certain number of bytes for backups and be allowed to decide how many backups to keep as long as it is under this limit. This would skew it towards keeping more of the earlier but "cheaper" backups.
+ I could have checked if someone has already done something similar before I played with this, but that would not have been as much fun, now would it?

I have a Github project called *checkpoint* with source files for Python, Haskell and Elm [here](http://github.com/gurgeh/checkpoint).