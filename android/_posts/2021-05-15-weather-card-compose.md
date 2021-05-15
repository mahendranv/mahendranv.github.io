---
# layout: post
# title: Jetpack, Compose, Android, UIDesign
# image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/llkri0z57qisq0vpsb0i.jpg
# accent_image: 
#   background: url('https://res.cloudinary.com/practicaldev/image/fetch/s--gZp_HrPI--/c_imagga_scale,f_auto,fl_progressive,h_420,q_auto,w_1000/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/llkri0z57qisq0vpsb0i.jpg') center/cover
#   overlay: false
# accent_color: '#414a4c'
# theme_color: '#003366'


# invert_sidebar: false


layout: post
title: Jetpack, Compose, Android, UIDesign
image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/llkri0z57qisq0vpsb0i.jpg
description: >
  Exploring Jetpack Compose, Box, Card and Pager.
sitemap: false
hide_last_modified: false
tags: [android, compose]
---

# Weather forecast card design using Jetpack Compose

Horizontal weather cards are the second portion in my forecast screen. It contains a message, relative timestamp and an image depicting the weather. Since I don't have images in handy, I picked one and used for all the cards. I got creative with messages though.

![img](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/s5niaz21lg6kw8cyel06.gif)

* toc
{:toc}

---

## Why box layout?

We have card and an image stacked in z order and they overlap. A constraint layout can be used to build this card, but I wanted to try box. It's like FrameLayout to me. Also using constraint layout for simple layout is an overkill. Be it android native UI or Compose.

---

## Designing a Weather Card

The overall composable blueprint looks like in below image. To understand better, let's build this layout bottom-up. 

<img src="https://i.imgur.com/2KDwWXj.png" alt="image-20210515164325569" style="zoom:80%;" />


...


### Card content

```kotlin
// card content
Column(modifier = Modifier.padding(16.dp)) {

    Spacer(modifier = Modifier.height(24.dp))

    // Time
    Text(
        text = time, // "20 minutes ago",
        style = MaterialTheme.typography.caption
    )

    Spacer(modifier = Modifier.height(8.dp))

    // Message
    Text(
        text = message, // "If you don't want to get wet today, don't forget your umbrella.",
        style = MaterialTheme.typography.body1
    )

    Spacer(modifier = Modifier.height(24.dp))
}
```



Two labels wrapped in a `Column` with spacer makes our card content. Spacers are added in between the elements to make up required spacing between the labels and the parent (`Column`). 



![image-20210515165656824](https://i.imgur.com/JYZ6UAQ.png)


...



### Card design

Now put a card around it and apply few modifiers to it. 

- a shape —  `RoundedCornerShape`
- top padding — to make space for weather icon on top right



```kotlin
Card(
  shape = RoundedCornerShape(16.dp),
  modifier = Modifier
        .padding(top = 40.dp, start = 16.dp, end = 16.dp, bottom = 8.dp)
) {
  Column(...)
}
```



<img src="https://i.imgur.com/krP8dvl.png" alt="image-20210515171455191" style="zoom:90%;" />



...




### Putting all in a Box

Box layout stacks all the elements in given z order. Most recent one will be on the top. So, let's put the Card first then the Image.

```kotlin
Box { 
    Card(...)
    Image(
        painter = painterResource(id = R.drawable.cloudy),
        contentDescription = "weather overlap image",
        modifier = Modifier
            .size(100.dp)
    )
}	
```



<img src="https://i.imgur.com/nT9nSFd.png" alt="image-20210515172249253" style="zoom:50%;" />



Hmm... The rain wasn't supposed to shower there. Let's pull it to the right and shift a bit to the left. Children of Box has access to few properties to help with alignment inside the container.

- alignment — `Alignment.TopEnd`
- offset         — `x = (-40).dp` negative translation in horizontal direction



```kotlin
Image(
    painter = painterResource(id = R.drawable.cloudy),
    contentDescription = "weather overlap image",
    modifier = Modifier
        .size(100.dp)
        .align(alignment = Alignment.TopEnd)
        .offset(x = (-40).dp)
)
```



<img src="https://i.imgur.com/ZZxXMPJ.png" alt="image-20210515180739796" style="zoom:50%;" />



---

## Horizontal pager implementation

[Google accompanist](https://google.github.io/accompanist/) has an incredibly good set of extension for compose. To implement the pager & indicator, we need corresponding dependencies added to our codebase.



```groovy
implementation "com.google.accompanist:accompanist-pager:0.9.0"
implementation "com.google.accompanist:accompanist-pager-indicators:0.9.0"
```



Next step is to wrap the pager and indicator inside a Column and provide a list of mock objects to the render in WeatherCard that we built in above section.



```kotlin
fun WeatherCardCarousal(cards: List<WeatherCard>) {
    val pagerState = rememberPagerState(pageCount = cards.size)
    Column {
        HorizontalPager(
            state = pagerState
        ) { page ->
            WeatherUpdateCard(cards[page])
        }

        HorizontalPagerIndicator(
            pagerState = pagerState,
            modifier = Modifier
                .align(Alignment.CenterHorizontally)
                .padding(16.dp),
        )
    }
}
```



- `pagerState` provides the current index of element for the `HorizontalPager`

- `WeatherUpdateCard(cards[page])` with the given index, corresponding element picked from mock data and rendered in WeatherCard

- `HorizontalPagerIndicator` placed below the pager to react with page swipes in Pager. If it needs to be overlapped on the Pager. We'll wrap them inside a Box and align it.

  

This is again an example for [thinking in compose](https://developer.android.com/jetpack/compose/mental-model). Pager updates the `pagerState`, and `PagerIndicator` reads the same. Both of 'em doesn't know other component exist. Decoupled and yet communicating. 


---

## Complete source

Embedded the entire gist below. If not loaded properly, use this [link](https://gist.github.com/mahendranv/e380359f3c034a5f875ad7cdf3ca40bf).

{% gist e380359f3c034a5f875ad7cdf3ca40bf %}


Happy composing!!