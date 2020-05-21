---
layout: post
title:  "Background sync in Android with Workmanager"
date:   2020-05-21 22:11:00 +0100
---

A while ago [I posted](/2019/04/16/event-sourcing-app-synchronization.html) about a problem I was experiencing with one
of my applications. I was using event sourcing to create a more lightweight synchronization mechanism between a backend
and my Android app, the goal being that instead of reading the entire state on each update, the app would only request whatever
had changed since its last synchronization. While good in theory, there was a flaw in my implementation of the synchronization
logic (the part which reports the app's changes back to the server), causing each change to be registered hundreds of times.

The effect was that the time needed to sync the app was growing longer as time went by, and if I ever managed to wipe the
app's local data, it would spend an hour catching up to all history (which, considering the app is task list, is a bit long).

<!--more-->

The bug turned out to be rather simple: the synchronization job was not running once, but many times, simultaneously. I had
already solved the worst of the problem (changes registered multiple times) by adding a simple 
[semaphore](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/Semaphore.html) to the 
synchronization handler, but the sync job was now running dozens of times in sequence. Not exactly a problem, but still
sloppy (and with Android 8+ displaying a notification whenever an app does something in the foreground, it was very 
distracting to see my app's icon pop up a dozen times in quick succession).

So how did I fix this? By making the following change:

```kotlin

val request = PeriodicWorkRequest.Builder(
            SynchronizationWorker::class.java,
            Duration.ofMinutes(60L)).addTag(SynchronizationWorker::class.java.name
        ).build()

// OLD
WorkManager.getInstance().enqueue(request)

// NEW
WorkManager.getInstance().enqueueUniquePeriodicWork(
            "MyAppNameBackgroundSync", 
            ExistingPeriodicWorkPolicy.KEEP, 
            request)

```

And that is all! Whenever my device boots, or the application starts, this code is run, and if 
[WorkManager](https://developer.android.com/topic/libraries/architecture/workmanager) finds an existing
job for this request, that one is kept instead of creating a new one.