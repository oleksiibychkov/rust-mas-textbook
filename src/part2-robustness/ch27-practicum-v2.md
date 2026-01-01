# Розділ 27: Практикум — Робастний агент v2.0

## Вступ: від фрагментів до цілісної системи

Ви пройшли велику подорож через Частину II. Навчились працювати з колекціями (Vec, HashMap), обробляти помилки (Result, Option, thiserror), логувати події (tracing), створювати абстракції через traits та generics, серіалізувати дані (serde). Кожен з цих інструментів потужний сам по собі, але справжня сила проявляється коли вони працюють разом.

Цей розділ — **інтеграційний практикум**. Ви побудуєте повноцінного агента, який:

```text
┌─────────────────────────────────────────────────────────────────┐
│                    РОБАСТНИЙ АГЕНТ v2.0                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  📄 config.toml  ────►  🤖 AGENT  ────►  📊 report.json        │
│                              │                                  │
│         ┌────────────────────┼────────────────────┐            │
│         ▼                    ▼                    ▼            │
│    ┌─────────┐         ┌─────────┐          ┌─────────┐        │
│    │Navigator│         │ Sensors │          │ Logger  │        │
│    └─────────┘         └─────────┘          └─────────┘        │
│                                                                 │
│  ✓ Завантажує конфігурацію з TOML                              │
│  ✓ Gracefully обробляє всі помилки                             │
│  ✓ Логує свої дії структуровано                                │
│  ✓ Зберігає детальний звіт у JSON                              │
│  ✓ Має модульну архітектуру з traits                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

Порівняйте з агентом v1.0 з Розділу 15:

| Аспект | v1.0 (Розділ 15) | v2.0 (Цей розділ) |
|--------|------------------|-------------------|
| Конфігурація | Hardcoded у коді | TOML файл |
| Помилки | `panic!`, `unwrap()` | Graceful handling |
| Логування | `println!()` | Структуроване tracing |
| Архітектура | Монолітний файл | Модулі + traits |
| Результати | Тільки консоль | JSON звіти |

---

## 27.1 Структура проєкту

### Модульна організація

**Чому модулі?** Кожен модуль відповідає за одну чітко визначену задачу. Це спрощує розуміння, тестування та підтримку коду.

```text
agent_v2/
├── Cargo.toml           # Залежності проєкту
├── config.toml          # Конфігурація агента
├── src/
│   ├── main.rs          # Точка входу, ініціалізація
│   ├── config.rs        # Завантаження та валідація конфігурації
│   ├── error.rs         # Ієрархія типів помилок
│   ├── types.rs         # Базові типи (Position, AgentState)
│   ├── agent.rs         # Головна структура агента
│   ├── navigator.rs     # Навігація та планування шляху
│   ├── sensors.rs       # Сенсори та виявлення
│   └── report.rs        # Генерація звітів
└── mission_report.json  # Результат місії (генерується)
```

### Залежності

**Постановка задачі:** Налаштувати Cargo.toml з усіма необхідними крейтами.

```toml
# Cargo.toml
[package]
name = "agent_v2"
version = "0.1.0"
edition = "2021"

[dependencies]
# Серіалізація (Розділ 26)
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
toml = "0.8"

# Обробка помилок (Розділ 20)
thiserror = "1.0"

# Логування (Розділ 21)
tracing = "0.1"
tracing-subscriber = "0.3"
```

**Як це працює:**

1. `serde` з `features = ["derive"]` — для `#[derive(Serialize, Deserialize)]`
2. `serde_json` — JSON для звітів та API
3. `toml` — TOML для конфігурації (як Cargo.toml!)
4. `thiserror` — derive макроси для типів помилок
5. `tracing` + `tracing-subscriber` — структуроване логування

---

## 27.2 Базові типи (types.rs)

Почнемо з фундаменту — типів, що використовуються по всьому проєкту. Це "словник" нашої предметної області.

### Позиція агента

**Постановка задачі:** Створити тип `Position` для представлення координат у 3D-просторі з методами обчислення відстані та зручним виводом.

```rust
// src/types.rs
use serde::{Serialize, Deserialize};
use std::fmt;

/// Позиція агента в 3D просторі.
/// 
/// Координати x та y — горизонтальна площина,
/// z — висота над землею.
#[derive(Debug, Clone, Copy, PartialEq, Default, Serialize, Deserialize)]
pub struct Position {
    pub x: f64,
    pub y: f64,
    pub z: f64,
}

impl Position {
    /// Створює нову позицію.
    pub fn new(x: f64, y: f64, z: f64) -> Self {
        Self { x, y, z }
    }
    
    /// Обчислює відстань до іншої позиції (Евклідова відстань у 3D).
    pub fn distance_to(&self, other: &Position) -> f64 {
        let dx = self.x - other.x;
        let dy = self.y - other.y;
        let dz = self.z - other.z;
        (dx * dx + dy * dy + dz * dz).sqrt()
    }
    
    /// Позиція бази (початок координат).
    pub fn base() -> Self {
        Self::default()
    }
}

// Красивий вивід для користувача
impl fmt::Display for Position {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({:.1}, {:.1}, {:.1})", self.x, self.y, self.z)
    }
}
```

**Що застосовано з попередніх розділів:**

| Derive | Розділ | Призначення |
|--------|--------|-------------|
| `Debug` | 22 | Вивід для налагодження (`{:?}`) |
| `Clone, Copy` | 22 | Копіювання без `.clone()` |
| `PartialEq` | 22 | Порівняння `==` |
| `Default` | 22 | `Position::default()` = (0, 0, 0) |
| `Serialize, Deserialize` | 26 | JSON/TOML |
| `Display` | 22 | Вивід для користувача (`{}`) |

### Стан агента

**Постановка задачі:** Enum для машини станів агента з підтримкою серіалізації.

```rust
// продовження src/types.rs

/// Можливі стани агента під час місії.
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]  // Idle → "idle" у JSON
pub enum AgentState {
    /// Очікування на базі
    Idle,
    /// Політ до цілі
    Flying,
    /// Сканування території
    Scanning,
    /// Повернення на базу
    Returning,
    /// Приземлився
    Landed,
    /// Аварійний стан з причиною
    Emergency(String),
}

impl Default for AgentState {
    fn default() -> Self {
        Self::Idle  // Агент стартує в стані очікування
    }
}

impl fmt::Display for AgentState {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AgentState::Idle => write!(f, "Очікування"),
            AgentState::Flying => write!(f, "Політ"),
            AgentState::Scanning => write!(f, "Сканування"),
            AgentState::Returning => write!(f, "Повернення"),
            AgentState::Landed => write!(f, "Приземлився"),
            AgentState::Emergency(reason) => write!(f, "Аварія: {}", reason),
        }
    }
}
```

**Як це працює:**

1. `#[serde(rename_all = "snake_case")]` — `Emergency` стає `"emergency"` у JSON
2. `Default` повертає `Idle` — природний початковий стан
3. `Display` — людиночитабельний вивід українською
4. `Emergency(String)` — зберігає причину аварії

---

## 27.3 Ієрархія помилок (error.rs)

Правильна обробка помилок — ознака production-ready коду. Ми створимо **ієрархію** помилок, де кожен шар системи має свій тип.

### Архітектура помилок

```text
AgentError (верхній рівень)
├── ConfigError (конфігурація)
│   ├── ReadError (IO)
│   ├── ParseError (TOML)
│   └── ValidationError
├── NavigationError (навігація)
│   ├── OutOfBounds
│   └── InsufficientBattery
├── SensorError (сенсори)
│   ├── Offline
│   └── InvalidReading
└── Mission (загальні помилки місії)
```

**Постановка задачі:** Створити ієрархію помилок з thiserror, де помилки нижчих рівнів автоматично конвертуються у верхній.

```rust
// src/error.rs
use thiserror::Error;

/// Головний тип помилок агента.
/// Агрегує помилки з усіх підсистем.
#[derive(Debug, Error)]
pub enum AgentError {
    #[error("Помилка конфігурації: {0}")]
    Config(#[from] ConfigError),  // Автоматична конвертація через #[from]
    
    #[error("Помилка навігації: {0}")]
    Navigation(#[from] NavigationError),
    
    #[error("Помилка сенсора: {0}")]
    Sensor(#[from] SensorError),
    
    #[error("Помилка місії: {0}")]
    Mission(String),
}

/// Помилки завантаження та валідації конфігурації.
#[derive(Debug, Error)]
pub enum ConfigError {
    #[error("Не вдалось прочитати файл: {0}")]
    ReadError(#[from] std::io::Error),  // IO помилки автоматично конвертуються
    
    #[error("Не вдалось розпарсити TOML: {0}")]
    ParseError(#[from] toml::de::Error),
    
    #[error("Невалідна конфігурація: {0}")]
    ValidationError(String),
}

/// Помилки навігаційної підсистеми.
#[derive(Debug, Error)]
pub enum NavigationError {
    #[error("Ціль {position} за межами дозволеної зони")]
    OutOfBounds { position: String },
    
    #[error("Недостатньо батареї: є {current}%, потрібно {required}%")]
    InsufficientBattery { current: u8, required: u8 },
    
    #[error("Шлях заблокований")]
    PathBlocked,
}

/// Помилки сенсорної підсистеми.
#[derive(Debug, Error)]
pub enum SensorError {
    #[error("Сенсор '{id}' не відповідає")]
    Offline { id: String },
    
    #[error("Некоректні дані від сенсора '{id}'")]
    InvalidReading { id: String },
}

/// Зручний alias для Result з AgentError.
pub type AgentResult<T> = Result<T, AgentError>;
```

**Як це працює:**

1. `#[derive(Error)]` з thiserror генерує реалізацію `std::error::Error`
2. `#[error("...")]` задає повідомлення для `Display`
3. `#[from]` автоматично реалізує `From<X>` для конвертації
4. Оператор `?` автоматично конвертує помилки вгору по ієрархії
5. Type alias `AgentResult<T>` спрощує сигнатури функцій

---

## 27.4 Конфігурація (config.rs)

### Структура конфігурації

**Постановка задачі:** Створити структури для завантаження конфігурації з TOML з валідацією та значеннями за замовчуванням.

```rust
// src/config.rs
use serde::Deserialize;
use std::fs;
use crate::error::ConfigError;

/// Головна структура конфігурації агента.
#[derive(Debug, Deserialize)]
pub struct AgentConfig {
    pub agent: AgentParams,
    pub flight: FlightParams,
    pub mission: MissionParams,
}

/// Ідентифікація агента.
#[derive(Debug, Deserialize)]
pub struct AgentParams {
    pub id: String,
    pub name: String,
}

/// Параметри польоту.
#[derive(Debug, Deserialize)]
pub struct FlightParams {
    pub max_altitude: f64,
    pub max_speed: f64,
    
    #[serde(default = "default_battery_threshold")]
    pub battery_threshold: u8,  // Мінімальний рівень батареї
}

fn default_battery_threshold() -> u8 { 20 }

/// Параметри місії.
#[derive(Debug, Deserialize)]
pub struct MissionParams {
    pub waypoints: Vec<WaypointConfig>,
    
    #[serde(default)]  // false за замовчуванням
    pub return_on_complete: bool,
}

/// Конфігурація одного waypoint.
#[derive(Debug, Deserialize)]
pub struct WaypointConfig {
    pub x: f64,
    pub y: f64,
    
    #[serde(default)]
    pub altitude: Option<f64>,  // None = зберігати поточну висоту
    
    #[serde(default)]
    pub scan: bool,  // Чи потрібно сканувати в цій точці
}

impl AgentConfig {
    /// Завантажує конфігурацію з файлу.
    /// 
    /// # Помилки
    /// - `ConfigError::ReadError` — файл не знайдено або недоступний
    /// - `ConfigError::ParseError` — невалідний TOML
    /// - `ConfigError::ValidationError` — некоректні значення
    pub fn load(path: &str) -> Result<Self, ConfigError> {
        // Крок 1: Читаємо файл
        let content = fs::read_to_string(path)?;  // ? конвертує io::Error → ConfigError
        
        // Крок 2: Парсимо TOML
        let config: Self = toml::from_str(&content)?;  // ? конвертує toml::de::Error
        
        // Крок 3: Валідуємо
        config.validate()?;
        
        Ok(config)
    }
    
    /// Перевіряє коректність значень конфігурації.
    fn validate(&self) -> Result<(), ConfigError> {
        if self.flight.max_altitude <= 0.0 {
            return Err(ConfigError::ValidationError(
                "max_altitude має бути додатнім".to_string()
            ));
        }
        
        if self.mission.waypoints.is_empty() {
            return Err(ConfigError::ValidationError(
                "Потрібен хоча б один waypoint".to_string()
            ));
        }
        
        if self.flight.battery_threshold > 100 {
            return Err(ConfigError::ValidationError(
                "battery_threshold не може бути більше 100".to_string()
            ));
        }
        
        Ok(())
    }
}
```

### Файл конфігурації

**Постановка задачі:** Створити TOML файл з конфігурацією місії.

```toml
# config.toml — конфігурація агента БПЛА

[agent]
id = "SCOUT-001"
name = "Розвідник Альфа"

[flight]
max_altitude = 100.0   # метрів
max_speed = 15.0       # м/с
battery_threshold = 25 # мінімальний % батареї

[mission]
return_on_complete = true  # Повертатись на базу після місії

# Waypoints місії — масив таблиць
[[mission.waypoints]]
x = 100.0
y = 0.0
scan = true  # Сканувати в цій точці

[[mission.waypoints]]
x = 100.0
y = 100.0
altitude = 50.0  # Піднятись на 50м
scan = true

[[mission.waypoints]]
x = 0.0
y = 100.0
scan = false  # Просто пролетіти
```

**Що застосовано:**

| Елемент | Розділ | Призначення |
|---------|--------|-------------|
| `#[serde(default)]` | 26 | Значення за замовчуванням |
| `#[serde(default = "fn")]` | 26 | Власна функція default |
| `Option<f64>` | 18 | Необов'язкове поле |
| Оператор `?` | 19 | Ланцюжок помилок |
| Валідація | 18 | Result для перевірок |

---

## 27.5 Навігатор (navigator.rs)

**Постановка задачі:** Створити модуль навігації, що валідує цілі, обчислює шляхи та оцінює витрати батареї.

```rust
// src/navigator.rs
use crate::types::Position;
use crate::error::NavigationError;
use tracing::{info, warn, debug};

/// Навігаційна підсистема агента.
/// 
/// Відповідає за:
/// - Валідацію цільових координат
/// - Планування шляху
/// - Оцінку витрат батареї
pub struct Navigator {
    max_altitude: f64,
    boundary: (f64, f64),  // Максимальні координати (x, y)
}

impl Navigator {
    /// Створює навігатор з заданою максимальною висотою.
    pub fn new(max_altitude: f64) -> Self {
        Self {
            max_altitude,
            boundary: (500.0, 500.0),  // Зона операцій 500x500
        }
    }
    
    /// Перевіряє, чи ціль у дозволених межах.
    /// 
    /// # Помилки
    /// - `OutOfBounds` — координати за межами зони операцій
    pub fn validate_target(&self, target: &Position) -> Result<(), NavigationError> {
        // Перевірка горизонтальних меж
        if target.x < 0.0 || target.x > self.boundary.0 ||
           target.y < 0.0 || target.y > self.boundary.1 {
            return Err(NavigationError::OutOfBounds {
                position: target.to_string(),
            });
        }
        
        // Попередження якщо висота перевищує максимум
        if target.z > self.max_altitude {
            warn!(
                target_alt = target.z,
                max_alt = self.max_altitude,
                "Цільова висота перевищує максимальну"
            );
        }
        
        debug!(target = %target, "Ціль валідна");
        Ok(())
    }
    
    /// Обчислює шлях між двома точками.
    /// 
    /// Поки що повертає пряму лінію. У реальній системі
    /// тут був би алгоритм обходу перешкод.
    pub fn calculate_path(&self, from: &Position, to: &Position) -> Vec<Position> {
        debug!(from = %from, to = %to, "Розраховуємо шлях");
        
        // Спрощена реалізація — пряма лінія
        vec![*from, *to]
    }
    
    /// Оцінює витрати батареї на переліт.
    /// 
    /// Формула: 1% на кожні 10 одиниць відстані, мінімум 1%.
    pub fn estimate_battery_cost(&self, from: &Position, to: &Position) -> u8 {
        let distance = from.distance_to(to);
        let cost = ((distance / 10.0) as u8).max(1);
        
        debug!(
            distance = distance,
            cost = cost,
            "Оцінка витрат батареї"
        );
        
        cost
    }
}
```

**Як це працює:**

1. `validate_target()` перевіряє межі та повертає `Result`
2. `warn!()` логує попередження без зупинки виконання
3. `debug!()` логує деталі для налагодження
4. `estimate_battery_cost()` — проста модель витрат

---

## 27.6 Сенсори (sensors.rs)

**Постановка задачі:** Створити модуль сенсорів з можливістю виявлення об'єктів та перевірки стану.

```rust
// src/sensors.rs
use crate::types::Position;
use crate::error::SensorError;
use tracing::{info, debug};
use serde::Serialize;

/// Результат виявлення об'єкта.
#[derive(Debug, Clone, Serialize)]
pub struct Detection {
    pub position: Position,
    pub object_type: String,
    pub confidence: f64,  // 0.0 - 1.0
}

/// Масив сенсорів агента.
/// 
/// Містить GPS для позиціонування та камеру для сканування.
pub struct SensorArray {
    gps_online: bool,
    camera_online: bool,
}

impl SensorArray {
    /// Створює масив сенсорів з усіма сенсорами онлайн.
    pub fn new() -> Self {
        Self {
            gps_online: true,
            camera_online: true,
        }
    }
    
    /// Читає поточну позицію з GPS.
    /// 
    /// # Помилки
    /// - `SensorError::Offline` — GPS не відповідає
    pub fn read_position(&self) -> Result<Position, SensorError> {
        if !self.gps_online {
            return Err(SensorError::Offline { 
                id: "GPS".to_string() 
            });
        }
        
        // У реальній системі тут було б читання з hardware
        Ok(Position::default())
    }
    
    /// Сканує територію навколо центральної точки.
    /// 
    /// # Аргументи
    /// - `center` — центр сканування
    /// - `radius` — радіус сканування в метрах
    /// 
    /// # Помилки
    /// - `SensorError::Offline` — камера не відповідає
    pub fn scan(&self, center: &Position, radius: f64) -> Result<Vec<Detection>, SensorError> {
        if !self.camera_online {
            return Err(SensorError::Offline { 
                id: "Camera".to_string() 
            });
        }
        
        debug!(
            center = %center, 
            radius = radius, 
            "Сканування території"
        );
        
        // Симуляція виявлення — у реальності тут був би AI
        let detections = vec![
            Detection {
                position: Position::new(center.x + 10.0, center.y + 5.0, 0.0),
                object_type: "vehicle".to_string(),
                confidence: 0.85,
            }
        ];
        
        info!(
            count = detections.len(),
            "Виявлено об'єктів"
        );
        
        Ok(detections)
    }
    
    /// Перевіряє стан усіх сенсорів.
    /// 
    /// # Помилки
    /// - `SensorError::Offline` — якийсь сенсор офлайн
    pub fn check_all(&self) -> Result<(), SensorError> {
        if !self.gps_online {
            return Err(SensorError::Offline { id: "GPS".to_string() });
        }
        if !self.camera_online {
            return Err(SensorError::Offline { id: "Camera".to_string() });
        }
        Ok(())
    }
    
    /// Вимикає GPS (для тестування помилок).
    pub fn disable_gps(&mut self) {
        self.gps_online = false;
    }
}
```

---

## 27.7 Головна структура агента (agent.rs)

Це серце системи — агент, що координує роботу всіх підсистем.

**Постановка задачі:** Створити структуру `Agent`, яка виконує повну місію з обробкою помилок та логуванням.

```rust
// src/agent.rs
use crate::config::AgentConfig;
use crate::types::{Position, AgentState};
use crate::error::{AgentError, AgentResult, NavigationError};
use crate::navigator::Navigator;
use crate::sensors::{SensorArray, Detection};
use crate::report::{MissionReport, MissionEvent};
use tracing::{info, warn, error, instrument, span, Level};

/// Автономний агент БПЛА.
/// 
/// Координує роботу навігації, сенсорів та генерації звітів.
pub struct Agent {
    // Ідентифікація
    pub id: String,
    pub name: String,
    
    // Стан
    position: Position,
    battery: u8,
    state: AgentState,
    
    // Підсистеми
    navigator: Navigator,
    sensors: SensorArray,
    
    // Дані для звіту
    events: Vec<MissionEvent>,
    detections: Vec<Detection>,
    distance_traveled: f64,
}

impl Agent {
    /// Створює агента з конфігурації.
    pub fn from_config(config: &AgentConfig) -> Self {
        info!(
            id = %config.agent.id,
            name = %config.agent.name,
            "Створюємо агента"
        );
        
        Self {
            id: config.agent.id.clone(),
            name: config.agent.name.clone(),
            position: Position::default(),
            battery: 100,
            state: AgentState::Idle,
            navigator: Navigator::new(config.flight.max_altitude),
            sensors: SensorArray::new(),
            events: Vec::new(),
            detections: Vec::new(),
            distance_traveled: 0.0,
        }
    }
    
    // Геттери
    pub fn position(&self) -> Position { self.position }
    pub fn battery(&self) -> u8 { self.battery }
    pub fn state(&self) -> &AgentState { &self.state }
    
    /// Виконує повну місію.
    /// 
    /// Фази місії:
    /// 1. Передпольотна перевірка
    /// 2. Зліт
    /// 3. Обхід waypoints
    /// 4. Повернення на базу (якщо налаштовано)
    /// 5. Посадка
    /// 
    /// # Помилки
    /// Повертає помилку якщо будь-яка фаза провалилась.
    #[instrument(skip(self, config), fields(agent_id = %self.id))]
    pub fn execute_mission(&mut self, config: &AgentConfig) -> AgentResult<MissionReport> {
        info!("🚀 Початок місії");
        self.log_event("mission_start", "Місію розпочато");
        
        // Фаза 1: Передпольотна перевірка
        self.preflight_check()?;
        
        // Фаза 2: Зліт
        self.takeoff(30.0)?;
        
        // Фаза 3: Обхід waypoints
        for (i, wp) in config.mission.waypoints.iter().enumerate() {
            // Створюємо span для кожного waypoint
            let wp_span = span!(Level::INFO, "waypoint", number = i + 1);
            let _enter = wp_span.enter();
            
            // Формуємо цільову позицію
            let target = Position::new(
                wp.x, 
                wp.y, 
                wp.altitude.unwrap_or(self.position.z)
            );
            
            info!(target = %target, "Обробляємо waypoint");
            
            // Летимо до точки
            self.fly_to(target)?;
            
            // Скануємо якщо потрібно
            if wp.scan {
                self.scan_area(25.0)?;
            }
            
            // Перевіряємо батарею
            if self.battery < config.flight.battery_threshold {
                warn!(
                    battery = self.battery,
                    threshold = config.flight.battery_threshold,
                    "⚠️ Низький заряд, завершуємо достроково"
                );
                break;
            }
        }
        
        // Фаза 4: Повернення (якщо налаштовано)
        if config.mission.return_on_complete {
            self.return_to_base()?;
        }
        
        // Фаза 5: Посадка
        self.land()?;
        
        // Генеруємо звіт
        let report = self.generate_report();
        info!("✅ Місію завершено успішно");
        
        Ok(report)
    }
    
    /// Передпольотна перевірка систем.
    fn preflight_check(&mut self) -> AgentResult<()> {
        info!("🔍 Передпольотна перевірка");
        
        // Перевіряємо сенсори
        self.sensors.check_all()?;  // ? конвертує SensorError → AgentError
        
        // Перевіряємо батарею
        if self.battery < 50 {
            return Err(AgentError::Mission(
                format!("Недостатньо батареї для місії: {}%", self.battery)
            ));
        }
        
        self.log_event("preflight", "Усі системи в нормі");
        Ok(())
    }
    
    /// Зліт на задану висоту.
    fn takeoff(&mut self, altitude: f64) -> AgentResult<()> {
        info!(altitude = altitude, "🛫 Злітаємо");
        
        self.position.z = altitude;
        self.battery -= 5;
        self.state = AgentState::Flying;
        
        self.log_event("takeoff", &format!("Висота: {}м", altitude));
        Ok(())
    }
    
    /// Політ до цільової позиції.
    fn fly_to(&mut self, target: Position) -> AgentResult<()> {
        // Валідація цілі
        self.navigator.validate_target(&target)?;
        
        // Перевірка батареї
        let cost = self.navigator.estimate_battery_cost(&self.position, &target);
        if cost > self.battery {
            return Err(NavigationError::InsufficientBattery {
                current: self.battery,
                required: cost,
            }.into());  // .into() конвертує NavigationError → AgentError
        }
        
        // Виконуємо переліт
        let distance = self.position.distance_to(&target);
        self.distance_traveled += distance;
        self.position = target;
        self.battery -= cost;
        
        info!(
            position = %self.position, 
            battery = self.battery, 
            "📍 Досягли цілі"
        );
        
        Ok(())
    }
    
    /// Сканування території.
    fn scan_area(&mut self, radius: f64) -> AgentResult<()> {
        self.state = AgentState::Scanning;
        
        let results = self.sensors.scan(&self.position, radius)?;
        
        info!(detections = results.len(), "🔭 Сканування завершено");
        
        // Зберігаємо виявлення
        for detection in &results {
            self.log_event("detection", &format!(
                "{} на {} (впевненість: {:.0}%)",
                detection.object_type, 
                detection.position, 
                detection.confidence * 100.0
            ));
        }
        self.detections.extend(results);
        
        self.state = AgentState::Flying;
        self.battery -= 2;
        
        Ok(())
    }
    
    /// Повернення на базу.
    fn return_to_base(&mut self) -> AgentResult<()> {
        info!("🏠 Повертаємось на базу");
        self.state = AgentState::Returning;
        
        self.fly_to(Position::base())?;
        
        Ok(())
    }
    
    /// Посадка.
    fn land(&mut self) -> AgentResult<()> {
        info!("🛬 Сідаємо");
        
        self.position.z = 0.0;
        self.battery -= 3;
        self.state = AgentState::Landed;
        
        self.log_event("landing", "Посадка виконана");
        Ok(())
    }
    
    /// Додає подію до журналу місії.
    fn log_event(&mut self, event_type: &str, details: &str) {
        self.events.push(MissionEvent {
            event_type: event_type.to_string(),
            details: details.to_string(),
        });
    }
    
    /// Генерує звіт про місію.
    fn generate_report(&self) -> MissionReport {
        MissionReport::new(
            &self.id,
            self.events.clone(),
            self.detections.clone(),
            self.distance_traveled,
            100 - self.battery,
        )
    }
}
```

---

## 27.8 Звіт про місію (report.rs)

**Постановка задачі:** Створити структуру звіту, що серіалізується в JSON.

```rust
// src/report.rs
use serde::Serialize;
use crate::sensors::Detection;
use std::fs;

/// Повний звіт про виконану місію.
#[derive(Debug, Serialize)]
pub struct MissionReport {
    pub agent_id: String,
    pub status: MissionStatus,
    pub statistics: MissionStatistics,
    pub events: Vec<MissionEvent>,
    pub detections: Vec<Detection>,
}

/// Статус завершення місії.
#[derive(Debug, Serialize)]
#[serde(rename_all = "lowercase")]
pub enum MissionStatus {
    Completed,  // → "completed"
    Aborted,    // → "aborted"
    Failed,     // → "failed"
}

/// Статистика місії.
#[derive(Debug, Serialize)]
pub struct MissionStatistics {
    pub distance_traveled: f64,
    pub battery_used: u8,
    pub waypoints_visited: usize,
    pub detections_count: usize,
}

/// Подія під час місії.
#[derive(Debug, Clone, Serialize)]
pub struct MissionEvent {
    pub event_type: String,
    pub details: String,
}

impl MissionReport {
    /// Створює новий звіт.
    pub fn new(
        agent_id: &str,
        events: Vec<MissionEvent>,
        detections: Vec<Detection>,
        distance: f64,
        battery_used: u8,
    ) -> Self {
        // Рахуємо скільки waypoints відвідали (за подіями)
        let waypoints_visited = events.iter()
            .filter(|e| e.event_type == "waypoint_reached")
            .count();
        
        Self {
            agent_id: agent_id.to_string(),
            status: MissionStatus::Completed,
            statistics: MissionStatistics {
                distance_traveled: distance,
                battery_used,
                waypoints_visited,
                detections_count: detections.len(),
            },
            events,
            detections,
        }
    }
    
    /// Зберігає звіт у JSON файл.
    pub fn save(&self, path: &str) -> Result<(), Box<dyn std::error::Error>> {
        let json = serde_json::to_string_pretty(self)?;
        fs::write(path, json)?;
        Ok(())
    }
}
```

---

## 27.9 Точка входу (main.rs)

**Постановка задачі:** Об'єднати все разом — ініціалізація логування, завантаження конфігурації, виконання місії.

```rust
// src/main.rs
mod config;
mod error;
mod types;
mod agent;
mod navigator;
mod sensors;
mod report;

use tracing::{info, error, Level};
use tracing_subscriber::FmtSubscriber;

fn main() {
    // Ініціалізація логування
    let subscriber = FmtSubscriber::builder()
        .with_max_level(Level::DEBUG)
        .with_target(false)  // Без імен модулів
        .finish();
    tracing::subscriber::set_global_default(subscriber)
        .expect("Не вдалось ініціалізувати логування");
    
    // Красивий банер
    info!("╔════════════════════════════════════════╗");
    info!("║      РОБАСТНИЙ АГЕНТ v2.0              ║");
    info!("╚════════════════════════════════════════╝");
    
    // Запускаємо місію
    if let Err(e) = run_mission() {
        error!("❌ Місія провалилась: {}", e);
        std::process::exit(1);
    }
}

/// Головна логіка виконання місії.
fn run_mission() -> Result<(), Box<dyn std::error::Error>> {
    // Крок 1: Завантаження конфігурації
    info!("📄 Завантажуємо конфігурацію...");
    let config = config::AgentConfig::load("config.toml")?;
    info!(
        agent_id = %config.agent.id,
        waypoints = config.mission.waypoints.len(),
        "Конфігурацію завантажено"
    );
    
    // Крок 2: Створення агента
    let mut agent = agent::Agent::from_config(&config);
    
    // Крок 3: Виконання місії
    let report = agent.execute_mission(&config)?;
    
    // Крок 4: Збереження звіту
    report.save("mission_report.json")?;
    info!("💾 Звіт збережено в mission_report.json");
    
    // Вивід підсумку
    println!("\n📊 Підсумок місії:");
    println!("   Відстань: {:.1} одиниць", report.statistics.distance_traveled);
    println!("   Витрачено батареї: {}%", report.statistics.battery_used);
    println!("   Виявлено об'єктів: {}", report.statistics.detections_count);
    
    Ok(())
}
```

---

## 27.10 Приклад виконання

### Вивід програми

```rust
INFO ╔════════════════════════════════════════╗
INFO ║      РОБАСТНИЙ АГЕНТ v2.0              ║
INFO ╚════════════════════════════════════════╝
INFO 📄 Завантажуємо конфігурацію...
INFO agent_id=SCOUT-001 waypoints=3 Конфігурацію завантажено
INFO id=SCOUT-001 name="Розвідник Альфа" Створюємо агента
INFO agent_id=SCOUT-001 🚀 Початок місії
INFO agent_id=SCOUT-001 🔍 Передпольотна перевірка
INFO agent_id=SCOUT-001 altitude=30 🛫 Злітаємо
INFO waypoint{number=1} target=(100.0, 0.0, 30.0) Обробляємо waypoint
INFO waypoint{number=1} position=(100.0, 0.0, 30.0) battery=85 📍 Досягли цілі
INFO waypoint{number=1} detections=1 🔭 Сканування завершено
INFO waypoint{number=2} target=(100.0, 100.0, 50.0) Обробляємо waypoint
INFO waypoint{number=2} position=(100.0, 100.0, 50.0) battery=73 📍 Досягли цілі
INFO waypoint{number=2} detections=1 🔭 Сканування завершено
INFO waypoint{number=3} target=(0.0, 100.0, 50.0) Обробляємо waypoint
INFO waypoint{number=3} position=(0.0, 100.0, 50.0) battery=63 📍 Досягли цілі
INFO agent_id=SCOUT-001 🏠 Повертаємось на базу
INFO agent_id=SCOUT-001 🛬 Сідаємо
INFO agent_id=SCOUT-001 ✅ Місію завершено успішно
INFO 💾 Звіт збережено в mission_report.json

📊 Підсумок місії:
   Відстань: 341.4 одиниць
   Витрачено батареї: 40%
   Виявлено об'єктів: 2
```

---

## 27.11 Завдання для самостійної роботи

### Рівень 1: Базова реалізація (6 балів)

Реалізуйте систему за описом у розділі:

1. **Типи та помилки** (1 бал) — Position, AgentState, ієрархія помилок
2. **Конфігурація** (1 бал) — завантаження TOML з валідацією
3. **Компоненти** (2 бали) — Navigator та SensorArray
4. **Agent та Report** (2 бали) — виконання місії та JSON звіт

### Рівень 2: Розширення (2 бали)

- **Emergency handling** — при падінні батареї нижче 10% перехід у Emergency стан
- **Retry logic** — 3 спроби при помилці сенсора

### Рівень 3: Архітектура (2 бали)

- Trait `MissionExecutor` з методом `execute()`
- Реалізація для `Agent`
- `MockAgent` для тестування

---

## Підсумок Частини II

Ви завершили величезний шлях! Порівняйте еволюцію:

| v1.0 (Розділ 15) | v2.0 (Цей розділ) |
|------------------|-------------------|
| Hardcoded конфігурація | TOML файл |
| `panic!`, `unwrap()` | Graceful handling |
| `println!()` | Структуроване tracing |
| Монолітний файл | 7 модулів |
| Без збереження | JSON звіти |

---

## Зв'язок з наступною частиною

**Частина II завершена!** Ваш агент тепер:
- ✅ Завантажує конфігурацію
- ✅ Обробляє помилки gracefully
- ✅ Логує свої дії структуровано
- ✅ Зберігає результати в JSON
- ✅ Має модульну архітектуру

Але він все ще **однопотоковий** і працює **локально**.

У **Частині III: Smart Pointers та спільна пам'ять** ви дізнаєтесь:
- Як кілька агентів можуть працювати **паралельно**
- Як безпечно **ділити дані** між потоками
- Як використовувати `Arc`, `Mutex`, channels

**Вітаємо із завершенням Частини II!** 🎉
