---
## Common
title: Flow is non-blocking but the collector is not
tags: [kotlin, flow]
# description: 
published: true

## Github pages
layout: post
image: /assets/covers/flow1.jpg
sitemap: false
hide_last_modified: false
no_break_layout: false
categories: [Kotlin]

## Dev.to
cover_image: https://mahendranv.github.io/assets/covers/flow1.jpg
canonical_url: https://mahendranv.github.io/posts/kotlin-flow1/
# series:

---

[Flow](https://kotlinlang.org/docs/flow.html) is an idiomatic way in kotlin to publish sequence of values. While the flow itself suspendable, the collector will block the coroutine from proceeding further. Let's see with an example.

## ⛲ Flows
Let's create two flows — one finite and an infinite one. Both have a suspension point where they free up the thread 2s/1s. The `flow` builder creates a flow from a suspendable block and `emit`s values with given delay.

```kotlin
fun finiteEmissions(): Flow<Int> = flow {
    repeat(3) {
        delay(2000)
        emit(it + 1)
    }
}

val random = Random(7659)
fun infiniteEmissions() = flow {
    while (true) {
        delay(1000)
        emit(random.nextInt(10, 100))
    }
}
```

## 🚰 Collecting the flow

When collecting the above flows in the same coroutine, you'll notice the `infiniteEmissions` collect statement blocks the next set of statements from running. Whole point of flow is every single element emitted without blocking the current thread. But we're not proceeding further in the coroutine. Why?

```kotlin
fun main() {
    runBlocking {
        infiniteEmissions().collect { logger.debug(it) }
        logger.debug("won't complete")
        finiteEmissions().collect { logger.debug("Finite: $it") }
        logger.debug("completed")
    }
}
```

```
10:23:18.018[main] 80
10:23:19.019[main] 36
10:23:20.020[main] 30
10:23:21.021[main] 11
10:23:22.022[main] 10
10:23:23.023[main] 36
>>> goes on
```

Reason is, the collect statement itself a suspend function and it will naturally block the current coroutine from within it is called.

```kotlin
public suspend inline fun <T> Flow<T>.collect(crossinline action: suspend (value: T) -> Unit): Unit =
    collect(object : FlowCollector<T> {
        override suspend fun emit(value: T) = action(value)
    })
```

So, how do we unblock the rest of the statements or collect the flows in parallel? 

As usual — the `launch` coroutine builder. Launch starts a new coroutine without blocking the current one. Putting each collect statements in separate launch builder will unblock each other and collect them simultaneuosly.

```kotlin
runBlocking {
     launch {
         infiniteEmissions()
             .collect {
                 logger.debug(it)
             }
         logger.debug("won't complete")
     }   
     launch {
         finiteEmissions().collect { logger.debug("Finite: $it") }
         logger.debug("completed")
     }
}
```

```
10:53:3.003[main] 80
10:53:4.004[main] Finite: 1
10:53:4.004[main] 36
10:53:5.005[main] 30
10:53:6.006[main] Finite: 2
10:53:6.006[main] 11
10:53:7.007[main] 10
10:53:8.008[main] Finite: 3
10:53:8.008[main] completed
10:53:8.008[main] 36
10:53:9.009[main] 55
10:53:10.010[main] 96
>> goes on
```

Now both flows emits values in parallel and notice we're staying on same thread[main].

## 🙅 Cancelling the infinite flow

Cancelling flow means to cancel the coroutine which collects it. This is applicable for even for a coroutine which doesn't contain a flow-collect. `launch` builder returns a job which can be cancelled. Calling `job.cancel()` will terminate the underlying coroutine. For our example, to cancel the infinite flow once the second flow complete, do like this.

```kotlin
val job = launch {
    infiniteEmissions()
        .collect {
            logger.debug(it)
        }
    logger.debug("won't complete")
}
launch {
    finiteEmissions().collect { logger.debug("Finite: $it") }
    logger.debug("completed")
    job.cancel()
}
```

```
10:54:10.010[main] 80
10:54:11.011[main] Finite: 1
10:54:11.011[main] 36
10:54:12.012[main] 30
10:54:13.013[main] Finite: 2
10:54:13.013[main] 11
10:54:14.014[main] 10
10:54:15.015[main] Finite: 3
10:54:15.015[main] completed
```

## 🍬 Wrapup
This article covered collecting two flows in parallel. Like any other coroutine, `launch` will start them in parallel (not necessarily in new thread) and cancelled individually. To collect the flows in serial manner, no need for any special care — just collecting them from same coroutine would do it.

## Kotlin playground
Play around it [here](https://pl.kotl.in/W1pGJ6kAj)
<iframe width="100%" height="300pt" src="https://pl.kotl.in/GEmQrFk12?theme=darcula&from=6&to=19"></iframe>