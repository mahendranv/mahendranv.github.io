---
## Common
title: Coroutines & threadlocal
tags: [kotlin, coroutine, thread]
description: Coroutine ThreadLocal behavior
published: false

## Github pages
layout: post
# image: 
sitemap: false
hide_last_modified: false
no_break_layout: false


## Dev.to
# cover_image: 
# canonical_url: 
# series:

---

## Background
On rare occassions we might want some values to be known inside the thread and not affected by other workers. This is where a [ThreadLocal](https://docs.oracle.com/javase/7/docs/api/java/lang/ThreadLocal.html) comes in. Threadlocal creates instances of a variable per thread.

### Quick go through on ThreadLocal

The below example illustrates API calls to the server. t1 & t2 are the computation threads spawned per API call. Each computation thread collects session info and uses it later to serve response. 



```kotlin


val session = ThreadLocal<String>()

fun main() {
    session.set("rick")
    logger.debug("Main thread:: initial name is ${session.get()}")

    val t1 = thread {
        buildSession("morty")
        returnResponse()
    }

    val t2 = thread {
        buildSession("jerry")
        returnResponse()
    }

    t1.join()
    t2.join()
    logger.debug("Main thread:: final name is ${session.get()}")
}

private fun buildSession(newName: String) {
    logger.debug("Start: ${session.get()}")
    session.set(newName)
    sleep(1000)
}

private fun returnResponse() {
    logger.debug("Changed name is ${session.get()}")
}

```