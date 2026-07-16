+++
path = "2026/07/09/Rust-1.97.0"
title = "Анонс Rust 1.97.0"
authors = ["The Rust Release Team"]
aliases = ["releases/1.97.0"]

[extra]
release = true
+++

Команда Rust рада объявить о выходе новой версии Rust — 1.97.0. Rust — это язык программирования, который даёт каждому возможность создавать надёжное и эффективное программное обеспечение.

Если у вас уже установлена предыдущая версия Rust через `rustup`, вы можете получить 1.97.0 командой:

```console
$ rustup update stable
```

Если Rust ещё не установлен, вы можете получить [`rustup`](https://www.rust-lang.org/install.html) на соответствующей странице нашего сайта и ознакомиться с [подробными примечаниями к выпуску 1.97.0](https://doc.rust-lang.org/stable/releases.html#version-1970-2026-07-09).

Если вы хотите помочь нам, тестируя будущие выпуски, рассмотрите возможность переключиться локально на канал beta (`rustup default beta`) или nightly (`rustup default nightly`). Пожалуйста, [сообщайте](https://github.com/rust-lang/rust/issues/new/choose) обо всех найденных ошибках!

## Что нового в стабильной версии 1.97.0

### Манглинг символов v0 включён по умолчанию

При компиляции Rust в объектные файлы и бинарники каждый элемент (функции,
статические переменные и т. д.) должен иметь глобально уникальный «символ»,
идентифицирующий его. Чтобы избежать конфликтов при линковке разных программ на
Rust, Rust преобразует (манглит) исходные имена элементов, добавляя
дополнительный контекст: путь модуля, крейт-определитель, обобщения и многое
другое. Исторически этот манглинг основывался на
[Itanium ABI](https://refspecs.linuxbase.org/cxxabi-1.86.html#mangling),
который также (иногда) используется в C++.

Новая схема манглинга устраняет ряд недостатков предыдущей:

* Инстанцирования обобщённых параметров сохраняют свои значения, а не отслеживаются только через хеш
* Несогласованности: не все части использовали Itanium ABI, поэтому всё равно требовался собственный деманглинг

Начиная с Rust 1.59, компилятор поддерживает переход на специфичную для Rust
схему манглинга через `-Csymbol-mangling-version=v0`. С ноября 2025 года эта
схема включена по умолчанию в nightly, а в 1.97 она включается в стабильном
Rust. Устаревшую схему манглинга можно включить только в nightly; текущий план —
полностью её удалить.

Подробнее см. в предыдущей [статье блога](https://blog.rust-lang.org/2025/11/20/switching-to-v0-mangling-on-nightly/).

### Поддержка Cargo для запрета предупреждений

В CI принято запрещать предупреждения. Исторически это обычно делалось через
`RUSTFLAGS=-Dwarnings`. В Rust 1.97 Cargo управляет тем, как предупреждения
влияют на успешность сборки: либо подавляя их (уровень `allow`), либо выводя
без сбоя (по умолчанию, `warn`), либо запрещая (уровень `deny`).

Поскольку поведение определяется конфигурацией Cargo, использование этой
возможности не инвалидирует кэш сборки, поэтому легко временно переключаться.
Например, если предупреждения мешают при исправлении ошибок после рефакторинга,
можно выполнить `CARGO_BUILD_WARNINGS=allow cargo check`, временно их подавив.

В CI задания могут вместо этого установить `CARGO_BUILD_WARNINGS=deny` для
запрета предупреждений. Это можно сочетать с `--keep-going`, чтобы собрать все
ошибки и предупреждения, а не останавливаться на первом упавшем пакете.

Подробнее см. в [документации](https://doc.rust-lang.org/cargo/reference/config.html#buildwarnings).

### Вывод линкера больше не скрывается по умолчанию

rustc вызывает линкер от имени пользователя. Исторически rustc по умолчанию
подавлял вывод линкера, если линковка завершалась успешно. Однако это может
скрывать реальные проблемы, поэтому в Rust 1.97 мы включаем сообщения линкера
по умолчанию. Они выводятся как предупреждение (lint), например:

```
warning: linker stderr: ignoring deprecated linker optimization setting '1'
  |
  = note: `#[warn(linker_messages)]` on by default
```

Распространённые сообщения линкера, которые были диагностированы как ложные
срабатывания или намеренное поведение, фильтруются rustc. Несколько дефектов
уже исправлено благодаря тому, что этот вывод больше не скрывается в nightly.

Обратите внимание, что сейчас `linker_messages` — особый lint, который *не*
затрагивается группой lint `warnings`. Это сделано намеренно: rustc в целом не
контролирует вывод линкера так точно, и нередко сообщения появляются только на
некоторых платформах. Если вы видите то, что считаете ложным срабатыванием
сообщения линкера, пожалуйста, [создайте issue].

Чтобы временно подавить предупреждение, можно настроить уровень lint на
`allow`. Это можно сделать через `Cargo.toml`, добавив [секцию lints] следующим
образом:

```toml
[lints.rust]
linker_messages = "allow"
```

[создайте issue]: https://github.com/rust-lang/rust/issues/new/choose
[секцию lints]: https://doc.rust-lang.org/nightly/cargo/reference/manifest.html#the-lints-section

### Стабилизированные API

- [`Default for RepeatN`](https://doc.rust-lang.org/stable/std/iter/struct.RepeatN.html#impl-Default-for-RepeatN%3CA%3E)
- [`Copy for ffi::FromBytesUntilNulError`](https://doc.rust-lang.org/stable/std/ffi/struct.FromBytesUntilNulError.html#impl-Copy-for-FromBytesUntilNulError)
- [`Send for std::fs::File` on UEFI](https://github.com/rust-lang/rust/pull/154003)
- [`<{integer}>::isolate_highest_one`](https://doc.rust-lang.org/stable/std/primitive.u32.html#method.isolate_highest_one)
- [`<{integer}>::isolate_lowest_one`](https://doc.rust-lang.org/stable/std/primitive.u32.html#method.isolate_lowest_one)
- [`<{integer}>::highest_one`](https://doc.rust-lang.org/stable/std/primitive.u32.html#method.highest_one)
- [`<{integer}>::lowest_one`](https://doc.rust-lang.org/stable/std/primitive.u32.html#method.lowest_one)
- [`<{uN}>::bit_width`](https://doc.rust-lang.org/stable/std/primitive.u32.html#method.bit_width)
- [`NonZero<{integer}>::isolate_highest_one`](https://doc.rust-lang.org/stable/std/num/struct.NonZero.html#method.isolate_highest_one)
- [`NonZero<{integer}>::isolate_lowest_one`](https://doc.rust-lang.org/stable/std/num/struct.NonZero.html#method.isolate_lowest_one)
- [`NonZero<{integer}>::highest_one`](https://doc.rust-lang.org/stable/std/num/struct.NonZero.html#method.highest_one)
- [`NonZero<{integer}>::lowest_one`](https://doc.rust-lang.org/stable/std/num/struct.NonZero.html#method.lowest_one)
- [`NonZero<{uN}>::bit_width`](https://doc.rust-lang.org/stable/std/num/struct.NonZero.html#method.bit_width)

Эти ранее стабилизированные API теперь стабильны в const-контекстах:

- [`char::is_control`](https://doc.rust-lang.org/stable/std/primitive.char.html#method.is_control)

### Прочие изменения

Ознакомьтесь со всеми изменениями в [Rust](https://github.com/rust-lang/rust/releases/tag/1.97.0), [Cargo](https://doc.rust-lang.org/nightly/cargo/CHANGELOG.html#cargo-197-2026-07-09) и [Clippy](https://github.com/rust-lang/rust-clippy/blob/master/CHANGELOG.md#rust-197).

## Участники выпуска 1.97.0

Многие люди объединили усилия, чтобы создать Rust 1.97.0. Без вас мы бы не справились. [Спасибо!](https://thanks.rust-lang.org/rust/1.97.0/)

