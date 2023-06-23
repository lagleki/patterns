# Передача строк

## Описание

При передаче строк в функции FFI необходимо следовать четырем принципам:

1. Сделайте время жизни собственных строк максимально долгим.
2. Минимизируйте использование `unsafe` кода во время преобразования.
3. Если код на языке C может изменять данные строки, используйте `Vec` вместо `CString`.
4. Если API внешней функции не требует этого, собственность строки не должна передаваться вызываемой функции.

## Мотивация

Rust имеет встроенную поддержку строк в стиле C с помощью типов `CString` и `CStr`. Однако, есть разные подходы, которые можно использовать с передачей строк во внешнюю функцию из функции Rust.

Лучшая практика проста: используйте `CString` таким образом, чтобы минимизировать использование `unsafe` кода. Однако, второстепенное предостережение заключается в том, что _объект должен жить достаточно долго_, что означает, что время жизни должно быть максимальным. Кроме того, документация объясняет, что "round-tripping" `CString` после изменения является UB, поэтому в этом случае требуется дополнительная работа.

## Пример кода

```rust,ignore
pub mod unsafe_module {

    // other module content

    extern "C" {
        fn seterr(message: *const libc::c_char);
        fn geterr(buffer: *mut libc::c_char, size: libc::c_int) -> libc::c_int;
    }

    fn report_error_to_ffi<S: Into<String>>(
        err: S
    ) -> Result<(), std::ffi::NulError>{
        let c_err = std::ffi::CString::new(err.into())?;

        unsafe {
            // SAFETY: calling an FFI whose documentation says the pointer is
            // const, so no modification should occur
            seterr(c_err.as_ptr());
        }

        Ok(())
        // The lifetime of c_err continues until here
    }

    fn get_error_from_ffi() -> Result<String, std::ffi::IntoStringError> {
        let mut buffer = vec![0u8; 1024];
        unsafe {
            // SAFETY: calling an FFI whose documentation implies
            // that the input need only live as long as the call
            let written: usize = geterr(buffer.as_mut_ptr(), 1023).into();

            buffer.truncate(written + 1);
        }

        std::ffi::CString::new(buffer).unwrap().into_string()
    }
}
```

## Преимущества

Пример написан таким образом, чтобы:

1. Блок `unsafe` был как можно меньше.
2. `CString` жила достаточно долго.
3. Ошибки с приведением типов всегда передаются, когда это возможно.

Частой ошибкой (настолько частой, что она присутствует в документации) является неправильное использование переменной в первом блоке:

```rust,ignore
pub mod unsafe_module {

    // other module content

    fn report_error<S: Into<String>>(err: S) -> Result<(), std::ffi::NulError> {
        unsafe {
            // SAFETY: whoops, this contains a dangling pointer!
            seterr(std::ffi::CString::new(err.into())?.as_ptr());
        }
        Ok(())
    }
}
```

Этот код приведет к висячему указателю, потому что время жизни `CString` не продлевается созданием указателя, в отличие от создания ссылки.

Еще одной часто возникающей проблемой является то, что инициализация 1k вектора нулей "медленная". Однако, последние версии Rust фактически оптимизируют этот конкретный макрос до вызова `zmalloc`, что означает, что это быстро, как способность операционной системы возвращать обнуленную память (что довольно быстро).

## Недостатки

Нет?
