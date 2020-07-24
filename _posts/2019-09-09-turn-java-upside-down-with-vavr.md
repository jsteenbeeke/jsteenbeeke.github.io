---
layout: post
title:  "Turn Java upside down with Vavr"
author: "Jeroen Steenbeeke"
date:   2019-11-01 14:15:00 +0100
---
Last June I gave a presentation about [vavr](https://vavr.io) at [our company's employee developer conference (site in Dutch)](https://topicus.nl/evenementen/topiconf/),
where I gave a small demo about the things you could do with it. This presentation barely scratched
the surface with regard to vavr, but it did start a bit of a chain reaction within the company: they wanted
me to publish a low-tech summary of how the library makes my life better (with some help from an external bureau to dumb it down),
and I was also approached to give a workshop about the library (which took place last Tuesday), and
two more offers to give a presentation which I had to decline due to scheduling conflicts.

The article in question is now live, but it isn't really all that exciting for a technical audience, so
I decided to tell a bit more about Vavr on my own blog.
<!--more-->

## What is it?

Vavr (formerly known as Javaslang) is a library to introduce functional programming concepts into Java.
It introduces a number of new collections and value types based on similar types found in (primarily) Scala.

## Collections

Java has a fairly complete collections library. However, when trying to do any functional
style operations, you run into the Stream API, which results in constructs similar to the following:

```java

List<String> streetNames = persons.stream()
                                  .flatMap(p -> p.getAddresses().stream())
                                  .map(Address::getStreetName)
                                  .collect(Collectors.toList());

```
While I do consider the Stream API a great addition to Java, using it always feels rather clunky to me. Having to call
`stream` twice, and having to call `collect` to turn the stream back into a list always felt unnecessarily complex. This
change was done to maintain backwards compatibility, and while I understand the rationale, it's a lot of extra typing,
and especially the implementation of `flatMap` that requires you the return a Stream annoys me, because you can't use
method references this way. So I started looking for a better collections framework, and ran into Vavr.

Using Vavr's collections, the above code becomes:

```java

List<String> streetNames = persons.flatMap(Person::getAddresses)
                                  .map(Address::getStreetName);

```

Much easier to read, and to type, but there are a few caveats:

* Vavr's collections are immutable. Changes yield new collections, so this requires a slightly different
approach for a lot of code, but in most cases this is an advantage.
* Vavr's collections are persistent, where possible. New collections created as a result of mutation may
still refer to the old collection, and many mutations work by creating a view on the old collection. 

If you're aware of these details you generally don't have any problem working with them. One particular situation
where this may cause issues is if JPA entities are held as part of a Vavr collection, then mapped to one
of their properties, let's say a String. Your resulting list is now of type `List<String>`, and as such, you assume it
to be serializable. However, the list may be a view to the list of JPA entities, which means you're now
serializing the session as well (which may lead to `LazyInitException` if the collection is read in a subsequent request).

Vavr's collections support all the operations that are available in Streams:

```java

List<Address> addresses = persons.flatMap(Person::getAddresses);
List<String> streetNames = addresses.map(Address::getStreetName);
List<String> streetNamesStartingWithA = streetNames.filter(n -> n.startsWith("A"));
String reduced = streetNamesStartingWithA.reduce((a,b) -> a + b);

```

But they also support a bunch of operations that the Stream API does not have:

##### Fold

```java

List<String> names = List.of("A", "B", "C");
String a = names.foldLeft("Start", (current, next) -> current + ", " + next);
System.out.println(a);  // Start, A, B, C
```

##### Scan

```java

List<String> names = List.of("A", "B", "C");

List<String> b = names.scanLeft("Start", (current, next) -> current + ", " + next);

b.forEach(System.out::println); // Start
                                // Start, A
                                // Start, A, B
                                // Start, A, B, C

```

##### Zip

```java

List<String> names = List.of("A", "B", "C");
List<Integer> numbers = List.of(1, 2, 3);

List<Tuple2<String,Integer>> namesAndNumbers = names.zip(numbers);
mamesAndNumbers.forEach(System.out::println); // (A, 1)
                                              // (B, 2)
                                              // (C, 3)


```

## Tuples

The last example also highlights a construct that is missing in Java, namely that of a Tuple. Suppose
you have a function that should return more than 1 value, then you have two options in Java:

* An array of objects: `Object[]`, which removes any type information and necessitaties the use of cast-expressions
* In case you have 2 values to return: `Map.Entry`, which is less than elegant.

Vavr includes Tuple implementations for 0 to 8 values:

```java

Tuple2<String,Integer> tuple2 = new Tuple2<>("C", 3); // (C, 3)
Tuple4<String,Integer,Boolean,Character> tuple4 = new Tuple4<>("D", 4, true, 'x'); // (D, 4, true, x)

tuple4 = tuple4.map3(b -> !b); // (D, 4, false, x)


```

When looking at Vavr you'll see Tuples used in a number of collections-related operations, especially
when it comes to Maps or Multimaps (a concept you may be familiar with from Guava).

## Values

Vavr supports a number of value types, which can be converted from one to another:

* Option: comparable to Java 8's Optional
* Either: a value having either a left (error state) or right (correct state) value
* Try: a value having either a value, or an exception explaining why the value is missing
* Lazy: a value that is not computed until read
* Future: a value that will be computed at some point in the future
* Validation: a value representing input validation of one or more values, with logic to create a new object if valid

## Functions

Java 8 introduced a number of functional interfaces for use with the Stream API, but Vavr introduces a few more. Java has 3 function types:

- Supplier (0 inputs to 1 output)
- Function (1 input to 1 output)
- BiFunction (2 inputs to 1 output)

Vavr has 18 function types:

- Function0, Function1, ..., Function8
- CheckedFunction0, CheckedFunction1, ..., Function8

The first set defines functions with N inputs to 1 output, but whose execution does not yield exceptions. The second
set represents functions according to the same logic, though they may throw exceptions.

These functions types support all operations that Java's functions do, but also a number of features that
Java's function do not:

- Lifting: wrapping a function that would otherwise throw an exception into a function that yields an empty Option instead
- Partial application: instead of applying all parameters at once, you can apply just the first, and the function returns another function
  for the remainder of the parameters
- Currying: like partial application, but it nests your entire function in a chain that each only takes 1 parameter

## Conclusion

Vavr has many features that can make life easier for Java programmers, while not requiring your team
to learn a new language. Granted, it does have a bit of a learning curve, especially if you have no experience
with functional programming.

For me, personally, Vavr is a godsend, and all my personal projects make heavy use of Vavr.

Go check out the [documentation](https://www.vavr.io/vavr-docs/) to learn more, it is very well written.

