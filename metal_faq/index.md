---
layout: page
title: Metal FAQ
---

# Drew's Metal FAQ

This page documents various odd Metal behaviors that I have not found anywhere else.

## Using Deployment Target of "Fall 2020" removes non-C++ features

After raising my deployment target to iOS 14 (macOS 11, etc...) I got numerous new compile errors.  These errors do not occur with the same compiler with a deployment target of iOS 13 (macOS 10.15, etc...)

It appears this is because various C99/C11 features were 'removed' (I gather they were never really supported) in Metal, which is based on C++, not on C.  In particular, this affects `restrict`, which is not a valid C++ keyword.

I am advised this works as intended.  For `restrict` anyway, the compiler keyword `__restrict` still compiles, although I don't know if it has any effect.

*FB7831220*

## `assert`

As of iOS 14, `assert` is defined as

```c
#define assert(condition) ((void) 0)
```

For this reason it has no effect.

If you want `assert`-like behavior in Metal, you can use

```c
#define assert(X) if (__builtin_expect(!(X),0)) {float device *f = 0; *f = 0;}
```

along with the "metal shader validation" diagnostic in [Debug GPU-side errors in Metal](https://developer.apple.com/wwdc20/10616).  It will not trip without this diagnostic.

For a production-ready solution, [stdmetal](https://github.com/drewcrawford/stdmetal) ships with `SM_ASSERT` and `SM_PRECONDITION` cross-platform macros.

*FB7731230*

## replayer terminated unexpectedly with error code 5

This is usually caused by performing a metal capture programmatically during application launch.  The workaround is to perform the capture "later", such as with `.asyncAfter`.

*FB7741457*


