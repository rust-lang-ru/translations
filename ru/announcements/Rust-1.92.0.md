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

### Запрещающая по умолчанию проверка типа never

Команды по языку и компилятору продолжают работу над стабилизацией [типа never](https://doc.rust-lang.org/stable/std/primitive.never.html). В этом выпуске проверки [`never_type_fallback_flowing_into_unsafe`](https://doc.rust-lang.org/beta/rustc/lints/listing/deny-by-default.html#dependency-on-unit-never-type-fallback) и [`dependency_on_unit_never_type_fallback`](https://doc.rust-lang.org/beta/rustc/lints/listing/deny-by-default.html#dependency-on-unit-never-type-fallback) запрещены по умолчанию. Это означает, что при обнаружении они будут вызывать ошибку компиляции.

Стоит отметить: несмотря на то, что это может привести к ошибкам компиляции, это всё ещё *проверка*. Её, как и другие проверки, можно разрешить с помощью `#[allow]`. Также не стоит забывать, что эти проверки срабатывают только при непосредственной сборке затронутых крейтов, а не когда они собираются как зависимости (хотя в таких случаях Cargo выдаст предупреждение).

Эти проверки обнаруживают код, который, вероятно, будет нарушен после стабилизации типа never. Настоятельно рекомендуется исправить их, если они обнаружены в вашем крейте.

Мы полагаем, что эта проверка затрагивает примерно 500 крейтов. Мы считаем это приемлемым, поскольку проверки не являются критическим изменением, а в будущем это позволит стабилизировать тип never. Более подробное обоснование см. в оценке [Language Team](https://github.com/rust-lang/rust/pull/146167#issuecomment-3363795006).

### `unused_must_use` больше не предупреждает о `Result<(), UninhabitedType>`

Проверка `unused_must_use` предупреждает об игнорировании возвращаемого значения функции, если сама функция или возвращаемый ею тип аннотированы атрибутом `#[must_use]`. Например, она предупреждает, если игнорируется возвращаемый тип `Result`, чтобы напомнить вам использовать `?` или, к примеру, `.expect("...")`.

Однако некоторые функции возвращают `Result`, но используемый ими тип ошибки на самом деле не является inhabited. Это означает, что вы не можете создать никаких значений этого типа (например, типы [`!`](https://doc.rust-lang.org/std/primitive.never.html) или [`Infallible`](https://doc.rust-lang.org/std/convert/enum.Infallible.html).

Проверка `unused_must_use` больше не выдаёт предупреждений для `Result<(), UninhabitedType>` или `ControlFlow<UninhabitedType, ()>`. Например, она не предупредит о `Result<(), Infillable>`. Это позволяет избежать необходимости проверять ошибку, которая никогда не может произойти.

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

Ранее бэктрейсы с `-Cpanic=abort` работали в Rust 1.22, но были сломаны в Rust 1.23, когда мы прекратили генерировать таблицу раскрутки стека при `-Cpanic=abort`. В Rust 1.45 появился обходной путь в виде `-Cforce-unwind-tables=yes`.

В Rust 1.92 таблицы раскрутки стека будут генерироваться по умолчанию, даже когда указан параметр `-Cpanic=abort`, что позволит бэктрейсам работать корректно. Если таблицы раскрутки стека не требуются, пользователям следует явно отключить их генерацию, используя `-Cforce-unwind-tables=no`.

### Валидация `#[macro_export]`

За последние несколько релизов было внесено множество изменений в то, как встроенные атрибуты обрабатываются в компиляторе. Это должно значительно улучшить сообщения об ошибках и предупреждениях, которые Rust выдаёт для встроенных атрибутов, и особенно сделать эту диагностику более согласованной среди более чем 100 встроенных атрибутов.

В качестве небольшого примера: именно в этом релизе Rust стал строже проверять, какие аргументы разрешены для `macro_export` за счёт [повышения уровня этой проверки до запрещённого по умолчанию, который будет выдаваться в зависимостях](https://github.com/rust-lang/rust/pull/143857).

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


