Rust 1.90.0: ldd для x86_64-unknown-linux-gnu, публикация рабочих пространств и понижение x86_64-apple-darwin до Tier 2

[extra] release = true +++

Команда Rust рада сообщить о новой версии языка — 1.90.0. Rust — это язык программирования, позволяющий каждому создавать надёжное и эффективное программное обеспечение.

Если у вас есть предыдущая версия Rust, установленная через `rustup`, то для обновления до версии 1.89.0 вам достаточно выполнить команду:

```console
$ rustup update stable
```

Если у вас ещё не установлен `rustup`, вы можете установить его с [соответствующей страницы](https://www.rust-lang.org/install.html) нашего веб-сайта, а также посмотреть [подробные примечания к выпуску](https://doc.rust-lang.org/stable/releases.html#version-1900-2025-09-18) на GitHub.

Если вы хотите помочь нам протестировать будущие выпуски, вы можете использовать канал beta (`rustup default beta`) или nightly (`rustup default nightly`). Пожалуйста, [сообщайте](https://github.com/rust-lang/rust/issues/new/choose) обо всех встреченных вами ошибках.

## Что стабилизировано в 1.90.0

# Теперь LLD — линковщик по умолчанию на `x86_64-unknown-linux-gnu`

Целевая платформа `x86_64-unknown-linux-gnu` теперь будет по умолчанию использовать линковщик LLD для сборки крейтов Rust. Это должно улучшить производительность линковки по сравнению со стандартным линковщиком Linux (BFD), особенно при работе с большими исполняемыми файлами, файлами с большим объёмом отладочной информации, а также при инкрементальной пересборке.

Для абсолютного большинства случаев LLD должен быть обратно совместим с BFD, и вы не заметите никаких различий, кроме сокращённого времени компиляции. Однако если вы всё же столкнётесь с новыми проблемами при компоновке, вы всегда можете отключить LLD, используя флаг компилятора `-C linker-features=-lld`. Его можно добавить либо в привычную переменную окружения `RUSTFLAGS`, либо в файл конфигурации проекта [`.cargo/config.toml`](https://doc.rust-lang.org/cargo/reference/config.html), например так:

```toml
[target.x86_64-unknown-linux-gnu]
rustflags = ["-Clinker-features=-lld"]
```

Если вы столкнётесь с какими-либо проблемами, связанными с линковщиком LLD, — пожалуйста, сообщите нам. Более подробную информацию о переходе на LLD и механизме отключения, а также некоторые результаты тестов производительности вы можете прочитать [здесь](https://github.com/rust-lang/rust/issues/new/choose).

### Cargo получает нативную поддержку публикации рабочих пространств

Теперь поддерживается команда `cargo publish --workspace`, которая автоматически публикует все крейты в рабочем пространстве в правильной последовательности (с учётом зависимостей между ними).

Данная возможность долгое время была доступна с помощью внешних инструментов или ручной публикации отдельных крейтов, но теперь эта функциональность встроена в сам Cargo.

Нативная интеграция позволяет проверке публикации в Cargo запускать сборку всех крейтов, предназначенных для публикации, так, как если бы они уже были опубликованы, в том числе во время «тестовых прогонов». Обратите внимание, что публикации по-прежнему не являются атомарными — сетевые ошибки или сбои на стороне сервера могут привести к частичной публикации рабочего пространства.

### Понижение `x86_64-apple-darwin` до Tier 2 с инструментами хоста

GitHub скоро [прекратит] предоставлять бесплатные раннеры macOS x86_64 для публичных репозиториев. Apple также [анонсировала] свои планы по прекращению поддержки архитектуры x86_64.

В соответствии с этими изменениями, мы [понижаем таргет `x86_64-apple-darwin`] с [Tier 1 с инструментами хоста](https://doc.rust-lang.org/stable/rustc/platform-support.html#tier-1-with-host-tools) до [Tier 2 с инструментами хоста](https://doc.rust-lang.org/stable/rustc/platform-support.html#tier-2-with-host-tools). Это означает, что для этого таргета, включая такие инструменты, как `rustc` и `cargo`, будет гарантирована сборка, но не гарантируется прохождение нашего автоматизированного набора тестов.

Для пользователей это изменение не окажет немедленного влияния. Сборки как стандартной библиотеки, так и компилятора по-прежнему будут распространяться проектом Rust для использования через `rustup` или альтернативные методы установки, пока таргет остаётся в Tier 2. Со временем, вероятно, уменьшение тестового покрытия для этого таргета приведёт к поломкам или потере совместимости без дополнительных объявлений.

### Стабилизированные API

- [`u{n}::checked_sub_signed`](https://doc.rust-lang.org/stable/std/primitive.usize.html#method.checked_sub_signed)
- [`u{n}::overflowing_sub_signed`](https://doc.rust-lang.org/stable/std/primitive.usize.html#method.overflowing_sub_signed)
- [`u{n}::saturating_sub_signed`](https://doc.rust-lang.org/stable/std/primitive.usize.html#method.saturating_sub_signed)
- [`u{n}::wrapping_sub_signed`](https://doc.rust-lang.org/stable/std/primitive.usize.html#method.wrapping_sub_signed)
- [`impl Copy for IntErrorKind`](https://doc.rust-lang.org/stable/std/num/enum.IntErrorKind.html#impl-Copy-for-IntErrorKind)
- [`impl Hash for IntErrorKind`](https://doc.rust-lang.org/stable/std/num/enum.IntErrorKind.html#impl-Hash-for-IntErrorKind)
- [`impl PartialEq<&CStr> for CStr`](https://doc.rust-lang.org/stable/std/ffi/struct.CStr.html#impl-PartialEq%3C%26CStr%3E-for-CStr)
- [`impl PartialEq<CString> for CStr`](https://doc.rust-lang.org/stable/std/ffi/struct.CStr.html#impl-PartialEq%3CCString%3E-for-CStr)
- [`impl PartialEq<Cow<CStr>> for CStr`](https://doc.rust-lang.org/stable/std/ffi/struct.CStr.html#impl-PartialEq%3CCow%3C'_,+CStr%3E%3E-for-CStr)
- [`impl PartialEq<&CStr> for CString`](https://doc.rust-lang.org/stable/std/ffi/struct.CString.html#impl-PartialEq%3C%26CStr%3E-for-CString)
- [`impl PartialEq<CStr> for CString`](https://doc.rust-lang.org/stable/std/ffi/struct.CString.html#impl-PartialEq%3CCStr%3E-for-CString)
- [`impl PartialEq<Cow<CStr>> for CString`](https://doc.rust-lang.org/stable/std/ffi/struct.CString.html#impl-PartialEq%3CCow%3C'_,+CStr%3E%3E-for-CString)
- [`impl PartialEq<&CStr> for Cow<CStr>`](https://doc.rust-lang.org/stable/std/borrow/enum.Cow.html#impl-PartialEq%3C%26CStr%3E-for-Cow%3C'_,+CStr%3E)
- [`impl PartialEq<CStr> for Cow<CStr>`](https://doc.rust-lang.org/stable/std/borrow/enum.Cow.html#impl-PartialEq%3CCStr%3E-for-Cow%3C'_,+CStr%3E)
- [`impl PartialEq<CString> for Cow<CStr>`](https://doc.rust-lang.org/stable/std/borrow/enum.Cow.html#impl-PartialEq%3CCString%3E-for-Cow%3C'_,+CStr%3E)

Следующие API теперь можно использовать в контексте `const`:

- [`<[T]>::reverse`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.reverse)
- [`f32::floor`](https://doc.rust-lang.org/stable/std/primitive.f32.html#method.floor)
- [`f32::ceil`](https://doc.rust-lang.org/stable/std/primitive.f32.html#method.ceil)
- [`f32::trunc`](https://doc.rust-lang.org/stable/std/primitive.f32.html#method.trunc)
- [`f32::fract`](https://doc.rust-lang.org/stable/std/primitive.f32.html#method.fract)
- [`f32::round`](https://doc.rust-lang.org/stable/std/primitive.f32.html#method.round)
- [`f32::round_ties_even`](https://doc.rust-lang.org/stable/std/primitive.f32.html#method.round_ties_even)
- [`f64::floor`](https://doc.rust-lang.org/stable/std/primitive.f64.html#method.floor)
- [`f64::ceil`](https://doc.rust-lang.org/stable/std/primitive.f64.html#method.ceil)
- [`f64::trunc`](https://doc.rust-lang.org/stable/std/primitive.f64.html#method.trunc)
- [`f64::fract`](https://doc.rust-lang.org/stable/std/primitive.f64.html#method.fract)
- [`f64::round`](https://doc.rust-lang.org/stable/std/primitive.f64.html#method.round)
- [`f64::round_ties_even`](https://doc.rust-lang.org/stable/std/primitive.f64.html#method.round_ties_even)

### Поддержка платформ

- `x86_64-apple-darwin` теперь на 2 уровне поддержки

Дополнительную информацию о уровнях поддержки платформ в Rust можно найти на [странице поддержки].

### Прочие изменения

Проверьте всё, что изменилось в [Rust](https://github.com/rust-lang/rust/releases/tag/1.90.0), [Cargo](https://doc.rust-lang.org/nightly/cargo/CHANGELOG.html#cargo-190-2025-09-18) и [Clippy](https://github.com/rust-lang/rust-clippy/blob/master/CHANGELOG.md#rust-190).

## Кто работал над 1.90.0

Многие люди собрались вместе, чтобы создать Rust 1.90.0. Без вас мы бы не справились. [Спасибо!](https://thanks.rust-lang.org/rust/1.90.0/)


[анонсировала]: https://en.wikipedia.org/wiki/Mac_transition_to_Apple_silicon#Timeline
[прекратит]: https://github.blog/changelog/2025-07-11-upcoming-changes-to-macos-hosted-runners-macos-latest-migration-and-xcode-support-policy-updates/#macos-13-is-closing-down
[понижаем таргет `x86_64-apple-darwin`]: https://github.com/rust-lang/rfcs/pull/3841
[странице поддержки]: https://doc.rust-lang.org/rustc/platform-support.html
