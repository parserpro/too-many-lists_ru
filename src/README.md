# Изучаем Rust реализуя большое количество связных списков

> Хотите сообщить об ошибке или забрать все исходники разом?
> [Заходите на Github!](https://github.com/parserpro/too-many-lists_ru)

> **ПРИМЕЧАНИЕ**: Нынешнее издание этой книги написано с опорой на стандарт Rust 2018,
> который впервые появился в компиляторе rustc 1.31 (8 декабря 2018 года).
> Если ваш набор инструментов Rust достаточно свежий, в файле Cargo.toml,
> который создаёт команда `cargo new`, должна быть строка `edition = "2018"`
> (а если вы читаете это в далёком будущем — возможно, там будет какое‑нибудь ещё большее число!).
> Использовать более старый набор инструментов можно, но это словно включить **секретный режим повышенной сложности**: вы получите дополнительные
> ошибки компилятора, о которых в этой книге ни слова. Впрочем, может вы этого и хотите! 😄

Мне довольно часто задают вопрос, как реализовать связный список в Rust. Ответ, честно говоря, зависит от ваших требований, и, разумеется, не так‑то просто выдать его на ходу. Поэтому я и решил написать эту книгу — чтобы раз и навсегда дать исчерпывающий ответ на этот вопрос.

В этой серии материалов я научу вас основам и продвинутым приёмам программирования на Rust — и всё это на примере реализации шести связных списков. В процессе вы освоите:

* следующие типы указателей: `&`, `&mut`, `Box`, `Rc`, `Arc`, `*const`, `*mut`, `NonNull` (?);
* владение, заимствование, унаследованную изменяемость, внутреннюю изменяемость, трейт `Copy`;
* все ключевые слова: `struct`, `enum`, `fn`, `pub`, `impl`, `use` и другие;
* сопоставление с образцом, обобщения (дженерики), деструкторы;
* тестирование, установку новых наборов инструментов, работу с `miri`;
* небезопасный Rust: сырые указатели, алиасинг, стековые заимствования, `UnsafeCell`, дисперсию.

Да, связные списки настолько по-настоящему кошмарны, что при их реализации приходится столкнуться со всеми этими концепциями.

Всё есть на боковой панели (на мобильных устройствах она может быть свёрнута), но для быстрого ориентира вот что мы собираемся реализовать:

1. [Плохой односвязный стек](first.md)
2. [Нормальный односвязный стек](second.md)
3. [Персистентный односвязный стек](third.md)
4. [Плохой, но безопасный двусвязный дек](fourth.md)
5. [Небезопасная односвязная очередь](fifth.md)
6. [TODO: Нормальный небезопасный двусвязный дек](sixth.md)
7. [Бонус: Куча забавных списков](infinity.md)

Чтобы мы все держались в одном ритме, я буду приводить все команды, которые ввожу в терминал. Для разработки проекта я также буду использовать стандартный менеджер пакетов Rust — Cargo. Cargo не обязателен для написания программы на Rust, но он *намного* удобнее, чем работа напрямую с rustc. Если вы просто хотите поэкспериментировать, можно запускать простые программы прямо в браузере через [play.rust-lang.org][play].

В последующих разделах мы будем использовать «rustup» для установки дополнительных инструментов Rust.

Я настоятельно рекомендую [устанавливать все наборы инструментов Rust с помощью rustup](https://www.rust-lang.org/tools/install).

Давайте приступим и создадим наш проект:

```text
> cargo new --lib lists
> cd lists
```

Мы будем размещать каждый список в отдельном файле, чтобы не потерять проделанную работу.

Стоит отметить, что *настоящий* опыт изучения Rust — это писать код, слушать, как на вас «кричит» компилятор,
и пытаться разобраться, что вообще всё это значит. Я буду старательно следить за тем, чтобы такое происходило
как можно чаще. Умение читать и понимать в целом отличные сообщения об ошибках компилятора и документацию Rust
*невероятно* важно для продуктивной работы программиста на этом языке.

Хотя, честно говоря, это не совсем правда. В процессе написания я столкнулся с *гораздо* большим количеством ошибок
компилятора, чем показываю. В частности, в последующих главах я не буду демонстрировать множество случайных ошибок
из серии «я неправильно ввёл (или скопировал) код» — с такими вы сталкиваетесь в любом языке. Это *экскурсия с гидом*
по тому, как компилятор на нас «кричит».

Мы будем двигаться довольно медленно, и, честно говоря, почти всё время я не собираюсь быть чересчур серьёзным.
По-моему, программирование должно быть весёлым, чёрт возьми!

Если вы из тех, кто хочет максимально насыщенное информацией, строгое и формальное изложение, — эта книга не для вас.
Вообще ничто из того, что я когда‑либо сделаю, не для вас. Имейте это в виду.





# Обязательное публичное заявление

Просто чтобы мы были на 100 % уверены друг в друге: я ненавижу связные списки.
Страстно. Связные списки — ужасные структуры данных.

Но, разумеется, у связных списков есть несколько отличных сценариев применения:

* Вам нужно *очень много* раз разделять или объединять большие списки. *Очень много*.
* Вы делаете какую‑нибудь потрясающую вещь с параллелизмом без блокировок (lock‑free).
* Вы пишете ядро системы или встраиваемое ПО и хотите использовать интрузивный список.
* Вы используете чисто функциональный язык, где ограниченная семантика и отсутствие мутаций делают работу со связными списками проще.
* … и многое другое!

Но все эти случаи *крайне редки* для тех, кто пишет программы на Rust. В 99 % случаев вам стоит просто использовать `Vec` (стек на базе массива), а в 99 % из оставшегося 1 % — `VecDeque` (дек на базе массива). Это явно более удачные структуры данных для большинства задач благодаря менее частым выделениям памяти, меньшим накладным расходам, истинному произвольному доступу и лучшей локальности в кэше.

Связные списки — такая же *нишевая* и *неоднозначная* структура данных, как и префиксное дерево (trie). Мало кто станет возражать, если я скажу, что trie — нишевая структура, которую среднестатистический программист вполне может ни разу не изучить за всю свою продуктивную карьеру. И тем не менее у связных списков какой‑то причудливый «звёздный» статус.

Мы учим каждого студента-бакалавра писать связный список. Это единственная нишевая коллекция, [которую я не смог убрать из std::collections][rust-std-list]. Это [*тот самый* список в C++][cpp-std-list]!

Нам всем, как сообществу, стоит сказать *«нет»* связным спискам в роли «стандартной» структуры данных. Это неплохая структура с несколькими отличными сценариями применения — но эти сценарии *исключительны*, а не обыденны.

Судя по всему, некоторые люди читают только первый абзац этого объявления — и на этом останавливаются. Буквально: они пытаются опровергнуть мой аргумент, приводя в пример один из пунктов из моего же списка *«отличных сценариев применения»*. Тот самый, что идёт сразу после первого абзаца!

Чтобы у меня была прямая ссылка на развёрнутый аргумент, ниже — несколько встречных доводов, с которыми я сталкивался, и мои ответы на них. Если вы просто хотите поучиться Rust — смело переходите к [первой главе](first.md)!




## Performance doesn't always matter

Yes! Maybe your application is I/O-bound or the code in question is in some
cold case that just doesn't matter. But this isn't even an argument for using
a linked list. This is an argument for using *whatever at all*. Why settle for
a linked list? Use a linked hash map!

If performance doesn't matter, then it's *surely* fine to apply the natural
default of an array.





## They have O(1) split-append-insert-remove if you have a pointer there

Yep! Although as [Bjarne Stroustrup notes][bjarne] *this doesn't actually
matter* if the time it takes to get that pointer completely dwarfs the
time it would take to just copy over all the elements in an array (which is
really quite fast).

Unless you have a workload that is heavily dominated by splitting and merging
costs, the penalty *every other* operation takes due to caching effects and code
complexity will eliminate any theoretical gains.

*But yes, if you're profiling your application to spend a lot of time in
splitting and merging, you may have gains in a linked list*.





## I can't afford amortization

You've already entered a pretty niche space -- most can afford amortization.
Still, arrays are amortized *in the worst case*. Just because you're using an
array, doesn't mean you have amortized costs. If you can predict how many
elements you're going to store (or even have an upper-bound), you can
pre-reserve all the space you need. In my experience it's *very* common to be
able to predict how many elements you'll need. In Rust in particular, all
iterators provide a `size_hint` for exactly this case.

Then `push` and `pop` will be truly O(1) operations. And they're going to be
*considerably* faster than `push` and `pop` on linked list. You do a pointer
offset, write the bytes, and increment an integer. No need to go to any kind of
allocator.

How's that for low latency?

*But yes, if you can't predict your load, there are worst-case
latency savings to be had!*





## Linked lists waste less space

Well, this is complicated. A "standard" array resizing strategy is to grow
or shrink so that at most half the array is empty. This is indeed a lot of
wasted space. Especially in Rust, we don't automatically shrink collections
(it's a waste if you're just going to fill it back up again), so the wastage
can approach infinity!

But this is a worst-case scenario. In the best-case, an array stack only has
three pointers of overhead for the entire array. Basically no overhead.

Linked lists on the other hand unconditionally waste space per element.
A singly-linked list wastes one pointer while a doubly-linked list wastes
two. Unlike an array, the relative wasteage is proportional to the size of
the element. If you have *huge* elements this approaches 0 waste. If you have
tiny elements (say, bytes), then this can be as much as 16x memory overhead
(8x on 32-bit)!

Actually, it's more like 23x (11x on 32-bit) because padding will be added
to the byte to align the whole node's size to a pointer.

This is also assuming the best-case for your allocator: that allocating and
deallocating nodes is being done densely and you're not losing memory to
fragmentation.

*But yes, if you have huge elements, can't predict your load, and have a
decent allocator, there are memory savings to be had!*





## I use linked lists all the time in &lt;functional language&gt;

Great! Linked lists are super elegant to use in functional languages
because you can manipulate them without any mutation, can describe them
recursively, and also work with infinite lists due to the magic of laziness.

Specifically, linked lists are nice because they represent an iteration without
the need for any mutable state. The next step is just visiting the next sublist.

Rust mostly does this kind of thing with [iterators][]. They can be infinite 
and you can map, filter, reverse, and concatenate them just like a functional list,
and it will all be done just as lazily!

Rust also lets you easily talk about sub-arrays with *[slices][]*. Your usual
head/tail split in a functional language is [just `slice.split_at_mut(1)`][split].
For a long time, Rust had an experimental system for pattern matching on
slices which was super cool, but the feature was simplified when it was
stabilized. Still, [basic slice patterns][slice-pats] are neat! And of course,
slices can be turned into iterators!

*But yes, if you're limited to immutable semantics, linked lists can be very
nice*.

Note that I'm not saying that functional programming is necessarily weak or
bad. However it *is* fundamentally semantically limited: you're largely only
allowed to talk about how things *are*, and not how they should be *done*. This
is actually a *feature*, because it enables the compiler to do tons of [exotic
transformations][ghc] and potentially figure out the *best* way to do things
without you having to worry about it. However this comes at the cost of being
*able* to worry about it. There are usually escape hatches, but at some limit
you're just writing procedural code again.

Even in functional languages, you should endeavour to use the appropriate data
structure for the job when you actually need a data structure. Yes,
singly-linked lists are your primary tool for control flow, but they're a
really poor way to actually store a bunch of data and query it.


## Linked lists are great for building concurrent data structures!

Yes! Although writing a concurrent data structure is really a whole different
beast, and isn't something that should be taken lightly. Certainly not something
many people will even *consider* doing. Once one's been written, you're also not
really choosing to use a linked list. You're choosing to use an MPSC queue or
whatever. The implementation strategy is pretty far removed in this case!

*But yes, linked lists are the defacto heroes of the dark world of lock-free
concurrency.*




## Mumble mumble kernel embedded something something intrusive.

It's niche. You're talking about a situation where you're not even using
your language's *runtime*. Is that not a red flag that you're doing something
strange?

It's also wildly unsafe.

*But sure. Build your awesome zero-allocation lists on the stack.*





## Iterators don't get invalidated by unrelated insertions/removals

That's a delicate dance you're playing. Especially if you don't have
a garbage collector. I might argue that your control flow and ownership
patterns are probably a bit too tangled, depending on the details.

*But yes, you can do some really cool crazy stuff with cursors.*





## They're simple and great for teaching!

Well, yeah. You're reading a book dedicated to that premise.
Well, singly-linked lists are pretty simple. Doubly-linked lists
can get kinda gnarly, as we'll see.




# Take a Breath

Ok. That's out of the way. Let's write a bajillion linked lists.

[On to the first chapter!](first.md)


[rust-std-list]: https://doc.rust-lang.org/std/collections/struct.LinkedList.html
[cpp-std-list]: http://en.cppreference.com/w/cpp/container/list
[github]: https://github.com/rust-unofficial/too-many-lists
[bjarne]: https://www.youtube.com/watch?v=YQs6IC-vgmo
[slices]: https://doc.rust-lang.org/std/primitive.slice.html
[split]: https://doc.rust-lang.org/std/primitive.slice.html#method.split_at_mut
[slice-pats]: https://doc.rust-lang.org/edition-guide/rust-2018/slice-patterns.html
[iterators]: https://doc.rust-lang.org/std/iter/trait.Iterator.html
[ghc]: https://wiki.haskell.org/GHC_optimisations#Fusion
[play]: https://play.rust-lang.org/
