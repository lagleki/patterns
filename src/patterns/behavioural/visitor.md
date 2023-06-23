# Посетитель

## Описание

Посетитель инкапсулирует алгоритм, который работает с гетерогенной коллекцией объектов. Он позволяет написать несколько различных алгоритмов для одних и тех же данных, не изменяя сами данные (или их основное поведение).

Кроме того, паттерн посетитель позволяет разделить обход коллекции объектов и операции, выполняемые над каждым объектом.

## Пример

```rust,ignore
// The data we will visit
mod ast {
    pub enum Stmt {
        Expr(Expr),
        Let(Name, Expr),
    }

    pub struct Name {
        value: String,
    }

    pub enum Expr {
        IntLit(i64),
        Add(Box<Expr>, Box<Expr>),
        Sub(Box<Expr>, Box<Expr>),
    }
}

// The abstract visitor
mod visit {
    use ast::*;

    pub trait Visitor<T> {
        fn visit_name(&mut self, n: &Name) -> T;
        fn visit_stmt(&mut self, s: &Stmt) -> T;
        fn visit_expr(&mut self, e: &Expr) -> T;
    }
}

use visit::*;
use ast::*;

// An example concrete implementation - walks the AST interpreting it as code.
struct Interpreter;
impl Visitor<i64> for Interpreter {
    fn visit_name(&mut self, n: &Name) -> i64 { panic!() }
    fn visit_stmt(&mut self, s: &Stmt) -> i64 {
        match *s {
            Stmt::Expr(ref e) => self.visit_expr(e),
            Stmt::Let(..) => unimplemented!(),
        }
    }

    fn visit_expr(&mut self, e: &Expr) -> i64 {
        match *e {
            Expr::IntLit(n) => n,
            Expr::Add(ref lhs, ref rhs) => self.visit_expr(lhs) + self.visit_expr(rhs),
            Expr::Sub(ref lhs, ref rhs) => self.visit_expr(lhs) - self.visit_expr(rhs),
        }
    }
}
```

Можно реализовать дополнительные посетители, например, проверку типов, не изменяя данных AST.

## Мотивация

Паттерн посетитель полезен везде, где нужно применять алгоритм к гетерогенным данным. Если данные однородны, можно использовать паттерн, похожий на итератор. Использование объекта посетителя (а не функционального подхода) позволяет сделать посетителя состояний и, таким образом, обмениваться информацией между узлами.

## Обсуждение

Обычно методы `visit_*` возвращают `void` (в отличие от примера). В этом случае можно выделить код обхода и использовать его между алгоритмами (а также предоставить методы по умолчанию). В Rust обычно используются функции `walk_*` для каждого элемента данных. Например,

```rust,ignore
pub fn walk_expr(visitor: &mut Visitor, e: &Expr) {
    match *e {
        Expr::IntLit(_) => {},
        Expr::Add(ref lhs, ref rhs) => {
            visitor.visit_expr(lhs);
            visitor.visit_expr(rhs);
        }
        Expr::Sub(ref lhs, ref rhs) => {
            visitor.visit_expr(lhs);
            visitor.visit_expr(rhs);
        }
    }
}
```

В других языках (например, в Java) обычно у данных есть метод `accept`, который выполняет ту же функцию.

## Смотрите также

Паттерн посетитель является распространенным в большинстве объектно-ориентированных языков.

[Статья на Википедии](https://en.wikipedia.org/wiki/Visitor_pattern)

Паттерн [fold](../creational/fold.md) похож на посетитель, но создает новую версию посещаемой структуры данных.