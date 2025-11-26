### Кратко об интерфейсе `rpl/producer.h`

- **Назначение**: определяет основной тип реактивного потока значений `rpl::producer<Value, Error, Generator>` и весь “стартовый” API для подписки (`start_*`) с управлением временем жизни через `rpl::lifetime`.

### Ключевые сущности

- **`rpl::producer<Value, Error, Generator>`**
  - Поток значений типа `Value` и ошибок типа `Error` (по умолчанию `no_error`).
  - Последний шаблонный параметр — стратегия генератора (`Generator`), по умолчанию тип-стираемый.
  - Непосредственно наследует реализацию из `details::producer_base`.

- **`details::producer_base<Value, Error, Generator>`**
  - Хранит генератор и реализует базовые способы подписки:
    - `start(next, error, done, lifetime&) &&`
    - `start(next, error, done) && -> lifetime`
    - `start_copy(...) const &` варианты — подписка без перемещения исходного producer.
    - Внутри все сводится к `start_existing(consumer, lifetime)`.

- **Тип-стираемый генератор**: `details::type_erased_generator<Value, Error>`
  - Обёртка на `std::function` для хранения генератора как копируемого, мутабельного коллбэка.
  - Позволяет собирать `producer` с единым runtime-представлением генератора (включается всегда в `_DEBUG`/MSVC).

- **`make_producer(generator)`**
  - Удобный конструктор producer из произвольного генератора, совместимого по сигнатуре (возвращает `lifetime` при вызове с `consumer`).
  - В дебаг/MSC — всегда возвращает тип-стираемый `producer<Value, Error>`.

- **`duplicate(producer)`**
  - Возвращает копию producer для повторного использования (важно, т.к. запуск “потребляет” producer).

- **Пайп-оператор**: `operator|`
  - Передаёт `producer` в “методы” (операторы) трансформации/старта.
  - Есть перегрузки для всех помощников `start_*` (см. ниже), как с возвратом `lifetime`, так и с внешним `lifetime&`.

### Подписка и хэндлеры

- Набор фабрик-хелперов, возвращающих “контейнеры хэндлеров”, чтобы запускать через `operator|`:
  - `start()` — без хэндлеров.
  - `start(lifetime&)` — без хэндлеров, но с внешним временем жизни.
  - `start_with_next(next[, lifetime&])`
  - `start_with_error(error[, lifetime&])`
  - `start_with_done(done[, lifetime&])`
  - Комбинированные:
    - `start_with_next_error(next, error[, lifetime&])`
    - `start_with_error_done(error, done[, lifetime&])`
    - `start_with_next_done(next, done[, lifetime&])`
    - `start_with_next_error_done(next, error, done[, lifetime&])`
- Для каждого варианта есть соответствующие перегрузки `operator|`, которые:
  - Либо возвращают `lifetime`, если вы его не передали,
  - Либо подписывают в переданный `lifetime&`.

### Взаимодействие с `lifetime`

- Любая подписка добавляет терминирующую очистку в `lifetime`.
- Если не передан внешний `lifetime&`, перегрузки возвращают новый `lifetime`, который нужно сохранить, иначе подписка немедленно завершится.

### Супертипные объединения

- `details::superset_type` специализации позволяют типобезопасно объединять producers с разными `Value/Error` или генераторами, выводя общий “супертип”.

### Прочее

- `producer` можно неявно приводить к `bool` для проверки “пустоты” (наличия генератора).
- В файле также есть служебные типы `details::*` для упаковки хэндлеров, и утилиты для проверки вызываемости (`details::is_callable_v`).

### Как этим пользоваться (паттерн)

- Строим поток, трансформируем, затем стартуем и управляем временем жизни:
```cpp
auto p = /* producer<int> */;
rpl::lifetime l;
std::move(p)
| rpl::start_with_next([](int v) { /* ... */ }, l);
```

- Если `lifetime` не передаёте — сохраните возвращаемый объект:
```cpp
auto life = std::move(p)
| rpl::start_with_next([](int v) { /* ... */ });
```

- Для повторного запуска — используйте `rpl::duplicate(p)` или храните исходник и копируйте.

- В кодовой базе предпочтительнее `auto` для переменных и конвейеров.