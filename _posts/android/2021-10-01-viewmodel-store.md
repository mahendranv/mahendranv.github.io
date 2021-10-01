---
## Common
title: Android — ViewModel factory and instantiation
tags: [android,viewmodel]
# description: 
published: true

## Github pages
layout: post
# image: 
sitemap: false
hide_last_modified: false
no_break_layout: false
categories: [Android, ViewModel]

## Dev.to
cover_image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qax7nln9huyz7t4zjano.png
canonical_url: https://mahendranv.github.io/posts/viewmodel-store/
# series:

---

Android ViewModel is one of the most helpful APIs exist in the ecosystem. It's a major release which changed how the Android apps built. Together with LiveData it decoupled the UI from business logic without much boilerplate. This post is about how the viewmodel is instatiated/cached/provided to the caller.

---

## 📝 API doc
Before we get into specific usecases, let's see few key players in this flow.

...

### Factory
Factory is an interface responsible for instantiating a viewmodel. It may or may not inject dependencies to the viewmodel.

...

### ViewModelStore & ViewModelStoreOwner
ViewModelStore keeps a hashmap of String & ViewModel. Once instantiated, viewmodel is stored in it. ViewModelStoreOwner is an interface implemented by Activity/Fragment. Under the hood a ViewModelStore is used by fragment/activity to store the reference.

...

### ViewModelProvider
ViewModelProvider is an umbrella object that coordinates Factory and ViewModelStore. Responsibility of this class is identifying the factory and caching viewmodel reference to ViewModelStore. There could be number of ViewModelProviders created in a fragment/activity. But when a viewmodel is requested, all of them will return the same instance. This is due to ViewModelStore's single source of truth nature.

---

## 🏗️ Instantiation
**Note**: Jetpack recommends `viewModels` / `activityViewModels` delegates to instantiate a ViewModel. However, for clarity this article will explain the usecases through `ViewModelProvider` instances.

### ViewModel without any constructor arguments

This is a rare case where viewmodel doesn't need any dependency. To instantiate such basic viewmodel, create a ViewModelProvider with current activity/fragment reference and invoke `get` method with ViewModel's class.

```kotlin
val viewModel = ViewModelProvider(this) // Create reference wrt current fragment
                 .get(ProfileViewModel::class.java) // query for a ProfileViewModel
```

![image](https://user-images.githubusercontent.com/6584143/135651955-57519633-c183-4081-8ac0-4cd496712ed8.png)

...

### ViewModel with dependencies

Most common pattern with viewmodel is to pass constructor arguments (of data source) and do CRUD and emit events through LiveData. This can be done by creating a custom Factory and use it in place of default one. Internally, ViewModelProvider will use this factory to create the instance and store it to the viewmodelstore. Let's see how it's done.

![image](https://user-images.githubusercontent.com/6584143/135660480-6fa7298a-513b-4dac-80c5-1bc1e2d05461.png)


1. Create a Factory which forwards constructor arguments to the ViewModel. This implements `ViewModelProvider.Factory`.
```kotlin
   class ProfileViewModel(private val repo: DataRepository) : ViewModel() { 

        class Factory(private val repo: DataRepository) : ViewModelProvider.Factory {
            override fun <T : ViewModel?> create(modelClass: Class<T>): T {
                return ProfileViewModel(repo) as T
            }
        }
   }
```

2. Inside the fragment/activity, create a factory instance. And pass it to ViewModelProvider.

```kotlin
val dataRepo = DataRepoImpl() // Data source
val factory = ProfileViewModel.Factory(dataRepo) // Factory
ViewModelProvider(this, factory).get(ProfileViewModel::class.java) // ViewModel
```

...

Interesting thing about the ViewModelProvider is after instantiating the ViewModel in the scope (activity/fragment) next time invoking it will return the same viewmodel. That is 

```kotlin
ViewModelProvider(this).get(ProfileViewModel::class.java)
```

will return the ViewModel created in second step. The order of calls matterns here.

---

## 🤔 Few questions

### How ViewModel survives orientation change?

This is not specific to ViewModel, but any retained object across activity recreation due to configuration change (orientation/locale). Android's activity (yes!! The original activity) has [`onRetainNonConfigurationInstance`](https://developer.android.com/reference/android/app/Activity#onRetainNonConfigurationInstance()) method which can pass around objects between old and recreated activity instances. [`getLastNonConfigurationInstance`](https://developer.android.com/reference/android/app/Activity#getLastNonConfigurationInstance()) can capture the same in newly instantiated activity.

Catch is these methods are deprecated in favor of ViewModel which can achieve the same in a very graceful manner. To answer the question, `ViewModelStore` is passed between the activity instances.

---

### How ViewModel is scoped to single Fragment?

When creating a Fragment with its scope, a nested ViewModelStore is created from its parent scope. This is applicable for Fragment inside an activity or even fragment. Few snippets to clarify things.

```java
// FragmentManager.java
if (host instanceof ViewModelStoreOwner) {
    ViewModelStore viewModelStore = ((ViewModelStoreOwner) host).getViewModelStore();
    mNonConfig = FragmentManagerViewModel.getInstance(viewModelStore);
}

// FragmentManagerViewModel.java
final class FragmentManagerViewModel extends ViewModel {
  
  private final HashMap<String, ViewModelStore> mViewModelStores = new HashMap<>();

  @NonNull
    ViewModelStore getViewModelStore(@NonNull Fragment f) {
        ViewModelStore viewModelStore = mViewModelStores.get(f.mWho);
        if (viewModelStore == null) {
            viewModelStore = new ViewModelStore();
            mViewModelStores.put(f.mWho, viewModelStore);
        }
        return viewModelStore;
  }


 // Fragment.java

 // Internal unique name for this fragment;
 @NonNull
 String mWho = UUID.randomUUID().toString();
}

```

This is how it works,
- Each fragment manager contains a FragmentManagerViewModel
- FragmentManagerViewModel contains a map of string vs viewmodelstore
- string in above step is UUID generated for each fragment and viewmodelstore is the one we've seen in API doc section.

---

### How unique is the key in ViewModelStore?

From ViewModelProvider who puts viewmodels into the ViewModelStore. Key is just the `CanonicalName` of the ViewModel. So, it's purely based in class name. Since every fragment get its own store, it will not collide. Same makes it possible that fragments of the same activity to share viewmodels.

```java
    private static final String DEFAULT_KEY =
            "androidx.lifecycle.ViewModelProvider.DefaultKey";
    
    @NonNull
    @MainThread
    public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
        String canonicalName = modelClass.getCanonicalName();
        if (canonicalName == null) {
            throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
        }
        return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
    }
```


---

## 📒 Resources

- [Activity](https://developer.android.com/reference/android/app/Activity#getLastNonConfigurationInstance())
- [ViewModelProvider](https://developer.android.com/reference/androidx/lifecycle/ViewModelProvider?hl=en)