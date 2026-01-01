# Розділ 18: Обробка помилок — Option та Result

## Вступ

У попередніх розділах ваш агент працював в ідеальному світі. Сенсори завжди відповідали. Координати завжди були валідними. Команди завжди виконувались успішно. Але реальний світ — інший.

Уявіть ситуацію: БПЛА летить над територією. Раптом GPS-сигнал зникає через перешкоди. Сенсор батареї повертає некоректні дані. Команда з центру управління приходить пошкодженою. Що має робити програма?

У багатьох мовах програмування для таких ситуацій використовують **exceptions** — механізм, де помилка "викидається" в одному місці і "ловиться" десь вище по стеку викликів. Проблема в тому, що exceptions невидимі: дивлячись на сигнатуру функції, ви не знаєте, які помилки вона може кинути. Легко забути обробити помилку, і програма впаде в найневідповідніший момент.

Rust обрав радикально інший підхід: **помилки — це звичайні значення**. Функція, що може провалитись, повертає `Result<T, E>`. Функція, що може не мати значення, повертає `Option<T>`. Компілятор бачить ці типи і **змушує** вас обробити всі варіанти. Неможливо "забути" перевірити помилку — код просто не скомпілюється.

---

## 18.1 Філософія обробки помилок у Rust

### Два типи "невдач"

Rust розрізняє два принципово різних типи ситуацій:

**Відсутність значення** — це не помилка, це нормальна ситуація:
- Пошук у словнику не знайшов ключа
- Перший елемент порожнього вектора
- Користувач не ввів опціональне поле

Для цього існує `Option<T>` — або є значення (`Some(T)`), або немає (`None`).

**Операція провалилась** — щось пішло не так:
- Файл не вдалося відкрити
- Мережевий запит не пройшов
- Дані не пройшли валідацію

Для цього існує `Result<T, E>` — або успіх (`Ok(T)`), або помилка (`Err(E)`).

### Чому не exceptions?

Порівняймо два підходи:

```java
// Java з exceptions — проблеми приховані
String readFile(String path) throws IOException {
    // Може кинути IOException, але це легко забути обробити
    return Files.readString(Path.of(path));
}

void process() {
    String content = readFile("data.txt");  // Компілятор НЕ змушує обробляти
    // Якщо файл не існує — програма впаде
}
```

```rust
// Rust з Result — проблеми явні
fn read_file(path: &str) -> Result<String, std::io::Error> {
    std::fs::read_to_string(path)
}

fn process() {
    let content = read_file("data.txt");  // content: Result<String, Error>
    // Не можна просто використати content як String!
    // Компілятор ЗМУШУЄ обробити Result
}
```

У Rust помилка — частина типу. Ви не можете її проігнорувати.

### Чому не null?

Багато мов використовують `null` для відсутності значення:

```java
// Java з null — NullPointerException чекає
String getName() {
    return null;  // Може повернути null
}

void greet() {
    String name = getName();
    System.out.println("Hello, " + name.toUpperCase());  // BOOM!
}
```

Проблема `null` — він "маскується" під звичайне значення. Компілятор не бачить різниці між `String` та "можливо String, а можливо null". Rust вирішує це через `Option<T>`:

```rust
fn get_name() -> Option<String> {
    None  // Явно кажемо: може не бути значення
}

fn greet() {
    let name = get_name();  // name: Option<String>
    // name.to_uppercase();  // Помилка компіляції! Option не має методу to_uppercase
    
    // Компілятор змушує обробити обидва варіанти:
    match name {
        Some(n) => println!("Hello, {}", n.to_uppercase()),
        None => println!("Hello, stranger"),
    }
}
```

---

## 18.2 Option — коли значення може бути відсутнім

### Визначення Option

`Option<T>` — це enum з двома варіантами:

```rust
// Визначення в стандартній бібліотеці (спрощено)
enum Option<T> {
    Some(T),  // Є значення типу T
    None,     // Немає значення
}
```

`T` — це параметр типу (generic). `Option<i32>` містить або `Some(42)`, або `None`. `Option<String>` містить або `Some(String)`, або `None`.

### Коли використовувати Option

Option підходить для ситуацій, де відсутність — **нормальна** і **очікувана**:

- Пошук елемента в колекції: `vec.get(index)` повертає `Option<&T>`
- Опціональні поля структури: `struct User { middle_name: Option<String> }`
- Перший/останній елемент: `vec.first()` повертає `Option<&T>`
- Парсинг з можливою невдачею: `str.parse::<i32>()` — краще Result, але концепція схожа

### Створення Option

Наступна програма демонструє різні способи створення значень типу Option. Ми побачимо явне створення, виведення типу компілятором, та отримання Option з методів стандартної бібліотеки.

```rust
fn main() {
    // Спосіб 1: явне створення з анотацією типу
    let some_number: Option<i32> = Some(42);
    let no_number: Option<i32> = None;
    
    // Спосіб 2: компілятор виводить тип з значення
    let some_string = Some("hello");  // Option<&str>
    let some_float = Some(3.14);       // Option<f64>
    
    // Спосіб 3: для None завжди потрібна анотація
    // (компілятор не знає, Option якого типу)
    let nothing: Option<String> = None;
    // let ambiguous = None;  // Помилка: тип невідомий
    
    // Спосіб 4: Option з методів колекцій
    let numbers = vec![1, 2, 3];
    let first = numbers.first();       // Option<&i32> = Some(&1)
    let tenth = numbers.get(10);       // Option<&i32> = None
    
    let empty: Vec<i32> = vec![];
    let empty_first = empty.first();   // Option<&i32> = None
    
    println!("some_number: {:?}", some_number);  // Some(42)
    println!("no_number: {:?}", no_number);      // None
    println!("first: {:?}", first);              // Some(1)
    println!("tenth: {:?}", tenth);              // None
}
```

**Як це працює:**

- `Some(42)` створює Option, що містить значення 42
- `None` представляє відсутність значення
- Для `None` компілятор потребує анотації типу, бо не може вивести тип "порожнечі"
- Методи `first()`, `get()`, `last()` повертають `Option`, бо колекція може бути порожньою або індекс невалідним

### Базова обробка Option через match

Найнадійніший спосіб обробити Option — pattern matching через `match`. Ця конструкція змушує обробити **обидва** варіанти.

Наступна програма демонструє пошук агента за ID. Функція повертає Option, і ми обробляємо обидва випадки — коли агент знайдений і коли ні.

```rust
/// Шукає агента за ідентифікатором.
/// Повертає Some(name) якщо знайдено, None якщо ні.
fn find_agent(id: &str) -> Option<String> {
    match id {
        "001" => Some(String::from("Scout Alpha")),
        "002" => Some(String::from("Scout Beta")),
        "003" => Some(String::from("Combat Gamma")),
        _ => None,  // Невідомий ID
    }
}

fn main() {
    // Спосіб 1: повний match — обробляємо обидва варіанти
    let agent = find_agent("001");
    match agent {
        Some(name) => println!("Знайдено агента: {}", name),
        None => println!("Агента не знайдено"),
    }
    
    // Спосіб 2: if let — коли цікавить тільки Some
    if let Some(name) = find_agent("002") {
        println!("Агент 002: {}", name);
    }
    // Якщо None — нічого не робимо
    
    // Спосіб 3: if let з else
    if let Some(name) = find_agent("999") {
        println!("Знайдено: {}", name);
    } else {
        println!("Агент 999 не існує");
    }
    
    // Спосіб 4: let else (Rust 1.65+) — ранній вихід
    let Some(name) = find_agent("003") else {
        println!("Агента не знайдено, завершуємо");
        return;
    };
    // Тут name: String (вже розпаковано)
    println!("Працюємо з агентом: {}", name);
}
```

**Як це працює:**

- `match` вимагає обробити **всі** варіанти enum — компілятор перевірить
- `if let Some(x) = ...` — зручний скорочений синтаксис, коли потрібен тільки `Some`
- `let else` — елегантний патерн для раннього виходу з функції при `None`
- Після `let Some(name) = ... else { return; }` змінна `name` вже розпакована і має тип `String`, не `Option<String>`

---

## 18.3 Методи Option для безпечного витягування

### Небезпечні методи: unwrap та expect

Іноді ви на 100% впевнені, що Option містить значення. Для таких випадків є `unwrap()` — він витягує значення або **панікує** якщо None.

```rust
fn main() {
    let definitely_exists = Some(42);
    let value = definitely_exists.unwrap();  // 42
    println!("Value: {}", value);
    
    // НЕБЕЗПЕКА: unwrap на None — паніка!
    let empty: Option<i32> = None;
    // let crash = empty.unwrap();  // panic: called `Option::unwrap()` on a `None` value
    
    // expect — як unwrap, але з власним повідомленням
    let also_exists = Some("hello");
    let text = also_exists.expect("Text should be present");
    println!("Text: {}", text);
    
    // НЕБЕЗПЕКА: expect на None — паніка з вашим повідомленням
    // let crash = empty.expect("This will be in the panic message");
}
```

**Коли використовувати unwrap/expect:**

- У тестах (паніка = провалений тест)
- У прототипах (швидкість розробки важливіша)
- Коли ви **доказово** знаєте, що значення є (наприклад, після перевірки `is_some()`)
- У документації як приклад (але з поясненням)

**Коли НЕ використовувати:**

- У production коді
- Коли Option приходить ззовні (користувач, мережа, файл)
- Коли є хоч найменший шанс на None

### Безпечні методи: unwrap_or, unwrap_or_else, unwrap_or_default

Ці методи витягують значення або повертають альтернативу — **без паніки**.

Наступна програма демонструє три способи безпечно отримати значення з Option, надаючи значення за замовчуванням.

```rust
fn main() {
    let some_value = Some(42);
    let no_value: Option<i32> = None;
    
    // unwrap_or — повертає вказане значення якщо None
    let a = some_value.unwrap_or(0);   // 42 (є значення)
    let b = no_value.unwrap_or(0);     // 0  (немає, беремо default)
    println!("unwrap_or: a={}, b={}", a, b);
    
    // unwrap_or_else — ліниве обчислення default
    // Функція викликається ТІЛЬКИ якщо Option = None
    let c = no_value.unwrap_or_else(|| {
        println!("Обчислюємо default...");
        expensive_computation()
    });
    println!("unwrap_or_else: c={}", c);
    
    // unwrap_or_default — використовує Default trait
    // String::default() = "", i32::default() = 0, Vec::default() = []
    let empty_string: Option<String> = None;
    let s = empty_string.unwrap_or_default();  // ""
    println!("unwrap_or_default: '{}'", s);
    
    let empty_number: Option<i32> = None;
    let n = empty_number.unwrap_or_default();  // 0
    println!("unwrap_or_default: {}", n);
}

fn expensive_computation() -> i32 {
    // Уявіть тут складний розрахунок
    -1
}
```

**Як це працює:**

- `unwrap_or(default)` — завжди обчислює `default`, навіть якщо є `Some`
- `unwrap_or_else(|| ...)` — обчислює замикання **тільки при None** (лінива оцінка)
- `unwrap_or_default()` — використовує стандартне значення типу (тип повинен реалізувати `Default`)

**Коли що використовувати:**

- `unwrap_or(константа)` — коли default простий і дешевий
- `unwrap_or_else(|| ...)` — коли обчислення default затратне
- `unwrap_or_default()` — коли стандартне значення типу підходить

---

## 18.4 Трансформація Option: map і and_then

### map — трансформація внутрішнього значення

Часто потрібно не просто витягти значення, а **трансформувати** його. Метод `map` застосовує функцію до значення всередині `Some`, а `None` залишає без змін.

Уявіть: у вас `Option<String>` (можливо є рядок), і ви хочете отримати `Option<usize>` (можливо є довжина). Без `map` довелося б писати match. З `map` — один рядок.

```rust
fn main() {
    let some_string: Option<String> = Some(String::from("Hello"));
    let no_string: Option<String> = None;
    
    // map трансформує Some, None залишає None
    let some_length: Option<usize> = some_string.as_ref().map(|s| s.len());
    let no_length: Option<usize> = no_string.as_ref().map(|s| s.len());
    
    println!("some_length: {:?}", some_length);  // Some(5)
    println!("no_length: {:?}", no_length);      // None
    
    // Ланцюжок map — послідовні трансформації
    let result = Some(2)
        .map(|x| x * 3)    // Some(6)
        .map(|x| x + 1)    // Some(7)
        .map(|x| x * 2);   // Some(14)
    
    println!("chain result: {:?}", result);  // Some(14)
    
    // Якщо десь None — весь ланцюжок повертає None
    let with_none: Option<i32> = None;
    let chain_none = with_none
        .map(|x| x * 3)
        .map(|x| x + 1);
    
    println!("chain with None: {:?}", chain_none);  // None
}
```

**Як це працює:**

- `option.map(f)` повертає `Some(f(x))` якщо `option = Some(x)`
- `option.map(f)` повертає `None` якщо `option = None`
- Функція `f` ніколи не викликається для `None`
- Ланцюжок `map` зупиняється на першому `None`

### and_then — коли трансформація теж повертає Option

Що якщо функція трансформації сама повертає `Option`? Наприклад, парсинг числа з рядка: `"42"` → `Some(42)`, `"abc"` → `None`.

Якщо використати `map`, отримаємо `Option<Option<i32>>` — вкладені Option. Це незручно. Метод `and_then` "сплющує" результат.

```rust
fn main() {
    // Функція, що повертає Option
    fn parse_positive(s: &str) -> Option<i32> {
        let n: i32 = s.parse().ok()?;  // Парсимо, при помилці — None
        if n > 0 { Some(n) } else { None }  // Тільки позитивні
    }
    
    let input = Some("42");
    
    // map дав би Option<Option<i32>> — вкладені!
    // let nested = input.map(parse_positive);  // Some(Some(42))
    
    // and_then "сплющує" результат
    let flat = input.and_then(parse_positive);  // Some(42)
    println!("and_then result: {:?}", flat);
    
    // Ланцюжок and_then для послідовних fallible операцій
    fn double_if_small(x: i32) -> Option<i32> {
        if x < 100 { Some(x * 2) } else { None }
    }
    
    let chain = Some(10)
        .and_then(double_if_small)   // Some(20)
        .and_then(double_if_small)   // Some(40)
        .and_then(double_if_small);  // Some(80)
    
    println!("chain: {:?}", chain);  // Some(80)
    
    // Якщо перевищимо 100 — ланцюжок обірветься
    let overflow = Some(30)
        .and_then(double_if_small)   // Some(60)
        .and_then(double_if_small);  // None (60*2=120 > 100)
    
    println!("overflow: {:?}", overflow);  // None
}
```

**Різниця між map та and_then:**

| Метод | Функція повертає | Результат |
|-------|-----------------|-----------|
| `map(f)` | `T → U` | `Option<U>` |
| `and_then(f)` | `T → Option<U>` | `Option<U>` |

Мнемоніка: `and_then` — "і потім" — виконує наступний крок **тільки якщо** попередній успішний.

---

## 18.5 Інші корисні методи Option

### filter — умовне збереження

Метод `filter` залишає `Some` тільки якщо значення задовольняє умову:

```rust
fn main() {
    // filter залишає Some тільки якщо предикат true
    let even = Some(4).filter(|&x| x % 2 == 0);
    let odd = Some(3).filter(|&x| x % 2 == 0);
    let none: Option<i32> = None;
    let none_filtered = none.filter(|&x| x % 2 == 0);
    
    println!("even: {:?}", even);           // Some(4)
    println!("odd filtered: {:?}", odd);    // None (3 не парне)
    println!("none filtered: {:?}", none_filtered);  // None
    
    // Практичний приклад: валідація
    fn get_battery_level() -> Option<u8> {
        Some(75)  // Симуляція
    }
    
    let safe_battery = get_battery_level()
        .filter(|&level| level > 20);  // Тільки якщо > 20%
    
    match safe_battery {
        Some(level) => println!("Батарея достатня: {}%", level),
        None => println!("Батарея критична або недоступна"),
    }
}
```

### is_some, is_none — перевірка стану

```rust
fn main() {
    let some_value: Option<i32> = Some(42);
    let no_value: Option<i32> = None;
    
    println!("some_value.is_some(): {}", some_value.is_some());  // true
    println!("some_value.is_none(): {}", some_value.is_none());  // false
    println!("no_value.is_some(): {}", no_value.is_some());      // false
    println!("no_value.is_none(): {}", no_value.is_none());      // true
    
    // Використання в умовах
    if some_value.is_some() {
        println!("Значення присутнє");
    }
    
    // is_some_and (Rust 1.70+) — перевірка з додатковою умовою
    let big = Some(100);
    let is_big = big.is_some_and(|x| x > 50);
    println!("is_big: {}", is_big);  // true
}
```

### or, and — комбінування Option

```rust
fn main() {
    let a: Option<i32> = Some(2);
    let b: Option<i32> = Some(3);
    let none: Option<i32> = None;
    
    // or — повертає перший Some
    println!("a.or(b): {:?}", a.or(b));       // Some(2) — a є
    println!("none.or(b): {:?}", none.or(b)); // Some(3) — a немає, беремо b
    println!("none.or(none): {:?}", none.or(none));  // None — обидва None
    
    // and — повертає другий, якщо перший Some
    println!("a.and(b): {:?}", a.and(b));     // Some(3) — a є, повертаємо b
    println!("none.and(b): {:?}", none.and(b)); // None — a немає
    
    // Практичне застосування: fallback значення
    fn get_from_cache() -> Option<String> { None }
    fn get_from_database() -> Option<String> { Some("data".to_string()) }
    
    let value = get_from_cache().or_else(|| get_from_database());
    println!("value: {:?}", value);  // Some("data")
}
```

### zip — комбінування двох Option

```rust
fn main() {
    let x = Some(1);
    let y = Some("hello");
    let none: Option<i32> = None;
    
    // zip — комбінує два Option в Option<(T, U)>
    let zipped = x.zip(y);
    println!("x.zip(y): {:?}", zipped);  // Some((1, "hello"))
    
    // Якщо хоча б один None — результат None
    let with_none = x.zip(none);
    println!("x.zip(none): {:?}", with_none);  // None
    
    // Практичне застосування: обидва значення потрібні
    fn get_latitude() -> Option<f64> { Some(50.45) }
    fn get_longitude() -> Option<f64> { Some(30.52) }
    
    if let Some((lat, lon)) = get_latitude().zip(get_longitude()) {
        println!("Координати: ({}, {})", lat, lon);
    } else {
        println!("Координати недоступні");
    }
}
```

---

## 18.6 Result — коли операція може провалитись

### Визначення Result

`Result<T, E>` — enum для операцій, що можуть завершитись успіхом або помилкою:

```rust
// Визначення в стандартній бібліотеці (спрощено)
enum Result<T, E> {
    Ok(T),   // Успіх з результатом типу T
    Err(E),  // Помилка типу E
}
```

Ключова відмінність від Option: Result має **два** параметри типу — тип успішного результату `T` і тип помилки `E`.

### Коли використовувати Result

Result підходить для операцій, де невдача — **помилка**, яку треба обробити:

- Читання файлу: `std::fs::read_to_string()` → `Result<String, io::Error>`
- Мережевий запит: може бути timeout, connection refused
- Парсинг даних: невалідний формат
- Валідація: дані не проходять перевірку

### Створення та обробка Result

Наступна програма демонструє функцію ділення, яка може провалитись (ділення на нуль), та різні способи обробки Result.

```rust
/// Безпечне ділення. Повертає помилку при діленні на нуль.
fn divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        Err(String::from("Ділення на нуль неможливе"))
    } else {
        Ok(a / b)
    }
}

fn main() {
    // Успішний результат
    let success = divide(10.0, 2.0);
    println!("10 / 2 = {:?}", success);  // Ok(5.0)
    
    // Помилка
    let failure = divide(10.0, 0.0);
    println!("10 / 0 = {:?}", failure);  // Err("Ділення на нуль неможливе")
    
    // Обробка через match — найнадійніший спосіб
    match divide(20.0, 4.0) {
        Ok(result) => println!("Результат: {}", result),
        Err(error) => println!("Помилка: {}", error),
    }
    
    // if let — коли цікавить тільки успіх
    if let Ok(result) = divide(15.0, 3.0) {
        println!("Успішно: {}", result);
    }
    
    // if let з else — обробка обох випадків
    if let Err(e) = divide(5.0, 0.0) {
        println!("Операція провалилась: {}", e);
    }
}
```

**Як це працює:**

- `Ok(value)` — створює успішний Result
- `Err(error)` — створює Result з помилкою
- `match` змушує обробити **обидва** варіанти
- Компілятор не дозволить використати `Result<f64, String>` як `f64` напряму

---

## 18.7 Методи Result

### Перевірка стану

```rust
fn main() {
    let ok: Result<i32, &str> = Ok(42);
    let err: Result<i32, &str> = Err("помилка");
    
    println!("ok.is_ok(): {}", ok.is_ok());    // true
    println!("ok.is_err(): {}", ok.is_err());  // false
    println!("err.is_ok(): {}", err.is_ok());  // false
    println!("err.is_err(): {}", err.is_err()); // true
}
```

### Витягування значення

```rust
fn main() {
    let ok: Result<i32, &str> = Ok(42);
    let err: Result<i32, &str> = Err("error");
    
    // unwrap / expect — ПАНІКА при Err
    println!("ok.unwrap(): {}", ok.unwrap());  // 42
    // err.unwrap();  // ПАНІКА!
    
    // Безпечні альтернативи:
    
    // unwrap_or — значення за замовчуванням
    println!("err.unwrap_or(0): {}", err.unwrap_or(0));  // 0
    
    // unwrap_or_else — ліниве обчислення + доступ до помилки
    let result = err.unwrap_or_else(|e| {
        println!("Помилка: {}, використовуємо default", e);
        -1
    });
    println!("result: {}", result);  // -1
    
    // unwrap_or_default — Default::default()
    let err_string: Result<String, &str> = Err("error");
    println!("default: '{}'", err_string.unwrap_or_default());  // ""
    
    // unwrap_err — витягує помилку (паніка при Ok)
    let error_value = err.unwrap_err();
    println!("error_value: {}", error_value);  // "error"
}
```

### map і map_err — трансформація

`map` трансформує успішне значення, `map_err` трансформує помилку:

```rust
fn main() {
    let ok: Result<i32, &str> = Ok(2);
    let err: Result<i32, &str> = Err("original error");
    
    // map — трансформує Ok, залишає Err без змін
    let doubled = ok.map(|x| x * 2);
    println!("doubled: {:?}", doubled);  // Ok(4)
    
    let err_mapped = err.map(|x| x * 2);
    println!("err_mapped: {:?}", err_mapped);  // Err("original error")
    
    // map_err — трансформує Err, залишає Ok без змін
    let wrapped_err = err.map_err(|e| format!("Wrapped: {}", e));
    println!("wrapped_err: {:?}", wrapped_err);  // Err("Wrapped: original error")
    
    // Комбінування map та map_err
    let processed = ok
        .map(|x| x * 10)
        .map_err(|e| format!("Error: {}", e));
    println!("processed: {:?}", processed);  // Ok(20)
}
```

### and_then — ланцюжок fallible операцій

Це найважливіший метод для композиції операцій, що можуть провалитись.

```rust
fn main() {
    // Функція, що може провалитись
    fn try_double(x: i32) -> Result<i32, &'static str> {
        if x < 100 {
            Ok(x * 2)
        } else {
            Err("Занадто велике число")
        }
    }
    
    // Ланцюжок and_then — кожен крок може провалитись
    let result = Ok(10)
        .and_then(try_double)   // Ok(20)
        .and_then(try_double)   // Ok(40)
        .and_then(try_double);  // Ok(80)
    
    println!("Успішний ланцюжок: {:?}", result);  // Ok(80)
    
    // Якщо один крок провалюється — весь ланцюжок провалюється
    let overflow = Ok(30)
        .and_then(try_double)   // Ok(60)
        .and_then(try_double);  // Err (60*2=120 > 100)
    
    println!("Провалений ланцюжок: {:?}", overflow);  // Err("Занадто велике число")
    
    // Err на вході — ланцюжок навіть не починається
    let from_err: Result<i32, &str> = Err("Початкова помилка");
    let chain_err = from_err
        .and_then(try_double)
        .and_then(try_double);
    
    println!("З помилкою на вході: {:?}", chain_err);  // Err("Початкова помилка")
}
```

---

## 18.8 Конвертація між Option та Result

Часто потрібно перетворити Option на Result або навпаки.

### Option → Result: ok_or та ok_or_else

```rust
fn main() {
    let some_value: Option<i32> = Some(42);
    let no_value: Option<i32> = None;
    
    // ok_or — перетворює None на вказану помилку
    let result_ok: Result<i32, &str> = some_value.ok_or("Значення відсутнє");
    let result_err: Result<i32, &str> = no_value.ok_or("Значення відсутнє");
    
    println!("some.ok_or: {:?}", result_ok);   // Ok(42)
    println!("none.ok_or: {:?}", result_err);  // Err("Значення відсутнє")
    
    // ok_or_else — ліниве створення помилки
    let result = no_value.ok_or_else(|| {
        format!("Помилка о {}", get_timestamp())
    });
    println!("ok_or_else: {:?}", result);
}

fn get_timestamp() -> u64 { 12345 }
```

### Result → Option: ok та err

```rust
fn main() {
    let ok: Result<i32, &str> = Ok(42);
    let err: Result<i32, &str> = Err("помилка");
    
    // ok() — перетворює Ok на Some, Err на None
    let opt_ok: Option<i32> = ok.ok();
    let opt_err: Option<i32> = err.ok();
    
    println!("ok.ok(): {:?}", opt_ok);    // Some(42)
    println!("err.ok(): {:?}", opt_err);  // None (помилка втрачена!)
    
    // err() — перетворює Err на Some, Ok на None
    let err_opt: Option<&str> = err.err();
    println!("err.err(): {:?}", err_opt); // Some("помилка")
}
```

**Важливо:** `ok()` **відкидає** інформацію про помилку. Використовуйте обережно!

---

## 18.9 Оператор ? — елегантне розповсюдження помилок

Оператор `?` — синтаксичний цукор для раннього повернення при помилці. Замість verbose match, один символ.

```rust
fn read_battery() -> Result<u8, String> {
    Ok(75)  // Симуляція
}

fn read_altitude() -> Result<f64, String> {
    Ok(100.0)  // Симуляція
}

fn validate_flight() -> Result<(), String> {
    // Без оператора ? — громіздко
    // let battery = match read_battery() {
    //     Ok(b) => b,
    //     Err(e) => return Err(e),
    // };
    
    // З оператором ? — елегантно
    let battery = read_battery()?;  // При Err — повертає Err з функції
    let altitude = read_altitude()?;
    
    if battery < 20 {
        return Err(String::from("Батарея занадто низька"));
    }
    
    println!("Battery: {}%, Altitude: {}m", battery, altitude);
    Ok(())
}

fn main() {
    match validate_flight() {
        Ok(()) => println!("Політ дозволено"),
        Err(e) => println!("Помилка: {}", e),
    }
}
```

**Як працює `?`:**

1. Якщо значення `Ok(x)` — розпаковує і повертає `x`
2. Якщо значення `Err(e)` — одразу повертає `Err(e)` з поточної функції

**Вимоги:**

- Функція повинна повертати `Result` (або `Option` для Option)
- Тип помилки повинен бути сумісним (або конвертуватись через `From`)

---

## 18.10 Практичне застосування: безпечний агент

Тепер об'єднаємо все вивчене у практичному прикладі. Ця програма демонструє агента БПЛА, який використовує Option для опціональних даних та Result для операцій, що можуть провалитись.

**Постановка задачі:** Створити агента, який:
- Має опціональні сенсори (можуть бути не встановлені)
- Перевіряє умови перед виконанням команд
- Обробляє помилки gracefully, без паніки
- Використовує ланцюжки операцій

```rust
/// Позиція агента в просторі
#[derive(Debug, Clone)]
pub struct Position {
    pub x: f64,
    pub y: f64,
    pub z: f64,
}

impl Position {
    pub fn origin() -> Self {
        Self { x: 0.0, y: 0.0, z: 0.0 }
    }
    
    pub fn new(x: f64, y: f64, z: f64) -> Self {
        Self { x, y, z }
    }
}

/// Помилки агента
#[derive(Debug)]
pub enum AgentError {
    NotOperational(String),
    InvalidCommand(String),
    SensorUnavailable(String),
    BatteryLow(u8),
}

/// Стан агента
#[derive(Debug, Clone, PartialEq)]
pub enum AgentState {
    Grounded,
    Flying,
    Emergency(String),
}

/// Показання сенсора
#[derive(Debug, Clone)]
pub struct SensorReading {
    pub value: f64,
    pub quality: f64,  // 0.0-1.0
}

/// Безпечний агент з обробкою помилок
pub struct SafeAgent {
    id: String,
    position: Position,
    battery: u8,
    state: AgentState,
    // Опціональні сенсори — можуть бути не встановлені
    gps_sensor: Option<String>,
    altimeter: Option<String>,
}

impl SafeAgent {
    pub fn new(id: &str) -> Self {
        Self {
            id: id.to_string(),
            position: Position::origin(),
            battery: 100,
            state: AgentState::Grounded,
            gps_sensor: None,
            altimeter: None,
        }
    }
    
    /// Встановлює GPS сенсор
    pub fn with_gps(mut self, sensor_id: &str) -> Self {
        self.gps_sensor = Some(sensor_id.to_string());
        self
    }
    
    /// Встановлює висотомір
    pub fn with_altimeter(mut self, sensor_id: &str) -> Self {
        self.altimeter = Some(sensor_id.to_string());
        self
    }
    
    // === Методи з Option — опціональні дані ===
    
    /// Повертає позицію тільки якщо агент операційний
    pub fn position(&self) -> Option<&Position> {
        if self.is_operational() {
            Some(&self.position)
        } else {
            None
        }
    }
    
    /// Читає GPS (Option<Result> — сенсор може бути відсутній, 
    /// а якщо є — читання може провалитись)
    pub fn read_gps(&self) -> Option<Result<SensorReading, AgentError>> {
        // Якщо сенсора немає — повертаємо None
        self.gps_sensor.as_ref().map(|_sensor_id| {
            // Симуляція читання — може провалитись
            if self.battery > 10 {
                Ok(SensorReading {
                    value: self.position.x,
                    quality: 0.95,
                })
            } else {
                Err(AgentError::BatteryLow(self.battery))
            }
        })
    }
    
    /// Читає висотомір з fallback значенням
    pub fn read_altitude_or(&self, default: f64) -> f64 {
        self.altimeter
            .as_ref()
            .and_then(|_| {
                // Симуляція успішного читання
                Some(self.position.z)
            })
            .unwrap_or(default)
    }
    
    // === Методи з Result — операції, що можуть провалитись ===
    
    /// Перевіряє, чи агент може працювати
    pub fn is_operational(&self) -> bool {
        !matches!(self.state, AgentState::Emergency(_)) && self.battery > 10
    }
    
    /// Перевіряє готовність до операції
    fn check_operational(&self) -> Result<(), AgentError> {
        if !self.is_operational() {
            return Err(AgentError::NotOperational(
                format!("Agent {} is not operational", self.id)
            ));
        }
        Ok(())
    }
    
    /// Перевіряє достатність батареї
    fn check_battery(&self, required: u8) -> Result<(), AgentError> {
        if self.battery < required {
            return Err(AgentError::BatteryLow(self.battery));
        }
        Ok(())
    }
    
    /// Зліт — може провалитись з різних причин
    pub fn take_off(&mut self, altitude: f64) -> Result<(), AgentError> {
        // Ланцюжок перевірок з ?
        self.check_operational()?;
        self.check_battery(20)?;
        
        if self.state != AgentState::Grounded {
            return Err(AgentError::InvalidCommand(
                "Можна злетіти тільки з землі".to_string()
            ));
        }
        
        if altitude <= 0.0 || altitude > 500.0 {
            return Err(AgentError::InvalidCommand(
                format!("Невалідна висота: {}", altitude)
            ));
        }
        
        // Виконуємо операцію
        self.position.z = altitude;
        self.state = AgentState::Flying;
        self.battery = self.battery.saturating_sub(5);
        
        Ok(())
    }
    
    /// Рух до позиції
    pub fn move_to(&mut self, target: Position) -> Result<Position, AgentError> {
        self.check_operational()?;
        self.check_battery(10)?;
        
        if self.state != AgentState::Flying {
            return Err(AgentError::InvalidCommand(
                "Для руху потрібно бути в повітрі".to_string()
            ));
        }
        
        self.position = target.clone();
        self.battery = self.battery.saturating_sub(3);
        
        Ok(target)
    }
    
    /// Посадка
    pub fn land(&mut self) -> Result<(), AgentError> {
        if self.state != AgentState::Flying {
            return Err(AgentError::InvalidCommand(
                "Можна сісти тільки з повітря".to_string()
            ));
        }
        
        self.position.z = 0.0;
        self.state = AgentState::Grounded;
        
        Ok(())
    }
    
    /// Виконання місії з кількома waypoints
    pub fn execute_mission(&mut self, waypoints: Vec<Position>) -> Result<usize, AgentError> {
        // Зліт
        self.take_off(50.0)?;
        
        let mut visited = 0;
        
        for waypoint in waypoints {
            // Перевірка батареї перед кожним рухом
            if self.battery < 20 {
                // Недостатньо — аварійне повернення
                println!("Батарея низька ({}%), повертаємось", self.battery);
                self.move_to(Position::origin())?;
                self.land()?;
                return Ok(visited);
            }
            
            self.move_to(waypoint)?;
            visited += 1;
        }
        
        // Успішне завершення
        self.move_to(Position::origin())?;
        self.land()?;
        
        Ok(visited)
    }
}

fn main() {
    println!("╔══════════════════════════════════════════════════════════════╗");
    println!("║                БЕЗПЕЧНИЙ АГЕНТ — ДЕМО                        ║");
    println!("╚══════════════════════════════════════════════════════════════╝\n");
    
    // Створюємо агента з опціональними сенсорами
    let mut agent = SafeAgent::new("SAFE-001")
        .with_gps("GPS-001")
        .with_altimeter("ALT-001");
    
    // === Демонстрація Option ===
    println!("=== Option Demo ===\n");
    
    // Позиція доступна, бо агент операційний
    match agent.position() {
        Some(pos) => println!("Позиція: ({}, {}, {})", pos.x, pos.y, pos.z),
        None => println!("Позиція недоступна"),
    }
    
    // GPS читання — Option<Result>
    match agent.read_gps() {
        Some(Ok(reading)) => println!("GPS: {} (якість: {})", reading.value, reading.quality),
        Some(Err(e)) => println!("GPS помилка: {:?}", e),
        None => println!("GPS сенсор не встановлено"),
    }
    
    // Висота з fallback
    let altitude = agent.read_altitude_or(0.0);
    println!("Висота (або 0): {}m", altitude);
    
    // === Демонстрація Result ===
    println!("\n=== Result Demo ===\n");
    
    // Успішний зліт
    print!("Зліт на 100м: ");
    match agent.take_off(100.0) {
        Ok(()) => println!("✅ Успішно"),
        Err(e) => println!("❌ {:?}", e),
    }
    
    // Спроба повторного зльоту — помилка
    print!("Повторний зліт: ");
    match agent.take_off(50.0) {
        Ok(()) => println!("✅ Успішно"),
        Err(e) => println!("❌ {:?}", e),
    }
    
    // Рух
    print!("Рух до (100, 50, 100): ");
    match agent.move_to(Position::new(100.0, 50.0, 100.0)) {
        Ok(pos) => println!("✅ Досягнуто {:?}", pos),
        Err(e) => println!("❌ {:?}", e),
    }
    
    // Посадка
    print!("Посадка: ");
    match agent.land() {
        Ok(()) => println!("✅ Успішно"),
        Err(e) => println!("❌ {:?}", e),
    }
    
    // === Демонстрація місії ===
    println!("\n=== Mission Demo ===\n");
    
    let mut mission_agent = SafeAgent::new("MISSION-001");
    let waypoints = vec![
        Position::new(100.0, 0.0, 50.0),
        Position::new(100.0, 100.0, 50.0),
        Position::new(0.0, 100.0, 50.0),
    ];
    
    match mission_agent.execute_mission(waypoints) {
        Ok(count) => println!("✅ Місія завершена, відвідано {} точок", count),
        Err(e) => println!("❌ Місія провалена: {:?}", e),
    }
}
```

**Як працює ця програма:**

1. **Option для опціональних даних:**
   - `gps_sensor: Option<String>` — сенсор може бути не встановлений
   - `position()` повертає `Option<&Position>` — позиція доступна тільки для операційного агента
   - `read_gps()` повертає `Option<Result<...>>` — вкладена структура для "може не бути сенсора" + "може провалитись читання"

2. **Result для fallible операцій:**
   - `take_off()`, `move_to()`, `land()` повертають `Result`
   - Оператор `?` для елегантного розповсюдження помилок
   - Різні типи помилок в `AgentError` enum

3. **Комбінування:**
   - `execute_mission()` — ланцюжок операцій з ранніми виходами
   - Перевірка умов перед кожною дією

---

## 18.11 Лабораторна робота

**Завдання:** Створіть систему валідації команд для агента.

**Структури:**

```rust
enum Command {
    MoveTo { x: f64, y: f64, z: f64 },
    TakeOff { altitude: f64 },
    Scan { radius: f64 },
    Land,
}

enum ValidationError {
    AltitudeTooHigh { max: f64, requested: f64 },
    AltitudeTooLow { min: f64, requested: f64 },
    OutOfBounds { coord: &'static str, value: f64 },
    RadiusTooLarge { max: f64, requested: f64 },
    InsufficientBattery { required: u8, available: u8 },
}

struct CommandValidator {
    max_altitude: f64,
    min_altitude: f64,
    max_x: f64,
    max_y: f64,
    max_scan_radius: f64,
}
```

**Методи для реалізації:**

1. `validate(&self, cmd: &Command, battery: u8) -> Result<(), ValidationError>`
2. `validate_altitude(&self, alt: f64) -> Result<(), ValidationError>`
3. `validate_position(&self, x: f64, y: f64, z: f64) -> Result<(), ValidationError>`
4. `validate_scan(&self, radius: f64, battery: u8) -> Result<(), ValidationError>`

**Критерії оцінювання:**

| Критерій | Бали |
|----------|------|
| Enum Command та ValidationError | 2 |
| CommandValidator структура | 1 |
| Метод validate_altitude | 2 |
| Метод validate_position | 2 |
| Метод validate_scan з перевіркою батареї | 2 |
| Тести | 1 |
| **Максимум** | **10** |

---

## Підсумок

`Option<T>` та `Result<T, E>` — фундамент надійного коду в Rust:

**Option:**
- Використовуйте для значень, які **можуть бути відсутніми**
- `Some(x)` — є значення, `None` — немає
- Методи: `unwrap_or`, `map`, `and_then`, `filter`

**Result:**
- Використовуйте для операцій, які **можуть провалитись**
- `Ok(x)` — успіх, `Err(e)` — помилка
- Методи: `unwrap_or`, `map`, `map_err`, `and_then`

**Ключові принципи:**
- Уникайте `unwrap()` у production коді
- Використовуйте `?` для розповсюдження помилок
- `map` — для трансформації значення
- `and_then` — для ланцюжка fallible операцій
- `ok_or` — для конвертації Option → Result

---

## Зв'язок з наступним розділом

Оператор `?` зручний, але що робити, коли функції повертають **різні** типи помилок? Як комбінувати `io::Error` від читання файлу з `ParseIntError` від парсингу числа?

У **Розділі 19: Оператор ? та ланцюжки помилок** ви навчитесь конвертувати між типами помилок і будувати складні ланцюжки операцій з різними джерелами помилок.
