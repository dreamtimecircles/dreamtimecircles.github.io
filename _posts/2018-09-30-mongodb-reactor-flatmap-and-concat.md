---
layout: single
title:  "MongoDB, Reactor, `flatMap` and `concat`"
date:   2018-09-30 12:54:47 +0200
# categories: mongodb reactive reactor
---
This one took some time to understand and figure out, possibly due to the fact that [Project Reactor](https://projectreactor.io/) is new playground for me, so I thought I'd share it.

I'm sure the following scenario is fairly common: replacing an in-memory, Reactor-based [DAO](https://www.oracle.com/technetwork/java/dataaccessobject-138824.html) stub with a [MongoDB](https://www.mongodb.com/) one while still retaining end-to-end reactivity.

Consider the following [Kotlin](http://kotlinlang.org/) (admittedly silly) snippet that builds a reactive master-detail retrieval computation:

```kotlin
    dao.findIds()
        .doOnNext { id: String ->
            LOGGER.info { "[findIds] $id" }
        }
        .flatMap { id: String ->
            dao.getDocById(id)
                .doOnSubscribe {
                    LOGGER.info { "before getDocById: $id" }
                }.doOnSuccess {
                    LOGGER.info { "after getDocById: $it" }
                }
        }
        .doOnNext { entity: Model ->
            LOGGER.info { "[getDocById] $entity" }
        }.then(Mono.fromCallable {
            LOGGER.info { "ALL DONE" }
        }).block() // Only for demo
```

In simplified terms we could say that, when we subscribe (in this case with `block()`), this code will first fetch the document IDs we're interested in (through `findIds()`) and "then" for each ID obtained it will also fetch document with that ID (through `getDocById()`).

The problem is that this simplified view can confuse us and cause us to think that [`flatMap`](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html#flatMap-java.util.function.Function-) acts as a sequential loop, which in general isn't true.

Here `findIds` is really builds a multi-element pipeline (or [`Flux`](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html)) that, once subscribed, will be pulling IDs from our DB. `flatMap` on the other hand is hooking in as many single-element pipelines (or [`Mono`](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Mono.html)s) as IDs we'll be receiving from `findIds`, each of them pulling the document with that specific ID and potentially operating concurrently with others.

The concurrency behavior might depend on the actual [`Publisher`](https://www.reactive-streams.org/reactive-streams-1.0.0-javadoc/org/reactivestreams/Publisher.html) implementation (as well as other factors such as how it is configured and the concurrency allowed on the specific platform/system) so it might well change when switching implementations: a typical example is when replacing a strictly single-threaded, in-memory stub where every shared-state access is [mutex](https://en.wikipedia.org/wiki/Mutual_exclusion)ed (e.g. via [JVM's `synchronized`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.11.10) or [j.u.c. locks](https://docs.oracle.com/javase/8/docs/api/?java/util/concurrent/package-summary.html)) with an implementation accessing a real database through a concurrency-capable driver like the [Reactive Streams MongoDB driver](https://mongodb.github.io/mongo-java-driver-reactivestreams/).

This means that the above code could legally produce such an execution trace:

```
before getDocById: 1
before getDocById: 2
...
after getDocById: 2
after getDocById: 1
```

Of course, if we don't depend on `getDocById()` to be called sequentially everything will be fine but if, on the other hand, we use reactive event handlers like [`doOnSubscribe`](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Mono.html#doOnSubscribe-java.util.function.Consumer-), [`doOnNext`](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Mono.html#doOnNext-java.util.function.Consumer-), [`doOnSuccess`](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Mono.html#doOnSuccess-java.util.function.Consumer-) etc. to hook in logic that is supposed to run sequentially then we're in for trouble because, as we've seen, `flatMap` will let `getDocById()` pipelines fire aggressively (or equivalently, in Reactor documentation's words, it "subscribes eagerly").

We should of course try the best we can to avoid depending on sequential behavior because this might limit scalability at some point but if we really need that we can use the [`Flux#concat`](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html#concat-java.lang.Iterable-) operator instead:

```kotlin
    Flux.concat(dao.findIds() // ADDED: Flux.concat
        .doOnNext { id: String ->
            LOGGER.info { "[findIds] $id" }
        }
        .map { id: String -> // WAS: flatMap
            dao.getDocById(id)
                .doOnSubscribe {
                    LOGGER.info { "before getDocById: $id" }
                }.doOnSuccess {
                    LOGGER.info { "after getDocById: $it" }
                }
        })
        .doOnNext { entity: Model ->
            LOGGER.info { "[getDocById] $entity" }
        }.then(Mono.fromCallable {
            LOGGER.info { "ALL DONE" }
        }).block() // Only for demo
```

Note that we don't use anymore `flatMap` to subscribe, and so "hook in", document-retrieval `Mono`s as soon as IDs arrive but rather we just ask to simply _transform_ these IDs into sleeping/unsubscribed `Mono`s with [`map`](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html#map-java.util.function.Function-) and finally we use `concat` on this resulting "publisher of publishers" to process each one sequentially.

This change will restrict the document retrieval to be sequential no matter how the publishers are implemented, thus it will make it impossible to produce traces such as:

```
before getDocById: 1
before getDocById: 2
...
after getDocById: 2
after getDocById: 1
```

I also recommend having a look at [Combining Publishers in Project Reactor](https://www.baeldung.com/reactor-combine-streams) for information on additional combinators.