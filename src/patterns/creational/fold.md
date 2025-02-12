# Fold

## Описание

Запустите алгоритм над каждым элементом в коллекции данных, чтобы создать новый элемент, тем самым создавая целую новую коллекцию.

Этимология здесь мне не ясна. Термины «fold» и «folder» используются в компиляторе Rust, хотя мне кажется, что это больше похоже на карту, чем на свертку в обычном смысле. См. Обсуждение ниже для получения более подробной информации.

## Пример

```rust,ignore
// The data we will fold, a simple AST.
mod ast {
    pub enum Stmt {
        Expr(Box<Expr>),
        Let(Box<Name>, Box<Expr>),
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

// The abstract folder
mod fold {
    use ast::*;

    pub trait Folder {
        // A leaf node just returns the node itself. In some cases, we can do this
        // to inner nodes too.
        fn fold_name(&mut self, n: Box<Name>) -> Box<Name> { n }
        // Create a new inner node by folding its children.
        fn fold_stmt(&mut self, s: Box<Stmt>) -> Box<Stmt> {
            match *s {
                Stmt::Expr(e) => Box::new(Stmt::Expr(self.fold_expr(e))),
                Stmt::Let(n, e) => Box::new(Stmt::Let(self.fold_name(n), self.fold_expr(e))),
            }
        }
        fn fold_expr(&mut self, e: Box<Expr>) -> Box<Expr> { ... }
    }
}

use fold::*;
use ast::*;

// An example concrete implementation - renames every name to 'foo'.
struct Renamer;
impl Folder for Renamer {
    fn fold_name(&mut self, n: Box<Name>) -> Box<Name> {
        Box::new(Name { value: "foo".to_owned() })
    }
    // Use the default methods for the other nodes.
}
```

Результат выполнения `Renamer` на AST - это новый AST, идентичный старому, но с каждым именем, измененным на `foo`. В реальной жизни папка может сохранять некоторое состояние между узлами в самой структуре.

Также можно определить папку для отображения одной структуры данных на другую (но обычно похожую) структуру данных. Например, мы могли бы свернуть AST в дерево HIR (HIR означает промежуточное представление высокого уровня).

## Мотивация

Часто хочется отобразить структуру данных, выполнив некоторую операцию над каждым узлом в структуре. Для простых операций над простыми структурами данных это можно сделать с помощью `Iterator::map`. Для более сложных операций, возможно, где более ранние узлы могут влиять на операцию на более поздних узлах, или где итерация по структуре данных не тривиальна, использование паттерна свертки более подходит.

Как и паттерн посетителя, паттерн свертки позволяет нам отделить обход структуры данных от операций, выполняемых для каждого узла.

## Обсуждение

Отображение структур данных таким образом распространено в функциональных языках. В объектно-ориентированных языках было бы более распространено изменение структуры данных на месте. «Функциональный» подход распространен в Rust, в основном из-за предпочтения неизменяемости. Использование новых структур данных, а не изменение старых, облегчает рассуждение о коде в большинстве случаев.

Компромисс между эффективностью и повторным использованием можно настроить, изменив способ принятия узлов методами `fold_*`.

В приведенном выше примере мы работаем с указателями `Box`. Поскольку они владеют своими данными исключительно, исходная копия структуры данных не может быть повторно использована. С другой стороны, если узел не изменен, его повторное использование очень эффективно.

Если бы мы работали с ссылками на заимствование, исходная структура данных могла бы быть повторно использована; однако узел должен быть клонирован даже в случае, если он не изменен, что может быть дорогостоящим.

Использование указателя с подсчетом ссылок дает лучшее из обоих миров - мы можем повторно использовать исходную структуру данных, и нам не нужно клонировать неизмененные узлы. Однако они менее эргономичны в использовании и означают, что структуры данных не могут быть изменяемыми.

## См. также

У итераторов есть метод `fold`, однако это сворачивает структуру данных в значение, а не в новую структуру данных. `map` итератора больше похож на этот паттерн свертки.

В других языках свертка обычно используется в смысле итераторов Rust, а не этого паттерна. Некоторые функциональные языки имеют мощные конструкции для выполнения гибких отображений над структурами данных.

Паттерн [посетитель](../behavioural/visitor.md) тесно связан со сверткой. Они делят концепцию обхода структуры данных, выполняя операцию на каждом узле. Однако посетитель не создает новую структуру данных и не потребляет старую.```
