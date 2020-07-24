---
layout: post
title:  "Determine test effectiveness with Mutation Testing"
author: "Jeroen Steenbeeke"
date:   2018-01-15 17:04:00 +0100
---
Has one of the following ever happened to you?

 * You looked at your project's bug database, and found that half the reported bugs were regressions?
   * Despite the fact that your project has 80% test coverage?
   * And every bug solved in the past three years had a unit test specifically to reproduce said bug? 
 * Or perhaps you were refactoring some crucial logic, and broke it
   * And you ran the unit tests to help debug it
   * And found they were all green
   
At this point, once the panic has passed, you should probably start an investigation:

 * What other features that we thought were properly tested are actually broken?
 * Which of our tests _are_ effective?
 * Which tests do we need to fix?
 
Manually analyzing your tests to answer these questions will take a considerable amount of time, and depending
on the size of your codebase is somewhere between extremely tedious and impossible.

Fortunately, there are tools to help you with this, and a technique known as Mutation Testing.
<!--more-->
## Mutation Testing

Mutation testing is a analysis technique that modifies the code executed by your automated tests in small ways,
to produce deviations called _mutants_, that are often minor errors that coders typically make:

 * Inverted operators (`<=` instead of `>`)
 * Inverted expressions (`return true` instead of `return false`)
 * Modified operators (replace `+` by `*` or `/`)
 * Modified variable values (replacing values with `null` or vice-versa)
 * Replacing instances with `null`
 * Removed method calls
 
 This technique was originally proposed in 1971 by Richard J. Lipton, though the technique became a lot more feasible to 
 automate with more recent advances in computing power.
 
 Mutation testing tools automatically create a number of mutants, and expose them to your test set. For a mutant
 to be _killed_, the following three conditions should be met:
 
  1. A test must **reach** the mutated statement
  2. The test's input data should **infect** the program: causing a different program state between the mutant and the
  original program.
  3. The incorrect program state should **propagate** to the program's output and be checked by the test
  
This is called the RIP model.

## How does this help?

Mutants are, in essence, broken versions of your software. They're small errors that could have been made by
a colleague new to the project and not entirely familiar with all boundary conditions and edge cases.

If your tests cannot catch these small errors, then they are not effective. By automatically creating dozens
or hundreds of these small errors, you can get a pretty good idea which parts of your code have effective tests.

## And how can you do this, exactly?

That depends on your programming language of choice, but I've found a dozen frameworks that allow you to run
mutation tests. The only one I've used, however, is [PIT](http://pitest.org/), which easily integrates with Maven:

```xml
<plugin>
    <groupId>org.pitest</groupId>
    <artifactId>pitest-maven</artifactId>
    <version>1.3.1</version>
 </plugin>
```

You can then invoke it with the following Maven command:

```
mvn org.pitest:pitest-maven:mutationCoverage
```

This will, depending on the size of your code base, take somewhere between **a long time** and **an exceptionally long time**,
so if you need an answer right away you should probably rent a time machine and start it an hour ago.

Either way, once PIT is done processing, you can find a bunch of reports in your project's target folder.

At the top of this report, you'll find a summary of packages where the observed line coverage is compared to
the mutation coverage. The package links yield a list of detected classes, and their respective coverages. Clicking on
a class will yield details about the mutations that were introduced and what their result was, giving
you very detailed scenarios to expand your tests with.

## What are the drawbacks?

PIT is not able to test **all** Java code, and if you're using another JVM language it might not yield any
usable results (Scala, for instance, does not work well with PIT, though there is some support for Kotlin and Groovy).

In addition, in some cases that involve a lot of reflection, PIT might crash and not be usable at all.

## Conclusion

If you **can** use a mutation testing tool with your project, it is an invaluable tool to check the quality of your unit
 tests. Where test coverage is a good quantitative metric, mutation coverage is the perfect qualitative metric of your automated tests,
 and you should check it on a regular basis.
