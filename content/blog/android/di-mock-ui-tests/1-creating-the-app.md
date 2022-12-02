---
title: Part 1 -  App Overview
date: "2022-11-14T00:00:02.000Z"
---

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
