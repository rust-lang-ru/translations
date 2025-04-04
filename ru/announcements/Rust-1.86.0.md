+++ layout = "post" date = 2025-04-03 title = "Rust 1.86.0" author = "The Rust Release Team" release = true +++

Команда Rust рада сообщить о новой версии языка — 1.86.0. Rust — это язык программирования, позволяющий каждому создавать надёжное и эффективное программное обеспечение.

Если у вас есть предыдущая версия Rust, установленная через `rustup`, то для обновления до версии 1.86.0 вам достаточно выполнить команду:

```console
$ rustup update stable
```

Если у вас ещё не установлен `rustup`, вы можете установить его с [соответствующей страницы](https://www.rust-lang.org/install.html) нашего веб-сайта, а также посмотреть [подробные примечания к выпуску](https://doc.rust-lang.org/stable/releases.html#version-1860-2025-04-03) на GitHub.

Если вы хотите помочь нам протестировать будущие выпуски, вы можете использовать канал beta (`rustup default beta`) или nightly (`rustup default nightly`). Пожалуйста, [сообщайте](https://github.com/rust-lang/rust/issues/new/choose) обо всех встреченных вами ошибках.

## Что стабилизировано в 1.86.0

### Преобразование в родительский трейт

В этом выпуске появилась долгожданная функция — возможность приводить объекты трейта к родительскому трейту. Если у трейта есть [супертрейт](https://doc.rust-lang.org/reference/items/traits.html#supertraits), вы можете привести ссылку на указанный трейт-объект к ссылке на трейт-объект супертрейта:

```rust
trait Trait: Supertrait {}
trait Supertrait {}

fn upcast(x: &dyn Trait) -> &dyn Supertrait {
    x
}
```

The same would work with any other kind of (smart-)pointer, like `Arc<dyn Trait> -> Arc<dyn Supertrait>` or `*const dyn Trait -> *const dyn Supertrait`.

Previously this would have required a workaround in the form of an `upcast` method in the `Trait` itself, for example `fn as_supertrait(&self) -> &dyn Supertrait`, and this would work only for one kind of reference/pointer. Such workarounds are not necessary anymore.

Обратите внимание, что это означает, что необработанные указатели на nhtqn-объекты содержат нетривиальный инвариант: "утёкший" в безопасный код необработанный указатель на трейт-объект с недопустимой виртуальной таблицей  может привести к неопределенному поведению. Пока не решено, приведет ли временное создание такого необработанного указателя в хорошо контролируемых обстоятельствах к немедленному неопределенному поведению или нет, поэтому код должен воздерживаться от создания таких указателей ни при каких условиях (и Miri обеспечивает это).

Преобразование к родительскому трейту может быть особенно полезно при использовании трейта `Any`, поскольку он позволяет преобразовать ваш трейт-объект в `dyn Any` для вызова `Any::downcast` без добавления к трейту каких-либо методов или использования внешних крейтов.

```rust
use std::any::Any;

trait MyAny: Any {}

impl dyn MyAny {
    fn downcast_ref<T>(&self) -> Option<&T> {
        (self as &dyn Any).downcast_ref()
    }
}
```

You can [learn more about trait upcasting in the Rust reference](https://doc.rust-lang.org/reference/type-coercions.html#unsized-coercions).

### `HashMap` и срезы теперь поддерживают изменяемую индексацию нескольких элементов

Анализатор заимствований предотвращает одновременное использование ссылок, полученных в результате повторных вызовов методов `get_mut`. Для безопасной поддержки этого шаблона стандартная библиотека теперь предоставляет метод `get_disjoint_mut` для срезов и `HashMap` для одновременного извлечения изменяемых ссылок на несколько элементов. Смотрите следующий пример, взятый из документации API для [`slice::get_disjoint_mut`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.get_disjoint_mut):

```rust
let v = &mut [1, 2, 3];
if let Ok([a, b]) = v.get_disjoint_mut([0, 2]) {
    *a = 413;
    *b = 612;
}
assert_eq!(v, &[413, 2, 612]);

if let Ok([a, b]) = v.get_disjoint_mut([0..1, 1..3]) {
    a[0] = 8;
    b[0] = 88;
    b[1] = 888;
}
assert_eq!(v, &[8, 88, 888]);

if let Ok([a, b]) = v.get_disjoint_mut([1..=2, 0..=0]) {
    a[0] = 11;
    a[1] = 111;
    b[0] = 1;
}
assert_eq!(v, &[1, 11, 111]);
```

### Разрешено помечать безопасные функции атрибутом `#[target_feature]`

Previously only `unsafe` functions could be marked with the `#[target_feature]` attribute as it is unsound to call such functions without the target feature being enabled. This release stabilizes the `target_feature_11` feature, allowing *safe* functions to be marked with the `#[target_feature]` attribute.

Safe functions marked with the target feature attribute can only be safely called from other functions marked with the target feature attribute. However, they cannot be passed to functions accepting generics bounded by the `Fn*` traits and only support being coerced to function pointers inside of functions marked with the `target_feature` attribute.

Inside of functions not marked with the target feature attribute they can be called inside of an `unsafe` block, however it is the callers responsibility to ensure that the target feature is available.

```rust
#[target_feature(enable = "avx2")]
fn requires_avx2() {
    // ... snip
}

#[target_feature(enable = "avx2")]
fn safe_callsite() {
    // Здесь вызов `requires_avx2` безопасен, так как  `safe_callsite`
    // сама требует `avx2`.
    requires_avx2();
}

fn unsafe_callsite() {
    // Здесь вызов `requires_avx2` не безопасен, и мы сначала
    // должны удостовериться, что `avx2` доступен.
    if is_x86_feature_detected!("avx2") {
        unsafe { requires_avx2() };
    }
}
```

You can check the [`target_features_11`](https://github.com/rust-lang/rfcs/blob/master/text/2396-target-feature-1.1.md) RFC for more information.

### Debug assertions that pointers are non-null when required for soundness

Компилятор теперь будет вставлять отладочные проверки о том, что указатель не равен null, при выполнении операций чтения и записи ненулевого размера, а также при повторном преобразовании указателя в ссылку. Например, следующий код теперь будет генерировать панику без размотки, когда включены отладочные утверждения:

```rust
let _x = *std::ptr::null::<u8>();
let _x = &*std::ptr::null::<u8>();
```

Trivial examples like this have produced a warning since Rust 1.53.0, the new runtime check will detect these scenarios regardless of complexity.

Эти проверки выполняются только тогда, когда включены отладочные проверки, что означает, что на их надежность **не следует** полагаться. Это также означает, что зависимости, которые были скомпилированы с отключенными отладочными проверка (например, стандартная библиотека), не будут запускать проверки, даже если они вызываются кодом с включенными отладочными проверками.

### `missing_abi` по умолчанию генерирует предупреждение

Пропуск ABI в extern-блоках и функциях (например, `extern {}` и `extern fn`) теперь приведет к появлению предупреждения (через проверку `missing_abi`). Пропуск ABI после ключевого слова `extern` всегда неявно приводил к включению ABI `"C"`. Теперь рекомендуется явно указывать `"C"` ABI (например, `extern "C" {}` и `extern "C" fn`).

You can check the [Explicit Extern ABIs RFC](https://rust-lang.github.io/rfcs/3722-explicit-extern-abis.html) for more information.

### Target deprecation warning for 1.87.0

The tier-2 target `i586-pc-windows-msvc` will be removed in the next version of Rust, 1.87.0. Its difference to the much more popular `i686-pc-windows-msvc` is that it does not require SSE2 instruction support, but Windows 10, the minimum required OS version of all `windows` targets (except the `win7` targets), requires SSE2 instructions itself.

All users currently targeting `i586-pc-windows-msvc` should migrate to `i686-pc-windows-msvc` before the `1.87.0` release.

You can check the [Major Change Proposal](https://github.com/rust-lang/compiler-team/issues/840) for more information.

### Стабилизированные API

- [`{float}::next_down`](https://doc.rust-lang.org/stable/std/primitive.f64.html#method.next_down)
- [`{float}::next_up`](https://doc.rust-lang.org/stable/std/primitive.f64.html#method.next_up)
- [`<[_]>::get_disjoint_mut`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.get_disjoint_mut)
- [`<[_]>::get_disjoint_unchecked_mut`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.get_disjoint_unchecked_mut)
- [`slice::GetDisjointMutError`](https://doc.rust-lang.org/stable/std/slice/enum.GetDisjointMutError.html)
- [`HashMap::get_disjoint_mut`](https://doc.rust-lang.org/std/collections/hash_map/struct.HashMap.html#method.get_disjoint_mut)
- [`HashMap::get_disjoint_unchecked_mut`](https://doc.rust-lang.org/std/collections/hash_map/struct.HashMap.html#method.get_disjoint_unchecked_mut)
- [`NonZero::count_ones`](https://doc.rust-lang.org/stable/std/num/struct.NonZero.html#method.count_ones)
- [`Vec::pop_if`](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.pop_if)
- [`sync::Once::wait`](https://doc.rust-lang.org/stable/std/sync/struct.Once.html#method.wait)
- [`sync::Once::wait_force`](https://doc.rust-lang.org/stable/std/sync/struct.Once.html#method.wait_force)
- [`sync::OnceLock::wait`](https://doc.rust-lang.org/stable/std/sync/struct.OnceLock.html#method.wait)

Следующие API теперь можно использовать в контексте `const`:

- [`hint::black_box`](https://doc.rust-lang.org/stable/std/hint/fn.black_box.html)
- [`io::Cursor::get_mut`](https://doc.rust-lang.org/stable/std/io/struct.Cursor.html#method.get_mut)
- [`io::Cursor::set_position`](https://doc.rust-lang.org/stable/std/io/struct.Cursor.html#method.set_position)
- [`str::is_char_boundary`](https://doc.rust-lang.org/stable/std/primitive.str.html#method.is_char_boundary)
- [`str::split_at`](https://doc.rust-lang.org/stable/std/primitive.str.html#method.split_at)
- [`str::split_at_checked`](https://doc.rust-lang.org/stable/std/primitive.str.html#method.split_at_checked)
- [`str::split_at_mut`](https://doc.rust-lang.org/stable/std/primitive.str.html#method.split_at_mut)
- [`str::split_at_mut_checked`](https://doc.rust-lang.org/stable/std/primitive.str.html#method.split_at_mut_checked)

### Прочие изменения

Проверьте всё, что изменилось в [Rust](https://github.com/rust-lang/rust/releases/tag/1.86.0), [Cargo](https://doc.rust-lang.org/nightly/cargo/CHANGELOG.html#cargo-186-2025-04-03) и [Clippy](https://github.com/rust-lang/rust-clippy/blob/master/CHANGELOG.md#rust-186).

## Кто работал над 1.86.0

Многие люди собрались вместе, чтобы создать Rust 1.86.0. Без вас мы бы не справились. [Спасибо!](https://thanks.rust-lang.org/rust/1.86.0/)
