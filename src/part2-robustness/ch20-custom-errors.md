# Розділ 20: Власні типи помилок та thiserror

## Вступ

У попередніх розділах ви використовували прості типи для помилок: рядки, базові enum'и, `Box<dyn Error>`. Для навчальних прикладів це працює, але уявіть production систему:

```text
Error: something went wrong
```

Така помилка нічого не каже оператору. Що сталось? Де? Чому? Що з цим робити? Коли агент БПЛА втрачає зв'язок посеред місії, повідомлення "Error: connection failed" — катастрофічно неінформативне.

Професійні типи помилок у Rust:
- Реалізують trait `std::error::Error` для стандартного інтерфейсу
- Мають структуровані дані про помилку (не просто текст)
- Підтримують ланцюжки причин (source chain)
- Дозволяють pattern matching для різних стратегій обробки

Крейт **thiserror** спрощує створення таких типів через derive макроси — замість десятків рядків boilerplate пишете кілька атрибутів.

---

## 20.1 Trait std::error::Error

### Що таке "повноцінна" помилка

У Rust помилкою вважається будь-який тип, що реалізує trait `Error`:

```rust
// Спрощене визначення з стандартної бібліотеки
pub trait Error: Debug + Display {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        None
    }
}
```

**Три вимоги для повноцінної помилки:**

1. **Debug** — для розробників (детальний формат: `{:?}`)
2. **Display** — для користувачів (гарний формат: `{}`)
3. **Error** — для стандартного інтерфейсу та екосистеми

Метод `source()` опційний — він повертає "причину" помилки, якщо вона є. Це дозволяє будувати ланцюжки: "Помилка A спричинена помилкою B, яка спричинена помилкою C".

### Реалізація Error вручну

Перш ніж вивчати thiserror, важливо зрозуміти, що відбувається "під капотом". Ця програма демонструє створення власного типу помилки вручну — без derive макросів.

**Постановка задачі:** Створити тип помилки для сенсора, який зберігає ID сенсора та повідомлення, і реалізує всі необхідні traits.

```rust
use std::error::Error;
use std::fmt;

/// Помилка сенсора — зберігає контекст
#[derive(Debug)]  // Debug можна derive'ити
struct SensorError {
    sensor_id: String,
    message: String,
}

// Display реалізуємо вручну — контролюємо формат для користувача
impl fmt::Display for SensorError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "Sensor '{}' error: {}", self.sensor_id, self.message)
    }
}

// Error реалізуємо вручну — базова версія без source
impl Error for SensorError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        None  // Ця помилка не має "причини"
    }
}

fn read_sensor(id: &str) -> Result<f64, SensorError> {
    match id {
        "GPS" => Ok(50.45),
        "TEMP" => Ok(23.5),
        "broken" => Err(SensorError {
            sensor_id: id.to_string(),
            message: "Connection timeout".to_string(),
        }),
        _ => Err(SensorError {
            sensor_id: id.to_string(),
            message: "Unknown sensor".to_string(),
        }),
    }
}

fn main() {
    let sensors = ["GPS", "TEMP", "broken", "UNKNOWN"];
    
    for id in sensors {
        match read_sensor(id) {
            Ok(value) => println!("✅ {}: {}", id, value),
            Err(e) => {
                // Display — для користувача
                println!("❌ {}", e);
                // Debug — для розробника
                println!("   Debug: {:?}", e);
            }
        }
    }
}
```

**Як це працює:**

1. `#[derive(Debug)]` автоматично генерує Debug реалізацію
2. `impl Display` — ми самі вирішуємо, як форматувати помилку для користувача
3. `impl Error` — робить тип "офіційною" помилкою Rust
4. При виведенні `{}` викликає Display, `{:?}` — Debug

### Помилка з причиною (source)

Часто помилка виникає через іншу помилку. Наприклад, "не вдалося прочитати конфігурацію" → "не вдалося відкрити файл" → "file not found". Метод `source()` дозволяє зберегти цей ланцюжок.

**Постановка задачі:** Створити помилку читання файлу, яка зберігає оригінальну `io::Error` як причину.

```rust
use std::error::Error;
use std::fmt;
use std::io;

/// Помилка читання файлу — має причину (source)
#[derive(Debug)]
struct FileReadError {
    path: String,
    source: io::Error,  // Оригінальна помилка
}

impl fmt::Display for FileReadError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "Failed to read file '{}'", self.path)
    }
}

impl Error for FileReadError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        Some(&self.source)  // Повертаємо причину
    }
}

fn read_config(path: &str) -> Result<String, FileReadError> {
    std::fs::read_to_string(path).map_err(|e| FileReadError {
        path: path.to_string(),
        source: e,
    })
}

/// Виводить повний ланцюжок помилок
fn print_error_chain(error: &dyn Error) {
    println!("Error: {}", error);
    
    let mut current = error.source();
    let mut level = 1;
    
    while let Some(cause) = current {
        println!("{}Caused by: {}", "  ".repeat(level), cause);
        current = cause.source();
        level += 1;
    }
}

fn main() {
    match read_config("nonexistent_config.toml") {
        Ok(content) => println!("Config: {}", content),
        Err(e) => print_error_chain(&e),
    }
}
```

**Як це працює:**

1. `FileReadError` зберігає `io::Error` у полі `source`
2. `source()` повертає `Some(&self.source)` — вказівник на причину
3. `print_error_chain` проходить по ланцюжку через `while let`
4. Кожен рівень вкладеності показує наступну причину

**Вивід:**
```rust
Error: Failed to read file 'nonexistent_config.toml'
  Caused by: No such file or directory (os error 2)
```

---

## 20.2 Проблема boilerplate

### Чому ручна реалізація незручна

Подивіться, скільки коду потрібно для простої помилки з кількома варіантами:

```rust
use std::error::Error;
use std::fmt;
use std::io;
use std::num::ParseIntError;

#[derive(Debug)]
pub enum ConfigError {
    Io { path: String, source: io::Error },
    Parse { key: String, value: String, source: ParseIntError },
    MissingKey { key: String },
    InvalidValue { key: String, value: String, reason: String },
}

// Display — 15+ рядків
impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ConfigError::Io { path, .. } => {
                write!(f, "Failed to read config file '{}'", path)
            }
            ConfigError::Parse { key, value, .. } => {
                write!(f, "Failed to parse '{}' for key '{}'", value, key)
            }
            ConfigError::MissingKey { key } => {
                write!(f, "Missing required key '{}'", key)
            }
            ConfigError::InvalidValue { key, value, reason } => {
                write!(f, "Invalid value '{}' for key '{}': {}", value, key, reason)
            }
        }
    }
}

// Error — ще 10+ рядків
impl Error for ConfigError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            ConfigError::Io { source, .. } => Some(source),
            ConfigError::Parse { source, .. } => Some(source),
            _ => None,
        }
    }
}

// From реалізації — ще код...
```

Для кожного enum з 4-5 варіантами — 30-40 рядків boilerplate. А якщо треба змінити формат повідомлення? Шукати потрібний match arm серед десятків рядків.

---

## 20.3 thiserror — derive для помилок

### Що таке thiserror

**thiserror** — крейт, що генерує всі ці реалізації автоматично через derive макроси:

```toml
# Cargo.toml
[dependencies]
thiserror = "1.0"
```

### Базове використання

Та сама `ConfigError`, але з thiserror:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ConfigError {
    #[error("Failed to read config file '{path}'")]
    Io {
        path: String,
        #[source]
        source: std::io::Error,
    },
    
    #[error("Failed to parse '{value}' for key '{key}'")]
    Parse {
        key: String,
        value: String,
        #[source]
        source: std::num::ParseIntError,
    },
    
    #[error("Missing required key '{key}'")]
    MissingKey { key: String },
    
    #[error("Invalid value '{value}' for key '{key}': {reason}")]
    InvalidValue {
        key: String,
        value: String,
        reason: String,
    },
}
```

**Що робить thiserror:**

1. `#[derive(Error)]` — генерує `impl Error`
2. `#[error("...")]` — генерує `impl Display` з вказаним форматом
3. `#[source]` — вказує поле для методу `source()`

**Переваги:**
- 15 рядків замість 50+
- Повідомлення поруч з варіантом — легко читати і змінювати
- Менше можливостей для помилок

### Атрибут #[from] — автоматична конвертація

Ця програма демонструє `#[from]` — атрибут, що автоматично генерує `impl From<T>` для використання з оператором `?`.

```rust
use thiserror::Error;
use std::io;
use std::num::ParseIntError;

#[derive(Debug, Error)]
pub enum AppError {
    // #[from] генерує impl From<io::Error> for AppError
    #[error("IO error")]
    Io(#[from] io::Error),
    
    // #[from] генерує impl From<ParseIntError> for AppError
    #[error("Parse error")]
    Parse(#[from] ParseIntError),
    
    // Варіант без #[from] — ручне створення
    #[error("Validation failed: {0}")]
    Validation(String),
}

fn read_and_parse(path: &str) -> Result<i32, AppError> {
    // io::Error автоматично конвертується через #[from]
    let content = std::fs::read_to_string(path)?;
    
    // ParseIntError автоматично конвертується через #[from]
    let number: i32 = content.trim().parse()?;
    
    // Валідацію створюємо вручну
    if number < 0 {
        return Err(AppError::Validation("Number must be non-negative".to_string()));
    }
    
    Ok(number)
}

fn main() {
    match read_and_parse("config.txt") {
        Ok(n) => println!("Value: {}", n),
        Err(e) => println!("Error: {}", e),
    }
}
```

**Як це працює:**

1. `#[from]` на полі генерує `impl From<io::Error> for AppError`
2. Оператор `?` автоматично викликає `.into()` при помилці
3. Конвертація відбувається прозоро — код залишається чистим

### Різниця між #[source] та #[from]

| Атрибут | Що генерує | Коли використовувати |
|---------|-----------|---------------------|
| `#[source]` | Метод `source()` | Коли потрібен ланцюжок причин |
| `#[from]` | `impl From<T>` + `source()` | Коли потрібна автоматична конвертація для `?` |

```rust
use thiserror::Error;
use std::io;

#[derive(Debug, Error)]
pub enum MyError {
    // Тільки source — потрібно вручну створювати
    #[error("IO error with path '{path}'")]
    IoWithPath {
        path: String,
        #[source]
        source: io::Error,
    },
    
    // From — автоматична конвертація, але без додаткових полів
    #[error("IO error")]
    IoAuto(#[from] io::Error),
}
```

**Обмеження #[from]:** можна використати тільки для tuple variants з одним полем. Якщо потрібні додаткові поля (як `path`), використовуйте `#[source]` і створюйте помилку вручну.

---

## 20.4 Форматування повідомлень

### Синтаксис #[error("...")]

thiserror підтримує багатий синтаксис форматування:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum FormatExamples {
    // Простий літерал
    #[error("Simple error message")]
    Simple,
    
    // Tuple variant — поля за індексом
    #[error("Error with value: {0}")]
    WithIndex(i32),
    
    // Struct variant — поля за назвою
    #[error("Error at position ({x}, {y})")]
    WithNames { x: i32, y: i32 },
    
    // Форматування чисел
    #[error("Value {value:.2} is out of range [{min}..{max}]")]
    OutOfRange { value: f64, min: f64, max: f64 },
    
    // Виклик методу на полі
    #[error("String length is {}", .0.len())]
    WithMethod(String),
    
    // Множинне форматування
    #[error("{count} error(s) occurred in '{context}'")]
    Multiple { count: usize, context: String },
}

fn main() {
    let errors = vec![
        FormatExamples::Simple,
        FormatExamples::WithIndex(42),
        FormatExamples::WithNames { x: 10, y: 20 },
        FormatExamples::OutOfRange { value: 3.14159, min: 0.0, max: 1.0 },
        FormatExamples::WithMethod("hello world".to_string()),
        FormatExamples::Multiple { count: 5, context: "validation".to_string() },
    ];
    
    for e in errors {
        println!("{}", e);
    }
}
```

**Вивід:**
```text
Simple error message
Error with value: 42
Error at position (10, 20)
Value 3.14 is out of range [0..1]
String length is 11
5 error(s) occurred in 'validation'
```

### #[error(transparent)]

Іноді потрібно "прозоро" делегувати Display та source внутрішній помилці:

```rust
use thiserror::Error;
use std::io;

#[derive(Debug, Error)]
pub enum AppError {
    // transparent — Display та source беруться з внутрішньої помилки
    #[error(transparent)]
    Io(#[from] io::Error),
    
    // Звичайний варіант з власним повідомленням
    #[error("Application error: {0}")]
    App(String),
}

fn main() {
    // Створюємо io::Error
    let io_err = io::Error::new(io::ErrorKind::NotFound, "file not found");
    
    // Конвертуємо в AppError
    let app_err: AppError = io_err.into();
    
    // Display виведе оригінальне повідомлення io::Error
    println!("{}", app_err);  // "file not found"
}
```

**Коли використовувати transparent:**
- Обгортка над іншою помилкою без додаткового контексту
- "Перекидання" помилки на вищий рівень без зміни повідомлення

---

## 20.5 Ієрархія помилок

### Навіщо ієрархія

Великі системи мають багато джерел помилок:
- Hardware: сенсори, мотори, батарея
- Navigation: шляхи, межі, GPS
- Communication: мережа, протоколи
- Command: валідація, стан

Плоский enum з 20+ варіантами — кошмар для підтримки. Краще організувати в ієрархію:

```text
MissionError (верхній рівень)
├── Hardware
│   ├── SensorOffline
│   ├── MotorFailure
│   └── BatteryFault
├── Navigation
│   ├── NoPath
│   ├── OutOfBounds
│   └── GpsLost
└── Command
    ├── Rejected
    └── Timeout
```

### Реалізація ієрархії

Ця програма демонструє трирівневу ієрархію помилок для агента БПЛА.

```rust
use thiserror::Error;
use std::fmt;

// ═══════════════════════════════════════════════════════════════════════════
// ДОПОМІЖНІ ТИПИ
// ═══════════════════════════════════════════════════════════════════════════

#[derive(Debug, Clone)]
pub struct Position {
    pub x: f64,
    pub y: f64,
    pub z: f64,
}

impl fmt::Display for Position {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({:.1}, {:.1}, {:.1})", self.x, self.y, self.z)
    }
}

// ═══════════════════════════════════════════════════════════════════════════
// РІВЕНЬ 1: НИЗЬКОРІВНЕВІ ПОМИЛКИ
// ═══════════════════════════════════════════════════════════════════════════

/// Помилки апаратного забезпечення
#[derive(Debug, Error)]
pub enum HardwareError {
    #[error("Sensor '{sensor_id}' is offline")]
    SensorOffline { sensor_id: String },
    
    #[error("Motor {motor_id} failure: {reason}")]
    MotorFailure { motor_id: u8, reason: String },
    
    #[error("Battery fault: {details}")]
    BatteryFault { details: String },
    
    #[error("Communication lost with {component}")]
    CommunicationLost { component: String },
}

/// Помилки навігації
#[derive(Debug, Error)]
pub enum NavigationError {
    #[error("No path found from {from} to {to}")]
    NoPath { from: Position, to: Position },
    
    #[error("Path blocked by obstacle at {position}")]
    PathBlocked { position: Position },
    
    #[error("Position {position} is outside boundary")]
    OutOfBounds { position: Position },
    
    #[error("GPS signal lost, last known: {last_known}")]
    GpsLost { last_known: Position },
}

/// Помилки команд
#[derive(Debug, Error)]
pub enum CommandError {
    #[error("Command rejected: {reason}")]
    Rejected { reason: String },
    
    #[error("Command timeout after {timeout_ms}ms")]
    Timeout { timeout_ms: u64 },
    
    #[error("Invalid state for command: {state}")]
    InvalidState { state: String },
}

// ═══════════════════════════════════════════════════════════════════════════
// РІВЕНЬ 2: ПОМИЛКА МІСІЇ (АГРЕГУЄ ВСЕ)
// ═══════════════════════════════════════════════════════════════════════════

/// Фаза місії — для контексту
#[derive(Debug, Clone)]
pub enum MissionPhase {
    Preflight,
    Takeoff,
    Cruise,
    Waypoint(usize),
    Return,
    Landing,
}

impl fmt::Display for MissionPhase {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            MissionPhase::Preflight => write!(f, "Preflight"),
            MissionPhase::Takeoff => write!(f, "Takeoff"),
            MissionPhase::Cruise => write!(f, "Cruise"),
            MissionPhase::Waypoint(n) => write!(f, "Waypoint {}", n),
            MissionPhase::Return => write!(f, "Return"),
            MissionPhase::Landing => write!(f, "Landing"),
        }
    }
}

/// Головна помилка місії
#[derive(Debug, Error)]
pub enum MissionError {
    #[error("Hardware failure during {phase}")]
    Hardware {
        phase: MissionPhase,
        #[source]
        source: HardwareError,
    },
    
    #[error("Navigation error during {phase}")]
    Navigation {
        phase: MissionPhase,
        #[source]
        source: NavigationError,
    },
    
    #[error("Command error during {phase}")]
    Command {
        phase: MissionPhase,
        #[source]
        source: CommandError,
    },
    
    #[error("Battery critical: {level}% (minimum: {required}%)")]
    BatteryCritical { level: u8, required: u8 },
    
    #[error("Mission aborted: {reason}")]
    Aborted { reason: String },
}

// ═══════════════════════════════════════════════════════════════════════════
// МЕТОДИ ПОМИЛКИ
// ═══════════════════════════════════════════════════════════════════════════

impl MissionError {
    /// Конструктор для hardware помилок
    pub fn hardware(phase: MissionPhase, source: HardwareError) -> Self {
        MissionError::Hardware { phase, source }
    }
    
    /// Конструктор для navigation помилок
    pub fn navigation(phase: MissionPhase, source: NavigationError) -> Self {
        MissionError::Navigation { phase, source }
    }
    
    /// Чи можна відновитись після цієї помилки?
    pub fn is_recoverable(&self) -> bool {
        match self {
            MissionError::BatteryCritical { .. } => false,
            MissionError::Aborted { .. } => false,
            MissionError::Hardware { source, .. } => {
                // Відмова мотора — невідновлювана
                !matches!(source, HardwareError::MotorFailure { .. })
            }
            _ => true,
        }
    }
    
    /// Рекомендована дія
    pub fn suggested_action(&self) -> &'static str {
        match self {
            MissionError::BatteryCritical { .. } => {
                "Initiate immediate return to base"
            }
            MissionError::Hardware { source: HardwareError::SensorOffline { .. }, .. } => {
                "Switch to backup sensor or manual mode"
            }
            MissionError::Navigation { source: NavigationError::PathBlocked { .. }, .. } => {
                "Calculate alternative route"
            }
            MissionError::Navigation { source: NavigationError::GpsLost { .. }, .. } => {
                "Switch to inertial navigation"
            }
            _ => "Retry or abort mission"
        }
    }
}

// ═══════════════════════════════════════════════════════════════════════════
// ВИВЕДЕННЯ ЛАНЦЮЖКА ПОМИЛОК
// ═══════════════════════════════════════════════════════════════════════════

fn print_error_report(error: &MissionError) {
    use std::error::Error;
    
    println!("╔══════════════════════════════════════════════════════════════╗");
    println!("║                    MISSION ERROR REPORT                      ║");
    println!("╚══════════════════════════════════════════════════════════════╝");
    println!();
    println!("Error: {}", error);
    
    // Виводимо ланцюжок причин
    if let Some(source) = error.source() {
        println!();
        println!("Cause chain:");
        let mut current: Option<&(dyn Error + 'static)> = Some(source);
        let mut level = 1;
        while let Some(err) = current {
            println!("  {}. {}", level, err);
            current = err.source();
            level += 1;
        }
    }
    
    println!();
    println!("Recoverable: {}", if error.is_recoverable() { "Yes" } else { "No" });
    println!("Action: {}", error.suggested_action());
}

fn main() {
    println!("=== Демонстрація ієрархії помилок ===\n");
    
    // Приклад 1: Hardware → Mission
    let hw_error = HardwareError::SensorOffline {
        sensor_id: "GPS-PRIMARY".to_string(),
    };
    let mission_error = MissionError::hardware(MissionPhase::Cruise, hw_error);
    print_error_report(&mission_error);
    
    println!("\n{}\n", "═".repeat(66));
    
    // Приклад 2: Navigation → Mission
    let nav_error = NavigationError::PathBlocked {
        position: Position { x: 150.0, y: 200.0, z: 50.0 },
    };
    let mission_error = MissionError::navigation(MissionPhase::Waypoint(3), nav_error);
    print_error_report(&mission_error);
    
    println!("\n{}\n", "═".repeat(66));
    
    // Приклад 3: Battery critical
    let battery_error = MissionError::BatteryCritical {
        level: 15,
        required: 20,
    };
    print_error_report(&battery_error);
}
```

**Як це працює:**

1. **Три рівні:** HardwareError, NavigationError, CommandError → MissionError
2. **Контекст:** MissionPhase показує, де саме сталась помилка
3. **#[source]:** зберігає ланцюжок причин
4. **Методи:** `is_recoverable()`, `suggested_action()` додають логіку до помилки
5. **print_error_report:** виводить повну діагностику

---

## 20.6 anyhow — для застосунків

### thiserror vs anyhow

**thiserror** — для бібліотек:
- Конкретні типи помилок
- Pattern matching по варіантах
- Повний контроль

**anyhow** — для застосунків:
- Швидке прототипування
- Легке додавання контексту
- Менше boilerplate

```toml
# Cargo.toml
[dependencies]
anyhow = "1.0"
thiserror = "1.0"  # Часто використовуються разом
```

### Основні можливості anyhow

Ця програма демонструє головні функції anyhow: context, bail, ensure, anyhow!.

```rust
use anyhow::{Result, Context, bail, ensure, anyhow};

#[derive(Debug)]
struct Config {
    host: String,
    port: u16,
}

fn parse_config(content: &str) -> Result<Config> {
    // anyhow! — створює помилку з повідомленням
    if content.is_empty() {
        return Err(anyhow!("Config content is empty"));
    }
    
    // Симуляція парсингу
    let port: u16 = "8080".parse()
        .context("Failed to parse port")?;  // Додає контекст
    
    // bail! — негайно повертає Err
    if port == 0 {
        bail!("Port cannot be zero");
    }
    
    // ensure! — перевірка умови
    ensure!(port < 65535, "Port {} is out of range", port);
    
    Ok(Config {
        host: "localhost".to_string(),
        port,
    })
}

fn load_config(path: &str) -> Result<Config> {
    // context додає контекст до будь-якої помилки
    let content = std::fs::read_to_string(path)
        .context("Failed to read config file")?;
    
    // with_context — ліниве обчислення контексту
    let config = parse_config(&content)
        .with_context(|| format!("Failed to parse config from '{}'", path))?;
    
    Ok(config)
}

fn main() -> Result<()> {
    match load_config("config.toml") {
        Ok(config) => {
            println!("Loaded: {:?}", config);
        }
        Err(e) => {
            // Повний ланцюжок помилок
            println!("Error: {:#}", e);
            
            // Або по кроках
            println!("\nError chain:");
            for (i, cause) in e.chain().enumerate() {
                println!("  {}: {}", i, cause);
            }
        }
    }
    
    Ok(())
}
```

**Ключові функції:**

- `anyhow!("...")` — створює помилку з повідомленням
- `bail!("...")` — одразу повертає Err (скорочення для `return Err(anyhow!(...))`)
- `ensure!(condition, "...")` — повертає Err якщо умова false
- `.context("...")` — додає контекст до помилки
- `.with_context(|| ...)` — ліниве обчислення контексту

### Комбінування thiserror та anyhow

Найкращий підхід: thiserror для типізованих помилок бібліотеки, anyhow для застосунку.

```rust
use thiserror::Error;
use anyhow::{Result, Context};

// Бібліотечний код — конкретні типи
#[derive(Debug, Error)]
pub enum SensorError {
    #[error("Sensor '{id}' is offline")]
    Offline { id: String },
    
    #[error("Invalid reading: {value}")]
    InvalidReading { value: f64 },
}

// Бібліотечна функція
pub fn read_sensor(id: &str) -> std::result::Result<f64, SensorError> {
    if id == "broken" {
        Err(SensorError::Offline { id: id.to_string() })
    } else {
        Ok(42.0)
    }
}

// Код застосунку — anyhow
fn run_mission() -> Result<()> {
    // SensorError автоматично конвертується в anyhow::Error
    let temp = read_sensor("temp")
        .context("Reading temperature sensor")?;
    
    let humidity = read_sensor("broken")
        .context("Reading humidity sensor")?;
    
    println!("Temp: {}, Humidity: {}", temp, humidity);
    Ok(())
}

fn main() {
    if let Err(e) = run_mission() {
        println!("Mission failed:");
        for (i, cause) in e.chain().enumerate() {
            println!("  {}: {}", i, cause);
        }
    }
}
```

**Вивід:**
```text
Mission failed:
  0: Reading humidity sensor
  1: Sensor 'broken' is offline
```

---

## 20.7 Best practices

### Правила проектування помилок

**1. Конкретні варіанти з контекстом:**

```rust
// ✅ ДОБРЕ — зрозуміло що, де, чому
#[error("Connection to {host}:{port} failed after {attempts} attempts")]
ConnectionFailed { host: String, port: u16, attempts: u32 }

// ❌ ПОГАНО — нічого не каже
#[error("Network error")]
Network
```

**2. Ієрархія для великих систем:**

```rust
// ✅ ДОБРЕ — організовано по категоріях
MissionError::Hardware { source: HardwareError::... }
MissionError::Navigation { source: NavigationError::... }

// ❌ ПОГАНО — плоский enum на 20 варіантів
enum Error { 
    SensorOffline, MotorFailed, BatteryLow, 
    NoPath, OutOfBounds, GpsLost, 
    Timeout, Rejected, Invalid, 
    // ще 10 варіантів...
}
```

**3. Методи для роботи з помилками:**

```rust
impl MissionError {
    pub fn is_recoverable(&self) -> bool { ... }
    pub fn error_code(&self) -> u32 { ... }
    pub fn suggested_action(&self) -> &str { ... }
}
```

### Коли що використовувати

| Ситуація | Рішення |
|----------|---------|
| Бібліотека | `thiserror` з конкретними типами |
| Застосунок | `anyhow` з context |
| Швидкий прототип | `anyhow` або `Box<dyn Error>` |
| Критична система | `thiserror` з повною ієрархією |
| CLI утиліта | `anyhow` + pretty printing |

---

## 20.8 Лабораторна робота

**Завдання:** Створити повну систему помилок для агента БПЛА.

### Частина 1: Базові помилки (3 бали)

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum SensorError {
    // Мінімум 4 варіанти з інформативними повідомленнями
}

#[derive(Debug, Error)]
pub enum CommunicationError {
    // Мінімум 3 варіанти
}
```

### Частина 2: Ієрархія (4 бали)

```rust
#[derive(Debug, Error)]
pub enum AgentError {
    // Обгортки з #[source]
    #[error("Sensor failure during {phase}")]
    Sensor {
        phase: String,
        #[source]
        source: SensorError,
    },
    
    // Власні варіанти
    #[error("Battery low: {level}% (required: {required}%)")]
    BatteryLow { level: u8, required: u8 },
}

impl AgentError {
    pub fn is_critical(&self) -> bool { ... }
    pub fn recovery_action(&self) -> &str { ... }
}
```

### Частина 3: Інтеграція (3 бали)

```rust
fn execute_mission(agent: &mut Agent) -> Result<Report, AgentError> {
    // Використовуйте ?, обробляйте різні помилки
}
```

**Критерії оцінювання:**

| Критерій | Бали |
|----------|------|
| Базові помилки з thiserror | 3 |
| Ієрархія з #[source] | 4 |
| Інтеграція та методи | 3 |
| **Максимум** | **10** |

---

## Підсумок

Професійні типи помилок у Rust:

**Ручна реалізація:**
- Потрібно: Debug, Display, Error
- source() для ланцюжків причин
- Багато boilerplate

**thiserror:**
- `#[derive(Error)]` генерує все автоматично
- `#[error("...")]` для Display
- `#[source]` для причин
- `#[from]` для автоматичної конвертації

**anyhow:**
- Для застосунків, не бібліотек
- `context()` додає контекст
- `bail!`, `ensure!` для швидких помилок

**Ієрархія:**
- Низький рівень: HardwareError, NavigationError
- Високий рівень: MissionError агрегує все
- Методи: is_recoverable(), suggested_action()

---

## Зв'язок з наступним розділом

Помилки — це діагностика проблем. Але що відбувається до помилки? Як відстежити, що агент робив у кожен момент часу?

У **Розділі 21: Логування та tracing** ви навчитесь:
- Налаштовувати структуроване логування
- Використовувати spans для контексту операцій
- Інструментувати функції автоматично через `#[instrument]`
- Фільтрувати логи за рівнем та модулем
