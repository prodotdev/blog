---
title: Setting Up Jetpack Compose UI Tests with Hilt (DI) and Mockk
date: "2022-11-14T00:00:01.000Z"
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