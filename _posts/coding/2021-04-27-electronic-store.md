---
layout: post
title: Electronic store problem -  Hackerrank - Kotlin solution - elimination approach
description: >
  Electronic store problem -  Hackerrank - Kotlin solution - elimination approach
sitemap: false
hide_last_modified: false
tags: [kotlin, coding]
categories: [Coding, Hackerrank]
no_break_layout: false
---

## Introduction

Problem definition is available [here](https://www.hackerrank.com/challenges/electronics-shop/problem)

Our goal is to compute the costly pair of keyboard & drive we can get from the the store, given the budget (***b***).

The input is **keyboards** and **drives** cost arrays and **budget**.

We'll start with brute force approach and optimize the solution around it.


* toc
{:toc}

## Brute force

Given two arrays of size m and n, the brute force will check each pair and compare against the current max value and the budget. So, `m*n` pair checks will be made to find the result.



```kotlin
fun getMoneySpent(keyboards: Array<Int>, drives: Array<Int>, b: Int): Int {
    var pairChecks = 0
    var max = -1
    for ( k in keyboards) {
        for (d in drives) {
            val total = k + d
            if (total <= b && total > max)  {
                max = total
            }
        }    
    }
    println("$pairChecks pairs checked")
    return max
}
```



```
// Input 1
Budget : 5
Keyboards = [6, 5, 3, 7, 1]
Drives = [2, 3, 7]
```



For the *input1*, we get 5x3 = 15 pairs checked. Going through the input we can see the pair would be [3, 2] which meets the budget of 5. Checking anything further is wasting CPU resource. Let's add some checks and see further.



```kotlin
outer@for ( k in keyboards) {
        for (d in drives) {
            val total = k + d
            if (total == b) {
                // budget met here, exit the loop
                max = total
                break@outer
            }
            if (total <= b && total > max)  {
                max = total
            }
            pairChecks++
        }    
}
```

> Output:
>
> 7 pairs checked

With this best-case budget check, we get only 7 pairs checked which nice, but it's not every day you meet the magic number. Going through the input again, it's evident some keyboards are costlier than our budget. For each costly keyboard, we can save 3 iterations.

```kotlin
outer@for ( k in keyboards) {
        if (k < b) {
            for (d in drives) {
                val total = k + d
                if (total == b) {
                    max = total
                    break@outer
                }
                if (total <= b && total > max)  {
                    max = total
                }
                pairChecks++
            }  
        } 
}
```

>Output:
>
>1 pairs checked



That's not bad. But we know the output is specific to this input set. Adjusting one parameter in the input set will affect the pair checks. 


```
// Input 2
Budget : 4
Keyboards = [6, 5, 3, 7, 1]
Drives = [2, 3, 7]
```

Take **input2**, where we are short of a dollar in the budget, and the output is,

>5 pairs checked



```
// Input 3
Budget : 60
Keyboards = [40, 50, 59]
Drives = [5, 8, 12]
```



Let's take another input :: **input3** and check the result.

> 9 pairs checked

So, whatever the optimization that we did, the **input3** just didn't fit in. We're relying on best case to break the loop, as the input is not ordered is some manner. If the input is ordered beforehand, we'll get best results.



## Sorted input

A sorted input has impact on how we compute the output.

```
// Input 4
Budget : 60
Keyboards = [59, 50, 50, 40] // descending order
Drives = [5, 8, 12, 14]      // ascending order
```



With current implementation, we get all the pairs checked in input 4. 

> 16 pairs checked

Let's take advantage of sorted arrays. Below we can visualize the sum for each pair. And form a hypothesis on how to eliminate extra iterations.


|      | 5    | 8    | 12   | 14   |
| ---- | ---- | ---- | ---- | ---- |
| 59   | 64   | 67   | 71   | 73   |
| 50   | 55   | 58   | 62   | 64   |
| 50   | 55   | 58   | 62   | 64   |
| 40   | 45   | 48   | 52   | 54   |



At [0, 0] the pair is already past the budget, which means [0,1] and [0,2] will exceed the budget (since drives will get costlier going right). So, we can break the inner loop and read from next row.

```kotlin
outer@for ( k in keyboards) {
        if (k < b) {
            inner@for (d in drives) {
                pairChecks++
                ...
                if (total > b) {
                    // For sorted array, if the total is already past the budget, it's not going to come down. 
                    break@inner
                }
								...
            }  
        } 
    }
```



Now the output is 

> 11 pairs checked



Now we have avoided few elements towards the right. 

|      | 5    | 8      | 12     | 14     |
| ---- | ---- | ------ | ------ | ------ |
| 59   | 64   | ~~67~~ | ~~71~~ | ~~73~~ |
| 50   | 55   | **58** | 62     | ~~64~~ |
| 50   | 55   | **58** | 62     | ~~64~~ |
| 40   | 45   | 48     | 52     | 54     |



Notice... for any column, the numbers getting smaller as we move downward. This is due to the descending order sorted *keyboards* array. So, we don't have to check this column (call cMax) anymore.

|      | 5    | 8      | 12     | 14     |
| ---- | ---- | ------ | ------ | ------ |
| 59   | 64   | ~~67~~ | ~~71~~ | ~~73~~ |
| 50   | 55   | **58** | 62     | ~~64~~ |
| 50   | 55   | ~~58~~ | 62     | ~~64~~ |
| 40   | 45   | ~~48~~ | 52     | 54     |



For a given column, elements on the left will be smaller (since the *drives* array is sorted in ascending order). So, we can strike down any element to the left of *cMax*.

|      | 5      | 8      | 12     | 14     |
| ---- | ------ | ------ | ------ | ------ |
| 59   | 64     | ~~67~~ | ~~71~~ | ~~73~~ |
| 50   | 55     | **58** | 62     | ~~64~~ |
| 50   | ~~55~~ | ~~58~~ | 62     | ~~64~~ |
| 40   | ~~45~~ | ~~48~~ | 52     | 54     |



We are going to use index for looping through the columns. So, few modifications needed for the inner loop.



```kotlin
    var cMax = -1
    outer@for ( k in keyboards) {
        if (k < b) {
						inner@for (dIndex in (cMax + 1) until drives.size) {
                val d = drives[dIndex]
								...
            }  
        } 
    }
```



We haven't changed the iteration logic yet, instead of forEach, we're using the index now. `cMax` will be our shifting pointer to start loop in each row. This value must be assigned each time we find the max value.



```kotlin
if (total <= b && total > max)  {
	max = total
  cMax = dIndex
}
```



With index shifting, we have the expected pair checks as in the table.

> 7 pairs checked



Remember, the solution expects the arrays to be sorted in above manner. So, we must sort the inputs beforehand. Since we already have better sorting algorithms, we can use inbuilt ones to do it. So, the complete solution looks like this.

## Solution

```kotlin
// Solution - for loop

fun getMoneySpent(keyboards: Array<Int>, drives: Array<Int>, b: Int): Int {
   // Data preparation
   keyboards.sortDescending()
   drives.sort()
  
    var max = -1
    var cMax = -1 // Which column holds the current max

    outer@for ( k in keyboards) {
        if (k < b) {
            inner@for (dIndex in (cMax + 1) until drives.size) {
                val d = drives[dIndex]
                
                val total = k + d
                if (total == b) {
                    max = total
                    break@outer
                }
                if (total > b) {
                    break@inner
                }
                if (total <= b && total > max)  {
                    max = total
                    cMax = dIndex
                }
            }  
        } 
    }
    return max
}

```



### Edit

This solution is bit clumsy as we started with for loop and refined over time. 



I removed the check where keyboard cost being greater than budget. Now it will cost at most one iteration compared to the brute force method.

```kotlin
fun getMoneySpent(keyboards: Array<Int>, drives: Array<Int>, b: Int): Int {
    var max = -1
    var cMax = -1 // Which column holds the current max

    outer@for ( k in keyboards) {
            inner@for (dIndex in (cMax + 1) until drives.size) {
                val d = drives[dIndex]
                val total = k + d
                if (total == b) {
                    max = total
                    break@outer
                }
                if (total > b) {
                    break@inner
                }
                if (total <= b && total > max)  {
                    max = total
                    cMax = dIndex
                }
        } 
    }
    return max
}
```



> Throw away the prototype

With lessons learned from above implementation, it is clear that index plays a key role here. So, implemented the same using while loop.



```kotlin
// While loop solution

fun getMoneySpent2(keyboards: Array<Int>, drives: Array<Int>, b: Int): Int {
    var max = 0
   keyboards.sortDescending()
   drives.sort()
    var i = 0
    var j = 0
    while(i < keyboards.size) {
        while (j < drives.size) {
            val total = keyboards[i] + drives[j]
            if ( total > b) break // Here we eliminate moving towards right
            if ( total > max) {
                max = total
            }
            j++  // This is equivalent to cMax+1
        }
        i++
    }
    return max
}
```

## Conclusion

It might look like overhead to sort the arrays before computation. This will work out well on a long run in the following cases

1. Eliminating the elements to the right, left and bottom with current max will drastically bring down comparisons in large data set. 

2. This solution can be scaled to provide weightage to the products. 
> eg. For a costlier keyboard, find a cheap mouse
>     Keep the keyboard cost below 70% budget

Sorting can give a clarity while applying any of the above constraints. 