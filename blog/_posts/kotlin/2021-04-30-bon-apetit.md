---
layout: post
title: Hackerrank - bon-appetit problem solution in Kotlin
# image: /assets/covers/kotlin_value_cls.jpg
description: >
  Hackerrank - bon-appetit problem solution in Kotlin
sitemap: false
hide_last_modified: false
tags: [coding, kotlin]
no_break_layout: false
---
# Hackerrank - bon-appetit problem solution in Kotlin

* toc
{:toc}

For detailed problem statement, head over to the bill division problem [here](https://www.hackerrank.com/challenges/bon-appetit/problem)

From the problem definition, we get two things

1. Element at index `k` should not be considered for bill computation
2. Rest of the elements should be added and then share computed by dividing total by two

Bear in mind that division should be done *after* computing the total. So, below I listed few aggregators to compute the sum from array (excluding an index). There are more ways to achieve this in kotlin. Each has slight difference and some of them are improved version of other.


### 1. sumBy 


```kotlin
array.withIndex().sumBy { // IndexedValue<Int>
    if (it.index != k) {
        it.value
    } else {
        0
    }
}
```



Above creates an `IndexingIterable`, emits `IndexedValue<Int>` for each item and consumed by the `sumBy` function



### 2. sumOf

```kotlin
array.withIndex().sumOf { // IndexedValue<Int>
    if (it.index != k) {
        it.value
    } else {
        0
    }
}
```



`sumOf` follows the same footprint of `sumBy`. Only difference is `sumOf` can return different result type (ex. Double or Long instead of Int). Below snippet explains the usage of each sum function.

```kotlin
val totalInt = array.sumBy { it }
val totalDouble1 = array.sumBy { it.toDouble() } //Error: Type mismatch
val totalDouble2 = array.sumByDouble { it.toDouble() }
val totalDouble3 = array.sumOf { it.toDouble() }
```



### 3. foldIndexed

```kotlin
array.foldIndexed(0) {
    index, acc, value ->
    acc + if (index == k) {
        0
    } else {
        value
    }
}
```

**foldIndexed** function takes in a initial value and an operation lambda to operate on the element. Notice the lambda returns the accumulated value on each iteration. Unlike sum functions which needs element to operate on, here the operation owned by the us.



### 4. reduceIndexed

```kotlin
array.reduceIndexed {
    index, acc, value ->
    acc + if (index == k) {
        0
    } else {
        value
    }
}
```



**reduceIndexed** is similar to foldIndexed function, and it lacks the initial value. 



There are few downsides using reduce instead of fold:

1. If the `array` is empty, reduce function will throw **UnsupportedOperationException** 

   *java.lang.UnsupportedOperationException: Empty array can't be reduced*

2. Accumulated value result type cannot be different from the array element type



> Functions listed above operates on original array and computes the result. Below ones creates a filtered list out of original array, it can be used to perform operations on it.



### 5. filterIndexed + sum()

```kotlin
array
	.filterIndexed { index, _ -> index != k }
	.sum()
```



### 6. fold

```kotlin
array
    .filterIndexed { index, _ -> index != k }
    .fold(0) { acc, value -> acc + value }
```



### 7. reduce

```kotlin
array
    .filterIndexed { index, _ -> index != k }
    .reduce{ acc, value -> acc + value }
```

---

## Solution:

Here is the complete solution to the problem.

```kotlin
fun bonAppetit(bill: Array<Int>, k: Int, b: Int): Unit {
    val diff = b - bill.foldIndexed(0) { 
        i, total, v ->
        if (i != k) {
            total + v
        } else {
            total
        }
     }/2
    
    println(if (diff == 0) {
        "Bon Appetit"
    } else {
        diff
    })
}

```

**Edit:** I left out the loop statements here to focus on aggregators. Feel free to comment if anything else turn up, I'll add it to the list.
