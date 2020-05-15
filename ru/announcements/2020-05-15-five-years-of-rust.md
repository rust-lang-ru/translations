---
title: "Пять лет Rust"
author: "The Rust Core Team"
---

В этом бардаке, который сейчас происходит в мире, легко забыть, что прошло уже пять лет с
выпуска 1.0 в 2015м году! Rust за эти пять лет сильно изменился,
так что мы хотели бы вспомнить о работе всех участников сообщества,
начиная с момента стабилизации языка.

Напомним, если кто забыл, что Rust это язык программирования общего назначения,
который обладает средствами, позволяющими строить надёжное и эффективное
программное обеспечение. Rust может быть использован в любом месте: от ядра
вашей операционной системы до вашего следующего web-приложения. Этот язык
простроен полностью участниками открытого многоликого сообщества, в основном
волонтёрами, кто щедро делился своим временем и знаниями для того, чтобы помочь
сделать Rust таким, какой он есть сейчас.

## Основные изменения с версии 1.0

#### 2015

**[1.2] — Параллельная кодогенерация:** уменьшение времени компиляции всегда являлось
главной темой в каждом выпуске Rust, и сейчас трудно представить, что когда-то
был короткий период времени, когда Rust вообще не имел параллельной кодогенерации.

[1.2]:https://blog.rust-lang.org/2015/08/06/Rust-1.2.html

**[1.3] — The Rustonomicon:** наш первый выпуск фантастической
книги "The Rustonomicon", книги, которая исследует тёмную сторону языка Rust,
Unsafe Rust и смежные темы. Эта книга стала прекрасным источником сведений для
любого, кто хочет изучить и понять самые трудные аспекты языка.

[1.3]:https://blog.rust-lang.org/2015/09/17/Rust-1.3.html

**[1.4] — Поддержка Windows MSVC уровня 1:** продвижение платформы на
уровень поддержки 1 принесло нативную поддержку 64-битной Windows с инструментарием
Microsoft Visual C++ (MSVC). До 1.4 вам нужно было устанавливать MinGW (порт
окружения GNU под Windows) чтобы использовать и компилировать ваши программы на
Rust. Поддержка Windows стала одним из самых больших улучшений Rust за эти пять
лет. Лишь недавно Microsoft [анонсировала публичный пре-релиз официальной поддержки Rust для WinRT API!]
И наконец, сейчас стало значительно легче строить высококачественные и
кроссплатформенные приложения, нежели это было когда-либо ранее.

[1.4]:https://blog.rust-lang.org/2015/10/29/Rust-1.4.html
[анонсировала публичный пре-релиз официальной поддержки Rust для WinRT API!]:https://blogs.windows.com/windowsdeveloper/2020/04/30/rust-winrt-public-preview/

**[1.5](https://blog.rust-lang.org/2015/12/10/Rust-1.5.html) — Cargo Install:** добавление возможности собирать
исполняемые файлы с помощью компилятора Rust вместе с поддержкой предустановленных
дополнений cargo породило целую экосистему приложений, утилит и инструментов
для разработчиков, которые сообщество обожает и от которых зависит. Cargo теперь
поддерживает довольно много команд, которые сначала были просто плагинами,
сделанными участниками сообщества и выложенными на crates.io!

#### 2016


**[1.6](https://blog.rust-lang.org/2016/01/21/Rust-1.6.html) — Libcore:** `libcore` - это подмножество
стандартной библиотеки, содержащее только API, которое не требует функций выделения
памяти или функций уровня операционной системы. Стабилизация `libcore` дала
возможность компилировать Rust без выделения памяти или зависимости от
операционной системы, что стало одним из первых основных шагов к поддержке
Rust для разработки встраиваемых систем.

**[1.10](https://blog.rust-lang.org/2016/07/07/Rust-1.10.html) — Динамические библиотеки с C ABI:**
крейт типа `cdylib` позволяет компилировать Rust в виде динамической C-библиотеки,
что позволяет встраивать проекты Rust в любую систему, которая поддерживает ABI языка C.
Некоторые из самых больших историй успеха Rust среди пользователей - это возможность
написать небольшую критическую часть системы на Rust и легко интегрировать в большую
кодовую базу. И теперь это стало проще, чем когда-либо.

**[1.12](https://blog.rust-lang.org/2016/09/29/Rust-1.12.html) - Cargo Workspaces:** позволяют
организовать несколько проектов Rust и совместно использовать один и тот же lock-файл.
Рабочие пространства были неоценимы при создании крупных проектов с много-уровневыми
крейтами.

**[1.13](https://blog.rust-lang.org/2016/11/10/Rust-1.13.html) — Оператор `Try`:** первым
основным синтаксическим изменением был оператор `?` или оператор "Try". Он позволяет легко
распространять ошибку по стеку вызовов. Раньше вам приходилось использовать макрос
`try!`, который требовал оборачивать всё выражение каждый раз, когда вы сталкивались с
`Result`, теперь вместо этого можно легко связать методы с помощью `?`.

```rust
try!(try!(expression).method()); // Old
expression?.method()?;           // New
```

**[1.14](https://blog.rust-lang.org/2016/12/22/Rust-1.14.html) — Rustup 1.0:** Rustup - это
менеджер инструментальных средств. Он позволяет беспрепятственно использовать любую
версию Rust или любой его инструментарий. То, что начиналось как скромный скрипт,
стало тем, что персонал по эксплуатации нежно называет *"химера"*. Возможность
обеспечить первоклассное управление версиями компилятора в Linux, macOS, Windows и
десятке целевых платформ была мифом ещё пять лет назад.

#### 2017

**[1.15](https://blog.rust-lang.org/2017/02/02/Rust-1.15.html) — Выводимые процедурные макросы:** выводимые
макросы позволяют создавать мощные, обширные и строго типизированные API без часто
повторяющегося кода. Это была первая стабильная версия Rust с которой стало можно
использовать макросы из библиотек `serde` или `diesel`.

**[1.17](https://blog.rust-lang.org/2017/04/27/Rust-1.17.html) — Rustbuild:** одним из самых больших
улучшений для наших помощников в разработке языка стал переход от начальной системы
сборки на основе `make` на использование Cargo. Благодаря этому участникам и новичкам
`rust-lang/rust` стало намного проще создавать и вносить свой вклад в проект.

**[1.20](https://blog.rust-lang.org/2017/08/31/Rust-1.20.html) — Ассоциированные константы:** ранее
константы могли быть связаны только с модулем. В 1.20 мы стабилизировали ассоциативные
константы для структур, перечислений и, что важно, для типажей. Это упростило
добавление богатых наборов предустановленных значений в типы вашего API, таких
как общие IP-адреса или использующиеся числовые константы.

#### 2018

**[1.24](https://blog.rust-lang.org/2018/02/15/Rust-1.24.html) — Инкрементальная компиляция:** до версии
1.24, когда вы вносили изменения в библиотеку, компилятору приходилось перекомпилировать
весь код. Теперь он стал намного умнее в плане кэширования, насколько это было
возможно, и ему нужно только заново перегенерировать изменения.

**[1.26](https://blog.rust-lang.org/2018/05/10/Rust-1.26.html) - `impl Trait`:** добавление `impl Trait`
дало вам выразительные динамические API с преимуществами и производительностью
статической диспетчеризации.

**[1.28](https://blog.rust-lang.org/2018/08/02/Rust-1.28.html) — Глобальные аллокаторы:** ранее
вы были ограничены использованием аллокатора, предоставленного Rust. Теперь с
помощью API глобального аллокатора можно выбрать аллокатор, который соответствует
вашим потребностям. Это был важный шаг в создании библиотеки `alloc`, другого
подмножества стандартной библиотеки, содержащей только те её части, для которых
требуется аллокатор, например `Vec` или `String`. Теперь стало проще, чем когда-либо,
переиспользовать части стандартной библиотеки на различных системах.

**[1.31](https://blog.rust-lang.org/2018/12/06/Rust-1.31-and-rust-2018.html) — 2018 редакция:** выпуск
2018 редакции стал нашим самым большим выпуском языка после версии 1.0, добавив
изменения синтаксиса и улучшения в код на Rust, написанного полностью обратно
совместимым образом. Это позволяет библиотекам, созданным с разными редакциями,
беспрепятственно работать вместе.

 - **Нелексические времена жизни (NLL)**: огромное улучшение в анализаторе заимствований
    Rust, позволяющее ему принимать более проверяемый безопасный код.
 - **Улучшения в системе модулей**: большие улучшения UX в объявлении и использовании модулей.
 - **Константные функции**: константные функции позволяют запускать и вычислять код Rust во
    время компиляции.
 - **Rustfmt 1.0**: новый инструмент форматирования кода, созданный специально для Rust.
 - **Clippy 1.0**: анализатор Rust для поиска распространённых ошибок. Clippy значительно
    упрощает проверку того, что ваш код не только безопасен, но и корректен.
 - **Rustfix**: со всеми изменениями синтаксиса мы знали, что хотим предоставить инструментарий,
    позволяющий сделать переход максимально простым. Теперь, когда требуются изменения в
    синтаксисе Rust, можно просто выполнить команду `cargo fix` для решения проблем,
    связанной с изменениями синтаксиса.

#### 2019

**[1.34](https://blog.rust-lang.org/2019/04/11/Rust-1.34.0.html) — Альтернативный реестр крейтов:** Поскольку
Rust всё больше и больше используется в производстве, возникает большая потребность
в возможности размещении и использовании проектов в не публичных пространствах. В то время
как в Cargo всегда разрешены зависимости из git-репозиториев, с помощью альтернативных
реестров ваша организация может легко создать и делиться своим собственным реестром
крейтов, которые можно использовать в ваших проектах, как если бы они были на crates.io.

**[1.39](https://blog.rust-lang.org/2019/11/07/Rust-1.39.0.html) — Async/Await:** Стабилизация
ключевых слов `async`/`await` для обработки `Future` была одной из главных вех в
превращении асинхронного программирования в Rust в "объект первого класса". Уже через
шесть месяцев после его выпуска, асинхронное программирование превратилось в
разнообразную и производительную экосистему.

#### 2020

**[1.42](https://blog.rust-lang.org/2020/03/12/Rust-1.42.html) — Шаблоны срезов:** хотя это было и не самое большое изменение, добавление
шаблона `..` (остальное) был долгожданной функцией, которая значительно улучшает
выразительность сопоставления образцов для срезов.

## Диагностики и ошибки

Одна вещь, которую мы не упомянули, это то насколько улучшены сообщения об ошибках
и диагностика в Rust с 1.0. Глядя на старые сообщения об ошибках кажется, что это
теперь другой язык.

Мы выделили несколько примеров, которые лучше всего демонстрируют насколько мы
улучшили сообщения об ошибках. Теперь они показывают пользователям, где они
допустили ошибки, и помогают понять, почему код не работает, а также учат,
как можно их исправить.

##### Первый пример (Типажи)

```rust
use std::io::Write;

fn trait_obj(w: &amp;Write) {
    generic(w);
}

fn generic&lt;W: Write&gt;(_w: &amp;W) {}
```

<div>  1.2.0 Error Message </div>

```
   Compiling error-messages v0.1.0 (file:///Users/usr/src/rust/error-messages)
src/lib.rs:6:5: 6:12 error: the trait `core::marker::Sized` is not implemented for the type `std::io::Write` [E0277]
src/lib.rs:6     generic(w);
                 ^~~~~~~
src/lib.rs:6:5: 6:12 note: `std::io::Write` does not have a constant size known at compile-time
src/lib.rs:6     generic(w);
                 ^~~~~~~
error: aborting due to previous error
Could not compile `error-messages`.

To learn more, run the command again with --verbose.
```

<img alt="A terminal screenshot of the 1.2.0 error message." data-md-type="image" src="https://github.com/ruRust/translations/blob/master/images/2020-05-15-five-years-of-rust/trait-error-1.2.0.png?raw=true">

<div>  1.43.0 Error Message </div>

```
   Compiling error-messages v0.1.0 (/Users/ep/src/rust/error-messages)
error[E0277]: the size for values of type `dyn std::io::Write` cannot be known at compilation time
 --&gt; src/lib.rs:6:13
  |
6 |     generic(w);
  |             ^ doesn't have a size known at compile-time
...
9 | fn generic&lt;W: Write&gt;(_w: &amp;W) {}
  |    ------- -       - help: consider relaxing the implicit `Sized` restriction: `+  ?Sized`
  |            |
  |            required by this bound in `generic`
  |
  = help: the trait `std::marker::Sized` is not implemented for `dyn std::io::Write`
  = note: to learn more, visit &lt;https://doc.rust-lang.org/book/ch19-04-advanced-types.html#dynamically-sized-types-and-the-sized-trait&gt;

error: aborting due to previous error

For more information about this error, try `rustc --explain E0277`.
error: could not compile `error-messages`.

To learn more, run the command again with --verbose.
```

<img alt="A terminal screenshot of the 1.43.0 error message." data-md-type="image" src="https://github.com/ruRust/translations/blob/master/images/2020-05-15-five-years-of-rust/trait-error-1.43.0.png?raw=true">

##### Второй пример (помощь)

```rust
fn main() {
    let s = "".to_owned();
    println!("{:?}", s.find("".to_owned()));
}
```

<div>  1.2.0 Error Message </div>

```
   Compiling error-messages v0.1.0 (file:///Users/ep/src/rust/error-messages)
src/lib.rs:3:24: 3:43 error: the trait `core::ops::FnMut&lt;(char,)&gt;` is not implemented for the type `collections::string::String` [E0277]
src/lib.rs:3     println!("{:?}", s.find("".to_owned()));
                                    ^~~~~~~~~~~~~~~~~~~
note: in expansion of format_args!
&lt;std macros&gt;:2:25: 2:56 note: expansion site
&lt;std macros&gt;:1:1: 2:62 note: in expansion of print!
&lt;std macros&gt;:3:1: 3:54 note: expansion site
&lt;std macros&gt;:1:1: 3:58 note: in expansion of println!
src/lib.rs:3:5: 3:45 note: expansion site
src/lib.rs:3:24: 3:43 error: the trait `core::ops::FnOnce&lt;(char,)&gt;` is not implemented for the type `collections::string::String` [E0277]
src/lib.rs:3     println!("{:?}", s.find("".to_owned()));
                                    ^~~~~~~~~~~~~~~~~~~
note: in expansion of format_args!
&lt;std macros&gt;:2:25: 2:56 note: expansion site
&lt;std macros&gt;:1:1: 2:62 note: in expansion of print!
&lt;std macros&gt;:3:1: 3:54 note: expansion site
&lt;std macros&gt;:1:1: 3:58 note: in expansion of println!
src/lib.rs:3:5: 3:45 note: expansion site
error: aborting due to 2 previous errors
Could not compile `error-messages`.

To learn more, run the command again with --verbose.
```

<img alt="A terminal screenshot of the 1.2.0 error message." data-md-type="image" src="https://github.com/ruRust/translations/blob/master/images/2020-05-15-five-years-of-rust/help-error-1.2.0.png?raw=true">

<div>  1.43.0 Error Message </div>

```
   Compiling error-messages v0.1.0 (/Users/ep/src/rust/error-messages)
error[E0277]: expected a `std::ops::FnMut&lt;(char,)&gt;` closure, found `std::string::String`
 --&gt; src/lib.rs:3:29
  |
3 |     println!("{:?}", s.find("".to_owned()));
  |                             ^^^^^^^^^^^^^
  |                             |
  |                             expected an implementor of trait `std::str::pattern::Pattern&lt;'_&gt;`
  |                             help: consider borrowing here: `&amp;"".to_owned()`
  |
  = note: the trait bound `std::string::String: std::str::pattern::Pattern&lt;'_&gt;` is not satisfied
  = note: required because of the requirements on the impl of `std::str::pattern::Pattern&lt;'_&gt;` for `std::string::String`

error: aborting due to previous error

For more information about this error, try `rustc --explain E0277`.
error: could not compile `error-messages`.

To learn more, run the command again with --verbose.
```

<img alt="A terminal screenshot of the 1.43.0 error message." data-md-type="image" src="https://github.com/ruRust/translations/blob/master/images/2020-05-15-five-years-of-rust/help-error-1.43.0.png?raw=true">

##### Третий пример (Проверка заимствований)

```rust
fn main() {
    let mut x = 7;
    let y = &amp;mut x;

    println!("{} {}", x, y);
}
```

<div>  1.2.0 Error Message </div>

```
   Compiling error-messages v0.1.0 (file:///Users/ep/src/rust/error-messages)
src/lib.rs:5:23: 5:24 error: cannot borrow `x` as immutable because it is also borrowed as mutable
src/lib.rs:5     println!("{} {}", x, y);
                                   ^
note: in expansion of format_args!
&lt;std macros&gt;:2:25: 2:56 note: expansion site
&lt;std macros&gt;:1:1: 2:62 note: in expansion of print!
&lt;std macros&gt;:3:1: 3:54 note: expansion site
&lt;std macros&gt;:1:1: 3:58 note: in expansion of println!
src/lib.rs:5:5: 5:29 note: expansion site
src/lib.rs:3:18: 3:19 note: previous borrow of `x` occurs here; the mutable borrow prevents subsequent moves, borrows, or modification of `x` until the borrow ends
src/lib.rs:3     let y = &amp;mut x;
                              ^
src/lib.rs:6:2: 6:2 note: previous borrow ends here
src/lib.rs:1 fn main() {
src/lib.rs:2     let mut x = 7;
src/lib.rs:3     let y = &amp;mut x;
src/lib.rs:4
src/lib.rs:5     println!("{} {}", x, y);
src/lib.rs:6 }
             ^
error: aborting due to previous error
Could not compile `error-messages`.

To learn more, run the command again with --verbose.
```

<img alt="A terminal screenshot of the 1.2.0 error message." data-md-type="image" src="https://github.com/ruRust/translations/blob/master/images/2020-05-15-five-years-of-rust/borrow-error-1.2.0.png?raw=true">

<div data-md-type="block_html">  1.43.0 Error Message </div>

```
   Compiling error-messages v0.1.0 (/Users/ep/src/rust/error-messages)
error[E0502]: cannot borrow `x` as immutable because it is also borrowed as mutable
 --&gt; src/lib.rs:5:23
  |
3 |     let y = &amp;mut x;
  |             ------ mutable borrow occurs here
4 |
5 |     println!("{} {}", x, y);
  |                       ^  - mutable borrow later used here
  |                       |
  |                       immutable borrow occurs here

error: aborting due to previous error

For more information about this error, try `rustc --explain E0502`.
error: could not compile `error-messages`.

To learn more, run the command again with --verbose.
```

<img alt="A terminal screenshot of the 1.43.0 error message." src="https://github.com/ruRust/translations/blob/master/images/2020-05-15-five-years-of-rust/borrow-error-1.43.0.png?raw=true">

## Цитаты от участников команд

Конечно, мы не можем охватить все произошедшие изменения. Поэтому мы обратились
к некоторым из наших команд и спросили, какими изменениями они больше всего гордятся:

> Для rustdoc значительными изменениями были:
> * Автоматически сгенерированная документация для автоматических реализованных типажей
> * Сам поиск и его оптимизации (последним является преобразование его в JSON)
> * Возможность более точного тестирования документационных блоков "compile_fail, should_panic, allow_fail"
> * Тесты документации теперь генерируются как отдельные двоичные файлы.
>
> — Guillaume Gomez ([rustdoc](https://www.rust-lang.org/governance/teams/rustdoc))

> Rust теперь имеет базовую поддержку IDE! Я полагаю, что между IntelliJ Rust, 
> RLS и rust-analyzer большинство пользователей должны получить «не ужасный» опыт 
> для своего редактора. Пять лет назад под «написанием Rust» подразумевалось 
> использование старой школы Vim/Emacs.
>
> — Алексей Кладов ([IDEs and editors](https://www.rust-lang.org/governance/teams/ides))

> Для меня это было бы следующим: добавление первоклассной поддержки популярных 
> встроенных архитектур и усилия по созданию экосистемы, позволяющей сделать 
> разработку микроконтроллеров с помощью Rust лёгкой, безопасной и в то же время 
> увлекательной.
> 
> — Daniel Egger ([Embedded WG](https://www.rust-lang.org/governance/wgs/embedded))

> Команда релизов работала только с (примерно) начала 2018 года, но даже за это время 
> мы получили ~ 40000 коммитов только в `rust-lang/rust` без каких-либо существенных 
> регрессий в стабильной версии.
> Учитывая, как быстро мы улучшаем компилятор и стандартные библиотеки, я думаю, что 
> это действительно впечатляет (хотя, конечно, команда релизов здесь не является 
> единственным участником). В целом, я обнаружил, что команда релизов проделала 
> отличную работу по управлению масштабированием для увеличения трафика на 
> трекерах, публикуемых PR и т. д.
> 
> — Mark Rousskov ([Release](https://www.rust-lang.org/governance/teams/release))

> За последние 3 года нам удалось превратить интерпретатор [Miri](https://github.com/rust-lang/miri) 
> из экспериментального в практический инструмент для изучения дизайна языка и 
> поиска ошибок в реальном коде, отличное сочетание теории и практики 
> проектирования языков. С теоретической точки зрения у нас есть 
> [Stacked Borrows](https://github.com/rust-lang/unsafe-code-guidelines/blob/master/wip/stacked-borrows.md), 
> на данный момент наиболее конкретное предложение для модели псевдонимов Rust. 
> С практической точки зрения, хотя изначально мы проверяли только несколько 
> ключевых библиотек в Miri, мы недавно увидели большое количество людей, 
> использующих Miri для [поиска и исправления ошибок](https://github.com/rust-lang/miri/#bugs-found-by-miri) в своих 
> собственных крейтах, зависимостях и аналогичное понимание среди участников, 
> улучшающих Miri, например, добавив поддержку доступа к файловой системе, 
> раскручивания стека и многопоточности.
>
> — Ralf Jung ([Miri](https://www.rust-lang.org/governance/teams/compiler#miri))


> Если бы мне пришлось выбрать одну вещь, которой я больше всего горжусь, 
> это была работа над (NLL). Это не только потому, что я думаю, что это сильно 
> повлияло на удобство использования Rust, но и потому, что мы реализовали его, 
> сформировав рабочую группу NLL. Эта рабочая группа привлекла много замечательных 
> участников, многие из которых до сих пор работают над компилятором. Открытый 
> исходный код в лучшем виде!
>
> — Niko Matsakis ([Language](https://www.rust-lang.org/governance/teams/lang))

## Сообщество

Поскольку язык изменился и сильно вырос за последние пять лет, изменилось и его сообщество.
На Rust было написано очень много замечательных проектов, и присутствие Rust в 
промышленной разработке выросло в геометрической прогрессии. Мы хотим поделиться 
статистикой о том, насколько вырос Rust.

 - Rust четыре года подряд признавался [«Самым любимым языком программированием»](https://insights.stackoverflow.com/survey/2019#most-loved-dreaded-and-wanted) 
    по опросам разработчиков Stack Overflow, проводимых в последние года, начиная с 
    версии 1.0.
 - Только в этом году мы обработали более 2,25 петабайта (1PB = 1000 ТБ) различных 
    версий компилятора, инструментария и документации!
 - В то же время мы обработали более 170 ТБ пакетов для примерно 1,8 миллиарда запросов 
    на crates.io, удвоив ежемесячный трафик по сравнению с прошлым годом.
 - Когда Rust версии 1.0 был выпущен, можно было подсчитать количество компаний, 
    которые использовали его в промышленной разработке. Сегодня его используют сотни 
    технологических компаний, а некоторые из крупнейших технологических компаний, таких
    как Apple, Amazon, Dropbox, Facebook, Google и Microsoft, решили использовать 
    Rust в своих проектах для надёжности и производительности.

## Заключение

Очевидно, что мы не смогли охватить все изменения или улучшения в Rust, которые 
произошли с 2015 года. Какими были ваши любимые изменения или новые любимые проекты 
Rust? Не стесняйтесь размещать свой ответ и обсуждение на [нашем форуме Discourse].

[нашем форуме Discourse]:link

Наконец, мы хотели бы поблагодарить всех кто внёс свой вклад в Rust, независимо от 
того, добавили ли вы новую функцию или исправили опечатку, ваша работа сделала 
Rust удивительным сегодня. Мы не можем дождаться, чтобы увидеть как Rust и его 
сообщество будут продолжать расти и меняться, и посмотреть, что вы все будете 
строить с Rust в ближайшее десятилетие!
