
+++
path = "2026/04/16/Rust-1.95.0"
title = "Анонс Rust 1.95.0"
authors = ["Команда выпуска Rust"]
aliases = ["releases/1.95.0"]

[extra]
release = true
+++

Команда Rust рада объявить о новом выпуске Rust 1.95.0. Rust — это язык программирования, который дает каждому возможность создавать надежное и эффективное программное обеспечение.

Если у вас уже установлена предыдущая версия Rust через `rustup`, вы можете получить 1.95.0 командой:

```console
$ rustup update stable
```

Если у вас еще не установлен Rust, вы можете [получить `rustup`](https://www.rust-lang.org/install.html) на соответствующей странице нашего сайта, а также ознакомиться с [подробными примечаниями к выпуску 1.95.0](https://doc.rust-lang.org/stable/releases.html#version-1950-2026-04-16).

Если вы хотите помочь нам, тестируя будущие релизы, попробуйте локально переключиться на beta-канал (`rustup default beta`) или nightly-канал (`rustup default nightly`). Пожалуйста, [сообщайте](https://github.com/rust-lang/rust/issues/new/choose) о любых найденных ошибках!

## Что вошло в стабильный 1.95.0

### `cfg_select!`

В Rust 1.95 появилась
макрокоманда [`cfg_select!`](https://doc.rust-lang.org/stable/std/macro.cfg_select.html),
которая работает примерно как `match` на этапе компиляции по `cfg`. Она
решает ту же задачу, что и популярный крейт
[`cfg-if`](https://crates.io/crates/cfg-if), хотя и с другим синтаксисом.
`cfg_select!` разворачивается в правую часть первой ветки, чей
предикат конфигурации вычисляется в `true`. Несколько примеров:

```rust
cfg_select! {
    unix => {
        fn foo() { /* функциональность только для unix */ }
    }
    target_pointer_width = "32" => {
        fn foo() { /* функциональность для не-unix, 32-бит */ }
    }
    _ => {
        fn foo() { /* резервная реализация */ }
    }
}

let is_windows_str = cfg_select! {
    windows => "windows",
    _ => "not windows",
};
```

### if-let guards в match-выражениях

В Rust 1.88 были стабилизированы [цепочки let](https://blog.rust-lang.org/2025/06/26/Rust-1.88.0/#let-chains). В Rust
1.95 эта возможность пришла и в `match`-выражения, позволяя задавать условия
на основе сопоставления с образцом.

```rust
match value {
    Some(x) if let Ok(y) = compute(x) => {
        // И `x`, и `y` доступны здесь
        println!("{}, {}", x, y);
    }
    _ => {}
}
```

Обратите внимание: сейчас компилятор не учитывает образцы, сопоставленные в `if
let`-guards, как часть проверки полноты покрытия (`exhaustiveness`) всего `match`,
так же как и в случае с обычными `if`-guards.

### Стабилизированные API

- [`MaybeUninit<[T; N]>: From<[MaybeUninit<T>; N]>`](https://doc.rust-lang.org/stable/std/mem/union.MaybeUninit.html#impl-From%3CMaybeUninit%3C%5BT;+N%5D%3E%3E-for-%5BMaybeUninit%3CT%3E;+N%5D)
- [`MaybeUninit<[T; N]>: AsRef<[MaybeUninit<T>; N]>`](https://doc.rust-lang.org/stable/std/mem/union.MaybeUninit.html#impl-AsRef%3C%5BMaybeUninit%3CT%3E;+N%5D%3E-for-MaybeUninit%3C%5BT;+N%5D%3E)
- [`MaybeUninit<[T; N]>: AsRef<[MaybeUninit<T>]>`](https://doc.rust-lang.org/stable/std/mem/union.MaybeUninit.html#impl-AsRef%3C%5BMaybeUninit%3CT%3E%5D%3E-for-MaybeUninit%3C%5BT;+N%5D%3E)
- [`MaybeUninit<[T; N]>: AsMut<[MaybeUninit<T>; N]>`](https://doc.rust-lang.org/stable/std/mem/union.MaybeUninit.html#impl-AsMut%3C%5BMaybeUninit%3CT%3E;+N%5D%3E-for-MaybeUninit%3C%5BT;+N%5D%3E)
- [`MaybeUninit<[T; N]>: AsMut<[MaybeUninit<T>]>`](https://doc.rust-lang.org/stable/std/mem/union.MaybeUninit.html#impl-AsMut%3C%5BMaybeUninit%3CT%3E%5D%3E-for-MaybeUninit%3C%5BT;+N%5D%3E)
- [`[MaybeUninit<T>; N]: From<MaybeUninit<[T; N]>>`](https://doc.rust-lang.org/stable/std/mem/union.MaybeUninit.html#impl-From%3C%5BMaybeUninit%3CT%3E;+N%5D%3E-for-MaybeUninit%3C%5BT;+N%5D%3E)
- [`Cell<[T; N]>: AsRef<[Cell<T>; N]>`](https://doc.rust-lang.org/stable/std/cell/struct.Cell.html#impl-AsRef%3C%5BCell%3CT%3E;+N%5D%3E-for-Cell%3C%5BT;+N%5D%3E)
- [`Cell<[T; N]>: AsRef<[Cell<T>]>`](https://doc.rust-lang.org/stable/std/cell/struct.Cell.html#impl-AsRef%3C%5BCell%3CT%3E%5D%3E-for-Cell%3C%5BT;+N%5D%3E)
- [`Cell<[T]>: AsRef<[Cell<T>]>`](https://doc.rust-lang.org/stable/std/cell/struct.Cell.html#impl-AsRef%3C%5BCell%3CT%3E%5D%3E-for-Cell%3C%5BT%5D%3E)
- [`bool: TryFrom<{integer}>`](https://doc.rust-lang.org/stable/std/primitive.bool.html#impl-TryFrom%3Cu128%3E-for-bool)
- [`AtomicPtr::update`](https://doc.rust-lang.org/stable/std/sync/atomic/struct.AtomicPtr.html#method.update)
- [`AtomicPtr::try_update`](https://doc.rust-lang.org/stable/std/sync/atomic/struct.AtomicPtr.html#method.try_update)
- [`AtomicBool::update`](https://doc.rust-lang.org/stable/std/sync/atomic/struct.AtomicBool.html#method.update)
- [`AtomicBool::try_update`](https://doc.rust-lang.org/stable/std/sync/atomic/struct.AtomicBool.html#method.try_update)
- [`AtomicIn::update`](https://doc.rust-lang.org/stable/std/sync/atomic/struct.AtomicIsize.html#method.update)
- [`AtomicIn::try_update`](https://doc.rust-lang.org/stable/std/sync/atomic/struct.AtomicIsize.html#method.try_update)
- [`AtomicUn::update`](https://doc.rust-lang.org/stable/std/sync/atomic/struct.AtomicUsize.html#method.update)
- [`AtomicUn::try_update`](https://doc.rust-lang.org/stable/std/sync/atomic/struct.AtomicUsize.html#method.try_update)
- [`cfg_select!`](https://doc.rust-lang.org/stable/std/macro.cfg_select.html)
- [`mod core::range`](https://doc.rust-lang.org/stable/core/range/index.html)
- [`core::range::RangeInclusive`](https://doc.rust-lang.org/stable/core/range/struct.RangeInclusive.html)
- [`core::range::RangeInclusiveIter`](https://doc.rust-lang.org/stable/core/range/struct.RangeInclusiveIter.html)
- [`core::hint::cold_path`](https://doc.rust-lang.org/stable/core/hint/fn.cold_path.html)
- [`<*const T>::as_ref_unchecked`](https://doc.rust-lang.org/stable/std/primitive.pointer.html#method.as_ref_unchecked)
- [`<*mut T>::as_ref_unchecked`](https://doc.rust-lang.org/stable/std/primitive.pointer.html#method.as_ref_unchecked-1)
- [`<*mut T>::as_mut_unchecked`](https://doc.rust-lang.org/stable/std/primitive.pointer.html#method.as_mut_unchecked)
- [`Vec::push_mut`](https://doc.rust-lang.org/stable/std/vec/struct.Vec.html#method.push_mut)
- [`Vec::insert_mut`](https://doc.rust-lang.org/stable/std/vec/struct.Vec.html#method.insert_mut)
- [`VecDeque::push_front_mut`](https://doc.rust-lang.org/stable/std/collections/struct.VecDeque.html#method.push_front_mut)
- [`VecDeque::push_back_mut`](https://doc.rust-lang.org/stable/std/collections/struct.VecDeque.html#method.push_back_mut)
- [`VecDeque::insert_mut`](https://doc.rust-lang.org/stable/std/collections/struct.VecDeque.html#method.insert_mut)
- [`LinkedList::push_front_mut`](https://doc.rust-lang.org/stable/std/collections/struct.LinkedList.html#method.push_front_mut)
- [`LinkedList::push_back_mut`](https://doc.rust-lang.org/stable/std/collections/struct.LinkedList.html#method.push_back_mut)
- [`Layout::dangling_ptr`](https://doc.rust-lang.org/stable/std/alloc/struct.Layout.html#method.dangling_ptr)
- [`Layout::repeat`](https://doc.rust-lang.org/stable/std/alloc/struct.Layout.html#method.repeat)
- [`Layout::repeat_packed`](https://doc.rust-lang.org/stable/std/alloc/struct.Layout.html#method.repeat_packed)
- [`Layout::extend_packed`](https://doc.rust-lang.org/stable/std/alloc/struct.Layout.html#method.extend_packed)

Эти API, ранее уже стабильные, теперь также стабильны в `const`-контекстах:

- [`fmt::from_fn`](https://doc.rust-lang.org/stable/std/fmt/fn.from_fn.html)
- [`ControlFlow::is_break`](https://doc.rust-lang.org/stable/core/ops/enum.ControlFlow.html#method.is_break)
- [`ControlFlow::is_continue`](https://doc.rust-lang.org/stable/core/ops/enum.ControlFlow.html#method.is_continue)

### Дестабилизированные JSON-спецификации целей

Rust 1.95 убирает в stable поддержку передачи пользовательской спецификации
цели в `rustc`. Это **не должно** затронуть пользователей Rust, работающих с полностью stable
toolchain, поскольку сборка стандартной библиотеки (включая только `core`) и так
требовала возможностей, доступных только в nightly.

Мы также собираем сценарии использования пользовательских целей в [tracking issue](https://github.com/rust-lang/rust/issues/151528),
пока решаем, стоит ли в будущем стабилизировать какую-либо форму этой функциональности.

### Другие изменения

Смотрите полный список изменений в [Rust](https://github.com/rust-lang/rust/releases/tag/1.95.0), [Cargo](https://doc.rust-lang.org/nightly/cargo/CHANGELOG.html#cargo-195-2026-04-16) и [Clippy](https://github.com/rust-lang/rust-clippy/blob/master/CHANGELOG.md#rust-195).

## Участники релиза 1.95.0

Над созданием Rust 1.95.0 работало много людей. Без вас это было бы невозможно. [Спасибо!](https://thanks.rust-lang.org/rust/1.95.0/)

[platform-support]: https://doc.rust-lang.org/rustc/platform-support.html

