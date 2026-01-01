# Розділ 19: Оператор ? та ланцюжки помилок

## Вступ

У попередньому розділі ви познайомились з `Option` та `Result`. Тепер уявіть функцію, що виконує місію агента БПЛА:

1. Перевірити preflight (може провалитись)
2. Зчитати GPS позицію (може провалитись)
3. Розрахувати маршрут (може провалитись)
4. Виконати політ (може провалитись)
5. Зчитати фінальну позицію (може провалитись)

Якщо кожен крок обробляти через `match`, код перетвориться на "сходинки" вкладених блоків — так зване **match-пекло**. П'ять рівнів вкладеності, де справжня логіка губиться серед `Ok(x) => x, Err(e) => return Err(e)`.

Оператор `?` — елегантне рішення цієї проблеми. Одна маленька позначка замінює весь match-блок: якщо значення `Ok` — розпаковує його, якщо `Err` — негайно повертає з функції.

Але `?` — це не просто синтаксичний цукор. Він також автоматично **конвертує типи помилок** через trait `From`. Це дозволяє будувати ланцюжки операцій, де кожен крок може мати свій тип помилки, а на виході — єдиний уніфікований тип.

---

## 19.1 Як працює оператор ?

### Базова механіка

Оператор `?` можна використовувати після будь-якого виразу типу `Result<T, E>`. Він робить одну з двох речей:

- Якщо значення `Ok(x)` — витягує `x` і виконання продовжується
- Якщо значення `Err(e)` — негайно повертає `Err(e)` з поточної функції

Ось проста демонстрація. Ця програма показує, як `?` автоматично розпаковує успішні результати або передає помилку вгору.

```rust
fn might_fail(succeed: bool) -> Result<i32, String> {
    if succeed {
        Ok(42)
    } else {
        Err(String::from("Операція провалилась"))
    }
}

fn use_result() -> Result<i32, String> {
    // Оператор ? розгортає Ok або повертає Err
    let value = might_fail(true)?;
    
    // Якщо might_fail повернув Ok(42), тут value = 42
    // Код продовжує виконуватись
    
    println!("Отримали значення: {}", value);
    Ok(value * 2)
}

fn use_result_fail() -> Result<i32, String> {
    let value = might_fail(false)?;
    
    // Цей код НІКОЛИ не виконається!
    // might_fail(false) повернув Err, тому ? вже повернув з функції
    println!("Цей рядок не буде надрукований");
    Ok(value * 2)
}

fn main() {
    println!("Успішний виклик:");
    match use_result() {
        Ok(v) => println!("Результат: {}", v),
        Err(e) => println!("Помилка: {}", e),
    }
    
    println!("\nПровалений виклик:");
    match use_result_fail() {
        Ok(v) => println!("Результат: {}", v),
        Err(e) => println!("Помилка: {}", e),
    }
}
```

**Як це працює:**

1. `might_fail(true)?` повертає `Ok(42)`, `?` розпаковує в `42`
2. `value` отримує значення `42`, виконання продовжується
3. `might_fail(false)?` повертає `Err(...)`, `?` одразу виконує `return Err(...)`
4. Весь код після `?` у провальному випадку не виконується

### Що ? робить "під капотом"

Оператор `?` — це синтаксичний цукор. Компілятор перетворює його на match-вираз:

```rust
// Цей код з ?:
fn with_question_mark() -> Result<i32, String> {
    let value = might_fail()?;
    Ok(value * 2)
}

// Компілятор перетворює на еквівалентний код:
fn without_question_mark() -> Result<i32, String> {
    let value = match might_fail() {
        Ok(v) => v,
        Err(e) => return Err(e.into()),  // Зверніть увагу на .into()!
    };
    Ok(value * 2)
}

fn might_fail() -> Result<i32, String> {
    Ok(42)
}
```

**Критична деталь:** компілятор додає `.into()` до помилки. Це означає автоматичну конвертацію типів через trait `From`. Ми розглянемо це детально пізніше.

### Обмеження: тип повернення функції

Оператор `?` можна використовувати **тільки** у функціях, що повертають `Result` або `Option`:

```rust
fn might_fail() -> Result<i32, String> {
    Ok(42)
}

// ❌ Помилка компіляції!
// fn main_wrong() {
//     let value = might_fail()?;  // main повертає (), не Result!
// }

// ✅ Правильно: main може повертати Result
fn main() -> Result<(), String> {
    let value = might_fail()?;
    println!("Значення: {}", value);
    Ok(())  // Потрібно повернути Ok в кінці
}
```

Якщо функція повертає `()` (unit), ви не можете використовувати `?`. Альтернативи:

1. Змінити тип повернення на `Result<(), Error>`
2. Обробити помилку через `match` або `if let`
3. Використати `.unwrap()` (тільки якщо впевнені, що помилки не буде)

---

## 19.2 Ланцюжки послідовних операцій

### Проблема без оператора ?

Уявіть функцію з кількома кроками, кожен з яких може провалитись. Без `?` код виглядає так:

```rust
fn step1() -> Result<i32, String> {
    Ok(10)
}

fn step2(input: i32) -> Result<i32, String> {
    if input > 0 { Ok(input * 2) } 
    else { Err("Від'ємне значення".to_string()) }
}

fn step3(input: i32) -> Result<String, String> {
    Ok(format!("Результат: {}", input))
}

// ❌ Match-пекло — 3 рівні вкладеності
fn pipeline_verbose() -> Result<String, String> {
    match step1() {
        Ok(a) => {
            match step2(a) {
                Ok(b) => {
                    match step3(b) {
                        Ok(c) => Ok(c),
                        Err(e) => Err(e),
                    }
                }
                Err(e) => Err(e),
            }
        }
        Err(e) => Err(e),
    }
}
```

Логіка проста — три послідовні кроки. Але код займає 15+ рядків і важко читається.

### Рішення з оператором ?

Та сама логіка з `?`:

```rust
fn step1() -> Result<i32, String> {
    println!("  Крок 1: Старт");
    Ok(10)
}

fn step2(input: i32) -> Result<i32, String> {
    println!("  Крок 2: Обробка {}", input);
    if input > 0 { 
        Ok(input * 2) 
    } else { 
        Err("Від'ємне значення".to_string()) 
    }
}

fn step3(input: i32) -> Result<String, String> {
    println!("  Крок 3: Фіналізація {}", input);
    Ok(format!("Результат: {}", input))
}

// ✅ Чистий код з ?
fn pipeline() -> Result<String, String> {
    let a = step1()?;   // Якщо Err — одразу повертає
    let b = step2(a)?;  // Якщо Err — одразу повертає
    let c = step3(b)?;  // Якщо Err — одразу повертає
    Ok(c)
}

fn main() {
    println!("Запуск pipeline:");
    match pipeline() {
        Ok(result) => println!("✅ Успіх: {}", result),
        Err(e) => println!("❌ Помилка: {}", e),
    }
}
```

**Переваги:**

- 4 рядки замість 15+
- Чітко видно послідовність кроків
- Логіка не губиться серед boilerplate
- Помилка з будь-якого кроку автоматично повертається

### Оператор ? у виразах

`?` можна використовувати прямо у виразах, не тільки як окремий statement:

```rust
fn get_a() -> Result<i32, &'static str> { Ok(10) }
fn get_b() -> Result<i32, &'static str> { Ok(20) }

fn calculate() -> Result<i32, &'static str> {
    // Обидва ? в одному виразі
    Ok(get_a()? + get_b()?)
}

fn get_length(s: Option<String>) -> Option<usize> {
    // ? працює і з Option
    Some(s?.len())
}

fn main() {
    println!("calculate: {:?}", calculate());  // Ok(30)
    
    println!("length Some: {:?}", get_length(Some("hello".to_string())));  // Some(5)
    println!("length None: {:?}", get_length(None));  // None
}
```

Якщо перший `?` провалюється, другий навіть не обчислюється — функція одразу повертає помилку.

---

## 19.3 Оператор ? з Option

`?` працює не тільки з `Result`, але й з `Option`. Логіка аналогічна:

- `Some(x)` → розпаковує в `x`
- `None` → одразу повертає `None` з функції

Ця програма демонструє пошук користувача та його email через ланцюжок Option.

```rust
fn find_user(id: u32) -> Option<String> {
    match id {
        1 => Some("Олена".to_string()),
        2 => Some("Петро".to_string()),
        _ => None,
    }
}

fn find_email(name: &str) -> Option<String> {
    match name {
        "Олена" => Some("olena@example.com".to_string()),
        "Петро" => Some("petro@example.com".to_string()),
        _ => None,
    }
}

fn get_user_email(id: u32) -> Option<String> {
    // Ланцюжок ? для Option
    let name = find_user(id)?;      // None → одразу повертає None
    let email = find_email(&name)?;  // None → одразу повертає None
    Some(email)
}

fn main() {
    println!("User 1 email: {:?}", get_user_email(1));  // Some("olena@example.com")
    println!("User 2 email: {:?}", get_user_email(2));  // Some("petro@example.com")
    println!("User 999 email: {:?}", get_user_email(999));  // None
}
```

### Навігація через вкладені Option

Особливо корисний `?` для навігації через структури з вкладеними Option:

```rust
struct Company {
    ceo: Option<Person>,
}

struct Person {
    name: String,
    contact: Option<Contact>,
}

struct Contact {
    email: Option<String>,
    phone: Option<String>,
}

// ❌ Без ? — жах вкладених match
fn get_ceo_email_verbose(company: &Company) -> Option<&String> {
    match &company.ceo {
        Some(person) => {
            match &person.contact {
                Some(contact) => {
                    match &contact.email {
                        Some(email) => Some(email),
                        None => None,
                    }
                }
                None => None,
            }
        }
        None => None,
    }
}

// ✅ З ? — один рядок
fn get_ceo_email(company: &Company) -> Option<&String> {
    company.ceo.as_ref()?.contact.as_ref()?.email.as_ref()
}

fn main() {
    let company = Company {
        ceo: Some(Person {
            name: "Марія".to_string(),
            contact: Some(Contact {
                email: Some("maria@corp.com".to_string()),
                phone: None,
            }),
        }),
    };
    
    println!("CEO email: {:?}", get_ceo_email(&company));
    
    let empty = Company { ceo: None };
    println!("Empty company: {:?}", get_ceo_email(&empty));
}
```

**Зверніть увагу на `as_ref()`:** без нього `?` спожив би `Option`, а нам потрібне лише посилання.

---

## 19.4 Проблема різних типів помилок

### Ситуація

Уявіть функцію, що читає число з файлу:

1. `File::open()` може повернути `io::Error`
2. `read_to_string()` може повернути `io::Error`
3. `parse()` може повернути `ParseIntError`

Це **різні типи помилок**. Як їх об'єднати?

```rust
use std::fs::File;
use std::io::Read;
use std::num::ParseIntError;

// ❌ Не компілюється! Який тип помилки?
// fn read_number(path: &str) -> Result<i32, ???> {
//     let mut file = File::open(path)?;        // io::Error
//     let mut content = String::new();
//     file.read_to_string(&mut content)?;       // io::Error
//     let number: i32 = content.trim().parse()?;  // ParseIntError
//     Ok(number)
// }
```

Компілятор скаржиться: тип помилки для `?` повинен бути однаковим або конвертуватись.

### Рішення 1: Box<dyn Error>

Найпростіше рішення — використати "коробку" для будь-якої помилки:

```rust
use std::fs::File;
use std::io::Read;
use std::error::Error;

// Box<dyn Error> — універсальний контейнер для будь-якої помилки
fn read_number(path: &str) -> Result<i32, Box<dyn Error>> {
    let mut file = File::open(path)?;
    let mut content = String::new();
    file.read_to_string(&mut content)?;
    let number: i32 = content.trim().parse()?;
    Ok(number)
}

fn main() {
    match read_number("number.txt") {
        Ok(n) => println!("Прочитане число: {}", n),
        Err(e) => println!("Помилка: {}", e),
    }
}
```

**Як це працює:**

- `Box<dyn Error>` — це "коробка" (heap-allocated) для будь-якого типу, що реалізує trait `Error`
- Стандартні типи помилок (`io::Error`, `ParseIntError`) реалізують `Error`
- `?` автоматично "пакує" помилку в `Box`

**Переваги:** простота, швидке прототипування
**Недоліки:** втрачається інформація про конкретний тип помилки, heap allocation

### Рішення 2: Власний тип помилки з From

Для production коду краще створити власний enum помилок:

```rust
use std::fs::File;
use std::io::{self, Read};
use std::num::ParseIntError;

/// Власний тип помилки — enum з варіантами для кожного джерела
#[derive(Debug)]
enum ReadError {
    Io(io::Error),
    Parse(ParseIntError),
}

// Реалізуємо From<io::Error> для ReadError
impl From<io::Error> for ReadError {
    fn from(err: io::Error) -> Self {
        ReadError::Io(err)
    }
}

// Реалізуємо From<ParseIntError> для ReadError
impl From<ParseIntError> for ReadError {
    fn from(err: ParseIntError) -> Self {
        ReadError::Parse(err)
    }
}

// Тепер ? автоматично конвертує помилки!
fn read_number(path: &str) -> Result<i32, ReadError> {
    let mut file = File::open(path)?;       // io::Error → ReadError::Io
    let mut content = String::new();
    file.read_to_string(&mut content)?;      // io::Error → ReadError::Io
    let number: i32 = content.trim().parse()?;  // ParseIntError → ReadError::Parse
    Ok(number)
}

fn main() {
    match read_number("number.txt") {
        Ok(n) => println!("Число: {}", n),
        Err(ReadError::Io(e)) => println!("Помилка IO: {}", e),
        Err(ReadError::Parse(e)) => println!("Помилка парсингу: {}", e),
    }
}
```

**Як це працює:**

1. Оператор `?` бачить, що повертаємо `Result<_, ReadError>`
2. При помилці `io::Error` він шукає `impl From<io::Error> for ReadError`
3. Знаходить і викликає `.into()` для конвертації
4. Те саме для `ParseIntError`

**Переваги:**
- Точна інформація про тип помилки
- Можливість pattern matching по варіантах
- Немає heap allocation

---

## 19.5 Trait From детальніше

### Що таке From

`From<T>` — це стандартний trait для конвертації типів:

```rust
// Визначення в стандартній бібліотеці
pub trait From<T>: Sized {
    fn from(value: T) -> Self;
}
```

Якщо тип `U` реалізує `From<T>`, це означає: "можна створити `U` з `T`".

### Into — зворотній trait

Коли ви реалізуєте `From<T> for U`, Rust автоматично дає вам `Into<U> for T`:

```rust
struct Celsius(f64);
struct Fahrenheit(f64);

// Реалізуємо From
impl From<Celsius> for Fahrenheit {
    fn from(c: Celsius) -> Self {
        Fahrenheit(c.0 * 9.0 / 5.0 + 32.0)
    }
}

fn main() {
    let c = Celsius(100.0);
    
    // Спосіб 1: явний виклик From
    let f1 = Fahrenheit::from(Celsius(100.0));
    
    // Спосіб 2: використання Into (автоматично доступний)
    let f2: Fahrenheit = Celsius(0.0).into();
    
    println!("100°C = {}°F", f1.0);  // 212
    println!("0°C = {}°F", f2.0);    // 32
}
```

### Як ? використовує From

Коли ви пишете `something()?`, компілятор генерує:

```rust
match something() {
    Ok(v) => v,
    Err(e) => return Err(e.into()),  // Виклик .into()
}
```

Тому якщо є `impl From<SourceError> for TargetError`, конвертація відбувається автоматично.

---

## 19.6 map_err для ручної конвертації

Іноді автоматична конвертація через `From` незручна або неможлива. Метод `map_err` дозволяє явно трансформувати помилку:

```rust
fn parse_positive(s: &str) -> Result<u32, String> {
    // Парсимо число, при помилці — форматуємо повідомлення
    s.parse::<u32>()
        .map_err(|e| format!("Не вдалося розпарсити '{}': {}", s, e))
}

fn validate_range(n: u32) -> Result<u32, String> {
    if n == 0 {
        Err("Значення не може бути нулем".to_string())
    } else if n > 100 {
        Err(format!("Значення {} перевищує максимум 100", n))
    } else {
        Ok(n)
    }
}

fn process(input: &str) -> Result<u32, String> {
    let number = parse_positive(input)?;
    let validated = validate_range(number)?;
    Ok(validated * 2)
}

fn main() {
    println!("42: {:?}", process("42"));       // Ok(84)
    println!("abc: {:?}", process("abc"));     // Err("Не вдалося розпарсити...")
    println!("0: {:?}", process("0"));         // Err("Значення не може бути нулем")
    println!("200: {:?}", process("200"));     // Err("Значення 200 перевищує...")
}
```

**Коли використовувати map_err:**

- Потрібне власне повідомлення про помилку
- Потрібно додати контекст (яке значення, де виникла помилка)
- Типи помилок несумісні і немає `From` реалізації

---

## 19.7 Конвертація Option → Result

Часто функція повертає `Option`, а вам потрібен `Result` для використання з `?`:

```rust
fn get_config_value(key: &str) -> Option<String> {
    match key {
        "host" => Some("localhost".to_string()),
        "port" => Some("8080".to_string()),
        _ => None,
    }
}

fn connect() -> Result<String, String> {
    // ok_or конвертує Option → Result
    let host = get_config_value("host")
        .ok_or("Відсутній ключ 'host'")?;
    
    // ok_or_else — ліниве обчислення повідомлення
    let port = get_config_value("port")
        .ok_or_else(|| format!("Відсутній ключ 'port'"))?;
    
    Ok(format!("{}:{}", host, port))
}

fn main() {
    match connect() {
        Ok(addr) => println!("З'єднуємось з {}", addr),
        Err(e) => println!("Помилка конфігурації: {}", e),
    }
}
```

**Різниця ok_or vs ok_or_else:**

- `ok_or(error)` — завжди створює помилку (навіть якщо Option = Some)
- `ok_or_else(|| error)` — створює помилку тільки при None (ефективніше)

---

## 19.8 Порівняння ? та and_then

Обидва підходи дозволяють будувати ланцюжки. Коли що використовувати?

```rust
fn validate(s: &str) -> Result<&str, &'static str> {
    if s.is_empty() { Err("Порожній рядок") } 
    else { Ok(s) }
}

fn parse_number(s: &str) -> Result<i32, &'static str> {
    s.parse().map_err(|_| "Невалідне число")
}

fn double_if_positive(n: i32) -> Result<i32, &'static str> {
    if n > 0 { Ok(n * 2) } 
    else { Err("Число має бути позитивним") }
}

// Спосіб 1: Послідовні ? — проміжні змінні
fn process_with_question(input: &str) -> Result<i32, &'static str> {
    let validated = validate(input)?;
    let number = parse_number(validated)?;
    let result = double_if_positive(number)?;
    Ok(result)
}

// Спосіб 2: and_then ланцюжок — функціональний стиль
fn process_with_and_then(input: &str) -> Result<i32, &'static str> {
    validate(input)
        .and_then(parse_number)
        .and_then(double_if_positive)
}

fn main() {
    let inputs = ["21", "", "-5", "abc"];
    
    for input in inputs {
        println!("Input '{}': {:?}", input, process_with_question(input));
    }
}
```

**Коли використовувати ?:**

- Потрібні проміжні змінні для логування/дебагу
- Між кроками є додаткова логіка
- Код читабельніший з явними іменами

**Коли використовувати and_then:**

- Простий pipeline без проміжної логіки
- Функціональний стиль
- Кожен крок — окрема функція

---

## 19.9 Збір помилок з колекції

Корисний патерн: обробка колекції елементів, де кожен може провалитись.

```rust
fn parse_numbers(inputs: Vec<&str>) -> Result<Vec<i32>, String> {
    inputs.iter()
        .map(|s| {
            s.parse::<i32>()
                .map_err(|e| format!("Помилка парсингу '{}': {}", s, e))
        })
        .collect()  // Магія! Result<Vec<i32>, String>
}

fn main() {
    let valid = vec!["1", "2", "3", "4", "5"];
    println!("Valid: {:?}", parse_numbers(valid));  // Ok([1, 2, 3, 4, 5])
    
    let invalid = vec!["1", "2", "abc", "4"];
    println!("Invalid: {:?}", parse_numbers(invalid));  // Err("Помилка парсингу 'abc'...")
}
```

**Як це працює:**

1. `map()` повертає ітератор `Iterator<Item = Result<i32, String>>`
2. `collect()` бачить, що хочемо `Result<Vec<i32>, String>`
3. Якщо всі елементи `Ok` — збирає в `Ok(Vec)`
4. Якщо хоч один `Err` — повертає перший `Err`

---

## 19.10 Практичне застосування: агент місії

Тепер об'єднаємо все вивчене у комплексному прикладі. Ця програма демонструє агента БПЛА, що виконує місію з кількома етапами. Кожен етап може провалитись зі своїм типом помилки, і всі помилки автоматично конвертуються через `From`.

**Постановка задачі:**

Агент має виконати місію:
1. Preflight перевірка (батарея, системи, погода)
2. Читання позиції GPS
3. Розрахунок маршруту до waypoints
4. Виконання польоту з перевіркою батареї
5. Читання фінальної позиції

Кожен компонент має власний тип помилки. Оператор `?` і реалізації `From` забезпечують чисту обробку.

```rust
use std::fmt;

// ═══════════════════════════════════════════════════════════════════════════
// ТИПИ ПОМИЛОК
// ═══════════════════════════════════════════════════════════════════════════

/// Головний тип помилки місії — агрегує всі можливі помилки
#[derive(Debug)]
pub enum MissionError {
    Preflight(PreflightError),
    Navigation(NavigationError),
    Sensor(SensorError),
    Execution(String),
}

#[derive(Debug)]
pub enum PreflightError {
    InsufficientBattery { required: u8, available: u8 },
    SystemNotReady(String),
    WeatherUnsafe,
}

#[derive(Debug)]
pub enum NavigationError {
    NoPath,
    OutOfBounds,
    ObstacleDetected,
}

#[derive(Debug)]
pub enum SensorError {
    Offline(String),
    InvalidReading,
    CalibrationRequired,
}

// Реалізації From — дозволяють ? автоматично конвертувати
impl From<PreflightError> for MissionError {
    fn from(e: PreflightError) -> Self {
        MissionError::Preflight(e)
    }
}

impl From<NavigationError> for MissionError {
    fn from(e: NavigationError) -> Self {
        MissionError::Navigation(e)
    }
}

impl From<SensorError> for MissionError {
    fn from(e: SensorError) -> Self {
        MissionError::Sensor(e)
    }
}

// Display для гарного виводу
impl fmt::Display for MissionError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            MissionError::Preflight(e) => write!(f, "Preflight: {:?}", e),
            MissionError::Navigation(e) => write!(f, "Navigation: {:?}", e),
            MissionError::Sensor(e) => write!(f, "Sensor: {:?}", e),
            MissionError::Execution(msg) => write!(f, "Execution: {}", msg),
        }
    }
}

// ═══════════════════════════════════════════════════════════════════════════
// КОМПОНЕНТИ АГЕНТА
// ═══════════════════════════════════════════════════════════════════════════

#[derive(Debug, Clone)]
pub struct Position {
    pub x: f64,
    pub y: f64,
    pub z: f64,
}

impl Position {
    pub fn new(x: f64, y: f64, z: f64) -> Self {
        Self { x, y, z }
    }
    
    pub fn origin() -> Self {
        Self::new(0.0, 0.0, 0.0)
    }
    
    pub fn distance_to(&self, other: &Position) -> f64 {
        let dx = self.x - other.x;
        let dy = self.y - other.y;
        let dz = self.z - other.z;
        (dx*dx + dy*dy + dz*dz).sqrt()
    }
}

/// Перевірка готовності до польоту
pub struct PreflightChecker {
    min_battery: u8,
    systems_ok: bool,
    weather_safe: bool,
}

impl PreflightChecker {
    pub fn new() -> Self {
        Self {
            min_battery: 30,
            systems_ok: true,
            weather_safe: true,
        }
    }
    
    /// Перевіряє готовність — повертає PreflightError
    pub fn check(&self, battery: u8) -> Result<(), PreflightError> {
        if battery < self.min_battery {
            return Err(PreflightError::InsufficientBattery {
                required: self.min_battery,
                available: battery,
            });
        }
        
        if !self.systems_ok {
            return Err(PreflightError::SystemNotReady(
                "Система навігації offline".to_string()
            ));
        }
        
        if !self.weather_safe {
            return Err(PreflightError::WeatherUnsafe);
        }
        
        Ok(())
    }
    
    pub fn set_systems_ok(&mut self, ok: bool) {
        self.systems_ok = ok;
    }
}

/// Навігаційна система
pub struct Navigator {
    boundary: (f64, f64, f64, f64),
    obstacles: Vec<Position>,
}

impl Navigator {
    pub fn new() -> Self {
        Self {
            boundary: (-500.0, -500.0, 500.0, 500.0),
            obstacles: Vec::new(),
        }
    }
    
    pub fn add_obstacle(&mut self, pos: Position) {
        self.obstacles.push(pos);
    }
    
    /// Розраховує шлях — повертає NavigationError
    pub fn calculate_path(
        &self, 
        from: &Position, 
        to: &Position
    ) -> Result<Vec<Position>, NavigationError> {
        // Перевірка меж
        let (min_x, min_y, max_x, max_y) = self.boundary;
        if to.x < min_x || to.x > max_x || to.y < min_y || to.y > max_y {
            return Err(NavigationError::OutOfBounds);
        }
        
        // Перевірка перешкод
        for obstacle in &self.obstacles {
            if obstacle.distance_to(to) < 10.0 {
                return Err(NavigationError::ObstacleDetected);
            }
        }
        
        // Простий шлях (в реальності — A* або інший алгоритм)
        Ok(vec![from.clone(), to.clone()])
    }
}

/// Масив сенсорів
pub struct SensorArray {
    gps_online: bool,
    altimeter_online: bool,
}

impl SensorArray {
    pub fn new() -> Self {
        Self {
            gps_online: true,
            altimeter_online: true,
        }
    }
    
    /// Читає позицію — повертає SensorError
    pub fn read_position(&self) -> Result<Position, SensorError> {
        if !self.gps_online {
            return Err(SensorError::Offline("GPS".to_string()));
        }
        Ok(Position::new(0.0, 0.0, 50.0))
    }
    
    /// Читає висоту — повертає SensorError
    pub fn read_altitude(&self) -> Result<f64, SensorError> {
        if !self.altimeter_online {
            return Err(SensorError::Offline("Висотомір".to_string()));
        }
        Ok(50.0)
    }
    
    pub fn set_gps_online(&mut self, online: bool) {
        self.gps_online = online;
    }
}

// ═══════════════════════════════════════════════════════════════════════════
// АГЕНТ З ЛАНЦЮЖКАМИ ?
// ═══════════════════════════════════════════════════════════════════════════

pub struct MissionAgent {
    id: String,
    battery: u8,
    position: Position,
    preflight: PreflightChecker,
    navigator: Navigator,
    sensors: SensorArray,
}

#[derive(Debug)]
pub struct MissionReport {
    pub waypoints_visited: usize,
    pub distance_traveled: f64,
    pub final_position: Position,
}

impl MissionAgent {
    pub fn new(id: &str) -> Self {
        Self {
            id: id.to_string(),
            battery: 100,
            position: Position::origin(),
            preflight: PreflightChecker::new(),
            navigator: Navigator::new(),
            sensors: SensorArray::new(),
        }
    }
    
    /// Виконує місію — демонструє ланцюжок ? з різними типами помилок
    pub fn execute_mission(
        &mut self, 
        waypoints: Vec<Position>
    ) -> Result<MissionReport, MissionError> {
        println!("  [{}] Початок місії", self.id);
        
        // Крок 1: Preflight — PreflightError автоматично → MissionError
        println!("  [{}] Preflight перевірка...", self.id);
        self.preflight.check(self.battery)?;
        
        // Крок 2: Читання позиції — SensorError автоматично → MissionError
        println!("  [{}] Читання GPS...", self.id);
        let current = self.sensors.read_position()?;
        self.position = current;
        
        // Крок 3: Виконання маршруту
        let mut visited = 0;
        let mut total_distance = 0.0;
        
        for (i, waypoint) in waypoints.iter().enumerate() {
            println!("  [{}] Waypoint {}/{}...", self.id, i + 1, waypoints.len());
            
            // NavigationError автоматично → MissionError
            let path = self.navigator.calculate_path(&self.position, waypoint)?;
            
            for point in path.iter().skip(1) {
                total_distance += self.position.distance_to(point);
                self.position = point.clone();
                self.battery = self.battery.saturating_sub(2);
                
                if self.battery < 20 {
                    return Err(MissionError::Execution(
                        format!("Критичний рівень батареї на waypoint {}", visited)
                    ));
                }
            }
            
            visited += 1;
        }
        
        // Крок 4: Фінальна перевірка — SensorError → MissionError
        println!("  [{}] Фінальна перевірка...", self.id);
        let final_pos = self.sensors.read_position()?;
        
        println!("  [{}] Місія завершена успішно!", self.id);
        Ok(MissionReport {
            waypoints_visited: visited,
            distance_traveled: total_distance,
            final_position: final_pos,
        })
    }
    
    /// Швидка перевірка готовності — ланцюжок ? без зайвого коду
    pub fn quick_check(&self) -> Result<(), MissionError> {
        self.preflight.check(self.battery)?;   // PreflightError → MissionError
        self.sensors.read_position()?;          // SensorError → MissionError
        self.sensors.read_altitude()?;          // SensorError → MissionError
        Ok(())
    }
    
    // Мутатори для тестування
    pub fn set_battery(&mut self, level: u8) {
        self.battery = level;
    }
    
    pub fn preflight_mut(&mut self) -> &mut PreflightChecker {
        &mut self.preflight
    }
    
    pub fn navigator_mut(&mut self) -> &mut Navigator {
        &mut self.navigator
    }
    
    pub fn sensors_mut(&mut self) -> &mut SensorArray {
        &mut self.sensors
    }
}

// ═══════════════════════════════════════════════════════════════════════════
// ДЕМОНСТРАЦІЯ
// ═══════════════════════════════════════════════════════════════════════════

fn main() {
    println!("╔══════════════════════════════════════════════════════════════╗");
    println!("║         MISSION AGENT — ДЕМО ОПЕРАТОРА ?                     ║");
    println!("╚══════════════════════════════════════════════════════════════╝\n");
    
    // Тест 1: Успішна місія
    println!("=== Тест 1: Успішна місія ===");
    let mut agent = MissionAgent::new("SCOUT-001");
    let waypoints = vec![
        Position::new(100.0, 0.0, 50.0),
        Position::new(100.0, 100.0, 50.0),
        Position::new(0.0, 100.0, 50.0),
    ];
    
    match agent.execute_mission(waypoints) {
        Ok(report) => {
            println!("✅ Успіх!");
            println!("   Відвідано waypoints: {}", report.waypoints_visited);
            println!("   Пройдено: {:.1}м", report.distance_traveled);
        }
        Err(e) => println!("❌ Помилка: {}", e),
    }
    
    // Тест 2: Низька батарея
    println!("\n=== Тест 2: Низька батарея ===");
    let mut agent2 = MissionAgent::new("SCOUT-002");
    agent2.set_battery(20);
    
    match agent2.execute_mission(vec![Position::new(50.0, 50.0, 50.0)]) {
        Ok(_) => println!("✅ Успіх"),
        Err(e) => println!("❌ Очікувана помилка: {}", e),
    }
    
    // Тест 3: Перешкода на шляху
    println!("\n=== Тест 3: Перешкода на шляху ===");
    let mut agent3 = MissionAgent::new("SCOUT-003");
    agent3.navigator_mut().add_obstacle(Position::new(100.0, 100.0, 50.0));
    
    match agent3.execute_mission(vec![Position::new(100.0, 100.0, 50.0)]) {
        Ok(_) => println!("✅ Успіх"),
        Err(e) => println!("❌ Очікувана помилка: {}", e),
    }
    
    // Тест 4: GPS offline
    println!("\n=== Тест 4: GPS offline ===");
    let mut agent4 = MissionAgent::new("SCOUT-004");
    agent4.sensors_mut().set_gps_online(false);
    
    match agent4.execute_mission(vec![Position::new(50.0, 50.0, 50.0)]) {
        Ok(_) => println!("✅ Успіх"),
        Err(e) => println!("❌ Очікувана помилка: {}", e),
    }
    
    // Тест 5: Quick check
    println!("\n=== Тест 5: Quick Check ===");
    let agent5 = MissionAgent::new("SCOUT-005");
    match agent5.quick_check() {
        Ok(()) => println!("✅ Всі системи готові"),
        Err(e) => println!("❌ Перевірка провалена: {}", e),
    }
}
```

**Як працює ця програма:**

1. **Ієрархія помилок:** `MissionError` агрегує `PreflightError`, `NavigationError`, `SensorError`

2. **From реалізації:** Кожен "маленький" тип помилки конвертується в `MissionError` автоматично

3. **Ланцюжок ?:** У `execute_mission()` кожен `?` може повернути свій тип помилки, який автоматично стає `MissionError`

4. **Чистий код:** Замість 5 вкладених match — 5 простих викликів з `?`

---

## 19.11 Лабораторна робота

**Завдання:** Створіть систему завантаження конфігурації для агента.

**Структури:**

```rust
enum ConfigError {
    MissingKey(String),
    InvalidValue { key: String, reason: String },
    OutOfRange { key: String, value: f64, min: f64, max: f64 },
}

struct ConfigLoader {
    values: HashMap<String, String>,
}

struct AgentConfig {
    max_altitude: f64,  // 10.0 - 500.0
    max_speed: f64,     // 1.0 - 50.0
    min_battery: u32,   // 10 - 100
    scan_radius: f64,   // 10.0 - 200.0
}
```

**Методи для реалізації:**

1. `ConfigLoader::get_f64(&self, key: &str) -> Result<f64, ConfigError>` — з використанням `?`
2. `ConfigLoader::get_u32(&self, key: &str) -> Result<u32, ConfigError>`
3. `AgentConfig::load(loader: &ConfigLoader) -> Result<Self, ConfigError>` — ланцюжок `?`
4. `AgentConfig::validate(&self) -> Result<(), ConfigError>` — перевірка діапазонів

**Критерії оцінювання:**

| Критерій | Бали |
|----------|------|
| ConfigError enum | 1 |
| ConfigLoader з get_f64, get_u32 | 2 |
| AgentConfig::load з ланцюжком ? | 3 |
| Валідація діапазонів | 2 |
| Тести різних сценаріїв | 2 |
| **Максимум** | **10** |

---

## Підсумок

Оператор `?` — потужний інструмент для чистого коду обробки помилок:

**Основи:**
- `?` розпаковує `Ok`/`Some` або повертає `Err`/`None`
- Працює тільки у функціях, що повертають `Result` або `Option`
- Автоматично викликає `.into()` для конвертації помилок

**Конвертація помилок:**
- `Box<dyn Error>` — швидке рішення для прототипів
- Власний enum + `impl From` — production рішення
- `map_err()` — ручна трансформація з контекстом

**Патерни:**
- `option.ok_or(error)?` — Option → Result + `?`
- `iter.map().collect::<Result<Vec<_>, _>>()` — збір з колекції
- `and_then` vs `?` — функціональний vs імперативний стиль

---

## Зв'язок з наступним розділом

Ви навчились елегантно обробляти помилки з `?`. Але наші повідомлення про помилки поки що прості — `Debug` формат або рядки. 

У **Розділі 20: Власні типи помилок та thiserror** ви навчитесь створювати професійні типи помилок з інформативними повідомленнями, ієрархіями причин, і автоматичною реалізацією стандартних traits через крейт `thiserror`.
