---
layout: post
title: Kotlin value class
image: /assets/covers/kotlin_value_cls.jpg
description: >
  Value classes introduced in Kotlin 1.5. What's so special about it and where do I use it?
  How the byte code looks for a value class?
sitemap: false
hide_last_modified: false
tags: [android, kotlin]
no_break_layout: false
---

# Kotlin value classes

Kotlin's `data class` is a fan favorite when it comes to store any model. Bundled with bunch of necessary methods, devs get a lot while writing way less. From ***Kotlin 1.5*** — we have  `value class`-s. Let's see what is a value class and where to use it.



While data class is used for holding model , `value class` adds attribute to a value and constraint it's usage. This class is nothing but a wrapper around a value, but the Kotlin compiler makes sure there is no overhead due to wrapping.


* toc
{:toc}

---



## The problem

Duration is a classical problem where anyone can make mistake, if they didn't read between the lines. So, I wrote a function to show a tooltip message for given duration in millis. The signature looks like this.



```kotlin
fun	showTooltip(message: String, duration: Long) { ... }
```



To make sure the caller pass in correct duration, I can do this.

```kotlin
fun	showTooltip(message: String, durationInMillis: Long) { ... }
```



even this,

```kotlin
/**
* Shows tooltip of message for given duration
*  @param message - message to display
*  @param durationInMillis - duration in milliseconds
**/
fun	showTooltip(message: String, durationInMillis: Long) { ... }
```



Despite all the naming and documentation, someone can invoke the tool tip with seconds. After all [NASA lost a spacecraft](https://www.simscale.com/blog/2017/12/nasa-mars-climate-orbiter-metric/) due to a unit mistake.

```kotlin
showTooltip("I'm going to pass duration in seconds", 2L)
```


---



## A no-brainer solution

To make sure, the user pass in millseconds, the duration can be wrapped in a class, and retrict the object creation to few meaningful helper methods, this wrapper class can make sure proper value passed in to the method.



```kotlin
class Duration private constructor (
    val millis: Long
) {
    companion object {

      fun millis(millis: Long) = Duration(millis)

      fun seconds(seconds: Long) = Duration(seconds * 1000)
    }
}

fun showTooltip(message: String, duration: Duration) {
  	println("Will show $message for ${duration.millis} milliseconds")
}

...
showTooltip("Hello - Seconds", Duration.seconds(2L))
showTooltip("Hello - Millis", Duration.millis(1200))
...
```



Now the `showTooltip` function takes in `Duration` and process it in `millis`. This ensures the caller send proper duration to the function and `showTooltip` can rely on the `Duration#millis` field. 



![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/o2hcaay1cp2s95wq0s1g.png)



However, for each duration usage, we box it inside a object, just to avoid communication gap. To pass in a parameter, a new object is created and allocated in memory.

---



## Enter kotlin - value class



The Kotlin value class **wraps a single value** and thus impose any restriction/conversion to it. Kotlin compiler takes away the boxing in possible cases to deliver performance.

```kotlin
@JvmInline
value class Duration private constructor (
    val millis: Long
) {
    companion object {

      fun millis(millis: Long) = Duration(millis)

      fun seconds(seconds: Long) = Duration(seconds * 1000)
    }
}
```



We don't see much difference but just two new keywords. But under the hood, a normal `Duration` class and value class is different altogether. I'm leaving the value class byte code here. In case you wonder how the normal class looks in bytecode, read through this and then reach to ByteCode of normal wrapper section.



```java
// Duration -- value class bytecode

public final class Duration {
   private final long millis;

   private Duration(long millis) {
      this.millis = millis;
   }

   public static final class Companion {

       // Comapanion that outputs Duration is mangled to return the wrapped value
      public final long millis_PZfE49U/* $FF was: millis-PZfE49U*/(long millis) {
         return Duration.constructor-impl(millis);
      }

       // Comapanion that outputs Duration is mangled to return the wrapped value
      public final long seconds_PZfE49U/* $FF was: seconds-PZfE49U*/(long seconds) {
         return Duration.constructor-impl(seconds * (long)1000);
      }
   }
}

// Caller function name mangled
public static final void showTooltip_fxiZ0zM/* $FF was: showTooltip-fxiZ0zM*/(@NotNull String message, long duration) {
    ...
}

...

// Invoking the function that consumes value class param.
showTooltip_fxiZ0zM("",
  Duration.Companion.millis_PZfE49U(2000L));      
showTooltip_fxiZ0zM("",                     
  Duration.Companion.seonds_PZfE49U(2L));

```



As you see, where ever the `Duration` object used, the function names are mangled and the param // output type changed to `Long`.



In case you wonder why the mangling is needed here, this avoids any conflict resolution when the function signature matches the wrapped type.



```kotlin
// Using a wrapped value arg
fun showTooltip(message: String, duration: Duration) {...}

// Using a primitive value as arg
fun showTooltip(message: String, duration: Long) {...}
```



If the compiler just replace the param type with wrapped primitive here, the code won't compile. So it is necessary to mangle up the name. 

---



##  ByteCode of normal wrapper 

```java
public final class Duration {
   
   private final long millis;

   private Duration(long millis) {
      this.millis = millis;
   }

   public static final class Companion {

      // Companion function return the wrapped object
      public final Duration millis(long millis) {
         return new Duration(millis, (DefaultConstructorMarker)null);
      }
      
      // Companions function return the wrapped object
      public final Duration seconds(long seconds) {
         return new Duration(seconds * (long)1000, (DefaultConstructorMarker)null);
      }
   }
}

// Called function signature looks same
public static final void showTooltip(String message, Duration duration) 
{ ...}

// Caller is 
showTooltip("", Duration.Companion.seconds(2L));
showTooltip("", Duration.Companion.millis(1200L));
```



That's what a normal wrapper class look like. In case you wonder why did I use a normal class instead of a `data class`. See below.



```kotlin
data class Duration private constructor(
    val millis: Long
) {
		// Same companion as normal class
}
```



```java
public final class Duration {

   private Duration(long millis) {
      this.millis = millis;
   }

   public final long component1() {
      return this.millis;
   }

   @NotNull
   public final Duration copy(long millis) {
      return new Duration(millis);
   }
   
  // Same companion byte code from normal class
  ...
}
```



`data class` creates a copy method, which allows you to create `Duration` given a long  param. Our goal is to restrict the object creation to the companion methods. Also data class leaves a componentN function, which we don't need in this case. We're just trying to wrap the value here.



## Endnote: 

Since Kotlin 1.2.xx, we have [inline classes](https://github.com/Kotlin/KEEP/blob/master/proposals/inline-classes.md), the *old name* for the value class. Since the class is not actually inlined in comparison to the inline function, it has been renamed to value class and the inline keyword is deprecated now.