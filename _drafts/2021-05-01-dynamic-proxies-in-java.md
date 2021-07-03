---
layout: post
title: "Dynamic proxies in Java"
categories: [Java]
---

As a challenge to myself, I decided to implement a little version of [Mockito](https://github.com/mockito/mockito){:target="_blank"} to understand a little better how to work with [dynamic proxies](https://docs.oracle.com/javase/8/docs/technotes/guides/reflection/proxy.html){:target="_blank"} in Java.

## JDK native support

A while back, while going through the code of [async-http-client](https://github.com/AsyncHttpClient/async-http-client){:target="_blank"}, I came across the use of [`Proxy.newProxyInstance()`](https://github.com/AsyncHttpClient/async-http-client/blob/9b7298b8f1cb41fed5fb5a1315267be323c875d6/client/src/main/java/org/asynchttpclient/filter/ReleasePermitOnComplete.java#L30){:target="_blank"}. In that particular case, a proxy is used to release a lock (more specifically a semaphore) when a request is done, for [rate limiting purposes](https://github.com/AsyncHttpClient/async-http-client/blob/9b7298b8f1cb41fed5fb5a1315267be323c875d6/extras/guava/src/main/java/org/asynchttpclient/extras/guava/RateLimitedThrottleRequestFilter.java#L55){:target="_blank"}.

Let's create our static `mock()` method using that:

```java
public static <T> T mock(Class<T> clazz) {
  return (T) Proxy.newProxyInstance(clazz.getClassLoader(), new Class[]{clazz}, (proxy, method, args) -> {
    // do something to lookup the value I should return here and return it
  });
}
```

But, if we try to `mock(Instant.class)`, we get a `java.lang.IllegalArgumentException: java.time.Instant is not an interface` error, which makes very clear what the main limitation of `Proxy.newProxyInstance()` is: it only works for classes that implements interfaces. And having a mocking library that can only mock interfaces is really limited, so we need to find a way to mock classes that do not implement anything.

Digging around the internet made me direct my attention to two projects: [cglib](https://github.com/cglib/cglib){:target="_blank"} and [Bytebuddy](https://github.com/raphw/byte-buddy){:target="_blank"}.

## Bytebuddy and cglib

Both libraries do the same thing: code generation. cglib is older and was created during Java's early days, while Bytebuddy is the new kid in the block (not really new as it was first published in 2014).

The majority of libraries seem to either use Bytebuddy or express the desire to migrate to it:
  - Mockito migrated from cglib to Bytebuddy [in 2015](https://github.com/mockito/mockito/pull/171){:target="_blank"}
  - Spring [uses cglib](https://github.com/spring-projects/spring-framework/blob/b595dc1dfad9db534ca7b9e8f46bb9926b88ab5a/spring-core/spring-core.gradle#L35){:target="_blank"}, but it seems like they have been considering migrating to Bytebuddy [for a while](https://github.com/spring-projects/spring-framework/issues/12840){:target="_blank"}
  - TODO: hibernate?

The main arguments for Bytebuddy seem to be: cleaner interface, better docs and [faster](https://www.jrebel.com/blog/java-code-generation-libraries-comparison){:target="_blank"}, so let's use it here. If you're interested on using cglib, these two articles really helped me understand it:
- [http://blog.rseiler.at/2014/06/explanation-how-proxy-based-mock.html](http://blog.rseiler.at/2014/06/explanation-how-proxy-based-mock.html)
- [http://blog.rseiler.at/2014/06/explanation-how-cglip-proxies-work.html](http://blog.rseiler.at/2014/06/explanation-how-cglip-proxies-work.html)

## Using bytebuddy

// TODO: use Bytebuddy to create a proxy in mock()

Ok, let's try again: `java.lang.IllegalArgumentException: Cannot subclass final class java.time.Instant`.

Instant is a final class and cannot be subclassed. I'm out of ideas, let's look at mockito. What happens if we try to mock a final class with Mockito? Well, we get a pretty error message:

```
org.mockito.exceptions.base.MockitoException: 
Cannot mock/spy class java.time.Instant
Mockito cannot mock/spy because :
 - final class
```

And here is where we finally link the proxy to the code generation concepts.
// TODO: how code generation is related to proxies?

If Mockito didn't solve that problem still, we don't care either and we hit the first restriction of using proxies to implement a mocking framework: final classes cannot be mocked. Let's forget about final classes then, adding a check and throwing an error if someone tries to mock a final class:

```java
if (Modifier.isFinal(clazz.getModifiers())) {
  throw new IllegalArgumentException("A final class cannot be mocked");
}
```

## Mocking non-final classes

What happens if we try to mock a non-final class then?

```java
public static void main(String[] args) {
  CustomService mockedService = mock(CustomService.class);
  mockedService.doSomething();
}

@Slf4j
static class CustomService {
  public void doSomething() {
    log.info("CustomService.doSomething()");
  }
}
```

We finally got our precious log:

```
[main] INFO io.lucaspin.LoggingInterceptor - Method called: doSomething
[main] INFO io.lucaspin.App$CustomService - CustomService.doSomething()
```

## Stubbing

Great, we finally created our proxy. But the main point of a mock is to stub its method returns. How do we do that?

// TODO: when() and .thenReturn()
// TODO: when() and .thenCallRealMethod()

## Verifying

Another important feature of a mock is the ability to verify that the methods that you expected to be called were actually called.

// TODO: verify(), times(), never()

## Conclusion

// TODO