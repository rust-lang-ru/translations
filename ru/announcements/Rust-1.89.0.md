Rust 1.89.0

[extra] release = true +++

Команда Rust рада сообщить о новой версии языка — 1.89.0. Rust — это язык программирования, позволяющий каждому создавать надёжное и эффективное программное обеспечение.

Если у вас есть предыдущая версия Rust, установленная через `rustup`, то для обновления до версии 1.89.0 вам достаточно выполнить команду:

```console
$ rustup update stable
```

Если у вас ещё не установлен `rustup`, вы можете установить его с [соответствующей страницы](https://www.rust-lang.org/install.html) нашего веб-сайта, а также посмотреть [подробные примечания к выпуску](https://doc.rust-lang.org/stable/releases.html#version-1890-2025-08-07) на GitHub.

Если вы хотите помочь нам протестировать будущие выпуски, вы можете использовать канал beta (`rustup default beta`) или nightly (`rustup default nightly`). Пожалуйста, [сообщайте](https://github.com/rust-lang/rust/issues/new/choose) обо всех встреченных вами ошибках.

## Что стабилизировано в 1.89.0

### Явный вывод аргументов для константных обобщений

Теперь Rust поддерживает использование `_` в качестве аргумента для константных обобщённых параметров, что позволяет выводить значение из окружающего контекста:

```rust
pub fn all_false<const LEN: usize>() -> [bool; LEN] {
  [false; _]
}
```

Аналогично правилам, когда `_` разрешается в качестве типа, `_` не разрешается в качестве аргумента для константных обобщений в сигнатуре:

```rust
// Так делать нельзя
pub const fn all_false<const LEN: usize>() -> [bool; _] {
  [false; LEN]
}

// И так тоже
pub const ALL_FALSE: [bool; _] = all_false::<10>();
```

### Проверка для несовпадающего синтаксиса времён жизни

[Сокращение времён жизни] в сигнатурах функций — это удобная особенность языка Rust, но она может стать камнем преткновения как для новичков, так и для опытных разработчиков. Это особенно заметно, когда времена жизни выводятся в типах, где синтаксически неочевидно их наличие:

```rust
// Возвращаемый тип `std::slice::Iter` имеет время жизни,
// но визуально это никак не показано.
//
// Сокращение времени жизни выводит, что время жизни
// возвращаемого типа такое же, как у `scores`.
fn items(scores: &[u8]) -> std::slice::Iter<u8> {
   scores.iter()
}
```

Подобный код теперь будет по умолчанию выдавать предупреждение:

```text
warning: hiding a lifetime that's elided elsewhere is confusing
 --> src/lib.rs:1:18
  |
1 | fn items(scores: &[u8]) -> std::slice::Iter<u8> {
  |                  ^^^^^     -------------------- the same lifetime is hidden here
  |                  |
  |                  the lifetime is elided here
  |
  = help: the same lifetime is referred to in inconsistent ways, making the signature confusing
  = note: `#[warn(mismatched_lifetime_syntaxes)]` on by default
help: use `'_` for type paths
  |
1 | fn items(scores: &[u8]) -> std::slice::Iter<'_, u8> {
  |                                             +++
```

[Впервые] мы попытались улучшить эту ситуацию ещё в 2018 году в рамках группы проверок [`rust_2018_idioms`], но [сильный отклик] на проверку `elided_lifetimes_in_paths` показал, что она была слишком "грубым инструментом", так как выдавала предупреждения о временах жизни, которые не важны для понимания функции:

```rust
use std::fmt;

struct Greeting;

impl fmt::Display for Greeting {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        //                         -----^^^^^^^^^ ожидаемый параметр времени жизни
        // Знание о том, что `Formatter` имеет время жизни, не помогает программисту
        "howdy".fmt(f)
    }
}
```

Затем мы поняли, что путаница, которую мы хотим устранить, возникает, когда:

1. Правила вывода сокращения времён жизни *связывают* входное время жизни с выходным;
2. Синтаксически неочевидно, что время жизни существует.

В Rust есть два синтаксических элемента, которые указывают на существование времени жизни: `&` и `'`. Символ `'` подразделяется на выводимое время жизни `'_` и именованные времена жизни `'a`. Когда тип использует именованное время жизни, сокращение не будет выводить время жизни для этого типа. Используя эти критерии, мы можем выделить три группы:

Очевидно, что есть время жизни | Разрешено сокращение времени жизни | Примеры
--- | --- | ---
Нет | Да | `ContainsLifetime`
Да | Да | `&T`, `&'_ T`, `ContainsLifetime<'_>`
Да | Нет | `&'a T`, `ContainsLifetime<'a>`

Проверка `mismatched_lifetime_syntaxes` проверяет, принадлежат ли входные и выходные данные функции к одной и той же группе. Для примера выше `&[u8]` относится ко второй группе, а `std::slice::Iter<u8>` — к первой. Мы говорим, что времена жизни в первой группе являются *скрытыми*.

Поскольку времена жизни на входе и выходе принадлежат к разным группам, проверка выдаст предупреждение для этой функции, уменьшая путаницу относительно того, когда значение имеет значимое время жизни, которое не очевидно визуально.

Проверка `mismatched_lifetime_syntaxes` заменяет проверку `elided_named_lifetimes`, которая делала нечто похожее специально для именованных времён жизни.

Дальнейшая работа над проверкой `elided_lifetimes_in_paths` предполагает её разделение на более сфокусированные подпроверки с целью в конечном итоге начать выдавать предупреждения для их подмножества.

### Больше возможностей для целевых платформ x86

Атрибут `target_feature` теперь поддерживает функции `sha512`, `sm3`, `sm4`, `kl` и `widekl` для x86. Кроме того, на x86 также поддерживается ряд встроенных функций и возможностей `avx512`:

```rust
#[target_feature(enable = "avx512bw")]
pub fn cool_simd_code(/* .. */) -> /* ... */ {
    /* ... */
}

```

### Кросс-компилируемые документационные тесты

Теперь документационные тесты будут запускаться при выполнении команды `cargo test --doc --target other_target`. Это может привести к поломкам, поскольку тесты, которые ранее не запускались и могли завершиться с ошибкой, теперь будут использованы.

Падающие тесты можно отключить, добавив к документационному тесту аннотацию `ignore-<target>` ([документация](https://doc.rust-lang.org/stable/rustdoc/write-documentation/documentation-tests.html#ignoring-targets)).

```rust
/// ```ignore-x86_64
/// panic!("something")
/// ```
pub fn my_function() { }
```

### `i128` и `u128` в функциях `extern "C"`

Типы `i128` и `u128` теперь можно использовать в функциях `extern "C"` без предупреждения, поскольку они больше не вызывают проверку `improper_ctypes_definitions`. Однако есть несколько нюансов:

- Типы Rust ABI- и layout- совместимы с (беззнаковым) `__int128` в C, если этот тип доступен.
- На платформах, где `__int128` недоступен, `i128` и `u128` не обязательно совпадают с каким-либо типом C.
- `i128` *не* обязательно совместим с `_BitInt(128)` на любой платформе, поскольку `_BitInt(128)` и `__int128` могут иметь разный ABI (как, например, на x86-64).

Это последнее обновление, связанное [с прошлогодними изменениями в layout](https://blog.rust-lang.org/2024/03/30/i128-layout-update/).

### Понижение `x86_64-apple-darwin` до Tier 2 с инструментами хоста

GitHub скоро [прекратит] предоставлять бесплатные раннеры macOS x86_64 для публичных репозиториев. Apple также [анонсировала] свои планы по прекращению поддержки архитектуры x86_64.

В соответствии с этими изменениями, проект Rust находится в [процессе понижения таргета `x86_64-apple-darwin`] с [Tier 1 с инструментами хоста](https://doc.rust-lang.org/stable/rustc/platform-support.html#tier-1-with-host-tools) до [Tier 2 с инструментами хоста](https://doc.rust-lang.org/stable/rustc/platform-support.html#tier-2-with-host-tools). Это означает, что для этого таргета, включая такие инструменты, как `rustc` и `cargo`, будет гарантирована сборка, но не гарантируется прохождение нашего автоматизированного набора тестов.

Мы ожидаем, что RFC о понижении до Tier 2 с инструментами хоста будет принят между релизами Rust 1.89 и 1.90. Это означает, что Rust 1.89 будет последним релизом, где `x86_64-apple-darwin` является таргетом Tier 1.

Для пользователей это изменение не окажет немедленного влияния. Сборки как стандартной библиотеки, так и компилятора по-прежнему будут распространяться проектом Rust для использования через `rustup` или альтернативные методы установки, пока таргет остаётся в Tier 2. Но вероятно, что со временем уменьшение тестового покрытия для этого таргета приведёт к поломкам или потере совместимости без дополнительных объявлений.

### C ABI, соответствующий стандартам, для таргета `wasm32-unknown-unknown`

Функции `extern "C"` для таргета `wasm32-unknown-unknown` теперь имеют ABI, соответствующий стандартам. Подробнее об этом можно узнать [в этой статье](https://blog.rust-lang.org/2025/04/04/c-abi-changes-for-wasm32-unknown-unknown).

### Поддержка платформ

- [`x86_64-apple-darwin` находится в процессе понижения до Tier 2 с инструментами хоста](https://github.com/rust-lang/rfcs/pull/3841)
- [Добавлены новые Tier-3 платформы `loongarch32-unknown-none<code data-md-type="codespan"> и <code data-md-type="codespan">loongarch32-unknown-none-softfloat`](https://github.com/rust-lang/rust/pull/142053)

Дополнительную информацию о уровнях поддержки платформ в Rust можно найти на [странице поддержки].

### Стабилизированные API

- [`NonZero<char>`](https://doc.rust-lang.org/stable/std/num/struct.NonZero.html)
- Множество встроенных функций для x86, не перечисленных здесь:
    - [Встроенные функции AVX512](https://github.com/rust-lang/rust/issues/111137)
    - [Встроенные функции `SHA512`, `SM3` и `SM4`](https://github.com/rust-lang/rust/issues/126624)
- [`File::lock`](https://doc.rust-lang.org/stable/std/fs/struct.File.html#method.lock)
- [`File::lock_shared`](https://doc.rust-lang.org/stable/std/fs/struct.File.html#method.lock_shared)
- [`File::try_lock`](https://doc.rust-lang.org/stable/std/fs/struct.File.html#method.try_lock)
- [`File::try_lock_shared`](https://doc.rust-lang.org/stable/std/fs/struct.File.html#method.try_lock_shared)
- [`File::unlock`](https://doc.rust-lang.org/stable/std/fs/struct.File.html#method.unlock)
- [`NonNull::from_ref`](https://doc.rust-lang.org/stable/std/ptr/struct.NonNull.html#method.from_ref)
- [`NonNull::from_mut`](https://doc.rust-lang.org/stable/std/ptr/struct.NonNull.html#method.from_mut)
- [`NonNull::without_provenance`](https://doc.rust-lang.org/stable/std/ptr/struct.NonNull.html#method.without_provenance)
- [`NonNull::with_exposed_provenance`](https://doc.rust-lang.org/stable/std/ptr/struct.NonNull.html#method.with_exposed_provenance)
- [`NonNull::expose_provenance`](https://doc.rust-lang.org/stable/std/ptr/struct.NonNull.html#method.expose_provenance)
- [`OsString::leak`](https://doc.rust-lang.org/stable/std/ffi/struct.OsString.html#method.leak)
- [`PathBuf::leak`](https://doc.rust-lang.org/stable/std/path/struct.PathBuf.html#method.leak)
- [`Result::flatten`](https://doc.rust-lang.org/stable/std/result/enum.Result.html#method.flatten)
- [`std::os::linux::net::TcpStreamExt::quickack`](https://doc.rust-lang.org/stable/std/os/linux/net/trait.TcpStreamExt.html#tymethod.quickack)
- [`std::os::linux::net::TcpStreamExt::set_quickack`](https://doc.rust-lang.org/stable/std/os/linux/net/trait.TcpStreamExt.html#tymethod.set_quickack)

Следующие API теперь можно использовать в контексте <code>const</code>:

- [`<[T; N]>::as_mut_slice`](https://doc.rust-lang.org/stable/std/primitive.array.html#method.as_mut_slice)
- [`<[u8]>::eq_ignore_ascii_case`](https://doc.rust-lang.org/stable/std/primitive.slice.html#impl-%5Bu8%5D/method.eq_ignore_ascii_case)
- [`str::eq_ignore_ascii_case`](https://doc.rust-lang.org/stable/std/primitive.str.html#impl-str/method.eq_ignore_ascii_case)

### Прочие изменения

Проверьте всё, что изменилось в [Rust](https://github.com/rust-lang/rust/releases/tag/1.89.0), [Cargo](https://doc.rust-lang.org/nightly/cargo/CHANGELOG.html#cargo-189-2025-08-07) и [Clippy](https://github.com/rust-lang/rust-clippy/blob/master/CHANGELOG.md#rust-189).

## Кто работал над 1.89.0

Многие люди собрались вместе, чтобы создать Rust 1.89.0. Без вас мы бы не справились. [Спасибо!](https://thanks.rust-lang.org/rust/1.89.0/)


[Сокращение времён жизни]: https://doc.rust-lang.org/1.89/book/ch10-03-lifetime-syntax.html#lifetime-elision
[Впервые]: https://github.com/rust-lang/rust/pull/46254
[`rust_2018_idioms`]: https://github.com/rust-lang/rust/issues/54910
[сильный отклик]: https://github.com/rust-lang/rust/issues/131725
[анонсировала]: https://en.wikipedia.org/wiki/Mac_transition_to_Apple_silicon#Timeline
[прекратит]: https://github.blog/changelog/2025-07-11-upcoming-changes-to-macos-hosted-runners-macos-latest-migration-and-xcode-support-policy-updates/#macos-13-is-closing-down
[процессе понижения таргета `x86_64-apple-darwin`]: https://github.com/rust-lang/rfcs/pull/3841
[странице поддержки]: https://doc.rust-lang.org/rustc/platform-support.html
