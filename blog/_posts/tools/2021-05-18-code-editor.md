---
## Common
title: A perfect setup for my Github pages project
tags: [tools, markdown, blog]
description: Treat your blog like repository. Write like a coder — commit like a coder. Make tools work for you. Get most out of your github pages and focus on content and let the tools do magic for you.
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
# series: GraphQL backend

---

# A perfect setup for my Github pages project
I hosted my blog in github pages and one thing was constantly bugging me. How can I efficiently get more with writing way less? And focus on the content rather than things like image hosting or previewing it. To start with blogging, lets checkout my setup.

> Treat your blog like repository. Write like a coder — commit like a coder. Make tools work for you.
{:.lead}

If you're not familiar with Jackyll & github pages, there is a [nice article](https://www.smashingmagazine.com/2014/08/build-blog-jekyll-github-pages/) from smashing magazine to quick setup your gihub page.

* toc
{:toc}

## My Focus
1. Distraction free Markdown editing
2. Easy editing of blog configuration [tags, config.yaml, frontmatter etc]
3. Previewing the blog as it is before going live
4. Seamless integration to host my image in the repo 
5. VCS control

## VS Code
My choice of editor is VSCode. I made this decision purely hoping, I'll find a plugin for anything. I wasn't wrong, I did find very nice plugins on my way, it made easier to edit-commit-preview the content.

### Language tools
Once I started content writing, I realized my language could be java/kotlin or javascript. But definetly not English. So, I installed this nice plugin called [LanguageTool](https://marketplace.visualstudio.com/items?itemName=adamvoss.vscode-languagetool), to help me with grammatical mistakes. It struggles to find errors inside markdown format, yet it feels good to fix things before going live.

Press `Option + Return` or `Alt + Enter` as you do for code errors and apply quick fix.

### Distraction free mode
VSCode zen mode is the best when I focus on content editing. Once I enter distraction free mode, all the other panels gone and editor line is highlighted. Still, I get inputs from Language Tools.

### PasteImage — plugin
One of my concern when editing markdown is to include images / gifs in my ReadME file. With image file in my hand, it is not a problem, I can move the file to asset directory and give relative path. But for simple usecase like attaching an image from my clipboard buffer, there is no elegant solution in VSCode — unless you install [PasteImage](https://marketplace.visualstudio.com/items?itemName=mushan.vscode-paste-image) plugin.

Copy a screenshot or image then inside the editor invoke command panel and type PasteImage. Now you have your image inserted next to the MD file and the path is linked. But, for a github page you need assets in a different location. So, I pasted my configuration snippet for pasteImage here.

```json
//File:".vscode/settings.json"

{
    "pasteImage.path": "${projectRoot}/assets/img",
    "pasteImage.insertPattern": "![](/assets/img/${imageFileName})"
}
```

When you insert an image, file saved to `/assets/img` directory and proper relative path added to markdown.
```
![](/assets/img/2021-05-18-12-30-35.png)
```

This might look trivial that, you commit image to your github instead of uploading to an image-host provider like *imgur*. In a long run having control over your image will come in handy. Your repo is your blog is a cool thing.

### Inbuilt console
Since our page is powered by jackyll, it is important to see the page in localhost for any visual artifacts. Though this can be easily done from an external terminal, I prefer to run it inside the IDE.

```bash

bundle exec jekyll serve

## For larger sites — if anything breaks, do a clean bundle.
bundle exec jekyll serve --incremental

```

Start your jackyll build with above command and visit your page in browser at http://localhost:4000/. This way I know how my page will look when going live. Any unsupported liquid tag or ui breakage can be identified in early stages of content writing.

### VCS control
I can do commit — revert — stage content right inside the IDE and push it to repo. This is something useful for a commit triggered publishing system like GitHub pages.

### Recommended extensions
I can right click on an extension and add it to recommendation. A pretty cool feature to consider when you can take all the tools with your blog when moving to a different machine. Listed my favorite extensions for this project below.

<img src="/assets/img/2021-05-18-14-07-25.png" width="50%">

```json

{
    "recommendations": [
        "mushan.vscode-paste-image",
        "dracula-theme.theme-dracula",
        "adamvoss.vscode-languagetool-en",
        "brunnerh.insert-unicode"
    ]
}

```

---
## Noteworthy mention — Typora — my previous editor
Typora is one of the best markdown editors out there for markdown editing. It has a very powerful markdwon rendering engine that updates content inline, so you don't have to worry about how the content will look after you type it. I tried blogging using typora for a while and moved on to VSCode for few reasons
1. In the blog, the spacing I see in typora will be different
2. Lack of language tool [or I haven't figured out that part yet!]
3. Editing the config yaml and going through the directory structure is bit off

I still use typora to take notes and making documentation that will go out as PDF. If your focus is on content alone and don't mind copy-pasting the markdown to your repo, 🛸 go for it.

## Endnote
Above is purely my opinion about the tools. I choose VSCode since, I don't have to leave the IDE to spell check / update the image. All in one place will let me focus on content. It is future proof as VSCode already has bunch of add-ons and I hope they'll add more to the line. 

Since some add-ons are made for available for other editors like Atom, there is no vendor lock here like Typora. Granular control over the IDE experience is an advantage. If any better extension released for markdown, I can use it in conjunction with Zenmode or any other tooling that I mentioned above. Also, keybindings can be created for extensions, it's just awesome.