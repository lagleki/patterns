# Принятие строк

## Описание

При принятии строк через FFI через указатели, следует придерживаться двух принципов:

1. Храните внешние (foreign) строки "заимствованными", а не копируйте их напрямую.
2. Минимизируйте количество сложности и `unsafe` кода, связанного с преобразованием из строки в стиле C в собственные строки Rust.

## Мотивация

Строки, используемые в C, имеют другое поведение, чем те, которые используются в Rust, а именно:

- Строки C завершаются нулем, в то время как строки Rust хранят свою длину.
- Строки C могут содержать любой произвольный ненулевой байт, в то время как строки Rust должны быть в формате UTF-8.
- Строки C доступны и манипулируются с помощью операций указателя `unsafe`, в то время как взаимодействие со строками Rust происходит через безопасные методы.

Стандартная библиотека Rust поставляется с эквивалентами C для `String` и `&str` Rust, называемыми `CString` и `&CStr`, которые позволяют нам избежать многих сложностей и `unsafe` кода, связанного с преобразованием между строками C и строками Rust.

Тип `&CStr` также позволяет нам работать с заимствованными данными, что означает, что передача строк между Rust и C является операцией с нулевой стоимостью.

## Пример кода

```rust,ignore
pub mod unsafe_module {

    // other module content

    /// Log a message at the specified level.
    ///
    /// # Safety
    ///
    /// It is the caller's guarantee to ensure `msg`:
    ///
    /// - is not a null pointer
    /// - points to valid, initialized data
    /// - points to memory ending in a null byte
    /// - won't be mutated for the duration of this function call
    #[no_mangle]
    pub unsafe extern "C" fn mylib_log(
        msg: *const libc::c_char,
        level: libc::c_int
    ) {
        let level: crate::LogLevel = match level { /* ... */ };

        // SAFETY: The caller has already guaranteed this is okay (see the
        // `# Safety` section of the doc-comment).
        let msg_str: &str = match std::ffi::CStr::from_ptr(msg).to_str() {
            Ok(s) => s,
            Err(e) => {
                crate::log_error("FFI string conversion failed");
                return;
            }
        };

        crate::log(msg_str, level);
    }
}
```

## Преимущества

Пример написан таким образом, чтобы:

1. Блок `unsafe` был как можно меньше.
2. Указатель с "ненаблюдаемым" временем жизни становится "наблюдаемым" общим ссылкой.

Рассмотрим альтернативу, где строка фактически копируется:

```rust,ignore
pub mod unsafe_module {

    // other module content

    pub extern "C" fn mylib_log(msg: *const libc::c_char, level: libc::c_int) {
        // DO NOT USE THIS CODE.
        // IT IS UGLY, VERBOSE, AND CONTAINS A SUBTLE BUG.

        let level: crate::LogLevel = match level { /* ... */ };

        let msg_len = unsafe { /* SAFETY: strlen is what it is, I guess? */
            libc::strlen(msg)
        };

        let mut msg_data = Vec::with_capacity(msg_len + 1);

        let msg_cstr: std::ffi::CString = unsafe {
            // SAFETY: copying from a foreign pointer expected to live
            // for the entire stack frame into owned memory
            std::ptr::copy_nonoverlapping(msg, msg_data.as_mut(), msg_len);

            msg_data.set_len(msg_len + 1);

            std::ffi::CString::from_vec_with_nul(msg_data).unwrap()
        }

        let msg_str: String = unsafe {
            match msg_cstr.into_string() {
                Ok(s) => s,
                Err(e) => {
                    crate::log_error("FFI string conversion failed");
                    return;
                }
            }
        };

        crate::log(&msg_str, level);
    }
}
```

Этот код уступает оригиналу в двух отношениях:

1. Есть гораздо больше `unsafe` кода, и, что более важно, больше инвариантов, которые он должен соблюдать.
2. Из-за обширной арифметики есть ошибка в этой версии, которая вызывает `undefined behaviour` в Rust.

Ошибка здесь заключается в простой ошибке в арифметике указателей: строка была скопирована, все `msg_len` байтов. Однако завершающий `NUL` не был скопирован.

Затем вектор был _установлен_ на длину _нулевой заполненной строки_, а не _изменен_ на нее, что могло бы добавить ноль в конце. В результате последний байт в векторе является неинициализированной памятью. Когда `CString` создается внизу блока, его чтение из вектора вызовет `undefined behaviour`!

Как и многие подобные проблемы, это было бы трудно отследить. Иногда он вызывал бы панику, потому что строка не была `UTF-8`, иногда он добавлял бы странный символ в конец строки, иногда он просто полностью завершал работу.

## Недостатки

Нет?
