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

## cross-compiling C code to metal

The best way I know of to do this is to concatenate `.c` files into a `.metal` file, and compile that.

1.  Create a 'run script' phase
2.  ```sh
	COUNTER=0
	rm -f "${SCRIPT_OUTPUT_FILE_0}"
	while [ $COUNTER -lt ${SCRIPT_INPUT_FILE_COUNT} ]; do
	    tmp="SCRIPT_INPUT_FILE_$COUNTER"
	    FILE=${!tmp}
	    cat "$FILE" >> "${SCRIPT_OUTPUT_FILE_0}"
	    let COUNTER=COUNTER+1
	done
	``` 
3.  Set the input files to all your `.c` input files.  You need to keep them up to date, alterantively you can use a file list
4.  Set the output files to your `.metal` file.  After this file is built, drag into xcode and attach it to the right project

See [blitcurveMetal.xcodeproj](https://github.com/drewcrawford/blitcurve/tree/master/blitcurveMetal.xcodeproj) for an example.


## distributing a library for use inside a metal shader

The best way I know of to do this is to build a `.a` file, or a script/xcodeproj to build one, and distribute that.

1.  Create a "run script" phase
2.  ```sh
	metal-libtool -static "${TARGET_TEMP_DIR}/Metal/mylib.air"  -o "${BUILT_PRODUCTS_DIR}/libmylib.a"
```
3.  Set the "input files" to `${TARGET_TEMP_DIR}/Metal/mylib.air`
4.  Set the "output files" to `${BUILT_PRODUCTS_DIR}/libmylib.a`

### If you are also doing this as part of a Swift package,

...nobody can use your xcodeproj directly, because the xcodeproj is readonly and you will get errors about "The file "project.pbxproj" could not be unlocked" (FB8095945)

Unfortunately, you have to instruct users to

1.  Install your project in a fixed location relative to their project, such as with a git submodule or copying the library into their repository.
3.  Set the "Metal Compiler - BuildOptions" "Header Search Paths" to include fixedlocation/Sources/target/include.  This will let Metal sources find the header files from the Swift package.  *Note this is distinct from "Header Search Paths" underneath "Search Paths" category.*
2.  Set the `MTLLINKER_FLAGS` to `-L ${BUILT_PRODUCTS_DIR} -l mylib`.  This assumes that your library has the `libmylib.a` naming scheme.  Also, this build setting is undocumented.

See [blitcurve](https://github.com/drewcrawford/blitcurve) for a complete example.

* *FB7776777*
* *FB7744335*
* *FB8095945*


#### Why a fixed directory?

[Prevoius versions of this FAQ](https://github.com/drewcrawford/drewcrawford.github.io/blob/93765a25e3dafd29f7b75e8114d7b7e066a38c5b/metal_faq/index.md#shared_precomps_dir--wtf) did some reverse-engineering of how xcode resolves swiftpm packages in order to calculate where xcode keeps swiftpm projects in terms of various other environment variables.

However, the environment variables can take on a wide range of values (such as when archiving, building as part of a playground, etc.) and I don't think there is one stable enough to use for this purpose.

* `FB8102669` - environment variable for swift packages
* `FB8095382` - environment variable for metal projects

# My "Metal Library" target does not have a product

The best solution I'm aware of on this problem is to create a custom phase for copying the `.metallib` into the target manually.

```bash
COUNTER=0
while [ $COUNTER -lt ${SCRIPT_INPUT_FILE_COUNT} ]; do
    tmp="SCRIPT_INPUT_FILE_$COUNTER"
    INPUT=${!tmp}
    
    tmp="SCRIPT_OUTPUT_FILE_$COUNTER"
    OUTPUT=${!tmp}

    cp ${INPUT} ${OUTPUT}

    let COUNTER=COUNTER+1
done

```

* input files: `${BUILT_PRODUCTS_DIR}/my.metallib`
* output files: `${BUILT_PRODUCTS_DIR}/${EXECUTABLE_FOLDER_PATH}/my.metallib`

*FB8276893*

# MTLCaptureError Code=1

`Error Domain=MTLCaptureError Code=1 "Capturing is not supported.`

Add `MetalCaptureEnabled=1` to Info.plist.

Apple [documents](https://developer.apple.com/documentation/metal/frame_capture_debugging_tools/enabling_frame_capture?language=objc) that this happens automatically, but I'm aware of some cases where it doesn't.

*FB7870713 – works as designed*



