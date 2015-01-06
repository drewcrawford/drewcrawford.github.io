---
layout: page
title: Swift optionals
---

In general, this is the list of types, from best (safest) to worst (not safest):

1.  Non-optional types (e.g. not `?` or `!`)
2. Optional types (`?`), when both the nil and non-nil case are both **handled** (e.g. code is written) and **tested** (e.g. somebody tried it)
3. Implicitly unwrapped optionals (`!`)
4. Optional types (`?`), when the nil and non-nil case are 'eyeballed' to 'probably work'.

Generally, always be thinking about how to move higher up the safety tree.  Improvements from 3 to 1 are possible in some cases, and case 4 should be eliminated in favor of one of the safer patterns.  Improvements from 3 to 2 are always possible, but are often too much work to be practical.