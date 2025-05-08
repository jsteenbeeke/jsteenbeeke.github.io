---
layout: page
title: Modern Java please
permalink: /recruiters/modern-java-please
---

As of writing this bit, the [most recent Java version](https://en.wikipedia.org/wiki/Java_version_history) is **Java 24**, 
and the latest **Long Term Support (LTS)** version is **Java 21**. All my personal applications run **Java 21**, and all 
professional applications I work on run **Java 17** or **Java 21**.

Here's what I think of other Java versions:

## Java 22 and 23

Are no longer supported, upgrade to 24 ASAP.

## Java 18, 19 or 20

Are no longer supported, upgrade to 21 ASAP.

## Java 12, 13, 14, 15 and 16

These are no longer supported. Upgrade to 17 or 21 ASAP

## Java 11

This version was released in 2018. Unless you're paying for extended support you should be migrated, and most major frameworks are compatible with Java 17 anyway. It's a
fairly painless migration, so get to it. Hell, even Android is compatible with 17 these days.

## Java 8

This version was released in 2014, so almost a decade old at this point, and older than the Rust language. There are still some people who refer
to this as a "cool new technology", and I find it very hard not to laugh. While some JVM implementations
still offer support for this version, you're really lagging behind, and you're going to get in trouble as more and more frameworks
move on to newer versions and no longer offer security fixes for Java 8-based projects.

The migration to 11 can be a bit of a hassle due to some of the changes introduced in versions 9, 10 and 11 with regards to type inference
and generics, but after that it should be smooth sailing.

## Java 7 and below

![Gif of Homer Simpsons retreating back into a bush](https://media.tenor.com/HdCNuyIa2c4AAAAC/the-simpsons-homer-simpson.gif)

