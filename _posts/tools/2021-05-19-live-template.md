---
## Common
title: Quickguide to intellij live template
tags: [tools, android, kotlin]
categories: [Tools]
description: How to create live templates to pre-populate boilerplate code into your codebase?
published: true

## Github pages
layout: post
image: /assets/img/live_template.gif
sitemap: false
hide_last_modified: false
no_break_layout: false


## Dev.to
# cover_image: 
# canonical_url: 
# series: GraphQL backend

---

LiveTemplate is a feature in Intellij based IDEs where you can expand a code snippet by typing an abbreviation and the IDE guide your cursor through the parts that need your attention (a variable name).

![img](/assets/img/live_template.gif)

Intellij has a whole set of live templates at your disposal. Few of them you might've been using it in day to day basis. Check this list blow. 

1. sout
2. logi
3. soutf
4. soutm
5. appNs
5. psvm and goes on...

> If you haven't used them before, for items 1-4 open a kotlin file ⇨ type the abbreviation and press tab to expand. 

💁‍♂️ You can write one too, it just takes 5 minutes and knowing what is what. Let's jump to it.
<!--  This is what we're trying to achieve. -->
<!-- ![Live template](/assets/img/live_template.gif) -->



* toc
{:toc}


## Know your template

Template is special kind of macro that allows you to enter your input in between / place cursor for you. It is located in Preferences ⇨ Editor ⇨ Live templates section. Let's have a look at `soutf` and analyze what cooks the live template.

![](/assets/img/2021-05-19-09-17-30.png)

```kotlin
// soutf : Prints current class and function name to System.out

println("$CLASS$.$METHOD$")

```

It's just a one-liner where class and method wrapped inside `$`. IDE reads thesee variables and fill in when expanding it. 

### Cursor context

One major thing to design a better template is `Cursor context`. It tells the IDE, when is a Live template is applicable. Expanding a kotlin code inside a `xml` file is totally useless.

For `soutf`, the context is set to work inside Kotlin files, and it can expand inside a function. Try to expand it outside a function. Check the below templates to know various contexts.

| abbreviation | scope      |
| ------------ | ---------- |
| main         | top-level  |
| sout         | statement  |
| fun0         | class      |
| ifn          | expression |

 The Best way understand the scope is to expand all these templates in their destined place in code. That's all about the context, on to the fun part where we edit the `sout` template.


### Placing cursor in template

 The sout template is great, it autofills classname and function name for us. But it places cursor outside the quotes after expanding it. Let's fix it. Add an $END$ to the template and try it now.

 ```kotlin

println("$CLASS$.$METHOD$  $END$")

 ```

That's a bit about scopes and cursor. Let's create our live template for LiveData.

### What is LiveData?
> An android class that connects UI to the Presenter/VieweModel.

 If you're not familiar with LiveData, think of it as an Observable. Any consumer class can attach to it and get updates when the underlying value change. It has two variants, LiveData(observe only model) and MutableLiveData(Superset of LiveData — where you can publish values).

Let me draw out the target code and inline why each line is needed.

```kotlin
    
    // A mutable backing property which is set to private.
    private val _email = MutableLiveData<String>()

    // Frontface to field, the observers will be able to read
    // but, cannot modify
    val emailLiveData : LiveData<String> = _email

    // Syntactic sugar — to validate the data in later part 
    // of the application — use this
    private val email: String?
            get() = emailLiveData.value
    
    // A setter method that updates our mutable property
    // write code to trigger validation in here. This is why
    // we don't expose the mutable version of livedata outside.
    fun updateEmail(value: String?) {
      _email.value = value
      // start your validation flow here
    }

```

Scanning though the live template window in IDE, you might've already figured out how to duplicate/create new live template. Ours will be called `ld`. I'll use the same name going further. Notice that `ld` is scoped to `Kotlin — Class`.


![](/assets/img/2021-05-19-10-52-09.png)

Live Template for LiveData
{:.figcaption}

### Placeholders

In `sout`, we didn't need user input on anything as cursor context can give us the function and class names. That's not the case with `ld` (LD), here we need two inputs from the user. Property name `$PROP_NAME$` and Type `$PROP_TYPE$` — rest can be filled by live template.

```kotlin

private val _$PROP_NAME$ = MutableLiveData<$PROP_TYPE$>()
val $PROP_NAME$LiveData : LiveData<$PROP_TYPE$> = _$PROP_NAME$
private val $PROP_NAME$: $PROP_TYPE$?
        get() = $PROP_NAME$LiveData.value

fun update$PROP_NAME_IN_FUNCTION$(value: $PROP_TYPE$?) {
  _$PROP_NAME$.value = value
}

```

By mentioning these placeholders in different places, we tell the IDE to fill-in the variable wherever we it inside the template. So, when you type email for the `PROP_NAME`, it is filled in for LiveData, MutableLiveData, in the field so on. Same for `PROP_TYPE`.

We have another placeholder called `PROP_NAME_IN_FUNCTION`, which is a capitalized version of our `PROP_NAME`. So, that's about template content, now we have to tell the IDE, what each placeholder is.

Click on *Edit Variables* and input the values as in below table.

| Name      | Expression  |
| --------- | ----------- |
| PROP_NAME |             |
| PROP_TYPE | className() |
| PROP_NAME_IN_FUNCTION | capitalize(PROP_NAME) |

The property name is left to our choice, there is not much IDE can do about it. When filling in PROP_TYPE, IDE will suggest classes. And `PROP_NAME_IN_FUNCTION` is a derived property that capitalizes the `PROP_NAME`(email ⇨ Email). Now we have a live template setup for future use. Create as many as you want and focus on what matters the most. Happy coding ⌨️.