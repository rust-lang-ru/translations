# Изучение Rust: компилятор - ваш друг

Язык программирования Rust нацелен не только на практичность, но и на удобство для работающих с ним людей. Это не только повышает продуктивность, но и помогает в изучении языка! Особенность, на которую я хочу обратить внимания - это анализатор заимствований компилятора Rust. Он позволяет устранить большое число ошибок, связанных с памятью, за счёт корректности распределения памяти и использования указателей. В то же время, он кажется источником разочарования, что отражено в фразе, которую мы часто видим: "борьба с компилятором".

Причина, по который мы можете использовать компилятор Rust в качестве персонального ментора при изучении основных концепций языка, - это информация, содержащаяся в сообщениях об ошибках. Я хочу продемонстрировать это на простом примере о изменяемости и заимствовании. Вы также можете попробовать сделать это в [песочнице](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=208af39ba7facc50154027834691ee74).

Мы объявим переменную `item` и привяжем к ней строку, содержащую слово *sausage*. Собака, Белка, проходит рядом и хочет откусить сосиску. Мы обозначим это усечением строки.

```rust
fn main() {
    let item = String::from("sausage");
    println!("Belka finds a {} and takes a bite off of it.", item);
    belka_takes_bite_off(item);

}

fn belka_takes_bite_off(sausage: String) {
    sausage.truncate(5);
    println!("The left over {} lies in front of her.", sausage);
}
```

Мы запустим код, но получим ошибку:

```
 Compiling playground v0.0.1 (/playground)
error[E0596]: cannot borrow `sausage` as mutable, as it is not
declared as mutable
 --> src/main.rs:9:5
  |
8 | fn belka_takes_bite_off(sausage: String) {
  |                       -------- help: consider changing this to be
mutable: `mut sausage`
9 |     sausage.truncate(5);
  |     ^^^^^^^^ cannot borrow as mutable

error: aborting due to previous error

For more information about this error, try `rustc --explain E0596`.
error: could not compile `playground`.

To learn more, run the command again with --verbose.
```

Мы получили от компилятора ошибку с номером `E0596` и соответствующим сообщением: `cannot borrow 'sausage' as mutable, as it is not declared as mutable`. В сообщении содержится не только название операции, которая нарушила правила Rust, но и описывается причина, почему эта операция невозможна. Дополнительно, чтобы точнее дать более полную информацию об ошибке, копилятор  указывает строку, где расположена ошибочная операция и подчёркнута затронутая переменная.

Но сообщение не останавливается на этом. Нам также предлагают помощь: мы должны рассмотреть возможно сделать параметр изменяемым. Место в коде, где это можно сделать, также подчёркнуто.

Из этого сообщения об ошибку, мы узнали, что переменные должны быть объявлены изменяемыми, если мы хотим изменять связанные с ними значения.

Благодаря специальным инструкциям, эта ошибка исправляется достаточно быстро при помощи изменения 8 строки следующим образом:

```rust
fn belka_takes_bite_off(mut sausage: String) {
```

Запуск программы теперь выведет следующий текст:

```
Belka finds a sausage and takes a bite off of it.
The left over sausa lies in front of her.
```

Изменение значения переменной прошло успешно.

<hr>

Компилятор не всегда предлагает такого рода помощь. Часто действия, необходимые для устранения ошибки, зависят от характеристик и назначения ваших данных, а также целей программы.

Давайте добавим в нашу картину другую собаку, Стрелку, которая хочет съесть оставшуюся часть сосиски. Вы можете делать всё в этой [песочнице](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=f8c6e55e62b90a8f3400198d3bd0a410).

```rust
fn main() {
    let item = String::from("sausage");
    println!("Belka finds a {} and takes a bite off of it.", item);
    belka_takes_bite_off(item);
    println!("Strelka steals the left overs.");
    strelka_takes_and_eats(item);

}

fn belka_takes_bite_off(mut sausage: String) {
    sausage.truncate(5);
    println!("The left over {} lies in front of her.", sausage);
}

fn strelka_takes_and_eats(mut sausage: String) {
    println!("Strelka swallows the {} in one bite.", sausage);
    sausage.clear();
    println!("Length of left over: {}", sausage.len());
}
```

При запуске этой программы, мы получаем следующее сообщение об ошибке: мы пытаемся использовать значение, которое уже было перемещено.

```
 Compiling playground v0.0.1 (/playground)
error[E0382]: use of moved value: `item`
 --> src/main.rs:6:29
  |
2 |     let item = String::from("sausage");
  |         ---- move occurs because `item` has type
`std::string::String`, which does not implement the `Copy` trait
3 |     println!("Belka finds a {} and takes a bite off of it.", item);
4 |     belka_takes_bite_off(item);
  |                        ---- value moved here
5 |     println!("Strelka steals the left overs.");
6 |     strelka_takes_and_eats(item);
  |                             ^^^^ value used here after move

error: aborting due to previous error

For more information about this error, try `rustc --explain E0382`.
error: could not compile `playground`.

To learn more, run the command again with --verbose.
```

На этот раз мы снова узнаём что стало причиной ошибки: используемый тип не реализует трейт `Copy`, но мы не получаем помощи. В этой ситуации, чтобы исправить ошибку, возникает желание просто реализовать трейт `Copy` так как он явно отсутствует. Отсутствующая реализация является причиной, почему наша предполагаемая операция вызывает ошибку. Это {strong}не означает, что реализация отсутствующего трейта является лучим способом исправления ошибки!

Но сообщение об ошибке всё же может нам помочь. запустим `rustc --explain E0382`, чтобы получить более детальное описание этой ошибки и варианты решения проблемы:

- использование изменяемой ссылки с `&mut`,
- реализация трейта `Clone` и дублирование данных,
- наличие совместного владения с подсчётом ссылок.

В этой точке мы достигли границ руководства компилятора. Нам предоставили информацию, но решение о дальнейших действиях остаётся за нами и обычно является результатом самой программы.

Посмотрим на нашу историю: есть одна сосиска, которую частично ест одна собака, а затем её полностью съедает вторая. Клонирование данных входит за пределы вопроса, так как должны получиться два одинаковых набора данных, по одному на каждую собаку. Было бы не плохо, если бы каждая собака съела целую сосиску, но наша история идёт не так. Говоря абстрактнее: имя копию данных в памяти, когда это не необходимо, является тратой памяти. Если набор данных требует изменения несколькими владельцами, изменения на различных копиях данных - это не то же самое, что изменения оригинала. Реализация трейта `Copy` также привела бы к клонированию данных.

Фактически, для этой программы, передавать владение не требуется: собакам не важно, кому принадлежит сосиска, пока у них может быть её часть, что делает совместное владение лишним. Так что нам нужна изменяемая ссылка `&mut`.

Мы изменили строки 10 и 15:

```rust
fn belka_takes_bite_off(sausage: &mut String) {
```

```rust
fn strelka_takes_and_eats(sausage: &mut String) {
```

И сборка программы упала:

```
   Compiling playground v0.0.1 (/playground)
error[E0308]: mismatched types
 --> src/main.rs:4:24
  |
4 |     belka_takes_bite_off(item);
  |                        ^^^^
  |                        |
  |                        expected `&mut std::string::String`, found
                           struct `std::string::String`
  |                        help: consider mutably borrowing here:
                           `&mut item`

error[E0308]: mismatched types
 --> src/main.rs:6:29
  |
6 |     strelka_takes_and_eats(item);
  |                             ^^^^
  |                             |
  |                             expected `&mut std::string::String`,
                                found struct `std::string::String`
  |                             help: consider mutably borrowing here:
                                `&mut item`

error: aborting due to 2 previous errors

For more information about this error, try `rustc --explain E0308`.
error: could not compile `playground`.

To learn more, run the command again with --verbose.
```

При внесении необходимых изменений, чтобы иметь изменяемые ссылки на элемент вместо передачи его владения, новички часто спотыкаются о том, что `T` и `&mut T` - это два разных типа. Но не волнуйтесь, компилятор по-прежнему полезен и поможет вам правильно выбрать типы: мы узнаем, что если функция ожидает определённый тип, вызов функции должен происходить с тем же типом.

Поэтому мы меняем переменную `item` на `&mut item` в вызовах функций:

```rust
    belka_takes_bite_off(&mut item);
    println!("Strelka steals the left overs.");
    strelka_takes_and_eats(&mut item);
```

При изучении Rust часто приходится ходить туда-сюда, вносить изменения, запускать код, получать сообщение об ошибке, повторять. Так что последних внесенных нами изменений все ещё недостаточно. Запуск кода даёт нам следующее сообщение об ошибке:

```
   Compiling playground v0.0.1 (/playground)
error[E0596]: cannot borrow `item` as mutable, as it is not declared
as mutable
 --> src/main.rs:4:24
  |
2 |     let item = String::from("sausage");
  |         ---- help: consider changing this to be mutable: `mut item`
3 |     println!("Belka finds a {} and takes a bite off of it.", item);
4 |     belka_takes_bite_off(&mut item);
  |                        ^^^^^^^^^ cannot borrow as mutable

error[E0596]: cannot borrow `item` as mutable, as it is not declared
as mutable
 --> src/main.rs:6:29
  |
2 |     let item = String::from("sausage");
  |         ---- help: consider changing this to be mutable:
                 `mut item`
...
6 |     strelka_takes_and_eats(&mut item);
  |                             ^^^^^^^^^ cannot borrow as mutable

error: aborting due to 2 previous errors

For more information about this error, try `rustc --explain E0596`.
error: could not compile `playground`.

To learn more, run the command again with --verbose.
```

Мы уже знакомы с этой ошибкой, и необходимое объявление изменяемости делается быстро:

```rust
fn main() {
    let mut item = String::from("sausage");
    println!("Belka finds a {} and takes a bite off of it.", item);
    belka_takes_bite_off(&mut item);
    println!("Strelka steals the left overs.");
    strelka_takes_and_eats(&mut item);

}

fn belka_takes_bite_off(sausage: &mut String) {
    sausage.truncate(5);
    println!("The left over {} lies in front of her.", sausage);
}

fn strelka_takes_and_eats(sausage: &mut String) {
    println!("Strelka swallows the {} in one bite.", sausage);
    sausage.clear();
    println!("Length of left over: {}", sausage.len());
}
```

Наконец, запуск кода дает нам такой результат:

```
Belka finds a sausage and takes a bite off of it.
The left over sausa lies in front of her.
Strelka steals the left overs.
Strelka swallows the sausa in one bite.
Length of left over: 0
```

Такое вышеупомянутое ходение по кругу из небольших изменений, компиляции и получения ошибок может расстраивать. Но даже если вам кажется, что компилятор работает против вас, он даёт вам решение: вам либо напрямую дают решение, либо предоставляют информацию, необходимую для принятия решения о дальнейших действиях. Поначалу это может показаться борьбой, но на самом деле компилятор - это ваш спарринг-партнёр, который держит перчатку, пока вы практикуете свои удары. Со временем, вы становитесь лучше, частота появления ошибок снижается, так как вы узнаёте где нужно сделать ссылку, где объявить мутабельность, когда memcopy более приемлемо, чем создание ссылки. Со временем, ваши удары будут сильнее.

Компилятор - строгий тренер. Но в конце даже новичок сможет создавать безопасную для памяти программу, что уже является достижением!

Выводы из этого сообщения в блоге:

- следуйте справке, содержащейся в сообщении об ошибке,
- если справки об ошибке нет, запустите `rustc --explain` и прочитайте вывод прежде чем исправить ошибку,
- компилятор Rust - ваш дружелюбный спарринг-партнёр.

## Следите за обновлениями …

В следующей части этой серии, мы поговорим о пути изучения Rust для разработчиков на C/C++. Мы поговорим о преимуществах и подводных камнях, на которые следует обратить внимание.

(реклама про тренинги и подписку)
