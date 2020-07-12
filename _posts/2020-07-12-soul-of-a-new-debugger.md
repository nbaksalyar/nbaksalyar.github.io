---
layout: post
title: "The Soul of a New Debugger"
index_title: "The Soul of a New Debugger"
tags: [rust]
---

_This blog post is a follow-up to [A Future for Rust Debugging](/2020/05/19/rust-debug.html)_

Why developers are reluctant to use interactive debuggers and prefer `print`s instead?

This is a question that has been perplexing me, and while it's not possible to have a definite answer, we can make some educated guesses.
`print`s are simple and very effective: you don't have to painstakingly step through your code to understand what's happening
and you don't need any external tools -- everything is already there, you just need to add a few statements and run your program again.

Here's the trick, though: if you code in a language like Python, JavaScript or Ruby, all you need to do to run your program is to execute it.
But there is a bit more friction with compiled native languages like C++ or Rust: every recompilation step costs time, and with larger
code bases with many dependencies it can quickly become non-negligible. It's even more complicated if you want to debug a problem that occurs on a remote machine (e.g., a production server) or on an embedded platform, since you will need to redeploy the newly compiled binary every time you want to run a new debug experiment.

Interactive debuggers solve these problems: you can take an existing program that was compiled in the debug mode and probe it to your liking,
setting up conditional breakpoints, looking up current values of variables, and doing a lot of other useful things. However, in my view, the widely-used interactive debuggers have a set of problems of their own:

- A lot of them were designed in a different era. At that time, we didn't have always-on Internet connections, high-resolution multi-colour displays, and devices with many gigabytes of memory. While the fundamental principles remain the same, the modern needs have changed and so do modern means.

- Many existing debuggers are general-purpose. While there is some support for scripting and language-specific extensions, it's harder to use them for specific domains and it's harder to integrate them with your dev environment. In practical terms, this means you have to use _other_ tools in addition to debuggers to observe the behaviour of your software, and oftentimes it's just easier & faster to move along with `print`s instead of trying to find a suitable tool.

I also see more general problems with regards to debuggers. They are perceived mostly as a tool to help you find and fix bugs in your code, but not as much as a tool of discovery and exploration. It's been 8 years since Bret Victor published his "[Learnable Programming](http://worrydream.com/#!/LearnableProgramming)", and these ideas are as relevant today as ever. Debuggers should strive to be a good learning tool too.

Lastly, interactive debuggers rely on underlying principles that are quite similar to profilers, dynamic tracers like [DTrace](https://en.wikipedia.org/wiki/DTrace) and [eBPF](https://en.wikipedia.org/wiki/eBPF), memory leak detectors, and other developer tools. Ideas and code can -- and should -- be shared across this ecosystem, but it seems like whilst one type of tools gets a lot of attention, the others may still lack support for important features.

So what can we do to try solving these problems?

## Elements of a modern debugger

In my [previous article](/2020/05/19/rust-debug.html), I argued for a case of extending the existing debuggers to provide better support for Rust. However, after some more research and thinking, I feel that we should consider the idea of creating a new debugger framework from scratch, taking inspiration from other great projects. Not-invented-here syndrome aside, I think there are some valid reasons for going in this direction.

Modern languages like Rust have lots of new important features that weren't available in languages that the classic debuggers were written in. Things like fearless concurrency, first-class modules & packages, async I/O and zero-cost abstractions can affect the design of a new debugger in a significant way. With the Rust's package manager, [Cargo](https://doc.rust-lang.org/cargo/), extensibility becomes even more relevant and viable. We've already seen what modular debuggers are capable of -- for example, the Illumos Modular Debugger, [mdb](https://illumos.org/books/mdb/preface.html), allows to debug both native and JavaScript code in Node.js with an [mdb_v8](https://github.com/joyent/mdb_v8) extension, and the architecture of the debugger itself allows to extend it further using a simple, modular interface. With language features like traits, it can become even easier to create your own domain-specific debuggers using a new framework.

Another project to take inspiration from is [Delve](https://github.com/go-delve/delve), a Go debugger. It's built with modern tools and has several features important for Go developers, but I want to highlight the [JSON-RPC API](https://github.com/go-delve/delve/tree/master/Documentation/api/json-rpc) it provides. This API addresses the important problem of integration with the development environment, and by using [a well-defined protocol](https://microsoft.github.io/debug-adapter-protocol/) we can create an ecosystem for debuggers that's similar to the one that's flourished around the [language server protocol](https://microsoft.github.io/language-server-protocol/). With an HTTP-based API, we can build custom debugger front-ends using HTML and WebAssembly and run them in web browsers. With the rich front-end tools & frameworks, it opens up lots of interesting options for data representation and visualisation. Integration doesn't have to be one-sided, too: language servers can be reused for debugging purposes, and integrating a Rust-specific debugger with the Rust compiler would allow us to utilise the full power of the existing language syntax parser and other compiler components.

## What's next?

Overall, I believe this is a project worth building. While we're seeing a lot of innovation in compilers and language design, debuggers are somewhat neglected, even though debugging is no less important; as Kernighan's law postulates, it's twice as hard as writing the code in the first place.

Creating a new debugger is an insurmountable task and a very long journey. But it can have a modest start: if we cover only a few popular operating systems and a few simple cases first, we can quickly achieve small wins that can save us a lot of time and frustration.

This post outlines the initial project plan for **[Headcrab](https://github.com/nbaksalyar/headcrab)**, a Rust debugger library. I will be publishing more code and documentation in the coming weeks. If you are interested in updates, please [follow me on Twitter](https://twitter.com/nbaksalyar). Progress updates will be also published on this blog.

* You can find a more detailed roadmap in the [project repository](https://github.com/nbaksalyar/headcrab/README.md).

## Resources and further reading

- [Debugging support in the Rust compiler](https://rustc-dev-guide.rust-lang.org/debugging-support-in-rustc.html)
- [mdb, Illumos Debugger](https://illumos.org/books/mdb/preface.html)
- Rosenberg, J.B. (1996). _How debuggers work : Algorithms, data structures, and architecture_. ISBN 0471149667.
- Uresh Vahalia (1996). _UNIX internals : the new frontiers_. ISBN 9780131019089.
