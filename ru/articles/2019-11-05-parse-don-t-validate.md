Еще в декабре мне попалась одна совершенно замечательная статья на английском, посвящённая использованию системы типов языка для более широкого класса задач, для повышения надежности приложений и простоты рефакторинга. К сожалению, в тот момент я был слишком занят написанием статей по ФП, которые крайне важно было написать, пока свежи воспоминания. Но теперь, когда с этой задачей я справился, наконец дошли руки перевести эту замечательную заметку. Оригинальный язык примеров - Хаскель, но я решил переписать их на раст, для более широкого охвата аудитории. Однако, язык тут совершенно неважен, советы этой статьи я применяю в ежедневной разработке на вполне себе "приземлённых" C# и TypeScript, так что если вы просто стараетесь писать надёжный и поддерживаемый код, то, вне зависимости от языка, статья вам будет в тему.

Благодарю за вычитку и помощь в переводе @Hirrolot и @funkill 

![](https://habrastorage.org/webt/a2/2z/ev/a22zeviaxt_a0pyngplxgaqaiz0.jpeg)
*Здесь должна быть КДВП, но я не нашел ничего подходящего, поэтому вот картинка, которую гугл считает релевантной заголовку*

<cut />

На протяжении достаточно длительного периода времени я изо всех сил старалась найти точный, простой способ объяснить, что такое "Type-Driven Design" (*здесь и далее - TypeDD, прим. пер.*). Довольно часто меня спрашивали "Как ты додумалась до такого решения?", но удовлетворительного ответа у меня не было. Я точно знаю, что оне пришло ко мне как видение: у меня был итеративный процесс проектирования, не обязывающий сию минуту из ничего получить "правильную" архитектуру, но до сих пор мне не особо удавалось истолковать этот процесс другим.

Но месяц назад случился поворотный момент: [я рассуждала в Твиттере](https://twitter.com/lexi_lambda/status/1182242561655746560) о различии парсинга JSON в статически и динамически типизированных языках, и, наконец, я нашла то, что искала. Теперь у меня есть единственный, цепкий слоган, являющийся квинтэссенцией того, чем для меня является TypeDD, и, что ещё лучше, состоящий всего из трёх слов:

**Парсите, а не валидируйте.**

# Суть TypeDD

Ладно, признаю: если вы ещё не в курсе про TypeDD, мой хитроумный слоган, скорее всего, ничего вам не скажет. К счастью, именно для этого и написана вся остальная статья. Я собираюсь объяснить что я имею в виду во всех подробностях, но сначала давайте немного пофантазируем.

## Ландшафт возможностей

Одна из самых замечательных вещей в статической типизации заключается в том, что она позволяет, а иногда и с лёгкостью, ответить на вопросы вроде "А возможно ли вообще написать такую функцию?". Для контрастного примера, давайте посмотрим на такую сигнатуру:

```rust
enum Void {}

fn foo(x: i32) -> Void
```

Возможно ли реализовать данную функцию? Очевидно, ответ на этот вопрос - *нет*, т.к. `Void` - это тип с нулём возможных значений, поэтому невозможно написать *ни одной* функции, возвращающей значение типа `Void`. Этот пример довольно скучный, но вопрос становится куда интереснее, если мы посмотрим на более реалистичную ситуацию:

```rust
fn head<T>(xs: Vec<T>) -> T
```

Эта функция возвращает первый элемент списка. Возможно ли её реализовать? Да, сперва это не выглядит чем-то сложным, но если мы попробуем её выразить, то компилятор будет недоволен:

```rust
fn head<T>(xs: Vec<T>) -> T {
    match xs.as_slice() {
        [x, ..] => *x
    }
}
```

```
error[E0004]: non-exhaustive patterns: `&[]` not covered
 --> src/lib.rs:2:11
  |
2 |     match xs.as_slice() {
  |           ^^^^^^^^^^^^^ pattern `&[]` not covered
  |
```

Компилятор нам заботливо сообщает, что данная функция *частичная*, что означает, что она не определена для всех возможных значений параметров. В частности, она не определена для случая, когда параметром является `[]`, пустой список. И это совершенно разумно, т.к. невозможно вернуть первый элемент списка, если список пустой, ведь в нём нет ни одного элемента, чтобы его можно было вернуть! Таким образом, как ни удивительно, мы узнали, что эту функцию тоже невозможно реализовать  (*не прибегая к хакам вроде паник и эксепшнов, прим. пер.*).

## Делаем частичные функции тотальными

Для людей с опытом динамически-типизированных языков, такой вывод может показаться довольно неожиданным. У нас есть список, и мы имеем полное право хотеть получить из него первый элемент. И конечно же, операция "получить первый элемент списка" не является невозможной в статически-типизированных языках, она просто требует небольшого количества церемоний. Существует два способа исправить функцию `head`, и мы начнём с того, который попроще.

### Управление ожиданиями

Как мы уже убедились, функция `head` является частичной, т.к. невозможно вернуть элемент из пустого списка: мы делаем обещание, которое невозможно соблюсти. К счастью, существует простое решение этой дилеммы: ослабить наше обещание. Исходя из того, что мы не можем гарантировать, что в списке будет элемент, подлежащий возврату, нам следует немного повлиять на ожидания вызывающего кода: мы сделаем всё возможное, чтобы вернуть элемент, в то же время оставляя за собой право ничего не вернуть. В Rust мы выражаем эту возможность при помощи типа `Option`:

```rust
fn head<T>(xs: Vec<T>) -> Option<T>
```

Это даёт достаточно свободы, чтобы реализовать функцию `head` — позволяя вернуть `None`, когда мы понимаем, что не можем вернуть никакого значения типа `T`:

```rust
fn head<T>(xs: Vec<T>) -> Option<T> {
    match xs.as_slice() {
        [x, ..] => Some(*x),
        [] => None,
    }
}
```

Проблема решена, так? Ну, на текущий момент - да... Однако это решение имеет неявную цену.

Возвращение `Option`, несомненно, удобно, когда мы *реализуем*  `head`. Однако, значительно менее удобно становится её использовать. Так как `head` всегда может вернуть `None`, на весь вызывающий код накладывается бремя проверок этого варианта, и иногда такой перевод стрелок очень напрягает. Чтобы понять почему, давайте посмотрим на такой код:

```rust
fn get_configuration_directories() -> Result<Vec<String>, &'static str> {
    let config_dirs_string = std::env::var("CONFIG_DIRS").map_err(|_| "cannot read env")?;
    let list: Vec<_> = config_dirs_string.split(',').map(|x| x.to_string()).collect();
    if list.is_empty() {
        return Err("CONFIG_DIRS cannot be empty");
    }
    Ok(list)
}


fn main() -> Result<(), &'static str> {
    let config_dirs = get_configuration_directories()?;
    match head(config_dirs) {
        Some(cacheDir) => initialize_cache(cacheDir),
        None => panic!("should never happen; already checked config_dirs is non-empty")
    }
    Ok(())
}
```

Когда функция `get_configuration_directories` получает список путей из окружения, она сразу проверяет что список непустой. Однако, когда мы используем `head` в `main`, чтобы получить первый элемент списка, результат типа `Option<&str>` требует от нас проверки случая `None`, который, как мы знаем, никогда не произойдёт! Это чрезвычайно плохо по многим причинам:

1. Во-первых это просто утомительно. Мы уже знаем, что список непустой, почему мы должны замусоривать код дополнительными ненужными проверками?

2. Во-вторых, оно имеет потенциальные последствия с точки зрения производительности. И хотя стоимость лишней проверки совсем незначительна в данном конкретном случае, легко можно придумать более сложный сценарий, при котором ненужные проверки будут накладываться друг на друга, как например в случае если они производятся в маленьком цикле с большим количеством итераций.

3. Наконец, и что хуже всего, в этом коде затаился баг. Что, если `get_configuration_directories` изменят так, что она перестанет проверять список на пустоту, и не важно, случайно или специально? Программист может и не помнить, что нужно обновить `main`, и внезапно "невозможная" ошибка станет не просто возможной, а очень даже вероятной.

Необходимость этой лишней проверки по сути принуждает нас оставить дыру в нашей системе типов. Если бы мы могли статически *доказать* что случай `None` невозможен, то описанное изменение `get_configuration_directories` перестало бы проходить проверку и вызвало ошибку компиляции.
Однако, в том виде как оно сейчас написано, чтобы найти этот баг, мы должны писать тесты или проводить ручную инспекцию кода.

### Платим вперёд

Понятно, что наша модифицированная версия функции `head` работает не совсем так, как нам хотелось бы. Мы хотели бы, чтобы она каким-то образом была умнее: если мы уже проверили, что список не пуст,  `head` должен безусловно вернуть первый элемент, не принуждая нас обрабатывать случай, про который мы точно знаем, что он невозможен. Как бы нам это сделать?

Давайте посмотрим на изначальную (частичную) сигнатуру `head` ещё раз.

```rust
fn head<T>(xs: Vec<T>) -> T
```

В предыдущем разделе мы превратили частичную сигнатуру в тотальную, ослабив требования к возвращаемому типу. Но раз нам это не подходит, у нас остаётся лишь одна вещь, которую мы можем поменять: тип аргумента (в нашем случае - `Vec<T>`). Вместо ослабления возвращаемого типа, мы можем *усилить* тип аргумента, устранив саму возможность вызова `head` на пустом списке.

Чтобы это сделать, нам нужен тип, представляющий непустые списки.
К счастью, такой тип `NonEmptyVec` несложно написать. У него будет следующее определение:

```rust
struct NonEmptyVec<T>(T, Vec<T>);
```

Заметьте, что `NonEmptyVec` - всего лишь пара из значения типа `T` и обычного (возможно, пустого) `Vec<T>`. Такая структура удобно моделирует непустой список путём сохранения первого элемента отдельно от хвоста, ведь даже если второй компонент `Vec<T>` представляет собой `[]`, то первый компонент всегда должен присутствовать. Благодаря этому, реализация функции `head` становится совершенно тривиальной:

```rust
fn head<T>(xs: NonEmptyVec<T>) -> T {
    xs.0
}
```

Компилятор, в отличие от предыдущей попытки, принимает такое определение без единого возражения, ведь оно *тотальное*, не частичное. Мы можем обновить нашу программу чтобы использовать новую реализацию:

```rust
fn get_configuration_directories() -> Result<NonEmptyVec<String>, &'static str> {
    let config_dirs_string = std::env::var("CONFIG_DIRS").map_err(|_| "cannot read env")?;
    let list: Vec<_> = config_dirs_string.split(',').map(|x| x.to_string()).collect();
    match non_empty(list) {
        Some(x) => Ok(x),
        None => Err("CONFIG_DIRS cannot be empty")
    }
}

fn main() -> Result<(), &'static str> {
    let config_dirs = get_configuration_directories()?;
    initialize_cache(head(config_dirs));
    Ok(())
}
```

Заметьте, что лишняя проверка в  `main` полностью исчезла! Вместо этого, мы производим проверку ровно один раз, в функции `get_configuration_directories`. Она конструирует `NonEmptyVec` из `Vec` с помощью функции  `non_empty`, имеющую следующую сигнатуру:

```rust
fn non_empty<T>(list: Vec<T>) -> Option<NonEmptyVec<T>>
```

Обратите внимание, что `Option` никуда не делся, однако в этот раз мы обрабатываем случай `None` в самом начале программы: в том месте где мы и раньше делали валидацию. Когда проверка пройдена, мы получаем значение типа  `NonEmptyVec`, которое сохраняет (на уровне типов!) тот факт, что список на самом деле не пуст. Другими словами, значение типа `NonEmptyVec` эквивалентно значению типа `Vec<T>` плюс *доказательству*, что список не пуст.

Усилив тип аргумента функции `head` вместо того чтобы ослабить тип результата, мы полностью избавились от всех проблем из предыдущего раздела:

- В коде отсутствуют лишние проверки, поэтому нет и никакого оверхеда по производительности.

- Более того, если функция `get_configuration_directories` перестанет проверять список на пустоту, то её результирующий тип тоже поменяется. Следовательно, функция `main` не сможет тайпчекнуться, сообщая о проблеме до того как мы вообще запустили программу!

Более того, мы можем тривиально восстановить старое поведение функции  `head` с помощью новой версии, композируя её с `non_empty`:

```rust
fn old_head<T>(xs: Vec<T>) -> Option<T> {
    non_empty(xs).map(head)
}
```

Обратите внимание, что обратное *неверно*: не существует способа получить новую версию функции `head` из старой. Таким образом, второй подход превосходит первый по всем параметрам.

## Сила парсинга

Но какое это всё имеет отношение к заголовку статьи? В конце концов, мы просто изучили два разных способа проверить список на пустоту — и, на первый взгляд, тут нет никакого парсинга. Эта интерпретация тоже верна, однако я предлагаю посмотреть на это с другой стороны: с моей точки зрения, вся разница между валидацией и парсингом полностью состоит в том, как сохраняется информация об этом процессе. Давайте сравним две такие функции:

```rust
fn validate_non_empty<T>(xs: Vec<T>) -> Result<(), UserError> {
    if !xs.is_empty() {
        Ok(())
    } else {
        Err(UserError::new("list cannot be empty"))
    }
}

fn parse_non_empty<T>(mut xs: Vec<T>) -> Result<NonEmptyVec<T>, UserError> {
    if !xs.is_empty() {
        let head = xs.remove(0);
        Ok(NonEmptyVec(head, xs))
    } else {
        Err(UserError::new("list cannot be empty"))
    }
}
```

Эти две функции практически идентичны: они проверяют переданный список на пустоту, и если он пустой, то они возвращают сообщение об ошибке. Вся разница заключается в возвращаемом значении: `validate_non_empty`всегда возвращает `()`, тип, который не содержит никакой информации, а `parse_non_empty` возвращает `NonEmptyVec<T>`, уточнение входного типа которое сохраняет полученное знание в системе типов. Обе функции проверяют одно и то же, но `parse_non_empty` даёт вызывающему коду доступ к полученной информации, а `validate_non_empty` просто выкидывает её.

Эти две функции элегантно иллюстрируют два различных взгляда на роль системы типов: `validate_non_empty` просто подчиняется тайпчекеру, но только `parse_non_empty` полностью использует те преимущества, которые он даёт. Если вы видите, почему функция `parse_non_empty` предпочтительнее, то вы должны уже понимать, что означает мантра "парсите, а не валидируйте". Однако, возможно вы скептически относитесь к имени `parse_non_empty`. Действительно ли она что-то *парсит*, или она просто валидирует вход и возвращает результат? И, хотя точное определение того, что означает парсинг или валидация, является предметом для обсуждения, я считаю что `parse_non_empty` это полноценный парсер, пусть и очень простой.

Подумайте: что такое парсер? В действительности, парсер это всего лишь функция, которая принимает менее структурированный вход, и производит более структурированный выход. По самой своей сути, парсер это частичная функция - некоторые значения домена не соответствуют ни одному допустимому значению - таким образом, все парсеры должны иметь какое-то представление об ошибке. Зачастую, входом парсера является текст, но это ни коим образом не является обязательным требованием, и наш `parse_non_empty` это совершенно законный парсер: он парсит списки в непустые списки, сигнализируя о неудаче сообщением с текстом ошибки.

По такому определению парсеры являются невероятно мощными инструментами: они позволяют производить проверки заранее, прямо на границе приложения и внешнего мира, и как только эти проверки пройдены, их не надо совершать снова! Rust разработчики знают об этой мощи, и они используют множество различных парсеров на постоянной основе:

- Библиотека [serde_json](https://github.com/serde-rs/json) предоставляет функцию `from_str` которая позволяет парсить данные в формате JSON в доменные типы

- Подобным образом [clap](https://github.com/clap-rs/clap) предоставляет набор парсер-комбинаторов для разбора аргументов командной строки

- Библиотеки вроде [diesel](https://github.com/diesel-rs/diesel) предоставляют механизм для парсинга значений, хранящихся во внешних хранилищах.

- Экосистема [actix-web](https://github.com/actix/actix-web) построена вокруг парсинга Rust типов из компонентов пути, строк запроса, HTTP заголовков и так далее.

Все эти библиотеки объединяет одно: они располагаются на границе между вашим приложением и внешним миром. Этот мир не общается в терминах типов-произведений и типов-сумм, он использует потоки байт, поэтому без парсинга тут не обойтись. И, совершая этот парсинг заранее, до того, как мы начинаем работать с этими данными, мы исключаем множество багов, часть из которых могут быть даже серьёзными уязвимостями.

У этого подхода, правда, есть один недостаток: иногда значения необходимо парсить задолго до того, как они действительно понадобятся.  Но есть и плюсы: в динамически-типизированных языках поддерживать в соответствии парсинг и бизнес логику довольно трудно без обширного покрытия тестами, многие из которых утомительно поддерживать. При этом, в статической системе типов проблема становится удивительно простой, как показано на примере `NonEmpty` выше: если парсинг и бизнес логика рассинхронизируются, то программа просто не скомпилируется.

## Опасность валидации

Надеюсь, к этому моменту вы хотя бы немного убедились, что парсинг предпочтительнее валидации, но у вас могут остаться смутные сомнения. Так ли плоха валидация, если система типов всё равно заставит вас расставить необходимые проверки? Возможно, сообщения об ошибках будут похуже, но в целом пара лишних проверок не сильно повредит, правда?

К сожалению, всё не так просто. Ad-hoc валидация ведёт к феномену, который [специалисты в области теоретико-языковой безопасности](http://langsec.org/) называют *парсинг наугад*. В статье 2016 года под названием [The Seven Turrets of Babel: A Taxonomy of LangSec Errors and How to Expunge Them](http://langsec.org/papers/langsec-cwes-secdev2016.pdf) авторы приводят следующее определение:

> Парсинг наугад - это антипаттерн, в котором код, выполняющий парсинг и валидацию, перемешан с бизнес логикой;  Разработчики пишут пачки проверок на входные аргументы, в надежде (без какого-либо формального обоснования), что так или иначе эти проверки поймают все "плохие" случаи.

Затем они объясняют проблемы, внутренне присущие подобной валидационной технике:

> Парсинг наугад неизбежно лишает программу возможности отвергнуть некорректные данные вместо того, чтобы их обрабатывать. Обнаруженные на поздних стадиях ошибки в переданных данных приводят к тому, что какая-то их часть уже оказывается обработанной, в результате чего итоговое состояние программы тяжело предугадать.

Другими словами, программа, которая не парсит заранее все необходимые ей данные, рискует обработать часть данных, затем обнаружить ошибку в другой части, и внезапно оказаться в ситуации, когда нужно откатить уже произведённые изменения, чтобы сохранить консистентность. Иногда - например, при работе с РСУБД, это возможно, но в общем случае - нет.

С первого взгляда может быть не очень понятно какое отношение парсинг наугад имеет к валидации - в конце концов, если сделать всю валидацию заранее, то можно избежать риска парсинга наугад. Проблема в том, что валидационный подход делает невероятно сложным, или даже, невозможным, определение действительно ли всё было проверено заранее или некоторые так называемые "невозможные" ситуации могут действительно возникнуть. Все места в программе обязаны предполагать, что возникновение исключения в любом месте не просто возможно, но зачастую необходимо.

Парсинг избегает этой проблемы, разделяя программу на две фазы: фазу парсинга и фазу исполнения, где ошибка, связанная с неверными входными данными, может произойти только в первой фазе. Множество оставшихся возможных ошибок намного меньше относительно ошибок во входных данных, и они могут быть обработаны с должным вниманием.

# Парсим, а не валидируем, на практике

До сих пор, весь текст был больше похож на рекламный слоган. Он говорит "Ты, дорогой читатель, должен парсить!", и, если я хорошо выполняю свою работу, то по крайней мере часть читателей должна со мной согласиться. Но даже если вы понимаете "что" и "почему", вы можете быть не до конца уверены насчёт "как".

Мой совет: фокусируйтесь на типах данных.

Предположим, вы пишите функцию, принимающую список пар ключей и значений, и вы не уверены в том, что делать в случае, если где-то в этом списке есть дубликаты по ключу. Одно из решений заключается в том, чтобы написать функцию, которая проверит, что в списке нет никаких дубликатов:

```rust
fn check_no_duplicate_keys<K: Eq, V>(xs: &[(K,V)]) { ... }
```

Но есть одно "но" -  эта проверка очень хрупкая: её чрезвычайно легко забыть. Из-за того, что результат функции не используется, её вызов всегда можно убрать, и полагающийся на неё код продолжит компилироваться. Лучшим решением было бы выбрать структуру данных, запрещающую дубликаты ключей по построению, например,  `HashMap`. Измените сигнатуру вашей функции так, чтобы она принимала  `HashMap` вместо списка пар, и реализуйте её так, как вы собирались.

После того, как вы это сделаете, место вызова этой функции, скорее всего, не скомпилируется, потому что в качестве аргумента ей всё ещё передаётся список пар. Если этот список передаётся в качестве одного из аргументов или если он получен как результат вызова какой-то другой функции, вам нужно продолжать заменять список на `HashMap` по всей цепочке вызовов. В конце концов, вы либо достигните точки, где это значение создаётся, либо найдёте место, где дубликаты должны быть разрешены. В этом месте вы должны вставить модифицированную версию `check_no_duplicate_keys`:

```rust
fn check_no_duplicate_keys<K: Eq, V>(xs: &[(K,V)]) -> HashMap<K,V> { ... }
```

И теперь проверка *не может* быть удалена, потому что результат функции необходим для продолжения работы программы!

В этом гипотетическом сценарии видны две простые идеи:

1. **Используйте структуры данных, которые делают некорректные состояния невозможными.** Моделируйте ваши данные используя наиболее точные структуры данных. Если искоренение некоторой возможности слишком сложно используя текущее представление, попробуйте использовать другое представление, которое позволяет более просто описать свойства, которые вы хотите выразить. Не бойтесь рефакторить

2. **Выносите бремя проверок как можно выше, но не дальше.** Переведите ваши данные в наиболее точное представление так рано, как это возможно. В идеале, это должно происходить на границе вашей системы, до того, как *любые* данные начали обрабатываться.

    Если более точное представление данных нужно только в одной конкретной ветке кода, то парсите данные как только эта ветвь была взята. Используйте тип-суммы с умом, чтобы ваши типы данных отражали возможные результаты выполнения кода.

Другими словами, пишите функции с типами данными, которые вы *хотели бы* иметь, а не те, которые у вас есть. Процесс проектирования тогда становится упражнением в устранении этого различия, над которым часто нужно работать с обоих концов, пока вы не сойдётесь на каком-то компромиссном решении. Не бойтесь итеративно изменять части программы , ведь в итоге в процессе рефакторинга вы можете даже узнать что-то новое!

Вот несколько дополнительных советов, расположенных в произвольном порядке:

- **Позвольте вашим типам данных информировать ваш код, не позволяйте коду контролировать типы данных.** Боритесь с искушением запихнуть `bool` в структуру потому что он нужен в функции, которую вы пишете (*На эту тему очень рекомендую к прочтению статью [**Булева слепота**](https://existentialtype.wordpress.com/2011/03/15/boolean-blindness/), прим. пер.*). Не бойтесь рефакторить код, чтобы использовать более правильное представление данных - система типов удостоверится, что вы учли все места, которые требуют изменения, и это спасёт вас от головной боли в дальнейшем

- **Относитесь к функциям без возвращаемого результата или с типом `Result<(), Error>` с большим подозрением.** Иногда они совершенно необходимы, потому что они могут производить императивный эффект без какого-либо разумного результата, но если единственная цель такой функции это вызвать ошибку, скорее всего есть лучший путь.

- **Не бойтесь парсить данные в несколько этапов.** Избегание парсинга наугад всего лишь означает, что вы не должны работать с выходными данными до того, как полностью их распарсите, а не то, что вы не можете использовать часть входных данных, чтобы решить, как парсить остальное. Множество полезных парсеров - контекстно-зависимы.

- **Избегайте денормализованного представления данных, *особенно*, если они изменяемые** Дублирование одних и тех же данных в множестве мест позволяет элементарно привести систему в некорректное состояние: данные в разных местах могут рассинхронизироваться. Стремитесь к единственному источнику истины

    - **Держите денормализованные данные за границей абстракций.** Если денормализованное состояние абсолютно необходимо, используйте инкапсуляцию, чтобы существовал только небольшой доверенный модуль, единственная роль которого заключается в поддержании данных в синхронизированном состоянии.

- **Используйте абстрактные типы данных, чтобы ваши валидаторы "выглядели как" парсеры.** Иногда, сделать данные действительно непредставимыми совершенно непрактично, учитывая возможности которые даёт ваш язык программирования, например, проверка, что число попадает в некоторый диапазон. В этом случае, используйте `декоратор (newtype)` с умным конструктором, чтобы сделать псевдо-парсер из валидатора.

Как обычно, используйте здравый смысл. Иногда просто не стоит ломать весь код и переписывать всё приложение чтобы избежать единственного потенциального `error "impossible"` где-то в коде - просто не забывайте относиться к таким местам как к радиоактивной субстанции, которой они и являются, и обходитесь с ними со всей необходимой аккуратностью. В самом крайнем случае, оставьте хотя бы комментарий, чтобы задокументировать инварианты для тех, кто в будущем будет модифицировать этот код.

# Резюме

Вот и всё, в общем-то. Надеюсь, эта статья об использовании преимуществ системы типов не требует докторской степени для понимания, и даже не требует последних и крутейших фишек из более мощных систем типов - хотя иногда они могут сильно помочь! Иногда наибольшим препятствием для использования системы типов на полную катушку является простое незнание, что это возможно.

Ни одна из идей этой статьи не нова. На самом деле, основная идея - "пишите тотальные функции" - концептуально очень проста. Несмотря на это, удивительно сложно придумать практичные, понятные объяснения того, как я пишу код в таком стиле. Можно легко потратить кучу времени, разговаривая об абстрактных концепциях - многие из которых очень ценны! - не сообщив ничего полезного о *процессе*. Я надеюсь, что эта статья - это небольшой шажок в этом направлении.

К сожалению, я знаю не так много ресурсов на эту тему, но я знаю одно: я никогда не стесняюсь порекомендовать фантастическую статью Мэта Парсона  [Type Safety Back and Forth](https://www.parsonsmatt.org/2017/10/11/type_safety_back_and_forth.html). Если вам нужна ещё одна доступная статью на тему этих идей, то я крайне рекомендую прочитать её. Для намного более серьёзного погружения в эти идеи я могу порекомендовать работу Мэта Нунана [Ghosts of Departed Proofs](https://kataskeue.com/gdp.pdf), в которой изложены некоторые более продвинутые техники по выражению более сложных инвариантов, нежели чем те, что я привела в статье.

В заключение, я хотел бы заметить, что рефакторинг в стиле, который описан в статье, не всегда прост. Примеры которые я использовал - просты, но реальная жизнь зачастую куда менее прямолинейна. Даже люди, имеющие большой опыт в TypeDD, могут испытывать затруднения в выражении некоторых инвариантов в системе типов, так что не падайте духом, если у вас не получается решить проблему так, как вам хочется. Считайте эти принципы идеалом, к которому надо стремиться, а не строгому обязательному требованию. Всё, что требуется - хотя бы попытаться.

---

## От переводчика

Некоторые примеры компилируются растом, но не проходят проверки боррочекера - это нормально, и сделано умышленно для упрощения и получения более наглядных примеров.
