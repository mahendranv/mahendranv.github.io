---
## Common
title: Android — Basic Hilt setup with viewmodel + fragment
tags: [android,hilt,dagger]
# description: 
published: true

## Github pages
layout: post
# image: 
sitemap: false
hide_last_modified: false
no_break_layout: false
categories: [Android, Hilt]


## Dev.to
cover_image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3g1z3iig59e3a5bn7hcq.png
canonical_url: https://mahendranv.github.io/posts/hilt-viewmodel/
series: Hilt

---

Hilt is a modern android DI framework for dependency injection. It is merely a wrapper around Dagger2. Forget [dagger-android](https://www.techyourchance.com/dagger-android-dead/), hilt brings a lot to our plate. This article covers steps to add hilt to the project and use along with viewmodel-fragment. 

...

This is the first installment in three part series.

**Part1: Android — Basic Hilt setup with viewmodel + fragment**

**Part2:** Smoke, mirrors & HiltViewModel

**Part3:** Fakes and espresso

...

Sample project used for this article is available in [github](https://github.com/mahendranv/hilt-espresso).

## 💉 What are we injecting?
For this example, we're going to provide `Profile` (a POJO) to a fragment through `ViewModel`. For simplicity let's not use `LiveData` in here. This is how the dependency graph looks like.

![image](https://user-images.githubusercontent.com/6584143/135462794-dfc37daa-96e7-49f3-bcda-35cc8f911e7d.png)

---

## 🔘 Little about scope

Dependencies could be of different scope (how long it can be in memory/when it can be garbage collected). When we speak about the scope of a dependency, we can easily define it in terms of android components. Below is the oversimplified version of commonly used scopes.

1. A user session details should be available throughout the app (Singleton)
2. I have a tabbed screen and want to share some in-memory fields between fragments (Activity)
3. My data is bound to current screen/fragment. When it is destroyed purge the dependency as well (Fragment)
4. Associate my dependency with ViewModel. Depends on the viewmodel's scope (activity / fragment) let my dependency live (ViewModelScope)

For our use-case, we'll inject DataRepository to the viewmodel using hilt. And there are few improvements on creating viewmodel for the fragment. We'll see it end to end in the following section.

---

## 💻 Code it

### Build setup

Like any framework we write less with hilt because most of the code is `generated` for us. For that purpose, we'll use hilt gradle plugin. And, hilt is expected to be used in multiple modules, so extract out the version to project level gradle file and use it in submodules.

```gradle
// File: build.gradle

buildscript {
    ext {
        hilt_version = '2.38.1'
    }
    repositories {
        // redacted
    }
    dependencies {
        // redacted
        classpath "com.google.dagger:hilt-android-gradle-plugin:${hilt_version}"
    }
}

```

```gradle
// File: app/build.gradle

plugins {
    id 'com.android.application'
    id 'kotlin-android'
    id 'dagger.hilt.android.plugin'
}

dependencies {
    // Hilt
    implementation "com.google.dagger:hilt-android:${hilt_version}"
    kapt "com.google.dagger:hilt-compiler:${hilt_version}"

    // Fragment / viewmodel
    def lifecycle = "2.3.1"
    implementation "androidx.lifecycle:lifecycle-viewmodel-ktx:${lifecycle}"
    implementation "androidx.lifecycle:lifecycle-livedata-ktx:${lifecycle}"
    implementation "androidx.fragment:fragment-ktx:1.3.6"
}
```

In the project level gradle we tell the build system to use hilt. And the app level gradle applies the plugin so that it can skim through our codebase and generate dependencies for us. 

...

### HiltAndroidApp setup (application context)

A singleton is expected to be alive through the app session.  Here, the dependency lives with application. In order to tell hilt about the application scope, create and annotate application class with `HiltAndroidApp`.

```kotlin
import android.app.Application
import dagger.hilt.android.HiltAndroidApp

@HiltAndroidApp
class MyApp : Application()
```

Above step is needed because the application context is provided through generated application class. Make the project and inspect the contents of `Hilt_MyApp` class.

```kotlin
// generated source
DaggerMyApp_HiltComponents_SingletonC.builder()
          .applicationContextModule(new ApplicationContextModule(Hilt_MyApp.this))
          .build();
```

Above steps enables smoother injection of application context to RoomDB / SharedPreference - what not.

...

### Creating data source and modules

Code speaks [thousand words](https://fnune.com/2020/12/05/code-is-worth-a-thousand-words/). Below code block shows resource and the data source.


```kotlin
// A resource
data class Profile(
    val name: String,
    val age: Int
)

// A simple interface which returns the resource. 
// This will help us mock the data source when executing tests.
interface DataRepository {

    fun getProfile(): Profile
}

// Simple implementation of data source
class DataRepoImpl : DataRepository {

    override fun getProfile(): Profile =
        Profile(name = "Bruce Wayne", age = 42)
}

```

From here, we'll work towards creating dependencies that can be recognized by Hilt. 

First, annotate `DataRepoImpl` constructor with `@Inject`.  This puts our class under Dagger/Hilt's radar.

```kotlin
class DataRepoImpl @Inject constructor() : DataRepository {
```

Second, create a module that can provide dependency to view model. 

```kotlin
@InstallIn(ViewModelComponent::class) // Scope our dependencies
@Module
abstract class ProfileModule {

    // To be read as — When someone asks for DataRepository, create a DataRepoImpl and return it.
    @Binds
    abstract fun getProfileSource(repo: DataRepoImpl): DataRepository
}
```

...

### Viewmodel setup

ViewModels can tell ask hilt to provide dependencies. A simple way to ask dependencies is to mark viewmodel with `HiltViewModel` annotation.

```kotlin
@HiltViewModel
class ProfileViewModel @Inject constructor(
    private val repo: DataRepository
) : ViewModel() {

    fun getProfile(): Profile = repo.getProfile()
}
```

Here we're doing constructor injection on viewmodel. Doing the same without Hilt will require a [Factory](https://developer.android.com/reference/androidx/lifecycle/ViewModelProvider.Factory) which pass on the dependencies to the constructor. Reason is the lifecycle of the viewmodel is managed by a lifecycle owner like activity/fragment. Internal mechanics of this injection is covered in last section. On to the fragment...

...

### Fragment setup

Fragment or activity is identified as `AndroidEntryPoint` in hilt. A fragment which is maked with `AndroidEntryPoint` will inject the dependencies without much boilerplate code. And this is our ProfileFragment which consumes the viewmodel.

```kotlin
@AndroidEntryPoint
class ProfileFragment : Fragment() {

    private val viewModel by viewModels<ProfileViewModel>()

    // redacted

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        // Here we go -- the profile resource is shown to the UI
        view.findViewById<TextView>(R.id.name_label)
            .text = viewModel.getProfile().name
    }
}
```

That's it!!

Nothing much changed, instead of creating a factory and use it in `viewModels` delegate we annotated with `AndroidEntryPoint`. And the resource is available to us and components are less coupled now.

![image](https://user-images.githubusercontent.com/6584143/135462680-9b99230e-ae7e-4170-88c9-c7fb5907a40f.png)


## 📖 Resources

- [Sample project source](https://github.com/mahendranv/hilt-espresso)
- [dagger.dev](https://dagger.dev/)
- [Commit: migrating from manual injection to hilt](https://github.com/mahendranv/hilt-espresso/commit/2ea5e386283ee4064d11b73e03fe8b31dfa69c25)