+++
path = "2026/10/01/Rust-1.99.0"
title = "Анонс Rust 1.99.0"
authors = ["Команда выпуска Rust"]
aliases = ["releases/1.99.0"]

[extra]
release = true
+++

Команда Rust рада сообщить о новой версии языка — 1.99.0. Rust — это язык программирования, позволяющий каждому создавать надёжное и эффективное программное обеспечение.

Если у вас есть предыдущая версия Rust, установленная через `rustup`, то для обновления до версии 1.99.0 вам достаточно выполнить команду:

```console
$ rustup update stable
```

Если у вас ещё не установлен `rustup`, вы можете установить его с [соответствующей страницы](https://www.rust-lang.org/install.html) нашего веб-сайта, а также посмотреть [подробные примечания к выпуску 1.99.0](https://doc.rust-lang.org/stable/releases.html#version-1990-2026-10-01).

Если вы хотите помочь нам протестировать будущие выпуски, вы можете использовать канал beta (`rustup default beta`) или nightly (`rustup default nightly`). Пожалуйста, [сообщайте](https://github.com/rust-lang/rust/issues/new/choose) обо всех встреченных вами ошибках!

## Что стабилизировано в 1.99.0

### Функции с переменным числом аргументов в extern "C"

В Rust 1.99.0 стабилизировано определение функций с переменным числом аргументов (variadic functions) с C-ABI для ABI «C» и «C-unwind». Определённые таким образом вариативные функции используют список аргументов переменной длины (`...`) и принимают произвольное количество аргументов. Ранее Rust уже мог вызывать вариативные функции, определённые во внешнем коде (например, `libc::printf`). Начиная с Rust 1.99, такие функции можно писать непосредственно на самом Rust:

```rust
/// БЕЗОПАСНОСТЬ: функция должна вызываться как минимум с 2 аргументами типа i32.
unsafe extern "C" fn sum(mut args: ...) -> i32 {
    // БЕЗОПАСНОСТЬ: гарантируется вызывающей стороной.
    let a = unsafe { args.next_arg::<i32>() };
    let b = unsafe { args.next_arg::<i32>() };
    a + b
}

fn foo() -> i32 {
    unsafe { sum(0i32, 2i32) }
}
```

Типом для `...` является [`VaList`](https://doc.rust-lang.org/std/ffi/struct.VaList.html), который на всех целевых платформах ABI-совместим с типом `va_list` в C. Допустимые для чтения из `VaList` типы ограничиваются типажом [`VaArgSafe`](https://doc.rust-lang.org/std/ffi/trait.VaArgSafe.html).

Более подробную информацию о c-вариативных функциях см. в [Справочнике](https://doc.rust-lang.org/reference/items/functions.html#c-variadic-functions). Также в этом выпуске стабилизирована поддержка определения naked-функций с переменным числом аргументов с отличными от «C» ABI, которые должны быть написаны с помощью встроенных ассемблерных вставок.

### Информация о схеме размещения типа из «сырых» указателей

В этом выпуске определены требования безопасности для получения размера и выравнивания по «сырым» указателям как для типов `Sized` (тривиально безопасно, уже было доступно на стабильном канале), так и для типов `!Sized`.

Для этого были стабилизированы три функции:

- [`Layout::for_value_raw`](https://doc.rust-lang.org/stable/core/alloc/struct.Layout.html#method.for_value_raw)
- [`mem::size_of_val_raw`](https://doc.rust-lang.org/stable/core/mem/fn.size_of_val_raw.html)
- [`mem::align_of_val_raw`](https://doc.rust-lang.org/stable/core/mem/fn.align_of_val_raw.html)

### Предостережение против освобождения памяти после `Box::leak`

Хотя семантика языка в Rust 1.99 не изменилась, мы обновили документацию к [`Box::leak`], добавив рекомендацию отказаться от шаблонов, в которых эта память освобождается в дальнейшем. Это связано с тем, что подобный код проблемно взаимодействует с текущими и будущими потенциальными оптимизациями компилятора, а также особенно проблематичен на фоне предстоящей стабилизации пользовательских распределителей памяти. Вместо этого рекомендуется отдавать предпочтение [`Box::into_non_null`] или [`Box::into_raw`].

Эта рекомендация распространяется и на другие функции `leak` в стандартной библиотеке.

[`Box::leak`]: https://doc.rust-lang.org/std/boxed/struct.Box.html#method.leak
[`Box::into_non_null`]: https://doc.rust-lang.org/std/boxed/struct.Box.html#method.into_non_null
[`Box::into_raw`]: https://doc.rust-lang.org/std/boxed/struct.Box.html#method.into_raw

### Стабилизированные API

- [`IntoIterator` для `Box<[T; N]>`](https://doc.rust-lang.org/stable/std/iter/trait.IntoIterator.html#impl-IntoIterator-for-Box%3C%5BT;+N%5D,+A%3E)
- [`IntoIterator` для `&Box<[T; N]>`](https://doc.rust-lang.org/stable/std/iter/trait.IntoIterator.html#impl-IntoIterator-for-%26Box%3C%5BT;+N%5D,+A%3E)
- [`IntoIterator` для `&mut Box<[T; N]>`](https://doc.rust-lang.org/stable/std/iter/trait.IntoIterator.html#impl-IntoIterator-for-%26mut+Box%3C%5BT;+N%5D,+A%3E)
- [`VecDeque::retain_back`](https://doc.rust-lang.org/stable/std/collections/struct.VecDeque.html#method.retain_back)
- [`core::ffi::VaList`](https://doc.rust-lang.org/stable/core/ffi/struct.VaList.html)
- [`Box::into_non_null`](https://doc.rust-lang.org/stable/std/boxed/struct.Box.html#method.into_non_null)
- [`Box::from_non_null`](https://doc.rust-lang.org/stable/std/boxed/struct.Box.html#method.from_non_null)
- [`Vec::into_parts`](https://doc.rust-lang.org/stable/std/vec/struct.Vec.html#method.into_parts)
- [`Vec::from_parts`](https://doc.rust-lang.org/stable/std/vec/struct.Vec.html#method.from_parts)
- [`core::mem::size_of_val_raw`](https://doc.rust-lang.org/stable/core/mem/fn.size_of_val_raw.html)
- [`core::mem::align_of_val_raw`](https://doc.rust-lang.org/stable/core/mem/fn.align_of_val_raw.html)
- [`core::alloc::Layout::for_value_raw`](https://doc.rust-lang.org/stable/core/alloc/struct.Layout.html#method.for_value_raw)
- [`String::from_utf8_lossy_owned`](https://doc.rust-lang.org/stable/std/string/struct.String.html#method.from_utf8_lossy_owned)
- [`string::FromUtf8Error::into_utf8_lossy`](https://doc.rust-lang.org/stable/std/string/struct.FromUtf8Error.html#method.into_utf8_lossy)
- [`FusedIterator` для `StepBy<I>`](https://doc.rust-lang.org/stable/std/iter/struct.StepBy.html#impl-FusedIterator-for-StepBy%3CI%3E)
- [`std::fs::set_times`](https://doc.rust-lang.org/stable/std/fs/fn.set_times.html)
- [`std::fs::set_times_nofollow`](https://doc.rust-lang.org/stable/std/fs/fn.set_times_nofollow.html)

### Прочие изменения

Проверьте всё, что изменилось в [Rust](https://github.com/rust-lang/rust/releases/tag/1.99.0), [Cargo](https://doc.rust-lang.org/nightly/cargo/CHANGELOG.html#cargo-199-2026-10-01) и [Clippy](https://github.com/rust-lang/rust-clippy/blob/master/CHANGELOG.md#rust-199).

## Кто работал над 1.99.0

Многие люди собрались вместе, чтобы создать Rust 1.99.0. Без вас мы бы не справились. [Спасибо!](https://thanks.rust-lang.org/rust/1.99.0/)

