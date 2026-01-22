Rust 1.93.0: обновление встроенного musl, глобальный аллокатор и tls, cfg в asm

[extra] release = true +++

Команда Rust рада сообщить о новой версии языка — 1.93.0. Rust — это язык программирования, позволяющий каждому создавать надёжное и эффективное программное обеспечение.

Если у вас есть предыдущая версия Rust, установленная через `rustup`, то для обновления до версии 1.93.0 вам достаточно выполнить команду:

```console
$ rustup update stable
```

Если у вас ещё не установлен `rustup`, вы можете установить его с [соответствующей страницы](https://www.rust-lang.org/install.html) нашего веб-сайта, а также посмотреть [подробные примечания к выпуску](https://doc.rust-lang.org/stable/releases.html#version-1930-2026-01-22) на GitHub.

Если вы хотите помочь нам протестировать будущие выпуски, вы можете использовать канал beta (`rustup default beta`) или nightly (`rustup default nightly`). Пожалуйста, [сообщайте](https://github.com/rust-lang/rust/issues/new/choose) обо всех встреченных вами ошибках.

## Что стабилизировано в 1.93.0

### Обновлён встроенный musl до версии 1.2.5

Различные целевые платформы `*-linux-musl` теперь [поставляются](https://github.com/rust-lang/rust/pull/142682) с musl 1.2.5. Это в первую очередь затрагивает статические сборки musl для `x86_64`, `aarch64` и `powerpc64le`, которые имели встроенный musl 1.2.3. Данное обновление включает [несколько исправлений и улучшений](https://musl.libc.org/releases.html), а также критическое изменение, которое затрагивает экосистему Rust.

Для экосистемы Rust основной причиной этого обновления является получение значительных улучшений в DNS-резолвере musl, которые появились в версии 1.2.4 и получили исправления ошибок в 1.2.5. При использовании целевых платформ `musl` для статической линковки это должно сделать переносимые бинарные файлы Linux, работающие с сетью, более надёжными — особенно при работе с большими DNS-записями и рекурсивными серверами имён.

Однако версия 1.2.4 также содержит критическое изменение: [удаление нескольких устаревших интерфейсов для обратной совместимости, которые использовал крейт libc](https://github.com/rust-lang/libc/issues/2934). Исправление для этого [было выпущено в libc 0.2.146 в июне 2023 года (2,5 года назад)](https://github.com/rust-lang/libc/pull/2935), и мы полагаем, что оно распространилось достаточно широко, чтобы мы могли внести это изменение в целевые платформы Rust.

Для получения большего количества деталей смотрите наш предыдущий [анонс](https://blog.rust-lang.org/2025/12/05/Updating-musl-1.2.5/).

### Глобальному аллокатору разрешено использовать локальное хранилище потока

Rust 1.93 корректирует внутренние механизмы стандартной библиотеки, чтобы позволить глобальным аллокаторам, написанным на Rust, использовать макрос [`thread_local!`](https://doc.rust-lang.org/stable/std/macro.thread_local.html) и функцию [`std::thread::current`](https://doc.rust-lang.org/stable/std/thread/fn.current.html) из std без риска повторного входа с системным аллокатором.

Больше деталей в [документации](https://doc.rust-lang.org/nightly/std/alloc/trait.GlobalAlloc.html#re-entrance).

### `cfg` в `asm!`

Ранее, если отдельные части блока ассемблерного кода требовали настройки через `cfg`, приходилось дублировать весь блок `asm!` целиком — в варианте с этой секцией и без неё. В версии 1.93 атрибут `cfg` можно применять к отдельным инструкциям внутри блока `asm!`.

```rust
asm!( // или global_asm!, или naked_asm!
    "nop",
    #[cfg(target_feature = "sse2")]
    "nop",
    // ...
    #[cfg(target_feature = "sse2")]
    a = const 123, // используется только в sse2
);
```

### Стабилизированные API

- [`<[MaybeUninit<T>]>::assume_init_drop`](https://doc.rust-lang.org/stable/core/primitive.slice.html#method.assume_init_drop)
- [`<[MaybeUninit<T>]>::assume_init_ref`](https://doc.rust-lang.org/stable/core/primitive.slice.html#method.assume_init_ref)
- [`<[MaybeUninit<T>]>::assume_init_mut`](https://doc.rust-lang.org/stable/core/primitive.slice.html#method.assume_init_mut)
- [`<[MaybeUninit<T>]>::write_copy_of_slice`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.write_copy_of_slice)
- [`<[MaybeUninit<T>]>::write_clone_of_slice`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.write_clone_of_slice)
- [`String::into_raw_parts`](https://doc.rust-lang.org/stable/std/string/struct.String.html#method.into_raw_parts)
- [`Vec::into_raw_parts`](https://doc.rust-lang.org/stable/std/vec/struct.Vec.html#method.into_raw_parts)
- [`<iN>::unchecked_neg`](https://doc.rust-lang.org/stable/std/primitive.isize.html#method.unchecked_neg)
- [`<iN>::unchecked_shl`](https://doc.rust-lang.org/stable/std/primitive.isize.html#method.unchecked_shl)
- [`<iN>::unchecked_shr`](https://doc.rust-lang.org/stable/std/primitive.isize.html#method.unchecked_shr)
- [`<uN>::unchecked_shl`](https://doc.rust-lang.org/stable/std/primitive.usize.html#method.unchecked_shl)
- [`<uN>::unchecked_shr`](https://doc.rust-lang.org/stable/std/primitive.usize.html#method.unchecked_shr)
- [`<[T]>::as_array`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.as_array)
- [`<[T]>::as_array_mut`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.as_mut_array)
- [`<*const [T]>::as_array`](https://doc.rust-lang.org/stable/std/primitive.pointer.html#method.as_array)
- [`<*mut [T]>::as_array_mut`](https://doc.rust-lang.org/stable/std/primitive.pointer.html#method.as_mut_array)
- [`VecDeque::pop_front_if`](https://doc.rust-lang.org/stable/std/collections/struct.VecDeque.html#method.pop_front_if)
- [`VecDeque::pop_back_if`](https://doc.rust-lang.org/stable/std/collections/struct.VecDeque.html#method.pop_back_if)
- [`Duration::from_nanos_u128`](https://doc.rust-lang.org/stable/std/time/struct.Duration.html#method.from_nanos_u128)
- [`char::MAX_LEN_UTF8`](https://doc.rust-lang.org/stable/std/primitive.char.html#associatedconstant.MAX_LEN_UTF8)
- [`char::MAX_LEN_UTF16`](https://doc.rust-lang.org/stable/std/primitive.char.html#associatedconstant.MAX_LEN_UTF16)
- [`std::fmt::from_fn`](https://doc.rust-lang.org/stable/std/fmt/fn.from_fn.html)
- [`std::fmt::FromFn`](https://doc.rust-lang.org/stable/std/fmt/struct.FromFn.html)

### Прочие изменения

Проверьте всё, что изменилось в [Rust](https://github.com/rust-lang/rust/releases/tag/1.93.0), [Cargo](https://doc.rust-lang.org/nightly/cargo/CHANGELOG.html#cargo-193-2026-01-22) и [Clippy](https://github.com/rust-lang/rust-clippy/blob/master/CHANGELOG.md#rust-193).

## Кто работал над 1.93.0

Многие люди собрались вместе, чтобы создать Rust 1.93.0. Без вас мы бы не справились. [Спасибо!](https://thanks.rust-lang.org/rust/1.93.0/)


