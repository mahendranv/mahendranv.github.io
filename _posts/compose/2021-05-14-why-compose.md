---
layout: post
title: Why should I go for compose?
image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/llkri0z57qisq0vpsb0i.jpg
description: >
  Why should I go for compose. Tools & what Google got this right this time.
sitemap: false
hide_last_modified: false
tags: [android, compose, kotlin]
categories: [Android, Compose]
no_break_layout: false
---


Way less boilerplate and better tooling. That's why.

---

## What's wrong with the old widgets?

The old Android widget is painstakingly (smart!?) enough to has its own state and it is not necessarily in alignment with the viewmodel that represents it. Having references to the view widely encourages devs to **update a view from another view**. This often doesn't end well. 

**Few rare issues that you might've faced** 

1. Save button disabled even though the correct data present in the input box. To fix it user must delete a character and retype it.

2. Incorrect tab header selected while the page is different.

3. Switch/checkbox show in selected state, but it is in off state as per shared preferences. To make it align with the correct state, user must exit from the screen and come back.

4. User see $0 total for a moment once landed on checkout screen.

The list goes on..



With the presence of `ViewModel` and `LiveData`, when effectively used, solves most of the above issues. Yet, there are two problems there to the core.

1. View references that are available to other views
2. Mutable views

...


## What is Compose?

Jetpack Compose is a modern UI toolkit for developing android apps. Compared to the native UI where we design in xml and bind using Kotlin/Java, `@Composable` functions are used in the toolkit to build UI around given data. Composable functions are dynamic, can bind data naturally and avoids boilerplate from old system.


...


## Why compose?

In compose UI kit, there are no referenceable views, a view either present or totally gone (I mean gone from the memory). A view cannot update other view directly — Everything must go through a state variable. **Views are not there for their own sake but to represent data or collect it. ** Putting the data as a main player makes it easy to do validations independent of view and even [share business logic](https://kotlinlang.org/lp/mobile/) between different UI clients.



### Write less, do more

A single Composable function can replace list item xml, view holder, recyclerview adapter and any callback set to propagate item clicks. Same goes for building compound components.


This is the composable of below counter.

```kotlin
@Composable
fun Counter() {

    var count by remember { mutableStateOf(1) }

    Row {
        Icon(Icons.Sharp.RemoveCircleOutline,
            contentDescription = "minus",
            modifier = Modifier
                .alpha(if (count > 1) 1f else .4f)
                .clickable {
                    if (count > 1) {
                        count--
                    }
                })

        Text(
            count.toString(),
            textAlign = TextAlign.Center,
            modifier = Modifier
                .width(40.dp)
        )

        Icon(Icons.Sharp.AddCircleOutline,
            contentDescription = "plus",
            modifier = Modifier
                .alpha(if (count < 10) 1f else .4f)
                .clickable {
                    if (count < 10) {
                        count++
                    }
                })
    }
}
```

...


### 🧐 Preview - interact - move on

Once you complete a `@Composable`, create a mock object (if needed) - feed it to composable through a `@Preview` function and interact with it — right inside AndroidStudio!!. Even deploy it to your device.

![counter](https://i.imgur.com/FufUeOU.gif)



Imagine building a custom/compound component and testing it on device in old UI kit. The whole setup will require a dummy activity, run config changes, no need to say the cleanup afterwards.


...


### 🎼 You're really composing views 

- To add an attribute to a composable: try a modifier or wrap it inside a container and apply modifier to it.
- Don't set visibility at all, just don't attach the view to tree
- A button with Text and Icon? Composable inside another composable is how you play it here.



```kotlin
@Composable
fun ShareButton() {

    var showIcon by remember {
        mutableStateOf(false)
    }

    Button(onClick = { showIcon = !showIcon },
        shape = RoundedCornerShape(8.dp)) {
        if (showIcon) {
            Icon(Icons.Default.Share, contentDescription = "share" )
            Spacer(Modifier.width(16.dp))
        }
        Text(text = "Share")
    }
}
```





![share_btn](https://i.imgur.com/ABHWVzu.gif)


...


### ✨ Less error prone

Views are inflated and then manipulated. A mandatory xml attribute might be missing for a custom view resulting in UI error. Composable are functions that reacts to the argument. Like any Kotlin function, nullable, non-nullable, default arguments can be defined here.


...



### 🧱 Brick by brick

To keep the codebase clean, you're forced to split a screen into multiple chunks of composables. So, you design brick by brick.

Also, it means a fat object from API can be mapped into domain model composed of small chunks to align with independent UI development. So, we define finite set of inputs and UI states per chunk.


This opens possibilities to reuse the composable in ways that would have been a nightmare before. Let's say I have a composable that represents, user in a list view — I can plug it inside my user details screen without any hassle.



```kotlin
@Composable
fun ProfileScreen(profile: Profile) {
  	Column {
      
      ProfileListItem(profile.general)

      if (profile.flairs != null) {
          ProfileFlairs(profile.flairs)
      }

      ProfileBioContent(profile.bio)

      if (profile.achievements != null) {
          AchievmentListView(profile.achievements)
      }

      ProfileWorkExperience(profile.workExperience)
    }
}
```



To achieve the above, we'll have to create multiple custom views and explicit binding. Now the complete set of inflater, xml & view references and visibility changes are not needed anymore.


...



### 🇦 Typography 

Compose supports fontweights, this is something that designers and android devs don't align with in widget ages. Finally, we're one step closer to building what is in our mocks — it sounds cool when we speak fontWeight to the designer. We have fontweight from Android 9, *released in 2018*. But we're fragmented anyway.

```kotlin
Text(
    text = "Weight 700",
    fontWeight = FontWeight(700)
)
```

Above and this — [annotatedString](https://developer.android.com/jetpack/compose/text#multiple-styles)


...



### 🖥️ Compose Desktop

Compose desktop is a thing now. UI that is built for android should work for desktop as well. I have created separate project for both to play out. But haven't shared the UI code yet. Even the package name in desktop carries `androidx`, let's see how it goes. Furthermore read [here](https://github.com/JetBrains/compose-jb/tree/master/tutorials/Getting_Started).



![dt_todo2](https://i.imgur.com/LTxpGiF.gif)






### But compose destroys views on each event resulting in overhead 🤫

No... composable functions are smart enough to patch update only the dirty portions. This is called recomposition. Recomposition is set to skip as many composables possible on each pass. Read more about this [here](https://developer.android.com/jetpack/compose/mental-model#recomposition).

---

## Endnote:

Above is purely my opinion about composable. Just sharing it here hoping fellow devs would add pro-cons. Developing native Android apps over years and felt painful some aspects of Native UI, trying Compose is very refreshing. For an analogy, this is like dealing with java and NPE for eternity and starting with Kotlin. (Also, I forgot, java mandates try-catch blocks and the semicolon).
