---
layout: page
title: Why not semver
---

Sometimes people ask me, why don't I like [semver](http://semver.org)?

The answer is that it does not solve any problem.  Worse, it tricks the gullible into thinking it solves an important problem, and then it doesn't do that.  So people rely on it, and that's dangerous.

# Why you can't have machine-readible version strings

The way semver is used in practice is that you have a package manager (like node, cocoapods, carthage, etc.) and you tell your package manager, for some package, to "pin to this major version" or "pin to this minor version".  The idea is that you can run one upgrade command and because in semver each version component has defined semantics, the package manager can do something intelligent like "don't upgrade breaking changes" or "security updates only".

The first problem with this theory is that it assumes package maintainers follow the specification.  Now I, deliberately, do not follow the semver specification because of the reasons in this essay.  (And I'm in good company--[plenty](https://github.com/jashkenas/backbone/issues/2888#issuecomment-29076249) of [smart](https://github.com/jashkenas/underscore/issues/1684) [people](http://www.jongleberry.com/semver-has-failed-us.html) [ignore](https://gist.github.com/jashkenas/cbd2b088e20279ae2c8e) it.)  So right out of the gate, you're going to have to deal with contrarian library authors.

But maybe you think you can avoid depending on my software, or that in the long arc of history I will come around.  But setting aside myself and the rest of the merry band of conscientious objectors, library authors may violate semver inadvertently.  A [search on GitHub](https://github.com/search?o=desc&p=1&q=semver+broke&ref=searchresults&s=comments&state=closed&type=Issues&utf8=✓) reveals some 450 issues, 68 of which are still open.  So if you are relying on semver to make sure you're software doesn't work... you're wrong.

There's a saying in software engineering that you can't test the quality *into* your software.  Testing just tells you that something is wrong; fixing it is a different department.

Similarly, you can't version the compatibility *into* your software.  The way these versions generally get applied is the developer who implements the feature, makes a guess about whether there are breaking changes.  But "doesn't break *my* programs" should not fill you with confidence.  If your project has QA engineers and a release process then I might believe your version number, but if it's just some developers shooting from the hip like most repos on GitHub then quite frankly you have no idea if your change is a breaking change or not.

If you want to find out if updating breaks anything, you should do the following:

1.  Have a good test suite
2.  Run the test suite after the upgrade and see if anything broke

Or use some automated version of those two steps like [next-update](https://github.com/bahmutov/next-update).  But using a version number instead of actually doing proper QA like a professional is bad, and you should feel bad.  It's cargo cult software engineering.

# Why you don't want machine-readible version strings

A conservative library author--which is generally the kind you want--is in a pickle.  Does this change break anything?  Well it works on *my* machine, but... let's be safe, and bump the major version anyway.

Now obviously, this creates ugly version numbers like 387.0.0.  But I have heard people (wrongly) opine that 387.0.0 is a fine version number, so I will leave that sidequest be.

The real problem is that you *thought* you were signing up for security updates by pinning to 2.0.x but in reality we go straight to 3.x and then 4.x because who knows if this security patch breaks some application somewhere or not.  Testing is hard, git push is easy.  And now you're stuck on 2.0.x and somebody roots your machine and you can't complain to me because my library followed semver!

The reality is that you probably need security updates for your dependencies, period.  But if you don't have a formal, contractual relationship with the authors of the library, they might not give you any.  Or they might give you some, but bundle them with major version improvements.  Because they don't work for you.

Really I'm saying that if you're using semver as your signal to determine when to upgrade, you're Wrong On Security™.

# Why you're not entitled to machine-readible version strings

The "we broke semver" issues on GitHub are littered with comments [like](https://github.com/jashkenas/underscore/issues/1684#issuecomment-46124904):

> We're talking about a choice that breaks production code. It's not cool to play a sloppy game with other people's time and money.

[and](https://github.com/jashkenas/underscore/issues/1805#issuecomment-53938293):

> Wait, so when people came to your project with real concerns about the health of the node.js ecosystem and the expenditure of thousands of man-hours fixing things and hunting bugs, your response was to write a script to sync something over to a version called "v170.0.0"? (which also does nothing to address the issues they identified with an existing install base)

So newsflash.  When you rely on a library, it does not magically make the maintainers work for you.  You can't just wander into somebody else's issue tracker and start demanding things.  Well, you *can*.  I probably wouldn't react as well as some of those folks did.

What professionals do, when they need only the security updates, is they buy a subscription.  That is how Red Hat etc. became billion-dollar businesses, because when a CIO is asked "How do we know your product is secure?" the answer "Some guy named Dmitry who we have never met and have no formal relationship with will probably release an update which we may or may not install because of our semver policy" is not an acceptable answer.  "We pay Red Hat $10,000/yr to make sure we get security patches" is.

By the way, I am happy to sell you a subscription for security updates for any software I maintain.  I will backport the security patch to whatever version you are using.  So if you are running my code in production, contact me so you are prepared to answer that question in your meeting.

However I won't backport everything to all versions whether they are commercially relevant or not.  Sorry.  I have new software to work on instead.

# How I version

I do not use semver, which apparently means I practice ["sentimental versioning"](http://sentimentalversioning.org).  (I still cannot figure out if that is a joke or if [dominictarr](https://github.com/dominictarr) really thought it was a good idea to design an entire webpage merely to insult 5 software projects for not following his versioning religion.)

* Major - strap yourself in, this is a big upgrade
* Minor - minor or no breakages, budget half an hour or so
* Patch - The cost/benefit ratio is such that no sane person should hold back updating.  This number is for things like security updates and simple bugfixes.  Note that even this number may very well break some code somewhere, because security fixes are more important than compatibility.

In other words, versions for humans.  Major means a major change.  Minor means a minor change.  Patch means the least possible change that you still want to install.

If this doesn't work for you, I am happy to sell you a subscription that has exactly what updates you want ("just security fixes", or "just nonbreaking changes from version X.")  But I don't follow semver, and you shouldn't either.

