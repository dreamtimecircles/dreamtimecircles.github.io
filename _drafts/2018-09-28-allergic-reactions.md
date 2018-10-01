---
layout: single
title:  "Allergic Reactions"
date:   2018-09-28 21:47:47 +0200
categories: reactor rant
---
It looks like folks like me that make a living with the JVM (among other things) will be still stuck for some more time with one form or another of [async](TODO), at least until [Kotlin coroutines](TODO) gain traction (for Kotlin users) or [Project Loom](TODO) sees the light.

Programming exclusively in the alien-minded, codebase-wrecking, stackless and pretty much undebuggable ghetto-DSL of your choice, rather than in the actual programming language, might be sad but that's all we've got for quite some time now and what we'll still need to cope with in the immediate future.

Admittedly there's more to [Reactive Programming](TODO) (or _Plumbing Programming_ as I like to call it) than suffering but still: if you can't stop at look
at your whole assembled plumbing while it does its work then you're in for a very bumpy, async-style time-travel back to barbaric "log-it-to-debug-it" witchery. Or, said another way, this whole embarassing async situation is a clear example of what happens when technology vendors don't really care about software development.

The basic unit of your plumb... Er, programming (a `Publisher`, that is a `Flux` or `Mono` in [Reactor](TODO) terminology) is a quite "behaviorally-rich" (i.e. complicated) sealed pipe whose operation you'll struggle to understand, let alone when combining many together via a hundreds-vast array of "operator" blackboxes (and exclusively those, because writing new ones is a nightmare) and more so because, as expected from the "market-it-fast" mindset of our times, these things came along stripped of (very much needed) introspection and debugging tools (which wouldn't have been too bad, had everybody wisely avoided to adopt them in such a shape).

Such publishers' features you should be aware of at all times are especially _subscription_, _advancement_, _completion_ and _failure_ events.

When you combine such complex objects together interesting questions arise, such as: what will the events' ordering and behavior be at various points of the plumbing? This can get so complicated so fast that the main, de-facto standard documentation for such an operator includes a plumbing drawing such as this one:

(TODO)

What's even more interesting is that some behaviors can be undefined, which means that, unless you're so very careful as to take that into account from the beginning, you'll get used to your implementation's until something will change and break everything in a racy way that you might or might not be able to spot soon enough. Anyway just stare at your big log pile's interleaving for some time and good luck with that.

