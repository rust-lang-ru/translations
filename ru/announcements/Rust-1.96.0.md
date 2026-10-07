+++
path = "2026/05/28/Rust-1.96.0"
title = "Анонс Rust 1.96.0"
authors = ["The Rust Release Team"]
aliases = ["releases/1.96.0"]

[extra]
release = true
+++

Команда Rust рада объявить о выходе новой версии языка — Rust 1.96.0. Rust — это язык программирования, который помогает каждому создавать надёжное и эффективное программное обеспечение.

Если у вас уже установлена предыдущая версия Rust через `rustup`, вы можете получить 1.96.0 командой:

```console
$ rustup update stable
```

Если Rust ещё не установлен, вы можете [получить `rustup`](https://www.rust-lang.org/install.html) на соответствующей странице нашего сайта и ознакомиться с [подробными release notes для 1.96.0](https://doc.rust-lang.org/stable/releases.html#version-1960-2026-05-28).

Если вы хотите помочь нам, тестируя будущие релизы, можете переключиться локально на beta-канал (`rustup default beta`) или nightly-канал (`rustup default nightly`). Пожалуйста, [сообщайте](https://github.com/rust-lang/rust/issues/new/choose) обо всех найденных ошибках!

## Что нового в stable 1.96.0

### Новые типы `Range*`

Многие пользователи ожидают, что `Range` и связанные типы из `core::ops` будут реализовывать `Copy`, но это не так: они напрямую реализуют `Iterator`, а [реализовывать одновременно `Iterator` и `Copy` для одного типа — плохая идея](https://rust-lang.github.io/rust-clippy/rust-1.95.0/index.html#copy_iterator), поэтому от этого сознательно отказались. [RFC3550] предложил набор заменяющих типов диапазонов, которые реализуют `IntoIterator`, а не `Iterator`, и поэтому могут также реализовывать `Copy`. Часть этого RFC, относящаяся к стандартной библиотеке, теперь стабилизирована и добавляет:

- `core::range::Range`
- `core::range::RangeFrom`
- `core::range::RangeInclusive`
- Сопутствующие итераторы

В одной из ближайших версий Rust также появятся `core::range::RangeFull` и `core::range::RangeTo` как re-export из `core::ops` (они не реализуют `Iterator` и уже реализуют `Copy`), а также `core::range::legacy::*` как новое место для текущих типов диапазонов. Синтаксис диапазонов вроде `0..1` пока по-прежнему создаёт legacy-типы, но в будущей edition будет обновлён на типы из `core::range`.

Благодаря этим стабилизациям теперь можно хранить аксессоры срезов в типах с `Copy`, не разделяя `start` и `end`:

```rust
use core::range::Range;

#[derive(Clone, Copy)]
pub struct Span(Range<usize>);

impl Span {
    pub fn of(self, s: &str) -> &str {
        &s[self.0]
    }
}
```

У нового `RangeInclusive` поля также публичные — в отличие от legacy-версии, которая избегала раскрывать состояние исчерпанного итератора. Для нового типа это не проблема, поскольку для начала итерации его нужно явно преобразовать.

Авторам библиотек стоит использовать в публичном API `impl RangeBounds`, который принимает и legacy-, и новые типы диапазонов. Если нужен конкретный тип, предпочтительнее использовать новые диапазоны — со временем они станут стандартом по умолчанию.

[RFC3550]: https://rust-lang.github.io/rfcs/3550-new-range.html

### Проверка соответствия паттерну

Новые макросы `assert_matches!` и `debug_assert_matches!` проверяют, что значение соответствует заданному паттерну, и в противном случае вызывают panic с `Debug`-представлением значения. По сути это то же самое, что `assert!(matches!(..))` и `debug_assert!(matches!(..))`, но выводимое значение упрощает диагностику сбоя.

Эти макросы не добавлены в стандартный prelude, потому что их имена конфликтуют с популярными сторонними crate, предоставляющими макросы с теми же названиями. Вместо этого их нужно вручную импортировать из `core` или `std` перед использованием.

```rust
use core::assert_matches;

/// [Случайное число](https://xkcd.com/221/)
fn get_random_number() -> u32 {
    // выбрано честным броском кубика.
    // гарантированно случайное.
    4
}

fn main() {
    assert_matches!(get_random_number(), 1..=6);
}
```

### Изменения в WebAssembly targets

WebAssembly targets больше не передают линкеру флаг `--allow-undefined`, поэтому неопределённые символы при линковке теперь вызывают ошибку линкера, а не превращаются в WebAssembly-импорты из модуля `"env"`. Это изменение не позволяет линковать модули, пока не определены все символы, связанные с линковкой, — так ошибки обнаруживаются раньше и предотвращаются случайные проблемы с именованием символов и подобные сбои.

Неопределённые символы, связанные с линковкой, часто указывают на ошибки сборки или неправильную конфигурацию. Если же нужно прежнее поведение, его можно вернуть через `RUSTFLAGS=-Clink-arg=--allow-undefined` или, изменив исходный код, с помощью `#[link(wasm_import_module = "env")]` в блоке, определяющем символ.

Об этом изменении [ранее сообщалось](https://blog.rust-lang.org/2026/04/04/changes-to-webassembly-targets-and-handling-undefined-symbols/) в блоге, и теперь оно вступает в силу в Rust 1.96.

### Стабилизированные API

- [`assert_matches!`](https://doc.rust-lang.org/stable/std/macro.assert_matches.html)
- [`debug_assert_matches!`](https://doc.rust-lang.org/stable/std/macro.debug_assert_matches.html)
- [`From<T> for AssertUnwindSafe<T>`](https://doc.rust-lang.org/stable/std/panic/struct.AssertUnwindSafe.html#impl-From%3CT%3E-for-AssertUnwindSafe%3CT%3E)
- [`From<T> for LazyCell<T, F>`](https://doc.rust-lang.org/stable/std/cell/struct.LazyCell.html#impl-From%3CT%3E-for-LazyCell%3CT,+F%3E)
- [`From<T> for LazyLock<T, F>`](https://doc.rust-lang.org/stable/std/sync/struct.LazyLock.html#impl-From%3CT%3E-for-LazyLock%3CT,+F%3E)
- [`core::range::RangeToInclusive`](https://doc.rust-lang.org/stable/core/range/struct.RangeToInclusive.html)
- [`core::range::RangeToInclusiveIter`](https://doc.rust-lang.org/stable/core/range/struct.RangeToInclusiveIter.html)
- [`core::range::RangeFrom`](https://doc.rust-lang.org/stable/core/ops/struct.RangeFrom.html)
- [`core::range::RangeFromIter`](https://doc.rust-lang.org/stable/core/ops/struct.RangeFromIter.html)
- [`core::range::Range`](https://doc.rust-lang.org/stable/std/range/struct.Range.html)
- [`core::range::RangeIter`](https://doc.rust-lang.org/stable/std/range/struct.RangeIter.html)

### Два security advisory для Cargo

Rust 1.96 содержит исправления двух уязвимостей для пользователей сторонних реестров.

- [CVE-2026-5223](https://blog.rust-lang.org/2026/05/25/cve-2026-5223/) — уязвимость **средней** степени серьёзности, связанная с распаковкой tarball crate с символическими ссылками.

- [CVE-2026-5222](https://blog.rust-lang.org/2026/05/25/cve-2026-5222/) — уязвимость **низкой** степени серьёзности, связанная с аутентификацией при нормализованных URL.

Пользователи crates.io **не затронуты** ни одной из этих уязвимостей.

### Прочие изменения

Посмотрите все изменения в [Rust](https://github.com/rust-lang/rust/releases/tag/1.96.0), [Cargo](https://doc.rust-lang.org/nightly/cargo/CHANGELOG.html#cargo-196-2026-05-28) и [Clippy](https://github.com/rust-lang/rust-clippy/blob/master/CHANGELOG.md#rust-196).

## Участники релиза 1.96.0

Многие люди объединили усилия, чтобы создать Rust 1.96.0. Без вас это было бы невозможно. [Спасибо!](https://thanks.rust-lang.org/rust/1.96.0/)
