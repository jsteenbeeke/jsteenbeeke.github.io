---
layout: page
title: Why are you still using ancient Java versions?
permalink: /recruiters/fuck-old-java-versions
---

As of writing this bit, the [most recent Java version](https://en.wikipedia.org/wiki/Java_version_history) is **Java 19**, 
and the latest **Long Term Support (LTS)** version is **Java 17**. All my personal applications, as well as all applications
I work with professionally, run **Java 17**.

Here's what I think of other Java versions:

## Java 18

Is no longer supported, upgrade to 19 ASAP.

## Java 12, 13, 14, 15 and 16

These are no longer supported. Upgrade to 17 or 19 ASAP

## Java 11

This version was released in 2018. While still under support, you're missing out on great features, and most major frameworks are compatible with Java 17 anyway. It's a
fairly painless migration, so get to it.

## Java 8

This version was released in 2014, so almost a decade old at this point. There are still some people who refer
to this as a "cool new technology", and I find it very hard not to laugh in their faces. While some JVM implementations
still offer support for this version, you're really lagging behind, and you're going to get in trouble as more and more frameworks
move on to newer versions and no longer offer security fixes for Java 8-based projects.

The migration to 11 can be a bit of a hassle due to some of the changes introduced in versions 9, 10 and 11 with regards to type inference
and generics, but after that it should be smooth sailing.

## Java 7 and below

Are you fucking nuts? If you still have active software running a version that was released over a decade ago and no longer receives security updates,
then you really need to get your shit together.

