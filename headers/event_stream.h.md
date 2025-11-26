## Общее описание

`event_stream` — шаблонный класс для создания потоков событий в RPL. Позволяет отправлять значения подписчикам и управлять подписками.

## Параметры шаблона

```cpp
template <typename Value = empty_value, typename Error = no_error>
```

- `Value` — тип отправляемых значений (по умолчанию `empty_value` — сигнал без данных)
- `Error` — тип ошибок (по умолчанию `no_error`)

## Конструкторы и деструктор

1. `event_stream() noexcept = default` — конструктор по умолчанию
2. `event_stream(event_stream &&other)` — move-конструктор
3. `operator=(event_stream &&other)` — move-оператор присваивания (при присваивании старый поток завершается через `fire_done()`)
4. `~event_stream()` — деструктор (автоматически вызывает `fire_done()` для всех подписчиков)

## Отправка значений (fire)

3. `fire_forward(OtherValue &&value)` (приватный шаблон) — отправляет значение с perfect forwarding
4. `fire(Value &&value)` — отправляет значение через move
   ```cpp
   stream.fire(std::move(myValue));  // Значение перемещается
   ```
5. `fire_copy(const Value &value)` — отправляет копию значения
   ```cpp
   stream.fire_copy(currentState);  // Значение копируется для всех подписчиков
   ```

## Отправка ошибок (fire_error)

6. `fire_error_forward(OtherError &&error)` (приватный шаблон) — отправляет ошибку с perfect forwarding
7. `fire_error(Error &&error)` — отправляет ошибку через move
8. `fire_error_copy(const Error &error)` — отправляет копию ошибки

## Завершение потока

9. `fire_done()` — сигнализирует всем подписчикам о завершении потока (вызывается автоматически в деструкторе)

## Получение producer'ов для подписки

10. `events()` — возвращает `rpl::producer<Value, Error>`, который эмитит только будущие события (без текущего значения)
    ```cpp
    auto producer = stream.events();  // Будут только новые события
    ```

11. `events_starting_with(Value &&value)` — возвращает producer, который сначала отправляет переданное значение, затем будущие события
    ```cpp
    auto producer = stream.events_starting_with(std::move(initialValue));
    ```

12. `events_starting_with_copy(const Value &value)` — то же, но принимает значение по константной ссылке (копирует)
    ```cpp
    auto producer = stream.events_starting_with_copy(currentState);
    ```

## Утилиты

13. `has_consumers()` — проверяет, есть ли активные подписчики
    ```cpp
    if (stream.has_consumers()) {
        // Кто-то подписан, можно отправлять события
    }
    ```

## Пример использования

```cpp
// Создание потока
rpl::event_stream<QString> textChanges;

// Подписка
rpl::lifetime lifetime;
textChanges.events_starting_with_copy(currentText)
    | rpl::start_with_next([=](const QString &text) {
        label->setText(text);
    }, lifetime);

// Отправка событий
textChanges.fire_copy("New text");  // Все подписчики получат это значение
```

## Особенности реализации

- Не потокобезопасен (см. комментарий на строке 20)
- Использует `shared_ptr` для хранения списка подписчиков
- Оптимизирует копирование: для всех подписчиков кроме последнего копирует значение, последний получает через move
- Поддерживает вложенные вызовы `fire()` (отслеживается через `depth`)

`event_stream` — базовый механизм для создания источников событий в реактивных цепочках RPL.