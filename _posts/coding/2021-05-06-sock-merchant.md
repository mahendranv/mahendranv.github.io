---
layout: post
title: Sock merchant problem - Hackerrank
# image: /assets/covers/kotlin_value_cls.jpg
description: >
  Sock merchant problem - Hackerrank
sitemap: false
hide_last_modified: false
tags: [coding]
categories: [Coding, Hackerrank]
no_break_layout: false
---

Problem definition is available [here](https://www.hackerrank.com/challenges/sock-merchant/problem)

This is a classification problem where the element can repeat multiple times in the given array. 


Like any other problem, this can be solved in multiple ways in kotlin.


### 1. Elegant - yet non performant solution

```kotlin
fun sockMerchant(n: Int, ar: Array<Int>): Int {
  return ar.groupBy { it }
      .map { it.value.size/2 } // it is Map<Int, List<Int>>
      .sum()
}
```

This is a straightforward solution

1. Read each value and create a map `Map<Int, List<Int>>`
2. Convert each key-value-entry into integer by computing the size of value array
3. Sum all elements in the integer list obtained from previous step


Though this looks elegant and **kotlin**ish, we shouldn't ignore the fact that it creates a Map with value list of identical elements. While what needed is just a counter mapped against each element.


### 2. Old school - manual computation

For this classification problem, `Map` is the ideal datastructure for grouping.

```kotlin
fun sockMerchant(n: Int, ar: Array<Int>): Int {
    val map = mutableMapOf<Int, Int>()
  	for (s in ar) {
        map[s] = (map[s] ?: 0) + 1
    }
  	return map.values.fold(0) { 
        acc, v ->
        acc + v/2
        }
}
```



Here, we create a map `Map<Int, Int>` & increase the counter for each bucket. Then fold it. So, there is no value `List<Int>` created. Moreover, based on the unique values, the Map size constrained. At the end, the map values iterated through and folded into count.



**Wait.. There is still room for improvement** 



We need a consolidated count, **not** the per element pair count. That means, the fold function can be avoided by adding another line to the first loop.

```kotlin
val map = mutableMapOf<Int, Int>()
var pairCount = 0
for (s in ar) {
    var count = (map[s] ?: 0) + 1
    map[s] = count
    if (count % 2 == 0) { 
      // For every second encounter, increase pairCount by one
        pairCount++
    }
}
return pairCount
```



> Be kind to GC, few more lines of code won't hurt