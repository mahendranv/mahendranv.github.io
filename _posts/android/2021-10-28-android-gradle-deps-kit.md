---
## Common
title: Android Utility belt — Collection of dependencies for a greenfield project
tags: [android,gradle]
# description: 
published: true

## Github pages
layout: post
# image: 
sitemap: false
hide_last_modified: false
no_break_layout: false
categories: [Android, Gradle]


## Dev.to
# cover_image: 
# canonical_url: 
# series: 

---

Starting an android project will bring in a new set of challenges 👀 aka adding tons of Gradle dependencies. 

Even though you use these dependencies in almost all your applications, you can't just grab them from the previous project because it might be an outdated version. By the time you find the latest version and add it to the project, you'd lose half the energy. Coming to the point!!

…

In this post, I composed a list of dependencies that I use in most of my projects. They're at the latest version when I write it(Oct 2021). For convenience, hyperlinked the maven central page so that picking the latest version is just a click away.

I highly recommend using this [template project](https://github.com/blocoio/android-template) for greenfield projects. From DI/logger to instrumentation test everything has been perfectly set up. Though, the network dependencies are not present in the template. So, look up below and add dependencies as you see fit.

…

## Network

### OkHttp3

```groovy
implementation(platform("com.squareup.okhttp3:okhttp-bom:4.9.2"))
implementation("com.squareup.okhttp3:okhttp")
implementation("com.squareup.okhttp3:logging-interceptor")

or

implementation 'com.squareup.okhttp3:okhttp:4.9.2'
implementation 'com.squareup.okhttp3:logging-interceptor:4.9.2'
testImplementation("com.squareup.okhttp3:mockwebserver:4.9.2")
implementation 'com.squareup.okhttp3:mockwebserver3:5.0.0-alpha.2'

```



📌[okhttp-bom](https://mvnrepository.com/artifact/com.squareup.okhttp3/okhttp-bom)  &emsp; 📌 [logging-interceptor](https://mvnrepository.com/artifact/com.squareup.okhttp3/logging-interceptor)

📌 [mockwebserver](https://mvnrepository.com/artifact/com.squareup.okhttp3/mockwebserver) &emsp; 📌 [mockwebserver3](https://mvnrepository.com/artifact/com.squareup.okhttp3/mockwebserver3)

…



### Retrofit

```groovy
implementation 'com.squareup.retrofit2:retrofit:2.9.0'

implementation 'com.squareup.retrofit2:converter-gson:2.9.0'
implementation 'com.squareup.retrofit2:converter-moshi:2.9.0'

implementation 'com.squareup.retrofit2:adapter-rxjava2:2.9.0'
implementation 'com.squareup.retrofit2:adapter-rxjava3:2.9.0'

```

📌 [retrofit](https://mvnrepository.com/artifact/com.squareup.retrofit2/retrofit) &emsp; 📌[converter-gson](https://mvnrepository.com/artifact/com.squareup.retrofit2/converter-gson)

📌 [converter-moshi](https://mvnrepository.com/artifact/com.squareup.retrofit2/converter-moshi)  &emsp; 📌 [adapter-rxjava2](https://mvnrepository.com/artifact/com.squareup.retrofit2/adapter-rxjava2)

📌 [adapter-rxjava3](https://mvnrepository.com/artifact/com.squareup.retrofit2/adapter-rxjava3)



---

## Threading and dispatchers

### Coroutines

```groovy
implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-core:1.5.2'
implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.5.2'
testImplementation 'org.jetbrains.kotlinx:kotlinx-coroutines-test:1.5.2'

```



📌[coroutines-core](https://mvnrepository.com/artifact/org.jetbrains.kotlinx/kotlinx-coroutines-core) &emsp; 📌 [coroutines-android](https://mvnrepository.com/artifact/org.jetbrains.kotlinx/kotlinx-coroutines-android)

📌 [coroutines-test](https://mvnrepository.com/artifact/org.jetbrains.kotlinx/kotlinx-coroutines-test)

…

### RXJava

```groovy
  implementation 'io.reactivex.rxjava3:rxjava:3.1.2'    
  implementation 'io.reactivex.rxjava3:rxandroid:3.0.0'

```

📌 [rxjava](https://mvnrepository.com/artifact/io.reactivex.rxjava3/rxjava) &emsp; 📌[rxandroid](https://mvnrepository.com/artifact/io.reactivex.rxjava3/rxandroid)

---



## Serialization

### Moshi

```groovy
implementation 'com.squareup.moshi:moshi:1.12.0'
implementation 'com.squareup.moshi:moshi-kotlin-codegen:1.12.0'

```



📌 [moshi](https://mvnrepository.com/artifact/com.squareup.moshi/moshi) &emsp; 📌[moshi-codegen](https://mvnrepository.com/artifact/com.squareup.moshi/moshi-kotlin-codegen)

…

### GSON

```groovy
implementation 'com.google.code.gson:gson:2.8.8'

```

📌 [gson](https://mvnrepository.com/artifact/com.google.code.gson/gson)



---



## Jetpack

Following dependencies are also available in [Google maven](https://maven.google.com/web/index.html).

### Livedata / viewmodel

```groovy
implementation 'androidx.lifecycle:lifecycle-viewmodel-ktx:2.3.1'
implementation 'androidx.lifecycle:lifecycle-livedata-ktx:2.3.1'

```



📌 [viewmodel-ktx](https://mvnrepository.com/artifact/androidx.lifecycle/lifecycle-viewmodel-ktx)&emsp;  📌 [livedata-ktx](https://mvnrepository.com/artifact/androidx.lifecycle/lifecycle-livedata-ktx)

…

### UI

```groovy
implementation 'com.google.android.material:material:1.4.0'
implementation 'androidx.constraintlayout:constraintlayout:2.1.1'
implementation 'androidx.recyclerview:recyclerview:1.2.0'

```

📌 [material](https://mvnrepository.com/artifact/com.google.android.material/material) &emsp; 📌[constraintlayout](https://mvnrepository.com/artifact/androidx.constraintlayout/constraintlayout)

📌 [recyclerview](https://mvnrepository.com/artifact/androidx.recyclerview/recyclerview)

…

### Fragment

```groovy
implementation 'androidx.fragment:fragment-ktx:1.3.6'

```

📌 [fragment-ktx](https://mvnrepository.com/artifact/androidx.fragment/fragment-ktx)

…

### Navigation

```groovy
implementation 'androidx.navigation:navigation-ui-ktx:2.3.5'
implementation 'androidx.navigation:navigation-fragment-ktx:2.3.5'


androidTestImplementation 'androidx.navigation:navigation-testing:2.3.5'

```

📌 [navigation-fragment-ktx](https://mvnrepository.com/artifact/androidx.navigation/navigation-fragment-ktx) &emsp; 📌[navigation-ui-ktx](https://mvnrepository.com/artifact/androidx.navigation/navigation-ui-ktx)

📌 [navigation-testing](https://mvnrepository.com/artifact/androidx.navigation/navigation-testing)

…

### Room - DB

```groovy

 def room_version = "2.3.0"

implementation "androidx.room:room-runtime:$room_version"
kapt "androidx.room:room-compiler:$room_version"
 // optional - Kotlin Extensions and Coroutines support for Room
implementation "androidx.room:room-ktx:$room_version"
// optional - Test helpers
testImplementation "androidx.room:room-testing:$room_version"

```
📌 [room-ktx](https://mvnrepository.com/artifact/androidx.room/room-ktx)

---


## Dependency injection

### Dagger

Guide: https://developer.android.com/training/dependency-injection/dagger-android

```groovy
// app level Gradle
apply plugin: 'kotlin-kapt'


{
	implementation 'com.google.dagger:dagger:2.39.1'
  kapt 'com.google.dagger:dagger-compiler:2.39.1'
}

```

📌 [dagger](https://mvnrepository.com/artifact/com.google.dagger/dagger)

…

### Hilt

Guide: https://developer.android.com/training/dependency-injection/hilt-android#groovy

```groovy
// project level gradle
buildscript {
    dependencies {
        classpath 'com.google.dagger:hilt-android-gradle-plugin:2.39.1'
    }
}

// app level gradle
apply plugin: 'kotlin-kapt'
apply plugin: 'dagger.hilt.android.plugin'

dependencies {
    implementation "com.google.dagger:hilt-android:2.39.1"
    kapt "com.google.dagger:hilt-compiler:2.39.1"
  
    implementation 'com.google.dagger:hilt-android-testing:2.39.1'
}

```

📌 [hilt-android](https://mvnrepository.com/artifact/com.google.dagger/hilt-android) &emsp; 📌[hilt-android-testing](https://mvnrepository.com/artifact/com.google.dagger/hilt-android-testing)

---

## Image loaders

### Glide

```groovy
implementation 'com.github.bumptech.glide:glide:4.12.0'

```

📌 [glide](https://mvnrepository.com/artifact/com.github.bumptech.glide/glide)

…

### Coil

```groovy
implementation 'io.coil-kt:coil:1.4.0'

```

📌 [coil](https://mvnrepository.com/artifact/io.coil-kt/coil)

---



## Testing

```groovy
testImplementation 'junit:junit:4.13.2'
androidTestImplementation 'androidx.test.ext:junit:1.1.3'
androidTestImplementation 'androidx.test.espresso:espresso-core:3.4.0'

testImplementation 'com.google.truth:truth:1.1.3'

```

📌 [Google truth](https://mvnrepository.com/artifact/com.google.truth/truth)

