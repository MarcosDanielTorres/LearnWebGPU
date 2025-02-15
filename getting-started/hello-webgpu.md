Hello WebGPU
============

```{lit-setup}
:tangle-root: 005 - Hello WebGPU
:parent: 001 - Opening a window
:fetch-files: ../data/webgpu-distribution.zip
```

*Resulting code:* [`step005`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step005)

For your C++ code, WebGPU is nothing more than a single header file listing all the available procedures and data structures: [`webgpu.h`](https://github.com/webgpu-native/webgpu-headers/blob/main/webgpu.h). When building the program though, your compiler must know in the end (at the final *linking* step) where to find the actual implementation of these functions.

```{figure} /images/rhi-vs-opengl.png
:align: center
A *Render Hardware Interface* (RHI) like WebGPU is not directly provided by the drivers: we need to link to a library that implements the API on top of the low-level one that the system supports.
```

Installing WebGPU
-----------------

There exists mostly two implementations of the WebGPU native header:

 - [wgpu-native](https://github.com/gfx-rs/wgpu-native), exposing a native interface to the [`wgpu`](https://github.com/gfx-rs/wgpu) Rust library developed for Firefox.
 - Google's [Dawn](https://dawn.googlesource.com/dawn), developed for Chrome.

```{figure} /images/different-backend.png
:align: center
There are (at least) two implementations of WebGPU, developed for the two main web engines.
```

These two implementations still have **some discrepancies**, but they will disappear as the WebGPU specification gets stable. I try to write this guide such that it **works for both** of them.

To make the integration of either of these in a CMake project as easy as with GLFW and without too many extra dependencies, I share a [WebGPU-distribution](https://github.com/eliemichel/WebGPU-distribution) repository that lets you chose one of the following options:

### Option A: The lightness of wgpu-native

Since `wgpu-native` is written in rust, we cannot easily build it from scratch so the distribution includes pre-compiled libraries:

 - [wgpu-native for Linux](../data/wgpu-native-for-linux.zip)
 - [wgpu-native for Windows](../data/wgpu-native-for-windows.zip)
 - [wgpu-native for MacOS](../data/wgpu-native-for-macos.zip)
 - [wgpu-native for any plateform](https://github.com/eliemichel/WebGPU-distribution/archive/refs/heads/wgpu.zip) (a bit heavier as it's a merge of all the above basically)

```{important}
**WIP:** Use the "any plateform" link rather than the platform-specific ones, I haven't automated their generation yet so they are usually behind the main one.
```

```{note}
The pre-compiled binaries are provided by the `wgpu-native` project itself so you can likely trust them. The only thing my distribution adds is a `CMakeLists.txt` that makes it easy to integrate.
```

**Pros**
 - This is the most lightweight to build with.

**Cons**
 - You do not build from source.
 - `wgpu-native` does not give as informative debug information as Dawn.

### Option B: The comfort of Dawn

Dawn gives much better error messages, and since it is written in C++ we can build it from source and thus inspect more deeply the stack trace in case of crash:

 - [Dawn for any plateform](https://github.com/eliemichel/WebGPU-distribution/archive/refs/heads/dawn.zip)

```{important}
**WIP:** The first steps of this guide were more intensively tested with `wgpu-native`. Some later parts were tested with both and I tried to includes notes about the differences but for now I recommend using `wgpu-native` at least at the beginning.
```

```{note}
The Dawn-based distribution I provide here fetches the source code of Dawn from its original repository, but builds it without the need for extra dependencies (I include a script that replaces them basically) and by pre-setting some options to avoid building parts that we do not use.
```

**Pros**

 - Dawn is much more comfortable to develop with, because it gives more detailed error messages
 - It is in general ahead of `wgpu-native` regarding the progress of implementation (but `wgpu-native` will catch up eventually).

**Cons**
 - Although I reduced the need for extra dependencies, you still need to [install Python](https://www.python.org/)
 - The distribution fetches Dawn's source code and its dependencies so the first time you build you need an **Internet connection**.
 - The initial build takes significantly longer, and occupies more disk space overall.

### Integration

Whichever distribution you choose, the integration is the same:

 1. Download the zip of your choice.
 2. Unzip it at the root of the project, there should be a `webgpu/` directory containing a `CMakeLists.txt` file and some other (.dll or .so).
 3. Add `add_subdirectory(webgpu)` in your `CMakeLists.txt`.

```{lit} CMake, Dependency subdirectories (append)
# Include webgpu directory, to define the 'webgpu' target
add_subdirectory(webgpu)
```

 4. Add the `webgpu` target as a dependency of our app, after GLFW in the `target_link_libraries` command.

```{lit} CMake, Link libraries (replace)
# Add the 'webgpu' target as a dependency of our App
target_link_libraries(App PRIVATE glfw webgpu)
```

One additional step when using pre-compiled binaries: call the function `target_copy_webgpu_binaries(App)` at the end of `CMakeLists.txt`, this makes sure that the .dll/.so file that your binary depends on at runtime is copied next to it. Whenever you distribute your application, make sure to also distribute this dynamic library file as well.

```{lit} CMake, Link libraries (append)
# The application's binary must find wgpu.dll or libwgpu.so at runtime,
# so we automatically copy it (it's called WGPU_RUNTIME_LIB in general)
# next to the binary.
target_copy_webgpu_binaries(App)
```

```{note}
In the case of Dawn, there is no precompiled binaries to copy but I define the `target_copy_webgpu_binaries` function anyway (it does nothing) so that you can really use the same CMakeLists with both distributions.
```

### Option C: The flexibility of both

Bonus option that I use in the accompanying code that enables to decide on one distribution or the other upon calling `cmake`:

```
cmake -B build -DWEBGPU_BACKEND=WGPU
# or
cmake -B build -DWEBGPU_BACKEND=DAWN
```

This is given by the `main` branch of my distribution repository:

 - [WebGPU any distribution](https://github.com/eliemichel/WebGPU-distribution/archive/refs/heads/main.zip)

**Pros**
 - You can have two `build` at the same time, one that uses Dawn and one that uses `wgpu-native`

**Cons**
 - This is a "meta-distribution" that fetches the one you want at configuration time (i.e., when calling `cmake` the first time) so you need an **Internet connection** at that time.

And of course depending on your choice the pros and cons of *Option A* and *Option B* apply.

Testing the installation
------------------------

To test the setup, we call the `wgpuCreateInstance` function at the beginning of our main function. Like many WebGPU functions meant to **create** an entity, it takes as argument a **descriptor**, which we can use to specify options regarding how to set up this object.

And once again we meet a WebGPU idiom in the `WGPUInstanceDescriptor` structure. Its first field is a pointer called `nextInChain`. This is a generic way for the API to enable custom extensions to be added in the future, or to return multiple entries of data. In a lot of cases, we set it to `nullptr`.

```{lit} C++, file: main.cpp
#include <webgpu/webgpu.h>
#include <iostream>

int main (int, char**) {
	// We create the equivalent of the navigator.gpu if this were web code

    // 1. We create a descriptor
	WGPUInstanceDescriptor desc = {};
    desc.nextInChain = nullptr;

    // 2. We create the instance using this descriptor
	WGPUInstance instance = wgpuCreateInstance(&desc);

    // 3. We can check whether there is actually an instance created
    if (!instance) {
        std::cerr << "Could not initialize WebGPU!" << std::endl;
        return 1;
    }

    // 4. Display the object (WGPUInstance is a simple pointer, it may be
    // copied around without worrying about its size).
	std::cout << "WGPU instance: " << instance << std::endl;
}
```

This should build correctly and display something like `WGPU instance: 000001C0D2637720` at startup.

Differences between implementations
-----------------------------------

In order to handle the slight differences between implementations, the distributions I provide also define the following preprocessor variables:

```C++
// If using Dawn
#define WEBGPU_BACKEND_DAWN

// If using wgpu-native
#define WEBGPU_BACKEND_WGPU

// If using emscripten
#define WEBGPU_BACKEND_EMSCRIPTEN
```

The case of emscripten only occurs when trying to compile our code as a WebAssembly module, which is covered in the [Building for the Web](../appendices/building-for-the-web.md) appendix.

*Resulting code:* [`step005`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step005)
