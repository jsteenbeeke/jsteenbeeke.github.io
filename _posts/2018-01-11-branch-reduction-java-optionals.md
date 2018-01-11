---
layout: post
title:  "Branch reduction with Optionals"
date:   2018-01-11 14:33:00 +0100
---
One thing I tend to put a lot of effort into is maintaining good test coverage for my projects. It's not always perfect,
or even universally measured, but focusing on coverage means you're spending effort on getting automated tests into your code.

It is important to note, though, that test coverage is a quantitative metric. It tells you how much of your code
is tested, but not if your tests are good at what they do (you'll need to look at mutation coverage for that, but that's
beyond the scope of this article).

To get good code coverage (or more specifically: branch coverage), your tests need to cover all possible
situations. Increasing your branch coverage requires one of two actions:

* Increase the number of tests
* Decrease the number of branches

Using Java 8, there is an interesting technique for the second option, that in my opinion increases readability.
<!--more-->
Consider the following code snippet:

```java
public Baz extractBaz(Foo foo) {
	if (foo != null) {
		Bar bar = foo.getBar();
		if (bar != null) {
			return bar.getBaz();
		}
	}
	
	return null;
}
```

This is a fairly typical piece of code in Java, that does what it's supposed to: navigate through an object graph if
possible. It is, however, not a construct I would encourage.

One of the problem with the above statement is branching. There are no fewer than 4 execution paths in the above pieces of
code, which means your unit tests will have to go through each branch to maintain a good coverage level.

There is a way to reduce the number of branches to just 1, while still maintaining the same functionality:

```java
public Baz extractBaz(Foo foo) {
	return Optional.ofNullable(foo)
	    .map(Foo::getBar)
	    .map(Bar::getBaz)
	    .orElse(null);
}
```

What I've done here is moved the branches outside of our source code, meaning they are no longer under the scope of our test
coverage tools (and the JDK itself has extensive tests for the Optional class).

There's still one problem with this code, and that is that we're still returning `null`. What would be even better is
to return the Optional itself:

```java
public Optional<Baz> extractBaz(Foo foo) {
	return Optional.ofNullable(foo)
	    .map(Foo::getBar)
	    .map(Bar::getBaz);
}
```

Why? Because this will eliminate the null check you were going to have to do from the calling method, thereby
eliminating another branch.

```java
// Old, 2 branches
Baz baz = extractBaz(foo);
if (baz != null) {
	turnUpTheBaz(baz);
}

// New, 1 branch
extractBaz(foo).ifPresent(this::turnUpTheBaz);

// Or why not inline it? Still 1 branch
Optional.ofNullable(foo)
    .map(Foo::getBar)
    .map(Bar::getBaz)
    .ifPresent(this::turnUpTheBaz);
```

As you can see, Optionals allow you to reduce the number of branches while also reducing the number of lines
of code, which has a positive effect on your test coverage metrics, as well as the overall readability.