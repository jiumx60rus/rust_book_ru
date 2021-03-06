% Владение

Эта глава является одной из трёх, описывающих систему владения ресурсами Rust.
Эта система представляет собой наиболее уникальную и привлекательную особенность
Rust, о которой разработчики должны иметь полное представление. Владение - это
то, как Rust достигает своей главной цели - безопасности памяти. Система
владения включает в себя несколько различных концепций, каждая из которых
рассматривается в своей собственной главе:

* владение, её вы сейчас читаете
* [заимствование][borrowing], и ассоциированная с ним возможность "ссылки"
* [время жизни][lifetimes], расширение концепции заимствования

Эти три главы взаимосвязаны, и их порядок важен. Вы должны будете освоить все
три главы, чтобы полностью понять систему владения.

[borrowing]: references-and-borrowing.html
[lifetimes]: lifetimes.html

# Мета

Прежде чем перейти к деталям, отметим два важных нюанса о системе владения.

Rust сфокусирован на безопасности и скорости. Это достигается за счёт
"абстракций с нулевой стоимостью" ("zero-cost abstractions"), а значит, в Rust
стоимость абстракций должна быть настолько минимальной, насколько это возможно,
без ущерба для работоспособности. Система владения ресурсами - это яркий пример
абстракции с нулевой стоимостью. Весь анализ, о котором мы будем говорить в этом
руководстве, выполняется _во время компиляции_. Вы не платите хоть сколько-
нибудь времени исполнения за какую-либо из возможностей.

Тем не менее, эта система всё же имеет определённую стоимость: кривая обучения.
Многие новые пользователи Rust испытывает то, что мы называем "борьба с проверкой
заимствования", когда компилятор Rust отказывается компилировать программу,
которая по мнению автора является абсолютно правильной. Это часто происходит
потому, что мысленное представление программиста о том, как должно работать
владение, не совпадает с реальными правилами, которыми оперирует Rust. Вы,
наверное, поначалу также будете испытывать подобные трудности. Однако существует
и хорошая новость: более опытные разработчики на Rust сообщают, что чем больше
они работают с правилами системы владения, тем меньше они борются с проверкой
заимствования.

Имея это в виду, давайте перейдём к изучению системы владения.

# Владение

[Связанные имена][bindings] имеют одну особенность в Rust: они "обладают
правом собственности" на то, с чем они связаны. Это означает, что, когда
связывание выходит за пределы области видимости, ресурс, с которым оно связано,
будут освобождён. Например:

```rust
fn foo() {
    let v = vec![1, 2, 3];
}
```

Когда `v` входит в область видимости, создаётся новый [`Vec<T>`][vect]. В данном
случае вектор также выделяет из [кучи][heap] пространство для трёх элементов.
Когда `v` выходит из области видимости, в конце `foo()`, Rust очищает все,
связанное с вектором, даже динамически выделенную память. Это происходит
детерминировано, в конце области видимости.

[vect]: http://doc.rust-lang.org/std/vec/struct.Vec.html
[heap]: the-stack-and-the-heap.html
[bindings]: variable-bindings.html

<a name="move-semantics"></a>
# Семантика перемещения

Хотя тут есть некоторые тонкости: Rust гарантирует, что существует _ровно одна_
привязка к какому-либо ресурсу. Например, если у нас есть вектор, то мы можем
присвоить этот вектор другой привязке:
Важным аспектом [владения][ownership] является "семантика перемещения".
Семантика перемещения контролирует, как и когда право собственности перемещается
между привязками (связываниями).

```rust
let v = vec![1, 2, 3];

let v2 = v;
```

Но, если после этого мы попытаемся использовать `v`, то получим ошибку:

```rust,ignore
let v = vec![1, 2, 3];

let v2 = v;

println!("v[0] = {}", v[0]);
```

Ошибка выглядит следующим образом:

```text
error: use of moved value: `v`
println!("v[0] = {}", v[0]);
                        ^
```

То же самое произойдёт, если мы определим функцию, которая принимает право
владения, и попробуем использовать что-то после того как мы передали это что-то
в качестве аргумента в эту функцию:

```rust,ignore
fn take(v: Vec<i32>) {
    // что будет здесь не очень важно
}

let v = vec![1, 2, 3];

take(v);

println!("v[0] = {}", v[0]);
```

Та же самая ошибка: "use of moved value" ("используется перемещённое значение").
Когда мы передаём право владения куда-то ещё, мы как бы говорим, что мы
"перемещаем" то, на что ссылаемся. При этом не нужно указывать какую-либо
специальную аннотацию, Rust делает это по умолчанию.

## Детали

Причина, по которой мы не можем использовать привязку после того как мы её
переместили, трудно заметная, но очень важная. Когда мы пишем код вроде этого:

```rust
let v = vec![1, 2, 3];

let v2 = v;
```

Первая строка создаёт некоторые данные для вектора в [стеке][sh], `v`. Данные
самого вектора, однако, сохраняются в [куче][sh], и поэтому стековые данные
содержат указатель на данные в куче. Когда мы перемещаем `v` в `v2`, то
создаётся копия стековых данных, для `v2`. Что будет означать, что два указателя
ссылаются на расположенный в куче вектор. Такое поведение могло бы быть
проблемой: оно нарушало бы гарантии безопасности Rust, привнося гонки данных.
Поэтому Rust запрещает использование `v` после того как мы выполнили его
перемещение.

[sh]: the-stack-and-the-heap.html

Важно также отметить, что оптимизация может удалить фактическую (точную) копию
байтов, в зависимости от обстоятельств. Так что это может быть не так уж
неэффективно, как выглядит на первый взгляд.

## Типы, реализующие типаж `Copy`

Мы установили, что, как только право собственности передаётся другой привязке,
то вы больше не можете использовать оригинальную привязку. Тем не менее,
существует [типаж][traits], который изменяет такое поведение, и он называется
`Copy`. Мы ещё не обсуждали типажи, но пока вы можете думать о них как об
аннотациях к конкретному типу, которые придают дополнительное поведение.
Например:

```rust
let v = 1;

let v2 = v;

println!("v = {}", v);
```

В этом примере `v` связан с типом `i32`. Этот тип реализует типаж `Copy`.
Это означает, что когда мы присваиваем привязку `v` привязке `v2`, будет создана
копия данных, как и при перемещении. Но, в отличие от перемещения, мы 
можем использовать `v` в дальнейшем. Это происходит потому, что в `i32` нет 
указателей на данные в каком-либо другом месте. При таком копировании 
создаётся полная копия.

Мы будем обсуждать, как сделать свои собственные типы, реализующие типаж `Copy`
в разделе [Типажи][traits].

[traits]: traits.html

# Больше, чем владение

Конечно, если бы нам нужно было вернуть право владения обратно из функции, то мы
бы написали:

```rust
fn foo(v: Vec<i32>) -> Vec<i32> {
    // делаем что-либо с v

    // возвращаем право владения
    v
}
```

Это сильно утомляет. Функция становится тем хуже, чем больше прав владения она
хочет забрать себе:

```rust
fn foo(v1: Vec<i32>, v2: Vec<i32>) -> (Vec<i32>, Vec<i32>, i32) {
    // делаем что-нибудь с v1 и v2

    // возвращаем право владения и результат нашей функции
    (v1, v2, 42)
}

let v1 = vec![1, 2, 3];
let v2 = vec![1, 2, 3];

let (v1, v2, answer) = foo(v1, v2);
```

Брр! Возвращаемый тип, возвращающая строка, и вызов функции получается намного 
более сложным.

К счастью, Rust предлагает такую возможность, как заимствование, которая
помогает нам  решить эту проблему. Это тема следующего раздела!











