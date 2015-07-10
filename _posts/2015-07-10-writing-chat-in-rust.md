---
layout: post
title: "Rust in Detail: Writing Scalable Chat Service from Scratch"
---

## Part 1: Implementing WebSocket. Introduction.

In this series of articles we'll follow the process of creating a scalable, real-time chat service.  
The objective is to learn more about practical usage of the emerging language Rust and to cover the basic problems of using system programming interfaces, going step by step.

*Part 1* covers the project setup and the implementation of a bare bones WebSocket server. You don't need to know Rust to follow, but some prior knowledge of POSIX API or C/C++ would certainly help. And what would help you even more is a good cup of coffee&nbsp;&mdash;&nbsp;beware that this article is fairly long and very low-level.

Let's begin the journey!

<!--more-->

## Table of Contents

1. [Why Rust?](#why-rust)
1. [Goals](#goals)
1. [Approaches to I/O](#approaches-to-io)
1. [Event Loop](#event-loop)
1. [Starting Project](#starting-project)
1. [Event Loop in Rust](#event-loop-in-rust)
1. [TCP Server](#tcp-server)
1. [Accepting Connections](#accepting-connections)
1. [Parsing HTTP](#parsing-http)
1. [Handshake](#handshake)
1. [Conclusion](#conclusion)
1. [Resources](#resources)
1. [References](#notes)

## 1 Why Rust?

I became interested in Rust because I've always liked systems programming. But while the area of low-level development is joyfully challenging and fulfilling, there's a common belief that it's very hard to do it correctly &mdash; and it's not surprising at all, considering how many non-obvious pitfalls awaiting neophytes and seasoned developers alike.

<img src="/static/rust-1/rust-logo.png" class="float-right" />

But arguably the most common pitfall is the memory safety. It is the root cause for a class of bugs such as [buffer overflows](https://en.wikipedia.org/wiki/Buffer_overflow), [memory leaks](https://en.wikipedia.org/wiki/Memory_leak), [double deallocations](http://stackoverflow.com/questions/21057393/what-does-double-free-mean), and [dangling pointers](https://en.wikipedia.org/wiki/Dangling_pointer). And these bugs can be really nasty: well known issues like the infamous OpenSSL [Heartbleed bug](http://heartbleed.com/) is caused by nothing more than the incorrect management of memory &mdash; and nobody knows how much more of these severe bugs are lurking around.

However, there are several good practice approaches in C++ such as smart pointers<sup>[[1]](#n1)</sup> and on-stack allocation<sup>[[2]](#n2)</sup> to mitigate these common issues. But, unfortunately, it's still too easy to "shoot yourself in the foot" by overflowing a buffer or by incorrectly using one of the low-level memory management functions because these practices and constraints aren't enforced on the language level.

Instead, it's believed that all well-grounded developers would do good and make no mistakes whatsoever. Conversely, I do believe that these critical issues have nothing to do with the level of skill of a developer because it's a machine's task to check for errors&nbsp;&mdash;&nbsp;human beings just don't have that much attention to find all weaknesses in large codebases.

That's the major reason of existence for a common way to automatically handle the memory management: the garbage collection. It's a complex topic and a vast field of knowledge in itself. Almost all modern languages and VMs use some form of GC, and while it's suitable in most cases, it has its own shortcomings &mdash; it's complex<sup>[[3]](#n3)</sup>, it introduces an overhead of a pause to reclaim unused memory<sup>[[4]](#n4)</sup>, and generally requires intricate tuning tricks for high-performance applications to reduce the pause time.

Rust takes a slightly different approach &mdash; basically, a middle ground: automatic memory and resources reclamation without a significant overhead and without the requirement of a tiresome and error-prone manual memory management. It's achieved by employing *[ownership](http://doc.rust-lang.org/stable/book/ownership.html)* and *borrowing* concepts.

The language itself is built upon the assumption that every value has exactly one *owner*. It means that there could be only one mutable variable pointing to the same memory region:

{% highlight rust %}
let foo = vec![1, 2, 3]; 
// We've created a new vector containing elements 1, 2, 3 and
// bound it to the local variable `foo`.

let bar = variable2;
// Now we've handled ownership of the object to the variable `bar`.
// `foo` can't be accessed further, because it has no binding now.
{% endhighlight %}

And, because values bind to exactly one place, resources that they hold (memory, file handles, sockets, etc.) can be automatically freed when the variable goes out of a scope (defined by code blocks within curly braces, `{` and `}`).

This may sound unnecessarily complicated, but, if you think about it, this concept is very pragmatic. Essentially, it's the Rust's killer feature&nbsp;&mdash;&nbsp;these restrictions provide the memory safety that makes Rust feel like a convenient high-level language, while at the same time it retains the efficiency of pure C/C++ code.

Despite all these interesting features, until recently Rust has had some significant disadvantages such as an unstable API, but the long way of almost 10 years<sup>[[5]](#n5)</sup> to the stable version 1.0 has finally come to an end, and the language has matured to the point when we're finally ready to put it into a practical usage.

## 2 Goals

I prefer to learn new languages and concepts by doing relatively simple projects with a real-world application&nbsp;&mdash; this way language features can be learned along the way when they're needed in practice at hand. 

As a project to learn Rust I've chosen an anonymous text chat service similar to Chat Roulette. In my opinion chat is a good choice because chat services require fast responses from a server and an ability to handle many connections at once (we'll strive for many thousands &mdash; that will be a good test for the Rust's raw performance and memory footprint).

Ultimately, the practical end result should be a binary executable and a bunch of deployment scripts to run the service in various cloud environments.

But before we'll start writing the actual code, we need to take a detour to understand the input & output operations, which are essential and crucial for such network services.

## 3 Approaches to I/O

To function properly, our service needs to send and receive data through the network sockets.

It may sound simple of a task, but there are several ways of varying complexity to handle the input and output operations efficiently. The major difference between the approaches lies in the treatment of blocking: the default is to prevent all CPU operations while we're waiting for data to arrive to a network socket.

Since we can't allow one user of our chat service to block others, we need to isolate them somehow.

A common way is to create a separate thread for an each user so that blocking would take effect only in the context of a single thread.
But while the concept is simple and such code is easy to write, each thread requires memory for its stack<sup>[[6]](#n6)</sup> and has an overhead of [*context switches*](https://en.wikipedia.org/wiki/Context_switch) &mdash; modern server CPUs usually have about 8 or 16 cores, and spawning many more threads requires an OS kernel scheduler to work hard to switch execution between them with an adequate speed. For this reason it's hard to scale multithreading to many connections&nbsp;&mdash;&nbsp;in our case it's barely practical (though certainly possible) to spawn several thousands of system threads, because we want to handle that many users without significant costs &mdash; think about the front pages of TechCrunch, HN, and Reddit linking our cool app at once!

## 4 Event Loop

<img src="/static/rust-1/io-multiplex.png" class="float-left" />

So, instead we'll use efficient I/O multiplexing system APIs that employ an event loop &mdash; that's *epoll* on Linux<sup>[[7]](#n7)</sup> and *kqueue* on FreeBSD and OS X<sup>[[8]](#n8)</sup>.

These different APIs work similarly in a straightforward way: bytes are coming over the network, arriving at the sockets, and instead of waiting when the data becomes available for a read, we're *telling a socket to notify us* when new data arrives.

Notifications come in a form of events that end up in the event loop. And that's where the blocking happens in this case: instead of periodic checks for thousands of sockets, we're just waiting for new events to arrive. That's an important distinction because particularly in WebSocket applications it's very common to have many idle clients just waiting for some activity. With asynchronous I/O we'll have a very little overhead of a socket handle and hundreds of bytes at most for an each client.

Interestingly enough, it works great not only for network communications but for disk I/O as well, as the event loop accepts all kinds of file handles (and sockets in the *nix world are just file handles too).

<aside>
Node.js event loop and Ruby's EventMachine work the same way. The same goes for
 the nginx webserver which is built using async I/O<sup><a href="#n9">[9]</a></sup>.
</aside>

## 5 Starting Project

<aside>
I would assume that you're already have installed Rust on your computer.<br/>
If you don't, please follow the <a href="https://doc.rust-lang.org/book/installing-rust.html">official manual</a>.
</aside>

Rust ships with a convenient tool called `cargo` that's similar to Maven/Composer/npm/rake. It manages library dependencies, handles the build process, runs test suites, and simplifies the process of creating a new project.

That's what we need to do now, so let's open a terminal app, and execute the following command:

	cargo new chat --bin

The `--bin` part tells Cargo to create a program instead of a library.

As a result, we get two files:

	Cargo.toml
	src/main.rs

`Cargo.toml` contains a description and dependencies of the project (similar to JavaScript's `package.json`).  
`src/main.rs` is the main source file of our project.  

We need nothing more to start, and now we can compile and execute the program with a single command `cargo run`. Also, it will show us compilation errors if we'd have any.

<aside>
If you're using Emacs, you'll be glad to find out that it's compatible with Cargo out of the box &mdash; you just need to install <code>rust-mode</code> package from MELPA and configure the  compile command to run <code>cargo build</code>.
</aside>

## 6 Event Loop in Rust

Now we're ready to put the theory into practice &mdash; let's start with creating a simple event loop that will wait for new events. Fortunately, we don't need to wire up all system calls to work with the according APIs ourselves &mdash; there's a Rust library, [*Metal IO*](https://github.com/carllerche/mio) (or *mio* for short), that does it for us.

As you remember, library dependencies are managed by Cargo. It gets libraries from [*crates.io*](https://crates.io), the Rust packages repository, but allows to retrieve dependencies straight from Git repositories as well. This feature can be useful when we need to use the latest version of a library that hasn't been packaged yet.

At the moment of this writing `mio` has a package only for the version 0.3, while v.0.4 has some new useful features and breaking API changes, so for now let's use the bleeding edge version by adding the reference to the library to `Cargo.toml`:

	[dependencies.mio]
	git = "https://github.com/carllerche/mio"

After we've added the dependency we need to import it in our code, so let's put it into `main.rs` as well:

{% highlight rust %}
extern crate mio;
use mio::*;
{% endhighlight %}

Usage of `mio` is pretty simple: first, we need to create the event loop by calling `EventLoop::new()` function, and, as the bare event loop isn't useful, we need to make it aware of our chat service. To do that we should define a [*structure*](http://doc.rust-lang.org/stable/book/structs.html) with functions that should conform to a `Handler` interface.

Though Rust doesn't support object-oriented programming in a "traditional" way, structures (or *structs*) are analogous in many ways to classes from the classic OOP, and they can implement interfaces that are enforced by a special language construct called [*traits*](http://doc.rust-lang.org/stable/book/traits.html).

Here's how we define the struct:

{% highlight rust %}
struct WebSocketServer;
{% endhighlight %}

Implement the [`Handler`](https://carllerche.github.io/mio/mio/trait.Handler.html) trait for it:

{% highlight rust %}
impl Handler for WebSocketServer {
    // Traits can have useful default implementations, so in fact the handler
    // interface requires us to provide only two things: concrete types for
    // timeouts and messages.
    // We're not ready to cover these fancy details, and we wouldn't get to them
    // anytime soon, so let's get along with the defaults from the mio examples:
    type Timeout = usize;
    type Message = ();
}
{% endhighlight %}

Start the event loop:

{% highlight rust %}
fn main() {
    let mut event_loop = EventLoop::new().unwrap();
    // Create a new instance of our handler struct:
    let mut handler = WebSocketServer;
    // ... and provide the event loop with a mutable reference to it:
    event_loop.run(&mut handler).unwrap();
}
{% endhighlight %}

That's our first encounter with *borrows*: notice the usage of `&mut` on the last line.

It tells that we're temporary moving the ownership of the value to another binding, with an option to *mutate* (change) the value.

<img src="/static/rust-1/mut-borrow.png" class="centered" />

To simplify matters, you can imagine that the borrowing works like this (in pseudocode):

{% highlight rust %}
// Bind a value to an owner:
let owner = value;

// Create a new scope and borrow the value from the owner:
{
    let borrow = owner;

    // Owner has no access to the value now.
    // But the borrower can read and modify the value:
    borrow.mutate();

    // And then return the borrowed value to the owner:
    owner = borrow;
}
{% endhighlight %}

That code is roughly equal to this:

{% highlight rust %}
// Bind a value to an owner:
let owner = value;
{
    // Borrow the value from the owner:
    let mut borrow = &mut owner;

    // Owner has a read-only access to the value.
    // And the borrower can modify it:
    borrow.mutate();

    // Borrowed value is automatically returned to the owner
    // when it goes out of the scope.
}
{% endhighlight %}

There could be only one *mutable borrow* of a value *per scope*. In fact, even the owner from which the value has been borrowed can't read or change it until the borrow will fall out of a scope.

However, there exists another, more simple way of borrowing that allows to read the value but doesn't allow to modify it&nbsp;&mdash;&nbsp;the *immutable borrowing*. In contrast with `&mut`, there's no limit on count of read-only borrows for a single variable, but as with `&mut` it imposes a limit on modifications: as long as there are immutable borrows of a variable in a scope, the value can't be changed or borrowed mutably.

Hopefully, that was a clear enough description. If it's not, bear with me&nbsp;&mdash;&nbsp;the borrows are everywhere in Rust, so soon we'll get a chance to practice more. Now, let's get back to the project.

Run "`cargo run`" and Cargo will download all prerequisite dependencies, compile the program (showing some warnings that we can ignore at the moment), and run it.

As a result, we'll get the terminal with just a blinking cursor. Not too encouraging, but actually that's a sign of correct execution &mdash; we've started the event loop successfully, although at the moment it does nothing useful for us. Let's fix that.

## 7 TCP Server

To start a TCP server that will be accepting WebSocket connections we'll use a special struct from the `mio::tcp` namespace, `TcpSocket`, and follow the standard workflow of establishing a server-side TCP socket: binding to an address, listening, and accepting connections.

Look at the code:

{% highlight rust %}
use mio::tcp::*;
...
let server_socket = TcpSocket::v4().unwrap();
let address = std::str::FromStr::from_str("0.0.0.0:10000").unwrap();
server_socket.bind(&address).unwrap();

let server_socket = server_socket.listen(128).unwrap();

event_loop.register_opt(&server_socket,
                        Token(0),
                        Interest::readable(),
                        PollOpt::edge()).unwrap();
{% endhighlight %}

And let's go over it, line by line.

First we need to add TCP namespace import to the top of the `main.rs` file:

{% highlight rust %}
use mio::tcp::*;
{% endhighlight %}

Create an IPv4 streaming (TCP) socket: 

{% highlight rust %}
let server_socket = TcpSocket::v4().unwrap();
{% endhighlight %}

Parse the string `"localhost:10000"` to an address structure and bind the socket to it:

{% highlight rust %}
let address = std::str::FromStr::from_str("0.0.0.0:10000").unwrap();
server_socket.bind(&address).unwrap();
{% endhighlight %}

Notice how the compiler infers types for us: because `server_socket.bind` expects an argument of the type `SockAddr`, the Rust compiler can figure out the appropriate type of the `address` for itself, so we don't need to clutter the code with explicit types information.

Start listening:

{% highlight rust %}
server_socket.listen(128).unwrap();
{% endhighlight %}

`listen` takes a single parameter: TCP backlog size<sup>[[10]](#n10)</sup>, that is a size of the queue of sockets waiting to be accepted. Both Linux and FreeBSD have a default maximum of 128 queued connections, so we'll just stick to that value.

Also, you might be wondering why we call `unwrap` almost on every line&nbsp;&mdash;&nbsp;we'll get to that soon.

For now we need to register the socket within the event loop:

{% highlight rust %}
event_loop.register_opt(&server_socket,
                        Token(0),
                        Interest::readable(),
                        PollOpt::edge()).unwrap();
{% endhighlight %}

Arguments for `register_opt` are slightly more complicated:

* *Token* is a unique identifier for a socket. Somehow we need to distinguish sockets among themselves when an event arrives in the loop, and the token serves as a link between a socket and its generated events. Here we link `Token(0)` with the listening socket.
* *Interest* describes our intent for the events subscription: are we waiting for new data to arrive at the socket, for a socket to become available for a write, or for both?
* *PollOpt* are options for the subscription. `PollOpt::edge()` means that we prefer *edge-triggered* events to *level-triggered*.  
<br/>The difference between two can be put this way: the level-triggered subscription notifies when a socket buffer has some data available to read, while the edge-triggered notifies only when new data has arrived to a socket. I.e., in case of edge-triggered notifications, if we haven't read all data available in the socket, we won't get new notifications until some *new* data will arrive. With level-triggered events we'll be getting new notifications as long as there's some data to read in the socket's buffer. You can refer to [StackOverflow answers](http://stackoverflow.com/questions/1966863/level-vs-edge-trigger-network-event-mechanisms) to get more details.

Now, if we'll run the resulting program with `cargo run` and check the `netstat` output, we will see that it's indeed listening on the port number 10000:

	$ netstat -ln | grep 10000
	tcp        0      0 127.0.0.1:10000         0.0.0.0:*               LISTEN

## 8 Accepting Connections

All WebSocket connections start with a *handshake* &mdash; a special sequence of HTTP requests and responses to negotiate the protocol. Hence, we need to teach the server to talk HTTP/1.1 to successfully implement WebSocket. 

We'll need only a subset of HTTP, though: a client wanting to negotiate a WebSocket connection just sends a request with `Connection: Upgrade` and `Upgrade: websocket` headers, and we need to reply in a predefined manner. And that's all: we won't need a full-blown web server to serve static content, etc. &mdash; there are plenty of other tools that will do this for us.

<figure>
	<img src="/static/rust-1/ws-headers.png" class="fig" />
	<figcaption>WebSocket negotiation request headers.</figcaption>
</figure>

But before we start to implement HTTP, we need to properly handle client connections, accepting them and subscribing to the socket events.

Here's the basic implementation:

{% highlight rust %}
use std::collections::HashMap;

struct WebSocketServer {
    socket: TcpListener,
    clients: HashMap<Token, TcpStream>,
    token_counter: usize
}

const SERVER_TOKEN: Token = Token(0);

impl Handler for WebSocketServer {
    type Timeout = usize;
    type Message = ();

    fn readable(&mut self, event_loop: &mut EventLoop<WebSocketServer>,
                token: Token, hint: ReadHint)
    {
        match token {
            SERVER_TOKEN => {
                let client_socket = match self.socket.accept() {
                    Err(e) => {
                        println!("Accept error: {}", e);
                        return;
                    },
                    Ok(None) => panic!("Accept has returned 'None'"),
                    Ok(Some(sock)) => sock
                };

                self.token_counter += 1;
                let new_token = Token(self.token_counter);

                self.clients.insert(new_token, client_socket);
                event_loop.register_opt(&self.clients[&new_token],
                                        new_token, Interest::readable(),
                                        PollOpt::edge() | PollOpt::oneshot()).unwrap();
            }
        }
    }
}
{% endhighlight %}

There is a lot more code now, so let's look at it in more detail.

First thing we need to do is to make the server struct `WebSocketServer` stateful: it needs to contain the listening socket and store connected clients.

{% highlight rust %}
use std::collections::HashMap;

struct WebSocketServer {
    socket: TcpListener,
    clients: HashMap<Token, TcpStream>,
    token_counter: usize
}
{% endhighlight %}

We're using [`HashMap`](https://doc.rust-lang.org/std/collections/struct.HashMap.html) from the standard collections library, `std::collections`, to store client connections. As a key for the hash map we're using a unique, non-overlapping *token* that we should generate for an each connection to identify it.

To provide reasonable uniqueness, we'll just use a simple counter to generate new tokens sequentially. That's why `token_counter` variable is there.

Next, the `Handler` trait from `mio` becomes useful again:

{% highlight rust %}
impl Handler for WebSocketServer
{% endhighlight %}

Here we need to *override* a callback function `readable` within the trait implementation. Overriding means that the `Handler` trait already contains a dummy `readable` implementation (besides other default stubs for several callback functions as well). It does nothing useful, so we need to write our own version to handle read events:

{% highlight rust %}
fn readable(&mut self, event_loop: &mut EventLoop<WebSocketServer>,
            token: Token, hint: ReadHint)
{% endhighlight %}

This function gets called each time a socket becomes available for a read, and we're provided with some useful info about the event through the call arguments: the event loop instance, the token linked to the event source (socket), and a *hint*&nbsp;&mdash;&nbsp;a set of flags that provide further details about the occurred event, as it could be not a read, but a signal that connection has been closed, for example.

The listening socket generate *read* events when a new client arrives into the acception queue and when we're ready to connect with it. And that's what we do next, but first we need to make sure that the event has sourced from the listening socket by doing *[pattern matching](https://doc.rust-lang.org/book/match.html)* on a token:

{% highlight rust %}
match token {
    SERVER_TOKEN => {
        ...
    }
}
{% endhighlight %}

What does it mean? Well, the `match` syntax resembles the standard *switch* construct you can find in "traditional" imperative languages, but it has a lot more power to it. While in e.g. Java `switch` can match only on numbers, strings, and enums, Rust's `match` works on ranges, multiple values, and structures as well, and more than that&nbsp;&mdash;&nbsp;it can capture values from patterns, similar to capturing groups from the regular expressions.

In this case we're matching on a token to determine what socket has generated the event &mdash; and, as you remember, `Token(0)` corresponds to the listening socket on our server. In fact, we've made it a *constant* to be more descriptive:

{% highlight rust %}
const SERVER_TOKEN: Token = Token(0);
{% endhighlight %}

So the above `match` expression is equivalent to `match { Token(0) => ... }`.

Now that we know that we're dealing with the server socket, we can proceed to accepting a client's connection:

{% highlight rust %}
let client_socket = match self.socket.accept() {
    Err(e) => {
        println!("Accept error: {}", e);
        return;
    },
    Ok(None) => unreachable!(),
    Ok(Some(sock)) => sock
};
{% endhighlight %}

Here we're doing the matching again, now over an outcome of the `accept()` function call that returns the result typed as `Result<Option<TcpStream>>`. *`Result`* is a special type fundamental to errors handling in Rust. It wraps around uncertain results such as errors, timeouts, etc., and we might decide what to do with them in each individual case.

But we don't have to&nbsp;&mdash;&nbsp;remember that strange `unwrap()` function that we were calling all the time? It has a standard implementation that terminates the program execution in case if the result is an error and returns it unwrapped if it's normal. So basically by using `unwrap` we're telling that we're interested in the immediate result only, and it's OK to shut down the program if an error has happened.

That's an acceptable behavior in some places. However, in case of `accept()` it's not wise to use `unwrap()` because it may accidentally shut down our entire service, effectively disconnecting all users, and we don't want that. Instead, we're just logging the fact of an error and continuing the execution:

{% highlight rust %}
Err(e) => {
    println!("Accept error: {}", e);
    return;
},
{% endhighlight %}

`Option` is another similar wrapper type that simply denotes that we either have some result or we don't. If we have no result, the type has a value of `None`, and the actual result is wrapped by `Some(value)`. As you might suggest, the type itself can be compared to *null* or *None* values that can be found in many programming languages, but actually `Option` is much safer&nbsp;&mdash;&nbsp;you'll never get the very common `NullReferenceException` error unless you want to, as it works the same way as the `Result` type: when you `unwrap()` the `Option` it shuts down the process if the result is `None`.

So, let's unwrap the `accept()` return value:

{% highlight rust %}
Ok(None) => unreachable!(),
{% endhighlight %}

In this case, there's simply no way for the result to be `None`&nbsp;&mdash;&nbsp;`accept()` would return such value only if we'll try to accept a connection on a non-listening socket. But as we're pretty sure that we're dealing with the server socket, it's safe to crash the program if `accept()` has returned an unexpected value.

So we're just continuing to do the matching:

{% highlight rust %}
let client_socket = match self.socket.accept() {
    ...
    Ok(Some(sock)) => sock
}
{% endhighlight %}

That's the most interesting part. Besides matching the pattern, this line *captures* the value that's wrapped inside the `Result<Option<TcpStream>>` type. This way we can effectively unwrap the value and return it as a result of an *expression*. That means that `match` operation acts as a kind of "function"&nbsp;&mdash;&nbsp;we can return a matching result to a variable.

That's what we do here, binding the unwrapped value to the `client_socket` variable. Next we're going to store it in the clients hash table, while increasing the token counter:

{% highlight rust %}
let new_token = Token(self.token_counter);
self.clients.insert(new_token, client_socket);
self.token_counter += 1;
{% endhighlight %}

And finally we should subscribe to events from the newly accepted client's socket by registering it within the event loop, in the very same fashion as with registration of the listening server socket, but providing another token & socket this time:

{% highlight rust %}
event_loop.register_opt(&self.clients[&new_token],
                        new_token, Interest::readable(),
                        PollOpt::edge() | PollOpt::oneshot()).unwrap();
{% endhighlight %}

You might have noticed another difference in the provided arguments: there is a `PollOpt::oneshot()` option along with the familiar `PollOpt::edge()`. It tells that we want the triggered event to temporarily unregister from the event loop. It helps us make the code more simple and straightforward because in the other case we would have needed to track the current state of a particular socket&nbsp;&mdash;&nbsp;i.e., maintain flags that we can write or read now, etc. Instead, we just simply reregister the event with a desired interest whenever it has been triggered.

Oh, and besides that, now that we've got more detailed `WebSocketServer` struct we must modify the event loop registration code in the main function a bit. Modifications mostly concern the struct initialization and are pretty simple:

{% highlight rust %}
let mut server = WebSocketServer {
    token_counter: 1,        // Starting the token counter from 1
    clients: HashMap::new(), // Creating an empty HashMap
    socket: server_socket    // Handling the ownership of the socket to the struct
};

event_loop.register_opt(&server.socket,
                        SERVER_TOKEN,
                        Interest::readable(),
                        PollOpt::edge()).unwrap();

event_loop.run(&mut server).unwrap();
{% endhighlight %}

## 9 Parsing HTTP

Afterwards, when we've accepted the client socket, by the protocol we should parse the incoming HTTP request to *upgrade* the connection to WebSocket protocol.

We won't do it by hand, because it's quite a boring task&nbsp;&mdash;&nbsp;instead, we'll add another dependency to the project, the `http-muncher` crate that wraps the Node.js's HTTP parser and adapts it for Rust. It allows to parse HTTP requests in a streaming mode that is very useful with TCP connections.

Let's add it to the `Cargo.toml` file:

	[dependencies]
	http-muncher = "0.2.0"

We won't review the API and just proceed to parsing HTTP:

{% highlight rust %}
extern crate http_muncher;
use http_muncher::{Parser, ParserHandler};

struct HttpParser;
impl ParserHandler for HttpParser { }

struct WebSocketClient {
    socket: TcpStream,
    http_parser: Parser<HttpParser>
}

impl WebSocketClient {
    fn read(&mut self) {
        loop {
            let mut buf = [0; 2048];
            match self.socket.try_read(&mut buf) {
                Err(e) => {
                    println!("Error while reading socket: {:?}", e);
                    return
                },
                Ok(None) =>
                    // Socket buffer has got no more bytes.
                    break,
                Ok(Some(len)) => {
                    self.http_parser.parse(&buf);
                    if self.http_parser.is_upgrade() {
                        // ...
                        break;
                    }
                }
            }
        }
    }

    fn new(socket: TcpStream) -> WebSocketClient {
        WebSocketClient {
            socket: socket,
            http_parser: Parser::request(HttpParser)
        }
    }
}
{% endhighlight %}

And we have some changes in the `WebSocketServer`'s `readable` function:

{% highlight rust %}
match token {
    SERVER_TOKEN => {
        ...
        self.clients.insert(new_token, WebSocketClient::new(client_socket));
        event_loop.register_opt(&self.clients[&new_token].socket, new_token, Interest::readable(),
                                PollOpt::edge() | PollOpt::oneshot()).unwrap();
        ...
    },
    token => {
        let mut client = self.clients.get_mut(&token).unwrap();
        client.read();
        event_loop.reregister(&client.socket, token, Interest::readable(),
                              PollOpt::edge() | PollOpt::oneshot()).unwrap();
    }
}
{% endhighlight %}

Now let's review the new code line-by-line again.

First, we import the HTTP parser library and define a handling struct for it:

{% highlight rust %}
extern crate http_muncher;
use http_muncher::{Parser, ParserHandler};

struct HttpParser;
impl ParserHandler for HttpParser { }
{% endhighlight %}

We need the `ParserHandler` trait because it contains callback functions, the same way as the mio's `Handler` for the `WebSocketServer`. These callbacks get called whenever the parser has some new info &mdash; HTTP headers, the request body, etc. But as for now we need only to determine whether the request asks for a WebSocket protocol upgrade, and the parser struct itself has a handy function to check that, so we'll stick to the stub callbacks implementation.

There's a detail: the HTTP parser is stateful, which means that we should create a new instance of it for each new client. Considering each client will contain its own parser state, we need to create a new struct to hold it:

{% highlight rust %}
struct WebSocketClient {
    socket: TcpStream,
    http_parser: Parser<HttpParser>
}
{% endhighlight %}

This struct will effectively replace the `HashMap<Token, TcpStream>` declaration with `HashMap<Token, WebSocketClient>`, so we've added the client's socket to the state as well.

Also, we can use the same `WebSocketClient` to hold code to manage data coming from a client. It'd be too inconvenient to put all the code in the `handle_read` &mdash; the function would quickly become messy and unreadable. So we're just adding a separate handler function:

{% highlight rust %}
impl WebSocketClient {
    fn read(&mut self) {
        ...
    }
}
{% endhighlight %}

It doesn't need to take any arguments because we already have the required state in the containing struct itself.

Now we can read the incoming data:

{% highlight rust %}
loop {
    let mut buf = [0; 2048];
    match self.socket.try_read(&mut buf) {
        ...
    }
}
{% endhighlight %}

Here's what's going on: we're starting an infinite loop, allocate some buffer space to hold the data, and trying to read it to the buffer.

As the `try_read` call may result in an error, we're matching the `Result` type to check for errors:

{% highlight rust %}
match self.socket.try_read(&mut buf) {
    Err(e) => {
        println!("Error while reading socket: {:?}", e);
        return
    },
    ...
}
{% endhighlight %}

Then we check if the read call has resulted in actual bytes:

{% highlight rust %}
match self.socket.try_read(&mut buf) {
    ...
    Ok(None) =>
        // Socket buffer has got no more bytes.
        break,
    ...
}
{% endhighlight %}

It returns `Ok(None)` in case if we've read all the data that the client has sent us. When that happens we go to wait for new events.

And, finally, here's the case when the `try_read` has written bytes to the buffer:

{% highlight rust %}
match self.socket.try_read(&mut buf) {
    ...
    Ok(Some(len)) => {
        self.http_parser.parse(&buf);

        if self.http_parser.is_upgrade() {
            // ...
            break;
        }
    }
}
{% endhighlight %}

Here we're providing the data to the parser, and then check if we have a request to "upgrade" the connection (which means that a user has provided the `Connection: Upgrade` header).

The final part is the `new` method to conveniently create new `WebSocketClient` instances:

{% highlight rust %}
fn new(socket: TcpStream) -> WebSocketClient {
    WebSocketClient {
        socket: socket,
        http_parser: Parser::request(HttpParser)
    }
}
{% endhighlight %}

This is an *associated function* which is analogous to static methods in the conventional OOP systems, and this particular function can be compared to a constructor. In this function we're just creating a new instance of `WebSocketClient` struct, but in fact we can perfectly do the job without the "constructor" function&nbsp;&mdash;&nbsp;it's just a shorthand, because without it the code would quickly become repetitive. After all, the <dfn title="Don't Repeat Yourself">[DRY](https://en.wikipedia.org/wiki/Don't_repeat_yourself)</dfn> principle exists for a reason.

There are few more details. First, notice that we don't use an explicit `return` statement to return the function result. Rust allows to return the result implicitly from a last expression of a function.

Second, this line deserves more elaboration:

{% highlight rust %}
http_parser: Parser::request(HttpParser)
{% endhighlight %}

Here we're creating a new instance of the `Parser` by using an associated function `Parser::request`, and we're creating and passing a new instance of the previously defined `HttpParser` struct as an argument.

Let's get back to the server code.

To finish up, we're making changes in the server socket handler:

{% highlight rust %}
match token {
    SERVER_TOKEN => { ... },
    token => {
        let mut client = self.clients.get_mut(&token).unwrap();
        client.read();
        event_loop.reregister(&client.socket, token, Interest::readable(),
                              PollOpt::edge() | PollOpt::oneshot()).unwrap();
    }
}
{% endhighlight %}

We've added a new match expression that captures all tokens besides `SERVER_TOKEN`, that is the client socket events.

After we've got the token, we can borrow a mutable reference to the corresponding client struct instance from the clients hash map:

{% highlight rust %}
let mut client = self.clients.get_mut(&token).unwrap();
{% endhighlight %}

And let's call the `read` function that we've written above:

{% highlight rust %}
client.read();
{% endhighlight %}

In the end, we've got to reregister the client, because of `oneshot()`:

{% highlight rust %}
event_loop.reregister(&client.socket, token, Interest::readable(),
                      PollOpt::edge() | PollOpt::oneshot()).unwrap();
{% endhighlight %}

As you can see, it doesn't differ much from a client registration routine; in essence, we're just calling `reregister` instead of `register_opt`.

Now we know about a client's intent to initiate a WebSocket connection, and we should think about how to reply to such requests. 

## 10 Handshake

Basically, we could send back just these headers:

    HTTP/1.1 101 Switching Protocols
    Connection: Upgrade
    Upgrade: websocket

Except there's one more important thing&nbsp;&mdash;&nbsp;the WebSocket protocol requires us to send a properly crafted `Sec-WebSocket-Accept` header as well. According to the [RFC](https://tools.ietf.org/html/rfc6455#section-4), there are certain rules: we need to get the `Sec-WebSocket-Key` header from a client, we need to append a long string to the key (`"258EAFA5-E914-47DA-95CA-C5AB0DC85B11"`), then hash the resulting string with the SHA-1 algorithm, and in the end encode the result in base64.

Rust doesn't have SHA-1 and base64 in the standard library, but sure it does have them as separate libraries on *crates.io*, so let's include them to our  `Cargo.toml`:

	[dependencies]
    ...
	rustc-serialize = "0.3.15"
	sha1 = "0.1.1"

`rustc-serialize` contains functions to encode binary data in base64, and `sha1`, obviously, is for SHA-1.

The actual function to encode the key is straightforward:

{% highlight rust %}
extern crate sha1;
extern crate rustc_serialize;

use rustc_serialize::base64::{ToBase64, STANDARD};

fn gen_key(key: &String) -> String {
    let mut m = sha1::Sha1::new();
    let mut buf = [0u8; 20];

    m.update(key.as_bytes());
    m.update("258EAFA5-E914-47DA-95CA-C5AB0DC85B11".as_bytes());

    m.output(&mut buf);

    return buf.to_base64(STANDARD);
}
{% endhighlight %}

We're getting a reference to the key string as an argument for the `gen_key` function, creating a new SHA-1 hash, appending the key to it, then appending the constant as required by the RFC, and return the base64-encoded string as a result.

But to make use of this function we should capture the `Sec-WebSocket-Key` header first. To do that, let's get back to the HTTP parser from the previous section. As you might remember, the `ParserHandler` trait allows us to define callbacks that get called whenever we receive new headers. Now is the right time to use this feature, so let's improve the parser struct implementation:

{% highlight rust %}
use std::cell::RefCell;
use std::rc::Rc;

struct HttpParser {
    current_key: Option<String>,
    headers: Rc<RefCell<HashMap<String, String>>>
}

impl ParserHandler for HttpParser {
    fn on_header_field(&mut self, s: &[u8]) -> bool {
        self.current_key = Some(std::str::from_utf8(s).unwrap().to_string());
        true
    }

    fn on_header_value(&mut self, s: &[u8]) -> bool {
        self.headers.borrow_mut()
            .insert(self.current_key.clone().unwrap(),
                    std::str::from_utf8(s).unwrap().to_string());
        true
    }

    fn on_headers_complete(&mut self) -> bool {
        false
    }
}
{% endhighlight %}

The piece of code is simple, but it introduces a new important concept: *shared ownership*.

As you already know, in Rust we can have only one owner of a certain value, but sometimes we might need to share the ownership. For instance, in this case we need to scan a set of headers for a key while at the same time we're collecting the headers in the parser handler. That means that we need to have 2 owners of the `headers` variable: in `WebSocketClient` and `ParserHandler`.

<img src="/static/rust-1/ref-count.png" class="float-right" />

That's where [`Rc`](https://doc.rust-lang.org/std/rc/) comes to help: it's a value wrapper with a *reference counter*, and in effect we're moving the ownership of a value to the `Rc` container, so the `Rc` itself can be safely shared between many owners with the help of a special language trickery&nbsp;&mdash;&nbsp;we just `clone()` the `Rc` value when we need to share it, and the wrapper safely manages memory for us.

But here's a caveat: the value `Rc` contains is *immutable*, i.e. because of the compiler constraints we can't change it. In fact, it's a natural consequence of Rust's rules about mutability&nbsp;&mdash;&nbsp;it allows to have as many borrows of a variable as you want, but you can change the value only within a single borrow.

That's not too convenient because we need to modify the list of headers as new data becomes available, and we're pretty sure that we're doing that only in one place, so no violations of Rust's rules here&nbsp;&mdash;&nbsp;yet the `Rc` container wouldn't allow us to modify its content.

Fortunately, [`RefCell`](https://doc.rust-lang.org/core/cell/index.html) fixes that&nbsp;&mdash;&nbsp;this is another special container type that allows us to overcome this limit with a mechanism called *interior mutability*. It simply means that we're deferring all compiler checks to the *run time* as opposed to checking borrows *statically*, at *compile time*. So all that we need to do is to double-wrap our value in a (quite monstrous looking) `Rc<RefCell<...>>` container.

Let's consider this code:

{% highlight rust %}
self.headers.borrow_mut()
    .insert(self.current_key.clone().unwrap(),
            ...
{% endhighlight %}

It corresponds to `&mut` borrow with the only difference that all checks for constrained number of mutatable borrows are performed dynamically, so it's up to us to make sure that we're borrowing the value only once.

Now, the actual owner of the `headers` variable would be the `WebSocketClient` struct, so let's define the according properties there and write a new constructor function:

{% highlight rust %}
// Import the RefCell and Rc crates from the standard library
use std::cell::RefCell;
use std::rc::Rc;

...

struct WebSocketClient {
    socket: TcpStream,
    http_parser: Parser<HttpParser>,

    // Adding the headers declaration to the WebSocketClient struct
    headers: Rc<RefCell<HashMap<String, String>>>
}

impl WebSocketClient {
    fn new(socket: TcpStream) -> WebSocketClient {
        let headers = Rc::new(RefCell::new(HashMap::new()));

        WebSocketClient {
            socket: socket,

            // We're making a first clone of the `headers` variable
            // to read its contents:
            headers: headers.clone(),

            http_parser: Parser::request(HttpParser {
                current_key: None,

                // ... and the second clone to write new headers to it:
                headers: headers.clone()
            })
        }
    }

    ...
}
{% endhighlight %}

Now `WebSocketClient` can get access to parsed headers, and, consequently, we can read the header that interests us most, `Sec-WebSocket-Key`. Considering we have the key from a client, the response routine boils down to just sending an HTTP string that we combine out of several parts.

But as we can't just send the data in the non-blocking environment, we need to switch the event loop to notify us when a client's socket becomes available for a write.

The solution is to switch our interest to `Interest::writable()` when we reregister the client's socket.  
Remember this line?

{% highlight rust %}
event_loop.reregister(&client.socket, token, Interest::readable(),
                      PollOpt::edge() | PollOpt::oneshot()).unwrap();
{% endhighlight %}

We just need to store our interest with the other client's state, so let's rewrite it this way:

{% highlight rust %}
struct WebSocketClient {
    socket: TcpStream,
    http_parser: Parser<HttpParser>,
    headers: Rc<RefCell<HashMap<String, String>>>,

    // Adding a new `interest` property:
    interest: Interest
}
{% endhighlight %}

And, accordingly, let's  modify the "reregister" procedure:

{% highlight rust %}
event_loop.reregister(&client.socket, token,
                      client.interest, // Providing `interest` from the client's struct
                      PollOpt::edge() | PollOpt::oneshot()).unwrap();
{% endhighlight %}

The only thing left to do is to change the client's interest value at certain places.

To make it more straightforward, let's formalize the process of tracking the connection states:

{% highlight rust %}
#[derive(PartialEq)]
enum ClientState {
    AwaitingHandshake,
    HandshakeResponse,
    Connected
}
{% endhighlight %}

Here we define an *enumeration* that will describe all possible states for a WebSocket client. These three are simple: first, `AwaitingHandshake` means that we're waiting for a handshake request in HTTP, `HandshakeResponse` represents the state when we're replying to the handshake (again, talking in HTTP), and, after that, `Connected` indicates that we've started to communicate using the WebSocket protocol.

Let's add the client state variable to the client struct:

{% highlight rust %}
struct WebSocketClient {
    socket: TcpStream,
    http_parser: Parser<HttpParser>,
    headers: Rc<RefCell<HashMap<String, String>>>,
    interest: Interest,

    // Add a client state:
    state: ClientState
}
{% endhighlight %}

And modify the constructor, providing the initial state and interest:

{% highlight rust %}
impl WebSocketClient {
    fn new(socket: TcpStream) -> WebSocketClient {
        let headers = Rc::new(RefCell::new(HashMap::new()));

        WebSocketClient {
            socket: socket,
            ...
            // Initial interest
            interest: Interest::readable(),

            // Initial state
            state: ClientState::AwaitingHandshake
        }
    }
}
{% endhighlight %}

Now we can actually change the state in the `read` function. Remember these lines?

{% highlight rust %}
match self.socket.try_read(&mut buf) {
    ...
    Ok(Some(len)) => {
        if self.http_parser.is_upgrade() {
            // ...
            break;
        }
    }
}
{% endhighlight %}

Finally we can replace the placeholder in the `is_upgrade()` condition block with the state change code:

{% highlight rust %}
if self.http_parser.is_upgrade() {
    // Change the current state
    self.state = ClientState::HandshakeResponse;

    // Change current interest to `Writable`
    self.interest.remove(Interest::readable());
    self.interest.insert(Interest::writable());

    break;
}
{% endhighlight %}

After we've changed our interest to `Writable`, let's add the required routines to reply to the handshake.
We will create a function in our `WebSocketServer` handler implementation, `writable`. It's almost identical to `readable`, the only obvious difference being that `writable` is called whenever the client's socket is available for a write:

{% highlight rust %}
fn writable(&mut self, event_loop: &mut EventLoop<WebSocketServer>, token: Token) {
    match token {
        token => {
            let mut client = self.clients.get_mut(&token).unwrap();
            client.write();
            event_loop.reregister(&client.socket, token, client.interest,
                                  PollOpt::edge() | PollOpt::oneshot()).unwrap();
        }
    }
}
{% endhighlight %}

If it's not straightforward enough, try to compare it to `readable` from the [8th section](#accepting-connections).

The remaining part is most simple, we need to build and send the response string:

{% highlight rust %}
use std::fmt;
...
impl WebSocketClient {
    fn write(&mut self) {
        // Get the headers HashMap from the Rc<RefCell<...>> wrapper:
        let headers = self.headers.borrow();

        // Find the header that interests us, and generate the key from its value:
        let response_key = gen_key(&headers.get("Sec-WebSocket-Key").unwrap());

        // We're using special function to format the string.
        // You can find analogies in many other languages, but in Rust it's performed
        // at the compile time with the power of macros. We'll discuss it in the next
        // part sometime.
        let response = fmt::format(format_args!("HTTP/1.1 101 Switching Protocols\r\n\
                                                 Connection: Upgrade\r\n\
                                                 Sec-WebSocket-Accept: {}\r\n\
                                                 Upgrade: websocket\r\n\r\n", response_key));

        // Write the response to the socket:
        self.socket.try_write(response.as_bytes()).unwrap();

        // Change the state:
        self.state = ClientState::Connected;

        // And change the interest back to `readable()`:
        self.interest.remove(Interest::writable());
        self.interest.insert(Interest::readable());
    }
}
{% endhighlight %}

Let's test it by connecting to the WebSocket server from a browser. Open the dev console in your favorite web browser (press `F12`) and put this code there:

{% highlight javascript %}
ws = new WebSocket('ws://127.0.0.1:10000');

if (ws.readyState == WebSocket.OPEN) {
    console.log('Connection is successful');
}
{% endhighlight %}

<img src="/static/rust-1/connection-success.png" class="centered shadowed" />

Looks like we've made it &mdash; the connection is successfully established!

## Conclusion

Whew. That was a whirlwind tour of language features and concepts, but that's just a start&nbsp;&mdash;&nbsp;be prepared for sequels to this article (of course, as lengthy and as boringly detailed as this one!). We have a lot more to cover: secure WebSockets, multithreaded event loop, benchmarking, and, of course, we still have to implement the communication protocol and write the actual application.

But before we get to the app, we might do further refactoring, separating the library code from the application, likely following with eventual publication of the resulting library to *crates.io*.

All current code is available on [GitHub](https://github.com/nbaksalyar/rust-chat)&nbsp;&mdash;&nbsp;feel free to fork it and play with it at your own discretion.

I suggest you to subscribe to the blog in [RSS](http://feeds.feedburner.com/NikitaBaksalyar) or on [Twitter](https://twitter.com/nbaksalyar) if you'd like to follow the updates.

See you soon!

## Resources

* [Journey to the Stack](http://duartes.org/gustavo/blog/post/journey-to-the-stack/) by Gustavo Duarte&nbsp;&mdash;&nbsp;a series of articles that provides a good insight on the inner implementation of the stack and stack frames. It'll help you to better understand the Rust memory management principles.
* [The Linux Programming Interface](http://man7.org/tlpi/) by Michael Kerrisk, 2010&nbsp;&mdash;&nbsp;awesome book on the topic of using system APIs. Despite the title, applicable not only to Linux but to most *nix systems as well.

## References

<a name="n1"></a>
<small>[1]</small> Interestingly enough, Rust actually brings [smart pointers](https://en.wikipedia.org/wiki/Smart_pointer) to the language level: borrowing is based around ideas akin to C++'s `unique_ptr` and `shared_ptr`.&nbsp;[&uarr;](javascript:history.back())

<a name="n2"></a>
<small>[2]</small> For instance, [NASA JPL C Programming Coding Standard](http://lars-lab.jpl.nasa.gov/JPL_Coding_Standard_C.pdf) and <dfn title="Motor Industry Software Reliability Association">MISRA</dfn> C standard explicitly forbid dynamic memory allocation with `malloc()` and implies local variables allocation on the stack and usage of pre-allocated memory instead.&nbsp;[&uarr;](javascript:history.back())

<a name="n3"></a>
<small>[3]</small> While basic garbage collection algos are relatively easy to implement, more smart approaches like concurrent GCs are very non-trivial. In fact, a sign of complexity is that Go getting concurrent GC only in version 1.5 &mdash; almost 3 years since the 1.0 release.&nbsp;[&uarr;](javascript:history.back())

<a name="n4"></a>
<small>[4]</small> Strictly speaking, many implementations of `malloc()` and `free()` functions suffer from a similar overhead as well becase of [fragmentation](https://en.wikipedia.org/wiki/Fragmentation_(computing)). More on that topic can be found in "[Dynamic Memory Allocation and Fragmentation in C and C++](http://www.design-reuse.com/articles/25090/dynamic-memory-allocation-fragmentation-c.html)" by Colin Walls.&nbsp;[&uarr;](javascript:history.back())

<a name="n5"></a>
<small>[5]</small> "Graydon Hoare [&hellip;] started working on a new programming language called Rust in 2006." &mdash; [InfoQ: "Interview On Rust"](http://www.infoq.com/news/2012/08/Interview-Rust)&nbsp;[&uarr;](javascript:history.back())

<a name="n6"></a>
<small>[6]</small> According to `pthread_create(3)` manual page, it defaults to 2 MB on 32-bit Linux systems.&nbsp;[&uarr;](javascript:history.back())

<a name="n7"></a>
<small>[7]</small> For comparison of *epoll* to other system APIs, refer to "[Comparing and Evaluating epoll, select, and poll Event Mechanisms](https://www.kernel.org/doc/ols/2004/ols2004v1-pages-215-226.pdf)" by Louay Gammo, Tim Brecht, et&nbsp;al., University of Waterloo, 2004.&nbsp;[&uarr;](javascript:history.back())

<a name="n8"></a>
<small>[8]</small> "[Kqueue: A generic and scalable event notification facility](http://people.freebsd.org/~jlemon/papers/kqueue.pdf)" by Jonathan Lemon, FreeBSD Project.&nbsp;[&uarr;](javascript:history.back())

<a name="n9"></a>
<small>[9]</small> "[Inside Nginx: How We Designed For Performance Scale](http://nginx.com/blog/inside-nginx-how-we-designed-for-performance-scale/)" by Owen Garrett.&nbsp;[&uarr;](javascript:history.back())

<a name="n10"></a>
<small>[10]</small> [General description](http://www.linuxjournal.com/files/linuxjournal.com/linuxjournal/articles/023/2333/2333s2.html) on LinuxJournal.com. If you're interested in more details, read "[How TCP backlog works in Linux](http://veithen.github.io/2014/01/01/how-tcp-backlog-works-in-linux.html)" by Andreas Veithen.&nbsp;[&uarr;](javascript:history.back())

<hr />

<small>
Many thanks to:  
[Andrey Baksalyar](mailto:andreybaksalyar@yandex.ru) for providing illustrations.  
[Vga](https://github.com/vgacich) for reading drafts and providing corrections.
</small>
