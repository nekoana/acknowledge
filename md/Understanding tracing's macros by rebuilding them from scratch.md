> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [dietcode.io](https://dietcode.io/p/tracing-macros/)

> The tracing / log crates expose complicated-looking macros like `trace!` that accept a whole bunch of......

The [`log`](https://docs.rs/log/0.4.17/log/index.html) crate exposes a bunch of useful macros like [`trace!`](https://docs.rs/log/0.4.17/log/macro.trace.html) that can take in a message in the form of a format string, a-la [std’s `println!`](https://doc.rust-lang.org/stable/std/macro.println.html). The (currently WIP) [`kv` module](https://docs.rs/log/0.4.17/log/kv/index.html) also lets you pass in key-value pairs to this macro to be able to do structured logging.

[`tracing`’s `trace!` macro](https://docs.rs/tracing/0.1.37/tracing/macro.trace.html) takes this one step further by letting you provide all kinds of different kinds of arguments. [Check out its usage](https://docs.rs/tracing/0.1.37/tracing/index.html#using-the-macros). It looks like a complex macro (and it is in some respects), so to understand how it works under the hood, let’s build it ourselves. Absolutely required reading for this is [my post about TT munchers](https://dietcode.io/t/tt-munchers) (it doesn’t have to be my post, just that familiarity with TT munchers is essential to understanding what we’re about to do). Believe it or not, TT munchers are _all_ you need!

### Setting goals

I want to build a `trace!` macro that takes in some arguments, creates a log message out of them, then writes that message to `stdout` in the [logfmt](https://brandur.org/logfmt) format. I chose logfmt because it is easy to read, implement, aggregate & filter on later. You’ve probably seen logs like this in the wild even if you’re reading that name for the first time. They look like this:

```
level=trace method=GET path=/ status=200 message="Request processed"
```

Here are my requirements:

*   The macro will take in a required `message` argument in the form of a format string (like the `println!` macro does) that may or may not have interpolated variables.
*   Optionally, I want to be able to pass in key-value pairs that will be treated as separate fields in my logfmt log. Keys are just an identifier, and the value is a variable currently in scope.
*   Because not everything implements `Display`, I also want to be able to tell the macro to use the `Debug` implementation when writing a key-value pair to `stdout`.
*   For convenience, I’d also like to be able to specify key-value pairs in a shorthand form when the key name is the same as the variable I’m logging.

The API is heavily inspired from [`tracing`’s `trace!` macro](https://docs.rs/tracing/0.1.37/tracing/macro.trace.html), and for good reason, it is really convenient to use. You can read about [its usage](https://docs.rs/tracing/0.1.37/tracing/index.html#using-the-macros), but here are a few examples as well:

```
let ip = "0.0.0.0";
let port = 7096;
let addr = Addr { ip, port }; // `Addr` only implements `Debug`, which prints `{ip}:{port}`

// Message only (as a format string)
trace!("Listening on {ip}:{}. Address is {:?}", port, addr);
// Output => level="trace" message="Listening on 0.0.0.0:7096"

// Key-value pairs, with a format-string message
trace!(ip = addr.ip, port = addr.port, "Incoming connection");
// Output => level=trace message="Incoming connection" ip="0.0.0.0" port=7096

// Key-value pairs with a Debug sigil
trace!(addr = ?addr, "Incoming connection");
// Output => level=trace message="Incoming connection" addr="0.0.0.0:81138"

// Shorthand key-value pairs
trace!(ip, port, "Listening on :{}", port);
// Output => level="trace" message="Listening on :7096" ip="0.0.0.0" port=7096

// Shorthand key-value pairs with a Debug sigil
trace!(?addr, ip, "Incoming connection");
// Output => level="trace" message="Incoming connection" addr="0.0.0.0:7096" ip=0.0.0.0
```

We’ll start simple, building one feature at a time, not worrying about efficiency, and then improve things towards the end.

Testing our macro is as simple as writing regular old tests with macro invocations. If the macro compiles, it works! Testing / verifying _what_ this macro does isn’t a huge deal, because presumably the code it generates will have tests of its own. I’m going to list bare-bones test cases as we go along.

### Message-only

Let’s begin by only accepting a format string as a message, and printing that to `stdout`. The [`tt`](https://veykril.github.io/tlborm/decl-macros/minutiae/fragment-specifiers.html#tt) fragment specifier comes in clutch for this, and it matches exactly what we want:

```
#[macro_export]
macro_rules! trace {
  ( $($msg:tt)+ ) => {
    println!($($msg)+);
  };
}

// --- Tests ---
trace!("Some message");
trace!("With vars {}", {port});
trace!("With interpolated var {port}");
trace!("With mixed vars {} {port}", ip);
trace!("With format directives {:?} {port}", addr);
```

Note how I’m expecting “at least one” match by using a `+` instead of a `*`. This is because it doesn’t quite make sense to have a log line without a `message` in our context.

`tt` was a weird specifier for me to grasp when starting out. I now see it as one that matches one format-string variable (roughly), but [this comment](https://www.reddit.com/r/rust/comments/iim7b7/comment/g37qxf0/?context=3) from Reddit user `u/A1oso` explains it really well (quoting here in case it gets deleted):

> ```
> `tt` stands for "token tree", which is either a single token, or a bunch of tokens surrounded by (parentheses), [brackets] or {braces}. Everything can be matched by a list of token trees. For example, `println!("{}", x)` consists of three token trees: `println`, `!` and `("{}", x)`.
> ```

Going by this description, `tt` sounds perfect! It might sound overly generic, but in fact the standard library’s [`println!`](https://doc.rust-lang.org/stable/std/macro.println.html) uses it too, so we’re in good company.

Alright, this was easy, let’s move on.

### Key-value pairs

Next I want to add support for the “full” form of key-value pairs, AKA `key = value`. We’ll tackle the `?` format sigil later, so for now let’s just assume the `value` variable implements `Display`. We’re going to use a TT muncher for these. This section is probably the most important in this post. Future sections build _directly_ on top of this!

Let’s first try and match a single key-value pair:

```
#[macro_export]
macro_rules! trace {
  ($key:ident = $value:expr, $($msg:tt)+) => {};
  ($($msg:tt)+ ) => {
    println!($($msg)+);
  };
}

// --- Tests ---
// Previous test cases
trace!(ip = ip, "Message");
trace!(ip = ip, "Message with vars: {ip}:{}", port);
```

`macro_rules!` matches from top-to-bottom, so it is important that our new rule comes in before the previous one. We’re matching a valid Rust identifier, followed by a literal `=`, then an expression, then a format message. This works, but as a little nitpick, [`logfmt`](https://brandur.org/logfmt) allows keys to have dots in their names, and that sounds useful, so I’m going to allow that by changing the rule slightly:

```
#[macro_export]
macro_rules! trace {
  ($($key:ident).+ = $value:expr, $($msg:tt)+) => {};
  ($($msg:tt)+ ) => {
    println!($($msg)+);
  };
}

// --- Tests ---
// Previous test cases
trace!(http.status = 200, "Message");
trace!(http.status = 401, "Message with vars: {ip}:{}", port);
```

All I’ve done is allow `$key` to repeat more than once, with a `.` separating the repetitions. Note that `http.status` is just an identifier (`ident`), it doesn’t have to be a living variable (even though it might _look_ like one)

Okay, so this is matching a single key-value pair now, but we want to be able to match multiple ones. You might think that surrounding the first rule in a `$()*` will do the trick, but it’ll become clear why I’m using a TT muncher later. What I want to do is match one key-value pair, print it to `stdout`, then recursively call this macro to match the next key-value pair. Once we’ve exhausted all of them, our last rule should act as the “base case” for our recursion, and it will print out the message to `stdout` as well. TT munchers are super easy to write, so this is all we need:

```
#[macro_export]
macro_rules! trace {
  ($($key:ident).+ = $value:expr, $($others:tt)+) => { // <-- Just added
    print!("{}={} ", $($key).+, $value);
    $crate::trace!($($others)+);
  };
  ($($msg:tt)+ ) => {
    println!($($msg)+);
  };
}

// --- Tests ---
// Previous test cases
trace!(ip = ip, "Message");
trace!(ip = ip, port = port, "Message");
```

Our macro’s a little bigger than before, but I’m doing exactly what I described above. If you run this though, you’ll notice some weird output:

```
0.0.0.0=0.0.0.0 Message
0.0.0.0=0.0.0.0 7096=7096 Message
```

Running `cargo expand` should make it obvious why. This is what we’re generating:

```
fn main() {
  let ip = "0.0.0.0";
  let port = 7096;
  let addr = Addr { ip, port };
  {
    ::std::io::_print(format_args!("{0}={1} ", ip, ip));
  };
  {
    ::std::io::_print(format_args!("{0}={1} ", port, port));
  };
  {
    ::std::io::_print(format_args!("Message\n"));
  };
}
```

Instead of using the key as an indentifier, we’re treating it as a variable! In fact if you use a key name that isn’t a live variable, this macro will not compile. What we want to do is stringify the key name as-is, without trying to resolve it to a live variable. Turns out `std`’s [`stringify!`](https://doc.rust-lang.org/stable/std/macro.stringify.html) is what we need. Let’s use this:

```
#[macro_export]
macro_rules! trace {
  ($($key:ident).+ = $value:expr, $($others:tt)+) => {
    print!("{}={} ", stringify!($($key).+), $value);
    $crate::trace!($($others)+);
  };
  ($($msg:tt)+ ) => {
    println!($($msg)+);
  };
}

// --- Tests ---
// Previous test cases
trace!(ip = ip, "Message");
trace!(ip = ip, port = port, "Message");
```

_Now_ things look good:

```
ip=0.0.0.0 port=7096 Message with vars: 0.0.0.0:7096
```

Believe it or not, you’ve just made it through the hump of the post. _Everything_ else that follows is EZ.

### Shorthand key-value pairs

You’ve made “full-form” key-value pairs work. Shorthand key-value pairs are just notations where the key name is the same as the value variable’s name. This means we can coerce the shorthand form into the “full” form and let our TT muncher take care of the rest. Here’s how:

```
#[macro_export]
macro_rules! trace {
  ($($key:ident).+, $($others:tt)+) => { // <-- Just added
    $crate::trace!($($key).+ = $($key).+, $($others)+);
  };
  ($($key:ident).+ = $value:expr, $($others:tt)+) => {
    print!("{}={} ", stringify!($($key).+), $value);
    $crate::trace!($($others)+);
  };
  ($($msg:tt)+ ) => {
    println!($($msg)+);
  };
}

// --- Tests ---
// Previous test cases
trace!(ip, "Message");
trace!(addr.ip, port = port, "Message");
```

See that? We’ve just called our macro again by transforming the short-hand form into the “full” form. It should start to become clear why we’re using a TT muncher here instead of a plain `$()*` repetition rule now: the former lets us re-use existing branches, and allows for “mix-and-match” styles of key-value pairs, where some are shorthand, and others are the “full” form.

### Format sigils

Because we’re using `println!("{}")` format our key-value pairs, variables that do not implement `Display` are currently impossible to work with. To fix this, we need to be able to `println!("{:?}")` instead right? We can do this just by adding two more similar-but-not-identical branches to our macro that will match when a `?` sigil is provided in front of our variable:

```
#[macro_export]
macro_rules! trace {
  ($($key:ident).+ = $value:expr, $($others:tt)+) => {
    print!("{}={} ", stringify!($($key).+), $value);
    $crate::trace!($($others)+);
  };
  ($($key:ident).+ = ?$value:expr, $($others:tt)+) => { // <-- Just added
    print!("{}={:?} ", stringify!($($key).+), $value);
    $crate::trace!($($others)+);
  };
  ($($key:ident).+, $($others:tt)+) => {
    $crate::trace!($($key).+ = $($key).+, $($others)+);
  };
  (?$($key:ident).+, $($others:tt)+) => { // <-- Just added
    $crate::trace!($($key).+ = ?$($key).+, $($others)+);
  };
  ($($msg:tt)+ ) => {
    println!($($msg)+);
  };
}

// --- Tests ---
// Previous test cases
trace!(addr = ?addr, "Message");
trace!(?addr, port, "Message");
```

The difference is pretty subtle, so look closely. I’ve added a rule that matches when the `$value` is preceeded by a `?`, and another that matches when the shorthand key-value pair is preceeded by a `?`. In these cases, I use the `{:?}` format specifier in my `print!()` call, which will tell the compiler to use the `Debug` implementation. The short-hand version still transforms itself into the “full” version and recurses, just like before.

What’s extremely cool is that because our macro is a TT muncher, we can mix-and-match all four forms of our key-value pairs! The _order_ of our `macro_rules!` is very important to make sure that happens.

At this point, we’re pretty much done. We’ve got a pretty robust macro that can do all kinds of log printing for us and is _super_ easy to use, but we can improve things quite a bit.

### Writing logs to a file

Printing to `stdout` is pretty “free”, in that the `stdout` streams are line-buffered automatically. This means that all our writes are buffered until we pass in a newline character, which helps with performance. However, our current implementation isn’t going to work very well if we want to write to a file in the same way. We don’t want to be doing frequent small writes all the time, instead, we’d want to at least buffer at the “log” level, AKA we want to write one log _line_ at a time, instead of one log _field_ at a time to begin with. To do this, we’ll want to collect our message & key-value pairs into an iterable data structure, then write it in one go.

We already have a TT muncher. Instead of “consuming” key-value pairs as we encounter them, what if we normalized all of them into an intermediate “format”, and then iterated over it once we’ve exhausted all of them? `tracing` uses a type-erased [`Value`](https://docs.rs/tracing/0.1.37/tracing/trait.Value.html) trait to represent this intermediate format, but we can just use a `String` to keep things simple. Another useful observation is that the “message” is just a key-value pair as well, with the key name `message` and the value as a formatted string.

Alright, so what I’m going to do is add an expression list surrounded by `{}` to the _beginning_ of all my `macro_rules!`. It’ll start off empty, and as we encounter key-value pairs, I will `format!()` them into a String, and add to this expression list. Why surround it in a `{}`? Just to differentiate it from the rest of the TT muncher, and so that I can ensure that my `macro_rules!` is always matching the right thing. I will also put this into a “hidden” macro for reasons I will get into in _just_ a little bit.

```
#[doc(hidden)]
#[macro_export]
macro_rules! __internal_log {
  ({ $($kv:expr),* }, $($k:ident).+ = $v:expr, $($fields:tt)*) => {
    // Matches "full" form key-value pairs
    $crate::__internal_log!({ $($kv,)* format!("{}={}", stringify!($($k).+), $v) }, $($fields)*)
  };
  ({ $($kv:expr),* }, $($k:ident).+ = ?$v:expr, $($fields:tt)*) => {
    // Matches "full" form key-value pairs with the debug sigil
    $crate::__internal_log!({ $($kv,)* format!("{}={:?}", stringify!($($k).+), $v) }, $($fields)*)
  };
  ({ $($kv:expr),* }, $($k:ident).+, $($rest:tt)*) => {
    // Matches shorthand key-value pairs
    $crate::__internal_log!({ $($kv,)* format!("{}={}", stringify!($($k).+), $($k).+) }, $($rest)*)
  };
  ({ $($kv:expr),* }, ?$($k:ident).+, $($rest:tt)*) => {
    // Matches shorthand key-value pairs with the debug sigil
    $crate::__internal_log!({ $($kv,)* format!("{}={:?}", stringify!($($k).+), $($k).+) }, $($rest)*)
  };
  ({ $($kv:expr),* }, $($msg:tt)*) => {
    // Matches when we've exhausted all key-value pairs and the only thing left is the message format-string
    $crate::__internal_log!({ $($kv,)* format!("message={}", format!($($msg)*)) })
  };
  ({ $($kv:expr),* }) => {
    // Matches when we've exhausted everything
    vec![ $($kv,)* ]
  };
}
```

This _looks_ big and scary but really all I’ve done is added a `{ $($kv:expr),* },` in front of all expressions from our previous macro, and added a “base case” that collects all key-value pairs into a vector instead of printing it out. Before we try and follow the logic here, this is what our `trace!` macro will look like:

```
#[macro_export]
macro_rules! trace {
  ($($key:ident).+ = $value:expr, $($others:tt)+) => {
    $crate::__internal_log!({ }, $($key).+ = $value, $($others)+)
  };
  ($($key:ident).+ = ?$value:expr, $($others:tt)+) => {
    $crate::__internal_log!({ }, $($key).+ = ?$value, $($others)+)
  };
  ($($key:ident).+, $($others:tt)+) => {
    $crate::__internal_log!({ }, $($key).+, $($others)+)
  };
  (?$($key:ident).+, ?$($others:tt)+) => {
    $crate::__internal_log!({ }, ?$($key).+, $($others)+)
  };
  ($($msg:tt)+) => {
    $crate::__internal_log!({ }, $($msg)+)
  };
}
```

Okay so a lot of things have happened all of a sudden, let’s break them down.

*   Our `trace!` macro no longer recurses. Instead, it initializes an empty `{}` and passes control over to our “internal” macro
*   The internal macro behaves almost exactly as the one we’ve been working on so far, but instead of `print!()`ing each key-value pair as it sees them, it `format!()`s them into a string, adds it to the expression list `{}`, then calls itself. The expression list will actually just be comma-separated `String`s.
*   Once we’ve exhausted all key-value pairs, the only thing remaining would be the log message, so it `format!`s it, adds it to our expression list, then calls itself again
*   Once we’ve exhausted everything, it creates a `Vec<String>` out of all the accumulated expression list `String`s

If we `cargo run` this:

```
fn main() {
  let ip = "0.0.0.0";
  let port = 7096;
  let addr = Addr { ip, port };

  let v = trace!(?ip, port = ?port, "Message");
  println!("{:?}", v);
}
```

you should see:

```
["ip=\"0.0.0.0\"", "port=7096", "message=Message"]
```

Pretty neat! Now we have everything we need, and instead of creating a `Vec<String>` out of our key-value pairs, we can process them “in one go”. If we wanted to keep the key & values separate, we could accumulate a tuple of strings `(key: String, value: String)` instead of a single `format!`ed `String`.

A secondary advantage of doing all this is that we can now create `info!` / `debug!` / `warn!` / `error!` macros as well. These will basically be a copy of our `trace!` macro, but instead of initializing our expression list as an empty `{}`, we can hard-code a `level` key with an appropriate value for all of them! The bulk of our logic stays in the `__internal_log` macro, but the user-facing public API is controlled by the `trace!` / `info!` / `debug!` / `warn!` / `error!` macros’ `macro_rules!`, which I think is awesome.

The last piece of the puzzle now is writing to a file. Of course you do not want to be opening and closing a file all the time, so the way I do this is by holding a global, `static` file handle in a `OnceCell`, and then locking it when I want to write to it. The [`once_cell`](https://docs.rs/once_cell/latest/once_cell/) crate (if it hasn’t been stabilized into `std` already) is your friend.

To further reduce the number of writes we do, we can even maintain a global, static struct with a [`BufWriter`](https://doc.rust-lang.org/stable/std/io/struct.BufWriter.html), and use _that_ to lock and write to the file.

To get around the tons of `String`allocations we’re doing,`tracing` uses the [`sharded-slab`](https://docs.rs/sharded-slab/latest/sharded_slab/) crate, which is pretty cool.

Chances are that if you’ve followed up until this point, you’ll be able to figure this stuff out. Please reach out if you have any questions!