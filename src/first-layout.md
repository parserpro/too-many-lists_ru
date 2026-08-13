# Базовая компоновка данных

Ладно, что же такое связный список? В сущности, это набор фрагментов данных в куче (тсс, люди из ядра!), которые последовательно указывают друг на друга.

Связные списки — это то, к чему процедурным программистам не стоит приближаться даже на расстояние десяти футов, и то, что функциональные программисты используют для всего подряд. Поэтому кажется справедливым спросить у функциональных программистов, как они определяют связный список. Скорее всего, они дадут вам примерно такое определение:

```haskell
List a = Empty | Elem a (List a)
```

Это примерно читается как «Список — либо пустой, либо элемент, за которым следует Список». Это рекурсивное определение, выраженное в виде *суммарного типа* — заумное название для «типа, который может принимать разные значения, потенциально разных типов». В Rust суммарные типы называются `enum`! Если вы пришли из C‑подобного языка, это тот самый `enum`, который вы знаете и любите, — только на стероидах. Итак, давайте перенесём это функциональное определение в Rust!

Пока обойдёмся без обобщений, чтобы всё было проще. Будем поддерживать хранение только знаковых 32‑битных целых чисел:

```rust ,ignore
// в first.rs

// pub означает, что мы хотим, чтобы люди вне этого модуля могли использовать List
pub enum List {
    Empty,
    Elem(i32, List),
}
```

*фух*, голова кругом от всего этого. Давайте продолжим и попробуем это скомпилировать:

```text
> cargo build

error[E0072]: recursive type `first::List` has infinite size
 --> src/first.rs:4:1
  |
4 | pub enum List {
  | ^^^^^^^^^^^^^ recursive type has infinite size
5 |     Empty,
6 |     Elem(i32, List),
  |               ---- recursive without indirection
  |
  = help: insert indirection (e.g., a `Box`, `Rc`, or `&`) at some point to make `first::List` representable
```

Ну… Не знаю, как вы, а я определённо чувствую себя преданным сообществом функционального программирования.

Если мы действительно прочитаем сообщение об ошибке (после того как переживём весь этот момент предательства), то увидим, что rustc фактически прямо говорит нам, как решить проблему:

> insert indirection (e.g., a `Box`, `Rc`, or `&`) at some point to make `first::List` representable

Ладно, `Box`. Что это такое? Давайте погуглим `rust box`...

> [std::boxed::Box - Rust](https://doc.rust-lang.org/std/boxed/struct.Box.html)

*так-с*, поглядим…

> `pub struct Box<T>(_);`
>
> Тип указателя для выделения памяти в куче.
> Подробнее см. в [документации на уровне модуля](https://doc.rust-lang.org/std/boxed/).

*кликает по ссылке*

> `Box<T>`, в обиходе называемый «боксом», предоставляет простейшую форму выделения памяти в куче в Rust. Боксы обеспечивают владение этим выделением и удаляют своё содержимое, когда выходят из области видимости.
>
> Примеры
>
> Создание бокса:
>
> `let x = Box::new(5);`
>
> Создание рекурсивной структуры данных:
>
```rust
#[derive(Debug)]
enum List<T> {
    Cons(T, Box<List<T>>),
    Nil,
}
```
>
```rust ,ignore
fn main() {
    let list: List<i32> = List::Cons(1, Box::new(List::Cons(2, Box::new(List::Nil))));
    println!("{:?}", list);
}
```
>
> Это выведет `Cons(1, Box(Cons(2, Box(Nil))))`.
>
> Рекурсивные структуры должны быть «забоксованы», потому что, если бы определение Cons выглядело так:
>
> `Cons(T, List<T>),`
>
> оно не сработало бы. Дело в том, что размер List зависит от количества элементов в списке, а значит, мы не знаем, сколько памяти выделить под Cons. Добавив Box, у которого размер фиксирован, мы точно знаем, какого размера должен быть Cons.

Вау, ну это, пожалуй, самая релевантная и полезная документация, которую я когда‑либо видел. Буквально первое, что написано в документации, — это *ровно то, что мы пытаемся написать, почему это не работает и как это исправить*.

Чёрт, документация — это сила!

Ок, давайте сделаем так:

```rust ,ignore
pub enum List {
    Empty,
    Elem(i32, Box<List>),
}
```

```text
> cargo build

   Finished dev [unoptimized + debuginfo] target(s) in 0.22s
```

Ого, оно собралось!

…но на самом деле это очень неудачное определение List — и на то есть несколько причин.

Рассмотрим список из двух элементов:


```text
[] = Stack
() = Heap

[Elem A, ptr] -> (Elem B, ptr) -> (Empty, *junk*)
```

Есть две ключевые проблемы:

* Мы выделяем узел, который по сути просто говорит: «На самом деле я не узел».
* Один из наших узлов вообще не выделяется в куче.

На первый взгляд кажется, что эти две вещи компенсируют друг друга: мы выделяем в куче лишний узел, но зато один из узлов вообще не требует выделения в куче. Однако давайте рассмотрим следующую возможную компоновку нашего списка:

```text
[ptr] -> (Elem A, ptr) -> (Elem B, *null*)
```

В этой компоновке мы теперь безусловно выделяем узлы в куче. Ключевое отличие — отсутствие *мусора* из нашей первой компоновки. Что это за мусор? Чтобы разобраться, нужно посмотреть, как enum располагается в памяти.

В общем случае, если у нас есть enum вроде:

```rust ,ignore
enum Foo {
    D1(T1),
    D2(T2),
    ...
    Dn(Tn),
}
```

У `Foo` должно храниться какое‑то целое число, чтобы указать, какой *вариант* enum он представляет (`D1`, `D2`, … `Dn`). Это *тег* enum. Кроме того, ему нужно достаточно места, чтобы вместить *самый большой* из типов `T1`, `T2`, … `Tn` (плюс немного дополнительного места для соблюдения требований выравнивания).

Главный вывод здесь в том, что, хотя `Empty` — это по сути всего один бит информации, он неизбежно занимает столько места, сколько нужно для указателя и элемента: ведь в любой момент он должен быть готов превратиться в `Elem`. Поэтому в первой компоновке мы выделяем в куче лишний элемент, который по факту заполнен «мусором» и потребляет чуть больше памяти, чем во второй компоновке.

Тот факт, что один из наших узлов вообще не выделяется в памяти, тоже, как ни странно, *хуже*, чем если бы мы всегда его выделяли. Всё из‑за того, что это даёт нам *неоднородную* компоновку узлов. На операциях push и pop это почти не сказывается, а вот на разбиении и слиянии списков — очень даже.

Рассмотрим, как происходит разбиение списка в обеих компоновках:

```text
layout 1:

[Elem A, ptr] -> (Elem B, ptr) -> (Elem C, ptr) -> (Empty *junk*)

split off C:

[Elem A, ptr] -> (Elem B, ptr) -> (Empty *junk*)
[Elem C, ptr] -> (Empty *junk*)
```

```text
layout 2:

[ptr] -> (Elem A, ptr) -> (Elem B, ptr) -> (Elem C, *null*)

split off C:

[ptr] -> (Elem A, ptr) -> (Elem B, *null*)
[ptr] -> (Elem C, *null*)
```

При разбиении в компоновке 2 нужно просто скопировать указатель на B в стек и обнулить старое значение. В компоновке 1 в итоге делается то же самое, но ещё приходится копировать C из кучи в стек. Слияние — это тот же процесс, только в обратном порядке.

Одно из немногих преимуществ связного списка в том, что элемент можно создать прямо внутри узла, а потом свободно перемещать его между списками, вообще не трогая сам элемент. Вы просто возитесь с указателями — и элемент как будто «перемещается». Компоновка 1 лишает нас этого преимущества.

Ладно, я вполне убеждён, что компоновка 1 — плохая идея. Как нам переписать List? Ну, можно сделать примерно так:

```rust ,ignore
pub enum List {
    Empty,
    ElemThenEmpty(i32),
    ElemThenNotEmpty(i32, Box<List>),
}
```

Hopefully this seems like an even worse idea to you. Most notably, this really
complicates our logic, because there is now a completely invalid state:
`ElemThenNotEmpty(0, Box(Empty))`. It also *still* suffers from non-uniformly
allocating our elements.

However it does have *one* interesting property: it totally avoids allocating
the Empty case, reducing the total number of heap allocations by 1. Unfortunately,
in doing so it manages to waste *even more space*! This is because the previous
layout took advantage of the *null pointer optimization*.

We previously saw that every enum has to store a *tag* to specify which variant
of the enum its bits represent. However, if we have a special kind of enum:

```rust,ignore
enum Foo {
    A,
    B(ContainsANonNullPtr),
}
```

the null pointer optimization kicks in, which *eliminates the space needed for
the tag*. If the variant is A, the whole enum is set to all `0`'s. Otherwise,
the variant is B. This works because B can never be all `0`'s, since it contains
a non-zero pointer. Slick!

Can you think of other enums and types that could do this kind of optimization?
There's actually a lot! This is why Rust leaves enum layout totally unspecified.
There are a few more complicated enum layout optimizations that Rust will do for
us, but the null pointer one is definitely the most important!
It means `&`, `&mut`, `Box`, `Rc`, `Arc`, `Vec`, and
several other important types in Rust have no overhead when put in an `Option`!
(We'll get to most of these in due time.)

So how do we avoid the extra junk, uniformly allocate, *and* get that sweet
null-pointer optimization? We need to better separate out the idea of having an
element from allocating another list. To do this, we have to think a little more
C-like: structs!

While enums let us declare a type that can contain *one* of several values,
structs let us declare a type that contains *many* values at once. Let's break
our List into two types: A List, and a Node.

As before, a List is either Empty or has an element followed by another List.
By representing the "has an element followed by another List" case by an
entirely separate type, we can hoist the Box to be in a more optimal position:

```rust ,ignore
struct Node {
    elem: i32,
    next: List,
}

pub enum List {
    Empty,
    More(Box<Node>),
}
```

Let's check our priorities:

* Tail of a list never allocates extra junk: check!
* `enum` is in delicious null-pointer-optimized form: check!
* All elements are uniformly allocated: check!

Alright! We actually just constructed exactly the layout that we used to
demonstrate that our first layout (as suggested by the official Rust
documentation) was problematic.

```text
> cargo build

warning: private type `first::Node` in public interface (error E0446)
 --> src/first.rs:8:10
  |
8 |     More(Box<Node>),
  |          ^^^^^^^^^
  |
  = note: #[warn(private_in_public)] on by default
  = warning: this was previously accepted by the compiler but
    is being phased out; it will become a hard error in a future release!
```

:(

Rust is mad at us again. We marked the `List` as public (because we want people
to be able to use it), but not the `Node`. The problem is that the internals of
an `enum` are totally public, and we're not allowed to publicly talk about
private types. We could make all of `Node` totally public, but generally in Rust
we favour keeping implementation details private. Let's make `List` a struct, so
that we can hide the implementation details:

```rust ,ignore
pub struct List {
    head: Link,
}

enum Link {
    Empty,
    More(Box<Node>),
}

struct Node {
    elem: i32,
    next: Link,
}
```

Because `List` is a struct with a single field, its size is the same as that
field. Yay zero-cost abstractions!

```text
> cargo build

warning: field is never used: `head`
 --> src/first.rs:2:5
  |
2 |     head: Link,
  |     ^^^^^^^^^^
  |
  = note: #[warn(dead_code)] on by default

warning: variant is never constructed: `Empty`
 --> src/first.rs:6:5
  |
6 |     Empty,
  |     ^^^^^

warning: variant is never constructed: `More`
 --> src/first.rs:7:5
  |
7 |     More(Box<Node>),
  |     ^^^^^^^^^^^^^^^

warning: field is never used: `elem`
  --> src/first.rs:11:5
   |
11 |     elem: i32,
   |     ^^^^^^^^^

warning: field is never used: `next`
  --> src/first.rs:12:5
   |
12 |     next: Link,
   |     ^^^^^^^^^^

```

Alright, that compiled! Rust is pretty mad, because as far as it can tell,
everything we've written is totally useless: we never use `head`, and no one who
uses our library can either since it's private. Transitively, that means Link
and Node are useless too. So let's solve that! Let's implement some code for our
List!
