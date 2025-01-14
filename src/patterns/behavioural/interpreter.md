# Интерпретатор

## Описание

Если проблема возникает очень часто и требует долгих и повторяющихся шагов для ее решения, то экземпляры проблем могут быть выражены на простом языке, и объект-интерпретатор может решить ее, интерпретируя предложения, написанные на этом простом языке.

В основном, для любых видов проблем мы определяем:

- [Язык, специфичный для предметной области](https://en.wikipedia.org/wiki/Domain-specific_language),
- Грамматика для этого языка,
- Интерпретатор, который решает экземпляры проблем.

## Мотивация

Наша цель - перевести простые математические выражения в постфиксные выражения
(или [обратную польскую запись](https://en.wikipedia.org/wiki/Reverse_Polish_notation))
Для простоты наши выражения состоят из десяти цифр `0`, ..., `9` и двух
операций `+`, `-`. Например, выражение `2 + 4` переводится в
`2 4 +`.

## Контекстно-свободная грамматика для нашей проблемы

Наша задача - перевод инфиксных выражений в постфиксные. Давайте определим контекстно-свободную грамматику для набора инфиксных выражений над `0`, ..., `9`, `+` и `-`,
где:

- Терминальные символы: `0`, `...`, `9`, `+`, `-`
- Нетерминальные символы: `exp`, `term`
- Стартовый символ - `exp`
- И следующие правила производства

```ignore
exp -> exp + term
exp -> exp - term
exp -> term
term -> 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9
```

**ПРИМЕЧАНИЕ:** Эта грамматика должна быть дополнительно преобразована в зависимости от того, что мы собираемся сделать с ней. Например, мы можем потребовать удаления левой рекурсии. Для получения более подробной информации см. [Компиляторы: принципы, техники и инструменты](https://en.wikipedia.org/wiki/Compilers:_Principles,_Techniques,_and_Tools)
(также известная как книга Dragon).

## Решение

Мы просто реализуем рекурсивный спуск парсера. Для упрощения код
выдает ошибку при синтаксической неправильности выражения (например, `2-34` или `2+5-`
неправильны согласно определению грамматики).

```rust
pub struct Interpreter<'a> {
    it: std::str::Chars<'a>,
}

impl<'a> Interpreter<'a> {

    pub fn new(infix: &'a str) -> Self {
        Self { it: infix.chars() }
    }

    fn next_char(&mut self) -> Option<char> {
        self.it.next()
    }

    pub fn interpret(&mut self, out: &mut String) {
        self.term(out);

        while let Some(op) = self.next_char() {
            if op == '+' || op == '-' {
                self.term(out);
                out.push(op);
            } else {
                panic!("Unexpected symbol '{}'", op);
            }
        }
    }

    fn term(&mut self, out: &mut String) {
        match self.next_char() {
            Some(ch) if ch.is_digit(10) => out.push(ch),
            Some(ch) => panic!("Unexpected symbol '{}'", ch),
            None => panic!("Unexpected end of string"),
        }
    }
}

pub fn main() {
    let mut intr = Interpreter::new("2+3");
    let mut postfix = String::new();
    intr.interpret(&mut postfix);
    assert_eq!(postfix, "23+");

    intr = Interpreter::new("1-2+3-4");
    postfix.clear();
    intr.interpret(&mut postfix);
    assert_eq!(postfix, "12-3+4-");
}
```

## Обсуждение

Может быть неверное восприятие того, что паттерн проектирования Интерпретатор связан с проектированием грамматик для формальных языков и реализацией парсеров для этих грамматик.
На самом деле этот паттерн связан с выражением экземпляров проблемы более конкретным
способом и реализацией функций/классов/структур, которые решают эти экземпляры проблемы.
Язык Rust имеет `macro_rules!`, которые позволяют нам определять специальный синтаксис и правила
для расширения этого синтаксиса в исходный код.

В следующем примере мы создаем простой `macro_rules!`, который вычисляет
[Евклидову длину](https://en.wikipedia.org/wiki/Euclidean_distance) `n`
мерных векторов. Запись `norm!(x,1,2)` может быть проще выразить и более
эффективно, чем упаковка `x,1,2` в `Vec` и вызов функции, вычисляющей длину.

```rust
macro_rules! norm {
    ($($element:expr),*) => {
        {
            let mut n = 0.0;
            $(
                n += ($element as f64)*($element as f64);
            )*
            n.sqrt()
        }
    };
}

fn main() {
    let x = -3f64;
    let y = 4f64;

    assert_eq!(3f64, norm!(x));
    assert_eq!(5f64, norm!(x, y));
    assert_eq!(0f64, norm!(0, 0, 0)); 
    assert_eq!(1f64, norm!(0.5, -0.5, 0.5, -0.5));
}
```

## Смотрите также

- [Паттерн Интерпретатор](https://en.wikipedia.org/wiki/Interpreter_pattern)
- [Контекстно-свободная грамматика](https://en.wikipedia.org/wiki/Context-free_grammar)
- [macro_rules!](https://doc.rust-lang.org/rust-by-example/macros.html)
