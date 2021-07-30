---
## Common
title: Designing a bottom navigation bar with Jetpack compose
tags: [android, compose]
# description: 
published: true

## Github pages
layout: post
# image: 
sitemap: false
hide_last_modified: false
no_break_layout: false
categories: [Android, Compose]

## Dev.to
cover_image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ty8isvlu9e1d7v3ptjm9.png
canonical_url: https://mahendranv.github.io/posts/compose-bottom-bar/
# series:

---

## Intro

This article covers the design aspect of the bottom navigation bar using Jetpack compose. We're looking at a simple composable with few customizable params for the icons. Take this as an approach doc for designing UI rather than an API guide. The result will look like this.

![Final](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/2dku2am921cqdw5nmj0s.gif)

---

## 📐 Analysis
Let's dissect this view and see how to build this. 

1. This is a row with equal space around each icon
2. At a time only one icon can be selected
3. All the icons are of the same size except the selected one, which is at 1.5x
4. Row accounts for the larger icon as well and all the icons are vertically center aligned

### Interactions
When an icon is selected, it scales up to 1.5x and changes its color change to blue. And previously selected icon resets to 1x and goes back to grey color. This transition is not sudden and happens over a particular time.

---

## 🏗️ Building the UI

UI is straightforward here. Icons and a Row that holds them. So, the first step is to prepare a few ingredients to build the UI. Add material extended icons in Gradle dependency and pick some for the navbar.

```kotlin
private val navBarItems = arrayOf(
    Icons.Outlined.Home,
    Icons.Outlined.Send,
    Icons.Outlined.FavoriteBorder,
    Icons.Outlined.PersonOutline,
)
```

Choose two colors for selected and unselected state:

```kotlin
private val COLOR_NORMAL = Color(0xffEDEFF4)
private val COLOR_SELECTED = Color(0xff496DE2)
```

Define a base icon size. This will be scaled up to 1.5x when selected. Also, it's a good practice to make most of the properties configurable with default values. After all, we're dealing with functions here.

```kotlin
private val ICON_SIZE = 24.dp
```

...

Let's start with icons and build a row of them.

### Animated icons:

![Icon animation](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/9unw3efj3p7wj3tyqkcz.gif)

From the icons aspect, it hoists its click to the parent and expects its parent to tell which state it should move to. The icon has one responsibility - to animate its color and scale when the properties are set. It doesn't know about its state. All it knows is to smoothly change the values.

> Quick note about compose animation - it always tries to transition from the current value to the target value. Asking for the start value might lead to a jump in animations, which is avoided here. IMO compose team did it well here.

```kotlin
@Composable
fun AnimatableIcon(
    imageVector: ImageVector,
    modifier: Modifier = Modifier,
    iconSize: Dp = ICON_SIZE,
    scale: Float = 1f,
    color: Color = COLOR_NORMAL,
    onClick: () -> Unit
) {
    // Animation params
    val animatedScale: Float by animateFloatAsState(
        targetValue = scale,
        // Here the animation spec serves no purpose but to demonstrate in slow speed.
        animationSpec = TweenSpec(
            durationMillis = 2000,
            easing = FastOutSlowInEasing
        )
    )
    val animatedColor by animateColorAsState(
        targetValue = color,
        animationSpec = TweenSpec(
            durationMillis = 2000,
            easing = FastOutSlowInEasing
        )
    )

    IconButton(
        onClick = onClick,
        modifier = modifier.size(iconSize)
    ) {
        Icon(
            imageVector = imageVector,
            contentDescription = "dummy",
            tint = animatedColor,
            modifier = modifier.scale(animatedScale)
        )
    }
}
```

Animated icon composable holds two animatable values and is set to corresponding attributes. To interact with its parent, it exposes an onClick lambda. And animation target values are passed from the parent.

...

Before plug this into Row, try it out with a preview. As our icon is a dumb view, the preview will have unusual responsibilities. It toggles selection and updates the correct scale/color to the icon. On the other hand, a wrapper Composable could be built with a preset of selected/normal attributes and take a boolean to update the same. Again, it's a function.

```kotlin
@Preview(group = "Icon")
@Composable
fun previewIcon() {
    Surface {

        var selected by remember {
            mutableStateOf(false)
        }

        AnimatableIcon(
            imageVector = Icons.Outlined.FavoriteBorder,
            scale = if (selected) 1.5f else 1f,
            color = if (selected) COLOR_SELECTED else COLOR_NORMAL,
        ) {
            selected = !selected
        }
    }
}
```

---

### Row design

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/wyhly65vqrc83h16cxo4.gif)

From the requirements, the row is responsible for horizontally placing all the icons and update their scale/color based on selected index. It doesn't know how the icons react to it. In order to observe whether an icon clicked, it listens to the click and updates the selected index.

Apart from the above responsibilities, the row aligns icons as described in the analysis part. To remember which position is selected, it uses a mutable state. This will be updated on icon click and read before setting scale/color for the icons.


```kotlin
@Composable
fun BottomNavBar2(
    modifier: Modifier = Modifier,
    iconSize: Dp = ICON_SIZE,
    selectedIconScale: Float = 1.5f
) {
    var selectedIndex by remember {
        mutableStateOf(0)
    }

    Row(
        verticalAlignment = Alignment.CenterVertically,
        horizontalArrangement = Arrangement.SpaceEvenly,
        modifier = modifier
            .fillMaxWidth()
            .padding(vertical = 8.dp)
            .height(iconSize.times(selectedIconScale))
    ) {
        for ((index, icon) in navBarItems.withIndex()) {
            AnimatableIcon(
                imageVector = icon,
                scale = if (selectedIndex == index) 1.5f else 1.0f,
                color = if (selectedIndex == index) COLOR_SELECTED else COLOR_NORMAL,
                iconSize = ICON_SIZE,
            ) {
                selectedIndex = index
            }
        }
    }
}

```
Finally, put the bottom nav bar inside a preview and interact with it.

```kotlin
@Preview(group = "Main", name = "Bottom bar - animated")
@Composable
fun previewBottomNavBar2() {
    Surface {
        BottomNavBar2()
    }
}
```

---

## 📖 Endnote:

This navigation bar is a very basic implementation. It doesn't even have a callback setup for selection change. And icons are hardcoded. It is a good idea to use the material `BottomNavigation` as it is well tested and can interact with Snackbar and also has a FAB cradle. Use this as a quickguide on how to keep a view dumb and do state hoisting.
