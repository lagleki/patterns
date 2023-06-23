# Простая инициализация документации

## Описание

Если требуется значительное усилие при инициализации struct-а во время написания документации, то быстрее обернуть Ваш пример в функцию-помощник, которая принимает struct в качестве аргумента.

## Мотивация

Иногда есть struct с несколькими или сложными параметрами и несколькими методами. Каждый из этих методов должен иметь примеры.

Например:

````rust,ignore
struct Connection {
    name: String,
    stream: TcpStream,
}

impl Connection {
    /// Sends a request over the connection.
    ///
    /// # Example
    /// ```no_run
    /// # // Boilerplate are required to get an example working.
    /// # let stream = TcpStream::connect("127.0.0.1:34254");
    /// # let connection = Connection { name: "foo".to_owned(), stream };
    /// # let request = Request::new("RequestId", RequestType::Get, "payload");
    /// let response = connection.send_request(request);
    /// assert!(response.is_ok());
    /// ```
    fn send_request(&self, request: Request) -> Result<Status, SendErr> {
        // ...
    }

    /// Oh no, all that boilerplate needs to be repeated here!
    fn check_status(&self) -> Status {
        // ...
    }
}
````

## Пример

Вместо того, чтобы набирать все эти паттерны для создания `Connection` и `Request`, легче создать обертывающую вспомогательную функцию, которая принимает их в качестве аргументов:

````rust,ignore
struct Connection {
    name: String,
    stream: TcpStream,
}

impl Connection {
    /// Sends a request over the connection.
    ///
    /// # Example
    /// ```
    /// # fn call_send(connection: Connection, request: Request) {
    /// let response = connection.send_request(request);
    /// assert!(response.is_ok());
    /// # }
    /// ```
    fn send_request(&self, request: Request) {
        // ...
    }
}
````

**Примечание** в приведенном выше примере строка `assert!(response.is_ok());` на самом деле не будет запущена во время тестирования, потому что она находится внутри функции, которая никогда не вызывается.

## Преимущества

Это намного более кратко и избегает повторяющегося кода в примерах.

## Недостатки

Поскольку пример находится в функции, код не будет протестирован. Хотя он все еще будет проверен, чтобы убедиться, что он компилируется при запуске `cargo test`. Поэтому этот паттерн наиболее полезен, когда вам нужен `no_run`. С этим вам не нужно добавлять `no_run`.

## Обсуждение

Если утверждения (assertions) не требуются, этот паттерн работает хорошо.

Если они есть, альтернативой может быть создание общедоступного метода для создания вспомогательного экземпляра, который аннотирован с `#[doc(hidden)]` (чтобы пользователи его не видели). Затем этот метод может быть вызван внутри rustdoc, потому что он является частью общедоступного API крейта.
