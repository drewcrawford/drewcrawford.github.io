---
layout: page
title: commit messages
---

Generally speaking, a commit message is useful to jog your memory about why some particular hunk of code is changed. Like a code comment, but one that's always present for every line of code. Messages shouldn't be judged on length, they can in fact be very short. But it's unlikely that you can get a good one with just 2 or 3 words.

As a project gets larger, some of the decisions you make get re-decided. Those new decisions may be better, or they may be worse, than your original design. But unless you have a record of why you made those decisions, you are never really sure if you are smarter now or if you were smarter then.

In fact, if you work in `blame` view, which I often do when I'm hunting bugs, you instantly get a lot of context whenever you're reading code.  For example, if the commit message in the gutter says "this isn't threadsafe" while you're debugging then you'll keep your eye out for strange threadery while you step through the program.  That information is worth its weight in gold.

Here's an example of a commit message I wrote recently:

```
Working around a (bug?) in UICollectionView

It turns out that UICollectionView likes to raise unsatisfiable 
constraints for some reason in this case.
Those contraints contain a (surprise) autoresizing mask constraint.
That resizing mask constraint is now manually removed in code.  But 
for some reason, doing that causes all trailing constraints to be 
interpreted as negative leading constraints (WTF) so your content 
appears to the left of the container (e.g. invisible).
I'm converting the layout to be all-leading, even though that's not 
a very natural flow for this view.  Such are prototypes.  
```

If you approach my example commit message from the perspective of someone debating about changing the code months or years later, they have a lot of information to work with:

* Why the change was made.  It involves a `UICollectionView` bug.
* A description of the bug.  An unexpected constraint appears in the view.
* Ideas that were tried.  Someone could, with this information, continue to investigate the bug to find a different solution, without duplicating effort.
* The ultimate solution, reversing the constraints, which is a bit strange, and might all by itself trigger questions that would lead to reading this commit message
* The fact that time pressure played a major role in this decision, suggesting that future self probably could generate a better solution than past self did, with more time to think about it.

If you're regularly making a lot of commits, don't worry about targeting this length, because many short commits in aggregate can tell the same story that this message does in one go.  Another reason to avoid this is if you have some external explanation, like a bug tracker, specification, or code-level documentation, that explains the codebase or the commit in detail.  But if you're committing once a day, or less, and there is no other resource to understand the change, you should be targeting this level of detail.  This is the kind of detail that helps you make sense of things you did long ago.

## Shooting for the stars

Probably the canonical example of good commit messages is the Linux kernel.  Of course, git was designed by Linus specifically to be Linux’s source control, so literally commit messages (and everything else in git) was designed around their workflow.

[Here's](http://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit/?id=e11da5b4fdf01d71d73c21cb92b00595b917d7fd) probably the greatest commit message of all time.  Go ahead and read it.  I'll wait.

Obviously that is complete overkill for everyday stuff, but it fixes a bug that eluded a large group of very smart people for a long time, so in that case it was completely appropriate, and that gives you an idea of how high of a note you can hit.  You’ve got tables, charts, benchmarks, and a bibliography.  You’ll notice that although it’s incredibly technical, it’s written in such a way that people with only incidental familiarity with Linux can immediately understand the general outline of the problem and solution.

The Linux kernel has a style guide that covers, among other things, writing good commit messages.  That is largely covered in [section 2](http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/Documentation/SubmittingPatches?id=HEAD).

Again I think working at that level is a bit of an overkill for single-developer workflow, but the principles are all really good and can be scaled down. Particularly the principles I would distill from that guide are things like:

* Have a brief overview
* Followed by a detailed description
* Providing benchmark results if the patch claims to improve any performance metric
* Explaining tradeoffs, downsides, or drawbacks of the change.  This is really important when fixing bugs later, because the very things you write might explain the bug.  For example if my commit message includes “this approach isn’t threadsafe” then when I have blame view open looking for some bug, I’m going to see that text and briefly consider if there’s a threading issue nearby.
* One problem per patch
* Imperative mood
* Referencing external stuff (bug trackers, specs, external notes) by ID or page number.  I had a design decision recently that I revisited 2 years later that was as fresh in my mind as the day I made it, because I was able to pull up the entire process of how the decision was made originally, spanning weeks and meetings with dozens of people.
* Referring to previous related commits as appropriate

Another factor working in Linux’s favor is that there are many more patches than reviewers.  So the commit message functions as “marketing material” to attract a reviewer’s attention in a sea of other patches.  This motivates good commit messages in a way that is hard to replicate in a single-developer workflow.

However, there are some process improvements that can approach this—for example, a policy of doing a code review that can reject patches from landing on the main tree can provide some benefit here on a process basis, but it generally requires a few developers to implement.

You can also browse the [Linux tree on GitHub](https://github.com/torvalds/linux/commits/master) to see more of an idea of their “ordinary” commit messages.

Keep in mind that their “merge commits” usually just summarize the actual patch commits, so try and judge them by the commits that actually make changes rather than the ones that merge unrelated things together.  But a lot of their very mundane patch commits are remarkably well-written.

## A workable strategy

I recommend recording information about what is being worked on contemporaneously, and then turning that information into a commit message at the end.  For more information on that strategy, check out my FAQ on [uninterruptible programming](http://127.0.0.1:4000/uninterruptible_programming_supply/).