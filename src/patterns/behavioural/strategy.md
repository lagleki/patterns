# Стратегия (также известна как Политика)

## Описание

[Шаблон проектирования Стратегия](https://ru.wikipedia.org/wiki/%D0%A8%D0%B0%D0%B1%D0%BB%D0%BE%D0%BD_%D0%BF%D1%80%D0%BE%D0%B5%D0%BA%D1%82%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D1%8F#.D0.A1.D1.82.D1.80.D0.B0.D1.82.D0.B5.D0.B3.D0.B8.D1.8F)

- это техника, которая позволяет разделить заботы.

Она также позволяет отделить модули программного обеспечения через [Принцип инверсии зависимостей](https://ru.wikipedia.org/wiki/%D0%9F%D1%80%D0%B8%D0%BD%D1%86%D0%B8%D0%BF_%D0%B8%D0%BD%D0%B2%D0%B5%D1%80%D1%81%D0%B8%D0%B8_%D0%B7%D0%B0%D0%B2%D0%B8%D1%81%D0%B8%D0%BC%D0%BE%D1%81%D1%82%D0%B5%D0%B9).

Основная идея, лежащая в основе шаблона Стратегия, заключается в том, что, имея алгоритм решения
определенной проблемы, мы определяем только скелет алгоритма на абстрактном
уровне, и мы разделяем конкретную реализацию алгоритма на разные части.

Таким образом, клиент, использующий алгоритм, может выбрать конкретную реализацию,
в то время как общий рабочий процесс алгоритма остается неизменным. Другими словами, абстрактная
спецификация класса не зависит от конкретной реализации производного класса, но конкретная реализация
должна соответствовать абстрактной спецификации. Вот почему мы называем это "Инверсия зависимостей".

## Мотивация

Представьте, что мы работаем над проектом, который генерирует отчеты каждый месяц.
Нам нужно, чтобы отчеты генерировались в разных форматах (стратегиях), например,
в форматах `JSON` или `Plain Text`.
Но вещи меняются со временем, и мы не знаем, какие требования мы можем получить
в будущем. Например, нам может потребоваться сгенерировать наш отчет в совершенно новом
формате или просто изменить один из существующих форматов.

## Пример

В этом примере наши инварианты (или абстракции) - это `Formatter` и `Report`, а `Text` и `Json` - наши стратегические структуры. Эти стратегии
должны реализовать трейт `Formatter`.

```rust
use std::collections::HashMap;

type Data = HashMap<String, u32>;

trait Formatter {
    fn format(&self, data: &Data, buf: &mut String);
}

struct Report;

impl Report {
    // Write should be used but we kept it as String to ignore error handling
    fn generate<T: Formatter>(g: T, s: &mut String) {
        // backend operations...
        let mut data = HashMap::new();
        data.insert("one".to_string(), 1);
        data.insert("two".to_string(), 2);
        // generate report
        g.format(&data, s);
    }
}

struct Text;
impl Formatter for Text {
    fn format(&self, data: &Data, buf: &mut String) {
        for (k, v) in data {
            let entry = format!("{} {}\n", k, v);
            buf.push_str(&entry);
        }
    }
}

struct Json;
impl Formatter for Json {
    fn format(&self, data: &Data, buf: &mut String) {
        buf.push('[');
        for (k, v) in data.into_iter() {
            let entry = format!(r#"{{"{}":"{}"}}"#, k, v);
            buf.push_str(&entry);
            buf.push(',');
        }
        if !data.is_empty() {
            buf.pop(); // remove extra , at the end
        }
        buf.push(']');
    }
}

fn main() {
    let mut s = String::from("");
    Report::generate(Text, &mut s);
    assert!(s.contains("one 1"));
    assert!(s.contains("two 2"));

    s.clear(); // reuse the same buffer
    Report::generate(Json, &mut s);
    assert!(s.contains(r#"{"one":"1"}"#));
    assert!(s.contains(r#"{"two":"2"}"#));
}
```

## Преимущества

Основное преимущество - это разделение забот. Например, в этом случае `Report`
ничего не знает о конкретных реализациях `Json` и `Text`,
в то время как реализации вывода не заботятся о том, как данные предварительно обрабатываются,
хранятся и извлекаются. Единственное, что им нужно знать, это конкретный
трейт для реализации и его метод, определяющий конкретную реализацию алгоритма обработки
результата, т.е. `Formatter` и `format(...)`.

## Недостатки

Для каждой стратегии должен быть реализован хотя бы один модуль, поэтому количество модулей
увеличивается с количеством стратегий. Если есть много стратегий для выбора,
то пользователям нужно знать, как стратегии отличаются друг от друга.

## Обсуждение

В предыдущем примере все стратегии реализованы в одном файле.
Способы предоставления разных стратегий включают:

- Все в одном файле (как показано в этом примере, аналогично разделению на модули)
- Разделены как модули, например, модуль `formatter::json`, модуль `formatter::text`
- Использовать флаги функций компилятора, например, функция `json`, функция `text`
- Разделены как ящики, например, ящик `json`, ящик `text`

Крейт Serde - хороший пример шаблона `Strategy` в действии. Serde позволяет
[полная настройка](https://serde.rs/custom-serialization.html) поведения сериализации путем ручной реализации трейтов `Serialize` и `Deserialize` для нашего
тип. Например, мы могли бы легко заменить `serde_json` на `serde_cbor`, так как они
представляют схожие методы. Иметь это делает помощник крейт `serde_transcode` намного
более полезным и эргономичным.

Однако нам не нужно использовать трейты, чтобы разработать этот шаблон на Rust.

Следующий игрушечный пример демонстрирует идею шаблона Стратегия с использованием Rust
`замыкания`:

```rust
struct Adder;
impl Adder {
    pub fn add<F>(x: u8, y: u8, f: F) -> u8
    where
        F: Fn(u8, u8) -> u8,
    {
        f(x, y)
    }
}

fn main() {
    let arith_adder = |x, y| x + y;
    let bool_adder = |x, y| {
        if x == 1 || y == 1 {
            1
        } else {
            0
        }
    };
    let custom_adder = |x, y| 2 * x + y;

    assert_eq!(9, Adder::add(4, 5, arith_adder));
    assert_eq!(0, Adder::add(0, 0, bool_adder));
    assert_eq!(5, Adder::add(1, 3, custom_adder));
}
```

Фактически, Rust уже использует эту идею для метода `map` `Options`:

```rust
fn main() {
    let val = Some("Rust");

    let len_strategy = |s: &str| s.len();
    assert_eq!(4, val.map(len_strategy).unwrap());

    let first_byte_strategy = |s: &str| s.bytes().next().unwrap();
    assert_eq!(82, val.map(first_byte_strategy).unwrap());
}
```

## Смотрите также

- [Шаблон проектирования Стратегия](https://ru.wikipedia.org/wiki/%D0%A8%D0%B0%D0%B1%D0%BB%D0%BE%D0%BD_%D0%BF%D1%80%D0%BE%D0%B5%D0%BA%D1%82%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D1%8F#.D0.A1.D1.82.D1.80.D0.B0.D1.82.D0.B5.D0.B3.D0.B8.D1.8F)
- [Внедрение зависимостей](https://ru.wikipedia.org/wiki/%D0%9F%D1%80%D0%B8%D0%BD%D1%86%D0%B8%D0%BF_%D0%B8%D0%BD%D0%B2%D0%B5%D1%80%D1%81%D0%B8%D0%B8_%D0%B7%D0%B0%D0%B2%D0%B8%D1%81%D0%B8%D0%BC%D0%BE%D1%81%D1%82%D0%B5%D0%B9)
- [Проектирование на основе политик](https://ru.wikipedia.org/wiki/Modern_C++_Design#Policy-based_design)
