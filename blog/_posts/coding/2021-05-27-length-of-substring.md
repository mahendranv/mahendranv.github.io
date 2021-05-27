---
## Common
title: Kotlin solution for the longest Substring Without Repeating Characters
tags: [coding]
description: Solution for leetcode /#3 — the Longest Substring Without Repeating Characters
published: true

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
# Kotlin solution for the longest Substring Without Repeating Characters

For problem definition head over to [leetcode problem#3](https://leetcode.com/problems/longest-substring-without-repeating-characters/).

For a given string, we need to find the **length of the longest substring**. Listed few sample input and outputs below for understanding.

```

Max() = 0
Max(a) = 1
Max(aa) = 1
Max(ab) = 2
Max(abca) = 3
Max(abcdefghijk) = 11
Max(abccabcde) = 5
Max(abccabdefghabcababcab) = 8

```

Emphasized on length as we don't need the actual substring itself but the length. So, let's jump into implementation. 

## Brute force
The brute force approach would be to pick a start and end index within the string and compare each character against themselves and have a global variable to track the max length. This solution involves three loops for picking start—end indexes and comparing characters. Let's not consider this solution and move on to the algorithimic approach.

## Solution — Sliding window
For a subset problem, one of the approaches to take in is a *Sliding window*. Sliding window restricts the looping to subset of elements. By maintaining certain global pointers, it avoids revisiting the same node multiple times. We can restate our problem statment in different way.

> We need the length of the substring, in other words, difference between two indices
{:.lead}

With above in mind, we setup our window in following manner. Our end index (i) will increase in linear manner in the range of (0, n-1) both inclusive. With end fixed, we update the start index based on last occurance. Since we need only the length, difference between the indices will do the trick. 


**Instruction set:**
- If an element is already present, move the start pointer if needed
- Compute the length between start and end indices and max length
- For each element, update a lookup table with the latest encounter index


I've dropped the solution below and walkthrough follows. 

```kotlin

    fun lengthOfLongestSubstring(s: String): Int {
        // Sliding window start pointer
        var start = 0
        // Result
        var max = 0
        // Occurance map
        var occ = mutableMapOf<Char, Int>()
        
        for ((i, c) in s.withIndex()) {

            if (occ.containsKey(c)) {
                // Found a clash, move the start pointer 
                // next to the previous occurance if applicable
                val seekPoint = occ[c]!! + 1
                start = Math.max(seekPoint, start)
            }

            val length = i - start + 1
            max = Math.max(length, max)

            // Update the map with latest occurance
            // when clash happen next time, use this index
            occ[c] = i
        }
        return max
    }

```

## Walkthrough

Let's have a walkthorugh on iteration. 
```
Given abcamnijabxyzzz, max-substring is mnijabxyz
```

| #   | start | end | char | subsequence | length |
| --- | ----- | --- | ---- | ----------- | ------ |
| 0   | 0     | 0   | a    | a           | 1      |
| 1   | 0     | 1   | b    | ab          | 2      |
| 2   | 0     | 2   | c    | abc         | 3      |
| 3   | 1     | 3   | a*   | bca         | 3      |
| 4   | 1     | 4   | m    | bcam        | 4      |
| 5   | 1     | 5   | n    | bcamn       | 5      |
| 6   | 1     | 6   | i    | bcamni      | 6      |
| 7   | 1     | 7   | j    | bcamnij     | 7      |
| 8   | 4     | 8   | a*   | mnija       | 5      |
| 9   | 4     | 9   | b*   | mnijab      | 6      |
| 10  | 4     | 10  | x    | mnijabx     | 7      |
| 11  | 4     | 11  | y    | mnijabxy    | 8      |
| 12  | 4     | 12  | z    | mnijabxyz   | 9      |
| 13  | 13    | 13  | z*   | z           | 1      |
| 14  | 14    | 14  | z*   | z           | 1      |

**0-2** — Each element is scanned, added to lookup map. Since there is no clash, start index is unchanged.

**3** — From lookup we come to know `a` was encountered in 0th index. So, start is adjusted to (0+1)

**4-7** — There is no clash, start is still at 1. At 7th iteration, you can see `bcamnij` is the current substring. i.e first `a` is excluded, new `a` is present in the substring.

**8** — From lookup, we found `a` was last encountered in 3rd index. Notice [a, 0] updated to  [a, 3] during last encounter at **3**. This is crucial because, if we keep the first encountered index in lookup, it will include duplicates. Now we update start to (3+1).

**9** — At 9, we found `b` is present in `1`. But we won't update the start with (1 + 1), since we're way past 1st index. So we retain the start at `4`. To verify the uniqueness, `mnijab` is the current substring. If at all, we've updated our start to `2` we'll have `camnijab` — here two `a`s present.

**10-12** — Through these iterations, we add character by character to the substring. Keeping the start index at 4. 

**13** — `z` is identified as duplicate. So index adjusted to previous encounter + 1 ⇨ 12 + 1. The substring contain single character. The second `z`.

**14** — Same as iteration 13.

**Note:** apart from the window movement, the lookup and max length is updated in each step. Since it is an obvious step to the algorithm, I skipped in walkthrough. Though I printed the substring in the table, it is not necessary for our final result.

...

With a lookup map and sliding window, we reduced the time complexity to O(n). Space complexity would be O(m). Where m is unique charsets.

...

👨‍💻 happy coding 👩‍💻