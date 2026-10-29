+++
date = '2026-04-28T00:00:00Z'
title = 'C++ std module on macOS with Clang'
categories = ["C++"]
tags = ['C++', 'Clang', 'LLVM', 'module']
+++

If you, like me, are modernising your C++ knowledge (or learning it for the first time) and have discovered the C++ modules feature, you probably thought: *how cool - it’s like self-contained libraries/assemblies/packages/crates from my favourite language X, which has had them forever!* No more **#includes**, which have been the source of all sorts of problems.

But the first thing you’ll likely encounter is that this feature is still not fully supported across compilers. And even when it is supported, the library that should lead this transition - the standard library - may not yet be available as a ready-to-use module. This is currently the case with Clang: the standard library is not shipped as a precompiled module (as of the time of writing). Instead, it is provided as source code, typically located at a path such as ***/usr/share/libc++/v1/*** or ***/opt/homebrew/opt/llvm/share/libc++/v1***, depending on how Clang is installed (e.g. via Xcode or Homebrew LLVM).

So you will need to compile the std module yourself: 

```bash
clang++ -std=c++20 --precompile \
  /usr/share/libc++/v1/std.cppm \
  -o std.pcm \
  -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk \
  -Wno-reserved-module-identifier
```

A few notes:

* On newer macOS versions, Clang may fail to locate the correct SDK automatically. That’s why the **-isysroot** flag is explicitly provided. You can also use **/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk** for this parameter.

* The **-Wno-reserved-module-identifier** switch suppresses warnings about using the reserved module name std.

### Using the compiled module

The module file std.pcm is output to the current directory. You'll need to pass it to the compiler using **-fmodule-file=std=std.pcm** parameter when building your program which consumes the std module. Also remember to specify -std=c++20 parameter to ensure it uses recent C++ version:

```bash
clang++ -std=c++20 main.cpp -fmodule-file=std=std.pcm
```

**Note:** Modules typically consist of a metadata (BMI) file (.pcm in clang) and an object file (.o file). However, the standard library is a special case: you only build the BMI (std.pcm), and the actual implementation is still provided by the existing libc++.dylib (on macOS). So even if you write **import std**, your program is still linked against the shared libc++ library as usual.

### Tooling limitations

If you're using Visual Studio Code, there’s some bad news: IntelliSense (via C++ extension) does not yet properly support C++ modules (although the code itself compiles fine). You can find more details here: https://github.com/microsoft/vscode-cpptools/issues/6302 . As an alternative, you can switch to Clangd language server, which currently has way better IntelliSense support for modern C++, and has richer features (hints, suggestions, linter etc.). The downside is that you’ll have to do a bit more manual setup.