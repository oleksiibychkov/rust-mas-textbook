# Розділ 26: Серіалізація та serde

## Вступ

Агент БПЛА не існує ізольовано. Він **завантажує конфігурацію** з файлу при старті, **надсилає телеметрію** на базову станцію, **зберігає звіт** про місію після завершення, **обмінюється повідомленнями** з іншими агентами в рої. Все це потребує **серіалізації** — перетворення структур даних у формат, який можна зберегти на диск або передати по мережі.

Що таке серіалізація? Це процес перетворення "живої" структури в пам'яті у послідовність байтів або текст:

```
┌─────────────────────────────────────────────────────────────────┐
│  Rust структура              →   JSON/TOML/bytes               │
│  ┌──────────────────┐            ┌──────────────────────────┐  │
│  │ Agent {          │            │ {"id":"A-001",           │  │
│  │   id: "A-001",   │ serialize  │  "x":10.5,               │  │
│  │   x: 10.5,       │ ─────────► │  "y":20.3}               │  │
│  │   y: 20.3,       │            └──────────────────────────┘  │
│  │ }                │                                          │
│  └──────────────────┘                                          │
└─────────────────────────────────────────────────────────────────┘
```

**Десеріалізація** — зворотний процес: з байтів/тексту відновити структуру.

**Serde** (SERialize/DEserialize) — де-факто стандарт серіалізації в Rust. Це не просто бібліотека — це **екосистема**. Додавши `#[derive(Serialize, Deserialize)]` до структури, ви автоматично отримуєте можливість конвертувати її в JSON, TOML, YAML, MessagePack та десятки інших форматів.

---

## 26.1 Архітектура serde

### Розділення відповідальностей

Serde має геніальну архітектуру — вона розділяє дві речі:

1. **Формат даних** (JSON, TOML, YAML, bincode) — це окремі крейти
2. **Типи даних** (ваші структури) — derive макроси Serialize/Deserialize

Це означає: один раз додавши `Serialize` до структури, ви можете серіалізувати її в *будь-який* підтримуваний формат без зміни коду.

```
Ваші типи ────► Serde Data Model ────► Формати
   │                  │                   │
 Agent              Map                 JSON
 Position           Seq                 TOML
 Vec<T>             String              YAML
                                       bincode
```

### Додавання залежностей

**Постановка задачі:** Налаштувати проєкт для використання serde з JSON та TOML.

```toml
# Cargo.toml
[dependencies]
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"    # Для JSON
toml = "0.8"          # Для TOML
```

**Як це працює:**

1. `serde` — базовий крейт з трейтами Serialize та Deserialize
2. `features = ["derive"]` — вмикає derive макроси
3. `serde_json` — формат JSON, окремий крейт
4. `toml` — формат TOML, окремий крейт

---

## 26.2 Derive Serialize та Deserialize

### Базове використання

Найпростіший спосіб зробити структуру серіалізованою — додати derive макроси.

**Постановка задачі:** Створити структуру `Point`, яку можна конвертувати в JSON та назад.

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
struct Point {
    x: f64,
    y: f64,
}

fn main() {
    // Створюємо структуру
    let point = Point { x: 10.5, y: 20.3 };
    
    // Серіалізація в JSON (структура → текст)
    let json = serde_json::to_string(&point).unwrap();
    println!("JSON: {}", json);
    // Вивід: {"x":10.5,"y":20.3}
    
    // Десеріалізація з JSON (текст → структура)
    let restored: Point = serde_json::from_str(&json).unwrap();
    println!("Restored: {:?}", restored);
    // Вивід: Point { x: 10.5, y: 20.3 }
}
```

**Як це працює:**

1. `#[derive(Serialize)]` — генерує код для перетворення структури в дані
2. `#[derive(Deserialize)]` — генерує код для відновлення структури з даних
3. `serde_json::to_string(&point)` — конвертує Point у JSON-рядок
4. `serde_json::from_str(&json)` — парсить JSON-рядок у Point
5. Типи полів (f64) автоматично конвертуються

### Вкладені структури

Serde рекурсивно серіалізує вкладені структури. Головне — щоб усі вкладені типи теж реалізовували Serialize/Deserialize.

**Постановка задачі:** Створити структуру агента з вкладеною позицією.

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
struct Position {
    x: f64,
    y: f64,
    z: f64,
}

#[derive(Debug, Serialize, Deserialize)]
struct Agent {
    id: String,
    position: Position,  // Вкладена структура
    battery: u8,
    active: bool,
}

fn main() {
    let agent = Agent {
        id: "SCOUT-001".to_string(),
        position: Position { x: 100.0, y: 50.0, z: 30.0 },
        battery: 85,
        active: true,
    };
    
    // to_string_pretty — з відступами для читабельності
    let json = serde_json::to_string_pretty(&agent).unwrap();
    println!("{}", json);
}
```

**Вивід:**
```json
{
  "id": "SCOUT-001",
  "position": {
    "x": 100.0,
    "y": 50.0,
    "z": 30.0
  },
  "battery": 85,
  "active": true
}
```

### Колекції

Serde автоматично працює з `Vec`, `HashMap`, `Option` та іншими стандартними типами.

**Постановка задачі:** Структура місії з вейпоінтами та метаданими.

```rust
use serde::{Serialize, Deserialize};
use std::collections::HashMap;

#[derive(Debug, Serialize, Deserialize)]
struct Position {
    x: f64,
    y: f64,
}

#[derive(Debug, Serialize, Deserialize)]
struct Mission {
    name: String,
    waypoints: Vec<Position>,           // Вектор
    metadata: HashMap<String, String>,  // HashMap
    priority: Option<u8>,               // Option
}

fn main() {
    let mut metadata = HashMap::new();
    metadata.insert("created_by".to_string(), "Operator-1".to_string());
    metadata.insert("zone".to_string(), "Alpha".to_string());
    
    let mission = Mission {
        name: "Patrol-01".to_string(),
        waypoints: vec![
            Position { x: 0.0, y: 0.0 },
            Position { x: 100.0, y: 0.0 },
            Position { x: 100.0, y: 100.0 },
        ],
        metadata,
        priority: Some(1),
    };
    
    let json = serde_json::to_string_pretty(&mission).unwrap();
    println!("{}", json);
}
```

### Enum — машина станів

Enum серіалізується разом з даними кожного варіанту.

**Постановка задачі:** Серіалізувати стан агента, включаючи дані кожного стану.

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
enum AgentState {
    Idle,                                    // Unit variant
    Moving { target_x: f64, target_y: f64 }, // Struct variant
    Scanning(f64),                           // Tuple variant (радіус)
    Emergency(String),                       // Tuple variant (причина)
}

#[derive(Debug, Serialize, Deserialize)]
struct Agent {
    id: String,
    state: AgentState,
}

fn main() {
    let agents = vec![
        Agent { id: "A1".to_string(), state: AgentState::Idle },
        Agent { 
            id: "A2".to_string(), 
            state: AgentState::Moving { target_x: 100.0, target_y: 50.0 } 
        },
        Agent { id: "A3".to_string(), state: AgentState::Scanning(25.0) },
        Agent { id: "A4".to_string(), state: AgentState::Emergency("Low battery".to_string()) },
    ];
    
    let json = serde_json::to_string_pretty(&agents).unwrap();
    println!("{}", json);
}
```

**Вивід (за замовчуванням — externally tagged):**
```json
[
  {"id": "A1", "state": "Idle"},
  {"id": "A2", "state": {"Moving": {"target_x": 100.0, "target_y": 50.0}}},
  {"id": "A3", "state": {"Scanning": 25.0}},
  {"id": "A4", "state": {"Emergency": "Low battery"}}
]
```

---

## 26.3 Робота з JSON

### Серіалізація — різні способи

**Постановка задачі:** Показати всі способи серіалізації в JSON.

```rust
use serde::Serialize;
use std::fs::File;

#[derive(Serialize)]
struct Data {
    name: String,
    value: i32,
}

fn main() {
    let data = Data { 
        name: "test".to_string(), 
        value: 42 
    };
    
    // 1. В String — компактний формат
    let json_string = serde_json::to_string(&data).unwrap();
    println!("Compact: {}", json_string);
    // {"name":"test","value":42}
    
    // 2. В String — з форматуванням (pretty)
    let json_pretty = serde_json::to_string_pretty(&data).unwrap();
    println!("Pretty:\n{}", json_pretty);
    
    // 3. В Vec<u8> — байти
    let json_bytes = serde_json::to_vec(&data).unwrap();
    println!("Bytes: {} bytes", json_bytes.len());
    
    // 4. Безпосередньо у файл
    let file = File::create("data.json").unwrap();
    serde_json::to_writer(file, &data).unwrap();
    
    // 5. У файл з форматуванням
    let file = File::create("data_pretty.json").unwrap();
    serde_json::to_writer_pretty(file, &data).unwrap();
    
    println!("Files saved!");
}
```

### Десеріалізація — різні способи

**Постановка задачі:** Показати всі способи десеріалізації з JSON.

```rust
use serde::Deserialize;
use std::fs::File;

#[derive(Debug, Deserialize)]
struct Data {
    name: String,
    value: i32,
}

fn main() {
    // 1. З рядка (&str)
    let json = r#"{"name": "test", "value": 42}"#;
    let data: Data = serde_json::from_str(json).unwrap();
    println!("From str: {:?}", data);
    
    // 2. З байтів (&[u8])
    let bytes = json.as_bytes();
    let data: Data = serde_json::from_slice(bytes).unwrap();
    println!("From slice: {:?}", data);
    
    // 3. З файлу
    let file = File::open("data.json").unwrap();
    let data: Data = serde_json::from_reader(file).unwrap();
    println!("From file: {:?}", data);
}
```

### Обробка помилок десеріалізації

Десеріалізація може провалитись з багатьох причин. Важливо правильно обробляти помилки.

**Постановка задачі:** Показати різні типи помилок та їх обробку.

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct Config {
    port: u16,
    host: String,
}

fn load_config(json: &str) -> Result<Config, String> {
    serde_json::from_str(json)
        .map_err(|e| format!("Помилка парсингу: {} (рядок {}, колонка {})", 
                             e, e.line(), e.column()))
}

fn main() {
    // 1. Валідний JSON
    let valid = r#"{"port": 8080, "host": "localhost"}"#;
    match load_config(valid) {
        Ok(config) => println!("✅ Завантажено: {:?}", config),
        Err(e) => println!("❌ {}", e),
    }
    
    // 2. Синтаксична помилка — пропущені лапки
    let syntax_error = r#"{"port": 8080, host: "localhost"}"#;
    match load_config(syntax_error) {
        Ok(config) => println!("✅ Завантажено: {:?}", config),
        Err(e) => println!("❌ {}", e),
    }
    // ❌ Помилка парсингу: key must be a string at line 1 column 17
    
    // 3. Помилка типу — рядок замість числа
    let type_error = r#"{"port": "not a number", "host": "localhost"}"#;
    match load_config(type_error) {
        Ok(config) => println!("✅ Завантажено: {:?}", config),
        Err(e) => println!("❌ {}", e),
    }
    // ❌ Помилка парсингу: invalid type: string "not a number", expected u16
    
    // 4. Відсутнє поле
    let missing_field = r#"{"host": "localhost"}"#;
    match load_config(missing_field) {
        Ok(config) => println!("✅ Завантажено: {:?}", config),
        Err(e) => println!("❌ {}", e),
    }
    // ❌ Помилка парсингу: missing field `port`
}
```

---

## 26.4 TOML для конфігурації

### Чому TOML?

**TOML** (Tom's Obvious Minimal Language) — формат, спеціально розроблений для конфігураційних файлів:

| Характеристика | TOML | JSON |
|----------------|------|------|
| Коментарі | ✅ Підтримує | ❌ Не підтримує |
| Читабельність | Відмінна | Добра |
| Багаторядкові рядки | ✅ | ❌ |
| Типізація | Явна (дати, часи) | Неявна |

**Факт:** Cargo.toml — саме TOML! Rust-спільнота обрала цей формат для конфігурації.

### Приклад конфігураційного файлу

**Постановка задачі:** Створити конфігурацію агента у форматі TOML.

```toml
# agent_config.toml

# Базові налаштування агента
id = "SCOUT-001"
name = "Primary Scout"

# Параметри польоту
[flight]
max_altitude = 500.0
max_speed = 25.0
min_battery = 20

# Налаштування сенсорів
[sensors]
gps_enabled = true
camera_resolution = "1080p"
scan_interval_ms = 500

# Waypoints місії — масив таблиць
[[waypoints]]
x = 100.0
y = 0.0
label = "Start"

[[waypoints]]
x = 100.0
y = 100.0

[[waypoints]]
x = 0.0
y = 100.0
label = "Finish"
```

### Завантаження TOML

**Постановка задачі:** Завантажити конфігурацію з TOML-файлу.

```rust
use serde::Deserialize;
use std::fs;

#[derive(Debug, Deserialize)]
struct AgentConfig {
    id: String,
    name: String,
    flight: FlightConfig,
    sensors: SensorConfig,
    waypoints: Vec<Waypoint>,
}

#[derive(Debug, Deserialize)]
struct FlightConfig {
    max_altitude: f64,
    max_speed: f64,
    min_battery: u8,
}

#[derive(Debug, Deserialize)]
struct SensorConfig {
    gps_enabled: bool,
    camera_resolution: String,
    scan_interval_ms: u32,
}

#[derive(Debug, Deserialize)]
struct Waypoint {
    x: f64,
    y: f64,
    label: Option<String>,  // Необов'язкове поле
}

fn load_config(path: &str) -> Result<AgentConfig, Box<dyn std::error::Error>> {
    let content = fs::read_to_string(path)?;
    let config: AgentConfig = toml::from_str(&content)?;
    Ok(config)
}

fn main() {
    match load_config("agent_config.toml") {
        Ok(config) => {
            println!("Агент: {} ({})", config.name, config.id);
            println!("Макс. висота: {}м", config.flight.max_altitude);
            println!("GPS: {}", if config.sensors.gps_enabled { "увімкнено" } else { "вимкнено" });
            println!("Waypoints: {}", config.waypoints.len());
            
            for (i, wp) in config.waypoints.iter().enumerate() {
                let label = wp.label.as_deref().unwrap_or("без назви");
                println!("  {}: ({}, {}) - {}", i + 1, wp.x, wp.y, label);
            }
        }
        Err(e) => println!("Помилка завантаження: {}", e),
    }
}
```

---

## 26.5 Атрибути serde — налаштування серіалізації

### Перейменування полів

Часто імена полів у Rust (snake_case) не відповідають іменам у JSON (camelCase).

**Постановка задачі:** Серіалізувати структуру з іменами полів у camelCase для JavaScript API.

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
struct Agent {
    #[serde(rename = "agentId")]       // Явне перейменування
    id: String,
    
    #[serde(rename = "batteryLevel")]
    battery: u8,
    
    #[serde(rename = "isActive")]
    active: bool,
}

fn main() {
    let agent = Agent {
        id: "A-001".to_string(),
        battery: 85,
        active: true,
    };
    
    let json = serde_json::to_string(&agent).unwrap();
    println!("{}", json);
    // {"agentId":"A-001","batteryLevel":85,"isActive":true}
    
    // Десеріалізація теж працює з новими іменами
    let json_input = r#"{"agentId":"B-002","batteryLevel":50,"isActive":false}"#;
    let restored: Agent = serde_json::from_str(json_input).unwrap();
    println!("{:?}", restored);
    // Agent { id: "B-002", battery: 50, active: false }
}
```

### Стратегії перейменування для всієї структури

Замість перейменування кожного поля окремо, можна застосувати стратегію до всіх полів.

```rust
use serde::{Serialize, Deserialize};

// Всі поля автоматично в camelCase
#[derive(Debug, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
struct ApiResponse {
    agent_id: String,        // → "agentId"
    battery_level: u8,       // → "batteryLevel"
    is_active: bool,         // → "isActive"
    last_update_time: u64,   // → "lastUpdateTime"
}

fn main() {
    let response = ApiResponse {
        agent_id: "A-001".to_string(),
        battery_level: 85,
        is_active: true,
        last_update_time: 1705312800,
    };
    
    println!("{}", serde_json::to_string(&response).unwrap());
    // {"agentId":"A-001","batteryLevel":85,"isActive":true,"lastUpdateTime":1705312800}
}
```

**Доступні стратегії:**
- `camelCase` — agentId
- `snake_case` — agent_id
- `SCREAMING_SNAKE_CASE` — AGENT_ID
- `kebab-case` — agent-id
- `PascalCase` — AgentId

### Значення за замовчуванням

Дозволяє десеріалізувати структуру, навіть якщо деякі поля відсутні.

**Постановка задачі:** Конфігурація з опціональними полями, що мають значення за замовчуванням.

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
struct Config {
    name: String,  // Обов'язкове
    
    #[serde(default)]  // false за замовчуванням (Default::default())
    enabled: bool,
    
    #[serde(default = "default_timeout")]  // Власна функція
    timeout_ms: u32,
    
    #[serde(default = "default_retries")]
    max_retries: u8,
}

fn default_timeout() -> u32 {
    5000  // 5 секунд
}

fn default_retries() -> u8 {
    3
}

fn main() {
    // Мінімальний JSON — тільки обов'язкове поле
    let minimal = r#"{"name": "TestConfig"}"#;
    let config: Config = serde_json::from_str(minimal).unwrap();
    
    println!("{:?}", config);
    // Config { name: "TestConfig", enabled: false, timeout_ms: 5000, max_retries: 3 }
    
    // Повний JSON — перевизначає defaults
    let full = r#"{"name": "Full", "enabled": true, "timeout_ms": 10000, "max_retries": 5}"#;
    let config: Config = serde_json::from_str(full).unwrap();
    
    println!("{:?}", config);
    // Config { name: "Full", enabled: true, timeout_ms: 10000, max_retries: 5 }
}
```

### Пропуск полів

Деякі поля не повинні серіалізуватись (внутрішній стан, паролі).

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
struct User {
    name: String,
    
    #[serde(skip)]  // Не серіалізувати І не десеріалізувати
    internal_counter: u32,
    
    #[serde(skip_serializing)]  // Не серіалізувати, але десеріалізувати
    password_hash: String,
    
    #[serde(skip_serializing_if = "Option::is_none")]  // Пропустити якщо None
    nickname: Option<String>,
}

fn main() {
    let user = User {
        name: "Alice".to_string(),
        internal_counter: 42,
        password_hash: "abc123hash".to_string(),
        nickname: None,
    };
    
    let json = serde_json::to_string(&user).unwrap();
    println!("{}", json);
    // {"name":"Alice"}
    // internal_counter — skip
    // password_hash — skip_serializing
    // nickname — skip_serializing_if None
}
```

### Flatten — "розплющення" вкладених структур

Атрибут `flatten` дозволяє "вбудувати" поля вкладеної структури на верхній рівень.

**Постановка задачі:** Метадані повинні бути на одному рівні з основними полями.

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
struct Metadata {
    created_at: String,
    updated_at: String,
}

#[derive(Debug, Serialize, Deserialize)]
struct Agent {
    id: String,
    name: String,
    
    #[serde(flatten)]  // "Вбудувати" поля Metadata сюди
    metadata: Metadata,
}

fn main() {
    let agent = Agent {
        id: "A-001".to_string(),
        name: "Scout".to_string(),
        metadata: Metadata {
            created_at: "2024-01-15T10:00:00Z".to_string(),
            updated_at: "2024-01-15T12:30:00Z".to_string(),
        },
    };
    
    println!("{}", serde_json::to_string_pretty(&agent).unwrap());
}
```

**Вивід з flatten:**
```json
{
  "id": "A-001",
  "name": "Scout",
  "created_at": "2024-01-15T10:00:00Z",
  "updated_at": "2024-01-15T12:30:00Z"
}
```

**Без flatten було б:**
```json
{
  "id": "A-001",
  "name": "Scout",
  "metadata": {
    "created_at": "2024-01-15T10:00:00Z",
    "updated_at": "2024-01-15T12:30:00Z"
  }
}
```

---

## 26.6 Серіалізація enum — різні стратегії

### Externally tagged (за замовчуванням)

```rust
#[derive(Serialize)]
enum Command {
    MoveTo { x: f64, y: f64 },
    Scan { radius: f64 },
    Land,
}
```
**Результат:** `{"MoveTo": {"x": 100.0, "y": 50.0}}`

### Internally tagged

Тег всередині об'єкта — зручніше для багатьох API.

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
#[serde(tag = "type")]  // Тег буде полем "type"
enum Command {
    MoveTo { x: f64, y: f64 },
    Scan { radius: f64 },
    Land,
}

fn main() {
    let cmd = Command::MoveTo { x: 100.0, y: 50.0 };
    println!("{}", serde_json::to_string(&cmd).unwrap());
    // {"type":"MoveTo","x":100.0,"y":50.0}
    
    // Десеріалізація
    let json = r#"{"type":"Scan","radius":25.0}"#;
    let cmd: Command = serde_json::from_str(json).unwrap();
    println!("{:?}", cmd);
    // Scan { radius: 25.0 }
}
```

### Adjacently tagged

Тег і дані в окремих полях.

```rust
#[derive(Serialize, Deserialize)]
#[serde(tag = "type", content = "data")]
enum Event {
    Created { id: String },
    Updated { id: String, changes: Vec<String> },
    Deleted { id: String },
}
```
**Результат:** `{"type": "Created", "data": {"id": "A-001"}}`

---

## 26.7 Практичний приклад: система повідомлень агентів

**Постановка задачі:** Створити систему повідомлень між агентами в рої — статуси, виявлення, команди.

```rust
use serde::{Serialize, Deserialize};

// Позиція — спільний тип
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Position {
    pub x: f64,
    pub y: f64,
    pub z: f64,
}

// Типи команд
#[derive(Debug, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum CommandType {
    MoveTo { x: f64, y: f64 },
    Scan { radius: f64 },
    Return,
    EmergencyLand,
}

// Головний enum повідомлень — internally tagged
#[derive(Debug, Serialize, Deserialize)]
#[serde(tag = "type")]
pub enum AgentMessage {
    #[serde(rename = "status")]
    Status {
        agent_id: String,
        position: Position,
        battery: u8,
        state: String,
    },
    
    #[serde(rename = "detection")]
    Detection {
        agent_id: String,
        position: Position,
        object_type: String,
        confidence: f64,
    },
    
    #[serde(rename = "command")]
    Command {
        target_id: String,
        command: CommandType,
    },
    
    #[serde(rename = "ack")]
    Acknowledgment {
        message_id: String,
        success: bool,
        #[serde(skip_serializing_if = "Option::is_none")]
        error: Option<String>,
    },
}

fn main() {
    // Статус агента
    let status = AgentMessage::Status {
        agent_id: "SCOUT-001".to_string(),
        position: Position { x: 100.0, y: 50.0, z: 30.0 },
        battery: 85,
        state: "flying".to_string(),
    };
    
    println!("Status:\n{}\n", serde_json::to_string_pretty(&status).unwrap());
    
    // Виявлення об'єкта
    let detection = AgentMessage::Detection {
        agent_id: "SCOUT-001".to_string(),
        position: Position { x: 150.0, y: 75.0, z: 0.0 },
        object_type: "vehicle".to_string(),
        confidence: 0.92,
    };
    
    println!("Detection:\n{}\n", serde_json::to_string_pretty(&detection).unwrap());
    
    // Команда
    let command = AgentMessage::Command {
        target_id: "SCOUT-002".to_string(),
        command: CommandType::MoveTo { x: 200.0, y: 100.0 },
    };
    
    println!("Command:\n{}\n", serde_json::to_string_pretty(&command).unwrap());
    
    // Десеріалізація
    let json = r#"{"type":"ack","message_id":"msg-123","success":true}"#;
    let msg: AgentMessage = serde_json::from_str(json).unwrap();
    println!("Parsed: {:?}", msg);
}
```

---

## 26.8 Лабораторна робота

**Завдання:** Реалізувати систему конфігурації та обміну даними для агента.

### Частина 1: Конфігурація з TOML (3 бали)

Створіть структури та функцію завантаження:

```rust
struct AgentConfig {
    id: String,
    name: String,
    flight: FlightParams,    // max_altitude, max_speed, battery_threshold
    sensors: SensorParams,   // gps_enabled, scan_interval_ms
    waypoints: Vec<Waypoint>,
}

fn load_config(path: &str) -> Result<AgentConfig, Box<dyn Error>>;
```

Використайте `#[serde(default)]` для опціональних полів.

### Частина 2: Повідомлення в JSON (4 бали)

```rust
#[serde(tag = "type")]
enum Message {
    StatusUpdate { agent_id: String, battery: u8, position: Position },
    Command { target_id: String, action: String },
    Alert { agent_id: String, severity: String, message: String },
}

fn serialize_message(msg: &Message) -> String;
fn deserialize_message(json: &str) -> Result<Message, Error>;
```

### Частина 3: Звіт про місію (3 бали)

```rust
struct MissionReport {
    mission_id: String,
    agent_id: String,
    start_time: String,
    end_time: String,
    status: MissionStatus,  // Completed, Aborted, Failed
    statistics: MissionStats,
    events: Vec<MissionEvent>,
}

fn save_report(report: &MissionReport, path: &str) -> Result<(), Error>;
fn load_report(path: &str) -> Result<MissionReport, Error>;
```

**Критерії оцінювання:**

| Критерій | Бали |
|----------|------|
| Конфігурація з defaults | 3 |
| Повідомлення з tagged enum | 4 |
| Звіт з подіями | 3 |
| **Максимум** | **10** |

---

## Підсумок

Serde — потужна екосистема для серіалізації:

| Концепція | Призначення |
|-----------|-------------|
| **Serialize/Deserialize** | Derive макроси для типів |
| **serde_json** | JSON формат |
| **toml** | TOML для конфігурації |
| **#[serde(rename)]** | Перейменування полів |
| **#[serde(default)]** | Значення за замовчуванням |
| **#[serde(skip)]** | Пропуск полів |
| **#[serde(flatten)]** | "Розплющення" структури |
| **#[serde(tag)]** | Internally tagged enum |

**Коли який формат:**
- **TOML** — конфігураційні файли (підтримує коментарі)
- **JSON** — API, обмін даними між системами
- **bincode** — компактний бінарний формат для мережі

---

## Зв'язок з наступним розділом

Ви завершили **Частину II: Робастність, дані та інструментарій**. Агент тепер:
- Обробляє помилки gracefully
- Логує свої дії
- Має типобезпечні абстракції
- Серіалізує та десеріалізує дані

У **Розділі 27: Практикум — Робастний агент v2.0** ви об'єднаєте все вивчене в цілісну систему.
