# Трейт `Default`

## Описание

Многие типы в Rust имеют [конструктор]. Однако это _специфично_ для типа; Rust не может абстрагироваться от "всего, что имеет метод `new()`". Для этого был создан трейт [`Default`], который может использоваться с контейнерами и другими обобщенными типами (например, см. [`Option::unwrap_or_default()`]). Следует отметить, что некоторые контейнеры уже реализуют его там, где это применимо.

Не только одноэлементные контейнеры, такие как `Cow`, `Box` или `Arc`, реализуют `Default` для содержащихся типов `Default`, но также можно автоматически `#[derive(Default)]` для структур, поля которых все реализуют его, так что чем больше типов реализуют `Default`, тем более полезным он становится.

С другой стороны, конструкторы могут принимать несколько аргументов, в то время как метод `default()` - нет. Даже может быть несколько конструкторов с разными именами, но может быть только одна реализация `Default` на тип.

## Пример

```rust
use std::{path::PathBuf, time::Duration};

// note that we can simply auto-derive Default here.
#[derive(Default, Debug, PartialEq)]
struct MyConfiguration {
    // Option defaults to None
    output: Option<PathBuf>,
    // Vecs default to empty vector
    search_path: Vec<PathBuf>,
    // Duration defaults to zero time
    timeout: Duration,
    // bool defaults to false
    check: bool,
}

impl MyConfiguration {
    // add setters here
}

fn main() {
    // construct a new instance with default values
    let mut conf = MyConfiguration::default();
    // do something with conf here
    conf.check = true;
    println!("conf = {:#?}", conf);
        
    // partial initialization with default values, creates the same instance
    let conf1 = MyConfiguration {
        check: true,
        ..Default::default()
    };
    assert_eq!(conf, conf1);
}
```

## Смотрите также

- Идиома [конструктора] - это еще один способ создания экземпляров, которые могут быть "по умолчанию" или нет.
- Документация по [`Default`] (прокрутите вниз для списка реализаций)
- [`Option::unwrap_or_default()`]
- [`derive(new)`]

[конструктор]: ctor.md
[`Default`]: https://doc.rust-lang.org/stable/std/default/trait.Default.html
[`Option::unwrap_or_default()`]: https://doc.rust-lang.org/stable/std/option/enum.Option.html#method.unwrap_or_default
[`derive(new)`]: https://crates.io/crates/derive-new/