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

Note that, this sort of control flow can confuse the compiler and lead to issues debugging metal shaders.  So you may want to pull this out if you have trouble attaching a debugger.

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

Note that doing this may case `DYPShaderDebuggerErrorDomain:1 "Failed instrument library"`, see error discussion below.

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

# stdlib

## math functions

I am aware of various "relaxed" behavior of metal math functions, relative to the familiar c or c++ stdlib behavior.  For example,

> In [Metal] fast math, pow(x,y) is defined to be exp2(y* log2(x))...For x in the domain [0.5, 2], the maximum absolute error is <= 2-22; Otherwise, if x > 0 the maximum error is <= 2 ulp; Or, the results are undefined.  Based on hardware, this relaxed definition may cause issues for negative values. To get well defined for negative, one should use metal::precise:pow. 

(this limitation is undocumented.)

Should you encounter such behaviors, you may have better results using a `precise` function or disabling `fno-fast-math`.  However, see the discussion for `simd_length` and `fno-fast-math` below.

FB8904929


## `simd_length`

I am aware of cases where the switching between `simd_length` (`metal::length`), `simd_fast_length` (`metal::fast::length`) and `simd_precise_length` (`metal::precise::length`) seems not to work.

FB8882598


# `fno-fast-math`

In addition to the behavior in `simd_length`, I am aware of cases where Metal seems to ignore disabling `ffast-math` and continues illegal fp optimization.

FB8880572

# Metal errors and where to find them

## MTLCaptureError Code=1

`Error Domain=MTLCaptureError Code=1 "Capturing is not supported.`

Add `MetalCaptureEnabled=1` to Info.plist.

Apple [documents](https://developer.apple.com/documentation/metal/frame_capture_debugging_tools/enabling_frame_capture?language=objc) that this happens automatically, but I'm aware of some cases where it doesn't.

*FB7870713 – works as designed*

## MTLCaptureError Code=3 "Capture Destination ‘Developer Tools’ is not supported."

Generally caused by trying to capture in an unusual environment, like in unit tests.  The workaround is to set `captureDescriptor.destination = .gpuTraceDocument`  rather than the default `.developerTools`

* FB8095102

Note that captures taken in this way may not be replayed against the simulator.

* FB8904809

## Runtime compiler errors

Sometimes functions fail to compile (e.g., at runtime, when you create a PSO).  In your application, this usually presents as some `CompilerError`.  Separately, a crash report for `MTLCompilerService`

The two `CompilerErrors` I'm aware of are

* CompilerError Code=2 "Compiler encountered an internal error" (usually iOS or Simulator)
* CompilerError Code=1 "Compiler encountered an internal error" (usually macOS).

For more information, dig into the appropriate MTLCompilerService crash report.  The backtrace may identify a particular affected GPU, as many of these issues are GPU-specific.

### Other runtime compile application-level errors

#### Compute function exceeds available temporary registers

May have too many local variables.  Interestingly, shader validator appears to work around this issue, for reasons unknown.

* FB8811182

### libAMDIL902.dylib: llvm::ILAnnotateResourceIntrinsics::createResourceIntrinsic(llvm::Use&, llvm::RsrcUsageTy) + 222

Related to control flow analysis on AMD.  simplifying control flow may help.

* FB7863589


### libigc.dylib: llvm::MemoryDependenceResults::removeInstruction(llvm::Instruction*) + 1390

May be related to vertex or fragment functions with very bad typesignatures.  One case I'm aware of involves swapping a vertex and fragment function.

* FB8276262

### libigc.dylib computeKnownBitsFromOperator(llvm::Operator const*, llvm::KnownBits&, unsigned int, (anonymous namespace)::Query const&) + 383

Possible duplicate of `libigc.dylib: llvm::MemoryDependenceResults::removeInstruction(llvm::Instruction*) + 1390`

* FB8276262

### libLLVM.dylib llvm::SelectionDAG::LegalizeTypes() + 54

May be related to use of null pointers on Intel.

* FB8962634
* FB8973899

###  libLLVM.dylib llvm::Instruction::getFunction+ 6782680 () const + 0

May be related to Shader Validator.

* FB8469191


## Problems attaching the debugger

For various reasons, the Metal Debugger often fails to attach.  Below are some errors and their possible causes.

### DYPShaderDebuggerErrorDomain:4 "Failed to create data source" DYPShaderDebuggerDataErrorDomain:0 "Thread data not found"

Can be caused by a GPU abort or IOAF that took place during the capture.  Workaround is to not do that.

Has other causes.  I believe that generlaly, the debugger is struggling with memory layout. To fix this, simplify the memory layout, at least while you're trying to debug:

* simplify array indexes.  Ideally, index by constants.  Avoid dependent array indexes (indexing by some value to look up another index).  Avoid indexing by opaque values such as the return value of a non-inline function.
* flatten nested loops
* reorder structs

* FB7872191
* FB8969664
* FB8968619
* FB8924507
* FB8827782
* FB8889577

### DYPShaderDebuggerErrorDomain:1 "Failed instrument library" DYMetalLibraryToolsServiceCompilerErrorDomain:1 "Task Failed: Building library: tools_instrumented_iphoneos_xxxxxxxx.xx.thin.metallib."

Variant error has "NSCocoaErrorDomain:260" in the subhead.

It appears that Xcode 12.3 / iOS 14.3 has added a diagnostic identifying a particular function at issue:

> LLVM ERROR: Undefined symbol: _Z17SadTromboneOpaquev

This seems to be caused by some problem locating sourcecode or debugging info.  One reason this can occur is if you are linking in a static library made with `metal-libtool`.  Interestingly, this can occur even if the symbols in the library are not referenced by the shader being debugged.  Maybe referenced by a different shader in the capture, or just in the library itself?

The workaround is to compile in a local version of the library into your metal target.

* FB8095102
* FB8826801
* FB8826081
* FB8545458
* FB8276481

An additional cause of this issue is calling a `__attribute__((pure))` function that is relocated or optimized out by the compiler.  To work around this, declare the function without pure, or avoid calling functions that will be optimized out.

* FB8889367

### Replayer terminated unexpectedly with error code 5.  timed out (5)

Appears to be a hung system process.  For me the issue is usually in macOS rather than xcode or on device, so rebooting clears it.   Collecting more data on this one.

An additional root cause is performing a metal capture programmatically during application launch.  The workaround is to perform the capture "later", such as with `.asyncAfter`.

*FB7741457 - works as intended*


## Xcode behavior

### "build succeeded" beachball

I am aware of some cases where Xcode will beachball after "build succeeded".  This usually takes progressively longer and longer each time, and may be associated with taking GPU captures.  The workaround is to restart xcode.

* FB8287558

### Xcode crash - EXC_BAD_ACCESS (SIGSEGV)

Xcode can crash when starting a metal debugging session.  This usually happens with

```
KERN_INVALID_ADDRESS at 0x0000000000000020
EXC_CORPSE_NOTIFY
Dispatch queue: GPUShaderDebuggerSession.queue (QOS: USER_INITIATED)

Thread 36 Crashed:: Dispatch queue: GPUShaderDebuggerSession.queue (QOS: USER_INITIATED)
0   com.apple.GPUToolsPlatformSupport-iOS	0x000000015013e6dc 0x14ff6c000 + 1910492
1   com.apple.GPUToolsPlatformSupport-iOS	0x000000015013d884 0x14ff6c000 + 1906820
2   com.apple.GPUToolsPlatformSupport-iOS	0x000000015013e771 0x14ff6c000 + 1910641
3   com.apple.GPUToolsPlatformSupport-iOS	0x000000015013eb99 0x14ff6c000 + 1911705
4   com.apple.GPUToolsPlatformSupport-iOS	0x000000015013ef01 0x14ff6c000 + 1912577
5   com.apple.dt.gpu.GPUDebugger  	0x000000012ac4571a -[GPUShaderDebuggerDataSource variablesForExecutionHistoryNode:] + 110
6   com.apple.dt.gpu.GPUDebugger  	0x000000012abdc17f __88-[GPUShaderDebuggerSession variableSnapshotsForExecutionHistoryNodes:completionHandler:]_block_invoke + 260
7   com.apple.Foundation          	0x00007fff343a9ac5 __NSBLOCKOPERATION_IS_CALLING_OUT_TO_A_BLOCK__ + 7

```

This is typically caused by some problem understanding shader parameters.  In particular, `void*` pointers can cause this.

### n/a

When debugging a Metal shader, sometimes xcode will say a particular variable holds the value `n/a`.  Usually the lldb pane has the real value of the variable.

* FB8959261

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
