# Практичне заняття 10: Обробка помилок (Result, оператор ?)

## Мета заняття

Після цього заняття ви зможете:
- Розуміти філософію обробки помилок у Rust
- Використовувати `Result<T, E>` для операцій, що можуть провалитись
- Застосовувати оператор `?` для елегантного пробросу помилок
- Створювати власні типи помилок
- Обирати правильну стратегію обробки помилок

---

## Теоретичний вступ

### Філософія Rust: помилки — це дані

У багатьох мовах помилки обробляються через **винятки (exceptions)**:
```python
# Python
try:
    result = dangerous_operation()
except SomeError as e:
    handle_error(e)
```

Rust обрав **інший шлях**: помилки — це звичайні значення, що повертаються з функцій. Це має переваги:

| Exceptions | Result у Rust |
|------------|---------------|
| Неявний control flow | Явний control flow |
| Легко забути обробити | Компілятор змушує обробити |
| Runtime overhead | Zero-cost abstraction |
| Складно відстежити | Видно в сигнатурі функції |

### Два типи "відсутності": Option та Result

| Тип | Коли використовувати | Приклад |
|-----|---------------------|---------|
| `Option<T>` | Значення може бути відсутнім | Пошук у колекції |
| `Result<T, E>` | Операція може провалитись | Читання файлу |

```rust
// Option — "є чи немає"
fn find_drone(id: u32) -> Option<Drone> { ... }

// Result — "успіх чи помилка"
fn connect_to_drone(id: u32) -> Result<Connection, ConnectionError> { ... }
```

---

## Result<T, E> — детально

### Визначення

```rust
enum Result<T, E> {
    Ok(T),   // Успіх зі значенням типу T
    Err(E),  // Помилка зі значенням типу E
}
```

### Створення Result

```rust
fn divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        Err(String::from("Ділення на нуль!"))
    } else {
        Ok(a / b)
    }
}

fn main() {
    let result1 = divide(10.0, 2.0);   // Ok(5.0)
    let result2 = divide(10.0, 0.0);   // Err("Ділення на нуль!")
    
    println!("{:?}", result1);
    println!("{:?}", result2);
}
```

### Обробка Result через match

```rust
fn main() {
    let result = divide(10.0, 2.0);
    
    match result {
        Ok(value) => println!("Результат: {}", value),
        Err(error) => println!("Помилка: {}", error),
    }
}
```

### Методи Result

```rust
fn main() {
    let ok_result: Result<i32, &str> = Ok(42);
    let err_result: Result<i32, &str> = Err("помилка");
    
    // is_ok / is_err
    println!("ok_result.is_ok(): {}", ok_result.is_ok());     // true
    println!("err_result.is_err(): {}", err_result.is_err()); // true
    
    // unwrap — паніка якщо Err!
    let value = ok_result.unwrap();  // 42
    // let boom = err_result.unwrap();  // ПАНІКА!
    
    // expect — паніка з власним повідомленням
    let value = ok_result.expect("Має бути значення");
    
    // unwrap_or — значення за замовчуванням
    let value = err_result.unwrap_or(0);  // 0
    
    // unwrap_or_else — обчислити значення за замовчуванням
    let value = err_result.unwrap_or_else(|e| {
        println!("Помилка: {}", e);
        -1
    });
    
    // ok() — перетворити Result<T, E> в Option<T>
    let option = ok_result.ok();  // Some(42)
    
    // err() — перетворити Result<T, E> в Option<E>
    let error = err_result.err();  // Some("помилка")
}
```

### Комбінатори Result

```rust
fn main() {
    let result: Result<i32, &str> = Ok(10);
    
    // map — трансформувати Ok значення
    let doubled = result.map(|x| x * 2);  // Ok(20)
    
    // map_err — трансформувати Err значення
    let result: Result<i32, &str> = Err("error");
    let mapped = result.map_err(|e| format!("Wrapped: {}", e));
    
    // and_then — ланцюжок операцій (flatMap)
    let result: Result<i32, &str> = Ok(10);
    let chained = result
        .and_then(|x| if x > 5 { Ok(x * 2) } else { Err("занадто мале") });
    
    // or_else — альтернатива при помилці
    let result: Result<i32, &str> = Err("first error");
    let recovered = result.or_else(|_| Ok(0));  // Ok(0)
}
```

---

## Оператор ? — елегантний проброс помилок

### Проблема: verbose код

```rust
fn read_config() -> Result<String, io::Error> {
    let file = File::open("config.txt");
    let mut file = match file {
        Ok(f) => f,
        Err(e) => return Err(e),
    };
    
    let mut contents = String::new();
    match file.read_to_string(&mut contents) {
        Ok(_) => Ok(contents),
        Err(e) => Err(e),
    }
}
```

### Рішення: оператор ?

```rust
fn read_config() -> Result<String, io::Error> {
    let mut file = File::open("config.txt")?;  // Якщо Err — повертає його
    let mut contents = String::new();
    file.read_to_string(&mut contents)?;
    Ok(contents)
}
```

### Як працює ?

```rust
// Цей код:
let value = some_result?;

// Еквівалентний цьому:
let value = match some_result {
    Ok(v) => v,
    Err(e) => return Err(e.into()),  // .into() для конвертації типів
};
```

### Ланцюжки з ?

```rust
fn process_drone_data(id: u32) -> Result<Report, Error> {
    let drone = find_drone(id)?;           // Може повернути Err
    let data = drone.fetch_telemetry()?;   // Може повернути Err
    let analyzed = analyze_data(&data)?;   // Може повернути Err
    let report = generate_report(analyzed)?;
    Ok(report)
}
```

### ? з Option

Оператор `?` працює і з `Option` (у функціях, що повертають `Option`):

```rust
fn get_drone_battery(drones: &HashMap<String, Drone>, name: &str) -> Option<i32> {
    let drone = drones.get(name)?;  // None → повертає None
    Some(drone.battery)
}
```

### Конвертація між Option та Result

```rust
fn find_and_process(id: u32) -> Result<Data, Error> {
    // Option → Result через ok_or
    let drone = find_drone(id).ok_or(Error::NotFound)?;
    
    // Option → Result через ok_or_else (lazy)
    let data = drone.data.ok_or_else(|| Error::NoData(id))?;
    
    Ok(data)
}
```

---

## Власні типи помилок

### Простий enum помилок

```rust
#[derive(Debug)]
enum DroneError {
    NotFound,
    LowBattery(i32),
    ConnectionFailed(String),
    InvalidCommand,
}

fn execute_command(drone_id: u32, cmd: &str) -> Result<(), DroneError> {
    let drone = find_drone(drone_id).ok_or(DroneError::NotFound)?;
    
    if drone.battery < 10 {
        return Err(DroneError::LowBattery(drone.battery));
    }
    
    if cmd.is_empty() {
        return Err(DroneError::InvalidCommand);
    }
    
    // Виконання команди...
    Ok(())
}
```

### Реалізація Display та Error

```rust
use std::fmt;
use std::error::Error;

#[derive(Debug)]
enum DroneError {
    NotFound(u32),
    LowBattery { current: i32, required: i32 },
    ConnectionFailed(String),
    Timeout,
}

impl fmt::Display for DroneError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            DroneError::NotFound(id) => write!(f, "Дрон #{} не знайдено", id),
            DroneError::LowBattery { current, required } => {
                write!(f, "Низький заряд: {}% (потрібно {}%)", current, required)
            },
            DroneError::ConnectionFailed(reason) => {
                write!(f, "Помилка з'єднання: {}", reason)
            },
            DroneError::Timeout => write!(f, "Таймаут операції"),
        }
    }
}

impl Error for DroneError {}
```

### From trait для автоматичної конвертації

```rust
use std::io;

#[derive(Debug)]
enum AppError {
    Io(io::Error),
    Parse(std::num::ParseIntError),
    Drone(DroneError),
}

impl From<io::Error> for AppError {
    fn from(err: io::Error) -> Self {
        AppError::Io(err)
    }
}

impl From<std::num::ParseIntError> for AppError {
    fn from(err: std::num::ParseIntError) -> Self {
        AppError::Parse(err)
    }
}

// Тепер ? автоматично конвертує помилки!
fn load_drone_config(path: &str) -> Result<DroneConfig, AppError> {
    let contents = std::fs::read_to_string(path)?;  // io::Error → AppError
    let id: u32 = contents.trim().parse()?;          // ParseIntError → AppError
    Ok(DroneConfig { id })
}
```

---

## Стратегії обробки помилок

### 1. Паніка — для невідновлюваних помилок

```rust
fn get_config() -> Config {
    // Якщо конфіг відсутній — програма не може працювати
    Config::load().expect("Критична помилка: конфігурація не знайдена")
}
```

### 2. Проброс — нехай викликаючий код вирішує

```rust
fn process_file(path: &str) -> Result<Data, Error> {
    let content = std::fs::read_to_string(path)?;
    // ...
}
```

### 3. Обробка на місці — коли можна відновитись

```rust
fn get_battery_or_default(drone: &Drone) -> i32 {
    drone.query_battery().unwrap_or(0)
}
```

### 4. Fallback — спробувати альтернативу

```rust
fn connect_to_drone(id: u32) -> Result<Connection, Error> {
    // Спробувати WiFi, якщо не вдалось — Bluetooth
    connect_wifi(id).or_else(|_| connect_bluetooth(id))
}
```

### 5. Логування та продовження

```rust
fn process_drones(ids: &[u32]) -> Vec<Report> {
    ids.iter()
        .filter_map(|&id| {
            match generate_report(id) {
                Ok(report) => Some(report),
                Err(e) => {
                    eprintln!("Помилка для дрона {}: {}", id, e);
                    None  // Пропускаємо, продовжуємо з іншими
                }
            }
        })
        .collect()
}
```

---

## Типові помилки

### Помилка 1: Надмірне використання unwrap()

```rust
// ПОГАНО — програма впаде при будь-якій помилці
let data = fetch_data().unwrap();
let parsed = parse(data).unwrap();
let result = process(parsed).unwrap();

// ДОБРЕ — явна обробка
let data = fetch_data()?;
let parsed = parse(data)?;
let result = process(parsed)?;
```

### Помилка 2: Ігнорування Result

```rust
// ПОГАНО — компілятор попередить!
std::fs::remove_file("temp.txt");  // Result ігнорується

// ДОБРЕ
let _ = std::fs::remove_file("temp.txt");  // Явно ігноруємо
// або
std::fs::remove_file("temp.txt").ok();  // Перетворюємо в Option і ігноруємо
// або
if let Err(e) = std::fs::remove_file("temp.txt") {
    eprintln!("Не вдалось видалити: {}", e);
}
```

### Помилка 3: ? у функції без Result

```rust
// ПОМИЛКА компіляції!
fn main() {
    let file = File::open("test.txt")?;  // main не повертає Result!
}

// РІШЕННЯ 1: main з Result
fn main() -> Result<(), Box<dyn Error>> {
    let file = File::open("test.txt")?;
    Ok(())
}

// РІШЕННЯ 2: обробити на місці
fn main() {
    match File::open("test.txt") {
        Ok(file) => { /* використовуємо file */ },
        Err(e) => eprintln!("Помилка: {}", e),
    }
}
```

### Помилка 4: Втрата контексту помилки

```rust
// ПОГАНО — незрозуміло, що саме не вдалось
fn load_config() -> Result<Config, io::Error> {
    let content = std::fs::read_to_string("config.json")?;
    // Якщо помилка — незрозуміло який файл
}

// ДОБРЕ — додаємо контекст
fn load_config() -> Result<Config, String> {
    let content = std::fs::read_to_string("config.json")
        .map_err(|e| format!("Не вдалось прочитати config.json: {}", e))?;
    Ok(parse_config(&content)?)
}
```

---

## Практичні задачі

### Задача 1: Валідація даних дрона

**Умова:** Створіть функції валідації параметрів дрона, що повертають `Result`. Валідуйте: ім'я (непорожнє, без спецсимволів), батарею (0-100), координати (в межах зони).

**Розв'язання:**

```rust
#[derive(Debug)]
enum ValidationError {
    EmptyName,
    InvalidNameChar(char),
    NameTooLong(usize),
    BatteryOutOfRange(i32),
    CoordinateOutOfBounds { coord: &'static str, value: i32, limit: i32 },
}

impl std::fmt::Display for ValidationError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            ValidationError::EmptyName => write!(f, "Ім'я не може бути порожнім"),
            ValidationError::InvalidNameChar(c) => write!(f, "Недопустимий символ у імені: '{}'", c),
            ValidationError::NameTooLong(len) => write!(f, "Ім'я занадто довге: {} символів (макс 20)", len),
            ValidationError::BatteryOutOfRange(b) => write!(f, "Батарея поза межами: {}% (допустимо 0-100)", b),
            ValidationError::CoordinateOutOfBounds { coord, value, limit } => {
                write!(f, "Координата {} = {} поза межами (±{})", coord, value, limit)
            }
        }
    }
}

fn validate_name(name: &str) -> Result<(), ValidationError> {
    if name.is_empty() {
        return Err(ValidationError::EmptyName);
    }
    
    if name.len() > 20 {
        return Err(ValidationError::NameTooLong(name.len()));
    }
    
    for c in name.chars() {
        if !c.is_alphanumeric() && c != '-' && c != '_' {
            return Err(ValidationError::InvalidNameChar(c));
        }
    }
    
    Ok(())
}

fn validate_battery(battery: i32) -> Result<(), ValidationError> {
    if battery < 0 || battery > 100 {
        Err(ValidationError::BatteryOutOfRange(battery))
    } else {
        Ok(())
    }
}

fn validate_coordinates(x: i32, y: i32, max_range: i32) -> Result<(), ValidationError> {
    if x.abs() > max_range {
        return Err(ValidationError::CoordinateOutOfBounds {
            coord: "X",
            value: x,
            limit: max_range,
        });
    }
    
    if y.abs() > max_range {
        return Err(ValidationError::CoordinateOutOfBounds {
            coord: "Y",
            value: y,
            limit: max_range,
        });
    }
    
    Ok(())
}

#[derive(Debug)]
struct DroneConfig {
    name: String,
    battery: i32,
    x: i32,
    y: i32,
}

fn validate_drone_config(config: &DroneConfig) -> Result<(), Vec<ValidationError>> {
    let mut errors = Vec::new();
    
    if let Err(e) = validate_name(&config.name) {
        errors.push(e);
    }
    
    if let Err(e) = validate_battery(config.battery) {
        errors.push(e);
    }
    
    if let Err(e) = validate_coordinates(config.x, config.y, 1000) {
        errors.push(e);
    }
    
    if errors.is_empty() {
        Ok(())
    } else {
        Err(errors)
    }
}

fn main() {
    println!("=== Валідація даних дрона ===\n");
    
    // Тестові конфігурації
    let configs = vec![
        DroneConfig { name: String::from("Alpha-1"), battery: 85, x: 100, y: 200 },
        DroneConfig { name: String::from(""), battery: 50, x: 0, y: 0 },
        DroneConfig { name: String::from("Drone@#$"), battery: 150, x: 2000, y: -500 },
        DroneConfig { name: String::from("Valid_Drone"), battery: -10, x: 500, y: 500 },
    ];
    
    for (i, config) in configs.iter().enumerate() {
        println!("--- Конфігурація {} ---", i + 1);
        println!("  {:?}", config);
        
        match validate_drone_config(config) {
            Ok(()) => println!("  ✓ Валідна\n"),
            Err(errors) => {
                println!("  ✗ Помилки:");
                for e in errors {
                    println!("    • {}", e);
                }
                println!();
            }
        }
    }
    
    // Демонстрація окремих функцій
    println!("--- Окремі перевірки ---");
    
    match validate_name("Good-Name_123") {
        Ok(()) => println!("✓ Ім'я валідне"),
        Err(e) => println!("✗ {}", e),
    }
    
    match validate_battery(75) {
        Ok(()) => println!("✓ Батарея валідна"),
        Err(e) => println!("✗ {}", e),
    }
    
    match validate_coordinates(500, -300, 1000) {
        Ok(()) => println!("✓ Координати валідні"),
        Err(e) => println!("✗ {}", e),
    }
}
```

**Результат:**
```
=== Валідація даних дрона ===

--- Конфігурація 1 ---
  DroneConfig { name: "Alpha-1", battery: 85, x: 100, y: 200 }
  ✓ Валідна

--- Конфігурація 2 ---
  DroneConfig { name: "", battery: 50, x: 0, y: 0 }
  ✗ Помилки:
    • Ім'я не може бути порожнім

--- Конфігурація 3 ---
  DroneConfig { name: "Drone@#$", battery: 150, x: 2000, y: -500 }
  ✗ Помилки:
    • Недопустимий символ у імені: '@'
    • Батарея поза межами: 150% (допустимо 0-100)
    • Координата X = 2000 поза межами (±1000)
...
```

---

### Задача 2: Ланцюжок операцій з ?

**Умова:** Реалізуйте систему виконання місії дрона, де кожен крок може провалитись. Використовуйте оператор `?` для елегантного пробросу помилок.

**Розв'язання:**

```rust
use std::collections::HashMap;

#[derive(Debug)]
enum MissionError {
    DroneNotFound(String),
    InsufficientBattery { current: i32, required: i32 },
    WaypointUnreachable(i32, i32),
    SensorFailure(String),
    CommunicationLost,
    MissionAborted(String),
}

impl std::fmt::Display for MissionError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            MissionError::DroneNotFound(id) => write!(f, "Дрон '{}' не знайдено", id),
            MissionError::InsufficientBattery { current, required } => {
                write!(f, "Недостатньо батареї: {}% (потрібно {}%)", current, required)
            },
            MissionError::WaypointUnreachable(x, y) => {
                write!(f, "Точка ({}, {}) недосяжна", x, y)
            },
            MissionError::SensorFailure(sensor) => write!(f, "Збій сенсора: {}", sensor),
            MissionError::CommunicationLost => write!(f, "Втрачено зв'язок"),
            MissionError::MissionAborted(reason) => write!(f, "Місію перервано: {}", reason),
        }
    }
}

#[derive(Debug, Clone)]
struct Drone {
    id: String,
    battery: i32,
    x: i32,
    y: i32,
    sensors_ok: bool,
}

struct MissionControl {
    drones: HashMap<String, Drone>,
}

impl MissionControl {
    fn new() -> Self {
        MissionControl {
            drones: HashMap::new(),
        }
    }
    
    fn add_drone(&mut self, drone: Drone) {
        self.drones.insert(drone.id.clone(), drone);
    }
    
    fn get_drone(&self, id: &str) -> Result<&Drone, MissionError> {
        self.drones.get(id).ok_or_else(|| MissionError::DroneNotFound(id.to_string()))
    }
    
    fn get_drone_mut(&mut self, id: &str) -> Result<&mut Drone, MissionError> {
        self.drones.get_mut(id).ok_or_else(|| MissionError::DroneNotFound(id.to_string()))
    }
    
    fn check_battery(&self, drone: &Drone, required: i32) -> Result<(), MissionError> {
        if drone.battery >= required {
            Ok(())
        } else {
            Err(MissionError::InsufficientBattery {
                current: drone.battery,
                required,
            })
        }
    }
    
    fn check_sensors(&self, drone: &Drone) -> Result<(), MissionError> {
        if drone.sensors_ok {
            Ok(())
        } else {
            Err(MissionError::SensorFailure(String::from("GPS")))
        }
    }
    
    fn move_drone(&mut self, id: &str, x: i32, y: i32) -> Result<(), MissionError> {
        let drone = self.get_drone_mut(id)?;
        
        // Перевірка досяжності
        let distance = (((x - drone.x).pow(2) + (y - drone.y).pow(2)) as f64).sqrt();
        if distance > 500.0 {
            return Err(MissionError::WaypointUnreachable(x, y));
        }
        
        // Витрата батареї
        let cost = (distance / 10.0) as i32;
        if drone.battery < cost {
            return Err(MissionError::InsufficientBattery {
                current: drone.battery,
                required: cost,
            });
        }
        
        drone.x = x;
        drone.y = y;
        drone.battery -= cost;
        
        Ok(())
    }
    
    // Головна функція місії — використовує ? для пробросу помилок
    fn execute_mission(&mut self, drone_id: &str, waypoints: &[(i32, i32)]) -> Result<MissionReport, MissionError> {
        println!("Запуск місії для дрона '{}'...", drone_id);
        
        // Крок 1: Знайти дрон
        let drone = self.get_drone(drone_id)?;
        println!("  ✓ Дрон знайдено: {:?}", drone);
        
        // Крок 2: Перевірити батарею
        self.check_battery(drone, 20)?;
        println!("  ✓ Батарея достатня: {}%", drone.battery);
        
        // Крок 3: Перевірити сенсори
        self.check_sensors(drone)?;
        println!("  ✓ Сенсори працюють");
        
        // Крок 4: Виконати маршрут
        let mut visited = Vec::new();
        for (i, &(x, y)) in waypoints.iter().enumerate() {
            println!("  → Рухаємось до точки {} ({}, {})", i + 1, x, y);
            self.move_drone(drone_id, x, y)?;
            visited.push((x, y));
            
            let drone = self.get_drone(drone_id)?;
            println!("    Позиція: ({}, {}), батарея: {}%", drone.x, drone.y, drone.battery);
        }
        
        // Крок 5: Повернення на базу
        println!("  → Повернення на базу...");
        self.move_drone(drone_id, 0, 0)?;
        
        let drone = self.get_drone(drone_id)?;
        
        Ok(MissionReport {
            drone_id: drone_id.to_string(),
            waypoints_visited: visited.len(),
            final_battery: drone.battery,
            success: true,
        })
    }
}

#[derive(Debug)]
struct MissionReport {
    drone_id: String,
    waypoints_visited: usize,
    final_battery: i32,
    success: bool,
}

fn main() {
    println!("=== Ланцюжок операцій з ? ===\n");
    
    let mut control = MissionControl::new();
    
    // Додаємо дрони
    control.add_drone(Drone {
        id: String::from("Alpha"),
        battery: 80,
        x: 0,
        y: 0,
        sensors_ok: true,
    });
    
    control.add_drone(Drone {
        id: String::from("Beta"),
        battery: 15,
        x: 0,
        y: 0,
        sensors_ok: true,
    });
    
    control.add_drone(Drone {
        id: String::from("Gamma"),
        battery: 90,
        x: 0,
        y: 0,
        sensors_ok: false,  // Сенсори не працюють!
    });
    
    // Тестові місії
    let missions = vec![
        ("Alpha", vec![(100, 50), (150, 100), (200, 50)]),
        ("Beta", vec![(50, 50)]),      // Мало батареї
        ("Gamma", vec![(100, 100)]),   // Збій сенсорів
        ("Delta", vec![(0, 0)]),       // Неіснуючий дрон
        ("Alpha", vec![(100, 100), (600, 600)]),  // Занадто далеко
    ];
    
    for (drone_id, waypoints) in missions {
        println!("\n{'='!s:->50}", "");
        match control.execute_mission(drone_id, &waypoints) {
            Ok(report) => {
                println!("\n✓ Місія успішна!");
                println!("  Звіт: {:?}", report);
            },
            Err(e) => {
                println!("\n✗ Місія провалена: {}", e);
            }
        }
    }
}
```

---

### Задача 3: Власний тип помилок

**Умова:** Створіть повноцінний тип помилок для системи управління дронами з реалізацією `std::error::Error`, `Display`, та `From` для конвертації з інших типів помилок.

**Розв'язання:**

```rust
use std::fmt;
use std::error::Error;
use std::num::ParseIntError;
use std::io;

// Основний тип помилок
#[derive(Debug)]
enum DroneSystemError {
    // Помилки дрона
    DroneNotFound { id: String },
    DroneOffline { id: String, last_seen: u64 },
    
    // Помилки операцій
    InsufficientBattery { current: i32, required: i32 },
    InvalidCommand { command: String, reason: String },
    OperationTimeout { operation: String, timeout_ms: u64 },
    
    // Помилки комунікації
    ConnectionFailed { target: String, cause: String },
    MessageCorrupted,
    
    // Обгорнуті зовнішні помилки
    Io(io::Error),
    Parse(ParseIntError),
    
    // Інше
    Internal(String),
}

impl fmt::Display for DroneSystemError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            DroneSystemError::DroneNotFound { id } => {
                write!(f, "Дрон '{}' не знайдено в системі", id)
            },
            DroneSystemError::DroneOffline { id, last_seen } => {
                write!(f, "Дрон '{}' офлайн (останній зв'язок: {} сек тому)", id, last_seen)
            },
            DroneSystemError::InsufficientBattery { current, required } => {
                write!(f, "Недостатньо заряду: {}% (потрібно {}%)", current, required)
            },
            DroneSystemError::InvalidCommand { command, reason } => {
                write!(f, "Невалідна команда '{}': {}", command, reason)
            },
            DroneSystemError::OperationTimeout { operation, timeout_ms } => {
                write!(f, "Таймаут операції '{}' після {} мс", operation, timeout_ms)
            },
            DroneSystemError::ConnectionFailed { target, cause } => {
                write!(f, "Не вдалось з'єднатись з '{}': {}", target, cause)
            },
            DroneSystemError::MessageCorrupted => {
                write!(f, "Отримано пошкоджене повідомлення")
            },
            DroneSystemError::Io(e) => {
                write!(f, "Помилка вводу/виводу: {}", e)
            },
            DroneSystemError::Parse(e) => {
                write!(f, "Помилка парсингу: {}", e)
            },
            DroneSystemError::Internal(msg) => {
                write!(f, "Внутрішня помилка: {}", msg)
            },
        }
    }
}

impl Error for DroneSystemError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            DroneSystemError::Io(e) => Some(e),
            DroneSystemError::Parse(e) => Some(e),
            _ => None,
        }
    }
}

// Автоматична конвертація з io::Error
impl From<io::Error> for DroneSystemError {
    fn from(err: io::Error) -> Self {
        DroneSystemError::Io(err)
    }
}

// Автоматична конвертація з ParseIntError
impl From<ParseIntError> for DroneSystemError {
    fn from(err: ParseIntError) -> Self {
        DroneSystemError::Parse(err)
    }
}

// Зручні конструктори
impl DroneSystemError {
    fn not_found(id: &str) -> Self {
        DroneSystemError::DroneNotFound { id: id.to_string() }
    }
    
    fn low_battery(current: i32, required: i32) -> Self {
        DroneSystemError::InsufficientBattery { current, required }
    }
    
    fn invalid_command(command: &str, reason: &str) -> Self {
        DroneSystemError::InvalidCommand {
            command: command.to_string(),
            reason: reason.to_string(),
        }
    }
    
    fn timeout(operation: &str, timeout_ms: u64) -> Self {
        DroneSystemError::OperationTimeout {
            operation: operation.to_string(),
            timeout_ms,
        }
    }
}

// Тип Result для зручності
type DroneResult<T> = Result<T, DroneSystemError>;

// Демонстраційні функції
fn parse_drone_id(input: &str) -> DroneResult<u32> {
    let id: u32 = input.trim().parse()?;  // ParseIntError → DroneSystemError
    Ok(id)
}

fn read_config_file(path: &str) -> DroneResult<String> {
    // Симуляція помилки читання файлу
    if path.contains("missing") {
        return Err(io::Error::new(io::ErrorKind::NotFound, "Файл не знайдено").into());
    }
    Ok(String::from("config data"))
}

fn execute_drone_command(drone_id: &str, command: &str) -> DroneResult<String> {
    // Перевірка дрона
    if drone_id == "unknown" {
        return Err(DroneSystemError::not_found(drone_id));
    }
    
    // Перевірка команди
    if command.is_empty() {
        return Err(DroneSystemError::invalid_command(command, "порожня команда"));
    }
    
    // Симуляція виконання
    if command == "TIMEOUT" {
        return Err(DroneSystemError::timeout("execute", 5000));
    }
    
    Ok(format!("Команда '{}' виконана для дрона '{}'", command, drone_id))
}

fn main() {
    println!("=== Власний тип помилок ===\n");
    
    // Тест 1: Парсинг
    println!("--- Парсинг ID ---");
    for input in ["123", "abc", "  456  "] {
        match parse_drone_id(input) {
            Ok(id) => println!("  '{}' → ID: {}", input, id),
            Err(e) => println!("  '{}' → Помилка: {}", input, e),
        }
    }
    
    // Тест 2: Читання файлу
    println!("\n--- Читання конфігурації ---");
    for path in ["config.json", "missing.json"] {
        match read_config_file(path) {
            Ok(data) => println!("  '{}' → {}", path, data),
            Err(e) => println!("  '{}' → Помилка: {}", path, e),
        }
    }
    
    // Тест 3: Виконання команд
    println!("\n--- Виконання команд ---");
    let test_cases = [
        ("Alpha", "TAKEOFF"),
        ("unknown", "LAND"),
        ("Beta", ""),
        ("Gamma", "TIMEOUT"),
    ];
    
    for (drone, cmd) in test_cases {
        match execute_drone_command(drone, cmd) {
            Ok(result) => println!("  ✓ {}", result),
            Err(e) => println!("  ✗ {}", e),
        }
    }
    
    // Тест 4: Ланцюжок помилок (source)
    println!("\n--- Ланцюжок помилок ---");
    let err = DroneSystemError::Io(io::Error::new(
        io::ErrorKind::ConnectionRefused,
        "Connection refused"
    ));
    println!("Помилка: {}", err);
    if let Some(source) = err.source() {
        println!("Причина: {}", source);
    }
}
```

---

### Задача 4: Комплексна обробка з fallback

**Умова:** Реалізуйте систему з резервними варіантами: якщо основна операція провалюється — спробувати альтернативу, логувати помилки, збирати статистику.

**Розв'язання:**

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
enum ConnectionMethod {
    WiFi,
    Bluetooth,
    Radio,
    Satellite,
}

impl std::fmt::Display for ConnectionMethod {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            ConnectionMethod::WiFi => write!(f, "WiFi"),
            ConnectionMethod::Bluetooth => write!(f, "Bluetooth"),
            ConnectionMethod::Radio => write!(f, "Radio"),
            ConnectionMethod::Satellite => write!(f, "Satellite"),
        }
    }
}

#[derive(Debug)]
struct ConnectionError {
    method: ConnectionMethod,
    reason: String,
}

struct DroneConnection {
    drone_id: String,
    method: ConnectionMethod,
}

struct ConnectionStats {
    attempts: HashMap<String, u32>,
    successes: HashMap<String, u32>,
    failures: Vec<String>,
}

impl ConnectionStats {
    fn new() -> Self {
        ConnectionStats {
            attempts: HashMap::new(),
            successes: HashMap::new(),
            failures: Vec::new(),
        }
    }
    
    fn record_attempt(&mut self, method: &str) {
        *self.attempts.entry(method.to_string()).or_insert(0) += 1;
    }
    
    fn record_success(&mut self, method: &str) {
        *self.successes.entry(method.to_string()).or_insert(0) += 1;
    }
    
    fn record_failure(&mut self, message: String) {
        self.failures.push(message);
    }
    
    fn print_summary(&self) {
        println!("\n═══ Статистика з'єднань ═══");
        println!("Спроби:");
        for (method, count) in &self.attempts {
            let success = self.successes.get(method).unwrap_or(&0);
            let rate = if *count > 0 {
                (*success as f64 / *count as f64) * 100.0
            } else {
                0.0
            };
            println!("  {}: {} спроб, {} успішних ({:.0}%)", method, count, success, rate);
        }
        
        if !self.failures.is_empty() {
            println!("\nОстанні помилки:");
            for (i, msg) in self.failures.iter().rev().take(5).enumerate() {
                println!("  {}. {}", i + 1, msg);
            }
        }
    }
}

// Симуляція підключення різними методами
fn connect_wifi(drone_id: &str) -> Result<DroneConnection, ConnectionError> {
    // Симуляція: WiFi працює для дронів з парним хешем
    let hash: u32 = drone_id.bytes().map(|b| b as u32).sum();
    if hash % 3 == 0 {
        Ok(DroneConnection {
            drone_id: drone_id.to_string(),
            method: ConnectionMethod::WiFi,
        })
    } else {
        Err(ConnectionError {
            method: ConnectionMethod::WiFi,
            reason: String::from("Сигнал занадто слабкий"),
        })
    }
}

fn connect_bluetooth(drone_id: &str) -> Result<DroneConnection, ConnectionError> {
    let hash: u32 = drone_id.bytes().map(|b| b as u32).sum();
    if hash % 4 == 0 {
        Ok(DroneConnection {
            drone_id: drone_id.to_string(),
            method: ConnectionMethod::Bluetooth,
        })
    } else {
        Err(ConnectionError {
            method: ConnectionMethod::Bluetooth,
            reason: String::from("Пристрій не знайдено"),
        })
    }
}

fn connect_radio(drone_id: &str) -> Result<DroneConnection, ConnectionError> {
    let hash: u32 = drone_id.bytes().map(|b| b as u32).sum();
    if hash % 5 == 0 {
        Ok(DroneConnection {
            drone_id: drone_id.to_string(),
            method: ConnectionMethod::Radio,
        })
    } else {
        Err(ConnectionError {
            method: ConnectionMethod::Radio,
            reason: String::from("Частота зайнята"),
        })
    }
}

fn connect_satellite(_drone_id: &str) -> Result<DroneConnection, ConnectionError> {
    // Супутник завжди працює (резервний)
    Ok(DroneConnection {
        drone_id: _drone_id.to_string(),
        method: ConnectionMethod::Satellite,
    })
}

// Функція з fallback-ланцюжком
fn connect_with_fallback(
    drone_id: &str,
    stats: &mut ConnectionStats,
) -> Result<DroneConnection, Vec<ConnectionError>> {
    let mut errors = Vec::new();
    
    // Спроба 1: WiFi (найшвидший)
    stats.record_attempt("WiFi");
    match connect_wifi(drone_id) {
        Ok(conn) => {
            stats.record_success("WiFi");
            return Ok(conn);
        },
        Err(e) => {
            stats.record_failure(format!("[{}] WiFi: {}", drone_id, e.reason));
            errors.push(e);
        }
    }
    
    // Спроба 2: Bluetooth (близька відстань)
    stats.record_attempt("Bluetooth");
    match connect_bluetooth(drone_id) {
        Ok(conn) => {
            stats.record_success("Bluetooth");
            return Ok(conn);
        },
        Err(e) => {
            stats.record_failure(format!("[{}] Bluetooth: {}", drone_id, e.reason));
            errors.push(e);
        }
    }
    
    // Спроба 3: Radio (середня відстань)
    stats.record_attempt("Radio");
    match connect_radio(drone_id) {
        Ok(conn) => {
            stats.record_success("Radio");
            return Ok(conn);
        },
        Err(e) => {
            stats.record_failure(format!("[{}] Radio: {}", drone_id, e.reason));
            errors.push(e);
        }
    }
    
    // Спроба 4: Satellite (завжди працює)
    stats.record_attempt("Satellite");
    match connect_satellite(drone_id) {
        Ok(conn) => {
            stats.record_success("Satellite");
            return Ok(conn);
        },
        Err(e) => {
            stats.record_failure(format!("[{}] Satellite: {}", drone_id, e.reason));
            errors.push(e);
        }
    }
    
    Err(errors)
}

// Альтернатива: використання or_else
fn connect_smart(drone_id: &str) -> Result<DroneConnection, ConnectionError> {
    connect_wifi(drone_id)
        .or_else(|_| connect_bluetooth(drone_id))
        .or_else(|_| connect_radio(drone_id))
        .or_else(|_| connect_satellite(drone_id))
}

fn main() {
    println!("=== Комплексна обробка з fallback ===\n");
    
    let mut stats = ConnectionStats::new();
    
    let drones = [
        "Alpha", "Beta", "Gamma", "Delta", "Epsilon",
        "Zeta", "Eta", "Theta", "Iota", "Kappa",
    ];
    
    println!("--- Підключення дронів ---\n");
    
    for drone_id in drones {
        match connect_with_fallback(drone_id, &mut stats) {
            Ok(conn) => {
                println!("✓ {} підключено через {}", conn.drone_id, conn.method);
            },
            Err(errors) => {
                println!("✗ {} не вдалось підключити:", drone_id);
                for e in errors {
                    println!("    {} - {}", e.method, e.reason);
                }
            }
        }
    }
    
    stats.print_summary();
    
    // Демонстрація or_else
    println!("\n--- Використання or_else ---");
    match connect_smart("TestDrone") {
        Ok(conn) => println!("✓ Підключено через {}", conn.method),
        Err(e) => println!("✗ Помилка: {}", e.reason),
    }
}
```

---

## Домашнє завдання

### Завдання 1: Парсер конфігурації

**Умова:** Створіть парсер конфігурації дрона з рядка формату `key=value`. Обробіть помилки: відсутній роздільник, невідомий ключ, невалідне значення.

**Розв'язання:**

```rust
use std::collections::HashMap;

#[derive(Debug)]
enum ConfigError {
    MissingSeparator(String),
    EmptyKey(String),
    EmptyValue(String),
    UnknownKey(String),
    InvalidValue { key: String, value: String, expected: String },
    MissingRequired(String),
}

impl std::fmt::Display for ConfigError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            ConfigError::MissingSeparator(line) => {
                write!(f, "Відсутній роздільник '=' у рядку: '{}'", line)
            },
            ConfigError::EmptyKey(line) => {
                write!(f, "Порожній ключ у рядку: '{}'", line)
            },
            ConfigError::EmptyValue(key) => {
                write!(f, "Порожнє значення для ключа '{}'", key)
            },
            ConfigError::UnknownKey(key) => {
                write!(f, "Невідомий ключ: '{}'", key)
            },
            ConfigError::InvalidValue { key, value, expected } => {
                write!(f, "Невалідне значення для '{}': '{}' (очікується {})", key, value, expected)
            },
            ConfigError::MissingRequired(key) => {
                write!(f, "Відсутній обов'язковий параметр: '{}'", key)
            },
        }
    }
}

#[derive(Debug, Default)]
struct DroneConfig {
    name: String,
    id: u32,
    max_altitude: i32,
    max_speed: f64,
    battery_threshold: i32,
}

fn parse_line(line: &str) -> Result<(String, String), ConfigError> {
    let line = line.trim();
    
    if line.is_empty() || line.starts_with('#') {
        return Ok((String::new(), String::new()));  // Пропускаємо
    }
    
    let parts: Vec<&str> = line.splitn(2, '=').collect();
    
    if parts.len() != 2 {
        return Err(ConfigError::MissingSeparator(line.to_string()));
    }
    
    let key = parts[0].trim();
    let value = parts[1].trim();
    
    if key.is_empty() {
        return Err(ConfigError::EmptyKey(line.to_string()));
    }
    
    if value.is_empty() {
        return Err(ConfigError::EmptyValue(key.to_string()));
    }
    
    Ok((key.to_string(), value.to_string()))
}

fn parse_config(text: &str) -> Result<DroneConfig, Vec<ConfigError>> {
    let mut config = DroneConfig::default();
    let mut errors = Vec::new();
    let mut found_keys: HashMap<String, bool> = HashMap::new();
    
    let valid_keys = ["name", "id", "max_altitude", "max_speed", "battery_threshold"];
    
    for line in text.lines() {
        match parse_line(line) {
            Ok((key, value)) => {
                if key.is_empty() {
                    continue;
                }
                
                if !valid_keys.contains(&key.as_str()) {
                    errors.push(ConfigError::UnknownKey(key.clone()));
                    continue;
                }
                
                found_keys.insert(key.clone(), true);
                
                match key.as_str() {
                    "name" => config.name = value,
                    "id" => {
                        match value.parse::<u32>() {
                            Ok(v) => config.id = v,
                            Err(_) => errors.push(ConfigError::InvalidValue {
                                key: key.clone(),
                                value: value.clone(),
                                expected: String::from("ціле число"),
                            }),
                        }
                    },
                    "max_altitude" => {
                        match value.parse::<i32>() {
                            Ok(v) if v > 0 && v <= 1000 => config.max_altitude = v,
                            Ok(v) => errors.push(ConfigError::InvalidValue {
                                key: key.clone(),
                                value: v.to_string(),
                                expected: String::from("число від 1 до 1000"),
                            }),
                            Err(_) => errors.push(ConfigError::InvalidValue {
                                key: key.clone(),
                                value: value.clone(),
                                expected: String::from("ціле число"),
                            }),
                        }
                    },
                    "max_speed" => {
                        match value.parse::<f64>() {
                            Ok(v) if v > 0.0 => config.max_speed = v,
                            _ => errors.push(ConfigError::InvalidValue {
                                key: key.clone(),
                                value: value.clone(),
                                expected: String::from("додатне число"),
                            }),
                        }
                    },
                    "battery_threshold" => {
                        match value.parse::<i32>() {
                            Ok(v) if v >= 0 && v <= 100 => config.battery_threshold = v,
                            _ => errors.push(ConfigError::InvalidValue {
                                key: key.clone(),
                                value: value.clone(),
                                expected: String::from("число від 0 до 100"),
                            }),
                        }
                    },
                    _ => {}
                }
            },
            Err(e) => errors.push(e),
        }
    }
    
    // Перевірка обов'язкових полів
    for required in ["name", "id"] {
        if !found_keys.contains_key(required) {
            errors.push(ConfigError::MissingRequired(required.to_string()));
        }
    }
    
    if errors.is_empty() {
        Ok(config)
    } else {
        Err(errors)
    }
}

fn main() {
    println!("=== Парсер конфігурації ===\n");
    
    let valid_config = r#"
# Конфігурація дрона
name=Alpha-1
id=42
max_altitude=150
max_speed=25.5
battery_threshold=20
"#;

    let invalid_config = r#"
name=
id=not_a_number
max_altitude=9999
unknown_key=value
missing separator
battery_threshold=-50
"#;

    println!("--- Валідна конфігурація ---");
    match parse_config(valid_config) {
        Ok(config) => println!("✓ {:?}", config),
        Err(errors) => {
            println!("✗ Помилки:");
            for e in errors {
                println!("  • {}", e);
            }
        }
    }
    
    println!("\n--- Невалідна конфігурація ---");
    match parse_config(invalid_config) {
        Ok(config) => println!("✓ {:?}", config),
        Err(errors) => {
            println!("✗ Помилки ({}):", errors.len());
            for e in errors {
                println!("  • {}", e);
            }
        }
    }
}
```

---

### Завдання 2: Retry з експоненційною затримкою

**Умова:** Реалізуйте функцію, що повторює операцію N разів з експоненційно зростаючою затримкою між спробами. Зберігайте всі помилки.

**Розв'язання:**

```rust
use std::thread;
use std::time::Duration;

#[derive(Debug)]
struct RetryError {
    attempts: Vec<(u32, String)>,  // (номер спроби, помилка)
    total_time_ms: u64,
}

impl std::fmt::Display for RetryError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        writeln!(f, "Всі {} спроб провалились ({}мс):", self.attempts.len(), self.total_time_ms)?;
        for (attempt, error) in &self.attempts {
            writeln!(f, "  Спроба {}: {}", attempt, error)?;
        }
        Ok(())
    }
}

struct RetryConfig {
    max_attempts: u32,
    initial_delay_ms: u64,
    max_delay_ms: u64,
    multiplier: f64,
}

impl Default for RetryConfig {
    fn default() -> Self {
        RetryConfig {
            max_attempts: 3,
            initial_delay_ms: 100,
            max_delay_ms: 5000,
            multiplier: 2.0,
        }
    }
}

fn retry_with_backoff<T, E, F>(
    config: &RetryConfig,
    mut operation: F,
) -> Result<T, RetryError>
where
    F: FnMut(u32) -> Result<T, E>,
    E: std::fmt::Display,
{
    let mut attempts = Vec::new();
    let mut current_delay = config.initial_delay_ms;
    let mut total_time = 0u64;
    
    for attempt in 1..=config.max_attempts {
        match operation(attempt) {
            Ok(result) => return Ok(result),
            Err(e) => {
                attempts.push((attempt, e.to_string()));
                
                if attempt < config.max_attempts {
                    println!("    Спроба {} провалилась, очікування {}мс...", attempt, current_delay);
                    
                    // Симуляція затримки (у реальному коді: thread::sleep)
                    total_time += current_delay;
                    
                    current_delay = ((current_delay as f64) * config.multiplier) as u64;
                    current_delay = current_delay.min(config.max_delay_ms);
                }
            }
        }
    }
    
    Err(RetryError {
        attempts,
        total_time_ms: total_time,
    })
}

// Симуляція нестабільної операції
fn unstable_connect(drone_id: &str, fail_until: u32) -> impl FnMut(u32) -> Result<String, String> {
    let drone_id = drone_id.to_string();
    let mut attempt_count = 0;
    
    move |attempt| {
        attempt_count += 1;
        
        if attempt_count < fail_until {
            Err(format!("Timeout connecting to {}", drone_id))
        } else {
            Ok(format!("Connected to {}", drone_id))
        }
    }
}

fn main() {
    println!("=== Retry з експоненційною затримкою ===\n");
    
    let config = RetryConfig {
        max_attempts: 5,
        initial_delay_ms: 100,
        max_delay_ms: 2000,
        multiplier: 2.0,
    };
    
    // Тест 1: Успіх на 3-й спробі
    println!("--- Тест 1: Успіх на 3-й спробі ---");
    let operation = unstable_connect("Alpha", 3);
    match retry_with_backoff(&config, operation) {
        Ok(result) => println!("✓ {}", result),
        Err(e) => println!("✗ {}", e),
    }
    
    // Тест 2: Успіх на 1-й спробі
    println!("\n--- Тест 2: Успіх одразу ---");
    let operation = unstable_connect("Beta", 1);
    match retry_with_backoff(&config, operation) {
        Ok(result) => println!("✓ {}", result),
        Err(e) => println!("✗ {}", e),
    }
    
    // Тест 3: Всі спроби провалились
    println!("\n--- Тест 3: Всі спроби провалились ---");
    let operation = unstable_connect("Gamma", 10);  // Ніколи не вдасться
    match retry_with_backoff(&config, operation) {
        Ok(result) => println!("✓ {}", result),
        Err(e) => println!("✗ {}", e),
    }
}
```

---

### Завдання 3: Result з контекстом

**Умова:** Реалізуйте extension trait для Result, що додає метод `context(msg)` — обгортає помилку з додатковим контекстом (подібно до anyhow).

**Розв'язання:**

```rust
use std::fmt;

#[derive(Debug)]
struct ContextError {
    context: String,
    source: Box<dyn std::error::Error + Send + Sync>,
}

impl fmt::Display for ContextError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.context)?;
        
        // Друкуємо ланцюжок причин
        let mut current: Option<&(dyn std::error::Error + 'static)> = Some(&*self.source);
        while let Some(err) = current {
            write!(f, "\n  Причина: {}", err)?;
            current = err.source();
        }
        
        Ok(())
    }
}

impl std::error::Error for ContextError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        Some(&*self.source)
    }
}

// Extension trait для Result
trait ResultExt<T, E> {
    fn context(self, msg: &str) -> Result<T, ContextError>
    where
        E: std::error::Error + Send + Sync + 'static;
    
    fn with_context<F>(self, f: F) -> Result<T, ContextError>
    where
        E: std::error::Error + Send + Sync + 'static,
        F: FnOnce() -> String;
}

impl<T, E> ResultExt<T, E> for Result<T, E> {
    fn context(self, msg: &str) -> Result<T, ContextError>
    where
        E: std::error::Error + Send + Sync + 'static,
    {
        self.map_err(|e| ContextError {
            context: msg.to_string(),
            source: Box::new(e),
        })
    }
    
    fn with_context<F>(self, f: F) -> Result<T, ContextError>
    where
        E: std::error::Error + Send + Sync + 'static,
        F: FnOnce() -> String,
    {
        self.map_err(|e| ContextError {
            context: f(),
            source: Box::new(e),
        })
    }
}

// Приклад використання
#[derive(Debug)]
struct ParseError(String);

impl fmt::Display for ParseError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "Parse error: {}", self.0)
    }
}

impl std::error::Error for ParseError {}

fn parse_coordinate(s: &str) -> Result<i32, ParseError> {
    s.trim()
        .parse::<i32>()
        .map_err(|_| ParseError(format!("'{}' is not a valid integer", s)))
}

fn load_waypoint(line: &str, line_num: usize) -> Result<(i32, i32), ContextError> {
    let parts: Vec<&str> = line.split(',').collect();
    
    if parts.len() != 2 {
        return Err(ContextError {
            context: format!("Рядок {}: очікується формат 'x,y'", line_num),
            source: Box::new(ParseError(format!("got {} parts", parts.len()))),
        });
    }
    
    let x = parse_coordinate(parts[0])
        .with_context(|| format!("Рядок {}: невалідна координата X", line_num))?;
    
    let y = parse_coordinate(parts[1])
        .with_context(|| format!("Рядок {}: невалідна координата Y", line_num))?;
    
    Ok((x, y))
}

fn load_route(text: &str) -> Result<Vec<(i32, i32)>, ContextError> {
    let mut waypoints = Vec::new();
    
    for (i, line) in text.lines().enumerate() {
        let line = line.trim();
        if line.is_empty() || line.starts_with('#') {
            continue;
        }
        
        let point = load_waypoint(line, i + 1)
            .context("Помилка завантаження маршруту")?;
        
        waypoints.push(point);
    }
    
    if waypoints.is_empty() {
        return Err(ContextError {
            context: String::from("Маршрут порожній"),
            source: Box::new(ParseError(String::from("no waypoints found"))),
        });
    }
    
    Ok(waypoints)
}

fn main() {
    println!("=== Result з контекстом ===\n");
    
    let valid_route = r#"
# Маршрут патрулювання
100, 50
200, 100
150, 200
0, 0
"#;

    let invalid_route = r#"
100, 50
abc, 200
300, xyz
"#;

    println!("--- Валідний маршрут ---");
    match load_route(valid_route) {
        Ok(waypoints) => {
            println!("✓ Завантажено {} точок:", waypoints.len());
            for (i, (x, y)) in waypoints.iter().enumerate() {
                println!("    {}. ({}, {})", i + 1, x, y);
            }
        },
        Err(e) => println!("✗ {}", e),
    }
    
    println!("\n--- Невалідний маршрут ---");
    match load_route(invalid_route) {
        Ok(waypoints) => println!("✓ {:?}", waypoints),
        Err(e) => println!("✗ {}", e),
    }
}
```

---

### Завдання 4: Агрегація помилок

**Умова:** Реалізуйте функцію, що виконує операцію для списку елементів і збирає всі успіхи та помилки окремо, не зупиняючись на першій помилці.

**Розв'язання:**

```rust
#[derive(Debug)]
struct BatchResult<T, E> {
    successes: Vec<(usize, T)>,
    failures: Vec<(usize, E)>,
}

impl<T, E> BatchResult<T, E> {
    fn new() -> Self {
        BatchResult {
            successes: Vec::new(),
            failures: Vec::new(),
        }
    }
    
    fn is_complete_success(&self) -> bool {
        self.failures.is_empty()
    }
    
    fn success_count(&self) -> usize {
        self.successes.len()
    }
    
    fn failure_count(&self) -> usize {
        self.failures.len()
    }
    
    fn total(&self) -> usize {
        self.successes.len() + self.failures.len()
    }
    
    fn success_rate(&self) -> f64 {
        if self.total() == 0 {
            0.0
        } else {
            (self.success_count() as f64 / self.total() as f64) * 100.0
        }
    }
}

fn process_batch<T, R, E, F>(items: &[T], mut operation: F) -> BatchResult<R, E>
where
    F: FnMut(&T) -> Result<R, E>,
{
    let mut result = BatchResult::new();
    
    for (i, item) in items.iter().enumerate() {
        match operation(item) {
            Ok(value) => result.successes.push((i, value)),
            Err(error) => result.failures.push((i, error)),
        }
    }
    
    result
}

// Приклад: валідація та обробка команд дронів
#[derive(Debug, Clone)]
struct DroneCommand {
    drone_id: String,
    command: String,
    param: i32,
}

#[derive(Debug)]
enum CommandError {
    InvalidDrone(String),
    InvalidCommand(String),
    InvalidParam(i32),
}

impl std::fmt::Display for CommandError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            CommandError::InvalidDrone(id) => write!(f, "Невідомий дрон: {}", id),
            CommandError::InvalidCommand(cmd) => write!(f, "Невідома команда: {}", cmd),
            CommandError::InvalidParam(p) => write!(f, "Невалідний параметр: {}", p),
        }
    }
}

fn execute_command(cmd: &DroneCommand) -> Result<String, CommandError> {
    // Валідація дрона
    let valid_drones = ["Alpha", "Beta", "Gamma"];
    if !valid_drones.contains(&cmd.drone_id.as_str()) {
        return Err(CommandError::InvalidDrone(cmd.drone_id.clone()));
    }
    
    // Валідація команди
    let valid_commands = ["MOVE", "HOVER", "LAND", "PHOTO"];
    if !valid_commands.contains(&cmd.command.as_str()) {
        return Err(CommandError::InvalidCommand(cmd.command.clone()));
    }
    
    // Валідація параметра
    if cmd.param < 0 || cmd.param > 1000 {
        return Err(CommandError::InvalidParam(cmd.param));
    }
    
    Ok(format!("[{}] {} ({})", cmd.drone_id, cmd.command, cmd.param))
}

fn main() {
    println!("=== Агрегація помилок ===\n");
    
    let commands = vec![
        DroneCommand { drone_id: String::from("Alpha"), command: String::from("MOVE"), param: 100 },
        DroneCommand { drone_id: String::from("Unknown"), command: String::from("MOVE"), param: 50 },
        DroneCommand { drone_id: String::from("Beta"), command: String::from("INVALID"), param: 200 },
        DroneCommand { drone_id: String::from("Gamma"), command: String::from("HOVER"), param: 30 },
        DroneCommand { drone_id: String::from("Alpha"), command: String::from("PHOTO"), param: -5 },
        DroneCommand { drone_id: String::from("Beta"), command: String::from("LAND"), param: 0 },
    ];
    
    let result = process_batch(&commands, execute_command);
    
    println!("═══ Результати обробки ═══\n");
    println!("Всього команд: {}", result.total());
    println!("Успішних: {}", result.success_count());
    println!("Провалених: {}", result.failure_count());
    println!("Успішність: {:.1}%\n", result.success_rate());
    
    if !result.successes.is_empty() {
        println!("✓ Успішно виконані:");
        for (i, msg) in &result.successes {
            println!("    [{}] {}", i, msg);
        }
    }
    
    if !result.failures.is_empty() {
        println!("\n✗ Помилки:");
        for (i, err) in &result.failures {
            println!("    [{}] {}", i, err);
        }
    }
    
    // Перевірка чи всі успішні
    if result.is_complete_success() {
        println!("\n✓ Всі команди виконано успішно!");
    } else {
        println!("\n⚠ Деякі команди провалились");
    }
}
```

---

## Підсумок заняття

На цьому занятті ви навчились:

1. **Розуміти Result<T, E>**: Ok, Err, методи, комбінатори
2. **Використовувати оператор ?**: елегантний проброс помилок
3. **Створювати власні типи помилок**: enum, Display, Error, From
4. **Обирати стратегію обробки**: паніка, проброс, fallback, логування
5. **Працювати з контекстом помилок**: додавання інформації

---

## Перевірте себе

1. Чим `Result` відрізняється від `Option`?
2. Що робить оператор `?`?
3. Коли доречно використовувати `unwrap()`?
4. Навіщо реалізовувати `From` для типу помилки?
5. Як додати контекст до помилки?
6. Що краще: одна помилка чи список помилок?

**Відповіді:**
1. `Option` — є/немає значення; `Result` — успіх/помилка з інформацією
2. Повертає `Err` з функції, якщо `Result` є `Err`; розгортає `Ok`
3. У тестах, прототипах, коли впевнені що не буде помилки
4. Для автоматичної конвертації при використанні `?`
5. Через `map_err()` або власний extension trait
6. Залежить: одна — для fail-fast; список — для валідації всіх полів

---

## Наступне заняття

На наступному занятті ми вивчимо **Traits та Generics** — як створювати абстракції та писати узагальнений код, що працює з різними типами.
