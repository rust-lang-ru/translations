+++
path = "2026/03/05/Rust-1.94.0"
title = "Анонс Rust 1.94.0"
authors = ["Команда выпуска Rust"]
aliases = ["releases/1.94.0"]

[extra]
release = true
+++

Команда Rust рада объявить о новом выпуске Rust 1.94.0. Rust — это язык программирования, который дает каждому возможность создавать надежное и эффективное программное обеспечение.

Если у вас уже установлена предыдущая версия Rust через `rustup`, вы можете получить 1.94.0 командой:

```console
$ rustup update stable
```

Если у вас еще не установлен Rust, вы можете [получить `rustup`](https://www.rust-lang.org/install.html) на соответствующей странице нашего сайта, а также ознакомиться с [подробными примечаниями к выпуску 1.94.0](https://doc.rust-lang.org/stable/releases.html#version-1940-2026-03-05).

Если вы хотите помочь нам, тестируя будущие релизы, попробуйте локально переключиться на beta-канал (`rustup default beta`) или nightly-канал (`rustup default nightly`). Пожалуйста, [сообщайте](https://github.com/rust-lang/rust/issues/new/choose) о любых найденных ошибках!

## Что вошло в стабильный 1.94.0

### Окна массива

В Rust 1.94 добавлен [`array_windows`] — метод итерации для срезов. Он работает так же, как [`windows`], но с постоянной длиной, поэтому элементы итератора имеют тип `&[T; N]`, а не динамически-размерный `&[T]`. Во многих случаях длина окна может даже выводиться из того, как используется итератор!

Например, в одной из задач [Advent of Code 2016](https://adventofcode.com/2016/day/7) нужно находить шаблоны ABBA: «две разные буквы, за которыми следует обратный порядок этой пары, например `xyyx` или `abba`». Если предположить, что используются только ASCII-символы, это можно записать, проходя окнами по байтовому срезу так:

```rust
fn has_abba(s: &str) -> bool {
    s.as_bytes()
        .array_windows()
        .any(|[a1, b1, b2, a2]| (a1 != b1) && (a1 == a2) && (b1 == b2))
}
```

Паттерн деструктуризации аргумента в этом замыкании позволяет компилятору вывести, что здесь нужны окна длины 4. Если бы мы использовали более старый итератор `.windows(4)`, тогда аргументом был бы срез, который пришлось бы индексировать вручную, *надеясь*, что проверки границ во время выполнения будут оптимизированы.

[`array_windows`]: https://doc.rust-lang.org/stable/std/primitive.slice.html#method.array_windows
[`windows`]: https://doc.rust-lang.org/stable/std/primitive.slice.html#method.windows

### Подключение конфигов Cargo

Cargo теперь поддерживает ключ `include` в конфигурационных файлах (`.cargo/config.toml`), что улучшает организацию, совместное использование и управление конфигурациями Cargo между проектами и окружениями. Эти пути включения также можно помечать как `optional`, если в некоторых ситуациях файлы могут отсутствовать, например в зависимости от локальных настроек разработчика.

```toml
# массив путей
include = [
    "frodo.toml",
    "samwise.toml",
]

# inline-таблицы для более тонкого контроля
include = [
    { path = "required.toml" },
    { path = "optional.toml", optional = true },
]
```

Подробности смотрите в полной [документации `include`](https://doc.rust-lang.org/nightly/cargo/reference/config.html#including-extra-configuration-files).

### Поддержка TOML 1.1 в Cargo

Cargo теперь разбирает [TOML v1.1](https://toml.io/en/v1.1.0) для манифестов и конфигурационных файлов. Подробные изменения смотрите в [примечаниях к выпуску TOML](https://github.com/toml-lang/toml/releases/tag/1.1.0), среди них:

* Inline-таблицы на нескольких строках и с завершающими запятыми
* Символы экранирования строк `\xHH` и `\e`
* Необязательные секунды во времени (устанавливаются в 0)

Например, зависимость вида:

```toml
serde = { version = "1.0", features = ["derive"] }
```

... теперь можно записывать так:

<!-- FIXME: this should be toml, but the blog highlighting doesn't support 1.1 yet -->
```text
serde = {
    version = "1.0",
    features = ["derive"],
}
```

Обратите внимание, что использование этих возможностей в `Cargo.toml` поднимет ваш MSRV разработки (минимальную поддерживаемую версию Rust) до уровня, требующего нового парсера Cargo, а сторонним инструментам, читающим манифест, также может понадобиться обновить свои парсеры. Однако при публикации Cargo автоматически переписывает манифесты так, чтобы они оставались совместимыми со старыми парсерами, поэтому для пользователей вашего крейта все еще можно поддерживать более ранний MSRV.

### Стабилизированные API

- [`<[T]>::array_windows`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.array_windows)
- [`<[T]>::element_offset`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.element_offset)
- [`LazyCell::get`](https://doc.rust-lang.org/stable/std/cell/struct.LazyCell.html#method.get)
- [`LazyCell::get_mut`](https://doc.rust-lang.org/stable/std/cell/struct.LazyCell.html#method.get_mut)
- [`LazyCell::force_mut`](https://doc.rust-lang.org/stable/std/cell/struct.LazyCell.html#method.force_mut)
- [`LazyLock::get`](https://doc.rust-lang.org/stable/std/sync/struct.LazyLock.html#method.get)
- [`LazyLock::get_mut`](https://doc.rust-lang.org/stable/std/sync/struct.LazyLock.html#method.get_mut)
- [`LazyLock::force_mut`](https://doc.rust-lang.org/stable/std/sync/struct.LazyLock.html#method.force_mut)
- [`impl TryFrom<char> for usize`](https://doc.rust-lang.org/stable/std/convert/trait.TryFrom.html#impl-TryFrom%3Cchar%3E-for-usize)
- [`std::iter::Peekable::next_if_map`](https://doc.rust-lang.org/stable/std/iter/struct.Peekable.html#method.next_if_map)
- [`std::iter::Peekable::next_if_map_mut`](https://doc.rust-lang.org/stable/std/iter/struct.Peekable.html#method.next_if_map_mut)
- [x86 `avx512fp16` intrinsic-функции](https://github.com/rust-lang/rust/issues/127213)
  (за исключением тех, которые напрямую зависят от нестабильного типа `f16`)
- [AArch64 NEON fp16 intrinsic-функции](https://github.com/rust-lang/rust/issues/136306)
  (за исключением тех, которые напрямую зависят от нестабильного типа `f16`)
- [`f32::consts::EULER_GAMMA`](https://doc.rust-lang.org/stable/std/f32/consts/constant.EULER_GAMMA.html)
- [`f64::consts::EULER_GAMMA`](https://doc.rust-lang.org/stable/std/f64/consts/constant.EULER_GAMMA.html)
- [`f32::consts::GOLDEN_RATIO`](https://doc.rust-lang.org/stable/std/f32/consts/constant.GOLDEN_RATIO.html)
- [`f64::consts::GOLDEN_RATIO`](https://doc.rust-lang.org/stable/std/f64/consts/constant.GOLDEN_RATIO.html)

Эти API, ранее уже стабильные, теперь также стабильны в `const`-контекстах:

- [`f32::mul_add`](https://doc.rust-lang.org/stable/std/primitive.f32.html#method.mul_add)
- [`f64::mul_add`](https://doc.rust-lang.org/stable/std/primitive.f64.html#method.mul_add)

### Другие изменения

Смотрите полный список изменений в [Rust](https://github.com/rust-lang/rust/releases/tag/1.94.0), [Cargo](https://doc.rust-lang.org/nightly/cargo/CHANGELOG.html#cargo-194-2026-03-05) и [Clippy](https://github.com/rust-lang/rust-clippy/blob/master/CHANGELOG.md#rust-194).

## Участники релиза 1.94.0

Над созданием Rust 1.94.0 работало много людей. Без вас это было бы невозможно. [Спасибо!](https://thanks.rust-lang.org/rust/1.94.0/)

[platform-support]: https://doc.rust-lang.org/rustc/platform-support.html
