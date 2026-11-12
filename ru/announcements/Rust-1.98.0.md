+++
path = "2026/08/20/Rust-1.98.0"
title = "Анонс Rust 1.98.0"
authors = ["The Rust Release Team"]
aliases = ["releases/1.98.0"]

[extra]
release = true
+++

Команда Rust рада объявить о выходе новой версии Rust — 1.98.0. Rust — это язык программирования, который помогает каждому создавать надёжное и эффективное программное обеспечение.

Если у вас уже установлена предыдущая версия Rust через `rustup`, вы можете получить 1.98.0 командой:

```console
$ rustup update stable
```

Если Rust ещё не установлен, вы можете получить [`rustup`](https://www.rust-lang.org/install.html) на соответствующей странице нашего сайта и ознакомиться с [подробными release notes для 1.98.0](https://doc.rust-lang.org/stable/releases.html#version-1980-2026-08-20).

Если вы хотите помочь нам, тестируя будущие релизы, рассмотрите возможность переключиться локально на beta-канал (`rustup default beta`) или nightly-канал (`rustup default nightly`). Пожалуйста, [сообщайте](https://github.com/rust-lang/rust/issues/new/choose) о любых обнаруженных ошибках!

## Что нового в stable 1.98.0

### Алгебраические методы для чисел с плавающей точкой

Типы с плавающей точкой `f32` и `f64` теперь имеют «алгебраические» методы для сложения, вычитания, умножения, деления и взятия остатка. Они позволяют оптимизировать эти операции, используя алгебраические свойства действительных чисел, хотя эти свойства не выполняются с учётом ограничений представления чисел с плавающей точкой. Точный набор оптимизаций не специфицирован, но может быть похож на оптимизации, которые вы видите с опцией `-ffast-math` в других языках.

Например, сложение чисел с плавающей точкой [не ассоциативно](https://en.wikipedia.org/wiki/Associative_property#Nonassociativity_of_floating-point_calculation), поэтому сумму вида `a + b + c + d` необходимо вычислять в левоассоциативном порядке, в котором она разобрана парсером, то есть `((a + b) + c) + d`. Если записать ту же сумму как цепочку вызовов `algebraic_add`, компилятор может свободно менять порядок вычислений, например `(a + b) + (c + d)`, чтобы вычислять частичные суммы одновременно. Использование этих алгебраических методов также часто включает более широкую векторизацию циклов.

Эти методы недетерминированы, поскольку компилятор может выбирать разные оптимизации, но они никогда не приводят к неопределённому поведению. Подробнее см. в [документации библиотеки](https://doc.rust-lang.org/stable/core/primitive.f32.html#algebraic-operators) и в исходном [предложении об изменении API](https://github.com/rust-lang/libs-team/issues/532).

### Буферизованное форматирование целых чисел

Все примитивные целочисленные типы теперь имеют метод [`format_into`](https://doc.rust-lang.org/stable/core/primitive.usize.html#method.format_into), принимающий параметр `&mut NumBuffer<Self>` — буфер, достаточно большой для хранения десятичного представления любого значения этого типа. Сам буфер непрозрачен, но метод возвращает отформатированную строку `&str` с временем жизни, заимствованным из этого буфера.

Этот метод также обходит большую часть динамической диспетчеризации, которую вы получили бы при буферизованном форматировании через `write!`, что может существенно повысить производительность. Репозиторий [`itoa-benchmark`](https://github.com/dtolnay/itoa-benchmark) теперь показывает, что `format_into` работает сопоставимо с самим `itoa`, так что этот метод может служить стандартной заменой этой зависимости и подобных ей.

### Исправление взаимодействия между `ManuallyDrop` и `Box`

До Rust 1.96.0 в компиляторе Rust была ошибка, из-за которой следующий код приводил к неопределённому поведению:

```rust
let mut x = ManuallyDrop::new(Box::new(1));
unsafe { ManuallyDrop::drop(&mut x) };
let x = x; // UB!
```

Это происходит потому, что компилятор считает неопределённым поведением перемещение `Box`, который уже был освобождён (deallocated), и `ManuallyDrop` ранее распространял это свойство, так что перемещение `ManuallyDrop<Box<_>>`, где box уже освобождён, также считалось UB.

В Rust 1.96.0 мы исправили это, и этот код больше не является UB. В этом релизе мы обновили документацию `ManuallyDrop`, дав стабильную гарантию, что этот код и в будущем не будет UB. Подробнее см. в [документации `ManuallyDrop`](https://doc.rust-lang.org/stable/std/mem/struct.ManuallyDrop.html#pre-196-interaction-with-box) и в связанном [RFC 3336](https://rust-lang.github.io/rfcs/3336-maybe-dangling.html).

### Стабилизированные API

- [`str::substr_range`](https://doc.rust-lang.org/stable/std/primitive.str.html#method.substr_range)
- [`[T]::subslice_range`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.subslice_range)
- [`core::fmt::NumBuffer`](https://doc.rust-lang.org/stable/core/fmt/struct.NumBuffer.html)
- [`<{integer}>::format_into`](https://doc.rust-lang.org/stable/core/primitive.usize.html#method.format_into)
- [`Send/Sync for std::process::CommandArgs`](https://doc.rust-lang.org/stable/std/process/struct.CommandArgs.html#impl-Send-for-CommandArgs%3C'a%3E)
- [`{fN}::algebraic_add`](https://doc.rust-lang.org/stable/core/primitive.f32.html#method.algebraic_add)
- [`{fN}::algebraic_sub`](https://doc.rust-lang.org/stable/core/primitive.f32.html#method.algebraic_sub)
- [`{fN}::algebraic_mul`](https://doc.rust-lang.org/stable/core/primitive.f32.html#method.algebraic_mul)
- [`{fN}::algebraic_div`](https://doc.rust-lang.org/stable/core/primitive.f32.html#method.algebraic_div)
- [`{fN}::algebraic_rem`](https://doc.rust-lang.org/stable/core/primitive.f32.html#method.algebraic_rem)
- [`NonZero<{integer}>::from_str_radix`](https://doc.rust-lang.org/stable/core/num/struct.NonZero.html#method.from_str_radix-4)
- [`String::from_utf16le`](https://doc.rust-lang.org/stable/std/string/struct.String.html#method.from_utf16le)
- [`String::from_utf16le_lossy`](https://doc.rust-lang.org/stable/std/string/struct.String.html#method.from_utf16le_lossy)
- [`String::from_utf16be`](https://doc.rust-lang.org/stable/std/string/struct.String.html#method.from_utf16be)
- [`String::from_utf16be_lossy`](https://doc.rust-lang.org/stable/std/string/struct.String.html#method.from_utf16be_lossy)
- [`[T]::strip_circumfix`](https://doc.rust-lang.org/stable/core/primitive.slice.html#method.strip_circumfix)
- [`str::strip_circumfix`](https://doc.rust-lang.org/stable/core/primitive.str.html#method.strip_circumfix)
- [`Atomic<T>::from_mut`](https://doc.rust-lang.org/stable/core/sync/atomic/struct.Atomic.html#method.from_mut)
- [`Atomic<T>::get_mut_slice`](https://doc.rust-lang.org/stable/core/sync/atomic/struct.Atomic.html#method.get_mut_slice)
- [`Atomic<T>::from_mut_slice`](https://doc.rust-lang.org/stable/core/sync/atomic/struct.Atomic.html#method.from_mut_slice)
- [`std::range::legacy`](https://doc.rust-lang.org/stable/std/range/legacy/index.html)

### Прочие изменения

Ознакомьтесь со всем, что изменилось в [Rust](https://github.com/rust-lang/rust/releases/tag/1.98.0), [Cargo](https://doc.rust-lang.org/nightly/cargo/CHANGELOG.html#cargo-198-2026-08-20) и [Clippy](https://github.com/rust-lang/rust-clippy/blob/master/CHANGELOG.md#rust-198).

## Участники релиза 1.98.0

Многие люди объединили усилия, чтобы создать Rust 1.98.0. Без вас мы бы не справились. [Спасибо!](https://thanks.rust-lang.org/rust/1.98.0/)
