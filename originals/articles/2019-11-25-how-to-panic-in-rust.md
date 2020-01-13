# How to Panic in Rust

What exactly happens when you `panic!()`?
I recently spent a lot of time looking at the parts of the standard library concerned with this, and it turns out the answer is quite complicated!
I have not been able to find docs explaining the high-level picture of panicking in Rust, so this feels worth writing down.

<!-- MORE -->

(Shameless plug: the reason I looked at this is that @Aaron1011 implemented unwinding support in Miri.
I wanted to see that in Miri since forever and never had the time to implement it myself, so it was really great to see someone just submit PRs for that out of the blue.
After a lot of review rounds, this landed just recently.
There [are still some rough edges](https://github.com/rust-lang/miri/issues?q=is%3Aissue+is%3Aopen+label%3AA-panics), but the foundations are solid.)

The purpose of this post is to document the high-level structure and the relevant interfaces that come into play on the Rust side of this.
The actual mechanism of unwinding is a totally different matter (and one that I am not qualified to speak about).

*Note:* This post describes panicking as of [this commit](https://github.com/rust-lang/rust/commit/7d761fe0462ba0f671a237d0bb35e3579b8ba0e8).
Many of the interfaces described here are unstable internal details of libstd, and subject to change any time.

## High-level structure

When trying to figure out how panicking works by reading the code in libstd, one can easily get lost in the maze.
There are multiple layers of indirection that are only put together by the linker,
there is the [`#[panic_handler]` attribute](https://doc.rust-lang.org/nomicon/panic-handler.html) and the ["panic runtime"](https://github.com/rust-lang/rfcs/blob/master/text/1513-less-unwinding.md) (controlled by the panic *strategy*, which is set via `-C panic`) and ["panic hooks"](https://doc.rust-lang.org/std/panic/fn.set_hook.html),
and it turns out panicking in `#[no_std]` context takes an entirely different code path... there is just a lot going on.
To make things worse, [the RFC describing panic hooks](https://github.com/rust-lang/rfcs/blob/master/text/1328-global-panic-handler.md) calls them "panic handler", but that term has since been re-purposed.

I think the best place to start are the interfaces controlling the two indirections:

* The *panic runtime* is used by libstd to control what happens after the panic information has been printed to stderr.
  It is determined by the panic *strategy*: either we abort (`-C panic=abort`) or we unwind (`-C panic=unwind`).
  (The panic runtime also provides the implementation for [`catch_unwind`](https://doc.rust-lang.org/std/panic/fn.catch_unwind.html) but we are not concerned with that here.)

* The *panic handler* is used by libcore to implement (a) panics inserted by code generation (such as panics caused by arithmetic overflow or out-of-bounds array/slice indexing) and (b) the `core::panic!` macro (this is the `panic!` macro in libcore itself and in `#[no_std]` context in general).

Both of these interfaces are implemented through `extern` blocks: listd/libcore, respectively, just import some function that they delegate to, and somewhere entirely else in the crate tree, that function gets implemented.
The import only gets resolved during linking; looking locally at that code, one cannot tell where the actual implementation of the respective interface lives.
No wonder that I got lost several times along the way.

In the following, both of these interfaces will come up a lot; when you get confused, the first thing to check is if you just mixed up panic *handler* and panic *runtime*.
(And remember there's also panic *hooks*, we will get to those.)
That happens to me all the time.

Moreover, `core::panic!` and `std::panic!` are *not* the same; as we will see, they take very different code paths.
libcore and libstd each implement their own way to cause panics:

* libcore's `core::panic!` does very little, it basically just delegates to the panic *handler* immediately.
* libstd's `std::panic!` (the "normal" `panic!` macro in Rust) triggers a fully-featured panic machinery that provides a user-controlled [panic *hook*](https://doc.rust-lang.org/std/panic/fn.set_hook.html).
  The default hook will print the panic message to stderr.
  After the hook is done, libstd delegates to the panic *runtime*.

    libstd also provides a panic *handler* that calls the same machinery, so `core::panic!` also ends up here.


Let us now look at these pieces in a bit more detail.

## Panic Runtime

The [interface to the panic runtime](https://github.com/rust-lang/rust/blob/7d761fe0462ba0f671a237d0bb35e3579b8ba0e8/src/libstd/panicking.rs#L48) (introduced by [this RFC](https://github.com/rust-lang/rfcs/blob/master/text/1513-less-unwinding.md)) is a function `__rust_start_panic(payload: usize) -> u32` that gets imported by libstd, and that is later resolved by the linker.

The `usize` argument here actually is a `*mut &mut dyn core::panic::BoxMeUp` -- this is where the "payload" of the panic (the information available when it gets caught) gets passed in.
`BoxMeUp` is an unstable internal implementation detail, but [looking at the trait](https://github.com/rust-lang/rust/blob/7d761fe0462ba0f671a237d0bb35e3579b8ba0e8/src/libcore/panic.rs#L268-L271) we can see that all it really does is wrap a `dyn Any + Send`, which is the [type of the panic payload](https://doc.rust-lang.org/std/thread/type.Result.html) as returned by `catch_unwind` and `thread::spawn`.
`BoxMeUp::box_me_up` returns a `Box<dyn Any + Send>`, but as a raw pointer (because `Box` is not available in the context where this trait is defined); `BoxMeUp::get` just borrows the contents.

The two implementations of this interface Rust ships with are [`libpanic_unwind`](https://github.com/rust-lang/rust/tree/7d761fe0462ba0f671a237d0bb35e3579b8ba0e8/src/libpanic_unwind) for `-C panic=unwind` (the default on most platforms) and [`libpanic_abort`](https://github.com/rust-lang/rust/tree/7d761fe0462ba0f671a237d0bb35e3579b8ba0e8/src/libpanic_abort) for `-C panic=abort`.

## `std::panic!`

On top of the panic *runtime* interface, libstd implements the default Rust panic machinery in the internal [`std::panicking`](https://github.com/rust-lang/rust/blob/7d761fe0462ba0f671a237d0bb35e3579b8ba0e8/src/libstd/panicking.rs) module.

#### `rust_panic_with_hook`

The key function that almost everything passes through is [`rust_panic_with_hook`](https://github.com/rust-lang/rust/blob/7d761fe0462ba0f671a237d0bb35e3579b8ba0e8/src/libstd/panicking.rs#L435-L437):

```rust
fn rust_panic_with_hook(
    payload: &mut dyn BoxMeUp,
    message: Option<&fmt::Arguments<'_>>,
    file_line_col: &(&str, u32, u32),
) -> !
```

This function takes a panic source location, an optional unformatted panic message (see the [`fmt::Arguments`](https://doc.rust-lang.org/std/fmt/struct.Arguments.html) docs), and a payload.

Its main job is to call whatever the current panic hook is.
Panic hooks have a [`PanicInfo`](https://doc.rust-lang.org/core/panic/struct.PanicInfo.html) argument, so we need a panic source location, format information for a panic message, and a payload.
This matches `rust_panic_with_hook`'s arguments quite nicely!
`file_line_col` and `message` can be used directly for the first two elements; `payload` gets turned into `&(dyn Any + Send)` through the `BoxMeUp` interface.

Interestingly, the *default* panic hook entirely ignores the `message`; what you actually see printed is [the `payload` downcast to `&str` or `String`](https://github.com/rust-lang/rust/blob/7d761fe0462ba0f671a237d0bb35e3579b8ba0e8/src/libstd/panicking.rs#L171-L177) (whatever works).
Supposedly, the caller should ensure that formatting `message`, if present, gives the same result.
(And the ones we discuss below do ensure this.)

Finally, `rust_panic_with_hook` dispatches to the current panic *runtime*.
At this point, only the `payload` is still relevant -- and that is important: `message` (as the `'_` lifetime indicates) may contain short-lived references, but the panic payload will be propagated up the stack and hence must be `'static`.
The `'static` bound is quite well hidden there, but after a while I realized that [`Any`](https://doc.rust-lang.org/std/any/trait.Any.html) implies `'static` (and remember `dyn BoxMeUp` is just used to obtain a `Box<dyn Any + Send>`).

#### libstd panicking entry points

`rust_panic_with_hook` is a private function to `std::panicking`; the module provides three entry points on top of this central function, and one that circumvents it:

* the [default panic handler implementation](https://github.com/rust-lang/rust/blob/7d761fe0462ba0f671a237d0bb35e3579b8ba0e8/src/libstd/panicking.rs#L301), backing (as we will see) panics from `core::panic!` and built-in panics (from arithmetic overflow or out-of-bounds array/slice indexing).
  This obtains as input a [`PanicInfo` ](https://doc.rust-lang.org/core/panic/struct.PanicInfo.html), and it has to turn that into arguments for `rust_panic_with_hook`.
  Curiously, even though the components of `PanicInfo` and the arguments of `rust_panic_with_hook` match up pretty well and seem like they could just be forwarded, that is *not* what happens.
  Instead, libstd entirely *ignores* the `payload` component of the `PanicInfo`, and sets up the actual payload (passed to `rust_panic_with_hook`) such that it contains [the formatted `message`](https://github.com/rust-lang/rust/blob/7d761fe0462ba0f671a237d0bb35e3579b8ba0e8/src/libstd/panicking.rs#L348).

    In particular, this means that the panic *runtime* is irrelevant for `no_std` applications.
  It only comes into play when libstd's panic handler implementation is used.
  (The panic *strategy* selected via `-C panic` still has an effect as it also influences code generation.
  For example, with `-C panic=abort` code can become simpler as it does not need to support unwinding.)

* [`begin_panic_fmt`](https://github.com/rust-lang/rust/blob/7d761fe0462ba0f671a237d0bb35e3579b8ba0e8/src/libstd/panicking.rs#L319), backing the [format string version of `std::panic!`](https://github.com/rust-lang/rust/blob/7d761fe0462ba0f671a237d0bb35e3579b8ba0e8/src/libstd/macros.rs#L22-L23) (i.e., this is used when you pass multiple arguments to the macro).
  This basically just packages the format string arguments into a `PanicInfo` (with a [dummy payload](https://github.com/rust-lang/rust/blob/7d761fe0462ba0f671a237d0bb35e3579b8ba0e8/src/libcore/panic.rs#L56)) and calls the default panic handler that we just discussed.

* [`begin_panic`](https://github.com/rust-lang/rust/blob/7d761fe0462ba0f671a237d0bb35e3579b8ba0e8/src/libstd/panicking.rs#L388), backing the [single-argument version of `std::panic!`](https://github.com/rust-lang/rust/blob/7d761fe0462ba0f671a237d0bb35e3579b8ba0e8/src/libstd/macros.rs#L16).
  Interestingly, this uses a very different code path than the other two entry points!
  In particular, this is the only entry point that permits passing in an *arbitrary payload*.
  That payload is just [converted into a `Box<dyn Any + Send>`](https://github.com/rust-lang/rust/blob/7d761fe0462ba0f671a237d0bb35e3579b8ba0e8/src/libstd/panicking.rs#L415) so that it can be passed to `rust_panic_with_hook`, and that's it.

    In particular, a panic hook that looks at the `message` field of the `PanicData` it is passed will *not* be able to see the message in a `std::panic!("do panic")`, but it *will* see the message in a `std::panic!("panic with data: {}", data)` as the latter passes through `begin_panic_fmt` instead.
  That seems quite surprising. (But also note that `PanicData::message()` is not stable yet.)

* [`update_count_then_panic`](https://github.com/rust-lang/rust/blob/7d761fe0462ba0f671a237d0bb35e3579b8ba0e8/src/libstd/panicking.rs#L488) is the odd one out: this entry point backs [`resume_unwind`](https://doc.rust-lang.org/nightly/std/panic/fn.resume_unwind.html), and it actually does *not* call the panic hook.
  Instead, it dispatches to the panic runtime immediately.
  Like, `begin_panic`, it lets the caller pick an arbitrary payload.
  Unlike `begin_panic`, the caller is responsible for boxing and unsizing the payload; `update_count_then_panic` just forwards that pretty much verbatim to the panic runtime.

## Panic Handler

All of the `std::panic!` machinery is really useful, but it relies on heap allocations through `Box` which is not always available.
To give libcore a way to cause panics, [panic handlers were introduced](https://github.com/rust-lang/rfcs/blob/master/text/2070-panic-implementation.md).
As we have seen, if libstd is available, it provides an implementation of that interface to wire `core::panic!` into the libstd panic machinery.

The [interface to the panic handler](https://github.com/rust-lang/rust/blob/7d761fe0462ba0f671a237d0bb35e3579b8ba0e8/src/libcore/panicking.rs#L78) is a function `fn panic(info: &core::panic::PanicInfo) -> !` that libcore imports and that is later resolved by the linker.
The [`PanicInfo` ](https://doc.rust-lang.org/core/panic/struct.PanicInfo.html) type is the same as for panic hooks: it contains a panic source location, a panic message, and a payload (a `dyn Any + Send`).
The panic message is represented as [`fmt::Arguments`](https://doc.rust-lang.org/std/fmt/struct.Arguments.html), i.e., a format string with its arguments that has not been formatted yet.

## `core::panic!`

On top of the panic handler interface, libcore provides a [minimal panic API](https://github.com/rust-lang/rust/blob/7d761fe0462ba0f671a237d0bb35e3579b8ba0e8/src/libcore/panicking.rs).
The [`core::panic!`](https://github.com/rust-lang/rust/blob/7d761fe0462ba0f671a237d0bb35e3579b8ba0e8/src/libcore/macros/mod.rs#L10-L26) macro creates a `fmt::Arguments` which is then [passed to the panic handler](https://github.com/rust-lang/rust/blob/7d761fe0462ba0f671a237d0bb35e3579b8ba0e8/src/libcore/panicking.rs#L82).
No formatting happens here as that would require heap allocations; this is why `PanicInfo` contains an "uninterpreted" format string with its arguments.

Curiously, the `payload` field of the `PanicInfo` that gets passed to the panic handler is always set to [a dummy value](https://github.com/rust-lang/rust/blob/7d761fe0462ba0f671a237d0bb35e3579b8ba0e8/src/libcore/panic.rs#L56).
This explains why the libstd panic handler ignores the payload (and instead constructs a new payload from the `message`), but that makes me wonder why that field is part of the panic handler API in the first place.
Another consequence of this is that [`core::panic!("message")`](https://github.com/rust-lang/rust/blob/7d761fe0462ba0f671a237d0bb35e3579b8ba0e8/src/libcore/macros/mod.rs#L15) and [`std::panic!("message")`](https://github.com/rust-lang/rust/blob/7d761fe0462ba0f671a237d0bb35e3579b8ba0e8/src/libstd/macros.rs#L16) (the variants without any formatting) actually result in very different panics: the former gets turned into `fmt::Arguments`, passed through the panic handler interface, and then libstd creates a `String` payload by formatting it.
The latter, however, directly uses the `&str` as a payload, and the `message` field remains `None` (as already mentioned).

Some elements of the libcore panic API are lang items because the compiler inserts calls to these functions during code generation:
* The [`panic` lang item](https://github.com/rust-lang/rust/blob/7d761fe0462ba0f671a237d0bb35e3579b8ba0e8/src/libcore/panicking.rs#L39) is called when the compiler needs to raise a panic that does not require any formatting (such as arithmetic overflow); this is the same function that also backs single-argument [`core::panic!`](https://github.com/rust-lang/rust/blob/7d761fe0462ba0f671a237d0bb35e3579b8ba0e8/src/libcore/macros/mod.rs#L15).
* The [`panic_bounds_check`](https://github.com/rust-lang/rust/blob/7d761fe0462ba0f671a237d0bb35e3579b8ba0e8/src/libcore/panicking.rs#L55) lang item is called on a failed array/slice bounds check.
  It calls into the same method as [`core::panic!` with formatting](https://github.com/rust-lang/rust/blob/7d761fe0462ba0f671a237d0bb35e3579b8ba0e8/src/libcore/macros/mod.rs#L21-L24).

## Conclusion

We have walked through 4 layers of APIs, 2 of which are indirected through imported function calls and resolved by the linker.
That's quite a journey!
But we have reached the end now.
I hope you [didn't panic](https://en.wikipedia.org/wiki/Phrases_from_The_Hitchhiker%27s_Guide_to_the_Galaxy#Don't_Panic) yourself along the way. ;)

I mentioned some things as being surprising.
Turns out they all have to do with the fact that panic hooks and panic handlers share the `PanicInfo` struct in their interface, which contains *both* an optional not-yet-formatted `message` and a type-erased `payload`:
* The panic *hook* can always find the already formatted message in the `payload`, so the `message` seems pointless for hooks.
  In fact, `message` can be missing even if `payload` contains a message (e.g., for `std::panic!("message")`).
* The panic *handler* will never actually receive a useful `payload`, so that field seems pointless for handlers.

Reading the [panic handler RFC](https://github.com/rust-lang/rfcs/blob/master/text/2070-panic-implementation.md), it seems like the plan was for `core::panic!` to also support arbitrary payloads, but so far that has not materialized.
However, even with that future extension, I think we have the invariant that when `message` is `Some`, then either `payload == &NoPayload` (so the payload is redundant) or `payload` is the formatted message (so the message is redundant).
I wonder if there is any case where *both* fields will be useful -- and if not, couldn't we encode that by making them two variants of an `enum`?
There are probably good reasons against that proposal and for the current design; it would be great to get them documented somewhere. :)

There is a lot more to say, but at this point, I invite you to follow the links to the source code that I included above.
With the high-level structure in mind, you should be able to follow that code.
If people think this overview would be worth putting somewhere more permanently, I'd be happy to work this blog post into some form of docs -- I am not sure what would be a good place for those, though.
And if you find any mistakes in what I wrote, please let me know!
