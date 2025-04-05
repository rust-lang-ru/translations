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

В этом выпуске появилась долгожданная функция — возможность приводить объекты трейта к родительскому. Если у трейта есть [супертрейт](https://doc.rust-lang.org/reference/items/traits.html#supertraits), вы можете привести ссылку на указанный трейт-объект к ссылке на трейт-объект супертрейта:

```rust
trait Trait: Supertrait {}
trait Supertrait {}

fn upcast(x: &dyn Trait) -> &dyn Supertrait {
    x
}
```

То же самое будет работать с любым другим типом (умного) указателя, например `Arc<dyn Trait> -> Arc<dyn Supertrait>` или `*const dyn Trait -> *const dyn Supertrait`.

Раньше это потребовало бы обходного пути в виде метода `upcast` в самом `Trait`, например `fn as_supertrait(&self) -> &dyn Supertrait` — и это работало бы только для одного вида ссылки/указателя. Такие обходные пути больше не нужны.

Обратите внимание, что это означает, что необработанные указатели на трейт-объекты содержат нетривиальный инвариант: "утёкший" в безопасный код необработанный указатель на трейт-объект с недопустимой виртуальной таблицей  может привести к неопределённому поведению. Пока нет уверенности в том, приведет ли временное создание такого необработанного указателя в хорошо контролируемых обстоятельствах к немедленному неопределённому поведению или нет, поэтому в коде нужно воздержаться от создания таких указателей при любых условиях (и Miri обеспечивает это).

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

[Дополнительную информацию о преобразовании типов можно найти в Rust reference](https://doc.rust-lang.org/reference/type-coercions.html#unsized-coercions).

### `HashMap` и срезы теперь поддерживают изменяемую индексацию нескольких элементов

Анализатор заимствований предотвращает одновременное использование ссылок, полученных в результате повторных вызовов методов `get_mut`. Для безопасной поддержки этого шаблона стандартная библиотека теперь предоставляет метод `get_disjoint_mut` для срезов и `HashMap` для одновременного извлечения изменяемых ссылок на несколько элементов. Вы можете изучить следующий пример, взятый из документации API для [`slice::get_disjoint_mut`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.get_disjoint_mut):

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

Ранее только `unsafe` функции могли быть помечены атрибутом `#[target_feature]`, поскольку такие функции нецелесообразно вызывать без включения целевой функции. Этот релиз стабилизирует функцию `target_feature_11`, позволяя помечать *безопасные* функции атрибутом `#[target_feature]`.

Безопасные функции, отмеченные атрибутом `#[target_feature]`, могут быть безопасно вызваны только из других функций, отмеченных атрибутом `#[target_feature]`. Однако их нельзя передать функциям, принимающим обобщения, ограниченные чертами `Fn*`. Также они поддерживают только приведение к указателям функций внутри функций, отмеченных атрибутом `target_feature`.

Внутри функций, не отмеченных атрибутом целевой функции, они могут быть вызваны внутри `unsafe` блока, однако вызывающая сторона несёт ответственность за обеспечение доступности целевой функции.

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

Более подробную информацию можно найти в RFC [`target_features_11`](https://github.com/rust-lang/rfcs/blob/master/text/2396-target-feature-1.1.md).

### Отладка утверждений о том, что указатели не равны нулю, когда это необходимо для корректности

Компилятор теперь будет вставлять отладочные проверки о том, что указатель не равен null, при выполнении операций чтения и записи ненулевого размера, а также при повторном преобразовании указателя в ссылку. Например, следующий код теперь будет генерировать панику без размотки, когда включены отладочные утверждения:

```rust
let _x = *std::ptr::null::<u8>();
let _x = &*std::ptr::null::<u8>();
```

Такие тривиальные примеры, как этот, вызывали предупреждение, начиная с Rust 1.53.0. Новая же проверка во время выполнения обнаружит такие сценарии независимо от их сложности.

Эти проверки выполняются только тогда, когда включены отладочные проверки. Это означает, что на их надёжность **не следует** полагаться, а также что зависимости, которые были скомпилированы с отключёнными отладочными проверками (например, стандартная библиотека), не будут запускать проверки, даже если они вызываются кодом с включёнными отладочными проверками.

### `missing_abi` по умолчанию генерирует предупреждение

Пропуск ABI в extern-блоках и функциях (например, `extern {}` и `extern fn`) теперь приведёт к появлению предупреждения (через проверку `missing_abi`). Пропуск ABI после ключевого слова `extern` всегда неявно приводил к включению ABI `"C"`. Теперь рекомендуется явно указывать `"C"` ABI (например, `extern "C" {}` и `extern "C" fn`).

Более подробную информацию можно найти в [документе RFC «Explicit Extern ABIs»](https://rust-lang.github.io/rfcs/3722-explicit-extern-abis.html).

### Предупреждение об устаревании таргета для 1.87.0

tier-2 `i586-pc-windows-msvc` будет удален в следующей версии Rust 1.87.0. Его отличие от гораздо более популярного `i686-pc-windows-msvc` заключается в том, что он не требует поддержки инструкций SSE2, но Windows 10 — минимально необходимая версия ОС для всех целевых объектов `windows` (кроме целевых объектов `win7`) — сама требует инструкций SSE2.

Всем пользователям, которые в настоящее время используют `i586-pc-windows-msvc`, следует перейти на `i686-pc-windows-msvc` до выхода версии `1.87.0`.

Более подробную информацию можно найти [в Предложении о крупных изменениях](https://github.com/rust-lang/compiler-team/issues/840).

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
