---
title:  "Linker Issues When Replacing Global New Delete"
excerpt: "MSVC link.exe shenanigans when replacing global new delete."
tags: ["MSVC", "Linker", "Allocator"]
---

## Abstract

I started working on my own 2D Game Engine, and I decided to create 3 build configurations:
1. Debug
2. Development
3. Release

With Debug and Development, if you don't explicitly specify the library type,
it will be configured as a shared library.
In Release, we do a monolithic build instead, where we default with static libraries.

This shouldn't be an issue though, except when you start doing very niche code patterns
like replacing the global new/delete to route them to your global allocator.

It is defined by the standard that you can replace those functions.
From [operator new](https://en.cppreference.com/w/cpp/memory/new/operator_new), Global replacements: 
> The versions (1-4) are implicitly declared in each translation unit
> even if the <new> header is not included.
> Versions (1-8) are replaceable: a user-provided non-member function
> with the same signature defined anywhere in the program, in any source file,
> replaces the default version. Its declaration does not need to be visible.

_What's the issue then?_

So, my game engine, has various modules, and platform-specific "runners", like Win32 etc.
Those runners have the bare minimum code to process OS messages and then forwards them to the game engine itself.

To understand what was going on, I needed to add the `/verbose` flag to the linker's flags, and see that:
1. When the `win32runner.exe` is being linked, the linker searches for a `.lib` to resolve external symbols,
2. Finds `MSVCRT.lib`, and links it with the executable,
3. Then links the executable with my monolithic `.lib`, and complains about multiple defined symbols, as my replacements for new/delete are also found in the `MSVCRT.lib`.

Sadly, that's an issue with the toolchain itself, and my solution consisted in:
1. The Engine module is now a shared library
2. The Engine module, during monolithic builds, statically links against all other modules
3. The Runner executable, dynamically links against the Engine module.

Since we do not perform any heap allocations in the runner (remember it's a simple wrapper of the native event loop), we still manage to use

## Credits
* [Visual Studio 2017 MSVCRT.lib link error - Microsoft Developer Community](https://developercommunity.visualstudio.com/t/visual-studio-2017-msvcrtlib-link-error/534202)
* [operator new](https://en.cppreference.com/w/cpp/memory/new/operator_new)
* [operator delete](https://en.cppreference.com/w/cpp/memory/new/operator_delete)