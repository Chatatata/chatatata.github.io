---
layout: post
title:  "Elegant ways to do mocking in Kotlin"
date:   2018-12-25 21:21:16 +0300
categories: mocking kotlin spring framework mockito powermock gradle functional programming generics
image: /assets/article_images/2018-12-25-elegant-way-to-setup-mocks-in-kotlin/shebang.jpg
---
Mocking is not elegant itself, though. Kotlin, on the other hand, can make anything look elegant.
-----

Assume that we have a class named `PushyNotification` with one dependency, `Notification`.
To do this, we interpret the implementation of `PushyNotification` and mock up the dependency.

```kotlin
class PushyNotificationTest {
  //  Dependencies
  lateinit var notification: Notification

  //  Mocks
  lateinit var pushyNotification: PushyNotification

  @BeforeEach
  fun setUp() {
    notification = `mock`(Notification::class.java)

    `when`(notification.token)
      .thenReturn(TOKEN)
    `when`(notification.expiration)
      .thenReturn(EXPIRATION_DATE)
    `when`(notification.qualityOfService)
      .thenReturn(QUALITY_OF_SERVICE)
    `when`(notification.collapseId)
      .thenReturn(COLLAPSE_ID)

    pushyNotification = PushyNotification(notification, TOPIC, false)
  }

  companion object {
    private val TOKEN = "token"

    private val EXPIRATION_DATE = Date.from(Instant.EPOCH)

    private val QUALITY_OF_SERVICE = QualityOfService.TRANSACTIONAL

    private val COLLAPSE_ID = "collapse id"
  }
}
```

We're gonna implement two helper functions. Hence, we could create our mock as a one-liner,
and we could avoid stating `notification` in each call to the `when` of Mockito.

#### Phase I: Kotlin-ish `mock`

We're not able to use to good-old `mock` static method of Mockito, since it requires a Java class
which will be available at runtime. So, this is impossible.

```kotlin
class PushyNotificationTest {
  ...

  //  Mocks
  val pushyNotification = `mock`(Notification::class.java)     //  compile-time error

  ...
}
```

Using *reified* generics of Kotlin, this can be fixed with the given implementation.

```kotlin
public inline fun <reified T: Any> mock() = Mockito.mock(T::class.java)
```

We can now declare the mock in a one-liner.

```kotlin
class PushyNotificationTest {
  ...

  //  Mocks
  val pushyNotification: PushyNotification = `mock`()

  ...
}
```

We're explicitly declaring our type here, but I think it's completely understandable. We could
therefore understand the mocks' actual type, in future.

#### Phase II: Contextual mockups

No matter what, I think, this looks ugly.

```kotlin
    ...
    `when`(notification.token)
      .thenReturn(TOKEN)
    `when`(notification.expiration)
      .thenReturn(EXPIRATION_DATE)
    `when`(notification.qualityOfService)
      .thenReturn(QUALITY_OF_SERVICE)
    `when`(notification.collapseId)
      .thenReturn(COLLAPSE_ID)
    ...
```

Thanks to contextual closures, we could bind another `this` to a block on a function.

```kotlin
public inline fun <T> T.apply(block: T.() -> Unit): T { block(); return this }
```

This is not implemented by me, but standard library. We could utilize this function to
setup or mock objects.

```kotlin
class PushyNotificationTest {
  val notification: Notification = mock()

  lateinit var pushyNotification: PushyNotification

  @BeforeEach
  fun setUp() {
    notification.apply {
      `when`(token)
        .thenReturn(TOKEN)
      `when`(expiration)
        .thenReturn(EXPIRATION_DATE)
      `when`(qualityOfService)
        .thenReturn(QUALITY_OF_SERVICE)
      `when`(collapseId)
        .thenReturn(COLLAPSE_ID)
    }

    pushyNotification = PushyNotification(notification, TOPIC, false)
  }
  ...
}
```

