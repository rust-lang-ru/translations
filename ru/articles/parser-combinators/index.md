# Изучение комбинаторных парсеров с Rust

*This article teaches the fundamentals of parser combinators to people who are
already Rust programmers. It assumes no other knowledge, and will explain
everything that isn't directly related to Rust, as well as a few of the more
unexpected aspects of using Rust for this purpose. It will not teach you Rust if
you don't already know it, and, if so, it probably also won't teach you parser
combinators very well. If you would like to learn Rust, I recommend the book
[The Rust Programming Language](https://doc.rust-lang.org/book/).*

<!-- more -->

### С точки зрения новичка

There comes a point in the life of every programmer when they find themselves in
need of a parser.

The novice programmer will ask, "what is a parser?"

The intermediate programmer will say, "that's easy, I'll write a regular
expression."

Мастер-программист скажет: «Отойди, я знаю lex и yacc».

The novice has the right idea.

Не то, чтобы регулярные выражения не хороши. (Но, пожалуйста, не пытайтесь писать
сложный парсер как регулярное выражение.) Не то, чтобы не было никакой радости
в использовании мощных инструментов, таких как парсер и генераторы lexer, которые были
отточенные до совершенства на протяжении тысячелетий. Но изучение основ парсеров -
это *удовольствие*. Это также то, что вы будете упускать, если у вас паника при использовании регулярных выражений или генераторов синтаксических анализаторов, оба из которых являются
только абстракциями над реальной проблемой. В сознании новичка, [как
сказал человек](https://en.wikipedia.org/wiki/Shunry%C5%AB_Suzuki#Quotations), там
есть много возможностей. По мнению эксперта, есть только один вариант использования.

В этой статье мы узнаем, как построить парсер с нуля с
использованием метода, распространенного в функциональных языках программирования, известного как *комбинаторный парсер*. У них есть преимущество быть удивительно мощными, как только вы
схватите их основную идею, в то же время оставаясь очень близко к единственному
принципу, поскольку единственными абстракциями здесь будут те, которые вы создадите сами. Топ основных комбинаторов которые вы будете использовать будет состоять из тех, которые вы построите до этого сами.

### Как работать с этой статьёй

Настоятельно рекомендуется, что вы начинаете с нового проекта Rust и вводите фрагменты кода в `src/lib.rs ` по мере чтения (вы можете вставить его непосредственно со страницы,
но вводить его лучше, так как этого автоматически гарантирует вам
прочитайте код полностью). Каждый кусок кода, который вам понадобится, располагается в статье по порядку.  Имейте в виду, что он иногда вводит *измененые*
версии функций, которые вы написали ранее, и что в этих случаях вы
должны заменить старую версию на новую.

Код был написан для `rustc` версии 1.34.0 с использованием edition 2018 года. Вы можете использовать любую версию компилятора, убедитесь, что вы используете выпуск с поддержкой edition 2018
(проверьте, что ваш `Cargo.toml` содержит `edition = "2018"`). Для данного проекта не нужны внешние зависимости.

Для выполнения тестов, представленных в статье, Как и следовало ожидать, используется `cargo test`.

### The Xcruciating Markup Language

We're going to write a parser for a simplified version of XML. It looks like
this:

```xml
<parent-element>
  <single-element attribute="value" />
</parent-element>
```

XML-элементы открываются символом  `<` и идентификатором, состоящим из буквы
далее следует любое количество букв, цифр и `-`. За этим следуют некоторые
пробелы и необязательный список пар атрибутов: другой идентификатор как
определено ранее, за которым следуют `=` и строка в двойных кавычках. Наконец, есть
это либо закрывающий символ `/>` для обозначения одного элемента без детей, либо `>`
для обозначения существования следующей последовательности дочерних элементов, и, наконец закрывающий тег, начинающийся с `</`, за которым следует идентификатор, который должен соответствовать
открытие тега, и закрывающий `>`.

Это все, что мы будем поддерживать. Ни пространств имен, ни текстовых узлов, и *определенно* нет проверки схемы. Мы даже не будем беспокоиться о 
поддержке экранирования кавычек для этих строк - они начинаются с первого дубля
кавычки, и они заканчиваются на следующем, и все. Если вы хотите двойные кавычки
внутри ваших фактических строк, вы можете принять ваши необоснованные требования где-то
еще.

Мы собираемся разобрать эти элементы в структуру, которая выглядит следующим образом:

```rust
#[derive(Clone, Debug, PartialEq, Eq)]
struct Element {
    name: String,
    attributes: Vec<(String, String)>,
    children: Vec<Element>,
}
```

No fancy types, just a string for a name (that's the identifier at the start of
each tag), attributes as pairs of strings (identifier and value), and a list of
child elements that look exactly the same as the parent.

(If you're typing along, make sure you include those derives. You're going to
need them later.)

### Defining The Parser

Well, then, it's time to write the parser.

Parsing is a process of deriving structure from a stream of data. A parser is
something which teases out that structure.

В дисциплине, которую мы собираемся исследовать, синтаксический анализатор, в его самой простой форме, является функцией, которая принимает некоторый ввод и возвращает либо проанализированный вывод вместе с оставшейся частью ввода, либо ошибку, говорящую "Я не смог разобрать это".

It turns out that's also, in a nutshell, what a parser looks like in its more
complicated forms. You might complicate what the input, the output and the error
all mean, and if you're going to have good error messages you'll need to, but
the parser stays the same: something that consumes input and either produces
some parsed output along with what's left of the input or lets you know it
couldn't parse the input into the output.

Let's write that down as a function type.

```rust
Fn(Input) -> Result<(Input, Output), Error>
```

Более конкретно, в нашем случае мы захотим заполнить типы, чтобы мы получили
что-то вроде этого, потому что то, что мы собираемся сделать, это преобразовать строку в структуру `Element`, и на данный момент мы не хотим вдаваться в тонкости
сообщения об ошибке, поэтому мы просто вернем часть строки, которую мы не смогли
разбрать:

```rust
Fn(&str) -> Result<(&str, Element), &str>
```

Мы используем фрагмент строки, потому что это эффективный указатель на фрагмент строки, и мы можем разрезать его дальше, как нам угодно, «потребляя» входные данные, вырезая проанализированный бит и возвращая остаток вместе с результатом.

Возможно, было бы чище использовать `&[u8]` (фрагмент байтов, соответствующий
символы, если мы ограничиваемся ASCII) в качестве типа ввода, особенно
потому что строковые срезы ведут себя немного иначе, чем большинство срезов - особенно
в том, что вы не можете индексировать их с помощью одного числа `input[0]`, вы должны использовать
фрагмент `input[0..1]`. С другой стороны, у них есть много методов, которые
полезны для разбора строк, которые не имеют срезы байтов.

In fact, in general we're going to be relying on those methods rather than
indexing it like that, because, well, Unicode. In UTF-8, and all Rust strings
are UTF-8, these indexes don't always correspond to single characters, and it's
better for all parties concerned that we ask the standard library to just please
deal with this for us.

### Our First Parser

Давайте попробуем написать парсер, который просто смотрит на первый символ в строке
и решает, является ли это буквой `a`.

```rust
fn the_letter_a(input: &str) -> Result<(&str, ()), &str> {
  match input.chars().next() {
      Some('a') => Ok((&input['a'.len_utf8()..], ())),
      _ => Err(input),
  }
}
```

Во-первых, давайте посмотрим на типы ввода и вывода: мы берем строковый срез как
ввод, как мы уже обсуждали, и мы возвращаем `Result` с `(&str, ())`, либо
тип ошибки `&str`. Пара `(&str, ())` - это интересный момент: как мы
говорили о том, что мы должны вернуть кортеж следующего содержания,
результат разбора и оставшуюся часть ввода. `&str` - это следующий вход, и результат - это просто
тип блока `()`, потому что если этот синтаксический анализатор успешно работает, он может иметь только один
результат (мы нашли букву `a`), и нам не особенно нужно возвращать
буква `a` в этом случае нам просто нужно указать, что нам удалось её найти.

Итак, давайте посмотрим на код самого парсера. Мы начинаем с получения
первого символ ввода: `input.chars().next()`. Мы не шутили по этому поводу
опираясь на стандартную библиотеку, чтобы не получить головной боли с Unicode - мы получаем итератор по символам строки через `chars()`, и просим первый символ из него. Это будет элемент типа  `char`, завернутый в
`Option`, поэтому `Option<char>`, где `None` означает, что мы пытались вытащить `char` из
пустой строки.

Что еще хуже, `char` не обязательно даже то, что вы думаете о нем как о символе Unicode. Это, скорее всего, будет то, что Unicode называет "[grapheme
cluster](http://www.unicode.org/glossary/#grapheme_cluster)," который может
состоять из нескольких `char`, которые на самом деле представляют собой "[scalar
values](http://www.unicode.org/glossary/#unicode_scalar_value)," около двух
уровей вниз от кластеров графем. Однако, этот путь ведёт к безумию, и для наших
целей мы честно даже вряд ли увидим `chars` вне ASCII,
так что давай остановимся на этом.

Мы сопоставляем шаблон на `Some('a')`, что является конкретным результатом, который мы ищем,
и если это соответствует, мы возвращаем наше значение успеха: `Ok((&input['a'.len_utf8()..], ()))`. То есть мы удаляем часть, которую мы только что проанализировали (`'a'`) из среза строки и вернём остальное вместе с нашим анализируемым значением, которое является просто пустым
`()`. Всегда помня о монстре Unicode, мы просим длину для `'a'` в UTF-8 через стандартную библиотеку перед нарезкой,  но никогда, никогда не предполагайте что длина фрагмента равна 1, монстр Unicode.

Если мы не получим любой другой `Some(char)`, или если мы получим `None`, то возвращаем ошибку. Как
вы помните, что наш тип ошибки прямо сейчас просто будет строковым срезом, где разбор не удался, которая является той, которую мы прошли в качестве
`input`. Он не начинался с `a`, так что это наша ошибка. Это не большая ошибка,
но, по крайней мере, это немного лучше, чем просто "что-то где-то не так."

На самом деле нам не нужен этот парсер для разбора XML, но первое, что мы
должны сделать, это искать это открывающий символ `<`, так что нам понадобится что-то очень
похожее. Нам также нужно будет проанализировать `>`,`/` и `=` конкретно,
так, может быть, мы можем сделать функцию, которая строит парсер для символа, который мы хотим?

### Создание пасера

Давайте даже подумаем об этом: давайте напишем функцию, которая создаёт парсер
для статической строки *любой длины*, а не только одного символа. Это даже
проще, потому что фрагмент строки уже является допустимым срезом строки UTF-8, и нам не нужно думать о монстре Unicode.

```rust
fn match_literal(expected: &'static str)
    -> impl Fn(&str) -> Result<(&str, ()), &str>
{
    move |input| match input.get(0..expected.len()) {
        Some(next) if next == expected => {
            Ok((&input[expected.len()..], ()))
        }
        _ => Err(input),
    }
}
```

Now this looks a bit different.

First of all, let's look at the types. Instead of our function looking like a
parser, now it takes our `expected` string as an argument, and *returns*
something that looks like a parser. It's a function that returns a function - in
other words, a *higher order* function. Basically, we're writing a function that
*makes* a function like our `the_letter_a` function from before.

So, instead of doing the work in the function body, we return a closure that
does the work, and that matches our type signature for a parser from previously.

Сопоставление с шаблоном выглядит одинаково, за исключением того, что мы не можем сопоставить наш строковый литерал напрямую, потому что мы не знаем, что именно, поэтому мы используем условие сопоставления, `if next == expected` вместо этого. В остальном он точно такой же, как и раньше, он просто внутри тела замыкания.

### Testing Our Parser

Let's write a test for this to make sure we got it right.

```rust
#[test]
fn literal_parser() {
    let parse_joe = match_literal("Hello Joe!");
    assert_eq!(
        Ok(("", ())),
        parse_joe("Hello Joe!")
    );
    assert_eq!(
        Ok((" Hello Robert!", ())),
        parse_joe("Hello Joe! Hello Robert!")
    );
    assert_eq!(
        Err("Hello Mike!"),
        parse_joe("Hello Mike!")
    );
}
```

Сначала мы создаем парсер: `match_literal("Hello Joe!")`. Это должно потреблять строку `"Hello Joe!"` и вернуть остаток строки, или он должен завершиться с ошибкой и вернуть всю строку.

In the first case, we just feed it the exact string it expects, and we see that
it returns an empty string and the `()` value that means "we parsed the expected
string and you don't really need it returned back to you."

Во втором мы подаем строку `"Hello Joe! Hello Robert!"` и мы видим, что он действительно использует строку `"Hello Joe!"` и возвращает остаток от ввода: `"Hello Robert!"` (ведущий космос и все).

In the third, we feed it some incorrect input, `"Hello Mike!"`, and note that it
does indeed reject the input with an error. Not that Mike is incorrect as a
general rule, he's just not what this parser was looking for.

### A Parser For Something Less Specific

Так что это позволяет нам анализировать `<` , `>` , `=` и даже `</` и `/>`. Мы уже практически закончили!

Следующий после открывающего символа `<` - это имя элемента. Мы не можем сделать это с помощью простого сравнения строк. Но мы *могли бы* сделать это с помощью регулярного выражения ...

…but let's restrain ourselves. It's going to be a regular expression that
would be very easy to replicate in simple code, and we don't really need to pull
in the `regex` crate just for this. Let's see if we can write our own parser for
this using nothing but Rust's standard library.

Напоминая правило для идентификатора имени элемента, оно выглядит следующим образом: один
алфавитный символ, за которым следует ноль или более из любого алфавитного символа,
символ, число или тире `-`.

```rust
fn identifier(input: &str) -> Result<(&str, String), &str> {
    let mut matched = String::new();
    let mut chars = input.chars();

    match chars.next() {
        Some(next) if next.is_alphabetic() => matched.push(next),
        _ => return Err(input),
    }

    while let Some(next) = chars.next() {
        if next.is_alphanumeric() || next == '-' {
            matched.push(next);
        } else {
            break;
        }
    }

    let next_index = matched.len();
    Ok((&input[next_index..], matched))
}
```

As always, we look at the type first. This time we're not writing a function to
build a parser, we're just writing the parser itself, like our first time. The
notable difference here is that instead of a result type of `()`, we're
returning a `String` in the tuple along with the remaining input. This `String`
is going to contain the identifier we've just parsed.

Имея это в виду, сначала мы создаем пустую `String` и называем ее `matched` . Это будет наш результат. Мы также получаем итератор для `input` символов, который мы собираемся начать разделять.

Первый шаг - посмотреть, есть ли символ в начале. Мы вытаскиваем первый символ из итератора и проверяем, является ли он буквой: `next.is_alphabetic()`. Стандартная библиотека Rust, конечно, здесь, чтобы помочь нам с Unicodes - она будет соответствовать буквам в любом алфавите, а не только в ASCII. Если это буква, мы помещаем ее в нашу `matched` `String`, а если нет, то, разумеется, мы не смотрим на идентификатор элемента, поэтому немедленно возвращаемся с ошибкой.

На втором шаге мы продолжаем вытаскивать символы из итератора, `is_alphanumeric()` `String`, пока не найдём тот, который не является `is_alphanumeric()` (это похоже на `is_alphabetic()` за исключением того, что он также соответствует числам в любом алфавите) или тире {code4}'-'{/code4}.

В первый раз, когда мы видим что-то, что не соответствует этим критериям, это означает, что мы закончили синтаксический анализ, поэтому мы вырываемся из цикла и возвращаем `String`, не забывая вырезать фрагмент, который мы использовали из {code2}input{/code2}. Аналогично, если в итераторе заканчиваются символы, это означает, что мы достигли конца ввода.

Стоит отметить, что мы не возвращаемся с ошибкой, когда видим что-то не буквенно-цифровое или тире. У нас уже есть достаточно, чтобы создать действительный идентификатор после того, как мы сопоставим эту первую букву, и вполне нормально, что после разбора нашего идентификатора во входной строке будет больше элементов для анализа, поэтому мы просто прекращаем синтаксический анализ и возвращаем наш результат. Только если мы не можем найти даже эту первую букву, мы на самом деле возвращаем ошибку, потому что в этом случае здесь точно не было идентификатора.

Помните, что структура `Element`, это то  во что мы собираемся разобрать наш XML документ.

```rust
struct Element {
    name: String,
    attributes: Vec<(String, String)>,
    children: Vec<Element>,
}
```

На самом деле мы только что закончили анализатор для первой части, поля `name`. `String` возвращает наш парсер, идет прямо туда. Это также правильный парсер для первой части каждого `attribute`.

Let's test that.

```rust
#[test]
fn identifier_parser() {
    assert_eq!(
        Ok(("", "i-am-an-identifier".to_string())),
        identifier("i-am-an-identifier")
    );
    assert_eq!(
        Ok((" entirely an identifier", "not".to_string())),
        identifier("not entirely an identifier")
    );
    assert_eq!(
        Err("!not at all an identifier"),
        identifier("!not at all an identifier")
    );
}
```

We see that in the first case, the string `"i-am-an-identifier"` is parsed in
its entirety, leaving only the empty string. In the second case, the parser
returns `"not"` as the identifier, and the rest of the string is returned as the
remaining input. In the third case, the parser fails outright because the first
character it finds is not a letter.

### Combinators

Так что теперь мы можем проанализировать открывающий символ `<`, и мы можем проанализировать следующий идентификатор, но нам нужно проанализировать *оба*, чтобы иметь возможность добиться прогресса здесь. Поэтому следующим шагом будет написание другой функции комбинаторного синтаксического анализатора, но той, которая принимает два *синтаксических анализатора в* качестве входных данных и возвращает новый анализатор, который анализирует их обоих по порядку. Другими словами, *комбинатор* парсеров, потому что он объединяет два парсера в новый. Посмотрим, сможем ли мы это сделать.

```rust
fn pair<P1, P2, R1, R2>(parser1: P1, parser2: P2) -> impl Fn(&str) -> Result<(&str, (R1, R2)), &str>
where
    P1: Fn(&str) -> Result<(&str, R1), &str>,
    P2: Fn(&str) -> Result<(&str, R2), &str>,
{
    move |input| match parser1(input) {
        Ok((next_input, result1)) => match parser2(next_input) {
            Ok((final_input, result2)) => Ok((final_input, (result1, result2))),
            Err(err) => Err(err),
        },
        Err(err) => Err(err),
    }
}
```

It's getting slightly complicated here, but you know what to do: start by
looking at the types.

Прежде всего, у нас есть четыре типа переменных: `P1`, `P2`, `R1` и `R2`. Это Parser1, Parser2, Result1 и Result2. `P1` и `P2` являются функциями, и вы заметите, что они следуют хорошо установленному шаблону функций синтаксического анализатора: так же, как и возвращаемое значение, они принимают `&str` в качестве входных данных и возвращают `Result` пары оставшихся входных данных и результата, или ошибку.

Но посмотрите на типы результатов каждой функции: `P1` - это синтаксический анализатор, который выдает `R1` случае успеха, и `P2` также выдает `R2`. И результат последнего синтаксического анализатора, который был возвращен из нашей функции, равен `(R1, R2)`. Таким образом, задача этого синтаксического анализатора состоит в том, чтобы сначала запустить анализатор `P1` на входе, сохранить его результат, затем запустить `P2` на входе, который вернул `P1`, и, если оба из них сработали, мы объединяем два результата в кортеж `(R1, R2)`.

Глядя на код, мы видим, что это именно то, что он делает. Мы начинаем с запуска первого анализатора на входе, затем второго анализатора, затем объединяем два результата в кортеж и возвращаем его. Если какой-либо из этих парсеров завершится ошибкой, мы немедленно вернемся с ошибкой, которую он выдал.

Таким образом, мы должны иметь возможность объединить два наших анализатора, `match_literal` и `identifier`, чтобы фактически проанализировать первый фрагмент нашего первого тега XML. Давайте напишем тест, чтобы увидеть, работает ли он.

```rust
#[test]
fn pair_combinator() {
    let tag_opener = pair(match_literal("<"), identifier);
    assert_eq!(
        Ok(("/>", ((), "my-first-element".to_string()))),
        tag_opener("<my-first-element/>")
    );
    assert_eq!(Err("oops"), tag_opener("oops"));
    assert_eq!(Err("!oops"), tag_opener("<!oops"));
}
```

Кажется, работает! Но посмотрите на этот тип результата: `((), String)`. Очевидно, что нас интересует только правое значение, `String`. Это будет иметь место довольно часто - некоторые из наших синтаксических анализаторов сопоставляют только шаблоны во входных данных, не производя значений, и поэтому их выходные данные можно безопасно игнорировать. Чтобы приспособиться к этому шаблону, мы собираемся использовать наш комбинатор `pair` чтобы написать два других комбинатора: `left`, который отбрасывает результат первого синтаксического анализатора и возвращает только второй, и его противоположное число, `right`, которое мы и хотели бы использовать. Я хотел использовать в нашем тесте выше вместо `pair` - тот, который отбрасывает, что `()` на левой стороне пары и сохраняет только нашу `String` .

### Ведение в Функтор

Но прежде чем мы зайдем так далеко, давайте представим еще один комбинатор, который значительно упростит написание этих двух: `map`.

Этот комбинатор имеет одну цель: изменить тип результата. Например, допустим, у вас есть парсер, который возвращает `((), String)` и вы хотели изменить его, чтобы он возвращал только эту `String`, вы знаете, просто в качестве произвольного примера.

Для этого мы передаем ему функцию, которая знает, как преобразовать исходный тип в новый. В нашем примере это просто: `|(_left, right)| right`. Более обобщенно, это выглядело бы как `Fn(A) -> B` где `A` - исходный тип результата парсера, а `B` - новый.

```rust
fn map<P, F, A, B>(parser: P, map_fn: F) -> impl Fn(&str) -> Result<(&str, B), &str>
where
    P: Fn(&str) -> Result<(&str, A), &str>,
    F: Fn(A) -> B,
{
    move |input| match parser(input) {
        Ok((next_input, result)) => Ok((next_input, map_fn(result))),
        Err(err) => Err(err),
    }
}
```

А что говорят типы? `P` наш парсер. Возвращает `A` при успехе. `F` - это функция, которую мы будем использовать для отображения `P` как наше возвращаемое значение, которое выглядит так же, как `P` за исключением того, что его тип результата - `B` вместо `A`

В коде мы запускаем `parser(input)` и, если он успешен, мы берем `result` и запускаем на нем нашу функцию `map_fn(result)`, превращая `A` в `B`, и это наш конвертированный парсер.

На самом деле, давайте побалуем себя и немного укоротим эту функцию, потому что эта `map` оказывается общим шаблоном, который фактически реализует `Result` :

```rust
fn map<P, F, A, B>(parser: P, map_fn: F) -> impl Fn(&str) -> Result<(&str, B), &str>
where
    P: Fn(&str) -> Result<(&str, A), &str>,
    F: Fn(A) -> B,
{
    move |input|
        parser(input)
            .map(|(next_input, result)| (next_input, map_fn(result)))
}
```

Этот паттерн - то, что называется «функтором» в Haskell и его математическом родственнике, теории категорий. Если у вас есть вещь с типом `A`, и у вас есть доступная функция `map`, вы можете передать функцию из `A` в `B` чтобы превратить ее в то же самое, но с типом `B` в ней, это функтор. Вы видите это во многих местах в Rust, таких как [`Option`](https://doc.rust-lang.org/std/option/enum.Option.html#method.map), [`Result`](https://doc.rust-lang.org/std/result/enum.Result.html#method.map), [`Iterator`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.map) и даже [`Future`](https://docs.rs/futures/0.1.26/futures/future/trait.Future.html#method.map), без явного имени как такового. И тому есть веская причина: вы не можете по-настоящему выразить функтор как обобщенную вещь в системе типов Rust, потому что в ней отсутствуют типы с более высоким родом, но это другая история, поэтому давайте просто отметим, что это функторы, и вам просто нужно искать функцию `map`.

### Время для Типажа

Возможно, вы уже заметили, что мы продолжаем повторять форму сигнатуры типа синтаксического анализатора: `Fn(&str) -> Result<(&str, Output), &str>`. Возможно, вы устали от того, что читаете это полностью, так же, как я пишу, поэтому я думаю, что пришло время представить типаж, сделать вещи немного более читабельными и позволить нам добавить некоторую расширяемость к нашему парсеру.

But first of all, let's make a type alias for that return type we keep using:

```rust
type ParseResult<'a, Output> = Result<(&'a str, Output), &'a str>;
```

So that now, instead of typing that monstrosity out all the time, we can just
type `ParseResult<String>` or similar. We've added a lifetime there, because the
type declaration requires it, but a lot of the time the Rust compiler should be
able to infer it for you. As a rule, try leaving the lifetime out and see if
rustc gets upset, then just put it in if it does.

Время жизни `'a`, в данном случае, относится конкретно к времени жизни *входа* .

Теперь за типаж. Мы также должны указать здесь время жизни, и когда вы используете типаж, обычно всегда требуется время жизни. Это немного лишняя работа, но она превосходит предыдущую версию.

```rust
trait Parser<'a, Output> {
    fn parse(&self, input: &'a str) -> ParseResult<'a, Output>;
}
```

На данный момент у него есть только один метод: метод `parse()`, который должен выглядеть знакомо: он такой же, как функция парсера, которую мы пишем.

Чтобы сделать это еще проще, мы можем реализовать этот типаж для любой функции, которая соответствует сигнатуре парсера:

```rust
impl<'a, F, Output> Parser<'a, Output> for F
where
    F: Fn(&'a str) -> ParseResult<Output>,
{
    fn parse(&self, input: &'a str) -> ParseResult<'a, Output> {
        self(input)
    }
}
```

Таким образом, мы не только можем передавать те же функции, что и ранее, когда парсеры полностью реализуют типаж `Parser`, мы также открываем возможность использовать другие типы в качестве парсеров.

But, more importantly, it saves us from having to type out those function
signatures all the time. Let's rewrite the `map` function to see how it works
out.

```rust
fn map<'a, P, F, A, B>(parser: P, map_fn: F) -> impl Parser<'a, B>
where
    P: Parser<'a, A>,
    F: Fn(A) -> B,
{
    move |input|
        parser.parse(input)
            .map(|(next_input, result)| (next_input, map_fn(result)))
}
```

Здесь следует особо отметить одну вещь: вместо непосредственного вызова синтаксического анализатора как функции, мы теперь должны перейти к `parser.parse(input)`, потому что мы не знаем, является ли тип `P` функцией, мы просто знаем, что он реализует `Parser`, и поэтому мы должны придерживаться интерфейса, который обеспечивает `Parser`. В противном случае тело функции выглядит точно так же, а типы выглядят намного аккуратнее. Существует новый срок службы `'a'` для дополнительного шума, но в целом это довольно значительное улучшение.

If we rewrite the `pair` function in the same way, it's even more tidy now:

```rust
fn pair<'a, P1, P2, R1, R2>(parser1: P1, parser2: P2) -> impl Parser<'a, (R1, R2)>
where
    P1: Parser<'a, R1>,
    P2: Parser<'a, R2>,
{
    move |input| match parser1.parse(input) {
        Ok((next_input, result1)) => match parser2.parse(next_input) {
            Ok((final_input, result2)) => Ok((final_input, (result1, result2))),
            Err(err) => Err(err),
        },
        Err(err) => Err(err),
    }
}
```

Same thing here: the only changes are the tidied up type signatures and the need
to go `parser.parse(input)` instead of `parser(input)`.

Actually, let's tidy up `pair`'s function body too, the same way we did with
`map`.

```rust
fn pair<'a, P1, P2, R1, R2>(parser1: P1, parser2: P2) -> impl Parser<'a, (R1, R2)>
where
    P1: Parser<'a, R1>,
    P2: Parser<'a, R2>,
{
    move |input| {
        parser1.parse(input).and_then(|(next_input, result1)| {
            parser2.parse(next_input)
                .map(|(last_input, result2)| (last_input, (result1, result2)))
        })
    }
}
```

Метод `and_then` в `Result` похож на `map`, за исключением того, что функция отображения не возвращает новое значение для перехода в `Result`, но в целом новый `Result`. Приведенный выше код идентичен эффекту от предыдущей версии выписанной со всеми этими `match` блоками. Мы вернемся к `and_then` позже, но здесь и сейчас давайте реализуем эти `left` и `right` комбинаторы, теперь, когда у нас есть красивая и аккуратная `map` .

### Left And Right

With `pair` and `map` in place, we can write `left` and `right` very succinctly:

```rust
fn left<'a, P1, P2, R1, R2>(parser1: P1, parser2: P2) -> impl Parser<'a, R1>
where
    P1: Parser<'a, R1>,
    P2: Parser<'a, R2>,
{
    map(pair(parser1, parser2), |(left, _right)| left)
}

fn right<'a, P1, P2, R1, R2>(parser1: P1, parser2: P2) -> impl Parser<'a, R2>
where
    P1: Parser<'a, R1>,
    P2: Parser<'a, R2>,
{
    map(pair(parser1, parser2), |(_left, right)| right)
}

```

We use the `pair` combinator to combine the two parsers into a parser for a
tuple of their results, and then we use the `map` combinator to select just the
part of the tuple we want to keep.

Переписав наш тест для первых двух частей нашего элемента тега, теперь он стал немного чище, и в процессе мы получили некоторые важные новые возможности комбинатора синтаксического анализатора.

We have to update our two parsers to use `Parser` and `ParseResult` first,
though. `match_literal` is the more complicated one:

```rust
fn match_literal<'a>(expected: &'static str) -> impl Parser<'a, ()> {
    move |input: &'a str| match input.get(0..expected.len()) {
        Some(next) if next == expected => Ok((&input[expected.len()..], ())),
        _ => Err(input),
    }
}
```

В дополнение к изменению типа возвращаемого значения, мы также должны убедиться, что тип ввода для замыкания - `&'a str` , или rustc расстроится.

For `identifier`, just change the return type and you're done, inference takes
care of the lifetimes for you:

```rust
fn identifier(input: &str) -> ParseResult<String> {
```

And now the test, satisfyingly absent that ungainly `()` in the result.

```rust
#[test]
fn right_combinator() {
    let tag_opener = right(match_literal("<"), identifier);
    assert_eq!(
        Ok(("/>", "my-first-element".to_string())),
        tag_opener.parse("<my-first-element/>")
    );
    assert_eq!(Err("oops"), tag_opener.parse("oops"));
    assert_eq!(Err("!oops"), tag_opener.parse("<!oops"));
}
```

### One Or More

Давайте продолжим разбор этого тега элемента. У нас есть открывающий символ `<` и у нас есть идентификатор. Что дальше? Это должна быть наша первая пара атрибутов.

No, actually, those attributes are optional. We're going to have to find a way
to deal with things being optional.

Нет, подожди, подожди, на самом деле мы должны иметь дело с чем-то еще, *прежде чем* мы доберемся до первой необязательной пары атрибутов: пробел.

Between the end of the element name and the start of the first attribute name
(if there is one), there's a space. We need to deal with that space.

Это еще хуже - нам нужно иметь дело с *одним или несколькими пробелами*, потому что `<element      attribute="value"/>` является допустимым синтаксисом, даже если он немного перегружен пробелами. Поэтому сейчас самое время подумать о том, можем ли мы написать комбинатор, выражающий идею *одного или нескольких* синтаксических анализаторов.

Мы уже имели дело с этим в нашем анализаторе `identifier`, но все это было сделано вручную. Не удивительно, что код для общей идеи не так уж и отличается.

```rust
fn one_or_more<'a, P, A>(parser: P) -> impl Parser<'a, Vec<A>>
where
    P: Parser<'a, A>,
{
    move |mut input| {
        let mut result = Vec::new();

        if let Ok((next_input, first_item)) = parser.parse(input) {
            input = next_input;
            result.push(first_item);
        } else {
            return Err(input);
        }

        while let Ok((next_input, next_item)) = parser.parse(input) {
            input = next_input;
            result.push(next_item);
        }

        Ok((input, result))
    }
}
```

Прежде всего, типом возвращаемого значения парсера, из которого мы строим, является `A`, а типом возврата комбинированного синтаксического анализатора является `Vec<A>` - любое количество `A`.

The code does indeed look very similar to `identifier`. First, we parse the
first element, and if it's not there, we return with an error. Then we parse as
many more elements as we can, until the parser fails, at which point we return
the vector with the elements we collected.

Looking at that code, how easy would it be to adapt it to the idea of *zero* or
more? We just need to remove that first run of the parser:

```rust
fn zero_or_more<'a, P, A>(parser: P) -> impl Parser<'a, Vec<A>>
where
    P: Parser<'a, A>,
{
    move |mut input| {
        let mut result = Vec::new();

        while let Ok((next_input, next_item)) = parser.parse(input) {
            input = next_input;
            result.push(next_item);
        }

        Ok((input, result))
    }
}
```

Let's write some tests to make sure those two work.

```rust
#[test]
fn one_or_more_combinator() {
    let parser = one_or_more(match_literal("ha"));
    assert_eq!(Ok(("", vec![(), (), ()])), parser.parse("hahaha"));
    assert_eq!(Err("ahah"), parser.parse("ahah"));
    assert_eq!(Err(""), parser.parse(""));
}

#[test]
fn zero_or_more_combinator() {
    let parser = zero_or_more(match_literal("ha"));
    assert_eq!(Ok(("", vec![(), (), ()])), parser.parse("hahaha"));
    assert_eq!(Ok(("ahah", vec![])), parser.parse("ahah"));
    assert_eq!(Ok(("", vec![])), parser.parse(""));
}
```

Обратите внимание на разницу между ними: для `one_or_more` поиск пустой строки является ошибкой, потому что он должен видеть хотя бы один случай своего `zero_or_more`, но для {code2}zero_or_more{/code2} пустая строка просто означает нулевой случай, который не является ошибкой.

На этом этапе разумно начать думать о способах обобщения этих двух, потому что один является точной копией другого, с удалением только одного фрагмента. Может быть заманчиво выразить `one_or_more` в терминах `zero_or_more` что-то вроде этого:

```rust
fn one_or_more<'a, P, A>(parser: P) -> impl Parser<'a, Vec<A>>
where
    P: Parser<'a, A>,
{
    map(pair(parser, zero_or_more(parser)), |(head, mut tail)| {
        tail.insert(0, head);
        tail
    })
}
```

Здесь мы сталкиваемся с проблемами Rust, и я даже не имею в виду проблему отсутствия метода `cons` для `Vec`, но я знаю, что каждый программист на Lisp, читающий этот фрагмент кода, думал об этом. Нет, это еще хуже: это владение.

У нас есть этот синтаксический анализатор, поэтому мы не можем дважды передать его в качестве аргумента, компилятор начнет кричать вам, что вы пытаетесь переместить уже перемещенное значение. Так можем ли мы сделать так, чтобы наши комбинаторы брали ссылки? Нет, оказывается, не без столкновения с еще одним целым рядом проблем с проверкой заимствований - и мы не собираемся даже идти туда прямо сейчас. А поскольку эти синтаксические анализаторы являются функциями, они не делают ничего такого простого, чтобы реализовать `Clone`, который бы очень аккуратно спас день, поэтому мы столкнулись с ограничением, которое не позволяет нам легко повторять наши анализаторы в комбинаторах.

Хотя это не обязательно *большая* проблема. Это означает, что мы не можем выразить `one_or_more` с помощью комбинаторов, но оказывается, что эти два обычно являются единственными комбинаторами, которые вам в любом случае нужны, которые имеют тенденцию повторно использовать синтаксические анализаторы, и, если вы хотите по-настоящему вычурно, вы можете написать комбинатор, который принимает `RangeBound` в дополнение к синтаксическому анализатору и повторяет его в соответствии с диапазоном: `range(0..)` для `zero_or_more`, `range(1..)` для `one_or_more`, `range(5..=6)` для ровно пяти или шести, куда бы вас ни {code8}one_or_more{/code8} ваше сердце,

Давайте оставим это как упражнение для читателя. Прямо сейчас у нас все будет в порядке только с `zero_or_more` и `one_or_more`.

Another exercise might be to find a way around those ownership issues - maybe by
wrapping a parser in an `Rc` to make it clonable?

### A Predicate Combinator

We now have the building blocks we need to parse that whitespace with
`one_or_more`, and to parse the attribute pairs with `zero_or_more`.

Actually, hold on a moment. We don't really want to parse the whitespace *then*
parse the attributes. If you think about it, if there are no attributes, the
whitespace is optional, and we could encounter an immediate `>` or `/>`. But if
there's an attribute, there *must* be whitespace first. Lucky for us, there must
also be whitespace between each attribute, if there are several, so what we're
really looking at here is a sequence of *zero or more* occurrences of *one or
more* whitespace items followed by the attribute.

Сначала нам нужен парсер для одного элемента пробела. Мы можем пойти одним из трех путей.

Во-первых. Мы можем быть глупыми и использовать наш парсер `match_literal` со строкой, содержащей только один пробел. Почему это глупо? Потому что пробел это также разрывы строк, табуляции и целый ряд странных символов Юникода, которые отображаются как пробелы. Нам снова придётся опираться на стандартную библиотеку Rust, и, конечно, у `char` есть метод `is_whitespace` как у `is_alphabetic` и `is_alphanumeric`.

Во-вторых. Мы можем просто написать парсер, который использует любое количество пробельных символов, используя предикат `is_whitespace` же, как мы писали наш `identifier` ранее.

В-третьем. Мы можем быть умными, и нам нравится быть умными. Мы могли бы написать синтаксический анализатор `any_char` который возвращает один `char` до тех пор, пока на входе остается один, и комбинатор `pred` который принимает синтаксический анализатор и функцию предиката, и объединить их так: `pred(any_char, |c| c.is_whitespace())`. Это дает дополнительный бонус, заключающийся в упрощении написания финального парсера, который нам тоже понадобится: строка в кавычках для значений атрибутов.

Парсер `any_char` прост, как синтаксический анализатор, но мы должны помнить, что нужно помнить об этих ошибках UTF-8.

```rust
fn any_char(input: &str) -> ParseResult<char> {
    match input.chars().next() {
        Some(next) => Ok((&input[next.len_utf8()..], next)),
        _ => Err(input),
    }
}
```

And the `pred` combinator also doesn't hold any surprises to our now seasoned
eyes. We invoke the parser, then we call our predicate function on the value if
the parser succeeded, and only if that returns true do we actually return a
success, otherwise we return as much of an error as a failed parse would.

```rust
fn pred<'a, P, A, F>(parser: P, predicate: F) -> impl Parser<'a, A>
where
    P: Parser<'a, A>,
    F: Fn(&A) -> bool,
{
    move |input| {
        if let Ok((next_input, value)) = parser.parse(input) {
            if predicate(&value) {
                return Ok((next_input, value));
            }
        }
        Err(input)
    }
}
```

And a quick test to make sure everything is in order:

```rust
#[test]
fn predicate_combinator() {
    let parser = pred(any_char, |c| *c == 'o');
    assert_eq!(Ok(("mg", 'o')), parser.parse("omg"));
    assert_eq!(Err("lol"), parser.parse("lol"));
}
```

С помощью этих двух, мы можем написать наш `whitespace_char` анализатор с быстрым однострочником:

```rust
fn whitespace_char<'a>() -> impl Parser<'a, char> {
    pred(any_char, |c| c.is_whitespace())
}
```

И теперь, когда у нас есть `whitespace_char`, мы также можем выразить идею, к которой мы идем, *один или несколько пробелов*, и его родственную идею, *ноль или более пробелов*. Давайте побалуем себя краткостью и назовем их `space1` и `space0` соответственно.

```rust
fn space1<'a>() -> impl Parser<'a, Vec<char>> {
    one_or_more(whitespace_char())
}

fn space0<'a>() -> impl Parser<'a, Vec<char>> {
    zero_or_more(whitespace_char())
}
```

### Quoted Strings

Можем ли мы, наконец, разобрать эти атрибуты после всего этого? Да, нам просто нужно убедиться, что у нас есть все отдельные парсеры для компонентов атрибутов. У нас есть `identifier` уже имя атрибута (хотя это заманчиво, чтобы переписать его с помощью `any_char` и `pred` плюс наши `*_or_more` комбинаторы). `=` просто `match_literal("=")`. У нас короткий анализатор строк в кавычках, так что давайте его построим. К счастью, у нас уже есть все комбинаторы, которые нам нужны для этого.

```rust
fn quoted_string<'a>() -> impl Parser<'a, String> {
    map(
        right(
            match_literal("\""),
            left(
                zero_or_more(pred(any_char, |c| *c != '"')),
                match_literal("\""),
            ),
        ),
        |chars| chars.into_iter().collect(),
    )
}
```

Вложенность комбинаторов становится немного раздражающим в этот момент, но мы будем сопротивляться рефакторингу всего, чтобы исправить это только сейчас, и вместо этого сосредоточимся на том, что здесь происходит.

Самым внешним комбинатором является `map` из-за вышеупомянутого назойливого вложения, и это ужасное место, с которого нужно начинать, если мы собираемся понять это, поэтому давайте попробуем найти, где оно действительно начинается: первый символ кавычки. Внутри `map` есть `right`, и первая часть `right` - это то, что мы ищем: `match_literal("\"")`. Это наша начальная двойная ковычка.

Вторая часть этого `right` - остальная часть строки. Это внутри `left`, и мы быстро заметим, что *правый* аргумент этого `left`, тот, который мы игнорируем, является другим `match_literal("\"")` - закрывающей кавычкой. Таким образом, левая часть - наша строка в кавычках.

Мы воспользоваться нашей новой `pred` и `any_char`, чтобы получить парсер, который принимает *ничего кроме другой кавычки* и предположим, что в {code4}zero_or_more{/code4}, так что то, что мы говорим, заключается в следующем:

- одна кавычка
- следуют ноль или более вещей, которые *не* являются другой кавычкой
- сопровождаемый другой кавычкой

And, between the `right` and the `left`, we discard the quotes from the result
value and get our quoted string back.

Но подождите, это не строка. Помните, что возвращается `zero_or_more`? `Vec<A>` для типа возвращаемого внутреннего анализатора `A`. Для `any_char` это `char`. Таким образом, мы имеем не строку, а `Vec<char>`. Вот тут и появляется `map`: мы используем ее, чтобы превратить `Vec<char>` в `String`, используя тот факт, что вы можете создать `String` из `Iterator<Item = char>`, поэтому мы можем просто вызвать `vec_of_chars.into_iter().collect()` и благодаря возможности вывода типов, у нас есть `String`.

Let's just write a quick test to make sure that's all right before we go on,
because if we needed that many words to explain it, it's probably not something
we should leave to our faith in ourselves as programmers.

```rust
#[test]
fn quoted_string_parser() {
    assert_eq!(
        Ok(("", "Hello Joe!".to_string())),
        quoted_string().parse("\"Hello Joe!\"")
    );
}
```

So, now, finally, I swear, let's get those attributes parsed.

### At Last, Parsing Attributes

We can now parse whitespace, identifiers, `=` signs and quoted strings. That,
finally, is all we need for parsing attributes.

Сначала напишем парсер для пары атрибутов. Как вы помните, мы будем хранить их как `Vec<(String, String)>`, поэтому `zero_or_more` что нам нужен анализатор для этого `(String, String)` для {code3}zero_or_more{/code3} нашему надежному комбинатору {code4}zero_or_more{/code4} . Посмотрим, сможем ли мы его построить.

```rust
fn attribute_pair<'a>() -> impl Parser<'a, (String, String)> {
    pair(identifier, right(match_literal("="), quoted_string()))
}
```

Даже не потея! Подводя итог: у нас уже есть удобный комбинатор для разбора кортежа значений, `pair`, поэтому мы используем его с анализатором `identifier`, результатом которого является `String`, а `right` со знаком `=`, значение которого мы не хотим сохранять, и наш свежий `quoted_string` анализатор, который возвращает нам другую `String` .

Now, let's combine that with `zero_or_more` to build that vector - but let's not
forget that whitespace in between them.

```rust
fn attributes<'a>() -> impl Parser<'a, Vec<(String, String)>> {
    zero_or_more(right(space1(), attribute_pair()))
}
```

Ноль или более вхождений: один или несколько пробельных символов,
затем пара атрибутов. Мы используем `right` чтобы отбросить пробел и сохранить пару атрибутов.

Let's test it.

```rust
#[test]
fn attribute_parser() {
    assert_eq!(
        Ok((
            "",
            vec![
                ("one".to_string(), "1".to_string()),
                ("two".to_string(), "2".to_string())
            ]
        )),
        attributes().parse(" one=\"1\" two=\"2\"")
    );
}
```

Tests are green! Ship it!

На самом деле, нет, в этот момент повествования мой rustc жаловался, что мои типы становятся ужасно сложными, и что мне нужно увеличить максимально допустимый размер вложенности типов. Это хороший шанс, что вы получите ту же ошибку на этом этапе, и если вы это делаете, вам нужно знать, как ее устранить. К счастью, в таких ситуациях rustc, как правило, дает хороший совет, поэтому, когда вам нужно добавить `#![type_length_limit = "…some big number…"]` в начало файла, просто сделайте, как говорится. На самом деле, просто сделайте это `#![type_length_limit = "16777216"]` , что позволит нам продвинуться немного дальше в стратосферу сложных типов. Вперед, мы типа космонавты!

### Так теперь завершение

На данный момент, кажется, что они только начинают собираться вместе, что является небольшим облегчением, так как наши типы быстро приближаются к NP-полноте. У нас просто есть две версии тега element: один элемент и родительский элемент с дочерними элементами, но мы уверены, что когда они появятся, разбор дочерних `zero_or_more` будет просто вопросом {code1}zero_or_more{/code1}, верно?

Итак, давайте сначала сделаем один элемент, немного отложив вопрос о детях. Или, что еще лучше, давайте сначала напишем парсер для всего, что у них общего: открывающий символ `<`, имени элемента и атрибутов. Давайте посмотрим, сможем ли мы получить тип результата `(String, Vec<(String, String)>)` из пары комбинаторов.

```rust
fn element_start<'a>() -> impl Parser<'a, (String, Vec<(String, String)>)> {
    right(match_literal("<"), pair(identifier, attributes()))
}
```

With that in place, we can quickly tack the tag closer on it to make a parser
for the single element.

```rust
fn single_element<'a>() -> impl Parser<'a, Element> {
    map(
        left(element_start(), match_literal("/>")),
        |(name, attributes)| Element {
            name,
            attributes,
            children: vec![],
        },
    )
}
```

Ура, у нас такое ощущение, что мы достигли своей цели - сейчас мы действительно строим `Element`!

Let's test this miracle of modern technology.

```rust
#[test]
fn single_element_parser() {
    assert_eq!(
        Ok((
            "",
            Element {
                name: "div".to_string(),
                attributes: vec![("class".to_string(), "float".to_string())],
                children: vec![]
            }
        )),
        single_element().parse("<div class=\"float\"/>")
    );
}
```

…and I think we just ran out of stratosphere.

The return type of `single_element` is so complicated that the compiler will
grind away for a very long time until it runs into the very large type size
limit we gave it earlier, asking for an even larger one. It's clear we can no
longer ignore this problem, as it's a rather trivial parser and a compilation
time of several minutes - maybe even several hours for the finished product -
seems mildly unreasonsable.

Before proceeding, you'd better comment out those two functions and tests while
we fix things…

### To Infinity And Beyond

If you've ever tried writing a recursive type in Rust, you might already know
the solution to our little problem.

A very simple example of a recursive type is a singly linked list. You can
express it, in principle, as an enum like this:

```rust
enum List<A> {
    Cons(A, List<A>),
    Nil,
}
```

На что rustc будет очень разумно возражать, что ваш рекурсивный тип `List<A>` имеет бесконечный размер, потому что внутри каждого `List::<A>::Cons` есть еще один `List<A>`, а это значит, что это `List<A>` вплоть до бесконечности. Что касается rustc, то мы спрашиваем для бесконечного списка, и мы просим, чтобы быть в состоянии *выделить* бесконечный список.

Во многих языках бесконечный список в принципе не является проблемой для системы типов, и на самом деле это не для Rust. Проблема заключается в том, что в Rust, как уже упоминалось, нам нужно иметь возможность *разместить* его в памяти, или, скорее, нам нужно уметь определять *размер* типа заранее, когда мы его конструируем и когда тип бесконечен, что значит, размер тоже должен быть бесконечным.

Решение состоит в том, чтобы использовать немного косвенности. Вместо того, чтобы наш `List::Cons` был элементом `A` и другим *списком* `A`, вместо этого мы делаем его элементом `A` и *указателем* на список `A`. Мы знаем размер указателя, и он одинаков, независимо от того, на что он указывает, и поэтому наш `List::Cons` теперь имеет фиксированный и предсказуемый размер, независимо от размера списка. И способ превратить данные, которыми  вы владеете на стеке в указатель в куче в Rust - это `Box`.

```rust
enum List<A> {
    Cons(A, Box<List<A>>),
    Nil,
}
```

Еще одна интересная особенность `Box` заключается в том, что тип внутри него может быть абстрактным. Это означает, что вместо наших теперь невероятно сложных типов функций синтаксического анализатора мы можем позволить средству проверки типов иметь дело с очень лаконичным `Box<dyn Parser<'a, A>>`.

Это звучит великолепно. Какой минус? Что ж, мы можем потерять цикл или два из-за необходимости следовать этому указателю, и может случиться так, что компилятор потеряет некоторые возможности для оптимизации нашего синтаксического анализатора. Но вспомните предостережение Кнута о преждевременной оптимизации: все будет хорошо. Вы можете позволить себе эти циклы. Вы здесь, чтобы узнать о комбинатрных синтаксических анализаторах, а не узнать о рукописных гиперспециализированных [синтаксических анализаторах SIMD](https://github.com/lemire/simdjson) (хотя они интересны сами по себе).

Итак, давайте приступим к реализации `Parser` для функции парсера в *Box* в дополнение к голым функциям, которые мы использовали до сих пор.

```rust
struct BoxedParser<'a, Output> {
    parser: Box<dyn Parser<'a, Output> + 'a>,
}

impl<'a, Output> BoxedParser<'a, Output> {
    fn new<P>(parser: P) -> Self
    where
        P: Parser<'a, Output> + 'a,
    {
        BoxedParser {
            parser: Box::new(parser),
        }
    }
}

impl<'a, Output> Parser<'a, Output> for BoxedParser<'a, Output> {
    fn parse(&self, input: &'a str) -> ParseResult<'a, Output> {
        self.parser.parse(input)
    }
}
```

Мы создаем новый тип `BoxedParser` для хранения нашего `box`, ради приличия. Чтобы создать новый `BoxedParser` из любого другого вида синтаксического анализатора (включая другой `BoxedParser`, даже если это было бы бессмысленно), мы предоставляем функцию `BoxedParser::new(parser)`, которая делает только то, что помещает этот анализатор в `Box` внутри нашего нового типа. Наконец, мы реализуем `Parser` для него, так что он может быть использован взаимозаменяемо как анализатор.

Это оставляет нам возможность поместить функцию синтаксического анализа в `Box`, и `BoxedParser` будет работать как `Parser` так же, как и функция. Теперь, как упоминалось ранее, это означает, что нужно переместить упакованный синтаксический анализатор в кучу и разыменовывать указатель, чтобы добраться до него, что может стоить нам *нескольких драгоценных наносекунд*, поэтому мы, возможно, захотим отложить *все в* сторону. Достаточно просто собрать некоторые из наиболее популярных комбинаторов.

### An Opportunity Presents Itself

But, just a moment, this presents us with an opportunity to fix another thing
that's starting to become a bit of a bother.

Помните пару последних парсеров, которые мы написали? Поскольку наши комбинаторы являются автономными функциями, когда мы вкладываем их друг в друга, наш код начинает становиться немного не читаемым. Напомним наш парсер `quoted_string`:

```rust
fn quoted_string<'a>() -> impl Parser<'a, String> {
    map(
        right(
            match_literal("\""),
            left(
                zero_or_more(pred(any_char, |c| *c != '"')),
                match_literal("\""),
            ),
        ),
        |chars| chars.into_iter().collect(),
    )
}
```

Было бы намного лучше читать, если бы мы могли создавать эти методы комбинаторов в парсере вместо отдельных функций. Что если бы мы могли объявить наши комбинаторы как методы у типажа `Parser`?

Проблема в том, что если мы это сделаем, мы потеряем способность сделать `impl Trait` для наших типов возвращаемых данных, потому что `impl Trait` не допускается в объявлениях типажа.

… Но теперь у нас есть `BoxedParser`. Мы не можем объявить метод у типажа, который возвращает `impl Parser<'a, A>`, но мы наверняка *можем* объявить тот, который возвращает `BoxedParser<'a, A>`.

Самое приятное то, что мы можем даже объявить их с реализациями по умолчанию, так что нам не нужно переопределять каждый комбинатор для каждого типа, который реализует `Parser`.

Давайте попробуем это с `map`, расширив наш типаж `Parser` следующим образом:

```rust
trait Parser<'a, Output> {
    fn parse(&self, input: &'a str) -> ParseResult<'a, Output>;

    fn map<F, NewOutput>(self, map_fn: F) -> BoxedParser<'a, NewOutput>
    where
        Self: Sized + 'a,
        Output: 'a,
        NewOutput: 'a,
        F: Fn(Output) -> NewOutput + 'a,
    {
        BoxedParser::new(map(self, map_fn))
    }
}
```

Это очень много `'a`, но, увы, все они необходимы. К счастью, мы все еще можем повторно использовать наши старые функции комбинатора без изменений - и, в качестве дополнительного бонуса, мы не только получаем более приятный синтаксис для их применения, мы также избавляемся от взрывоопасных типов `impl Trait`.

Теперь мы можем немного улучшить наш `quoted_string` анализатор:

```rust
fn quoted_string<'a>() -> impl Parser<'a, String> {
    right(
        match_literal("\""),
        left(
            zero_or_more(pred(any_char, |c| *c != '"')),
            match_literal("\""),
        ),
    )
    .map(|chars| chars.into_iter().collect())
}
```

На первый взгляд, теперь более очевидно, что `.map()` вызывается по результату `right()`.

Мы могли бы также сделать одинаковую обработку для `pair`, `left` и `right`, но в случае этих трех я думаю, что они читаются легче, когда они являются функциями, потому что они отражают структуру выходного типа `pair`. Если вы не согласны, то вполне возможно добавить их в типаж, как мы сделали с `map`, и вы всегда можете попробовать это в качестве упражнения.

Другой главный кандидат `pred`. Давайте добавим определение для него в типаж `Parser`:

```rust
fn pred<F>(self, pred_fn: F) -> BoxedParser<'a, Output>
where
    Self: Sized + 'a,
    Output: 'a,
    F: Fn(&Output) -> bool + 'a,
{
    BoxedParser::new(pred(self, pred_fn))
}
```

Это позволяет нам переписать строку в `quoted_string` с `pred` вызова, как то так:

```rust
zero_or_more(any_char.pred(|c| *c != '"')),
```

Я думаю, что это выглядит немного лучше, и я думаю, что мы оставим значение `zero_or_more` как есть - оно читается как «ноль или более из `any_char` с применением следующего предиката», и это звучит хорошо. Еще раз, вы также можете пойти дальше и переместить `zero_or_more` и `one_or_more` в типаж.

В дополнение к переписыванию `quoted_string`, давайте также исправим `map` в `single_element`:

```rust
fn single_element<'a>() -> impl Parser<'a, Element> {
    left(element_start(), match_literal("/>")).map(|(name, attributes)| Element {
        name,
        attributes,
        children: vec![],
    })
}
```

Let's try and uncomment back `element_start` and the tests we commented out
earlier and see if things got better. Get that code back in the game and try
running the tests…

... и да, время компиляции вернулось к нормальному состоянию сейчас. Вы можете даже удалить эту настройку размера типа в верхней части вашего файла, она вам больше не понадобится.

И это стоило поместить  `map` и `pred` в `Box` - *и* мы получили более приятный синтаксис!

### Having Children

Теперь давайте напишем парсер для открывающего тега для родительского элемента. Он почти идентичен `single_element`, за исключением того, что он заканчивается на `>`, а не на `/>`. За ним также следует ноль или более дочерних элементов и закрывающий тег, но сначала нам нужно проанализировать фактический открывающий тег, так что давайте сделаем это.

```rust
fn open_element<'a>() -> impl Parser<'a, Element> {
    left(element_start(), match_literal(">")).map(|(name, attributes)| Element {
        name,
        attributes,
        children: vec![],
    })
}
```

Теперь, как мы получаем этих детей? Они будут либо отдельными элементами, либо самими родительскими элементами, и их будет ноль или более, так что у нас есть наш надежный комбинатор `zero_or_more`, но что нам его кормить? Одна вещь, с которой мы еще не сталкивались, - это синтаксический анализатор с множественным выбором: то, что анализирует *либо* один элемент, *либо* родительский элемент.

Чтобы попасть туда, нам нужен комбинатор, который пробует два парсера по порядку: если первый парсер завершается успешно, мы закончили, мы возвращаем его результат и все. Если это не удается, вместо того, чтобы вернуть ошибку, мы пробуем второй парсер *на том же входе*. Если это удается, отлично, а если нет, мы также возвращаем ошибку, поскольку это означает, что оба наших синтаксических анализатора вышли из строя, и это общая ошибка.

```rust
fn either<'a, P1, P2, A>(parser1: P1, parser2: P2) -> impl Parser<'a, A>
where
    P1: Parser<'a, A>,
    P2: Parser<'a, A>,
{
    move |input| match parser1.parse(input) {
        ok @ Ok(_) => ok,
        Err(_) => parser2.parse(input),
    }
}
```

Это позволяет нам объявлять `element` синтаксического анализатора, который соответствует либо одному элементу, либо родительскому элементу (и пока давайте просто используем `open_element` для его представления, и мы будем иметь дело с `element`).

```rust
fn element<'a>() -> impl Parser<'a, Element> {
    either(single_element(), open_element())
}
```

Now let's add a parser for the closing tag. It has the interesting property of
having to match the opening tag, which means the parser has to know what the
name of the opening tag is. But that's what function arguments are for, yes?

```rust
fn close_element<'a>(expected_name: String) -> impl Parser<'a, String> {
    right(match_literal("</"), left(identifier, match_literal(">")))
        .pred(move |name| name == &expected_name)
}
```

That `pred` combinator is proving really useful, isn't it?

And now, let's put it all together for the full parent element parser, children
and all:

```rust
fn parent_element<'a>() -> impl Parser<'a, Element> {
    pair(
        open_element(),
        left(zero_or_more(element()), close_element(…oops)),
    )
}
```

Oops. How do we pass that argument to `close_element` now? I think we're short
one final combinator.

Мы так близко сейчас. Как только мы решили эту последнюю проблему, чтобы заставить `parent_element` работать, мы сможем заменить заполнитель `open_element` в парсере `element` нашим новым `parent_element`, и все, у нас есть полностью работающий парсер XML.

Помните, я сказал, что мы вернемся к `and_then` потом? Ну, позже здесь. Фактически нам нужен `and_then`: нам нужно что-то, что принимает парсер, и функцию, которая берет результат парсера и возвращает *новый* парсер, который мы затем запустим. Это немного похоже на `pair`, за исключением того, что вместо того, чтобы просто собрать оба результата в кортеж, мы пропускаем их через функцию. Кроме того, `and_then` работает с `Result` и `Option`, за исключением того, что за ним немного проще следовать, потому что `Result` и `Option` на самом деле ничего не *делают*, это просто вещи, которые содержат некоторые данные (или нет, в зависимости от обстоятельств)).

So let's try writing an implementation for it.

```rust
fn and_then<'a, P, F, A, B, NextP>(parser: P, f: F) -> impl Parser<'a, B>
where
    P: Parser<'a, A>,
    NextP: Parser<'a, B>,
    F: Fn(A) -> NextP,
{
    move |input| match parser.parse(input) {
        Ok((next_input, result)) => f(result).parse(next_input),
        Err(err) => Err(err),
    }
}
```

У рассматриваемого типа, существует много переменных типа, но мы знаем `P`, наш анализатор ввода, который имеет тип результата `A`. Наша функция `F`, однако, где `map` имеет функцию от `A` до `B`, принципиальное отличие состоит в том, что `and_then` переводит функцию из `A` в *новый синтаксический анализатор* `NextP`, который имеет тип результата `B`. Тип конечного результата - `B`, поэтому мы можем предположить, что все, что выйдет из нашего `NextP` будет конечным результатом.

Код немного менее сложен: мы начинаем с запуска нашего анализатора ввода, и если он терпит неудачу, мы закончили, но если успешно, теперь мы вызываем нашу функцию `f` для результата (типа `A`), и то, что получается из `f(result)`, это новый парсер с типом результата `B`. Мы запускаем *этот* синтаксический анализатор для следующего фрагмента ввода и возвращаем результат напрямую. Если это терпит неудачу, это терпит неудачу там, и если это успешно, у нас есть наше значение типа `B`

Еще раз: сначала мы запускаем наш синтаксический анализатор типа `P`, и если это удается, мы вызываем функцию `f` с результатом синтаксического анализатора `P`, чтобы получить наш следующий синтаксический анализатор типа `NextP`, который мы затем запускаем, и это конечный результат.

Давайте также сразу добавим его в типаж `Parser`, потому что это `map`, определенно будет читаться лучше.

```rust
fn and_then<F, NextParser, NewOutput>(self, f: F) -> BoxedParser<'a, NewOutput>
where
    Self: Sized + 'a,
    Output: 'a,
    NewOutput: 'a,
    NextParser: Parser<'a, NewOutput> + 'a,
    F: Fn(Output) -> NextParser + 'a,
{
    BoxedParser::new(and_then(self, f))
}
```

OK, now, what's it good for?

First of all, we can *almost* implement `pair` using it:

```rust
fn pair<'a, P1, P2, R1, R2>(parser1: P1, parser2: P2) -> impl Parser<'a, (R1, R2)>
where
    P1: Parser<'a, R1> + 'a,
    P2: Parser<'a, R2> + 'a,
    R1: 'a + Clone,
    R2: 'a,
{
    parser1.and_then(move |result1| parser2.map(move |result2| (result1.clone(), result2)))
}
```

Это выглядит очень аккуратно, но есть проблема: `parser2.map()` потребляет `parser2` для создания упакованного синтаксического анализатора, и функция является `Fn`, а не `FnOnce`, поэтому не разрешено использовать `parser2`, просто возьмите ссылку на него. Другими словами, проблемы Rust. На языке более высокого уровня, где эти вещи не являются проблемой, это был бы действительно аккуратный способ определения `pair`.

Однако, что мы можем сделать с этим даже в Rust, это использовать эту функцию для ленивой генерации правильной версии нашего анализатора `close_element`, или другими словами, мы можем заставить его передать этот аргумент в него.

Recalling our failed attempt:

```rust
fn parent_element<'a>() -> impl Parser<'a, Element> {
    pair(
        open_element(),
        left(zero_or_more(element()), close_element(…oops)),
    )
}
```

Используя `and_then`, теперь мы можем получить это право, используя эту функцию для создания правильной версии `close_element` на месте.

```rust
fn parent_element<'a>() -> impl Parser<'a, Element> {
    open_element().and_then(|el| {
        left(zero_or_more(element()), close_element(el.name.clone())).map(move |children| {
            let mut el = el.clone();
            el.children = children;
            el
        })
    })
}
```

Теперь это выглядит немного сложнее, потому что `and_then` должен идти в `open_element()`, где мы узнаем имя, которое входит в `close_element`. Это означает, что остальная часть синтаксического анализатора после `open_element` должна быть построена внутри замыкания `and_then`. Более того, поскольку это замыкание теперь является единственным получателем результата `Element` от `open_element`, анализатор, который мы возвращаем, также должен передавать эту информацию дальше.

Внутреннее замыкание, которое `map` поверх сгенерированного синтаксического анализатора, имеет ссылку на `Element` ( `el`) из внешнего замыкания. Мы должны его `clone()`, потому что мы находимся в `Fn` и поэтому имеем только ссылку на него. Мы берем результат внутреннего синтаксического анализатора (наш `Vec<Element>` детей) и добавляем его к нашему клонированному `Element`, и мы возвращаем его как наш конечный результат.

Все, что нам нужно сделать сейчас, это вернуться к нашему анализатору `element` и убедиться, что мы изменили `open_element` на `parent_element`, чтобы он анализировал всю структуру элемента, а не только его начало, и я считаю, что мы закончили!

### Are You Going To Say The M Word Or Do I Have To?

Помните, мы говорили о том, как шаблон `map` называется «функтором» на планете Haskell?

`and_then` - это еще одна вещь, которую вы `and_then` видите в Rust, в основном в тех же местах, что и {code2}map{/code2}. Он называется {a3}`flat_map`{/a3} в {code5}Iterator{/code5}, но это тот же шаблон, что и остальные.

Причудливое слово для этого - "монада". Если у вас есть вещь `Thing<A>`, и у вас есть функция `and_then` в которую вы можете передать функцию из `A` в `Thing<B>`, так что теперь у вас есть новая `Thing<B>`, это монада,

Функция может быть вызвана мгновенно, например, когда у вас есть `Option<A>`, мы уже знаем, является ли это `Some(A)` или `None`, поэтому мы применяем функцию напрямую, если это `Some(A)`, давая нам `Some(B)`

Это также можно назвать ленивым. Например, если у вас есть `Future<A>` которая все еще ожидает разрешения, вместо `and_then` немедленно вызывающего функцию для создания `Future<B>`, вместо этого он создает новую `Future<B>` которая содержит обе `Future<A>` и функцию, которая затем ожидает завершения `Future<A>`. Когда это происходит вызывается функция с результатом `Future<A>`, и Боб - твой дядя <sup><a href="#footnote1">1</a></sup>, ты возвращаешь свое `Future<B>`. Другими словами, в случае `Future` вы можете думать о функции, которую вы передаете `and_then` как о *функции обратного вызова*, потому что она {code13}and_then{/code13} с результатом исходной `Future`, когда она завершается. Это также немного более интересным , чем это, потому что она возвращает {em14}новую{/em14} {code15}Future{/code15}, которое может или не может уже решено, так что путь к {em16}цепочке{/em16} `Future`s вместе.

As with functors, though, Rust's type system isn't currently capable of
expressing monads, so let's only note that this pattern is called a monad, and
that, rather disappointingly, it's got nothing at all to do with burritos,
contrary to what they say on the internets, and move on.

### Whitespace, Redux

Just one last thing.

Теперь у нас должен быть парсер, способный анализировать некоторые XML, но он не очень ценит пробелы. Произвольный пробел должен быть разрешен между тегами, чтобы мы могли свободно вставлять разрывы строк и тому подобное между нашими тегами (и в принципе, пробел должен быть разрешен между идентификаторами и литералами, как `< div / >`, но давайте пропустим это).

We should be able to put together a quick combinator for that with no effort at
this point.

```rust
fn whitespace_wrap<'a, P, A>(parser: P) -> impl Parser<'a, A>
where
    P: Parser<'a, A>,
{
    right(space0(), left(parser, space0()))
}
```

Если мы обернем `element` в это, он будет игнорировать все начальные и конечные пробелы вокруг `element`, что означает, что мы можем использовать столько разрывов строк и столько отступов, сколько нам нужно.

```rust
fn element<'a>() -> impl Parser<'a, Element> {
    whitespace_wrap(either(single_element(), parent_element()))
}
```

### We're Finally There!

I think we did it! Let's write an integration test to celebrate.

```rust
#[test]
fn xml_parser() {
    let doc = r#"
        <top label="Top">
            <semi-bottom label="Bottom"/>
            <middle>
                <bottom label="Another bottom"/>
            </middle>
        </top>"#;
    let parsed_doc = Element {
        name: "top".to_string(),
        attributes: vec![("label".to_string(), "Top".to_string())],
        children: vec![
            Element {
                name: "semi-bottom".to_string(),
                attributes: vec![("label".to_string(), "Bottom".to_string())],
                children: vec![],
            },
            Element {
                name: "middle".to_string(),
                attributes: vec![],
                children: vec![Element {
                    name: "bottom".to_string(),
                    attributes: vec![("label".to_string(), "Another bottom".to_string())],
                    children: vec![],
                }],
            },
        ],
    };
    assert_eq!(Ok(("", parsed_doc)), element().parse(doc));
}
```

И тот, который терпит неудачу из-за несоответствующего закрывающего тега, просто чтобы убедиться, что мы правильно поняли этот фрагмент:

```rust
#[test]
fn mismatched_closing_tag() {
    let doc = r#"
        <top>
            <bottom/>
        </middle>"#;
    assert_eq!(Err("</middle>"), element().parse(doc));
}
```

Хорошей новостью является то, что он возвращает несоответствующий закрывающий тег как ошибку. Плохая новость заключается в том, что в действительности это не *говорит* о том, что проблема заключается в несовпадающем закрывающем теге, именно *там* , {em2}где{/em2} ошибка. Это лучше, чем ничего, но, честно говоря, по мере появления сообщений об ошибках это все еще ужасно. Но превращение этого в вещь, которая на самом деле дает хорошие ошибки, является темой для другой, и, по крайней мере, такой же длинной статьи.

Давайте сосредоточимся на хороших новостях: мы написали парсер с нуля, используя комбинаторные парсероы! Мы знаем, что синтаксический анализатор образует и функтор, и монаду, так что теперь вы можете впечатлять людей на вечеринках своими устрашающими знаниями теории категорий <sup><a href="#footnote2">2</a></sup> .

Самое главное, теперь мы знаем, как комбинаторные синтаксические анализаторы работают с нуля. Никто не может остановить нас сейчас!

### Victory Puppies

<p class="gif"><img src="../../../originals/articles/parser-combinators/many-puppies.gif"></p>

### Further Resources

Прежде всего, я виновен в том, что объяснил вам монады в строго Rust выражениях, и я знаю, что Фил Уодлер был бы очень расстроен из-за меня, если бы я не указал вам на [его основную статью, в](https://homepages.inf.ed.ac.uk/wadler/papers/marktoberdorf/baastad.pdf) которой более подробно рассказывается о них. - включая его отношение к комбинаторам синтаксического анализа.

Идеи в этой статье очень похожи на идеи, библиотеку комбинаторов [`pom`](https://crates.io/crates/pom), и, если вы захотите работать с комбинаторными парсерами в одном стиле, я очень рекомендую это.

The state of the art in Rust parser combinators is still
[`nom`](https://crates.io/crates/nom), to the extent that the aforementioned
`pom` is clearly derivatively named (and there's no higher praise than that),
but it takes a very different approach from what we've built here today.

Другой популярной библиотекой комбинаторных синтаксических анализаторов для Rust является [`combine`](https://crates.io/crates/combine), которое также стоит посмотреть.

Основной библиотекой комбинаторного синтаксического анализа для Haskell является [Parsec](http://hackage.haskell.org/package/parsec) .

Наконец, я обязан своим первым знакомством с комбинаторными парсерами книге Грэма Хаттона «[*Программирование на Haskell*](http://www.cs.nott.ac.uk/%7Epszgmh/pih.html)», которая очень полезна для чтения и имеет положительный побочный эффект, так как обучает вас языку Haskell.

## Licence

This work is copyright Bodil Stokke and is licensed under the Creative Commons
Attribution-NonCommercial-ShareAlike 4.0 International Licence. To view a copy
of this licence, visit http://creativecommons.org/licenses/by-nc-sa/4.0/.

## Footnotes

<a name="footnote1">1</a>: He isn't really your uncle.

<a name="footnote2">2</a>: Please don't be that person at parties.
