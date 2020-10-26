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
4.  Set the output files to your `${DERIVED_SOURCES_DIR}/your.metal` file.
5.  Now create a compile step.  I reverse-engineered this by creating a dummy xcode project and seeing what it emitted.
    ```sh
		if [ $MTL_ENABLE_DEBUG_INFO = "INCLUDE_SOURCE" ]; then
		MOREARGS="-gline-tables-only -MO"
		else
		MOREARGS=""
		fi

		metal -c -target air64-apple-ios14.0 $MOREARGS -MO -I${MTL_HEADER_SEARCH_PATHS} -F${HEADER_SEARCH_PATHS} -isysroot "${SDKROOT}" -ffast-math  -o "${TARGET_TEMP_DIR}/Metal/your.air"  -MMD  "${DERIVED_SOURCES_DIR}/your.metal"
    ```
    You may have to massage this a bit for your situation.  For example, on macOS I use `-target air64-apple-macos10.15`.

See [blitcurveMetal.xcodeproj](https://github.com/drewcrawford/blitcurve/tree/master/blitcurveMetal.xcodeproj) for an example.

You may be wondering, why the `${DERIVED_SOURCES_DIR}`?  It's because in environments where the sourcetree isn't preserved, like CI, relying on its preservation may break incremental builds.

You may also be wondering, can't we leverage the Xcode build system a bit more?  The short answer is [no](http://www.openradar.me/10488973).

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

# Metal errors and where to find them

## MTLCaptureError Code=1

`Error Domain=MTLCaptureError Code=1 "Capturing is not supported.`

Add `MetalCaptureEnabled=1` to Info.plist.

Apple [documents](https://developer.apple.com/documentation/metal/frame_capture_debugging_tools/enabling_frame_capture?language=objc) that this happens automatically, but I'm aware of some cases where it doesn't.

*FB7870713 – works as designed*

## MTLCaptureError Code=3 "Capture Destination ‘Developer Tools’ is not supported."

Generally caused by trying to capture in an unusual environment, like in unit tests.  The workaround is to set `captureDescriptor.destination = .gpuTraceDocument`  rather than the default `.developerTools`

* FB8095102

## CompilerError Code=2 "Compiler encountered an internal error"

This is generally a crash in the metal compiler (the one that happens at runtime).  In my experience this is usually related to some problem in control flow analysis, so simplifying control flow may help.  It may also be specific to a particular GPU driver (see the crash in `MTLCompilerService` that occurred around the same time for more clues.)

* FB7863589
* FB8276262

## CompilerError Code=1 "Compiler encountered an internal error"

Similar to above, may be a variant for macOS

* FB8779951

## Compilation failed with XPC_ERROR_CONNECTION_INTERRUPTED on 3 try

Similar to Code=2 "Compiler encountered an internal error".  May be related to Shader Validator.

* FB8469191
* FB8824826

## Compute function exceeds available temporary registers

May have too many local variables.  Interestingly, shader validator appears to work around this issue, for reasons unknown.

* FB8811182

## DYPShaderDebuggerErrorDomain:4 "Failed to create data source" DYPShaderDebuggerDataErrorDomain:0 "Thread data not found"

Usually caused by a GPU abort or IOAF that took place during the capture.

* FB7872191

## DYPShaderDebuggerErrorDomain:1 "Failed instrument library" DYMetalLibraryToolsServiceCompilerErrorDomain:1 "Task Failed: Building library: tools_instrumented_iphoneos_xxxxxxxx.xx.thin.metallib."

Variant error has "NSCocoaErrorDomain:260" in the subhead.

This seems to be caused by some problem locating sourcecode or debugging info.  This can occur if you are linking in a static library made with `metal-libtool`.  Interestingly, this can occur even if the symbols in the library are not referenced by the shader being debugged.  Maybe referenced by a different shader in the capture, or just in the library itself?

The workaround is to compile in a local version of the library into your metal target.

* FB8095102
* FB8826801
* FB8826081
* FB8545458
* FB8276481

## Replayer terminated unexpectedly with error code 5.  timed out (5)

Appears to be a hung system process.  For me the issue is usually in macOS rather than xcode or on device, so rebooting clears it.   Collecting more data on this one.

## IOAF codes

IOAF codes are generally something going on in the GPU driver.  Apple apparently deliberately does not document what they are, and I get the feeling that for the non-Apple GPUs they are really generated by third-party code.  [According to apple](https://developer.apple.com/forums/thread/86789),

> If you're seeing one, it's probably an Apple kernel or driver bug and you should file a bug

### IOAF code 2

Generic error, but usually an invalid device load/store

### IOAF code 3

thread may have entered an infinite loop

### IOAF code 4

Generic error, but usually an invalid device load/store

### IOAF code 5

Generic error, but usually an invalid device load/store

### IOAF code 262

Render target had no texture

* FB8749612
