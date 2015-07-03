---
layout: page
title: Style guides considered harmful
---

It seems like every time I log onto Twitter, somebody new has published their style guide.  In Swift we have avoided that particular plague for the moment, but I sense that the language is now stabilizing to the point that any day somebody will foist their style guide upon the world with a triumphant `git push`.  So I'd like to take this opportunity to go on record about why you're wrong and your style guide sucks.

# Why are you even writing this

Style guides have exactly one legitimate purpose: to document and avoid language pitfalls.  ObjC is a scary language, and at one time I maintained a list of [things you should never do](http://sealedabstract.com/code/nsnotificationcenter-with-blocks-considered-harmful/).  I am told in languages like C++ it is about twelve orders of magnitude worse, so I would believe you need a style guide in that language.

However the best way to avoid language pitfalls is to work in a language that doesn't have them.  Thankfully, Swift solves roughly 95% of the serious ObjC pitfalls, and I am not convinced the ones that remain are things any sane person would ordinarily try to do.  (I am not a sane person.)

# The Whitespace Wars

However the vast majority of styleguides are preoccupied with the eternal battle between tabs and spaces, between braces-on-the-line and braces-on-the-next-line.  And let's be clear: this is bikeshedding in the purest form.  If you are even *considering* writing a document that micromanages how often people press the return key then either you need to step away from the computer and get an OCD diagnosis or else the programmers on your team are actually cats.

Because normal, healthy people do not need to be regularly told to use CamelCaseForClassNames.  "I almost checked in this variable name, but then I re-read the style guide" - said no developer ever.  

The fact is, things like "line length", "brace placement" and so on are jobs for computers, not for humans.  If you are annoyed by braces, an acceptable answer is to configure your text editor, write a linter, or configure the build server to automatically sort it out.  The acceptable answer is *not* to write a lot of arbitrary rules, distribute them, and then convince fallible humans by fiat to follow them *and* keep doing all the hard stuff that they normally do with no impact to their job responsibilities.

# I don't work for dictators

Ruling by decree is basically the worst way to lead.  Telling everybody else how to do things is what you do when you're not convinced they will come up with the "right" answers themselves.  It's basically a vote of no confidence.

If you don't have any confidence with the people on your team, somebody needs to get fired.  When faced with untrustworthy, suspicious developers who can't even be bothered to name variables in a sensible way, the normal thing to do is fire them, or get transferred, or quit, or any arbitrary step that keeps you out of working proximity with them.

Normal people do not write documents in order to lie in wait and later claim the moral high ground in an argument about whitespace placement.  That is the sign of organizational dysfunction I can't even imagine. "Aha, I caught you!  Violating a document which is really just my opinion about how to press the shift key, but somehow legimitized because it's in a PDF you can't edit and didn't bother to read until now!" is not exactly the *most* convincing way to tell someone to change their code.

# Style talks, not style laws

If you identify a pitfall, or some other area of concern, the right thing to do is to give a talk, or presentation, or paper, or circular, or whatever the equivalent is in your organization.  Where you say "Hey, I found this.  This is my recommendation.  This is how it's gone so far."

The insight here is that style is not absolute; but it is in fact a matter of opinion, and opinions are to be defended in the marketplace of ideas, not decided by organizational fiat.  The second insight is that languages and codebases and the people working on them change, but style guides generally don't.  And so styleguiding is favoring the few that are here over the many who will have to follow your rules afterwards, but weren't there to argue with you at the time.

# So that's why I won't work for you

I generally avoid places that have style guides that they take seriously.  First and foremost it's a sign of people who don't trust each other with the absolute basics of writing software.  Secondly it's a sign that you have small problems when the crisis of brace placement looms so large that at some point you had an entire meeting about it.

I realize that apparently large organizations (many of which regularly try to recruit me) take a different view, but quite frankly, I don't want to work there and I don't care.