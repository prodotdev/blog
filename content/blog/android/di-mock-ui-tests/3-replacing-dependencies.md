---
title: Part 3 - Replacing Dependencies
date: "2022-11-14T00:00:04.000Z"
---

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
