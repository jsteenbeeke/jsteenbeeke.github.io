---
layout: post
title:  JDK17 upgrades
author: "Jeroen Steenbeeke"
date:   2021-09-23 15:55:00 +0200
---
A little over a week ago version 17 of Java was released. Most of my applications were still running Java 11,
so the list of new features was pretty impressive, and included:

- Switch expressions (since JDK 14)
- Text blocks (since JDK 15)
- Records (since JDK 16)
- Pattern matching instanceof (since JDK 16)
- Sealed classes

Normally I'm quite conservative when it comes to upgrading to new version. I've stuck with 11 despite there being newer version,
mainly due to the short support cycles of the releases between 11 and 17. But 17, like 11,
is a long term support version. So feeling lucky, I went ahead and upgraded all my personal
projects.

For most of these projects, this was a very smooth process. For some of them, however, there were obstacles.
I will highlight them in this post.
<!--more-->
## Update libaries

Class files compiled by JDK17 have version 61, whereas files compiled by JDK11 have version 55. There
are many libraries, specifically those that use byte code manipulation or proxies, that are sensitive
to this. To fix this, I needed to upgrade ASM and Spring (which has a bundled version of cglib). Similar libraries
will probably require upgrades as well.

## Docker images

Most of my Docker images had OpenJDK 11 as a base image, either directly or indirectly. One of the
first things I did after I started upgrading was open [a PR for Jetty](https://github.com/eclipse/jetty.docker/pull/66) to replace JDK16 by JDK17, as 
the former will be unsupported a week from now. This PR has since been merged, but the updated images
have not yet been published. I'm running a private Docker repository for my applications, so I simply
made my own image and used that as a base. I can move back to an official image as soon as they are
released.

I use Jenkins to build my projects, so I also had to update the Maven images used. Maven, fortunately, already
had a JDK17 release.

## Code generation

I have code generators for a dozen or so different things. Getting these to work required a few tweaks.

### Annotation processors

To avoid warnings, make sure your `@SupportedSourceVersion` annotation contains value `SourceVersion.RELEASE_17`. This will avoid warnings,
and is a great test to check if your build tools are up-to-date (earlier JDK versions will throw an error).

### Maven Plugins

Custom maven plugins will require an update of the `maven-plugin-plugin` to work, otherwise they
won't recognize class file version 61. Version `3.6.1` of the plugin will work just fine.

## Spring

This was probably the trickiest one to get right. Most of my applications are based on Spring, and depending
on how you configure proxying you may run into issues. It is common for Spring applications to depend
on abstractions, where you inject by interface, and have the actual logic in an implementation of said interface.
This technique is quite common, and is one of the principles in [SOLID](https://en.wikipedia.org/wiki/SOLID) (the D,
to be exact).
While there are quite a few scenarios where such an approach makes sense (components that can have multiple
implementations, connectors/ports in hexagonal architectures), not all of my hobby projects follow this convention,
and some try to inject implementations directly.

To get this to work in Spring, I add the following to my configuration:

```java
@ComponentScan(
		basePackages = {"nl.jeroensteenbeeke.my.package"},
		scopedProxy = ScopedProxyMode.TARGET_CLASS)
```

If I do include interfaces the value of `scopedProxy` would have been `ScopedProxyMode.INTERFACES`. These projects
had 0 issues, of course.

The projects that injected concrete classes ran into some issues.

The first issue I ran into was in my test suite. My test classes generally launch an in-memory version of the
application to run tests against, and after each class the application is stopped (the next test start a fresh one).

This worked just fine with `ScopedProxyMode.INTERFACES`. For `ScopedProxyMode.TARGET_CLASS` I got a bunch
of LinkageErrors.

Naturally I tried Googling this issue, but I ran into the old "[only one other person on the internet ever had this problem before](https://xkcd.com/979/)" conundrum. Contrary
to the XKCD comic, there was an answer: "I solved this by switching to `ScopedProxyMode.INTERFACES` and just adding interfaces for everything".

Yeah... no.

Turns out that what was happening is that my JVM was being reused, and cglib was running into classes
it had generated with the previous iteration of the application. The solution was pretty simple, all I
had to do was to instruct the `maven-surefire-plugin` not to reuse forks:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <reuseForks>false</reuseForks>
    </configuration>
</plugin>
```

The second issue was harder to solve, and it involved a bug in [cglib](https://github.com/cglib/cglib/issues/191). Whenever
I would visit a page or tried to use a component that injected a concrete class, the error from this issue would pop up. 
Despite not explicitly using JPMS, it seems that the JVM prohibits access to certain APIs by default, unless
the library in question explicitly defines it wants access using the `opens` keyword.

I worked around this by adding the `--add-opens java.base/java.lang=ALL-UNNAMED` to my `maven-surefire-plugin` (the `argLine` parameter),
as well as to my JVM settings for docker-compose and Kubernetes (respectively). In the long term it would be better
for cglib to make this step unnecessary.

## In closing

Migrating all my hobby apps to JDK17 was relatively painless, the only tricky bit being the reflection
and linkage errors stated above. I'm eager to start working with all these new features.
