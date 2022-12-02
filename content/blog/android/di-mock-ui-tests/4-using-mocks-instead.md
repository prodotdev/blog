---
title: Part 4 - Using Mocks Instead
date: "2022-11-14T00:00:05.000Z"
---

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