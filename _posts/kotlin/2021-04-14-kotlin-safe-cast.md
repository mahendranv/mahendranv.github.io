---
layout: post
title: Kotlin safe casting
# image: /assets/covers/kotlin_value_cls.jpg
description: >
  How can we use kotlin safe casting operator to eliminate boilerplate code.
sitemap: false
hide_last_modified: false
tags: [android, kotlin]
no_break_layout: false
---

# Kotlin safe casting

![Kotlin safe cast operator](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fjg8xsdjcx33aj10rwh8.png)

In multiple cases you might've came across the situation where you have to cast a global variable and check the instance beforehand.

A most common case in Android environment is to check whether the (hosting) activity implements a callback or extended from Specific one.

## Before
```kotlin
if (activity is BaseActivity) {
   (activity as BaseActivity).hideKeyboard()
}
```

## After
Same can be written in sweet Kotlin syntax in a single line

```kotlin
(activity as? BaseActivity)?.hideKeyboard()
```

From the official doc
> Use the safe cast operator as? that returns null on failure.


## How this is better?
In surface it might not look like a better code overhaul. As calls to activity increase in numbers, it gets better.

```kotlin
// CreateTweetFragment.kt
fun submit() {
  
  // Fire and forget API call

  if (activity is BaseActivity) {
     (activity as BaseActivity).hideKeyboard()
  }
  
  if (activity is TweetSubmittedCallback) {
     (activity as TweetSubmittedCallback).onTweetSubmitted()
  }
}
```

We could easily cast the activity to the exact subclass and invoke both methods.
```kotlin
// CreateTweetFragment.kt
fun submit() {
  
  // Fire and forget API call

  if (activity is TwitterActivity) {
     (activity as TwitterActivity).hideKeyboard()
     (activity as TwitterActivity).onTweetSubmitted()
  }
}
```

But it enforces the *CreateTweetFragment* to be attached to the *TwitterActivity* to execute both statements. Anything that is just a BaseActivity or TweetSubmittedCallback won't execute the statement even though the implementation is in place.

> Prefer composition over inheritance

Now with the `as?` operator, both statements can be executed independent of each other.

```kotlin
fun submit() {
  
  // Fire and forget API call

  (activity as? BaseActivity)?.hideKeyboard()
  (activity as TweetSubmittedCallback)?.onTweetSubmitted()
}
```