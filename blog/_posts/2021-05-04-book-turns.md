---
layout: post
title: Drawing book problem - Hackerrank
# image: /assets/covers/kotlin_value_cls.jpg
description: >
  Drawing book problem - Hackerrank
sitemap: false
hide_last_modified: false
tags: [kotlin, coding]
no_break_layout: false
---

# Drawing book problem - Hackerrank

Problem statement is available [here](https://www.hackerrank.com/challenges/drawing-book/problem)


This problem needs minimum number of turns required to reach a particular page. 

**Input**
All the books start with page1 on the left and may or may not fill the left side of the page.

For a book contains **n** pages to reach page **p**, following is true.

* From front, turns (**t**) is always p/2. Below you can see few usecases

   ```kotlin
   page 1 --> 1/2 --> 0
   page 2 --> 2/2 --> 1
   page 3 --> 3/2 --> 1
   page 4 --> 4/2 --> 2
   
   i.e
   val turnsForPage = p/2
   ```

* When coming from back, there might be a fluctuation. Due to last page being an odd or even number. 

   For a book with **7** pages, reaching 2nd page from back takes 2 turns.

   **[-,1] [2,3] [4,5] [6,7]**

   

   If the book has **6** pages, the result is same

   **[-,1] [2,3] [4,5] [6,-]**



For turn count computation from back, the last pair should be treated equally. As we know, the turn count from front is consistent one. Lets try it for the above books.

```kotlin
val turnsForLastPage = n/2

6/2 = 3
7/3 = 3
```



Now we know, **the problem is never about the page number, but on which turn we land in it**. With this is mind, lets build a table to visualize turns vs pages mapping.

| Turns [Front] | 0    | 1    | 2    | 3    |
| ------------- | ---- | ---- | ---- | ---- |
| Turns[Back]   | 3    | 2    | 1    | 0    |
| Page - pair   | _, 1 | 2, 3 | 4, 5 | 6, 7 |



From the table we know, `Turn[Front] + Turn[Back] = turnsForLastPage`.



Let's apply this and get Turn[Back] from above computations.

```kotlin
val turnsFromLastPage = turnsForLastPage - turnsForPage
```



## Solution



So the complete solution is as below

```kotlin
fun pageCount(n: Int, p: Int): Int {
    val turnsForPage = p/2
    val turnsFromLastPage = n/2-turnsForPage
    return Math.min(turnsForPage, turnsFromLastPage)
}
```

