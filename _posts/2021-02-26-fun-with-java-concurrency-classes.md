---
layout: post
title:  Fun with Java concurrency classes
author: "Jeroen Steenbeeke"
date:   2021-02-26 21:15:00 +0200
---
One of the topics on the [OCP Java SE 11 Programmer exam](https://tech.jeroensteenbeeke.nl/2020/11/18/ocp-java-se-11-mnemonics.html) is concurrency, and several 
concurrency-related classes from the `java.util.concurrent` package are featured on the exam.

Having worked with Java since 2002, I've encountered my share of multithreaded code, and used a variety
of solutions for getting threads to behave nicely, such as the various ways you can use Java's `synchronized` keyword as well
as the [Semaphore](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/Semaphore.html) class.

Thanks to the OCP exam, however, I've also become acquainted with a bunch of classes I hadn't used before,
such as [ReentrantLock](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/locks/ReentrantLock.html) and [CyclicBarrier](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/CyclicBarrier.html).

In this post I'm going to give a few examples of each.

<!--more-->

## ReentrantLock

A [ReentrantLock](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/locks/ReentrantLock.html) is
something best compared to a rest room: only the person inside the room can lock and unlock the door. The thread
that locks the `ReentrantLock` also has to unlock it, and while locked, no other thread can enter the code in question:

```java
ReentrantLock lock = new ReentrantLock();
Runnable r = () -> {
    lock.lock();
    System.out.print("1");
    System.out.print("2");
    System.out.print("3");
    System.out.println();
    lock.unlock();
};
// 100 lines of "123"
for (int i = 0; i < 100; i++) {
    new Thread(r).start();
}
```

The concept is similar to using a `synchronized` block with a shared object, but where the synchronized block is limited 
to part of single method body, a ReentrantLock can be used anywhere, and exceed the scope of a single method.

## Semaphore

A [Semaphore](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/Semaphore.html) is a mechanism
that is best explained with a space analogy. Imagine you're on a space station and need to go out into space for an EVA. There are 5
people on the station, but only 2 space suits. A Semaphore represents these two space suits: two people can go into space,
the rest have to wait until the others get back (or brave the vacuum unprotected, but that's generally considered a bad idea).

```java
Semaphore spacesuit = new Semaphore(2);
Runnable astronaut = () -> {
	spacesuit.acquireUninterruptibly(); 
	System.out.println("Moving out of the airlock");
	System.out.println("Flying around");
	System.out.println("Going back inside");
	spacesuit.release();
        };
		for (int i = 0; i < 5; i++) {
		new Thread(astronaut).start();
		}
        
```

One thing to keep in mind with Semaphores is that unlike ReentrantLock they are not tied to a specific thread. Any thread
can call `release()` and the number of permits will increase. In terms of the space analogy: imagine the astronaut flying to a nearby
spaceship, entering it, taking off the spacesuit, giving it to another astronaut who
was already in the spaceship, who then flies back to the station and gives it to one of the astronauts who was already at the station.

## CyclicBarrier

A [CyclicBarrier](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/CyclicBarrier.html) is a mechanism
for thread orchestration: it prevents thread from continuing past a certain point until all related threads have also reached this point.

In keeping with our earlier space analogy, imagine the process of going into space as follows:

* Astronaut dons spacesuit
* Astronaut opens airlock inner door
* Astronaut enters airlock
* Astronaut closes airlock inner door
* Airlock depressurizes
* Astronaut opens airlock outer door

This works just fine with a single astronaut, but if you have more than 1 astronaut, and they act
without regard for what the other is doing, then the first astronaut may open the outer door while a second
one has just opened the inner door, depressurizing the entire station.

A CyclicBarrier solves this problem:

```java
final int ASTRONAUT_COUNT = 5;

CyclicBarrier catchingUp = new CyclicBarrier(ASTRONAUT_COUNT);
Runnable astronaut = () -> {
    try {
        System.out.println("Donning spacesuit");
        catchingUp.await();
        System.out.println("Open airlock inner door");
        catchingUp.await();
        System.out.println("Enter airlock");
        catchingUp.await();
        System.out.println("Close airlock inner door");
        catchingUp.await();
        System.out.println("Depressurize airlock");
        catchingUp.await();
        System.out.println("Open airlock outer door");
    } catch (BrokenBarrierException | InterruptedException e) {
        System.out.println("Horrific space accident");
    }
};
for (int i = 0; i < ASTRONAUT_COUNT; i++) {
    new Thread(astronaut).start();
}
```

For each step in the Runnable all threads must reach the successive `await()` commands before the application continues,
preventing our astronauts from ever being in danger.

## In conclusion

Multithreading is a complex problem, and there are hundreds of books dealing with all sorts of issues surrounding concurrency.
These classes are just a small part of the puzzle, but they've helped me solve simple concurrency issues on numerous occasions.

In many cases, however, a lot of problems surrounding concurrency result from mutable state. In a future post I will probably
address the concept of immutability, and give examples on how to work with immutable classes to avoid many concurrency issues.
