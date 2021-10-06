---
## Common
title: Smoke, mirrors & HiltViewModel
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
cover_image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/a9kbhplbw5haziql5lrp.png
canonical_url: https://mahendranv.github.io/posts/hilt-codegen/
series: Hilt-Espresso

---

This is the second installment in the three-part series. To understand better, you can read part1 or open the [github project](https://github.com/mahendranv/hilt-espresso) to explore the code.

[**Part1:** Android — Basic Hilt setup with viewmodel + fragment](https://mahendranv.github.io/posts/hilt-viewmodel/)

**Part2: Smoke, mirrors & HiltViewModel**

**Part3:** Fakes and espresso

...

## Introduction
In a typical Android project creating a ViewModel with dependencies require us to provide an explicit viewmodel-factory. However, in the previous post Hilt was able to create one without all the boilerplate. This post covers the smoke and mirrors behind viewmodel instantiation with hilt. 

As I write and revise the article, noticed it has a lot of moving parts. So, I tried to segregate it into four portions.

1. Dagger: Hilt is based on dagger, so this portion covers few dagger concepts
2. HiltViewModel: Hilt marks viewmodels with this annotation. What happens when you do this marking.
3. AndroidEntryPoint: A Fragment/Activity is called as AndroidEntryPoint in Hilt. This again bootstraps few things for us.
4. HiltViewModelFactory: It's better to read this at the very end.

...

## 🗡️  Stabbing with Dagger
In order to understand the mechanics of Hilt, let's recall few Dagger components.

### Providers & Factories

Our focus is on how the ViewModel is instantiated. So, this portion covers enough dagger to demonstrate a constructor injection. Let's see an example of how Car gets its Engine. Engine has no dependencies and car will need one Engine to work. The same can be written in code like this.

```kotlin
import javax.inject.Inject

// Hey Dagger! Create a factory for me, so that you can create my instances.
class Engine @Inject constructor() {
    fun start() { println("Engine.start") }
    fun stop() { println("Engine.stop") }
}

// Hey Dagger! Create a factory for me, so that you can create my instances.
// Also provide me with my dependencies.
class Car @Inject constructor(private val engine: Engine) {
    fun start() = engine.start()
    fun stop() = engine.stop()
}
```

...

For this piece to work, dagger generates `Factory`s. There are three things to remember here.

1. Factory is an interface defined in Dagger with only one getter to the target object. 
2. A Factory can include `Provider`s which can provide the dependencies for it. For example, Car factory will need a Provider for Engine.
3. Factory is extended from `Provider`. Which means Engine factory can act as a provider for car factory.

```java
// source: Dagger
public interface Provider<T> {
    /**
     * Provides a fully-constructed and injected instance of {@code T}.
     */
    T get();
}

public interface Factory<T> extends Provider<T> {}
```
...

This is the overview of dagger-factory for Car & Engine usecase.

![image](https://user-images.githubusercontent.com/6584143/135588702-8afdcbcc-8d49-436f-a175-ebb1e90e94b1.png)

...

Engine_Factory is simple. It extends dagger Factory and returns new instance when `get()` is called.

```java
import dagger.internal.DaggerGenerated;
import dagger.internal.Factory;

@DaggerGenerated
public final class Engine_Factory implements Factory<Engine> {
  @Override
  public Engine get() {
    return newInstance();
  }

  public static Engine newInstance() {
    return new Engine();
  }
  // redacted
}
```

`Car_Factory` needs an engine to create car. For this purpose, it needs a `Provider` which can get an engine for it. The car provider is defined as a constructor parameter and Dagger uses components & scopes to resolve it.

```java
@DaggerGenerated
@SuppressWarnings({
    "unchecked",
    "rawtypes"
})
public final class Car_Factory implements Factory<Car> {
  private final Provider<Engine> engineProvider;

  // Engine provider will be set through dagger module/components
  public Car_Factory(Provider<Engine> engineProvider) {
    this.engineProvider = engineProvider;
  }

  @Override
  public Car get() {
    return newInstance(engineProvider.get());
  }

  public static Car newInstance(Engine engine) {
    return new Car(engine);
  }
}
```
...

### Dagger binds

```java

public class Circle extends Shape {
  // redacted
}

@Module
interface ShapeModule {

  @Binds
  public abstract Shape binds(Circle c);
}
```

`Binds` is a contract that tells dagger to return the argument in place of the return type. The above statement is to be read as "when `Shape` is requested, return `Circle`". This could work well for one-to-one mapping where the abstraction is only there to loosely couple the implementation.


### Dagger map-multibindings

It is a common practice in programming where we keep a lookup registry and query an object by a unique key. [Dagger map](https://dagger.dev/dev-guide/multibindings.html) is the same, with a decoupled approach.

1. Map instance is scoped to component
2. Map elements are managed at compile time. That means multiple modules can contribute to the map without knowing about each other. In fact, they don't get the map instance to look up in the first place.
3. Map is an injectable dependency for the consumer. This consumer should have the knowledge about which key to lookup.

In the above example, whenever we need a `Shape`, `Circle` will be returned. What if I want a square?
Dagger multibinding can help with that.

```java
@Binds
@IntoMap
@StringKey("circle")
public abstract Shape binds(Circle c);

@Binds
@IntoMap
@StringKey("square")
public abstract Shape binds(Square c);

@Component
public interface MyComponent {
   
   Map<String, Shape> getShapesRegistry();
}


// target class

MyComponent myComponent;

myComponent.getShapesRegistry().get("square") // Square
myComponent.getShapesRegistry().get("circle")

```

The above code places both Square and Circle in the component's map `Map<String, Shape>`. Target class shall request this map from component and access the instance using a key string.

---

## 🎞️  Recap
Dagger section should give some idea of what happens with constructor injection & multi-bindings. This will help us go through the hilt workflow quickly. Recalling few things from the previous article:

- Sample project [github](https://github.com/mahendranv/hilt-espresso)
- Viewmodel-fragment dependency graph

![image](https://user-images.githubusercontent.com/6584143/135462794-dfc37daa-96e7-49f3-bcda-35cc8f911e7d.png)

We'll cover the workings of HiltViewModel and AndriodEntryPoint in the rest of the article.

---

## 🪡 HiltViewModel — annotation

When we mark a ViewModel as `HiltViewModel`, Hilt creates a module and dagger factory for it. This factory is something we've seen briefly in dagger section. Our viewmodel requests for a data-repository and respective provider is placed in the factory.

```kotlin
@HiltViewModel
class ProfileViewModel @Inject constructor(
    private val repo: DataRepository
) : ViewModel()
```

generates

```java
import dagger.internal.Factory;

@DaggerGenerated
public final class ProfileViewModel_Factory implements Factory<ProfileViewModel> {
  private final Provider<DataRepository> repoProvider;

  //redacted

  @Override
  public ProfileViewModel get() {
    return newInstance(repoProvider.get());
  }

  public static ProfileViewModel newInstance(DataRepository repo) {
    return new ProfileViewModel(repo);
  }
}
```

The factory resolves dependencies of the viewmodel. And it can create `ProfileViewModel` if someone requests it. The next step is to expose this factory to the outer layer. For this hilt generates a dagger module for each HiltViewModel. These modules will register themselves to Hilt's ViewModelComponent. Here, ViewModelComponent keeps a multibinding map of `Map<String, ViewModel>`. And the key is the fully qualified class name.

```java
public final class ProfileViewModel_HiltModules {
  
  @Module
  @InstallIn(ViewModelComponent.class)
  public abstract static class BindsModule {
    
    @Binds
    @IntoMap
    @StringKey("com.ex2.hiltespresso.ui.profile.ProfileViewModel")
    @HiltViewModelMap
    public abstract ViewModel binds(ProfileViewModel vm);
  }
}

```
So, our viewmodel code is read as **bind** ProfileViewModel for ViewModel and register to the component's map (**IntoMap**) against the **StringKey** which is `com.ex2.hiltespresso.ui.profile.ProfileViewModel`.

...

Till now, we covered the viewmodel instantiation and how ViewModelComponent keeps track of ViewModels. This is the producer side of the equation and the consumption is covered in the rest of the article.

---

## 🎯 AndroidEntryPoint

> <big>**Note**: In this article run into Dagger Factory & ViewModel Factory. So, to avoid ambiguity - lets call ViewModel Factory as VMFactory.</big>

In a traditional viewmodel usecase, fragment/activity (UI) takes responsibility to provide VMFactory if there is any dependency. [This article](https://dev.to/mahendranv/how-viewmodel-survives-configuration-change-67p#viewmodel-with-dependencies) covers how VMFactory instantiates viewmodel with dependencies.

![image](https://res.cloudinary.com/practicaldev/image/fetch/s--Vw5-IH9---/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://user-images.githubusercontent.com/6584143/135660480-6fa7298a-513b-4dac-80c5-1bc1e2d05461.png)

...

Hilt augments the existing viewmodel framework by generating VMFactory for us. For this, it uses marker annotations `AndroidEntryPoint` and `HiltViewModel`. 

AndroidEntryPoint is a Hilt annotation to mark an Activity or Fragment as a receiver of the dependency. So, things discussed here with fragment will work for activity as well. 

...

Our ProfileFragment is extended from Fragment right? Only it is not!! `hilt-android-gradle-plugin` gradle plugin will generate a proxy fragment and set it as base for ProfileFragment. Here, what we see vs the one that goes into the APK.

```kotlin
@AndroidEntryPoint
class ProfileFragment : Fragment()

vs

class ProfileFragment : Hilt_ProfileFragment()
```

...

Hilt uses the generated fragment to hook up its `HiltViewModelFactory`. This VMFactory has access to the viewmodel component and the rest of the dependency graph. Following block diagram shows the components and relations. Let's go through each of them.

![image](https://user-images.githubusercontent.com/6584143/135807886-b7ae5e81-db4f-4995-92f8-d71e7dee4f1a.png)


### Hilt_ProfileFragment — the generated fragment
The generated fragment injects the dependencies through the member injector and overrides the default VMFactory with HiltViewModelFactory. ViewModelProvider will recognize the HiltViewModelFactory and use it to create ProfileViewModel. There are in-line comments in respective places for understanding the generated fragment.

```java
public abstract class Hilt_ProfileFragment extends Fragment implements GeneratedComponentManagerHolder {

  public void onAttach(Context context) {
    inject();
  }

  public void onAttach(Activity activity) {
    // This will inject surface level dependencies (ex. sharedpreference 
    // or analytics client needed for reporting click etc.) 
    // used in view layer.
    inject();
  }

  protected void inject() {
    if (!injected) {
      injected = true;
      ((ProfileFragment_GeneratedInjector) this.generatedComponent()).injectProfileFragment(UnsafeCasts.<ProfileFragment>unsafeCast(this));
    }
  }

  // This method will be invoked by ViewModelProvider in case there is 
  // no explicit VMFactory is provided. Hilt uses this method to enroll its
  // HiltViewModelFactory with ViewModelProvider.
  @Override
  public ViewModelProvider.Factory getDefaultViewModelProviderFactory() {
    // DefaultViewModelFactories is a utility class that delegates calls to 
    // hilt generated viewmodel registry which created in previous section
    return DefaultViewModelFactories.getFragmentFactory(this, super.getDefaultViewModelProviderFactory());
  }
}

  // File: DefaultViewModelFactories.java

  /**
   * Retrieves the default view model factory for the activity.
   * <p>Do not use except in Hilt generated code!
   */
  public static ViewModelProvider.Factory getFragmentFactory(
      Fragment fragment, ViewModelProvider.Factory delegateFactory) {
    return EntryPoints.get(fragment, FragmentEntryPoint.class)
        .getHiltInternalFactoryFactory()
        .fromFragment(fragment, delegateFactory);
  }
```


### InternalFactoryFactory

`InternalFactoryFactory` can is responsible for forwarding the ViewModelComponent builder and list of known hilt viewmodel names to the HiltViewModelFactory. Here, the ViewmodelComponent is a dagger component defined by hilt. We have already provided it with dependencies for `ProfileViewModel`.


```java
public static final class InternalFactoryFactory {

    private final Application application;
    private final Set<String> keySet;
    private final ViewModelComponentBuilder viewModelComponentBuilder;

    private ViewModelProvider.Factory getHiltViewModelFactory(
        SavedStateRegistryOwner owner,
        @Nullable Bundle defaultArgs,
        @Nullable ViewModelProvider.Factory extensionDelegate) {
      ViewModelProvider.Factory delegate = extensionDelegate == null
          ? new SavedStateViewModelFactory(application, owner, defaultArgs)
          : extensionDelegate;
      return new HiltViewModelFactory(
          owner, defaultArgs, keySet, delegate, viewModelComponentBuilder);
    }
}
```

---

## 🧩 Final piece - The HiltViewModelFactory
`HiltViewModelFactory` is a subclass of ViewModelFactory. Responsibility of this class is to instantiate the given ViewModel. This block diagram explains the control flow

![image](https://user-images.githubusercontent.com/6584143/135831324-dce01038-9e08-4c5d-9481-93a0fa628623.png)

1. InternalFactoryFactory forwards the set of viewmodel names and a viewmodel component builder to HiltViewModelFactory through constructor. Now the `HiltViewModelFactory` has been created and await the consumer to call create.
2. Fragment/activity calls create to get instance of the viewmodel. Now, HiltViewModelFactory builds a viewmodel component from the existing builder and asks for a map of provider<ViewModel> instances.
3. From there, it picks the matching provider and uses it to instantiate the viewmodel.
4. Delivers it to the ViewModelProvider


Code looks like this: HiltViewModelFactory holds a list of known hilt viewmodel names and hiltViewModelFactory and an optional delegateFactory. List of ViewModel names  determine whether to use `delegateFactory` or `hiltViewModelFactory`.

```java
public final class HiltViewModelFactory implements ViewModelProvider.Factory {
  private final Set<String> hiltViewModelKeys;
  private final ViewModelProvider.Factory delegateFactory;
  private final AbstractSavedStateViewModelFactory hiltViewModelFactory;

  // redacted

  @Override
  public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
    if (hiltViewModelKeys.contains(modelClass.getName())) {
      return hiltViewModelFactory.create(modelClass);
    } else {
      return delegateFactory.create(modelClass);
    }
  }
}
```

---

## 🍬 Wrap up

- Hilt creates a Dagger Factory for the ViewModel. This contains the providers for the particular viewmodel's dependencies and a key that uniquely identifies it (fully qualified name). This is a registry step that holds info on a ViewModel's name and how to create it.
- When a fragment is marked as AndroidEntryPoint, it will be set with a generated base class that takes care of injecting fields to the fragment.
- However, ViewModel is a special case. So, hilt will try to provide a ViewModelFactory.
- Above step is done with `InternalFactoryFactory`. It creates `HiltViewModelFactory` and passes on the component to it. 
- HiltViewModelFactory lookup in the registry (created in the first step) and instantiates the requested viewmodel.


---

## 📖 Resources

- [Sample project source](https://github.com/mahendranv/hilt-espresso)
- [Dagger multi-binding](https://dagger.dev/dev-guide/multibindings.html)
- [ViewModel instantiation](https://dev.to/mahendranv/how-viewmodel-survives-configuration-change-67p)