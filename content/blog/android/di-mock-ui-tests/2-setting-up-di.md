---
title: Part 2 - Setting Up Dependency Injection (Hilt)
date: "2022-11-14T00:00:03.000Z"
---

In our demo app, we're not actually making a network request with the `fetchUser()` call to avoid more parts, but if we were, the implementation would still be the same. We'd like to skip calling the real `fetchUser()` in our test, and stub out a response. This way not only are we skipping a request, we'll also have more control within our test.

### ⚠️ Warning on Package Versions

DI requires several packages to work together. If you're just starting your Android app, **I highly recommend you set up DI in the beginning of your project**. Trying to set it up later may require you to try install different versions of packages to find ones that work with your installed `android`, `compose`, `junit` versions.


### Installing Dependencies

We'll need to install 2 sets of dependencies for Hilt:

**1. App dependencies**

Depending on how old this post is when you read this, you may be better off referencing the actual docs here:

https://developer.android.com/training/dependency-injection/hilt-android#setup

Otherwise, feel free to look at the commit, https://github.com/protrackdev/android-setting-up-ui-tests/commit/81fdad6d63d0a49d080dbd04edd4ebab6503f1e7, to see the dependencies used in this post.

**2. Testing dependencies**

We'll need some dependencies to allow Hilt to swap out our implementation in tests. You can find the docs here:

https://developer.android.com/training/dependency-injection/hilt-testing

And the corresponding commit:

https://github.com/protrackdev/android-setting-up-ui-tests/commit/69cbd6bff26ed04fb9ba8cdda4a405dfa85d13b1

#### Summary

This section mostly followed the docs, but here are a few pointers if you get stuck:

- Note that all packages had to have the same version, `2.41` in this case.
- **After installing dependencies, build and run your app to make sure everything still works.**

If you see the following error:

```
The binary version of its metadata is 1.7.1, expected version is 1.5.1.
```

You may be installing hilt dependencies that use a later version of `kotlin` than the one you have installed. Try installing older versions of hilt (and related) packages.

### Using DI in our App

Before we dive in to actually using DI, there's one point clear up: **in a compose app, you use DI to inject classes into view models**. In our case we'll be injecting `AuthService` into our `UserViewModel` as a dependency.

**1. Create and register an `Application` class**

We first need to annotate our `Application` as a `@HiltAndroidApp` for the DI to work. Compose doesn't come with one, so we'll create a new `Application` file:

```kotlin
package dev.protrack.uitestapp

import android.app.Application
import dagger.hilt.android.HiltAndroidApp

@HiltAndroidApp
class UITestApplication: Application()
```

Next we need to tell Android to use `UITestApplication` as our `Application` class:

**AndroidManifest.xml**

```xml
<manifest>
    <application
            ...
            android:name=".UITestApplication"
            >
        ...
    </application>
</manifest>
```

**2. Set `MainActivity` as `@AndroidEntryPoint`**

Pretty self-explanatory, but we need to tell Hilt which is our starting activity.

```kotlin
@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    ...
}
```
**3. Update our `UserViewModel` to receive injected `AuthService`**

Currently we're instantiating our `authService` directly, inside the view model:

```kotlin

class UserViewModel : ViewModel() {
  val authService = AuthService()
  ...
}
```

- `AuthService()` will be created each time `UserViewModel` is created.
- `UserViewModel` knows explicitly how to create an instance of `AuthService`. In this case there are no arguments required, but if there were, they would have to be passed in here.

Now that we have DI, let's ask Hilt to inject `AuthService` instead:

```kotlin
@HiltViewModel
class UserViewModel @Inject constructor(private val authService: AuthService) : ViewModel() {

    var name = mutableStateOf("No User")

    fun load() {
        name.value = authService.fetchUser()
    }
}
```

- Annotated with `@HiltViewModel`.
- We're asking for a `AuthService`, but we're not actually instantiating it.
- The rest of the code is unchanged.

If you tried to build, and run the app now, it will explode unceremoniously, and your editor may look like this:

![compiler error](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/f9s8bef1p70r9q2h1bzk.png)

Don't get thrown off by the generated `.java` file, or the `SingletonC` class. The only error that matters is this: `Dagger/MissingBinding`. Of course the app crashed, we haven't told Hilt how to create our `AuthService`! Let's do that next.

**4. Providing Dependencies**

To tell Hilt how to create our dependencies, we need to define them in a 'module':

**AuthModules.kt** 

```kotlin
@Module
@InstallIn(SingletonComponent::class)
internal object AuthModules {
    @Provides
    @Singleton
    fun provideAuthService(): AuthService {
        return AuthService()
    }
}
```

- `@Module` to tell Hilt that this is a module. Which really just means it's a class that will define how to create dependencies. 
- `@InstallIn(SingletonComponent::class)`, and `@Singleton` to tell Hilt, we only want `AuthService` to be a singleton. You can also use `@InstallIn(ViewModelComponent::class)`/`@ViewModelScoped` to have each view model have their own instance.
- `@Provides` says the following function is going to instantiate a dependency.

If you run the app now, you should see that it works again! 🎉 🎉 🎉  We still see **Real User** on the screen, except now we're getting it via DI!

### Updating Tests to Use DI

If you tried to run our `GetUserTest` again, you'll see that it.... fails ❌:

```
java.lang.RuntimeException: Cannot create an instance of class dev.protrack.uitestapp.UserViewModel
```

Yep. We need to tell our test runner that we're using Hilt. Yay, more boilerplate! 


**1. Create a `HiltTestRunner`**

Similar to what we had to do with `Application`, we need to tell the test runner, `junit`, to use a `HiltTestApplication` for tests.

Create the following file inside your **androidTest** directory:

**HiltTestRunner.kt**

```kotlin
class HiltTestRunner: AndroidJUnitRunner() {
    override fun newApplication(cl: ClassLoader?, name: String?, context: Context?): Application {
        return super.newApplication(cl, HiltTestApplication::class.java.name, context)
    }
}
```

- We're overriding the default `newApplication`, and sending through a `HiltTestApplication` instead.


**2. Register our `HiltTestRunner`**

**app/build.gradle**

```
android {
    defaultConfig {
        ...
        testInstrumentationRunner = "dev.protrack.uitestapp.HiltTestRunner"
    }

  ...
}
```

- The package identifier, **dev.protrack.uitestapp**, needs to match the one on top of **HiltTestRunner**.

**3. Update Test to be a `@HiltAndroidTest` 

We'll now update our test to one that can handle DI.


**GetUserTest.kt**

```kotlin
@HiltAndroidTest
class GetUserTest {

    @get:Rule(order = 0)
    val hiltRule = HiltAndroidRule(this)

    @get:Rule(order = 1)
    val composeTestRule = createAndroidComposeRule<MainActivity>()

   @Test
   fun it_shows_user_name() {
       composeTestRule.setContent {
           HomeScreen()
       }

       composeTestRule.onNodeWithText("Real User").assertIsDisplayed()
   }
}
```

- Need multiple `@get:Rule` so we'll need to use `order`.
- `val hiltRule = HiltAndroidRule(this)` is required (to manage component state).
- We still have a compose rule, but it now needs to know our activity, `createAndroidComposeRule<MainActivity>()`.
- The rest of the test is unchanged.

Go ahead, run the test again, and you'll see that sweet ✅... and we're back to square one. You may be thinking that was a whole bunch of code, and steps just to get back to where we started, you're not wrong, but I promise it's about to get better. Let's look at how we can replace dependencies in our tests next.