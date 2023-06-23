## Конструкторы

### Описание

Rust не имеет конструкторов как языковую конструкцию. Вместо этого
соглашением является использование [ассоциированной функции][associated function] `new` для создания объекта:

````rust
/// Time in seconds.
///
/// # Example
///
/// ```
/// let s = Second::new(42);
/// assert_eq!(42, s.value());
/// ```
pub struct Second {
    value: u64
}

impl Second {
    // Constructs a new instance of [`Second`].
    // Note this is an associated function - no self.
    pub fn new(value: u64) -> Self {
        Self { value }
    }

    /// Returns the value in seconds.
    pub fn value(&self) -> u64 {
        self.value
    }
}
````

### Конструкторы по умолчанию

Rust поддерживает конструкторы по умолчанию с помощью трейта [`Default`][std-default]:

````rust
/// Time in seconds.
///
/// # Example
///
/// ```
/// let s = Second::default();
/// assert_eq!(0, s.value());
/// ```
pub struct Second {
    value: u64
}

impl Second {
    /// Returns the value in seconds.
    pub fn value(&self) -> u64 {
        self.value
    }
}

impl Default for Second {
    fn default() -> Self {
        Self { value: 0 }
    }
}
````

`Default` также может быть получен автоматически, если все типы всех полей реализуют `Default`,
как это делается с `Second`:

````rust
/// Time in seconds.
///
/// # Example
///
/// ```
/// let s = Second::default();
/// assert_eq!(0, s.value());
/// ```
#[derive(Default)]
pub struct Second {
    value: u64
}

impl Second {
    /// Returns the value in seconds.
    pub fn value(&self) -> u64 {
        self.value
    }
}
````

**Примечание:** Распространенно и ожидаемо, что типы реализуют как `Default`, так и пустой конструктор `new`. `new` является соглашением о конструкторе в Rust, и пользователи ожидают его наличия, поэтому, если базовый конструктор может не принимать аргументы, то он должен существовать, даже если он функционально идентичен `default`.

**Подсказка:** Преимущество реализации или получения `Default` заключается в том, что ваш тип
теперь может использоваться там, где требуется реализация `Default`, наиболее заметно,
любые из [`*or_default` функций в стандартной библиотеке][std-or-default].

### Смотрите также

- [Идиома по умолчанию](default.md) для более подробного описания
  трейта `Default`.

- [Паттерн строителя](../patterns/creational/builder.md) для создания
  объектов, где есть несколько конфигураций.

- [API Guidelines/C-COMMON-TRAITS][API Guidelines/C-COMMON-TRAITS] для
  реализации как `Default`, так и `new`.

[associated function]: https://doc.rust-lang.org/stable/book/ch05-03-method-syntax.html#associated-functions
[std-default]: https://doc.rust-lang.org/stable/std/default/trait.Default.html
[std-or-default]: https://doc.rust-lang.org/stable/std/?search=or_default
[API Guidelines/C-COMMON-TRAITS]: https://rust-lang.github.io/api-guidelines/interoperability.html#types-eagerly-implement-common-traits-c-common-traits
