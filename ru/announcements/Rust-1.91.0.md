Rust 1.91.0: aarch64-pc-windows-msvc на Tier 1, отлавливание сырых указателей

[extra] release = true +++

Команда Rust рада сообщить о новой версии языка — 1.91.0. Rust — это язык программирования, позволяющий каждому создавать надёжное и эффективное программное обеспечение.

Если у вас есть предыдущая версия Rust, установленная через `rustup`, то для обновления до версии 1.91.0 вам достаточно выполнить команду:

```console
$ rustup update stable
```

Если у вас ещё не установлен `rustup`, вы можете установить его с [соответствующей страницы](https://www.rust-lang.org/install.html) нашего веб-сайта, а также посмотреть [подробные примечания к выпуску](https://doc.rust-lang.org/stable/releases.html#version-1910-2025-10-30) на GitHub.

Если вы хотите помочь нам протестировать будущие выпуски, вы можете использовать канал beta (`rustup default beta`) или nightly (`rustup default nightly`). Пожалуйста, [сообщайте](https://github.com/rust-lang/rust/issues/new/choose) обо всех встреченных вами ошибках.

## Что стабилизировано в 1.91.0

### `aarch64-pc-windows-msvc` теперь на 1 уровне поддержки

Компилятор Rust поддерживает [широкий спектр целевых платформ], но команда Rust не может обеспечить одинаковый уровень поддержки для каждой из них. Чтобы чётко обозначить, насколько поддерживается каждая платформа, мы используем уровни поддержки:

- Целевые платформы 3 уровня поддержки технически поддерживаются компилятором, однако мы не проверяем, собирается ли их код и проходят ли тесты, а также не предоставляем никаких предварительно собранных двоичных файлов в рамках наших релизов.
- Целевые платформы 2 уровня поддержки гарантированно проходят сборку, и мы предоставляем для них предварительно собранные двоичные файлы, но мы не запускаем набор тестов на этих платформах: произведённые двоичные файлы могут не работать или содержать ошибки.
- Целевые платформы 1 уровня поддержки предоставляют самую высокую гарантию поддержки, и мы запускаем на этих платформах полный набор тестов для каждого изменения, внесённого в компилятор. Предварительно собранные двоичные файлы также доступны.

В Rust версии 1.91.0 целевая платформа `aarch64-pc-windows-msvc` получает 1 уровень поддержки, что обеспечивает наши самые высокие гарантии пользователям 64-битных ARM-систем, работающих под управлением Windows.

### Добавлена проверка, направленная против висящих "сырых" указателей, созданных из локальных переменных

В то время как анализатор заимствований Rust предотвращает возврат висящих ссылок, он не отслеживает "сырые" указатели. С этим релизом  мы добавляем проверку, которая по умолчанию выдаёт предупреждение о "сырых" указателях на локальные переменные, возвращаемых из функций. Например, такой код:

```rust
fn f() -> *const u8 {
    let x = 0;
    &x
}
```

теперь вызовет предупреждение статического анализатора:

```
warning: a dangling pointer will be produced because the local variable `x` will be dropped
 --> src/lib.rs:3:5
  |
1 | fn f() -> *const u8 {
  |           --------- return type of the function is `*const u8`
2 |     let x = 0;
  |         - `x` is part the function and will be dropped at the end of the function
3 |     &x
  |     ^^
  |
  = note: pointers do not have a lifetime; after returning, the `u8` will be deallocated
    at the end of the function because nothing is referencing it as far as the type system is
    concerned
  = note: `#[warn(dangling_pointers_from_locals)]` on by default
```

Обратите внимание, что приведённый выше код не является небезопасным, поскольку сам по себе он не выполняет никаких опасных операций. Небезопасным будет только разыменование сырого указателя после возврата из функции. Мы ожидаем, что будущие релизы Rust добавят больше функциональности, которая поможет разработчикам безопасно взаимодействовать с сырыми указателями и с небезопасным кодом в целом.

### Стабилизированные API

- [`Path::file_prefix`](https://doc.rust-lang.org/stable/std/path/struct.Path.html#method.file_prefix)
- [`AtomicPtr::fetch_ptr_add`](https://doc.rust-lang.org/stable/std/sync/atomic/struct.AtomicPtr.html#method.fetch_ptr_add)
- [`AtomicPtr::fetch_ptr_sub`](https://doc.rust-lang.org/stable/std/sync/atomic/struct.AtomicPtr.html#method.fetch_ptr_sub)
- [`AtomicPtr::fetch_byte_add`](https://doc.rust-lang.org/stable/std/sync/atomic/struct.AtomicPtr.html#method.fetch_byte_add)
- [`AtomicPtr::fetch_byte_sub`](https://doc.rust-lang.org/stable/std/sync/atomic/struct.AtomicPtr.html#method.fetch_byte_sub)
- [`AtomicPtr::fetch_or`](https://doc.rust-lang.org/stable/std/sync/atomic/struct.AtomicPtr.html#method.fetch_or)
- [`AtomicPtr::fetch_and`](https://doc.rust-lang.org/stable/std/sync/atomic/struct.AtomicPtr.html#method.fetch_and)
- [`AtomicPtr::fetch_xor`](https://doc.rust-lang.org/stable/std/sync/atomic/struct.AtomicPtr.html#method.fetch_xor)
- [`{integer}::strict_add`](https://doc.rust-lang.org/stable/std/primitive.u32.html#method.strict_add)
- [`{integer}::strict_sub`](https://doc.rust-lang.org/stable/std/primitive.u32.html#method.strict_sub)
- [`{integer}::strict_mul`](https://doc.rust-lang.org/stable/std/primitive.u32.html#method.strict_mul)
- [`{integer}::strict_div`](https://doc.rust-lang.org/stable/std/primitive.u32.html#method.strict_div)
- [`{integer}::strict_div_euclid`](https://doc.rust-lang.org/stable/std/primitive.u32.html#method.strict_div_euclid)
- [`{integer}::strict_rem`](https://doc.rust-lang.org/stable/std/primitive.u32.html#method.strict_rem)
- [`{integer}::strict_rem_euclid`](https://doc.rust-lang.org/stable/std/primitive.u32.html#method.strict_rem_euclid)
- [`{integer}::strict_neg`](https://doc.rust-lang.org/stable/std/primitive.u32.html#method.strict_neg)
- [`{integer}::strict_shl`](https://doc.rust-lang.org/stable/std/primitive.u32.html#method.strict_shl)
- [`{integer}::strict_shr`](https://doc.rust-lang.org/stable/std/primitive.u32.html#method.strict_shr)
- [`{integer}::strict_pow`](https://doc.rust-lang.org/stable/std/primitive.u32.html#method.strict_pow)
- [`i{N}::strict_add_unsigned`](https://doc.rust-lang.org/stable/std/primitive.i32.html#method.strict_add_unsigned)
- [`i{N}::strict_sub_unsigned`](https://doc.rust-lang.org/stable/std/primitive.i32.html#method.strict_sub_unsigned)
- [`i{N}::strict_abs`](https://doc.rust-lang.org/stable/std/primitive.i32.html#method.strict_abs)
- [`u{N}::strict_add_signed`](https://doc.rust-lang.org/stable/std/primitive.u32.html#method.strict_add_signed)
- [`u{N}::strict_sub_signed`](https://doc.rust-lang.org/stable/std/primitive.u32.html#method.strict_sub_signed)
- [`PanicHookInfo::payload_as_str`](https://doc.rust-lang.org/stable/std/panic/struct.PanicHookInfo.html#method.payload_as_str)
- [`core::iter::chain`](https://doc.rust-lang.org/stable/core/iter/fn.chain.html)
- [`u{N}::checked_signed_diff`](https://doc.rust-lang.org/stable/std/primitive.u16.html#method.checked_signed_diff)
- [`core::array::repeat`](https://doc.rust-lang.org/stable/core/array/fn.repeat.html)
- [`PathBuf::add_extension`](https://doc.rust-lang.org/stable/std/path/struct.PathBuf.html#method.add_extension)
- [`PathBuf::with_added_extension`](https://doc.rust-lang.org/stable/std/path/struct.PathBuf.html#method.with_added_extension)
- [`Duration::from_mins`](https://doc.rust-lang.org/stable/std/time/struct.Duration.html#method.from_mins)
- [`Duration::from_hours`](https://doc.rust-lang.org/stable/std/time/struct.Duration.html#method.from_hours)
- [`impl PartialEq<str> for PathBuf`](https://doc.rust-lang.org/stable/std/path/struct.PathBuf.html#impl-PartialEq%3Cstr%3E-for-PathBuf)
- [`impl PartialEq<String> for PathBuf`](https://doc.rust-lang.org/stable/std/path/struct.PathBuf.html#impl-PartialEq%3CString%3E-for-PathBuf)
- [`impl PartialEq<str> for Path`](https://doc.rust-lang.org/stable/std/path/struct.Path.html#impl-PartialEq%3Cstr%3E-for-Path)
- [`impl PartialEq<String> for Path`](https://doc.rust-lang.org/stable/std/path/struct.Path.html#impl-PartialEq%3CString%3E-for-Path)
- [`impl PartialEq<PathBuf> for String`](https://doc.rust-lang.org/nightly/std/string/struct.String.html#impl-PartialEq%3CPathBuf%3E-for-String)
- [`impl PartialEq<Path> for String`](https://doc.rust-lang.org/nightly/std/string/struct.String.html#impl-PartialEq%3CPath%3E-for-String)
- [`impl PartialEq<PathBuf> for str`](https://doc.rust-lang.org/nightly/std/primitive.str.html#impl-PartialEq%3CPathBuf%3E-for-str)
- [`impl PartialEq<Path> for str`](https://doc.rust-lang.org/nightly/std/primitive.str.html#impl-PartialEq%3CPath%3E-for-str)
- [`Ipv4Addr::from_octets`](https://doc.rust-lang.org/stable/std/net/struct.Ipv4Addr.html#method.from_octets)
- [`Ipv6Addr::from_octets`](https://doc.rust-lang.org/stable/std/net/struct.Ipv6Addr.html#method.from_octets)
- [`Ipv6Addr::from_segments`](https://doc.rust-lang.org/stable/std/net/struct.Ipv6Addr.html#method.from_segments)
- [`impl<T> Default for Pin<Box<T>> where Box<T>: Default, T: ?Sized`](https://doc.rust-lang.org/stable/std/default/trait.Default.html#impl-Default-for-Pin%3CBox%3CT%3E%3E)
- [`impl<T> Default for Pin<Rc<T>> where Rc<T>: Default, T: ?Sized`](https://doc.rust-lang.org/stable/std/default/trait.Default.html#impl-Default-for-Pin%3CRc%3CT%3E%3E)
- [`impl<T> Default for Pin<Arc<T>> where Arc<T>: Default, T: ?Sized`](https://doc.rust-lang.org/stable/std/default/trait.Default.html#impl-Default-for-Pin%3CArc%3CT%3E%3E)
- [`Cell::as_array_of_cells`](https://doc.rust-lang.org/stable/std/cell/struct.Cell.html#method.as_array_of_cells)
- [`u{N}::carrying_add`](https://doc.rust-lang.org/stable/std/primitive.u64.html#method.carrying_add)
- [`u{N}::borrowing_sub`](https://doc.rust-lang.org/stable/std/primitive.u64.html#method.borrowing_sub)
- [`u{N}::carrying_mul`](https://doc.rust-lang.org/stable/std/primitive.u64.html#method.carrying_mul)
- [`u{N}::carrying_mul_add`](https://doc.rust-lang.org/stable/std/primitive.u64.html#method.carrying_mul_add)
- [`BTreeMap::extract_if`](https://doc.rust-lang.org/stable/std/collections/struct.BTreeMap.html#method.extract_if)
- [`BTreeSet::extract_if`](https://doc.rust-lang.org/stable/std/collections/struct.BTreeSet.html#method.extract_if)
- [`impl Debug for windows::ffi::EncodeWide<'_>`](https://doc.rust-lang.org/stable/std/os/windows/ffi/struct.EncodeWide.html#impl-Debug-for-EncodeWide%3C'_%3E)
- [`str::ceil_char_boundary`](https://doc.rust-lang.org/stable/std/primitive.str.html#method.ceil_char_boundary)
- [`str::floor_char_boundary`](https://doc.rust-lang.org/stable/std/primitive.str.html#method.floor_char_boundary)
- [`impl Sum for Saturating<u{N}>`](https://doc.rust-lang.org/stable/std/num/struct.Saturating.html#impl-Sum-for-Saturating%3Cu32%3E)
- [`impl Sum<&u{N}> for Saturating<u{N}>`](https://doc.rust-lang.org/stable/std/num/struct.Saturating.html#impl-Sum%3C%26Saturating%3Cu32%3E%3E-for-Saturating%3Cu32%3E)
- [`impl Product for Saturating<u{N}>`](https://doc.rust-lang.org/stable/std/num/struct.Saturating.html#impl-Product-for-Saturating%3Cu32%3E)
- [`impl Product<&u{N}> for Saturating<u{N}>`](https://doc.rust-lang.org/stable/std/num/struct.Saturating.html#impl-Product%3C%26Saturating%3Cu32%3E%3E-for-Saturating%3Cu32%3E)

Следующие API теперь можно использовать в контексте `const`:

- [`<[T; N]>::each_ref`](https://doc.rust-lang.org/stable/std/primitive.array.html#method.each_ref)
- [`<[T; N]>::each_mut`](https://doc.rust-lang.org/stable/std/primitive.array.html#method.each_mut)
- [`OsString::new`](https://doc.rust-lang.org/stable/std/ffi/struct.OsString.html#method.new)
- [`PathBuf::new`](https://doc.rust-lang.org/stable/std/path/struct.PathBuf.html#method.new)
- [`TypeId::of`](https://doc.rust-lang.org/stable/std/any/struct.TypeId.html#method.of)
- [`ptr::with_exposed_provenance`](https://doc.rust-lang.org/stable/std/ptr/fn.with_exposed_provenance.html)
- [`ptr::with_exposed_provenance_mut`](https://doc.rust-lang.org/stable/std/ptr/fn.with_exposed_provenance_mut.html)

### Поддержка платформ

- [Повышение поддержки `aarch64-pc-windows-msvc` до 1 уровня](https://github.com/rust-lang/rust/pull/145682)
- [Повышение поддержки `aarch64-pc-windows-gnullvm` и `x86_64-pc-windows-gnullvm` до 2 уровня с хост-инструментарием.](https://github.com/rust-lang/rust/pull/143031) Примечание: llvm-tools и установщики MSI отсутствуют, но будут добавлены в будущих релизах.

Дополнительную информацию о уровнях поддержки платформ в Rust можно найти на [странице поддержки](https://doc.rust-lang.org/rustc/platform-support.html).

### Прочие изменения

Проверьте всё, что изменилось в [Rust](https://github.com/rust-lang/rust/releases/tag/1.91.0), [Cargo](https://doc.rust-lang.org/nightly/cargo/CHANGELOG.html#cargo-191-2025-10-30) и [Clippy](https://github.com/rust-lang/rust-clippy/blob/master/CHANGELOG.md#rust-191).

## Кто работал над 1.91.0

Многие люди собрались вместе, чтобы создать Rust 1.91.0. Без вас мы бы не справились. [Спасибо!](https://thanks.rust-lang.org/rust/1.91.0/)


[широкий спектр целевых платформ]: https://doc.rust-lang.org/rustc/platform-support.html