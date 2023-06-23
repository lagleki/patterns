# Возвратить потребленный аргумент при ошибке

## Описание

Если функция, которая может вернуть ошибку, потребляет (перемещает) аргумент, то вернуть этот аргумент внутри ошибки.

## Пример

```rust
pub fn send(value: String) -> Result<(), SendError> {
    println!("using {value} in a meaningful way");
    // Simulate non-deterministic fallible action.
    use std::time::SystemTime;
    let period = SystemTime::now().duration_since(SystemTime::UNIX_EPOCH).unwrap();
    if period.subsec_nanos() % 2 == 1 {
        Ok(())
    } else {
        Err(SendError(value))
    }
}

pub struct SendError(String);

fn main() {
    let mut value = "imagine this is very long string".to_string();

    let success = 's: {
        // Try to send value two times.
        for _ in 0..2 {
            value = match send(value) {
                Ok(()) => break 's true,
                Err(SendError(value)) => value,
            }
        }
        false
    };

    println!("success: {}", success);
}
```

## Мотивация

В случае ошибки вы можете захотеть попробовать альтернативный путь или повторить действие в случае недетерминированной функции. Но если аргумент всегда потребляется, то вы вынуждены клонировать его при каждом вызове, что не очень эффективно.

Стандартная библиотека использует этот подход, например, в методе `String::from_utf8`. Если дан вектор, который не содержит допустимый UTF-8, возвращается `FromUtf8Error`. Вы можете получить исходный вектор обратно, используя метод `FromUtf8Error::into_bytes`.

## Преимущества

Лучшая производительность за счет перемещения аргументов, когда это возможно.

## Недостатки

Немного более сложные типы ошибок.
