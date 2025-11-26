Изучаю реализацию оператора `|` (pipe) в RPL:


[4 tools called]


Объясняю, как работает оператор `|` (pipe) в RPL:

## Как работает оператор Pipe (`|`) в RPL

### 1. Основная концепция

Оператор `|` в RPL — это перегруженный оператор C++, который позволяет строить цепочки операций над producers. Похоже на pipe в shell (`command1 | command2 | command3`).

### 2. Два типа использования pipe

#### Тип 1: С операторами трансформации (map, filter, take и т.д.)

```cpp
std::move(numbers)
    | rpl::map([](int x) { return x * 2; })      // ← Возвращает новый producer
    | rpl::filter([](int x) { return x > 10; })  // ← Возвращает новый producer
    | rpl::start_with_next(...);                 // ← Запускает подписку
```

#### Тип 2: С подпиской (start_with_next, start_with_done и т.д.)

```cpp
std::move(singleValue) | rpl::start_with_next([=](int value) {
    std::cout << value << std::endl;
}, lifetime);
```

### 3. Как это работает технически

#### Для операторов трансформации (map, filter):

```cpp
// Когда вы пишете:
auto result = std::move(producer) | rpl::map([](int x) { return x * 2; });

// Это эквивалентно:
auto mapHelper = rpl::map([](int x) { return x * 2; });  // Создает helper объект
auto result = mapHelper(std::move(producer));             // Вызывает operator()
```

Реализация `map`:

```cpp
template <typename Transform>
class map_helper {
    template <typename Value, typename Error, typename Generator>
    auto operator()(producer<Value, Error, Generator> &&initial) {
        // Создает НОВЫЙ producer, который:
        // 1. Подписывается на initial producer
        // 2. Применяет transform к каждому значению
        // 3. Отправляет преобразованное значение дальше
        return make_producer<NewValue, Error>([...](const auto &consumer) {
            return std::move(initial).start(
                map_transform(transform, consumer),  // ← Применяет transform
                [consumer](auto &&error) { ... },
                [consumer] { ... }
            );
        });
    }
};
```

#### Для подписки (start_with_next):

```cpp
// Когда вы пишете:
std::move(producer) | rpl::start_with_next(callback, lifetime);

// Это вызывает:
operator|(producer &&value, lifetime_with_next &&handlers) {
    std::move(value).start(
        std::move(handlers.next),  // ваш callback
        [] {},                      // обработчик ошибок
        [] {},                      // обработчик завершения
        handlers.alive_while        // lifetime
    );
}

    auto producer = rpl::ints(4);
    auto handlers = rpl::start_with_next([=](**int** value) {
        std::cout << "Получено значение: " << value << std::endl;
    }, lifetime);
    
	// это
    std::move(producer) | std::move(handlers);
    // эквивалнтно этому
    operator|(std::move(producer), std::move(handlers));
    // и этому
    std::move(producer).start(
        [=](**int** value) {
            std::cout << "Получено new new значение: " << value << std::endl;
        },                    // ← callback для значений (OnNext)
        [] {},                // ← callback для ошибок (OnError) - пустой
        [] {},                // ← callback для завершения (OnDone) - пустой
        lifetime              // ← lifetime для управления подпиской  
    );
```

### 4. Визуализация цепочки

```cpp
auto numbers = rpl::ints(1, 6);  // producer<int>

std::move(numbers)
    | rpl::map([](int x) { return x * 2; })      // producer<int> → producer<int>
    | rpl::filter([](int x) { return x > 5; })   // producer<int> → producer<int>
    | rpl::start_with_next([=](int value) {      // producer<int> → void (подписка)
        std::cout << value << std::endl;
    }, lifetime);
```

Выполнение:

```
numbers (1,2,3,4,5)
    ↓
[map: x * 2] → (2,4,6,8,10)
    ↓
[filter: x > 5] → (6,8,10)
    ↓
[start_with_next] → callback(6), callback(8), callback(10)
```

### 5. Важные особенности

#### 5.1. Ленивая оценка

```cpp
auto mapped = std::move(numbers) | rpl::map([](int x) { return x * 2; });
// mapped - это producer, но он еще НЕ активен!
// Трансформация начнется только при подписке
```

#### 5.2. Каждый оператор создает новый producer

```cpp
auto p1 = rpl::ints(1, 10);           // producer<int>
auto p2 = std::move(p1) | rpl::map(...);  // НОВЫЙ producer<int>
auto p3 = std::move(p2) | rpl::filter(...); // ЕЩЕ ОДИН НОВЫЙ producer<int>
// p1 и p2 больше не валидны после move
```

#### 5.3. Move-семантика

```cpp
auto numbers = rpl::ints(1, 6);
std::move(numbers) | rpl::map(...);  // ✅ Правильно
numbers | rpl::map(...);             // ❌ Может не скомпилироваться или работать неэффективно
```

### 6. Пример с разбором

```cpp
void example() {
    auto numbers = rpl::ints(1, 6);  // producer<int>: (1,2,3,4,5)
    rpl::lifetime lifetime;
    
    // Шаг 1: map создает новый producer
    auto doubled = std::move(numbers) 
        | rpl::map([](int x) { return x * 2; });
    // doubled: producer<int> который будет эмитить (2,4,6,8,10)
    // НО пока не активен!
    
    // Шаг 2: filter создает еще один новый producer
    auto filtered = std::move(doubled)
        | rpl::filter([](int x) { return x > 5; });
    // filtered: producer<int> который будет эмитить (6,8,10)
    // Тоже пока не активен!
    
    // Шаг 3: start_with_next АКТИВИРУЕТ всю цепочку
    std::move(filtered) | rpl::start_with_next([=](int value) {
        std::cout << value << std::endl;  // Выведет: 6, 8, 10
    }, lifetime);
    // Теперь:
    // 1. filtered подписывается на doubled
    // 2. doubled подписывается на numbers
    // 3. numbers начинает эмитить значения
    // 4. Каждое значение проходит через map → filter → callback
}
```

### 7. Порядок выполнения

```cpp
std::move(producer) | op1 | op2 | op3 | start_with_next(...);
```

Порядок:
1. Создается `op1` (helper объект)
2. Вызывается `op1(std::move(producer))` → новый producer
3. Создается `op2`
4. Вызывается `op2(std::move(producer_from_op1))` → еще один producer
5. И так далее...
6. В конце `start_with_next` активирует всю цепочку

### 8. Аналогия

Представьте конвейер:

```
[Сырье] → [Машина 1] → [Машина 2] → [Машина 3] → [Готовый продукт]
```

В RPL:
```
[Producer] | [map] | [filter] | [start_with_next] → [Результат]
```

Каждая "машина" (оператор) создает новую "линию конвейера" (новый producer), которая обрабатывает значения из предыдущей.

### 9. Итог

- Оператор `|` — это синтаксический сахар для цепочек операций
- Каждый оператор (map, filter и т.д.) возвращает новый producer
- Подписка (`start_with_next`) активирует всю цепочку
- Все работает лениво — вычисления начинаются только при подписке
- Move-семантика важна для эффективности

Это позволяет писать декларативный код, похожий на функциональное программирование.