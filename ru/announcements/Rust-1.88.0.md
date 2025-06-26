+++ path = "2025/06/26/Rust-1.88.0" title = "Announcing Rust 1.88.0" authors = ["The Rust Release Team"] aliases = ["releases/1.88.0"]

[extra] release = true +++

Команда Rust рада сообщить о новой версии языка — 1.88.0. Rust — это язык программирования, позволяющий каждому создавать надёжное и эффективное программное обеспечение.

Если у вас есть предыдущая версия Rust, установленная через `rustup`, то для обновления до версии 1.88.0 вам достаточно выполнить команду:

```
$ rustup update stable
```

Если у вас ещё не установлен `rustup`, вы можете установить его с [соответствующей страницы](https://www.rust-lang.org/install.html) нашего веб-сайта, а также посмотреть [подробные примечания к выпуску](https://doc.rust-lang.org/stable/releases.html#version-1880-2025-06-26) на GitHub.

Если вы хотите помочь нам протестировать будущие выпуски, вы можете использовать канал beta (`rustup default beta`) или nightly (`rustup default nightly`). Пожалуйста, [сообщайте](https://github.com/rust-lang/rust/issues/new/choose) обо всех встреченных вами ошибках.

## Что стабилизировано в 1.88.0

### Let цепочки

Эта функция позволяет объединять `let` выражения с помощью `&&` в условиях `if` и `while`, даже смешивая их с булевыми выражениями, таким образом уменьшая различие между `if`/`if let` и `while`/`while let`. Шаблоны внутри `let` подвыражений могут быть неопровержимыми или опровержимыми, а привязки доступны в последующих частях цепочки, а также в теле.

Например, этот фрагмент объединяет несколько условий, которые ранее потребовали бы вложенности блоков `if let` и `if`:

```rust
if let Channel::Stable(v) = release_info()
    && let Semver { major, minor, .. } = v
    && major == 1
    && minor == 88
{
    println!("`let_chains` стабилизированы в этой версии");
}
```

`let` цепочки доступны только в редакции Rust 2024, поскольку эта функция зависит от изменения [временной области видимости `if let`](https://doc.rust-lang.org/edition-guide/rust-2024/temporary-if-let-scope.html) для более согласованного порядка удаления.

Предыдущие попытки работали со всеми редакциями, но некоторые сложные граничные случаи угрожали целостности реализации. Редакция 2024 сделала это возможным, поэтому, пожалуйста, обновите редакцию вашего крейта, если вы хотите использовать эту функцию!

### Naked-функции

Rust теперь поддерживает написание naked-функций без генерируемого компилятором пролога и эпилога, что даёт полный контроль над сгенерированным ассемблерным кодом для конкретной функции. Это более эргономичная альтернатива определению функций в блоке `global_asm!`. Naked-функция помечается атрибутом `#[unsafe(naked)]`, а её тело состоит из единственного вызова `naked_asm!`.

Например:

```rust
#[unsafe(naked)]
pub unsafe extern "sysv64" fn wrapping_add(a: u64, b: u64) {
    // Эквивалентно `a.wrapping_add(b)`.
    core::arch::naked_asm!(
        "add rax, rdi, rsi",
        "ret"
    );
}
```

Блок ассемблерного кода, написанный вручную, определяет *всё* тело функции: в отличие от обычных функций, компилятор не добавляет специальной обработки для аргументов или возвращаемых значений. Naked-функции используются в низкоуровневых сценариях, таких как [`compiler-builtins`](https://github.com/rust-lang/compiler-builtins) Rust, операционные системы и встраиваемые приложения.

Ожидайте более подробную публикацию об этом в ближайшее время!

### Булевы литералы в `cfg`

Язык, используемый в предикатах `cfg`, теперь поддерживает булевы литералы `true` и `false`, действующие как конфигурации, которые всегда включены или выключены соответственно. Это работает в [условной компиляции](https://doc.rust-lang.org/reference/conditional-compilation.html) Rust с атрибутами `cfg` и `cfg_attr`, а также со встроенным макросом `cfg!`. Кроме того, это работает в таблицах `[target]` Cargo как [в конфигурации](https://doc.rust-lang.org/cargo/reference/config.html#target), так и [в манифестах](https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html#platform-specific-dependencies).

Ранее для безусловной конфигурации можно было использовать пустые списки предикатов, такие как `cfg(all())` для включения и `cfg(any())` для выключения. Однако это значение было довольно неявным и легко могло запутать. `cfg(true)` и `cfg(false)` предлагают более явный способ выразить то, что вы имеете в виду.

Более подробную информацию смотрите в [RFC 3695](https://rust-lang.github.io/rfcs/3695-cfg-boolean-literals.html)!

### Cargo теперь автоматически очищает кеш

Начиная с версии 1.88.0, Cargo будет автоматически запускать сборщик мусора для кэша в домашней директории!

При сборке проектов Cargo загружает и кэширует крейты, необходимые в качестве зависимостей. Исторически сложилось так, что эти загруженные файлы никогда не удалялись, что приводило к неограниченному росту использования дискового пространства в домашней директории Cargo. В этой версии Cargo представляет механизм сборки мусора для автоматической очистки старых файлов (например, файлов `.crate`). Cargo будет удалять файлы, загруженные по сети, если к ним не обращались в течение 3 месяцев, и файлы, полученные локально, если не было обращений в течение 1 месяца. Обратите внимание, что автоматическая сборка мусора не будет производиться, если Cargo работает в офлайн-режиме (с использованием флагов `--offline` или `--frozen`).

Cargo версии 1.78 и новее отслеживают информацию о доступе, необходимую для сборщика мусора. Это было внедрено задолго до включения фактической очистки, которая добавляется сейчас, чтобы уменьшить износ кэша для тех, кто всё ещё использует более ранние версии. Если вы регулярно используете версии Cargo старше 1.78, в дополнение к текущим версиям Cargo, и ожидаете, что к некоторым крейтам будут обращаться исключительно старые версии Cargo, и вы не хотите повторно загружать эти крейты каждые ~3 месяца, вы можете установить `cache.auto-clean-frequency = "never"` в конфигурации Cargo, как описано в [документации](https://doc.rust-lang.org/nightly/cargo/reference/config.html#cache).

Для получения более подробной информации смотрите [оригинальный анонс](https://blog.rust-lang.org/2023/12/11/cargo-cache-cleaning/) этой функции. Некоторые части этого дизайна остаются нестабильными, например, подкоманда `gc`, отслеживаемая в [cargo#13060](https://github.com/rust-lang/cargo/issues/13060), так что впереди ещё много интересного!

### Стабилизированные API

- [`Cell::update`](https://doc.rust-lang.org/stable/std/cell/struct.Cell.html#method.update)
- [`impl Default for *const T`](https://doc.rust-lang.org/stable/std/primitive.pointer.html#impl-Default-for-*const+T)
- [`impl Default for *mut T`](https://doc.rust-lang.org/stable/std/primitive.pointer.html#impl-Default-for-*mut+T)
- [`mod ffi::c_str`](https://doc.rust-lang.org/stable/std/ffi/c_str/index.html)
- [`HashMap::extract_if`](https://doc.rust-lang.org/stable/std/collections/struct.HashMap.html#method.extract_if)
- [`HashSet::extract_if`](https://doc.rust-lang.org/stable/std/collections/struct.HashSet.html#method.extract_if)
- [`hint::select_unpredictable`](https://doc.rust-lang.org/stable/std/hint/fn.select_unpredictable.html)
- [`proc_macro::Span::line`](https://doc.rust-lang.org/stable/proc_macro/struct.Span.html#method.line)
- [`proc_macro::Span::column`](https://doc.rust-lang.org/stable/proc_macro/struct.Span.html#method.column)
- [`proc_macro::Span::start`](https://doc.rust-lang.org/stable/proc_macro/struct.Span.html#method.start)
- [`proc_macro::Span::end`](https://doc.rust-lang.org/stable/proc_macro/struct.Span.html#method.end)
- [`proc_macro::Span::file`](https://doc.rust-lang.org/stable/proc_macro/struct.Span.html#method.file)
- [`proc_macro::Span::local_file`](https://doc.rust-lang.org/stable/proc_macro/struct.Span.html#method.local_file)
- [`<[T]>::as_chunks`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.as_chunks)
- [`<[T]>::as_rchunks`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.as_rchunks)
- [`<[T]>::as_chunks_unchecked`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.as_chunks_unchecked)
- [`<[T]>::as_chunks_mut`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.as_chunks_mut)
- [`<[T]>::as_rchunks_mut`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.as_rchunks_mut)
- [`<[T]>::as_chunks_unchecked_mut`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.as_chunks_unchecked_mut)

Следующие API теперь можно использовать в контексте <code>const</code>:

- [`NonNull<T>::replace`](https://doc.rust-lang.org/stable/std/ptr/struct.NonNull.html#method.replace)
- [`<*mut T>::replace`](https://doc.rust-lang.org/stable/std/primitive.pointer.html#method.replace)
- [`std::ptr::swap_nonoverlapping`](https://doc.rust-lang.org/stable/std/ptr/fn.swap_nonoverlapping.html)
- [`Cell::replace`](https://doc.rust-lang.org/stable/std/cell/struct.Cell.html#method.replace)
- [`Cell::get`](https://doc.rust-lang.org/stable/std/cell/struct.Cell.html#method.get)
- [`Cell::get_mut`](https://doc.rust-lang.org/stable/std/cell/struct.Cell.html#method.get_mut)
- [`Cell::from_mut`](https://doc.rust-lang.org/stable/std/cell/struct.Cell.html#method.from_mut)
- [`Cell::as_slice_of_cells`](https://doc.rust-lang.org/stable/std/cell/struct.Cell.html#method.as_slice_of_cells)

### Прочие изменения

Как было [объявлено ранее](https://blog.rust-lang.org/2025/05/26/demoting-i686-pc-windows-gnu/), уровень поддержки целевой платформы `i686-pc-windows-gnu`  был понижен до Tier 2. Сейчас это не затронет пользователей, так как и компилятор, и стандартная библиотека поставляются через `rustup` для этой платформы. Однако, без тестирования, которое было на Tier 1, становится больше шансов получить баги в будущем.

Проверьте всё, что изменилось в [Rust](https://github.com/rust-lang/rust/releases/tag/1.88.0), [Cargo](https://doc.rust-lang.org/nightly/cargo/CHANGELOG.html#cargo-188-2025-06-26) и [Clippy](https://github.com/rust-lang/rust-clippy/blob/master/CHANGELOG.md#rust-188).

## Кто работал над 1.88.0

Многие люди собрались вместе, чтобы создать Rust 1.88.0. Без вас мы бы не справились. [Спасибо!](https://thanks.rust-lang.org/rust/1.88.0/)
