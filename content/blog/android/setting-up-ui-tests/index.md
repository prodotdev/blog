---
title: Setting Up Jetpack Compose UI Tests with Hilt (DI) and Mockk
date: "2022-11-14"
description: "Setting up dependency injection, and mocks isn't straightforward in Android. This guide will take you from a brand new project to writing, injecting, and mocking services for your Jetpack Compose app."
---

This post will explain how to get started writing UI tests, with Dependency Injection (DI), and mocks (via Mockk) for a Jetpack Compose app. It uses the following versions:

- Kotlin: 1.7.0
- Junit: 4.13.2
- Compose: 1.1.0-beta01
- Hilt (and related dependencies): 2.41
- mockk: 1.12.1

**⚠️Fair warning, this article is dense by nature of Android ecosystem just requiring a metric ton of boilerplate, dependencies, and xml -- because why the f-not.⚠️**

## Motivation

Just finished setting up tests for an Android app, and I found it required several very specific steps to get things up, and running. Although the documentation was accurate, it mostly assumes you have experience with the current Android ecosystem, and are able to piece together all the required parts. This was not straightforward.

I'm writing this post to help new Android developers, like I am, to get up and running with testing. Hopefully quicker than I did.

## Requirements

I'm going to assume you have previous testing experience in another language, and won't be explaining high-level concepts like Dependency Injection, mocking, or assertions.

We won't be going into the details of other Android features such as View Models, Coroutines, and HTTP Requests, either, as the documentation explains those tools fairly well.

This post is only focused on testing, and it's idiomatic approach for Android.

## The App

In a Compose app, DI works via View Models. So here's our super simple app that represents a typical flow:

```kotlin
@Composable
fun HomeScreen() {
    val userViewModel: UserViewModel = viewModel()

    LaunchedEffect(key1 = true) {
        userViewModel.load()
    }

    Text(text = userViewModel.name.value)
}
```

- The first time `HomeScreen` composes, we tell `userViewModel` to load.

And here's our super simple view model:

```kotlin
class UserViewModel : ViewModel() {
    val authService = AuthService()

    var name = mutableStateOf("No User")

    fun load() {
        name.value = authService.fetchUser()
    }
}
```

- On `load`, we call `fetchUser` on `AuthService`, and update  `name`'s value.

Finally, here's our `AuthService`:

```kotlin
class AuthService  {
    fun fetchUser(): String {
        return "Real User"
    }
}
```

- `fetchUser` would typically be a `suspend` (async) function, but for our purposes here returning a static string is enough.

### App Summary

Ok, so we've got 3 parts:

1. Composable - `HomeScreen`
2. ViewModel - `UserViewModel`
3. Dependency - `AuthService`

Currently, `AuthService` is being instantiated inside `UserViewModel` directly. We want to mock `AuthService` in our test, so that we're not actually calling the real `fetchUser()` function, and instead return some fake response for our test.

## Our First Test

Before we get started on setting up Dependency Injection, let's write a simple test to verify our app so far. It's good practice to write your tests _before_ any major changes, so you can make sure everything still worked as they did before.

```kotlin
class GetUserTest {
    @get:Rule
    val composeTestRule = createComposeRule()

   @Test
   fun it_shows_user_name() {
       composeTestRule.setContent {
           HomeScreen()
       }

       composeTestRule.onNodeWithText("Real User").assertIsDisplayed()
   }
}
```

- Standard Android UI test using `createComposeRule` to compose (render/load/view) our composable.
- Check that the `Real User` text we set before is being displayed on the screen.

## Dependency Injection (DI) with Hilt

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

### Replacing a Dependency in Our test

First we'll need to create a new `@Module` that our tests will use. In your **androidTest** directory create the following file:

**TestModules.kt**

```kotlin
@Module
@TestInstallIn(
    components = [SingletonComponent::class],
    replaces = [
        AuthModules::class,
    ]
)
object TestModules {
   // ... more awesome things to come
}
```

- We're creating another `@Module`
- Similar to `@InstallIn`, we mark it with `@TestInstallIn` to tell Hilt we're wanting to replace modules in a test.
- We tell Hilt to replace (remove) the installed `AuthModules` module that we defined above.

If you try run your test again, you should see the following error:

```
java:8: error: cannot find symbol
```

Don't panic. This just means I know what I'm talking about, and Hilt has removed our module (and dependency) like we told it to. It's now complaining that it's missing a dependency.

Let's add it back, and fix our test:


```kotlin
object TestModules {
    @Provides
    @Singleton
    fun provideAuthService(): AuthService {
        return AuthService()
    }
}
```

We've copy-pasted the same code from `AuthModules`'s `provideAuthService()`, and if you re-run your test, you'll see we're back to ✅ again. Yay!!

Ok, so how do we actually replace `AuthService()` with a fake/mock in our test? Currently we're still calling `AuthService()` in our `TestModules`, and we're _still_ seeing `Real User` on the screen.

Before we implement, let's update our test to match what we expect. Currently we've got:

```kotlin
   @Test
   fun it_shows_user_name() {
       ...
       composeTestRule.onNodeWithText("Real User").assertIsDisplayed()
   }
```

Let's update it to check for `Fake User` instead:


```kotlin
   @Test
   fun it_shows_user_name() {
       composeTestRule.setContent {
           HomeScreen()
       }

       composeTestRule.onNodeWithText("Fake User").assertIsDisplayed()
   }
```

And if we re-run our test, it fails as expected. Good. Let's update our test to send in a fake auth service. There's just one problem, we can't! If you take a look at `AuthService`:

```kotlin
class AuthService  {
    ...
}
```

It's a `class`. That means every time we ask Hilt to give us an `AuthService` it has to be the real deal. That's not going to work. You're about to see why they say "you should bind to interfaces". Instead of telling Hilt we want a concrete `AuthService` class, we're going to say we want an `AuthService` _interface_, then provide real/fake implementation as required. Sound complicated? The code is pretty simple, check it out:

1. Rename our `AuthService` to `HttpAuthService`:

```kotlin
class HttpAuthService  {
    ...
}
```

2. Create a new interface called `AuthService`:

```kotlin
interface AuthService {
    fun fetchUser(): String
}
```

3. Update `HttpAuthService` to implement our new interface:

```
class HttpAuthService: AuthService  {
    override fun fetchUser(): String {
        return "Real User"
    }
}
```

4. Update `UserViewModel` to expect `AuthService` (interface)

```
@HiltViewModel
class UserViewModel @Inject constructor(private val authService: AuthService) : ViewModel() {
  ...
}
```

5. Update `AuthModules` to provide `AuthService`

```kotlin
internal object AuthModules {
    @Provides
    @Singleton
    fun provideAuthService(): AuthService {
        return HttpAuthService()
    }
}
```

We're returning the real implementation, `HttpAuthService`.

6. Create a `FakeAuthService` in **androidTests**

**FakeAuthService.kt**

```kotlin
class FakeAuthService: AuthService {
    override fun fetchUser(): String {
        return "Fake User"
    }
}
```

- Implements `AuthService` just like our real `HttpAuthService` does.
- Returns `Fake User` like we're expecting in the test.

7. Update `TestModules` to return `FakeAuthService`


```kotlin
object TestModules {
    @Provides
    @Singleton
    fun provideAuthService(): AuthService {
        return FakeAuthService()
    }
}
```

- Updated `provideAuthService` to return `AuthService`.
- Return `FakeAuthService`, everything else stayed the same.

8. Re-run tests, and .... ✅

Congratulations!! That was a LOT but you did it. We got DI up, and running in Android in 50 easy steps. 

Go grab a well deserved celebratory drink because we're not done yet! Keep reading to find out how we can use **Mockk** (the title wasn't a typo), a mocking library to assert (check) that our functions were called as we expected.

### Using Mockk to assert function calls

First, we'll need to add `mockk` as a dependency for our instrumented tests:

**build.gradle**

```
dependencies {
  ...
  androidTestImplementation "io.mockk:mockk-android:1.12.1"
}
```

Next, we're going to use `mockk` to create our `FakeAuthService` instead:

**TestModules.kt**

```kotlin
val mockAuthService = mockk<AuthService>(relaxed = true)

...
object TestModules {
    ...
    fun provideAuthService(): AuthService {
        return mockAuthService
    }
}
```

- Created a mocked `AuthService` using `mockk`.
- Delete `FakeAuthService`, and replace it with our mock, `mockAuthService`.

Now, we need to tell our mock, `mockAuthService`, what to do whenever somebody calls `fetchUser()`:

**GetUserTest.kt**

```
fun it_shows_user_name() {
  every { mockAuthService.fetchUser() } returns "Fake User"

  composeTestRule.setContent {
      HomeScreen()
  }

  composeTestRule.onNodeWithText("Fake User").assertIsDisplayed()
}
```

- We stub out the `Fake User` response by using `every`.

Re-run the test, and it should still pass 🚀. You did it!!

#### Asserting Function Calls

Other than being able to quickly stub out answers, **Mockk** also lets us assert functions were called:

**GetUserTest.kt**
```
fun it_shows_user_name() {
  ...
  verify {
      mockAuthService.fetchUser()
  }
}
```

- Add `verify` block to make sure `fetchUser` was called.

The test should still pass, except now we're asserting that `fetchUser` was called. Useful!


#### Common Build Errors

Depending on your versions, you may run into a build-time errors due to dependency versions (welcome to Java). Here are a couple that I ran into, and how to fix them:

- **`MethodHandle.invoke and MethodHandle.invokeExact are only supported starting with Android O (--min-api 26)`**

Mockk requires at least sdk version `26`, so let's update it:

**build.gradle**

```
android {
  ...
  defaultConfig {
    ...
    minSdk 26
}
```

- **`library "libmockkjvmtiagent.so" not found`**

Need to tell native libraries to use legacy packaging, compressing all. Not 100% on what this means, but it works 🙂.

**build.gradle**

```
android {
  ...
  packagingOptions {
    jniLibs.useLegacyPackaging = true
  }
}
```

### Summary

Great work, you've reached the end! I hope I was able to break that down a little bit,