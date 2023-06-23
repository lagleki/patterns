# Перебор `Option`

## Описание

`Option` можно рассматривать как контейнер, который содержит либо ноль, либо один элемент. В частности, он реализует трейт `IntoIterator` и, как таковой, может использоваться с обобщенным кодом, который требует такого типа.

## Примеры

Поскольку `Option` реализует `IntoIterator`, его можно использовать в качестве аргумента для [`.extend()`](https://doc.rust-lang.org/std/iter/trait.Extend.html#tymethod.extend):

```rust
let turing = Some("Turing");
let mut logicians = vec!["Curry", "Kleene", "Markov"];

logicians.extend(turing);

// equivalent to
if let Some(turing_inner) = turing {
    logicians.push(turing_inner);
}
```

Если вам нужно добавить `Option` в конец существующего итератора, вы можете передать его в [`.chain()`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.chain):

```rust
let turing = Some("Turing");
let logicians = vec!["Curry", "Kleene", "Markov"];

for logician in logicians.iter().chain(turing.iter()) {
    println!("{} is a logician", logician);
}
```

Обратите внимание, что если `Option` всегда `Some`, то более идиоматично использовать [`std::iter::once`](https://doc.rust-lang.org/std/iter/fn.once.html) на элементе.

Также, поскольку `Option` реализует `IntoIterator`, его можно перебирать с помощью цикла `for`. Это эквивалентно сопоставлению с `if let Some(..)`, и в большинстве случаев вы должны предпочитать последнее.

## Смотрите также

- [`std::iter::once`](https://doc.rust-lang.org/std/iter/fn.once.html) - это итератор, который выдает ровно один элемент. Это более читаемая альтернатива `Some(foo).into_iter()`.

- [`Iterator::filter_map`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.filter_map) - это версия [`Iterator::map`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.map), специализированная для отображения функций, которые возвращают `Option`.

- Крейт [`ref_slice`](https://crates.io/crates/ref_slice) предоставляет функции для преобразования `Option` в нулевой или одноэлементный срез.

- [Документация для `Option<T>`](https://doc.rust-lang.org/std/option/enum.Option.html)