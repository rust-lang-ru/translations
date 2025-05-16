Rust 1.87.0 и 10 лет Rust!

[extra] release = true +++

Команда Rust [празднует 10-летие Rust](https://2025.rustweek.org/celebration/) в Утрехте, Нидерланды, и рада сообщить о новой версии языка — 1.87.0!

![placeholder](https://placehold.co/800x500)

Сегодняшний день релиза выпал на 10-летний юбилей выхода [Rust 1.0](https://blog.rust-lang.org/2015/05/15/Rust-1.0/)!

Спасибо мириадам участников, кто работал или работает над Rust. Выпьем за ещё многие десятилетия впереди! 🎉

---

Как обычно, новая версия включает в себя все изменения, которые были внесены в бета-версию за последние шесть недель согласно последовательному и регулярному циклу выпуска. Мы следуем ему, начиная с Rust 1.0.

Если у вас есть предыдущая версия Rust, установленная через `rustup`, то для обновления до версии 1.87.0 вам достаточно выполнить команду:

```
$ rustup update stable
```

Если у вас ещё не установлен `rustup`, вы можете установить его с [соответствующей страницы](https://www.rust-lang.org/install.html) нашего веб-сайта, а также посмотреть [подробные примечания к выпуску](https://doc.rust-lang.org/stable/releases.html#version-1870-2025-04-03) на GitHub.

Если вы хотите помочь нам протестировать будущие выпуски, вы можете использовать канал beta (`rustup default beta`) или nightly (`rustup default nightly`). Пожалуйста, [сообщайте](https://github.com/rust-lang/rust/issues/new/choose) обо всех встреченных вами ошибках.

## Что стабилизировано в 1.87.0

### Анонимные конвееры

1.87 даёт доступ из стандартной библиотеки к анонимным каналам, включая интеграцию с методами ввода/вывода `std::process::Command`. Например, объединить stdout и stderr в один поток теперь относительно просто, в то время как раньше необходимо было использовать разные потоки или платформо-специфические функции.

```rust
use std::process::Command;
use std::io::Read;

let (mut recv, send) = std::io::pipe()?;

let mut command = Command::new("path/to/bin")
    // И stdout, и stderr теперь пишут в один канал.
    .stdout(send.try_clone()?)
    .stderr(send)
    .spawn()?;

let mut output = Vec::new();
recv.read_to_end(&mut output)?;

// Обратите внимание, что что мы читаем из канала до завершения процесса, для исключения
// переполнения буфера ОС, если программа создаст очень много вывода.
assert!(command.wait()?.success());
```

### Безопасные архитектурные интринсики

Большинство интринсиков из `std::arch`, которые небезопасны только из-за того, что требуют включения таргет фич, теперь могут быть вызваны из безопасного кода, в котором эти фичи включены. Например, следующая программа, реализующая суммирование элементов массива с использованием ручных интринсиков, теперь использует безопасный код в основном цикле.

```rust
#![forbid(unsafe_op_in_unsafe_fn)]

use std::arch::x86_64::*;

fn sum(slice: &[u32]) -> u32 {
    #[cfg(target_arch = "x86_64")]
    {
        if is_x86_feature_detected!("avx2") {
            // БЕЗОПАСНОСТЬ: Во время работы мы удостоверились, что необходимая функциональность присутствует,
            // так что вызывать эту функицю безопасно.
            return unsafe { sum_avx2(slice) };
        }
    }

    slice.iter().sum()
}

#[target_feature(enable = "avx2")]
#[cfg(target_arch = "x86_64")]
fn sum_avx2(slice: &[u32]) -> u32 {
    // БЕЗОПАСНОСТЬ: __m256i и u32 одинаково валидны.
    let (prefix, middle, tail) = unsafe { slice.align_to::<__m256i>() };
    
    let mut sum = prefix.iter().sum::<u32>();
    sum += tail.iter().sum::<u32>();
    
    // Основной цикл теперь полностью состоит из безопасного кода, потому что интринсики, требующие таргет фичу (avx2),
    // упакованы в саму функцию.
    let mut base = _mm256_setzero_si256();
    for e in middle.iter() {
        base = _mm256_add_epi32(base, *e);
    }
    
    // БЕЗОПАСНОСТЬ: __m256i и u32 одинаково валидны.
    let base: [u32; 8] = unsafe { std::mem::transmute(base) };
    sum += base.iter().sum::<u32>();
    
    sum
}
```

### `asm!` прыжки в Rust-код

Встроенный ассемблер (`asm!`) теперь может прыгать в помеченные участки Rust-кода. Это делает более гибким низкоуровневое программирование — например, реализацию оптимизированного контроля управления в ядрах ОС или более эффективное взаимодействие с железом.

- Макрос `asm!` теперь поддерживает операнд `label`, который работает как переход к метке
- Метка должна быть блочным выражением с возвращаемым типом `()` или `!`
- Блок выполняется, когда на него совершается прыжок. Выполнение продолжается после блока `asm!`.
- Использование операндов `output` и `label` в одном вызове `asm!` остаётся [unstable](https://github.com/rust-lang/rust/issues/119364).

```rust
unsafe {
    asm!(
        "jmp {}",
        label {
            println!("Выскочил из asm!");
        }
    );
}
```

Больше информации можно найти в [reference](https://doc.rust-lang.org/nightly/reference/inline-assembly.html#r-asm.operand-type.supported-operands.label).

### Прецизионный захват (`+ use<...>`) в `impl Trait` в объявлении трейтов

В этом выпуске стабилизировано указание конкретных захваченных обобщённых типов и времён жизни в объявлениях трейтов с использованием возвращаемых типов `impl Trait`. Благодаря чему теперь можно использовать эту функцию в определениях трейтов, расширяя возможности стабилизации для функций, не связанных с трейтами, в [1.82](https://blog.rust-lang.org/2024/10/17/Rust-1.82.0/#precise-capturing-use-syntax).

Несколько примеров:

```rust
trait Foo {
    fn method<'a>(&'a self) -> impl Sized;
    
    // ... преобразуется во что-то вроде:
    type Implicit1<'a>: Sized;
    fn method_desugared<'a>(&'a self) -> Self::Implicit1<'a>;
    
    // ... а при прецезионном захвате ...
    fn precise<'a>(&'a self) -> impl Sized + use<Self>;
    
    // ... преобразуется во что-то подобное:
    type Implicit2: Sized;
    fn precise_desugared<'a>(&'a self) -> Self::Implicit2;
}
```

### Стабилизированные API

- [`Vec::extract_if`](https://doc.rust-lang.org/stable/std/vec/struct.Vec.html#method.extract_if)
- [`vec::ExtractIf`](https://doc.rust-lang.org/stable/std/vec/struct.ExtractIf.html)
- [`LinkedList::extract_if`](https://doc.rust-lang.org/stable/std/collections/struct.LinkedList.html#method.extract_if)
- [`linked_list::ExtractIf`](https://doc.rust-lang.org/stable/std/collections/linked_list/struct.ExtractIf.html)
- [`<[T]>::split_off`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.split_off)
- [`<[T]>::split_off_mut`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.split_off_mut)
- [`<[T]>::split_off_first`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.split_off_first)
- [`<[T]>::split_off_first_mut`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.split_off_first_mut)
- [`<[T]>::split_off_last`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.split_off_last)
- [`<[T]>::split_off_last_mut`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.split_off_last_mut)
- [`String::extend_from_within`](https://doc.rust-lang.org/stable/alloc/string/struct.String.html#method.extend_from_within)
- [`os_str::Display`](https://doc.rust-lang.org/stable/std/ffi/os_str/struct.Display.html)
- [`OsString::display`](https://doc.rust-lang.org/stable/std/ffi/struct.OsString.html#method.display)
- [`OsStr::display`](https://doc.rust-lang.org/stable/std/ffi/struct.OsStr.html#method.display)
- [`io::pipe`](https://doc.rust-lang.org/stable/std/io/fn.pipe.html)
- [`io::PipeReader`](https://doc.rust-lang.org/stable/std/io/struct.PipeReader.html)
- [`io::PipeWriter`](https://doc.rust-lang.org/stable/std/io/struct.PipeWriter.html)
- [`impl From<PipeReader> for OwnedHandle`](https://doc.rust-lang.org/stable/std/os/windows/io/struct.OwnedHandle.html#impl-From%3CPipeReader%3E-for-OwnedHandle)
- [`impl From<PipeWriter> for OwnedHandle`](https://doc.rust-lang.org/stable/std/os/windows/io/struct.OwnedHandle.html#impl-From%3CPipeWriter%3E-for-OwnedHandle)
- [`impl From<PipeReader> for Stdio`](https://doc.rust-lang.org/stable/std/process/struct.Stdio.html)
- [`impl From<PipeWriter> for Stdio`](https://doc.rust-lang.org/stable/std/process/struct.Stdio.html#impl-From%3CPipeWriter%3E-for-Stdio)
- [`impl From<PipeReader> for OwnedFd`](https://doc.rust-lang.org/stable/std/os/fd/struct.OwnedFd.html#impl-From%3CPipeReader%3E-for-OwnedFd)
- [`impl From<PipeWriter> for OwnedFd`](https://doc.rust-lang.org/stable/std/os/fd/struct.OwnedFd.html#impl-From%3CPipeWriter%3E-for-OwnedFd)
- [`Box<MaybeUninit<T>>::write`](https://doc.rust-lang.org/stable/std/boxed/struct.Box.html#method.write)
- [`impl TryFrom<Vec<u8>> for String`](https://doc.rust-lang.org/stable/std/string/struct.String.html#impl-TryFrom%3CVec%3Cu8%3E%3E-for-String)
- [`<*const T>::offset_from_unsigned`](https://doc.rust-lang.org/stable/std/primitive.pointer.html#method.offset_from_unsigned)
- [`<*const T>::byte_offset_from_unsigned`](https://doc.rust-lang.org/stable/std/primitive.pointer.html#method.byte_offset_from_unsigned)
- [`<*mut T>::offset_from_unsigned`](https://doc.rust-lang.org/stable/std/primitive.pointer.html#method.offset_from_unsigned-1)
- [`<*mut T>::byte_offset_from_unsigned`](https://doc.rust-lang.org/stable/std/primitive.pointer.html#method.byte_offset_from_unsigned-1)
- [`NonNull::offset_from_unsigned`](https://doc.rust-lang.org/stable/std/ptr/struct.NonNull.html#method.offset_from_unsigned)
- [`NonNull::byte_offset_from_unsigned`](https://doc.rust-lang.org/stable/std/ptr/struct.NonNull.html#method.byte_offset_from_unsigned)
- [`<uN>::cast_signed`](https://doc.rust-lang.org/stable/std/primitive.usize.html#method.cast_signed)
- [`NonZero::<uN>::cast_signed`](https://doc.rust-lang.org/stable/std/num/struct.NonZero.html#method.cast_signed-5)
- [`<iN>::cast_unsigned`](https://doc.rust-lang.org/stable/std/primitive.isize.html#method.cast_unsigned)
- [`NonZero::<iN>::cast_unsigned`](https://doc.rust-lang.org/stable/std/num/struct.NonZero.html#method.cast_unsigned-5)
- [`<uN>::is_multiple_of`](https://doc.rust-lang.org/stable/std/primitive.usize.html#method.is_multiple_of)
- [`<uN>::unbounded_shl`](https://doc.rust-lang.org/stable/std/primitive.usize.html#method.unbounded_shl)
- [`<uN>::unbounded_shr`](https://doc.rust-lang.org/stable/std/primitive.usize.html#method.unbounded_shr)
- [`<iN>::unbounded_shl`](https://doc.rust-lang.org/stable/std/primitive.isize.html#method.unbounded_shl)
- [`<iN>::unbounded_shr`](https://doc.rust-lang.org/stable/std/primitive.isize.html#method.unbounded_shr)
- [`<iN>::midpoint`](https://doc.rust-lang.org/stable/std/primitive.isize.html#method.midpoint)
- [`<str>::from_utf8`](https://doc.rust-lang.org/stable/std/primitive.str.html#method.from_utf8)
- [`<str>::from_utf8_mut`](https://doc.rust-lang.org/stable/std/primitive.str.html#method.from_utf8_mut)
- [`<str>::from_utf8_unchecked`](https://doc.rust-lang.org/stable/std/primitive.str.html#method.from_utf8_unchecked)
- [`<str>::from_utf8_unchecked_mut`](https://doc.rust-lang.org/stable/std/primitive.str.html#method.from_utf8_unchecked_mut)

Следующие API теперь можно использовать в контексте `const`:

- [`core::str::from_utf8_mut`](https://doc.rust-lang.org/stable/std/str/fn.from_utf8_mut.html)
- [`<[T]>::copy_from_slice`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.copy_from_slice)
- [`SocketAddr::set_ip`](https://doc.rust-lang.org/stable/std/net/enum.SocketAddr.html#method.set_ip)
- [`SocketAddr::set_port`](https://doc.rust-lang.org/stable/std/net/enum.SocketAddr.html#method.set_port),
- [`SocketAddrV4::set_ip`](https://doc.rust-lang.org/stable/std/net/struct.SocketAddrV4.html#method.set_ip)
- [`SocketAddrV4::set_port`](https://doc.rust-lang.org/stable/std/net/struct.SocketAddrV4.html#method.set_port),
- [`SocketAddrV6::set_ip`](https://doc.rust-lang.org/stable/std/net/struct.SocketAddrV6.html#method.set_ip)
- [`SocketAddrV6::set_port`](https://doc.rust-lang.org/stable/std/net/struct.SocketAddrV6.html#method.set_port)
- [`SocketAddrV6::set_flowinfo`](https://doc.rust-lang.org/stable/std/net/struct.SocketAddrV6.html#method.set_flowinfo)
- [`SocketAddrV6::set_scope_id`](https://doc.rust-lang.org/stable/std/net/struct.SocketAddrV6.html#method.set_scope_id)
- [`char::is_digit`](https://doc.rust-lang.org/stable/std/primitive.char.html#method.is_digit)
- [`char::is_whitespace`](https://doc.rust-lang.org/stable/std/primitive.char.html#method.is_whitespace)
- [`<[[T; N]]>::as_flattened`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.as_flattened)
- [`<[[T; N]]>::as_flattened_mut`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.as_flattened_mut)
- [`String::into_bytes`](https://doc.rust-lang.org/stable/std/string/struct.String.html#method.into_bytes)
- [`String::as_str`](https://doc.rust-lang.org/stable/std/string/struct.String.html#method.as_str)
- [`String::capacity`](https://doc.rust-lang.org/stable/std/string/struct.String.html#method.capacity)
- [`String::as_bytes`](https://doc.rust-lang.org/stable/std/string/struct.String.html#method.as_bytes)
- [`String::len`](https://doc.rust-lang.org/stable/std/string/struct.String.html#method.len)
- [`String::is_empty`](https://doc.rust-lang.org/stable/std/string/struct.String.html#method.is_empty)
- [`String::as_mut_str`](https://doc.rust-lang.org/stable/std/string/struct.String.html#method.as_mut_str)
- [`String::as_mut_vec`](https://doc.rust-lang.org/stable/std/string/struct.String.html#method.as_mut_vec)
- [`Vec::as_ptr`](https://doc.rust-lang.org/stable/std/vec/struct.Vec.html#method.as_ptr)
- [`Vec::as_slice`](https://doc.rust-lang.org/stable/std/vec/struct.Vec.html#method.as_slice)
- [`Vec::capacity`](https://doc.rust-lang.org/stable/std/vec/struct.Vec.html#method.capacity)
- [`Vec::len`](https://doc.rust-lang.org/stable/std/vec/struct.Vec.html#method.len)
- [`Vec::is_empty`](https://doc.rust-lang.org/stable/std/vec/struct.Vec.html#method.is_empty)
- [`Vec::as_mut_slice`](https://doc.rust-lang.org/stable/std/vec/struct.Vec.html#method.as_mut_slice)
- [`Vec::as_mut_ptr`](https://doc.rust-lang.org/stable/std/vec/struct.Vec.html#method.as_mut_ptr)

### Удаление таргета `i586-pc-windows-msvc`

Таргет `i586-pc-windows-msvc` удалён из Tier 2. Отличие `i586-pc-windows-msvc` от более популярного таргета `i686-pc-windows-msvc` из Tier 1 в том, что `i586-pc-windows-msvc` не требует поддержки инструкций SSE2. Но Windows 10, минимально допустимая версия ОС для всех `windows`-таргетов (за исключением `win7`), сама по себе требует инструкций SSE2.

Все пользователи, использующие в качестве целевой платформы `i586-pc-windows-msvc`, должны мигрировать на `i686-pc-windows-msvc`.

Для большей информации вы можете изучить [Major Change Proposal](https://github.com/rust-lang/compiler-team/issues/840).

### Прочие изменения

Проверьте всё, что изменилось в [Rust](https://github.com/rust-lang/rust/releases/tag/1.87.0), [Cargo](https://doc.rust-lang.org/nightly/cargo/CHANGELOG.html#cargo-187-2025-05-15) и [Clippy](https://github.com/rust-lang/rust-clippy/blob/master/CHANGELOG.md#rust-187).

## Кто работал над 1.87.0

Многие люди собрались вместе, чтобы создать Rust 1.87.0. Без вас мы бы не справились. [Спасибо!](https://thanks.rust-lang.org/rust/1.87.0/)
