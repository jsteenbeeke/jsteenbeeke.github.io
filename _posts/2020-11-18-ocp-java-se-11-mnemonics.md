---
layout: post
title:  "OCP Java SE 11 Mnemonics"
author: "Jeroen Steenbeeke"
date:   2020-11-18 12:35:00 +0200
---

Two weeks ago I passed the [Oracle 1Z0-819 exam](https://education.oracle.com/java-se-11-developer/pexam_1Z0-819), which means I can now call myself an **Oracle Certified Professional: Java SE 11 Developer**.
I've already seen a massive increase in the number of recruiters contacting me, so if you're considering doing this exam,
there might be spam afterwards.

[![OCP Java SE 11 badge](/assets/img/oracle-certified-professional-java-se-11-developer.png)](https://www.youracclaim.com/badges/6fb72fa6-bfbd-4209-bb6a-efa8de316ef4/public_url)

This exam was interesting in a number of ways. First of all, I never originally intended to take this exam,
as originally there were two exams for this certification:

- 1Z0-815 (comparable to the old Java OCA exam)
- 1Z0-816 (comparable to the old Java OCP exam)

In fact, I had already passed the first exam back in February, and had only started studying for
the second one when Oracle contacted me to say the old exams were being invalidated, and I had a week left
to do the second one.

Not feeling too confident about what was supposed to be six weeks of study into four days, I cancelled my 1Z0-816 exam
and opted to take the 1Z0-819 exam on the same day I originally planned the old exam. The problem: this exam covers all topics of both old exams, so I still had to
study twice as much material in six weeks. And I succeeded.

One of the key ingredients to this success was the use of mnemonics. Short words or phrases that helped
me memorize key concepts. I had around 25 of these, though at the exam I think I only used two or three of them.

In this post I will share some of the more memorable ones.

<!--more-->

## Operator precedence: Paul Elstak

![Paul Elstak](/assets/img/paul-elstak.gif)

With a considerable part of my childhood taking place in the 90s, I grew up with a lot
of dance music. Paul Elstak is a well-known artist in the Netherlands, and as such was
one of the easiest mnemonics for me to memorize, even though it's only a shorthand for the
actual abbreviation you need to get operator precedence right: PPOMASR ELSTA (which I memorized as: `Paul Elstak -> Pommes Elstak -> PPOMASR ELSTA`).

From this you can distill the actual operator precedence in Java:

- **P**ostfix increments and decrements 
- **P**refix increments and decrements
- **O**ther unary operators
- **M**ultiplication, division and modulo
- **A**ddition and subtraction
- **S**hift operators
- **R**elational operators
- **L**ogical operators
- **S**hort-circuit operators
- **T**ernary operators
- **A**ssignments

## Switch statements: Biscuits

Switch statements only allow a limited number of variable types, which I remembered as "biscuits". Like "Paul Elstak",
this is a representation of the actual abbreviation: BISCES.

- **B**yte and `byte`
- **I**nteger and `int`
- **S**hort and `short`
- **C**haracter and `char`
- **E**num values
- **S**trings

## Stream.reduce parameters: Ia ia Cthulhu Fhtagn

![Cthulhu](/assets/img/cthulhu.jpg)

This one requires some familiarity with the Cthulhu mythos to make any sense, and like the others
this is a representation that needs some transformation to make sense: `A`, `IA`, `IAC`.

- **A**: Accumulator
- **IA**: Identity, Accumulator
- **IAC**: Identity, Accumulator, Combiner

This translates to the following method signatures:

```java
// A
Optional<T> reduce(BinaryOperator<T> accumulator);

// IA
T reduce(T identity, BinaryOperator<T> accumulator);

// IAC
<U> U reduce(U identity, BiFunction<U, ? super T, U> accumulator, BinaryOperator<U> combiner);
```

## Checked exceptions: Fish pen

Checked exceptions are a bit of a pitfall for me when it comes to these exams. Whenever you
encounter a method that throws a checked exception you need to either rethrow it, or catch it.
Having worked for Java for nearly 20 years I am so used to the compiler reminding me that it took
some training to actively be on the lookout for this.

For the exam, there are five checked exceptions that you need to be aware of, which I memorized
with the mnemonic "fish pen" (or rather: `FISPN`):

- **F**ileNotFoundException
- **I**Oexception
- **S**QLException
- **P**arseException
- **N**otSerializableException

I also had a mnemonic for unchecked exceptions ("I am a unicorn" - `IAMAUNICAN`), but I figured it was easier just to 
see if an exception was checked instead, since the list is much shorter.

## In closing

Using these mnemonics made it much easier for me to study for this exam. Whenever I had to go back
to a subject that I had studied a while back, just looking at the mnemonics was enough to remember
most of the material needed for that subject.

I would encourage anyone studying for this exam to consider using mnemonics. If they work for you,
you have an excellent tool to help memorization.