---
## Common
title: Android — Instrumentation test with hilt
tags: [android,hilt,espresso]
# description: 
published: true

## Github pages
layout: post
# image: 
sitemap: false
hide_last_modified: false
no_break_layout: false
categories: [Android, Hilt, Espresso]


## Dev.to
cover_image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/bne9hoetjp3rjueoehaw.png
canonical_url: https://mahendranv.github.io/posts/hilt-instrument/
series: Hilt-Espresso

---

Testing in Android has been a pain from the beginning and there is no standard architecture setup to execute the frictionless test. As you know there is no silver bullet for all the snags. This piece of article covers how do you fake dependencies to your hilt-viewmodel and assert the same from fragment using espresso.

What do we achieve here?
1. Decoupled UI from navigation and real data source
2. Feed fake data to our viewmodel
3. A less flaky test since we control the data

-or- Explained in a single picture

<img src="https://user-images.githubusercontent.com/6584143/138287967-00a8e391-54b1-4f7d-a239-239ac98c12b1.png" alt="image" style="zoom: 30%;" />

> Thanks: [Mr. Bean - Art disaster](https://www.youtube.com/watch?v=QTli1HU9axY&ab_channel=MrBean)

...


## Testing strategy

100% code coverage is a myth and the more you squeeze towards it, you end up with flaky tests. Provided the content is dynamic and involves latency due to API calls, it is simply hard to achieve. So, to what degree we should write tests?

<img src="https://miro.medium.com/max/1400/1*6M7_pT_2HJR-o-AXgkHU0g.jpeg" alt="image" style="zoom:50%;" />

> credits: [medium](https://medium.com/android-testing-daily/the-3-tiers-of-the-android-test-pyramid-c1211b359acd)

- **Unit tests:** Small & fast. Even with a large number of tests, there won't be much impact on execution time.
- **Integration tests**: Covers how a system integrates with others. ex. How viewmodel works with the data source. Cover the touchpoints. Runs on JVM – reliable.
- **UI tests**: Tests that cover UI interactions. In android, this means launching an emulator and running tests in it. Slow! dead slow!! So, write tests to assert fixed UI states. [Wrapup](#wrapup) at the end of the article should give a rough idea of execution time.

Here we cover how to write the integration and UI tests using hilt. Before the actual implementation, take a minute to read on [test double](https://martinfowler.com/bliki/TestDouble.html)s (literally a very short article!). We'll be using Fakes in our tests. Keep in mind while coding:

1. **Hilt resolves a dependency by its type.** So, our code should refer to the *Fake*able dependency as interface.
2. **Provide your dependency in module**. So that the whole module can be faked along with dependencies. More on this is covered in [faking modules](#faking-modules) section

...

## Dependency overview
To recap from [previous article](https://dev.to/mahendranv/android-basic-hilt-setup-with-viewmodel-fragment-32fd), we have a fragment that requires a Profile (POJO) object which is provided through a ViewModel. `DataRepository` acts as a source of truth here and returns a profile. `ProfileViewModel` is unaware of the implementation of `DataRepository` and Hilt resolves it at compile time.

<img src="https://user-images.githubusercontent.com/6584143/138072585-ec3fc907-88d7-40cd-bf2b-2d6cb0a28a98.png" alt="dependency overview" style="zoom:67%;" />

Adding to the existing implementation, we'll bring in our Fake data source `FakeDataRepoImpl` for tests. So, the rest of the post is about instructing Hilt to use `FakeDataRepoImpl` instead of `DataRepositoryImpl`.

---



## Test runner setup

Add test dependencies to app level gradle file. This brings a `HiltTestApplication` and annotation processor to the project now. As we've seen in the [scope section](https://dev.to/mahendranv/android-basic-hilt-setup-with-viewmodel-fragment-32fd#little-about-scope), `HiltTestApplication` will hold singleton component.

```groovy
// File: app/build.gradle

dependencies {
    // Hilt - testing
    androidTestImplementation 'com.google.dagger:hilt-android-testing:2.38.1'
    kaptAndroidTest 'com.google.dagger:hilt-android-compiler:2.38.1'
}
```

Although `HiltTestApplication` is present in our app, it is not used in tests yet. This hookup is done by defining a `CustomTestRunner`. It points to the test application when instantiating the application class for instrument tests.

```kotlin
// File: app/src/androidTest/java/com/ex2/hiltespresso/CustomTestRunner.kt

import androidx.test.runner.AndroidJUnitRunner
import dagger.hilt.android.testing.HiltTestApplication

class CustomTestRunner : AndroidJUnitRunner() {

    override fun newApplication(
        cl: ClassLoader?,
        className: String?,
        context: Context?
    ): Application {
        return super.newApplication(cl, HiltTestApplication::class.java.name, context)
    }
}
```

And the final step is to declare the test runner in app/gradle file.

```groovy
// File: app/build.gradle

android {

    defaultConfig {
    //  testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
        testInstrumentationRunner "com.ex2.hiltespresso.CustomTestRunner"
    }

```
---



## Shared test fakes

In android, there are Unit and Instrumentation tests. Although both identify as tests, they cannot share resources between them. Here, few places where tests will lookup for classes.

1. `main` source set - here we place the production code. Fakes have no place here

2. `testShared` source set - This is the recommended way to share resources. Create the below directory structure and place the `FakeDataRepoImpl` there.

   

![shared-test](https://user-images.githubusercontent.com/6584143/138258099-2215cf0f-ed2a-4269-9981-0f9e2b03c4a8.png)



```kotlin
// File: app/src/testShared/java/com/ex2/hiltespresso/data/FakeDataRepoImpl.kt

class FakeDataRepoImpl @Inject constructor() : DataRepository {

    override fun getProfile() = Profile(name = "Fake Name", age = 47)
}

```

The next step is to add this to `test` and `androidTest` source sets. In app level gradle file, include `testShared` source set to test sources.

```groovy
// File: app/build.gradle

android {
        sourceSets {
        test {
            java.srcDirs += "$projectDir/src/testShared"
        }
        androidTest {
            java.srcDirs += "$projectDir/src/testShared"
        }
    }
}

```
---



## Writing integration test

In the integration test, we'll verify data source and viewmodel coupling is proper. As seen in the testing strategy section,  tests run faster if the emulator is not involved. Here, *ViewModel* can be tested without UI. All it needs is the `DataRepository`. In the last section, we placed the `FakeDataRepoImpl` in the shared source set. Let's manually inject it and assert the data.

```kotlin
// File: app/src/test/java/com/ex2/hiltespresso/ui/profile/ProfileViewModelTest.kt

@RunWith(JUnit4::class)
class ProfileViewModelTest {

    lateinit var SUT: ProfileViewModel

    @Before
    fun init() {
        SUT = ProfileViewModel(FakeDataRepoImpl())
    }

    @Test
    fun `Profile fetched from repository`() {
        assertThat("Name consumed from data source", SUT.getProfile().name, `is`("Fake Name"))
    }
}
```



**💡 Why ProfileViewModel is instantiated manually instead of using Hilt?**

This is the piece of HiltViewModelFactory which instantiates `ProfileViewModel`. The `SavedStateRegistryOwner`, `defaultArgs` are coming from Activity/Fragment that is marked with `@AndroidEntryPoint`. So, instantiating the viewmodel with hilt also brings in the complexity of launching the activity and testing the viewmodel from there. This will result in a slower test whereas viewmodel test can run on JVM like we did above.

```kotlin
public HiltViewModelFactory(
      @NonNull SavedStateRegistryOwner owner,
      @Nullable Bundle defaultArgs,
      @NonNull Set<String> hiltViewModelKeys,
      @NonNull ViewModelProvider.Factory delegateFactory,
      @NonNull ViewModelComponentBuilder viewModelComponentBuilder)
```



---

## Instrumentation or UI tests

Instrumentation/UI tests run on emulator. For this project we'll assert whether the name in UI matches the one in (fake) data source. 

In a real-world application, the data source is dynamic and mostly involves a web service. This means UI tests tend to get flaky. So, the ideal approach is to bring down the UI to have [finite-states](https://en.wikipedia.org/wiki/Finite-state_machine) and map it to the ViewModel and fake it.  

> **<u>Example UI states</u>** :
>
> 1. UI when the network response has failed
> 2. UI when there are N items in the list vs the empty state.

Kotlin [sealed classes](https://kotlinlang.org/docs/sealed-classes.html) are a good fit to design an FSM. The aforementioned use-cases are not covered here!! (and won't fall into a straight line to write an article). So, here is the blueprint on how do we inject our fake for ViewModel. 

…

### Faking modules

For instrument tests (androidTest), hilt is responsible for instantiating the ViewModel. So, we need someone who speaks Hilt's language. Create a fake module that will provide our `FakeDataRepoImpl` to the viewmodel.

```kotlin
// File: app/src/androidTest/java/com/ex2/hiltespresso/di/FakeProfileModule.kt

import dagger.hilt.android.components.ViewModelComponent
import dagger.hilt.testing.TestInstallIn

// Hey Hilt!! Forget about the ProfileModule - use me instead
@TestInstallIn(
    components = [ViewModelComponent::class],
    replaces = [ProfileModule::class]
)
@Module
class FakeProfileModule {

    @Provides
    fun getProfileSource(): DataRepository = FakeDataRepoImpl()

}
```

Notice the `TestInstallIn` annotation. Defining the replaces array will make the original `ProfileModule` replaced with `FakeProfileModule`. While building the component, hilt will replace the original module (and thus dependencies) and instantiate the ViewModel with the fake repo. Our UI will use the viewmodel and tests will assert the same.

…

### The HiltAndroidTest

The final piece is to write a test that uses the faked component. All it needs is a couple of test rules and annotation from Hilt. Rest is generated!!

```kotlin
// File: app/src/androidTest/java/com/ex2/hiltespresso/MainActivityHiltTest.kt

@RunWith(AndroidJUnit4::class)
@HiltAndroidTest
class MainActivityHiltTest {

    @get:Rule(order = 0)
    var hiltRule = HiltAndroidRule(this)

    @get:Rule(order = 1)
    var activityRule: ActivityScenarioRule<MainActivity> =
        ActivityScenarioRule(MainActivity::class.java)

    @Test
    fun test_name_matches_data_source() {
        // Inject the dependencies to the test (if there is any @Inject field in the test)
        hiltRule.inject()
        Espresso.onView(ViewMatchers.withId(R.id.name_label))
            .check(
                ViewAssertions.matches(
                    Matchers.allOf(
                        ViewMatchers.isDisplayed(),
                        ViewMatchers.withText("Fake Name")
                    )
                )
            ))
    }
}
```



The rules are executed first before running the test. It ensures the activity has dependencies resolved before test execution.

1. `HiltAndroidRule` (order = 0) : First create components (singleton, activity, fragment, viewmodel) and obey the `replace` contract mentioned in previous step.
2. `ActivityScenarioRule` (order = 1): Launch the activity mentioned before each test

The espresso & hamcrest matchers are descriptive. In the view tree, it lookup for a view and assert whether it's visible and carries the text defined in fake data repo.

---



## Wrapup

This article gave a blueprint for organizing the code in order to achieve component links in isolation. Apart from the hilt-related setup, this practice could benefit even the code that was built with manual injection. Just follow these key takeaways.

1. Always define the data source as interface. So that it can be faked/mocked for tests.
2. Make `fragment / activity`'s UI controlled by the viewmodel. You don't have to fake the link here.
3. A viewmodel should emit finite states to the UI at any point of time. This state may be dictated by the data source or a reaction to user input in UI (eg. enabling a button based on content length)

…

In case you wonder about the execution time, here it is: 5ms vs 3.5 sec

![junit](https://user-images.githubusercontent.com/6584143/138306415-970c2d37-30df-4ce0-99a1-713740253031.png)



![espresso](https://user-images.githubusercontent.com/6584143/138306231-b0305b0d-9c06-47cc-aa81-59c34cdba1e8.png)

…

## Resources

- [Github project](https://github.com/mahendranv/hilt-espresso)
- [Hilt testing](https://developer.android.com/training/dependency-injection/hilt-testing)
- [Finite-state machine](https://en.wikipedia.org/wiki/Finite-state_machine)
- [Android test pyramid](https://medium.com/android-testing-daily/the-3-tiers-of-the-android-test-pyramid-c1211b359acd)
- [Test doubles](https://martinfowler.com/bliki/TestDouble.html)
