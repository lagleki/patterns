# Передача переменных в замыкание

## Описание

По умолчанию замыкания заимствуют свою среду. Или вы можете использовать
`move`-замыкание, чтобы переместить всю среду. Однако часто вы хотите переместить только
некоторые переменные в замыкание, передать ему копию некоторых данных, передать по ссылке или
выполнить какое-то другое преобразование.

Для этого используйте перепривязку переменных в отдельной области видимости.

## Пример

Используйте

```rust
use std::rc::Rc;

let num1 = Rc::new(1);
let num2 = Rc::new(2);
let num3 = Rc::new(3);
let closure = {
    // `num1` is moved
    let num2 = num2.clone();  // `num2` is cloned
    let num3 = num3.as_ref();  // `num3` is borrowed
    move || {
        *num1 + *num2 + *num3;
    }
};
```

вместо

```rust
use std::rc::Rc;

let num1 = Rc::new(1);
let num2 = Rc::new(2);
let num3 = Rc::new(3);

let num2_cloned = num2.clone();
let num3_borrowed = num3.as_ref();
let closure = move || {
    *num1 + *num2_cloned + *num3_borrowed;
};
```

## Преимущества

Скопированные данные группируются вместе с определением замыкания, поэтому их назначение более ясно,
и они будут немедленно удалены, даже если они не будут использованы замыканием.

Замыкание использует те же имена переменных, что и окружающий код, независимо от того, скопированы ли данные или
перемещены.

## Недостатки

Дополнительный отступ тела замыкания.
