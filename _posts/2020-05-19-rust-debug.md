---
layout: post
title: "A Future for Rust Debugging"
index_title: "A Future for Rust Debugging"
tags: [rust]
---

Recently, there has been a lot of progress in the Rust logging & tracing ecosystem: projects like [_tracing_](https://github.com/tokio-rs/tracing) make our lives much simpler, allowing us to track down bugs even in complex asynchronous environments in production. However, they still can't replace interactive debuggers like _gdb_ or _lldb_, which provide so much more than just stack traces and variables printing.

In an ideal world, you can use your debugger as a [read-eval-print loop](https://en.wikipedia.org/wiki/Read%E2%80%93eval%E2%80%93print_loop), writing, rewriting, and testing entire functions in an interactive session. Techniques like _post-mortem debugging_ make it possible to prod dead processes, providing immeasurable help in understanding the state of a program just before it crashed. But the harsh reality is that debuggers can be counter-intuituve, hard to use, or [just suck](https://robert.ocallahan.org/2019/11/your-debugger-sucks.html), and it's doubly challenging for Rust.

Considering the complexity of modern Rust projects, it's sometimes outright impossible to debug certain cases because the commonly used debug tools like _gdb_ or _lldb_ are tailored mostly for C or C++, and they make certain assumptions about the way your code is written. Needless to say, these assumptions sometimes can be drastically wrong because C++ and Rust are very different languages. For example, here's a relatively simple program that uses [_thread-local variables_](https://doc.rust-lang.org/stable/std/thread/struct.LocalKey.html):

```rust
use std::cell::RefCell;
thread_local! {
  pub static XYZ: RefCell<u64> = RefCell::new(0);
}
fn main() {
  XYZ.with(|val| {
    *val.borrow_mut() = 42;
  });
}
```

And here's what happens if I want to print the current value of `XYZ` in lldb:

<figure class="fit-to-page">
	<img src="/static/dbg/tls-not-found.png" alt="Attempting to read thread-local variables" style="width: 90" />
	<figcaption>Attempting to read thread-local variables.</figcaption>
</figure>

The first intuitive attempt to print out the variable using the standard Rust syntax fails miserably because _lldb_ doesn't understand Rust at all; in fact, it uses a C++ expression parser instead. But even if we peek into the Rust [thread-local vars implementation](https://github.com/rust-lang/rust/blob/d8878868c8d7ef3779e7243953fc050cbb0e0565/src/libstd/thread/local.rs#L165-L166) and use a C++ expression to print them, it wouldn't work because _lldb_ is [made to think](https://github.com/llvm/llvm-project/blob/e16111ce2fce7fd86c10d3f1dfe3e3b62b76d73b/lldb/source/Plugins/TypeSystem/Clang/TypeSystemClang.cpp#L106-L107) it works with C++, and it doesn't know that Rust has a [new name mangling scheme](https://github.com/rust-lang/rfcs/pull/2603) which has diverted from C++.

And thread-local variables isn't a very contrived case: for example, they are used in the [Tokio runtime implementation](https://github.com/tokio-rs/tokio/blob/f39c15334e74b07a44efaa0f7201262e17e4f062/tokio/src/runtime/context.rs#L1-L8) to store information about tasks, and it would be very helpful to inspect them for understanding how your async code is executed (or, at the very least, this kind of debug info can be helpful for developers of Tokio). Speaking of Tokio and the async ecosystem, it is impossible to print out async backtraces without also having to plod through async runtime internals. Sometimes a runtime can reschedule tasks, losing the original stack context in the process, and if you want to track the origin of an async failure in this case, you are left with only one option -- logs and traces (if there's an alternative way, I'd be happy to learn about it!).

All of these problems have the same underlying cause: Rust has to live in the C/C++-centric world. So what can we do about it (other than _Rewriting-It-In-Rust_, of course)? I think that the ideal systems programming language needs to have ideal, modern debugging tools.

Luckily for us, a debugger like _lldb_ does not insist on using C/C++ for everything. It has a modular core, and languages support can be added [as _plugins_](https://github.com/llvm/llvm-project/tree/master/lldb/source/Plugins) -- and that's exactly what _lldb_ does for C++, which is not really a special case, but a plugin that's built on top of [Clang](https://clang.llvm.org/), an LLVM-based C++ compiler. There has been an effort to implement a similar [language plugin for Rust](https://github.com/tromey/lldb/tree/832406ba3d682c1c59f801e5ee836bdbc26e8bd0/source/Plugins/Language/Rust) -- which, however, is a tricky enterprise, because the lldb plugin API is private<sup>[[1]](#n1)</sup> and it's written -- you guessed it -- in C++. Implementing a parser for a subset of Rust in C++ is hard enough, but keeping up with all the changes in the language is barely possible.

But now, with an effort to ["library-ify" Rust](http://smallcultfollowing.com/babysteps/blog/2020/04/09/libraryification/), I hope that this project can be reinvigorated. And while we are at it, we can try to apply the idea of _compiler as a library_ in another interesting way: _debugger as a library_.

<figure class="fit-to-page">
	<img src="/static/dbg/rust-debugger-scheme.png" alt="A possible structure of the debugger library" />
</figure>

In this example, the private lldb plugin API, which implements the language support, calls into the shared Rust debugger library to parse debug expressions. The debugger library can then delegate parsing to Rust compiler libraries, and in turn use debugger APIs provided by lldb when needed<sup>[[2]](#n2)</sup>.

It should be noted that this hypothetical debugger library will work entirely in the Rust domain, so it will have the access to knowledge and power of [mid-level internal representation](https://rustc-dev-guide.rust-lang.org/mir/index.html). Of course, this doesn't have to work for _lldb_ only: the abstract interface can be ported to other debuggers. And if we make this library itself modular, it opens up a lot of interesting possibilities, like a more complete Rust REPL, custom [visualizations](https://github.com/hediet/vscode-debug-visualizer/tree/master/extension), or _domain-specific debuggers_.

This is not a new idea and there are numerous good examples we can learn from. _mdb_, or a [Modular Debugger](https://illumos.org/books/mdb/preface.html), was [extended to inspect NodeJS programs](https://www.joyent.com/blog/mdb-and-node-js). The .NET Core [runtime diagnostics tool](https://github.com/dotnet/diagnostics) has a [debugger abstraction library](https://github.com/dotnet/diagnostics) with adapters for lldb and Windows Debugger. I'm sure there are many others, but most of these examples boil down to the same idea of extending the debugger core with language-specific capabilities.

## Conclusion

Obviously, this is a huge effort, which might (and most certainly will) require lots of changes across different projects like LLVM, the Rust compiler, or even the DWARF debug format specification<sup>[[3]](#n3)</sup>. Instability of private APIs is definitely one of the challenges here, but it can be partially solved by having stable forks -- which is how it's done in a large number of cases anyway<sup>[[4]](#n4)</sup>. Supporting abstractions like traits can be another difficult task because debuggers are not particularly suited to work with generic code (which is compiled down to concrete types). But I believe that even a basic Rust-aware debugger can make a huge difference in users experience, and I deem it a worthy goal to pursue.

Thank you for reading this!

## References

<a name="n1"></a>
<small>[1]</small> [[lldb-dev] Rust language support question](http://lists.llvm.org/pipermail/lldb-dev/2018-January/013200.html).

<a name="n2"></a>
<small>[2]</small> There is even a [bindings crate](https://github.com/endoli/lldb.rs) for the public part of the lldb API.

<a name="n3"></a>
<small>[3]</small> [Tom Tromey discusses debugging support in rustc](https://www.youtube.com/watch?v=elBxMRSNYr4) (video).

<a name="n4"></a>
<small>[4]</small> E.g., see an [Apple's fork of lldb](https://github.com/apple/llvm-project/tree/6c7c40694408d78286b3839964d144efd1a9669e/lldb/source/Plugins/Language/Swift) to add support for Swift.
