---
title: "LLVM to WASM"
date: 2018-02-10T13:17:46-05:00
draft: false
tags: ["WebAssembly", "LLVM"]
categories: ["WebAssembly"]
toc: true 
---

# Quick Overview of WebAssembly

Web assembly (WASM) is an assembly language that targets the JavaScript virtual machine.  It can be run in both modern web browsers as well as other environments such as `node` or `js`.  Although JavaScript might not be everyone’s favorite language its usage is omnipresent, and ton of time and effort has been put into making the underlying VM performant.  WASM opens the doors to this VM allowing other languages to use it as their target environment.  Simply put this will allow languages like C/C++/etc. to be compiled to WASM and executed within your web browser.  This also provides an opportunity for new performance improvements since we can now use battle tested Ahead of Time (AOT) compilers like LLVM and still make use of the standard VM Just in Time (JIT) optimizations.

# helloworld.c to WASM the Hard Way

At the time of writing this the WASM target architecture is an Experimental backend and not enabled in (most/all?) default distributions.  Below are the setups to build Clang/LLVM from master with the WASM backend target enabled.


You can see if your Clang/LLVM supports WASM by checking if `wasm32` is a registered target backend in the `LLVM static compiler` llc.  The output below is the registered targerts that ship with my system and you can see that `wasm32` is missing. Luckily building llvm/clang from source is not as daunting as it might seem. 

```bash
$ llc --version
LLVM (http://llvm.org/):
  LLVM version 5.0.0

  Optimized build.
  Default target: x86_64-unknown-linux-gnu
  Host CPU: ivybridge

  Registered Targets:
    aarch64    - AArch64 (little endian)
    aarch64_be - AArch64 (big endian)
    amdgcn     - AMD GCN GPUs
    arm        - ARM
    arm64      - ARM64 (little endian)
    armeb      - ARM (big endian)
    bpf        - BPF (host endian)
    bpfeb      - BPF (big endian)
    bpfel      - BPF (little endian)
    hexagon    - Hexagon
    lanai      - Lanai
    mips       - Mips
    mips64     - Mips64 [experimental]
    mips64el   - Mips64el [experimental]
    mipsel     - Mipsel
    msp430     - MSP430 [experimental]
    nvptx      - NVIDIA PTX 32-bit
    nvptx64    - NVIDIA PTX 64-bit
    ppc32      - PowerPC 32
    ppc64      - PowerPC 64
    ppc64le    - PowerPC 64 LE
    r600       - AMD GPUs HD2XXX-HD6XXX
    sparc      - Sparc
    sparcel    - Sparc LE
    sparcv9    - Sparc V9
    systemz    - SystemZ
    thumb      - Thumb
    thumbeb    - Thumb (big endian)
    x86        - 32-bit X86: Pentium-Pro and above
    x86-64     - 64-bit X86: EM64T and AMD64
    xcore      - XCore
```


## Building Clang/LLVM With WASM Backend

Below are the directions for building clang from source.  More detailed instructions and required dependencies can be found on llvm.org: <https://clang.llvm.org/get_started.html>.  The important step is to make sure that you enable the WebAssembly experimental target `LLVM_EXPERIMENTAL_TARGETS_TO_BUILD=WebAssembly`.  Other targets (such as `x86/x86-64`) can also be added if desired.

_note: The steps below use the current directory as the install location_ 

```bash
export DIR=`pwd`
mkdir $DIR/src
mkdir $DIR/bin

cd $DIR/src
svn co http://llvm.org/svn/llvm-project/llvm/trunk llvm

cd llvm/tools
svn co http://llvm.org/svn/llvm-project/cfe/trunk clang
svn co http://llvm.org/svn/llvm-project/lld/trunk lld
cd $DIR/src

mkdir build
cd build

# Set install target to $DIR/bin and enable the WebAssembly backend
cmake -DCMAKE_INSTALL_PREFIX=$DIR/bin \
-DLLVM_TARGETS_TO_BUILD= \
-DLLVM_EXPERIMENTAL_TARGETS_TO_BUILD=WebAssembly \
$DIR/src/llvm 

# Build with 2 * core count
make -j $(( `grep -c ^processor /proc/cpuinfo` * 2 ))

make install

# Add the bin directory to our path
export PATH=$DIR/bin:$PATH
```

Now if we run `LLVM static compiler` we should see WASM a registered backend target. 

```bash
$ llc --version
LLVM (http://llvm.org/):
  LLVM version 7.0.0svn
  DEBUG build with assertions.
  Default target: x86_64-unknown-linux-gnu
  Host CPU: ivybridge

  Registered Targets:
    wasm32 - WebAssembly 32-bit
    wasm64 - WebAssembly 64-bit
```


# Our helloworld.c


Below is our helloworld program:

```C
extern void out(const char *, int len);

int
main(int argc, char *argv[])
{
        out("Hello World", 11);
        return 0;
}
```

As you probably noticed this isn't exactly what you see in your typical helloworld.  Where is `out` defined?  Why aren't we using `printf` or `puts`?  The reason we aren't using standard `POSIX` functions or system calls is that they don't exist in the enviornment we are targeting. By default the VM does not provide a `libc` for us that defines all of the standard functions we know and love.  For now we will simply define an extern function named `out` that we will define late after going through the compilation steps and start interfacing with the browser.


## Building helloworld.c

Lets build our WASM binary. 

```bash
$ clang --target=wasm32 -Os helloworld.c -nostdlib -c -o out.wasm
```

* `--target=wasm32`: Use wasm32 as the target architecture
* `-Os`: Optimize for size
* `-c`: generate an object file
* `-o`: Write output to dest file


Great!  We now have a WebAssembly binary.  But how do we use this thing?  Well first let's take a peek inside:

```
$ xxd out.wasm
00000000: 0061 736d 0100 0000 018c 8080 8000 0260  .asm...........`
00000010: 027f 7f01 7f60 027f 7f00 02c4 8080 8000  .....`..........
00000020: 0303 656e 760f 5f5f 6c69 6e65 6172 5f6d  ..env.__linear_m
00000030: 656d 6f72 7902 0001 0365 6e76 195f 5f69  emory....env.__i
00000040: 6e64 6972 6563 745f 6675 6e63 7469 6f6e  ndirect_function
00000050: 5f74 6162 6c65 0170 0000 0365 6e76 036f  _table.p...env.o
00000060: 7574 0001 0382 8080 8000 0100 0686 8080  ut..............
00000070: 8000 017f 0041 000b 0791 8080 8000 0204  .....A..........
00000080: 6d61 696e 0001 062e 4c2e 7374 7203 000a  main....L.str...
00000090: 9480 8080 0001 1200 4180 8080 8000 410b  ........A.....A.
000000a0: 1080 8080 8000 4100 0b0b 9280 8080 0001  ......A.........
000000b0: 0041 000b 0c48 656c 6c6f 2057 6f72 6c64  .A...Hello World
000000c0: 0000 9480 8080 000a 7265 6c6f 632e 434f  ........reloc.CO
000000d0: 4445 0a02 0404 0000 000c 0000 bc80 8080  DE..............
000000e0: 0007 6c69 6e6b 696e 6702 8f80 8080 0002  ..linking.......
000000f0: 046d 6169 6e04 062e 4c2e 7374 7202 0381  .main...L.str...
00000100: 8080 8000 0c05 9280 8080 0001 0e2e 726f  ..............ro
00000110: 6461 7461 2e2e 4c2e 7374 7201 00         data..L.str..
```

Ok... that wasn't that useful, but do we see that there are some strings that might be useful.  Let's convert the binary to some human-readable form.  One of the more popular representations of this is known as `WAT` or the `WebAssembly Text Format`.  It's an interesting way to actually represent the binary since it uses [S-expressions]<https://en.wikipedia.org/wiki/S-expression>.  For this, we will need to use the `wasm2wat` tool that comes with `The WebAssembly Binary Toolkit` (WABT).  

## Building WABT

```bash
git clone --recursive https://github.com/WebAssembly/wabt
cd wabt
mkdir build
cd build
cmake ..
make -j $(( `grep -c ^processor /proc/cpuinfo` * 2 ))
export PATH=$PWD:$PATH
```

## Understanding Our binary - WASM to WAST

Below is the result of converting our binary to the `WAT` format. I have added some comments that briefly go over each instruction, but for a deeper explination the official documentation can be found here: <https://developer.mozilla.org/en-US/docs/WebAssembly/Understanding_the_text_format>

```
$ wasm2wat out.wasm
(module ;; Define our module
  (type (;0;) (func (param i32 i32) (result i32))) ;; declare our main funcition type
  (type (;1;) (func (param i32 i32))) ;; declare our out function type
  (import "env" "__linear_memory" (memory (;0;) 1)) ;; Memory Import
  (import "env" "__indirect_function_table" (table (;0;) 0 anyfunc)) ;; Table import 
  (import "env" "out" (func (;0;) (type 1))) ;; Our out function import from our extern function above.
  (func (;1;) (type 0) (param i32 i32) (result i32) ;; Our main function
    i32.const 0 ;; Push 0 to the stack which references our global string 
    i32.const 11 ;; Push 11 to the stack wich is the lenght of the buffer
    call 0 ;; Call function 0
    i32.const 0) ;; Return 0
  (global (;0;) i32 (i32.const 0)) ;;
  (export "main" (func 1)) ;; export our main function
  (export ".L.str" (global 0)) ;; export global
  (data (i32.const 0) "Hello World\00")) ;; export the helloworld data
```

### Imports

1. `env.__linear_memory` - This will reference a `WebAssembly.Memory` object that will be used to back our raw memory store/access. More information on this can be found [here](https://hacks.mozilla.org/2017/07/memory-in-webassembly-and-why-its-safer-than-you-think/)
2. `env.__indirect_function_table` - This will reference a `WebAssembly.Table` that can be used to store references.  We currently aren't making use of this.  The best resource I have found explaining this can be found [here](https://hacks.mozilla.org/2017/07/webassembly-table-imports-what-are-they/)
3. `env.out` - This is our out `print` function.  We will need to assign this to something in order to make it actually work, we can see how this will be covered below.

### Exports

1. `main` - Our main function
2. `.L.str` - Memory location of our "Hello World" string that is stored in the `global` section of our binary

## Loading WASM in the Browser


So we now have a WASM binary, but how do we use it? The below snippet shows how to put everything together.  


1. Fetch the WASM file to be loaded, in our case `out.wasm`
2. Grab the contents of the download as an `ArrayBuffer`
3. Compile the WebAssembly Binary
4. Setup the module imports
	1. `env.__linear_memory` Assign this to a `WebAssembly.Memory` model and initalize it to 256 pages.  This is an arbitary ammount and since our example doesn't allocate any memory this shouldn't be an issue. 
	2. `env.__indirect_function_table` is assignd to a  `WebAssembly.Table`.
5. Define and setup our `out` function.  


Below shows how we are mixing boundaries between our WASM binary and standard JavaScript functions. From the C/C++ side you can think of `out` as a symbol that doesn't get resolved until runtime, similar to a dynamically loaded library.  We can change the behavior of the `out` function without building our WASM binary. 

```html
<html>

<head>
  <script>
    if (!('WebAssembly' in window)) {
      var msg = 'WebAssembly not supported';
      alert(msg);
      console.error(msg);
    }

    function loadWebAssembly(filename, imports) {
      return fetch(filename)
        .then(response => response.arrayBuffer())
        .then(buffer => WebAssembly.compile(buffer))
        .then(module => {
          imports = imports || {};
          imports.env = imports.env || {};

          if (!imports.env.__linear_memory) {
            // Setup our Memory import, initializing it
            // to use 256 pages of memory.
            imports.env.__linear_memory = new WebAssembly.Memory({ initial: 256 });
          }

          if (!imports.env.__indirect_function_table) {
            // Setup our Table with an inital size of 0,
            // 'anyfunc' is currently the option here
            imports.env.__indirect_function_table = new WebAssembly.Table({ initial: 0, element: 'anyfunc' });
          }


          var consoleDiv = document.getElementById('console');
          imports.env.out = function consoleLogString(offset, length) {
            // Convert the bytes stored in our memory buffer
            // at position offset.
            var bytes = new Uint8Array(imports.env.__linear_memory.buffer, offset, length);

            // Convert our byte array to a utf8 string
            var string = new TextDecoder('utf8').decode(bytes);

            // Append the string to the DOM
            var content = document.createTextNode(string);
            consoleDiv.appendChild(content);
          }

          // Create a WebAssembly instance with our compiled
          // module and pass in our import object
          return new WebAssembly.Instance(module, imports);
        });
    }

    // Call our load function.
    loadWebAssembly('out.wasm').then(instance => {
      // Grab our exports and call our main function
      var exports = instance.exports;
      var main = exports.main;
      main();
    });
  </script>
</head>

<body>
  <div id="console"></div>
</body>

</html>
```

# Putting It All together

With our `out.wasm` and `helloworld.html` in the same directory we can start a webserver and visit our demo in a browser that supports `WebAssembly`.   You should see "Hello World" written on your screen! Browser support can be found here: <https://developer.mozilla.org/en-US/docs/WebAssembly>


Obviously, this is a lot of work just to print "Hello World" to the screen and is clearly not the best use case for WASM, but it does serve as a good starting point in understanding how all the pieces fit together. 


# helloworld.c to WASM the Easy Way

The example above required a lot of steps to get a simple example working.  Not having a `libc` available to use makes porting any code challenging.  Luckily there is already a project that fills this gap: [Emscripten](http://kripken.github.io/emscripten-site).  Emscripten is an entire toolchain/environments that comes with everything you need to get started, including a port of `libc` using [musl](https://www.musl-libc.org/).  It also comes with a few other library ports that make porting/building an app targeting WASM easier.

**Available Ports:**

```bash
$ emcc --show-ports 
Available ports:
    zlib (USE_ZLIB=1; zlib license)
    libpng (USE_LIBPNG=1; zlib license)
    SDL2 (USE_SDL=2; zlib license)
    SDL2_image (USE_SDL_IMAGE=2; zlib license)
    ogg (USE_OGG=1; zlib license)
    vorbis (USE_VORBIS=1; zlib license)
    bullet (USE_BULLET=1; zlib license)
    freetype (USE_FREETYPE=1; freetype license)
    SDL2_ttf (USE_SDL_TTF=2; zlib license)
    SDL2_net (zlib license)
    Binaryen (Apache 2.0 license)
    cocos2d
```

Emscripten is well documented so it's best to refer to their documenation for support.

# References


<https://hacks.mozilla.org/2018/01/shrinking-webassembly-and-javascript-code-sizes-in-emscripten/>

<https://developer.mozilla.org/en-US/docs/WebAssembly/Using_the_JavaScript_API>

<https://github.com/reklatsmasters/webassembly-examples/tree/master/%239-native-build>

<https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Table>

<https://hacks.mozilla.org/2018/01/shrinking-webassembly-and-javascript-code-sizes-in-emscripten/>

<https://github.com/WebAssembly/design/blob/master/TextFormat.md>

<https://github.com/WebAssembly/spec/tree/master/interpreter/>

<https://github.com/WebAssembly/design/blob/master/BinaryEncoding.md>

<https://llvm.org/docs/CommandGuide/llc.html>	
