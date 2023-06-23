# RAII с гвардами

## Описание

[RAII][wikipedia] расшифровывается как "Resource Acquisition is Initialisation" (инициализация приобретения ресурсов), что является ужасным названием. Суть паттерна заключается в том, что инициализация ресурсов выполняется в конструкторе объекта, а завершение - в деструкторе. В Rust этот паттерн расширяется с помощью RAII-объекта в качестве гварда некоторого ресурса, и полагаясь на систему типов, чтобы гарантировать, что доступ всегда осуществляется через объект-гвард.

## Пример

Гварды Mutex - это классический пример этого паттерна из стандартной библиотеки (это упрощенная версия реальной реализации):

```rust,ignore
use std::ops::Deref;

struct Foo {}

struct Mutex<T> {
    // We keep a reference to our data: T here.
    //..
}

struct MutexGuard<'a, T: 'a> {
    data: &'a T,
    //..
}

// Locking the mutex is explicit.
impl<T> Mutex<T> {
    fn lock(&self) -> MutexGuard<T> {
        // Lock the underlying OS mutex.
        //..

        // MutexGuard keeps a reference to self
        MutexGuard {
            data: self,
            //..
        }
    }
}

// Destructor for unlocking the mutex.
impl<'a, T> Drop for MutexGuard<'a, T> {
    fn drop(&mut self) {
        // Unlock the underlying OS mutex.
        //..
    }
}

// Implementing Deref means we can treat MutexGuard like a pointer to T.
impl<'a, T> Deref for MutexGuard<'a, T> {
    type Target = T;

    fn deref(&self) -> &T {
        self.data
    }
}

fn baz(x: Mutex<Foo>) {
    let xx = x.lock();
    xx.foo(); // foo is a method on Foo.
    // The borrow checker ensures we can't store a reference to the underlying
    // Foo which will outlive the guard xx.

    // x is unlocked when we exit this function and xx's destructor is executed.
}
```

## Мотивация

Если ресурс должен быть завершен после использования, RAII может использоваться для этого завершения. Если использование этого ресурса после завершения является ошибкой, то этот паттерн может использоваться для предотвращения таких ошибок.

## Преимущества

Предотвращает ошибки, когда ресурс не завершен и когда ресурс используется после завершения.

## Обсуждение

RAII - это полезный паттерн для обеспечения правильного освобождения или завершения ресурсов. Мы можем использовать borrow checker в Rust, чтобы статически предотвратить ошибки, связанные с использованием ресурсов после завершения.

Основная цель borrow checker - это гарантировать, что ссылки на данные не превышают их жизненный цикл. Паттерн гварда RAII работает, потому что объект-гвард содержит ссылку на базовый ресурс и только такие ссылки. Rust гарантирует, что гвард не может превысить базовый ресурс, и что ссылки на ресурс, осуществляемые через гвард, не могут превысить гвард. Чтобы понять, как это работает, полезно рассмотреть сигнатуру `deref` без опущения времени жизни:

```rust,ignore
fn deref<'a>(&'a self) -> &'a T {
    //..
}
```

Возвращаемая ссылка на ресурс имеет тот же срок жизни, что и `self` (`'a`). Borrow checker, следовательно, гарантирует, что срок жизни ссылки на `T` короче, чем срок жизни `self`.

Обратите внимание, что реализация `Deref` не является основной частью этого паттерна, это только делает использование объекта-гварда более эргономичным. Реализация метода `get` на гварде работает так же хорошо.

## Смотрите также

[Finalisation in destructors idiom](../../idioms/dtor-finally.md)

RAII - это распространенный паттерн в C++: [cppreference.com](http://en.cppreference.com/w/cpp/language/raii),
[wikipedia][wikipedia].

[wikipedia]: https://en.wikipedia.org/wiki/Resource_Acquisition_Is_Initialization

[Запись в стилевом руководстве](https://doc.rust-lang.org/1.0.0/style/ownership/raii.html)
(в настоящее время просто заполнитель).