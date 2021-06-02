---
layout: post
title: Jetpack compose — DropdownMenu — Weather UI
image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/llkri0z57qisq0vpsb0i.jpg
sitemap: false
hide_last_modified: false
tags: [android, compose, dribbble]
categories: [Android, Compose]
no_break_layout: false
---
## Introduction

Sharing my experience with jetpack while practicing with the weather - forecast screen(from uptechteam). While surfing dribble this particular shot caught my eye. Minimal - subtle colors also simple layout to implement. Best fit if you want to start with any UI framework.



![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/duvfi6mom2mk7u5aqne9.gif)


## Anatomy

This page comprise of multiple sections, each will cover different ui element from compose. To make use of stacked layouts (box, row, column), I didn't touch ConstraintLayout in this project.

1. Top app bar → Popup window
2. Carousel of weather cards with indicator
3. List of forecast for the week



## Appbar - Spinner

App bar holds a back arrow and a `Spinner?!!` that helps user to switch between cities. Let's focus on the popup window part.



In the old android system, this is a `Spinner` component with custom adapter (since we do color updates). This requires layouts defined for both anchor and list item, inflated at some point. Not to say about binding the content and updating the selected element. *Enogh with old Views.* Let's see how  to implement this in Jetpack.



### Implementation


### Anchor view

Create a `Row` container that holds the *Text* & *Icon*. This will act as anchor for dropdown and ensure clicking both items will show the menu.

```kotlin
Box {
  // Back arrow here
  Row { // Anchor view
    Text(text = "",) // City name label
    Icon(imageVector = Icons.Filled.ArrowDropDown,)
  }
}
```

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/aay0t6pzkwjatm5yudx9.png)


Now we don't see any city name displayed in it. We can take a param for the `@Compose` function and update it. But we expect the name to change while picking a city from menu. Thankfully, we have a nice way of doing this in **Compose**.



```kotlin
// State variables
var cityName : String by remember {mutableStateOf(cityList[0])}

...

Text(text = cityName,) // City name label
```



Here, **cityName** is a local state that update the UI once the value change. We set the initial value to first item in the list.


![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/odfa8m7gz7jvmdvv4hsd.png)


...

#### Dropdown menu design

Now we have the AppBar UI ready, let's design dropdown menu. 

```kotlin
var expanded by remember { mutableStateOf(false)}

...

Row { // Anchor view
    ... 
    DropdownMenu(
      expanded = expanded, 
      onDismissRequest = {}) {
       // TODO: Popup window content
    }
 }
```



Here, we introduced a new state called `expanded`, which controls the visibility of menu. To show the window, we set it to true otherwise false. We know anchor controls the visibility, so add a click listener there.



```kotlin
  Row(Modifier.clickable { // Anchor view
    expanded = !expanded
  })
```

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/isxrnc9z4jqt7lz0upt4.gif)


Running this, we get ripple - nothing more. This is because, we haven't setup the menu. Let's jump to it. Before adding, the actual content, let's experiment with popup content.



```kotlin
DropdownMenu {
  	Text("Popup content \nhere", Modifier.padding(24.dp))
}
```

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/e3c2ni15per7kpcxtjfx.gif)


Now the menu is shown, but we cannot dismiss it. As we saw earlier, the `expanded` variable is the only thing that can control the visibility. 

**But.. we toggle it on clicking on anchor**. This is because, the click event is not delivered to the anchor (note.. no ripple). A different callback stole it. Let's figure out what to do.



```kotlin
DropdownMenu(expanded = expanded, 
             onDismissRequest = {
               expanded = false
             })
```

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/b8is0ynzcydi7ssn6ba5.gif)


The mandatory callback `onDismissRequest` , should be implemented to dismiss the menu. Otherwise, we'll endup with a sticky popup like in previous step.


...

### Menu item design

We can place any **Composable** view in side the *DropdownMenu*. Current case, we need  list of Label/Text in a column - each clickable. To keep the article short, I'm not demonstrating with a Column and Text list items. Keep these two points in mind and fast forward to the snippet.

- `DropdownMenu` places the given content inside column. So we don't have to add a **Column** manually. From DropdownMenu source:

   ```kotlin
   @Composable
   fun DropdownMenu(
       expanded: Boolean,
       onDismissRequest: () -> Unit,
       ...
       properties: PopupProperties = PopupProperties(focusable = true),
       content: @Composable ColumnScope.() -> Unit
   )
   ```

   

- Likewise, we have `DropdownMenuItem` container, which is a `Row` internally - can take care of placing line item's content horizontally [ex. Country flag - Country name]. Also it ensures minimum height / width requirements met as per material design and has a click listener. From DropdownMenuItemContent source:

   ```kotlin
   @Composable
   internal fun DropdownMenuItemContent(
       onClick: () -> Unit,
       content: @Composable RowScope.() -> Unit
   ) 
   
   ...
   Row()
   .sizeIn(
     minWidth = DropdownMenuItemDefaultMinWidth,
     maxWidth = DropdownMenuItemDefaultMaxWidth,
     minHeight = DropdownMenuItemDefaultMinHeight)
   ```

   

Let's make use of above containers and cook the list items.

```kotlin
DropdownMenu(...) {
		cityList.forEach { city ->
				DropdownMenuItem(onClick = { }) {
					Text(city)
				}
			}
}
```

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/6fkfi9k7mlu2a2hmcdtn.gif)



We get a list with clickable items, but menu not dismissed and value not updated. It's just matter of implementing the `onClick` in menu-item.

```kotlin
cityList.forEach { city ->
   DropdownMenuItem(onClick = { 
     expanded = false
     cityName = city
   }) {
        Text(city)
      }
}
```

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5sb5w6zeks4ezikq1pva.gif)


With bit of style customization to menu item, we can highlight the current city in menu.

```kotlin
val isSelected = city == cityName
val style = if (isSelected) {
    MaterialTheme.typography.body1.copy(
        fontWeight = FontWeight.Bold,
        color = MaterialTheme.colors.secondary
    )
} else {
    MaterialTheme.typography.body1.copy(
        fontWeight = FontWeight.Normal,
        color = MaterialTheme.colors.onSurface
    )
}
Text(text= city, style = style)
```

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/x2nauk29tobby0th3ql2.png)

...

## Conclusion

We can see the benefit of using composable here. In the old view system, Views has it's own saved instance state backed by their ids this had few advantages, yet leaves the system unpredictable in many cases. Devs have to take care of keeping the UI and model in sync.



In compose, data drives the UI. Take a look at *cityName*, initial value set to the component — and then menu item click updates the variable name - not the Text/Label view. This is called unidirectional data flow. Data paints the UI, and UI updates the model. No UI vs UI communication happens here. That means, we don't take value from the UI — rather state can give it to us.