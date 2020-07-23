---
layout: post
title:  "Arquillan for Java EE testing"
date:   2020-07-23 17:15:00 +0200
---

At my day job, our application is built on top of Java EE. It is a fairly old monolithic application
 (project started in 2006), though we dedicate considerable time and resources to keep most of our
 tech stack current.
 
One area we focus a lot of our effort on is automated testing. We use Selenium-type tests for a lot of
UI-based testing, and have a pretty large suite of unit tests. While these tests cover a vast amount
of logic, our developers were still looking for a way of writing tests for logic more complex than a
 simple unit test, but not quite as involved as a UI-based test. All attempts so far either involved
 abstracting away a lot of details through interfaces, or involved heavy use of mocking frameworks such
 as Mockito (which had the side-effect of making the tests brittle).
 
What we needed was a way to run tests against parts of a running application, such as a REST endpoint
or part of the service layer. The solution we chose was [Arquillian](http://arquillian.org/).
 
<!--more-->

Arquillian, at its core, is a testing framework that allows JUnit tests to be run on Java EE application
 servers. There are [connectors for most major Java EE servers](https://docs.jboss.org/author/display/ARQ/Container%20adapters.html),
 and you can of course write your own if you have a special use case.
 
This article will give a brief look into using Arquillian. It is not meant as a complete tutorial, so
I may skip a few steps or omit some details here and there.

## Getting started

After you [add the relevant dependencies to your project](http://arquillian.org/guides/getting_started/), writing
an Arquillian test is pretty straightforward:

```java

@RunWith(Arquillian.class)
public class MyFirstArquillianTest {
    @Test
    public void shouldInvokeService() {
        Assert.fail("Not yet implemented");
    }
}

```

In this state the test will not yet run, as there are a few more missing ingredients. First, you need to
create a Deployment: this describes your application, and will cause an in-memory version of it to be constructed on the fly.

Every single Arquillian tutorial I've ever seen tells you that this Deployment should be created by
a static method in your test class annotated `@Deployment`. While this will certainly work, it is not
practical in a codebase of any size. Instead, you want to create a specialized class for this and
annotate it with `@ArquillianSuiteDeployment`:

```java
@ArquillianSuiteDeployment
public class MyFirstArquillianDeployment {
    @Deployment
    public static WebArchive createMyWar() {
        return ShrinkWrap.create(WebArchive.class);
    }
}
```
This class will then automatically be detected by Arquillian for any test on the classpath, saving you
a lot of copy/paste work. The above code only creates an empty EAR file however, so to create a useful
archive we need to do a bit more work. Fortunately, Arquillian (or, ShrinkWrap, to be exact) can also
integrate with Maven. Below is a more or less complete example of a deployment that creates a testable WAR.

```java

@ArquillianSuiteDeployment
public class MyFirstArquillianDeployment {
    @Deployment
    public static WebArchive createMyWar() {
        Path pathToPomFile = /* INSERT LOGIC TO DETERMINE POM FILE PATH HERE */;

        WebArchive archive = ShrinkWrap.create(WebArchive.class);
        // Add Maven dependencies. Compile and runtime
        archive.addAsLibraries(Maven.configureResolver()
                .workOffline()
                .loadPomFromFile(pathToPomFile.toFile()).importCompileAndRuntimeDependencies()
                                                        .resolve()
                                                        .withTransitivity()
                                                        .asResolvedArtifact());
        // Add all regular source classes from the given package
        archive.addPackage(SomeClassInRootPackage.class.getPackage());
        // Add all files from target/classes
        archive.addAsResource(pathToPomFile.resolve("target").resolve("classes").toFile(), "");
        return archive;
    }
}

```

With this simple example you can create a pretty good copy of your WAR file and deploy it to your
 application server. Depending on the exact testing scenario, you may need to add additional (test) dependencies
 and test classes.
 
In addition to the code you specify here, Arquillian will run one or more extensions that add additional
classes (mainly determined by what extensions are on your classpath).

Getting the Deployment right is probably the hardest part about using Arquillian, but once you've got that
figured out, you can use Arquillian to directly access services in your application:

```java

@RunWith(Arquillian.class)
public class MyFirstArquillianTest {
    @Inject
    private MyCDIService service;

    @Test
    public void shouldInvokeService() {
        assertThat(service.invoke(), notNullValue());
    }
}

```

## Under the hood

To many of my co-workers, Arquillian was considered a big magic black box that defied understanding, and
I understand this stance. Arquillian can be pretty complicated even in simple cases, and with our 14-year-old
monolith the code to build our Deployment is pretty involved (though much of the difficulty is abstracted away in
one of our proprietary libraries).

When trying to reason about Arquillian tests and identify problems with them, it is important to
understand the key components.

### The Deployment

The Deployment is the package that is sent to the container&mdash;it contains everything that is needed
to run your application, and includes test classes (and dependencies). All test code is executed **on the application server**,
and as such anything your tests needs should be in the deployment.

### The Container

The Container is the piece of the framework that talks with your application server. There are many
 different variations. Some Containers launch the application server for you, while others require
 a running instance to already be configured elsewhere.
 
If your application server is located elsewhere, then any breakpoints you have set will be ignored 
(see previous point about test code being deployed to the application server) unless you manage to
attach the debugger to the remote application server.

To configure the container, a file called `arquillian.xml` is used. The contents depend on the type 
of container you use.

### The Arquillian Servlet

Because the tests execute **on** the application server, but JUnit runs on your local machine, they
need a way to communicate. Arquillian achieves this by installing a servlet in your application that
responds to commands.

While this works out of the box in most cases, I've seen some interesting issues with this
approach:

- The security framework ([Shiro](https://shiro.apache.org/)) denying requests to the servlet, causing
tests to fail.
- A hardcoded URL in `arquillian.xml` which caused JUnit to try to connect to the wrong machine

## In closing

Arquillian is a complex project, that fills the gap in testing capabilities we experienced. There are
dozens of extensions, offering solutions for many use cases.

Getting it to work is fairly involved, but once you do, it makes writing integration tests for Java EE
applications much easier.
