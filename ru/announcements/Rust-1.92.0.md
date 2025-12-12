Rust 1.92.0:<br>The Rust Release Team

[extra] release = true +++

Команда Rust рада сообщить о новой версии языка — 1.92.0. Rust — это язык программирования, позволяющий каждому создавать надёжное и эффективное программное обеспечение.

Если у вас есть предыдущая версия Rust, установленная через `rustup`, то для обновления до версии 1.92.0 вам достаточно выполнить команду:

```console
$ rustup update stable
```

Если у вас ещё не установлен `rustup`, вы можете установить его с [соответствующей страницы](https://www.rust-lang.org/install.html) нашего веб-сайта, а также посмотреть [подробные примечания к выпуску](https://doc.rust-lang.org/stable/releases.html#version-1920-2025-12-11) на GitHub.

Если вы хотите помочь нам протестировать будущие выпуски, вы можете использовать канал beta (`rustup default beta`) или nightly (`rustup default nightly`). Пожалуйста, [сообщайте](https://github.com/rust-lang/rust/issues/new/choose) обо всех встреченных вами ошибках.

## Что стабилизировано в 1.92.0

### Deny-by-default never type lints

The language and compiler teams continue to work on stabilization of the [never type](https://doc.rust-lang.org/stable/std/primitive.never.html). In this release the [`never_type_fallback_flowing_into_unsafe`](https://doc.rust-lang.org/beta/rustc/lints/listing/deny-by-default.html#dependency-on-unit-never-type-fallback) and [`dependency_on_unit_never_type_fallback`](https://doc.rust-lang.org/beta/rustc/lints/listing/deny-by-default.html#dependency-on-unit-never-type-fallback) future compatibility lints were made deny-by-default, meaning they will cause a compilation error when detected.

It's worth noting that while this can result in compilation errors, it is still a *lint;* these lints can all be `#[allow]`ed. These lints also will only fire when building the affected crates directly, not when they are built as dependencies (though a warning will be reported by Cargo in such cases).

Эти проверки обнаруживают код, который, вероятно, будет нарушен после стабилизации типа never. Настоятельно рекомендуется исправить их, если они обнаружены в вашем крейте.

We believe there to be approximately 500 crates affected by this lint. Despite that, we believe this to be acceptable, as lints are not a breaking change and it will allow for stabilizing the never type in the future. For more in-depth justification, see the [Language Team's assessment](https://github.com/rust-lang/rust/pull/146167#issuecomment-3363795006).

### `unused_must_use` больше не предупреждает о `Result<(), UninhabitedType>`

Rust's `unused_must_use` lint warns when ignoring the return value of a function, if the function or its return type is annotated with `#[must_use]`. For instance, this warns if ignoring a return type of `Result`, to remind you to use `?`, or something like `.expect("...")`.

However, some functions return `Result`, but the error type they use is not actually "inhabited", meaning you cannot construct any values of that type (e.g. the [`!`](https://doc.rust-lang.org/std/primitive.never.html) or [`Infallible`](https://doc.rust-lang.org/std/convert/enum.Infallible.html) types).

The `unused_must_use` lint now no longer warns on `Result<(), UninhabitedType>`, or on `ControlFlow<UninhabitedType, ()>`. For instance, it will not warn on `Result<(), Infallible>`. This avoids having to check for an error that can never happen.

```rust
use core::convert::Infallible;
fn can_never_fail() -> Result<(), Infallible> {
    // ...
    Ok(())
}

fn main() {
    can_never_fail();
}
```

Это особенно полезно при использовании трейта с ассоциированным типом ошибки, где этот тип ошибки иногда *иногда* может быть непредотвратимым (infallible):

```rust
trait UsesAssocErrorType {
    type Error;
    fn method(&self) -> Result<(), Self::Error>;
}

struct CannotFail;
impl UsesAssocErrorType for CannotFail {
    type Error = core::convert::Infallible;
    fn method(&self) -> Result<(), Self::Error> {
        Ok(())
    }
}

struct CanFail;
impl UsesAssocErrorType for CanFail {
    type Error = std::io::Error;
    fn method(&self) -> Result<(), Self::Error> {
        Err(std::io::Error::other("something went wrong"))
    }
}

fn main() {
    CannotFail.method(); // Без предупреждения
    CanFail.method(); // Предупреждение: unused `Result` that must be used
}
```

### Генерация таблицы раскрутки стека на Linux даже при включённом `-Cpanic=abort`

Backtraces with `-Cpanic=abort` previously worked in Rust 1.22 but were broken in Rust 1.23, as we stopped emitting unwind tables with `-Cpanic=abort`. In Rust 1.45 a workaround in the form of `-Cforce-unwind-tables=yes` was stabilized.

In Rust 1.92 unwind tables will be emitted by default even when `-Cpanic=abort` is specified, allowing for backtraces to work properly. If unwind tables are not desired then users should use `-Cforce-unwind-tables=no` to explicitly disable them being emitted.

### Валидация `#[macro_export]`

За последние несколько релизов было внесено множество изменений в то, как встроенные атрибуты обрабатываются в компиляторе. Это должно значительно улучшить сообщения об ошибках и предупреждениях, которые Rust выдаёт для встроенных атрибутов, и особенно сделать эту диагностику более согласованной среди более чем 100 встроенных атрибутов.

To give a small example, in this release specifically, Rust became stricter in checking what arguments are allowed to `macro_export` by [upgrading that check to a "deny-by-default lint" that will be reported in dependencies](https://github.com/rust-lang/rust/pull/143857).

### Стабилизированные API

- [`NonZero<u{N}>::div_ceil`](https://doc.rust-lang.org/stable/std/num/struct.NonZero.html#method.div_ceil)
- [`Location::file_as_c_str`](https://doc.rust-lang.org/stable/std/panic/struct.Location.html#method.file_as_c_str)
- [`RwLockWriteGuard::downgrade`](https://doc.rust-lang.org/stable/std/sync/struct.RwLockWriteGuard.html#method.downgrade)
- [`Box::new_zeroed`](https://doc.rust-lang.org/stable/std/boxed/struct.Box.html#method.new_zeroed)
- [`Box::new_zeroed_slice`](https://doc.rust-lang.org/stable/std/boxed/struct.Box.html#method.new_zeroed_slice)
- [`Rc::new_zeroed`](https://doc.rust-lang.org/stable/std/rc/struct.Rc.html#method.new_zeroed)
- [`Rc::new_zeroed_slice`](https://doc.rust-lang.org/stable/std/rc/struct.Rc.html#method.new_zeroed_slice)
- [`Arc::new_zeroed`](https://doc.rust-lang.org/stable/std/sync/struct.Arc.html#method.new_zeroed)
- [`Arc::new_zeroed_slice`](https://doc.rust-lang.org/stable/std/sync/struct.Arc.html#method.new_zeroed_slice)
- [`btree_map::Entry::insert_entry`](https://doc.rust-lang.org/stable/std/collections/btree_map/enum.Entry.html#method.insert_entry)
- [`btree_map::VacantEntry::insert_entry`](https://doc.rust-lang.org/stable/std/collections/btree_map/struct.VacantEntry.html#method.insert_entry)
- [`impl Extend<proc_macro::Group> for proc_macro::TokenStream`](https://doc.rust-lang.org/stable/proc_macro/struct.TokenStream.html#impl-Extend%3CGroup%3E-for-TokenStream)
- [`impl Extend<proc_macro::Literal> for proc_macro::TokenStream`](https://doc.rust-lang.org/stable/proc_macro/struct.TokenStream.html#impl-Extend%3CLiteral%3E-for-TokenStream)
- [`impl Extend<proc_macro::Punct> for proc_macro::TokenStream`](https://doc.rust-lang.org/stable/proc_macro/struct.TokenStream.html#impl-Extend%3CPunct%3E-for-TokenStream)
- [`impl Extend<proc_macro::Ident> for proc_macro::TokenStream`](https://doc.rust-lang.org/stable/proc_macro/struct.TokenStream.html#impl-Extend%3CIdent%3E-for-TokenStream)

Следующие API теперь можно использовать в контексте `const`:

- [`<[_]>::rotate_left`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.rotate_left)
- [`<[_]>::rotate_right`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.rotate_right)

### Прочие изменения

Проверьте всё, что изменилось в [Rust](https://github.com/rust-lang/rust/releases/tag/1.92.0), [Cargo](https://doc.rust-lang.org/nightly/cargo/CHANGELOG.html#cargo-192-2025-12-11) и [Clippy](https://github.com/rust-lang/rust-clippy/blob/master/CHANGELOG.md#rust-192).

## Кто работал над 1.92.0

Многие люди собрались вместе, чтобы создать Rust 1.92.0. Без вас мы бы не справились. [Спасибо!](https://thanks.rust-lang.org/rust/1.92.0/)
