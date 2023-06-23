# Строитель

## Описание

Создание объекта с помощью вызовов помощника-строителя.

## Пример

```rust
#[derive(Debug, PartialEq)]
pub struct Foo {
    // Lots of complicated fields.
    bar: String,
}

impl Foo {
    // This method will help users to discover the builder
    pub fn builder() -> FooBuilder {
        FooBuilder::default()
    }
}

#[derive(Default)]
pub struct FooBuilder {
    // Probably lots of optional fields.
    bar: String,
}

impl FooBuilder {
    pub fn new(/* ... */) -> FooBuilder {
        // Set the minimally required fields of Foo.
        FooBuilder {
            bar: String::from("X"),
        }
    }

    pub fn name(mut self, bar: String) -> FooBuilder {
        // Set the name on the builder itself, and return the builder by value.
        self.bar = bar;
        self
    }

    // If we can get away with not consuming the Builder here, that is an
    // advantage. It means we can use the FooBuilder as a template for constructing
    // many Foos.
    pub fn build(self) -> Foo {
        // Create a Foo from the FooBuilder, applying all settings in FooBuilder
        // to Foo.
        Foo { bar: self.bar }
    }
}

#[test]
fn builder_test() {
    let foo = Foo {
        bar: String::from("Y"),
    };
    let foo_from_builder: Foo = FooBuilder::new().name(String::from("Y")).build();
    assert_eq!(foo, foo_from_builder);
}
```

## Мотивация

Полезно, когда вам в противном случае потребуется много конструкторов или когда
создание имеет побочные эффекты.

## Преимущества

Разделяет методы для создания от других методов.

Предотвращает разрастание конструкторов.

Может использоваться для инициализации в одну строку, а также для более сложного создания.

## Недостатки

Более сложный, чем создание объекта структуры напрямую или простой конструктор
функции.

## Обсуждение

Этот шаблон чаще встречается в Rust (и для более простых объектов), чем в
многих других языках, потому что Rust не имеет перегрузки. Поскольку вы можете иметь только один метод с данным именем, наличие нескольких конструкторов менее удобно в Rust, чем в C++, Java или других языках.

Этот шаблон часто используется там, где объект-строитель полезен сам по себе,
а не просто строитель. Например, см.
[`std::process::Command`](https://doc.rust-lang.org/std/process/struct.Command.html)
является строителем для [`Child`](https://doc.rust-lang.org/std/process/struct.Child.html)
(процесса). В этих случаях шаблон именования `T` и `TBuilder` не используется.

В примере строитель принимает и возвращает значение по значению. Часто более эргономично (и эффективно) принимать и возвращать строитель как изменяемую ссылку. Проверка заимствования делает это естественным образом. Этот подход имеет преимущество в том, что можно писать код, как

```rust,ignore
let mut fb = FooBuilder::new();
fb.a();
fb.b();
let f = fb.build();
```

а также в стиле `FooBuilder::new().a().b().build()`.

## Смотрите также

- [Описание в руководстве по стилю](https://web.archive.org/web/20210104103100/https://doc.rust-lang.org/1.12.0/style/ownership/builders.html)
- [derive_builder](https://crates.io/crates/derive_builder), крейт для автоматической
  реализации этого шаблона с избежанием шаблона.
- [Шаблон конструктора](../../idioms/ctor.md) для случаев, когда создание проще.
- [Шаблон строителя (wikipedia)](https://en.wikipedia.org/wiki/Builder_pattern)
- [Создание сложных значений](https://web.archive.org/web/20210104103000/https://rust-lang.github.io/api-guidelines/type-safety.html#c-builder)