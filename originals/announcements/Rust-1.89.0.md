+++
path = "2025/08/07/Rust-1.89.0"
title = "Announcing Rust 1.89.0"
authors = ["The Rust Release Team"]
aliases = ["releases/1.89.0"]

[extra]
release = true
+++

The Rust team is happy to announce a new version of Rust, 1.89.0. Rust is a programming language empowering everyone to build reliable and efficient software.

If you have a previous version of Rust installed via `rustup`, you can get 1.89.0 with:

```console
$ rustup update stable
```

If you don't have it already, you can [get `rustup`](https://www.rust-lang.org/install.html) from the appropriate page on our website, and check out the [detailed release notes for 1.89.0](https://doc.rust-lang.org/stable/releases.html#version-1890-2025-08-07).

If you'd like to help us out by testing future releases, you might consider updating locally to use the beta channel (`rustup default beta`) or the nightly channel (`rustup default nightly`). Please [report](https://github.com/rust-lang/rust/issues/new/choose) any bugs you might come across!

## What's in 1.89.0 stable

### Explicitly inferred arguments to const generics

Rust now supports `_` as an argument to const generic parameters, inferring the value from surrounding context:

```rust
pub fn all_false<const LEN: usize>() -> [bool; LEN] {
  [false; _]
}
```

Similar to the rules for when `_` is permitted as a type, `_` is not permitted as an argument to const generics when in a signature:

```rust
// This is not allowed
pub const fn all_false<const LEN: usize>() -> [bool; _] {
  [false; LEN]
}

// Neither is this
pub const ALL_FALSE: [bool; _] = all_false::<10>();
```

### Mismatched lifetime syntaxes lint

[Lifetime elision][elision] in function signatures is an ergonomic aspect of the Rust language, but it can also be a stumbling point for newcomers and experts alike. This is especially true when lifetimes are inferred in types where it isn't syntactically obvious that a lifetime is even present:

```rust
// The returned type `std::slice::Iter` has a lifetime, 
// but there's no visual indication of that.
//
// Lifetime elision infers the lifetime of the return 
// type to be the same as that of `scores`.
fn items(scores: &[u8]) -> std::slice::Iter<u8> {
   scores.iter()
}
```

Code like this will now produce a warning by default:

```text
warning: hiding a lifetime that's elided elsewhere is confusing
 --> src/lib.rs:1:18
  |
1 | fn items(scores: &[u8]) -> std::slice::Iter<u8> {
  |                  ^^^^^     -------------------- the same lifetime is hidden here
  |                  |
  |                  the lifetime is elided here
  |
  = help: the same lifetime is referred to in inconsistent ways, making the signature confusing
  = note: `#[warn(mismatched_lifetime_syntaxes)]` on by default
help: use `'_` for type paths
  |
1 | fn items(scores: &[u8]) -> std::slice::Iter<'_, u8> {
  |                                             +++
```

We [first attempted][elided_lifetime_in_path] to improve this situation back in 2018 as part of the [`rust_2018_idioms`][2018-by-default] lint group, but [strong feedback][bevy] about the `elided_lifetimes_in_paths` lint showed that it was too blunt of a hammer as it warns about lifetimes which don't matter to understand the function:

```rust
use std::fmt;

struct Greeting;

impl fmt::Display for Greeting {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        //                -----^^^^^^^^^ expected lifetime parameter
        // Knowing that `Formatter` has a lifetime does not help the programmer
        "howdy".fmt(f)
    }
}
```

We then realized that the confusion we want to eliminate occurs when both

1. lifetime elision inference rules *connect* an input lifetime to an output lifetime
2. it's not syntactically obvious that a lifetime exists 

There are two pieces of Rust syntax that indicate that a lifetime exists: `&` and `'`, with `'` being subdivided into the inferred lifetime `'_` and named lifetimes `'a`. When a type uses a named lifetime, lifetime elision will not infer a lifetime for that type. Using these criteria, we can construct three groups:

| Self-evident it has a lifetime | Allow lifetime elision to infer a lifetime | Examples                              |
|--------------------------------|--------------------------------------------|---------------------------------------|
| No                             | Yes                                        | `ContainsLifetime`                    |
| Yes                            | Yes                                        | `&T`, `&'_ T`, `ContainsLifetime<'_>` |
| Yes                            | No                                         | `&'a T`, `ContainsLifetime<'a>`       |

The `mismatched_lifetime_syntaxes` lint checks that the inputs and outputs of a function belong to the same group. For the initial motivating example above, `&[u8]` falls into the second group while `std::slice::Iter<u8>` falls into the first group. We say that the lifetimes in the first group are *hidden*. 

Because the input and output lifetimes belong to different groups, the lint will warn about this function, reducing confusion about when a value has a meaningful lifetime that isn't visually obvious.

The `mismatched_lifetime_syntaxes` lint supersedes the `elided_named_lifetimes` lint, which did something similar for named lifetimes specifically.

Future work on the `elided_lifetimes_in_paths` lint intends to split it into more focused sub-lints with an eye to warning about a subset of them eventually.

[elision]: https://doc.rust-lang.org/1.89/book/ch10-03-lifetime-syntax.html#lifetime-elision
[elided_lifetime_in_path]: https://github.com/rust-lang/rust/pull/46254
[2018-by-default]: https://github.com/rust-lang/rust/issues/54910
[bevy]: https://github.com/rust-lang/rust/issues/131725

### More x86 target features

The `target_feature` attribute now supports the `sha512`, `sm3`, `sm4`, `kl` and `widekl` target features on x86. Additionally a number of `avx512` intrinsics and target features are also supported on x86:

```rust
#[target_feature(enable = "avx512bw")]
pub fn cool_simd_code(/* .. */) -> /* ... */ {
    /* ... */
}

```

### Cross-compiled doctests

Doctests will now be tested when running `cargo test --doc --target other_target`, this may result in some amount of breakage due to would-be-failing doctests now being tested.

Failing tests can be disabled by annotating the doctest with `ignore-<target>`:
```rust
/// ```ignore-x86_64
/// panic!("something")
/// ```
pub fn my_function() { }
``` 

### Platform Support

- [Add new Tier-3 targets `loongarch32-unknown-none` and `loongarch32-unknown-none-softfloat`](https://github.com/rust-lang/rust/pull/142053)
- [WASM's C abi is now spec compliant](https://github.com/rust-lang/rust/pull/133952)

Refer to Rust’s [platform support page][platform_support_page] for more information on Rust’s tiered platform support.

### Stabilized APIs

- [`NonZero<char>`](https://doc.rust-lang.org/stable/std/num/struct.NonZero.html)
- Many intrinsics for x86, not enumerated here
  - [AVX512 intrinsics](https://github.com/rust-lang/rust/issues/111137)
  - [`SHA512`, `SM3` and `SM4` intrinsics](https://github.com/rust-lang/rust/issues/126624)
- [`File::lock`](https://doc.rust-lang.org/stable/std/fs/struct.File.html#method.lock)
- [`File::lock_shared`](https://doc.rust-lang.org/stable/std/fs/struct.File.html#method.lock_shared)
- [`File::try_lock`](https://doc.rust-lang.org/stable/std/fs/struct.File.html#method.try_lock)
- [`File::try_lock_shared`](https://doc.rust-lang.org/stable/std/fs/struct.File.html#method.try_lock_shared)
- [`File::unlock`](https://doc.rust-lang.org/stable/std/fs/struct.File.html#method.unlock)
- [`NonNull::from_ref`](https://doc.rust-lang.org/stable/std/ptr/struct.NonNull.html#method.from_ref)
- [`NonNull::from_mut`](https://doc.rust-lang.org/stable/std/ptr/struct.NonNull.html#method.from_mut)
- [`NonNull::without_provenance`](https://doc.rust-lang.org/stable/std/ptr/struct.NonNull.html#method.without_provenance)
- [`NonNull::with_exposed_provenance`](https://doc.rust-lang.org/stable/std/ptr/struct.NonNull.html#method.with_exposed_provenance)
- [`NonNull::expose_provenance`](https://doc.rust-lang.org/stable/std/ptr/struct.NonNull.html#method.expose_provenance)
- [`OsString::leak`](https://doc.rust-lang.org/stable/std/ffi/struct.OsString.html#method.leak)
- [`PathBuf::leak`](https://doc.rust-lang.org/stable/std/path/struct.PathBuf.html#method.leak)
- [`Result::flatten`](https://doc.rust-lang.org/stable/std/result/enum.Result.html#method.flatten)
- [`std::os::linux::net::TcpStreamExt::quickack`](https://doc.rust-lang.org/stable/std/os/linux/net/trait.TcpStreamExt.html#tymethod.quickack)
- [`std::os::linux::net::TcpStreamExt::set_quickack`](https://doc.rust-lang.org/stable/std/os/linux/net/trait.TcpStreamExt.html#tymethod.set_quickack)

These previously stable APIs are now stable in const contexts:

- [`<[T; N]>::as_mut_slice`](https://doc.rust-lang.org/stable/std/primitive.array.html#method.as_mut_slice)
- [`<[u8]>::eq_ignore_ascii_case`](https://doc.rust-lang.org/stable/std/primitive.slice.html#impl-%5Bu8%5D/method.eq_ignore_ascii_case)
- [`str::eq_ignore_ascii_case`](https://doc.rust-lang.org/stable/std/primitive.str.html#impl-str/method.eq_ignore_ascii_case)

### Other changes

Check out everything that changed in [Rust](https://github.com/rust-lang/rust/releases/tag/1.89.0), [Cargo](https://doc.rust-lang.org/nightly/cargo/CHANGELOG.html#cargo-189-2025-08-07), and [Clippy](https://github.com/rust-lang/rust-clippy/blob/master/CHANGELOG.md#rust-189).

## Contributors to 1.89.0

Many people came together to create Rust 1.89.0. We couldn't have done it without all of you. [Thanks!](https://thanks.rust-lang.org/rust/1.89.0/)

[platform_support_page]: https://doc.rust-lang.org/rustc/platform-support.html