# `tt` is for "Tools of the Trade"

This is a build of clang and `lld` using _mostly_ LLVM stuff statically built-in. Specifically:
* LLVM's `libc++`, compiler runtime and unwinder are statically built into clang, and used by default (without `-stdlib=libc++` and others).
* `lld` is used by default.

Notably, this build still requires glibc, and is dynamically linked against it. I don't particularly care about the version, so it's whatever ships with _some_ Ubuntu. That means it won't work on older Linux.
