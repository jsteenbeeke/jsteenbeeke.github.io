---
layout: post
title:  "When Optionals aren't enough"
date:   2018-01-26 16:06:00 +0100
---
I'm a big fan of [Optionals](https://docs.oracle.com/javase/9/docs/api/java/util/Optional.html), primarily
when it comes to [reduction of branches](/2018/01/11/branch-reduction-java-optionals.html) or the elimination of
null-checks.

But a situation that often arises when I write code is that an operation can yield one of two results:

1. A value of a type I specify
2. A message explaining why the value could not be determined

For traditional Java code, such a scenario is often expressed in methods as follows:
```java
public Type determineType() throws TypeCouldNotBeDeterminedException;
```
While this works, you're now forced to wrap the call to this method in a try/catch block, affecting the readability
of your code. What I would prefer is an approach more in line with functional programming.
<!--more-->
As it turns out, such a construct is already known in functional circles (though I am struggling to find a good source
here), as the Expected monad, which contains either a result, or the exception that prevented the result from being
computed.

That said, I was oblivious to the existence of this construct at the time I faced this problem, so I made my own solution,
which can be found in my "personal toolbox" [Hyperion](https://bitbucket.org/jsteenbeeke/hyperion/branch/experimental),
and is called [TypedResult](https://bitbucket.org/jsteenbeeke/hyperion/src/190ec502a6986e7049f2a46d7ff8cc6bb52b55c8/core/src/main/java/com/jeroensteenbeeke/hyperion/util/TypedResult.java?at=experimental&fileviewer=file-view-default).

The primary way of using this object is as follows:

```java
// Success
TypedResult.ok(result);

// Failure
TypedResult.fail("He's dead, Jim!");
TypedResult.fail("Look %s, I can use String formats!", user.getName());
```

This works for a lot of situations I encountered, but then I started writing code like this:

```java
TypedResult<String> result = determineGreeting();

if (result.isOk()) {
	System.out.println(result.getObject());
}
```

While this doesn't look so bad, it can cause an explosion of branches similar to the one I described in [my
article about Optionals](/2018/01/11/branch-reduction-java-optionals.html), especially if you need to pass
the result of one method that yields a `TypedResult` into another one that also yields a `TypedResult`. To solve
this, I added methods similar to the ones optional has:

```java
determineGreeting().ifOk(System.out::println);
determineGreeting().throwIfNotOk(error -> new IllegalStateException("Could not determine greeting: "+ error));
determineGreeting().orElse(msg -> "Generic greeting because "+ msg);
```

But what I really wanted was to chain them. Let's take the following method signatures:

```java
public TypedResult<T> determineT();
public R determineR(T input) throws IllegalArgumentException;
public TypedResult<V> determineV(R input);
```

Now, assuming we want to use the result of each consecutive method as input for the next one, we would have to write
something like this:

```java
TypedResult<V> result;
TypedResult<T> t = determineT();

if (t.isOk()) {
	try {
        R r = determineR(t.getObject());
	
	    result = determineV(r);
	} catch (IllegalArgumentException e) {
	    result = TypedResult.fail(e.getMessage());
	}
} else {
	result = TypedResult.fail(t.getMessage());
}
```

Which is incredibly ugly. So instead, I created a `map` and `flatMap` operation similar to how they behave in Optional:

```java
TypedResult<V> result = determineT()
    .map(       t -> determineR(t)  )
    .flatMap(   r -> determineV(r)  );
```

Much, much cleaner. If any of the operations in this process fails, it will pass the error message along, while
still yielding a result of the proper type to make the compiler happy.

All in all, this approach has made it much easier for me to report errors to users without having to throw
exceptions through multiple application layers, it has improved the readability and reduced the number of
branches for test coverage purposes.