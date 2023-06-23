# Команда

## Описание

Основная идея паттерна Команда заключается в том, чтобы выделить действия в отдельные объекты и передавать их в качестве параметров.

## Мотивация

Предположим, у нас есть последовательность действий или транзакций, инкапсулированных в объекты. Мы хотим, чтобы эти действия или команды были выполнены или вызваны в определенном порядке позже в разное время. Эти команды также могут быть вызваны в результате какого-то события. Например, когда пользователь нажимает кнопку или при получении пакета данных. Кроме того, эти команды могут быть отменены. Это может пригодиться для операций редактора. Мы можем хотеть сохранить журналы выполненных команд, чтобы мы могли повторно применить изменения позже, если система выйдет из строя.

## Пример

Определим две операции базы данных `create table` и `add field`. Каждая из этих операций является командой, которая знает, как отменить команду, например, `drop table` и `remove field`. Когда пользователь вызывает операцию миграции базы данных, каждая команда выполняется в определенном порядке, и когда пользователь вызывает операцию отката, весь набор команд вызывается в обратном порядке.

## Подход: Использование объектов трейтов

Мы определяем общий трейт, который инкапсулирует нашу команду с двумя операциями `execute` и `rollback`. Все структуры команд должны реализовывать этот трейт.

```rust
pub trait Migration {
    fn execute(&self) -> &str;
    fn rollback(&self) -> &str;
}

pub struct CreateTable;
impl Migration for CreateTable {
    fn execute(&self) -> &str {
        "create table"
    }
    fn rollback(&self) -> &str {
        "drop table"
    }
}

pub struct AddField;
impl Migration for AddField {
    fn execute(&self) -> &str {
        "add field"
    }
    fn rollback(&self) -> &str {
        "remove field"
    }
}

struct Schema {
    commands: Vec<Box<dyn Migration>>,
}

impl Schema {
    fn new() -> Self {
        Self { commands: vec![] }
    }

    fn add_migration(&mut self, cmd: Box<dyn Migration>) {
        self.commands.push(cmd);
    }

    fn execute(&self) -> Vec<&str> {
        self.commands.iter().map(|cmd| cmd.execute()).collect()
    }
    fn rollback(&self) -> Vec<&str> {
        self.commands
            .iter()
            .rev() // reverse iterator's direction
            .map(|cmd| cmd.rollback())
            .collect()
    }
}

fn main() {
    let mut schema = Schema::new();

    let cmd = Box::new(CreateTable);
    schema.add_migration(cmd);
    let cmd = Box::new(AddField);
    schema.add_migration(cmd);

    assert_eq!(vec!["create table", "add field"], schema.execute());
    assert_eq!(vec!["remove field", "drop table"], schema.rollback());
}
```

## Подход: Использование указателей на функции

Мы могли бы следовать другому подходу, создавая каждую отдельную команду как отдельную функцию и храня указатели на функции для вызова этих функций позже в другое время. Поскольку указатели на функции реализуют все три трейта `Fn`, `FnMut` и `FnOnce`, мы также можем передавать и хранить замыкания вместо указателей на функции.

```rust
type FnPtr = fn() -> String;
struct Command {
    execute: FnPtr,
    rollback: FnPtr,
}

struct Schema {
    commands: Vec<Command>,
}

impl Schema {
    fn new() -> Self {
        Self { commands: vec![] }
    }
    fn add_migration(&mut self, execute: FnPtr, rollback: FnPtr) {
        self.commands.push(Command { execute, rollback });
    }
    fn execute(&self) -> Vec<String> {
        self.commands.iter().map(|cmd| (cmd.execute)()).collect()
    }
    fn rollback(&self) -> Vec<String> {
        self.commands
            .iter()
            .rev()
            .map(|cmd| (cmd.rollback)())
            .collect()
    }
}

fn add_field() -> String {
    "add field".to_string()
}

fn remove_field() -> String {
    "remove field".to_string()
}

fn main() {
    let mut schema = Schema::new();
    schema.add_migration(|| "create table".to_string(), || "drop table".to_string());
    schema.add_migration(add_field, remove_field);
    assert_eq!(vec!["create table", "add field"], schema.execute());
    assert_eq!(vec!["remove field", "drop table"], schema.rollback());
}
```

## Подход: Использование объектов трейта `Fn`

Наконец, вместо определения общего трейта команд мы можем хранить каждую команду, реализующую трейт `Fn`, отдельно в векторах.

```rust
type Migration<'a> = Box<dyn Fn() -> &'a str>;

struct Schema<'a> {
    executes: Vec<Migration<'a>>,
    rollbacks: Vec<Migration<'a>>,
}

impl<'a> Schema<'a> {
    fn new() -> Self {
        Self {
            executes: vec![],
            rollbacks: vec![],
        }
    }
    fn add_migration<E, R>(&mut self, execute: E, rollback: R)
    where
        E: Fn() -> &'a str + 'static,
        R: Fn() -> &'a str + 'static,
    {
        self.executes.push(Box::new(execute));
        self.rollbacks.push(Box::new(rollback));
    }
    fn execute(&self) -> Vec<&str> {
        self.executes.iter().map(|cmd| cmd()).collect()
    }
    fn rollback(&self) -> Vec<&str> {
        self.rollbacks.iter().rev().map(|cmd| cmd()).collect()
    }
}

fn add_field() -> &'static str {
    "add field"
}

fn remove_field() -> &'static str {
    "remove field"
}

fn main() {
    let mut schema = Schema::new();
    schema.add_migration(|| "create table", || "drop table");
    schema.add_migration(add_field, remove_field);
    assert_eq!(vec!["create table", "add field"], schema.execute());
    assert_eq!(vec!["remove field", "drop table"], schema.rollback());
}
```

## Обсуждение

Если наши команды небольшие и могут быть определены как функции или переданы в виде замыкания, то использование указателей на функции может быть предпочтительнее, поскольку это не использует динамическую диспетчеризацию. Но если наша команда - это целый struct с набором функций и переменных, определенных как отдельный модуль, то использование объектов трейтов будет более подходящим. Пример применения можно найти в [`actix`](https://actix.rs/), который использует объекты трейтов при регистрации функции обработчика для маршрутов. В случае использования объектов трейта `Fn` мы можем создавать и использовать команды так же, как мы использовали указатели на функции.

Что касается производительности, всегда есть компромисс между производительностью и простотой и организацией кода. Статическая диспетчеризация обеспечивает более быструю производительность, а динамическая диспетчеризация обеспечивает гибкость при структурировании нашего приложения.

## Смотрите также

- [Паттерн Команда](https://en.wikipedia.org/wiki/Command_pattern)

- [Еще один пример для паттерна `command`](https://web.archive.org/web/20210223131236/https://chercher.tech/rust/command-design-pattern-rust)
