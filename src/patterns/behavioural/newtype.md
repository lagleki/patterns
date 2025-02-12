# Newtype

Что, если в некоторых случаях мы хотим, чтобы тип вел себя аналогично другому типу или
принуждал к некоторому поведению на этапе компиляции, когда использование только псевдонимов типа не будет достаточно?

Например, если мы хотим создать пользовательскую реализацию `Display` для `String`
из соображений безопасности (например, пароли).

Для таких случаев мы можем использовать паттерн `Newtype` для обеспечения **безопасности типов**
и **инкапсуляции**.

## Описание

Используйте кортежную структуру с одним полем, чтобы создать непрозрачную обертку для типа.
Это создает новый тип, а не псевдоним для типа (`type` элементы).

## Пример

```rust
use std::fmt::Display;

// Create Newtype Password to override the Display trait for String
struct Password(String);

impl Display for Password {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "****************")
    }
}

fn main() {
    let unsecured_password: String = "ThisIsMyPassword".to_string();
    let secured_password: Password = Password(unsecured_password.clone());
    println!("unsecured_password: {unsecured_password}");
    println!("secured_password: {secured_password}");
}
```

```shell
unsecured_password: ThisIsMyPassword
secured_password: ****************
```

## Мотивация

Основная мотивация для новых типов - это абстракция. Это позволяет вам использовать
детали реализации между типами, тщательно контролируя интерфейс.
Используя новый тип вместо раскрытия типа реализации как части
API, это позволяет вам изменять реализацию с обратной совместимостью.

Newtypes могут использоваться для различения единиц, например, обертывания `f64` для получения
различимых `Miles` и `Kilometres`.

## Преимущества

Обернутый и оберточный типы несовместимы по типу (в отличие от использования
`type`), поэтому пользователи нового типа никогда не будут "путать" обернутый и оберточный
типы.

Newtypes - это абстракция нулевой стоимости - нет накладных расходов на выполнение.

Система конфиденциальности гарантирует, что пользователи не могут получить доступ к обернутому типу (если
поле является частным, что по умолчанию).

## Недостатки

Недостатком новых типов (особенно по сравнению с псевдонимами типов) является то, что там
нет специальной поддержки языка. Это означает, что может быть _очень_ много паттернного кода.
Вам нужен метод "прохода" для каждого метода, который вы хотите выставить наружу на
обернутый тип, и impl для каждого трейта, который вы хотите также реализовать для
типа обертки.

## Обсуждение

Newtypes очень распространены в коде Rust. Абстракция или представление единиц - это
наиболее распространенные использования, но они могут использоваться по другим причинам:

- ограничение функциональности (сокращение выставленных функций или реализованных трейтов),
- сделать тип с копирующей семантикой иметь семантику перемещения,
- абстракция, предоставляя более конкретный тип и тем самым скрывая внутренние типы,
  например,

```rust,ignore
pub struct Foo(Bar<T1, T2>);
```

Здесь `Bar` может быть каким-то общедоступным, общим типом, а `T1` и `T2` - некоторыми внутренними
типы. Пользователи нашего модуля не должны знать, что мы реализуем `Foo`, используя `Bar`,
но то, что мы действительно скрываем здесь, это типы `T1` и `T2`, и как они используются
с `Bar`.

## Смотрите также

- [Расширенные типы в книге](https://doc.rust-lang.org/book/ch19-04-advanced-types.html?highlight=newtype#using-the-newtype-pattern-for-type-safety-and-abstraction)
- [Newtypes в Haskell](https://wiki.haskell.org/Newtype)
- [Псевдонимы типов](https://doc.rust-lang.org/stable/book/ch19-04-advanced-types.html#creating-type-synonyms-with-type-aliases)
- [derive_more](https://crates.io/crates/derive_more), крейт для производных многих
  встроенных трейтов на новых типах.
- [The Newtype Pattern In Rust](https://web.archive.org/web/20230519162111/https://www.worthe-it.co.za/blog/2020-10-31-newtype-pattern-in-rust.html)
