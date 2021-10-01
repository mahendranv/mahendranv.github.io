---
## Common
title: Jetpack compose - custom shapes
tags: [android, compose]
# description: 
published: false

## Github pages
layout: post
# image: 
sitemap: false
hide_last_modified: false
no_break_layout: false
categories: [Android, Compose]


## Dev.to
cover_image: 
canonical_url: https://mahendranv.github.io/posts/compose-shape2/
# series:

---

The [previous article](https://mahendranv.github.io/posts/compose-shapes/) covered the out-of-box shapes that mostly cut corners in a rectangle. Although it meets most of the use-cases where a rounded rectangle or corner cut shape is needed, there is always a demand for shapes like tags. So, we'll **build an outline path** and give it to the shape to cut our views.

For this purpose, compose facilitates few options
1. Extending the `Shape` interface
2. Use `GenericShape` and provide a path

Since the GenericShape is a final class and meant for building paths on the go, I'll extend the Shape itself. For this example, I'm covering a car shape from design to implementation.

## Tracing path with Figma
If you have an svg outline path, you can move to next step. Otherwise this one will give a basic idea on path. For better visualization look at the [video](https://vimeo.com/569544993) on how to trace the shape.

1. Download the image that you want to trace the outline.
2. Open figma and create a frame (press F) of image size. Adjust size on design panel on the right.
3. Enable grid for guidelines.
![frame](/assets/img/2021-07-01-00-18-11.png)

4. Paste the image in frame
![car in frame](/assets/img/2021-07-01-00-14-40.png)

5. In the left (layers) panel, select the pasted image. On the design panel, set alpha for the image layer to 10%. We'll start tracing from here.
![layer](/assets/img/2021-07-01-00-26-59.png)


6. Select the pen tool (press P) and click on converging points in the shape. It'll be a straight line, but it's fine!! We'll bend them in the next step.
![car-path](/assets/img/gif/tracing_path1.gif)

7. Double click on the path we drew. It'll highlight the points in the path. Press & Hold the command button and pull each line in the path. Here, the [vimeo link](https://vimeo.com/569544993) for the above steps.



## Let's build a home