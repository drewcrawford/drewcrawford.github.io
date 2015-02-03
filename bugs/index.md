---
layout: page
title: The trouble with bugfixes
---


People are sometimes puzzled why "simple" bugs are so hard and expensive to fix.

# A simple bug

Supose we have a program that divides two numbers.  So we give it some input like

```
10, 5
```

and it outputs

```
10 / 5 == 2
```

So far so simple.

As you may remember from math class, we cannot divide by zero.  You can divide by practically all numbers, except that one particularly.

Computers cannot divide by zero either.  What generally happens if you write a program that divides by zero is you get a crash or an error.

The difficulty is that we cannot test all the numbers someone might conceivably want our program to divide.  There are an infinite amount of numbers, and we only have a finite amount of time.  So we must pick a much smaller set of numbers to test with.  And if we forget to try zero, we will never notice that our division program will crash in that case.  We can try millions of numbers and test for a very long time, but if zero *in particular* isn't included, we will miss that bug.

# The trouble with simple solutions

Now you may say, the solution is straightforward.  We simply check if the number if zero, before we divide.  If it is not-zero, we let it through, and if it is zero, we tell the user that we cannot divide by zero.

The first problem is that it assumes that zero is a number that somebody, at some point, thought might be a problem.  That's not necessarily true in general.  For example, [Intel once had a bug in how their processors did division](https://en.wikipedia.org/wiki/Pentium_FDIV_bug), that involved numbers that nobody thought of to check.  The bug ended up costing them $475 million.

The second problem is that it assumes that the program is set up in a way that the number of places where somebody attempts to do a divsion is low and they can be easily found.  For example, Microsoft Windows is hundreds of millions of lines of code, so if you are Microsoft, there is just no chance you can even find all the dividing going on.  More subtly, if you're just some guy who writes a  program that uses Windows at any point, you run the risk that something you are doing causes Windows to internally divide by zero deep down in the haystack somewhere.  And customers will not blame Microsoft for that, since everything was fine until they installed *your* program.

# Structural engineering

You can see at once that there is a question of architecture here.  If the programmer had the foresight to put all the division *in one place*--that is to *isolate* it, then it is simple enough to fix.  You just go to that place.

If they did not, however, it's a mess.  You will never find all the places that things get divided.  You will be looking forever.

What this should suggest is **you can really only fix bugs that were, in some sense, expected**.  This is a problem because most bugs are *not* expected.  That makes them hard to fix.

# Just paint the door red

A common approach by "manager types" is to try and come up with something simple to fix the problem.  When these "simple" fixes fail to contemplate the consequences of the entire system, they can fail or introduce unintended consequences.

For example, passenger-side airbags improved the safety factor of car travel for adult passengers.  But child fatalities actually increased, since children were knocked quite strongly as the bags were deployed.  As parents became aware of this problem, they started shifting children to the backseat.  As a result the number of children forgotten in hot vehicles increased.  So what started out as a simple solution of reusing existing technology on the other side of the car became a very complex situation with long-term follow-on effects.

The only way out of this problem is to develop some general idea of the full breadth and scope of the entire program before being confident that introducing a change somewhere will not cause problems somewhere else.  A specification is a good way to achieve this.

