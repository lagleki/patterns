# Линзы и призмы

Это чисто функциональный концепт, который не часто используется в Rust.
Тем не менее, изучение концепции может быть полезно для понимания других
паттернов в Rust API, таких как [посетители](../patterns/behavioural/visitor.md).
Они также имеют узкую область применения.

## Линзы: единый доступ к типам

Линза - это концепция из языков функционального программирования, которая позволяет
обращаться к частям типа данных в абстрактном, унифицированном виде.[^1]
В основном концептуальном плане это похоже на то, как работают трейты Rust с
стиранием типов, но оно имеет немного больше мощности и гибкости.

Например, предположим, что в банке содержится несколько форматов JSON для клиентов
данных.
Это происходит потому, что они поступают из разных баз данных или устаревших систем.
Одна база данных содержит данные, необходимые для выполнения проверки кредита:

```json
{ "name": "Jane Doe",
  "dob": "2002-02-24",
  [...]
  "customer_id": 1048576332,
}
```

Другая содержит информацию об учетной записи:

```json
{ "customer_id": 1048576332,
  "accounts": [
      { "account_id": 2121,
        "account_type: "savings",
        "joint_customer_ids": [],
        [...]
      },
      { "account_id": 2122,
        "account_type: "checking",
        "joint_customer_ids": [1048576333],
        [...]
      },
  ]
}
```

Обратите внимание, что оба типа имеют номер идентификации клиента, который соответствует человеку.
Как одна функция может обрабатывать оба записи разных типов?

В Rust `struct` может представлять каждый из этих типов, и трейт будет иметь
функцию `get_customer_id`, которую они реализуют:

```rust
use std::collections::HashSet;

pub struct Account {
    account_id: u32,
    account_type: String,
    // other fields omitted
}

pub trait CustomerId {
    fn get_customer_id(&self) -> u64;
}

pub struct CreditRecord {
    customer_id: u64,
    name: String,
    dob: String,
    // other fields omitted
}

impl CustomerId for CreditRecord {
    fn get_customer_id(&self) -> u64 {
        self.customer_id
    }
}

pub struct AccountRecord {
    customer_id: u64,
    accounts: Vec<Account>,
}

impl CustomerId for AccountRecord {
    fn get_customer_id(&self) -> u64 {
        self.customer_id
    }
}

// static polymorphism: only one type, but each function call can choose it
fn unique_ids_set<R: CustomerId>(records: &[R]) -> HashSet<u64> {
    records.iter().map(|r| r.get_customer_id()).collect()
}

// dynamic dispatch: iterates over any type with a customer ID, collecting all
// values together
fn unique_ids_iter<I>(iterator: I) -> HashSet<u64>
    where I: Iterator<Item=Box<dyn CustomerId>>
{
    iterator.map(|r| r.as_ref().get_customer_id()).collect()
}
```

Однако линзы позволяют переместить код, поддерживающий идентификатор клиента, из
типа в функцию доступа.
Вместо реализации трейта на каждом типе все соответствующие структуры могут
быть просто доступны одним и тем же способом.

Хотя сам язык Rust этого не поддерживает (стирание типов является
предпочтительным решением этой проблемы), [крейт lens-rs](https://github.com/TOETOE55/lens-rs/blob/master/guide.md) позволяет написать код,
который выглядит как это с помощью макросов:

```rust,ignore
use std::collections::HashSet;

use lens_rs::{optics, Lens, LensRef, Optics};

#[derive(Clone, Debug, Lens /* derive to allow lenses to work */)]
pub struct CreditRecord {
    #[optic(ref)] // macro attribute to allow viewing this field
    customer_id: u64,
    name: String,
    dob: String,
    // other fields omitted
}

#[derive(Clone, Debug)]
pub struct Account {
    account_id: u32,
    account_type: String,
    // other fields omitted
}

#[derive(Clone, Debug, Lens)]
pub struct AccountRecord {
    #[optic(ref)]
    customer_id: u64,
    accounts: Vec<Account>,
}

fn unique_ids_lens<T>(iter: impl Iterator<Item = T>) -> HashSet<u64>
where
    T: LensRef<Optics![customer_id], u64>, // any type with this field
{
    iter.map(|r| *r.view_ref(optics!(customer_id))).collect()
}
```

Показанная здесь версия `unique_ids_lens` позволяет использовать любой тип в итераторе,
пока у него есть атрибут с именем `customer_id`, к которому можно получить доступ через
функцию.
Так работают большинство языков функционального программирования на линзах.

Вместо макросов они достигают этого с помощью техники, известной как "каррирование".
То есть они "частично конструируют" функцию, оставляя тип
последнего параметра (значение, над которым выполняется операция) незаполненным до тех пор, пока функция не
будет вызвана.
Таким образом, его можно вызывать с разными типами динамически даже из одного места в
коде.
Это то, что имитируют `optics!` и `view_ref` в приведенном выше примере.

Функциональный подход не обязательно ограничивается доступом к членам.
Можно создавать более мощные линзы, которые как _устанавливают_, так и _получают_ данные в
структуре.
Но концепция действительно становится интересной, когда используется в качестве строительного блока для
композиции.
Именно там концепция становится более ясной в Rust.

## Призмы: высший порядок формы "оптики"

Простая функция, такая как `unique_ids_lens` выше, работает с одной линзой.
_Призма_ - это функция, которая работает с _семейством_ линз.
Она находится на одном концептуальном уровне выше, используя линзы в качестве строительного блока, и
продолжая метафору, является частью семейства "оптики".
Это главное, что полезно для понимания Rust API, поэтому здесь будет фокус.

Точно так же, как трейты позволяют проектировать "подобные линзам" с помощью статического полиморфизма и
динамическая отправка, призмоподобные конструкции появляются в Rust API, которые разбивают проблемы
на несколько связанных типов для композиции.
Хорошим примером этого являются трейты в крейте парсинга _Serde_.

Попытка понять, как работает _Serde_, только читая API, является
сложной задачей, особенно в первый раз.
Рассмотрим трейт `Deserializer`, реализованный некоторым типом в любой библиотеке,
которая разбирает новый формат:

```rust,ignore
pub trait Deserializer<'de>: Sized {
    type Error: Error;

    fn deserialize_any<V>(self, visitor: V) -> Result<V::Value, Self::Error>
    where
        V: Visitor<'de>;

    fn deserialize_bool<V>(self, visitor: V) -> Result<V::Value, Self::Error>
    where
        V: Visitor<'de>;

    // remainder ommitted
}
```

Для трейта, который должен только разбирать данные из формата и возвращать
значение, это выглядит странно.

Почему все типы возвращаемых значений стираются?

Чтобы понять
