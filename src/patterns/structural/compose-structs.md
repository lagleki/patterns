# Декомпозиция структур для независимого заимствования

## Описание

Иногда большая структура может вызвать проблемы с проверкой заимствования - хотя поля могут быть заимствованы независимо, иногда целая структура используется сразу, что препятствует другим использованиям. Решением может быть декомпозиция структуры на несколько меньших структур. Затем объедините их в исходную структуру. Тогда каждую структуру можно заимствовать отдельно и она будет иметь более гибкое поведение.

Это часто приводит к лучшему дизайну в других аспектах: применение этого шаблона проектирования часто позволяет выявлять более мелкие единицы функциональности.

## Пример

Вот выдуманный пример, где проверка заимствования мешает нам в нашем плане использования структуры:

```rust
struct Database {
    connection_string: String,
    timeout: u32,
    pool_size: u32,
}

fn print_database(database: &Database) {
    println!("Connection string: {}", database.connection_string);
    println!("Timeout: {}", database.timeout);
    println!("Pool size: {}", database.pool_size);
}

fn main() {
    let mut db = Database {
        connection_string: "initial string".to_string(),
        timeout: 30,
        pool_size: 100,
    };

    let connection_string = &mut db.connection_string;
    print_database(&db);  // Immutable borrow of `db` happens here
    // *connection_string = "new string".to_string();  // Mutable borrow is used
                                                       // here
}
```

Мы можем применить этот шаблон проектирования и перестроить `Database` на три меньшие структуры, тем самым решив проблему проверки заимствования:

```rust
// Database is now composed of three structs - ConnectionString, Timeout and PoolSize.
// Let's decompose it into smaller structs
#[derive(Debug, Clone)]
struct ConnectionString(String);

#[derive(Debug, Clone, Copy)]
struct Timeout(u32);

#[derive(Debug, Clone, Copy)]
struct PoolSize(u32);

// We then compose these smaller structs back into `Database`
struct Database {
    connection_string: ConnectionString,
    timeout: Timeout,
    pool_size: PoolSize,
}

// print_database can then take ConnectionString, Timeout and Poolsize struct instead
fn print_database(connection_str: ConnectionString, 
                  timeout: Timeout, 
                  pool_size: PoolSize) {
    println!("Connection string: {:?}", connection_str);
    println!("Timeout: {:?}", timeout);
    println!("Pool size: {:?}", pool_size);
}

fn main() {
    // Initialize the Database with the three structs
    let mut db = Database {
        connection_string: ConnectionString("localhost".to_string()),
        timeout: Timeout(30),
        pool_size: PoolSize(100),
    };

    let connection_string = &mut db.connection_string;
    print_database(connection_string.clone(), db.timeout, db.pool_size);
    *connection_string = ConnectionString("new string".to_string());
}
```

## Мотивация

Этот шаблон наиболее полезен, когда у вас есть структура, которая закончилась с большим количеством полей, которые вы хотите заимствовать независимо. Таким образом, в конечном итоге получается более гибкое поведение.

## Преимущества

Декомпозиция структур позволяет обойти ограничения проверки заимствования. И это часто приводит к лучшему дизайну.

## Недостатки

Это может привести к более многословному коду. И иногда меньшие структуры не являются хорошими абстракциями, и мы получаем худший дизайн. Это, вероятно, "запах кода", указывающий на то, что программу нужно перестроить каким-то образом.

## Обсуждение

Этот шаблон не требуется в языках, которые не имеют проверки заимствования, поэтому в этом смысле он уникален для Rust. Однако создание более мелких единиц функциональности часто приводит к более чистому коду: широко признанный принцип программной инженерии, независимо от языка.

Этот шаблон основан на проверке заимствования Rust, чтобы иметь возможность заимствовать поля независимо друг от друга. В примере проверка заимствования знает, что `a.b` и `a.c` различны и могут быть заимствованы независимо, она не пытается заимствовать все `a`, что сделало бы этот шаблон бесполезным.
