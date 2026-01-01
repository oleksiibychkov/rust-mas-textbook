# Розділ 13: Модулі та видимість — Організація коду агента

## Вступ

Уявіть, що ви будуєте будинок. Спочатку це одна кімната — все під рукою, все на виду. Але коли будинок росте до десяти кімнат, вам потрібні стіни, двері, коридори. Без них — хаос: спальня перетікає в кухню, ванна кімната — у вітальню. Гості бачать ваші особисті речі, ви не можете знайти потрібне.

Так само відбувається з кодом. Маленький проєкт на сто рядків чудово живе в одному файлі. Але коли проєкт розростається до тисяч рядків, десятків структур і функцій — потрібна організація. Модульна система Rust дозволяє розділити код на логічні одиниці, сховати деталі реалізації, і побудувати чітку архітектуру.

У цьому розділі ви навчитесь створювати модулі, керувати видимістю, організовувати файлову структуру проєкту, і перетворите код агента з монолітного файлу на елегантну модульну архітектуру.

---

## 13.1 Чому один файл — це біль

Подивимось, як виглядає проєкт без модулів. Уявіть файл main.rs на 500+ рядків:

```rust
// main.rs — все в одному файлі

// Позиція (20 рядків)
struct Position { x: f64, y: f64, z: f64 }
impl Position { /* методи */ }

// Швидкість (15 рядків)
struct Velocity { vx: f64, vy: f64, vz: f64 }
impl Velocity { /* методи */ }

// Стан агента (10 варіантів enum + методи)
enum AgentState { /* ... */ }

// Команди (15 варіантів enum)
enum Command { /* ... */ }

// Сенсори (3 структури по 20 рядків кожна)
struct GpsSensor { /* ... */ }
struct TemperatureSensor { /* ... */ }
struct BatterySensor { /* ... */ }

// Агент (50+ методів)
struct Agent { /* ... */ }
impl Agent { /* ... */ }

// Диспетчер (ще 30 методів)
struct Dispatcher { /* ... */ }
impl Dispatcher { /* ... */ }

// Утиліти (20+ функцій)
fn calculate_distance() { /* ... */ }
fn format_coordinates() { /* ... */ }
// ...

fn main() {
    // Нарешті головний код десь внизу
}
```

Такий код породжує низку проблем.

**Нескінченний скролінг.** Де та функція, яку я шукаю? Ctrl+F допомагає, але не завжди. Особливо коли є три різних `update()` для різних структур.

**Конфлікти імен.** Що робити, якщо і в агента, і в диспетчера потрібен тип `Status`? Або два різних `Error`?

**Все публічне.** Будь-хто може напряму змінити внутрішній стан агента, обійшовши валідацію. Немає захисту від випадкових помилок.

**Складність тестування.** Як протестувати GPS-сенсор окремо від усього іншого? Все переплетене.

**Командна робота.** Два розробники редагують один файл — постійні конфлікти при злитті.

Тепер порівняйте з модульною структурою:

```
src/
├── main.rs           # Точка входу — 20 рядків
├── lib.rs            # Публічний API бібліотеки
├── agent/
│   ├── mod.rs        # Модуль агента
│   ├── state.rs      # Стани агента
│   └── commands.rs   # Команди
├── sensors/
│   ├── mod.rs        # Модуль сенсорів
│   ├── gps.rs
│   └── battery.rs
├── navigation/
│   ├── mod.rs
│   └── pathfinding.rs
└── utils.rs          # Утиліти
```

Кожен файл — окрема відповідальність. Знайти потрібне легко. Тестувати просто. Працювати в команді комфортно.

---

## 13.2 Що таке модуль

Модуль у Rust — це іменований контейнер для коду. Він може містити структури, функції, константи, enum'и, і навіть інші модулі. Модулі створюють ієрархію та простори імен.

Найпростіший модуль оголошується прямо у файлі за допомогою ключового слова `mod`:

```rust
mod greetings {
    pub fn hello() {
        println!("Привіт!");
    }
    
    pub fn goodbye() {
        println!("До побачення!");
    }
    
    fn secret() {
        println!("Це приватне");
    }
}

fn main() {
    greetings::hello();
    greetings::goodbye();
    // greetings::secret();  // Помилка! Функція приватна
}
```

Ми створили модуль `greetings` з трьома функціями. Дві з них позначені `pub` — вони публічні, доступні ззовні модуля. Третя — приватна, доступна тільки всередині `greetings`.

Щоб викликати функцію з модуля, використовуємо синтаксис з подвійною двокрапкою: `module_name::function_name`. Це схоже на шлях у файловій системі: `папка/файл`, тільки з `::` замість `/`.

Результат виконання:
```
Привіт!
До побачення!
```

### Чому приватність — це добре

У Rust все приватне за замовчуванням. Це не параноя — це інструмент проєктування. Приватність дає кілька важливих переваг.

**Інкапсуляція.** Ви ховаєте деталі реалізації. Користувач модуля бачить тільки те, що йому потрібно.

**Безпека.** Неможливо випадково "зламати" внутрішній стан, обійшовши валідацію.

**Гнучкість.** Ви можете змінити внутрішню реалізацію, не ламаючи код, який використовує ваш модуль.

**Документація.** Публічне — це контракт, обіцянка. Приватне — деталі, які можуть змінитись.

Подивимось на практичному прикладі. Уявімо модуль для роботи з батареєю агента:

```rust
mod battery {
    pub struct Battery {
        level: u8,  // Приватне поле
    }
    
    impl Battery {
        pub fn new() -> Self {
            Self { level: 100 }
        }
        
        pub fn percentage(&self) -> u8 {
            self.level
        }
        
        pub fn drain(&mut self, amount: u8) {
            // Захищена логіка — рівень не може бути від'ємним
            self.level = self.level.saturating_sub(amount);
        }
        
        pub fn charge(&mut self) {
            self.level = 100;
        }
    }
}

fn main() {
    let mut bat = battery::Battery::new();
    
    bat.drain(20);
    println!("Батарея: {}%", bat.percentage());  // 80%
    
    bat.drain(100);  // Спроба витратити більше, ніж є
    println!("Батарея: {}%", bat.percentage());  // 0%, не -20%
    
    // bat.level = 200;  // Помилка компіляції!
    // Неможливо встановити невалідне значення
}
```

Поле `level` приватне. Єдиний спосіб змінити його — через методи `drain` і `charge`, які гарантують коректність. Ніхто не може встановити `level = 200` і зламати інваріант "рівень батареї від 0 до 100".

---

## 13.3 Публічність структур і полів

Коли ви робите структуру публічною через `pub struct`, це не означає, що всі її поля автоматично стають публічними. Кожне поле має свою видимість:

```rust
mod agent {
    // Приватна структура — видима тільки всередині модуля
    struct InternalData {
        secret: Vec<u8>,
    }
    
    // Публічна структура
    pub struct Agent {
        pub id: String,           // Публічне поле — доступне ззовні
        battery: u8,              // Приватне поле — доступне тільки в модулі
        internal: InternalData,   // Приватне поле приватного типу
    }
    
    impl Agent {
        // Публічний конструктор — без нього не створити Agent ззовні
        pub fn new(id: &str) -> Self {
            Self {
                id: id.to_string(),
                battery: 100,
                internal: InternalData { secret: vec![] },
            }
        }
        
        // Публічний getter — контрольований доступ до приватного поля
        pub fn battery(&self) -> u8 {
            self.battery
        }
        
        // Публічний метод модифікації з логікою
        pub fn drain_battery(&mut self, amount: u8) {
            self.battery = self.battery.saturating_sub(amount);
        }
        
        // Приватний метод — внутрішня логіка
        fn update_internal(&mut self) {
            self.internal.secret.push(42);
        }
    }
}

fn main() {
    let mut agent = agent::Agent::new("SCOUT-001");
    
    // Публічне поле — пряий доступ
    println!("ID: {}", agent.id);
    agent.id = String::from("SCOUT-002");  // Можна змінювати
    
    // Приватне поле — тільки через методи
    println!("Battery: {}", agent.battery());  // OK
    // println!("{}", agent.battery);  // Помилка!
    
    agent.drain_battery(10);  // OK
    // agent.battery = 50;  // Помилка!
    
    // agent.update_internal();  // Помилка! Приватний метод
}
```

Зверніть увагу: якщо всі поля приватні, користувач не може створити екземпляр структури безпосередньо через `Agent { id: ..., battery: ... }`. Йому потрібен конструктор. Це дає вам повний контроль над ініціалізацією.

---

## 13.4 Шляхи та ключове слово use

Доступ до елементів модуля здійснюється через шляхи. Шлях — це послідовність імен модулів, розділених `::`.

### Абсолютні та відносні шляхи

Є два типи шляхів:

**Абсолютний шлях** — починається від кореня crate (ключове слово `crate`):
```rust
crate::outer::inner::function()
```

**Відносний шлях** — починається від поточного модуля:
```rust
inner::function()
```

Розглянемо приклад з вкладеними модулями:

```rust
mod outer {
    pub mod inner {
        pub fn do_something() {
            println!("inner::do_something");
        }
    }
    
    pub fn call_inner() {
        // Відносний шлях — inner знаходиться в тому ж модулі
        inner::do_something();
        
        // Абсолютний шлях — від кореня crate
        crate::outer::inner::do_something();
    }
}

fn main() {
    // З main (який у корені) обидва способи працюють
    outer::inner::do_something();
    crate::outer::inner::do_something();
    
    outer::call_inner();
}
```

### Імпорт через use

Писати довгі шляхи щоразу незручно. Ключове слово `use` імпортує елементи в поточний scope:

```rust
mod sensors {
    pub mod gps {
        pub struct Reading {
            pub lat: f64,
            pub lon: f64,
        }
        
        pub fn current_position() -> Reading {
            Reading { lat: 50.45, lon: 30.52 }
        }
    }
    
    pub mod temperature {
        pub fn read() -> f64 {
            23.5
        }
    }
}

// Імпорт конкретних елементів
use sensors::gps::Reading;
use sensors::gps::current_position;

// Імпорт з перейменуванням (щоб уникнути конфліктів)
use sensors::temperature::read as read_temp;

fn main() {
    // Тепер можна без повного шляху
    let pos: Reading = current_position();
    let temp = read_temp();
    
    println!("Позиція: ({}, {})", pos.lat, pos.lon);
    println!("Температура: {}°C", temp);
}
```

Результат:
```
Позиція: (50.45, 30.52)
Температура: 23.5°C
```

### Групові імпорти

Коли потрібно імпортувати кілька елементів з одного модуля, можна згрупувати їх у фігурних дужках:

```rust
mod navigation {
    pub struct Position { pub x: f64, pub y: f64 }
    pub struct Velocity { pub vx: f64, pub vy: f64 }
    pub struct Heading(pub f64);
    
    pub fn calculate_distance(a: &Position, b: &Position) -> f64 {
        let dx = a.x - b.x;
        let dy = a.y - b.y;
        (dx*dx + dy*dy).sqrt()
    }
}

// Груповий імпорт
use navigation::{Position, Velocity, Heading, calculate_distance};

fn main() {
    let p1 = Position { x: 0.0, y: 0.0 };
    let p2 = Position { x: 3.0, y: 4.0 };
    
    println!("Відстань: {}", calculate_distance(&p1, &p2));  // 5.0
}
```

Існує також "glob import" через `*`, який імпортує все публічне:

```rust
use navigation::*;  // Імпортувати все
```

Але будьте обережні з ним — він робить неочевидним, звідки взялися імена, і може призвести до конфліктів. Зазвичай його використовують тільки в тестах або prelude-модулях.

### Навігація: self, super, crate

Rust надає спеціальні ключові слова для навігації по ієрархії модулів:

**`self`** — поточний модуль. Рідко потрібен явно, але корисний для ясності.

**`super`** — батьківський модуль. Дозволяє підніматися на рівень вгору.

**`crate`** — корінь поточного crate. Початок абсолютного шляху.

```rust
mod parent {
    pub fn parent_function() {
        println!("Функція батьківського модуля");
    }
    
    pub mod child {
        pub fn child_function() {
            println!("Функція дочірнього модуля");
            
            // super — звернення до батьківського модуля
            super::parent_function();
        }
        
        pub mod grandchild {
            pub fn deep_function() {
                println!("Глибоко вкладена функція");
                
                // Два рівні вгору
                super::super::parent_function();
                
                // Або абсолютний шлях від кореня
                crate::parent::parent_function();
            }
        }
    }
}

fn main() {
    parent::child::child_function();
    println!("---");
    parent::child::grandchild::deep_function();
}
```

Результат:
```
Функція дочірнього модуля
Функція батьківського модуля
---
Глибоко вкладена функція
Функція батьківського модуля
Функція батьківського модуля
```

### Re-export через pub use

Іноді внутрішня структура модулів не відповідає зручному публічному API. `pub use` дозволяє "переекспортувати" елементи під іншим шляхом:

```rust
mod internal {
    pub mod deep {
        pub mod nested {
            pub struct ImportantType {
                pub value: i32,
            }
        }
    }
}

// Створюємо зручний публічний API
mod api {
    // Re-export під коротким шляхом
    pub use crate::internal::deep::nested::ImportantType;
}

fn main() {
    // Користувач бачить простий шлях
    use api::ImportantType;
    
    let item = ImportantType { value: 42 };
    println!("Value: {}", item.value);
}
```

Це типова практика для бібліотек: внутрішня структура оптимізована для розробки, а публічний API — для користувачів.

---

## 13.5 Рівні видимості

Rust пропонує кілька рівнів видимості, не тільки "приватне" та "публічне".

### Чотири варіанти pub

```rust
mod outer {
    // 1. Приватне (за замовчуванням) — тільки цей модуль
    fn private_fn() {
        println!("Приватна");
    }
    
    // 2. pub(super) — видиме батьківському модулю
    pub(super) fn visible_to_parent() {
        println!("Видима батьку");
    }
    
    // 3. pub(crate) — видиме всьому crate, але не ззовні
    pub(crate) fn visible_in_crate() {
        println!("Видима в crate");
    }
    
    // 4. pub — повністю публічне (видиме всім)
    pub fn fully_public() {
        println!("Публічна");
    }
    
    pub mod inner {
        pub fn test_access() {
            // super::private_fn();      // Помилка! Приватна
            super::visible_to_parent();  // OK — ми дочірній модуль
            super::visible_in_crate();   // OK
            super::fully_public();       // OK
        }
    }
}

fn main() {
    // outer::private_fn();           // Помилка!
    // outer::visible_to_parent();    // Помилка! Ми не батько outer
    outer::visible_in_crate();        // OK — ми в тому ж crate
    outer::fully_public();            // OK
}
```

### Коли що використовувати

| Рівень | Призначення |
|--------|-------------|
| (приватне) | Деталі реалізації, внутрішні helper-функції |
| `pub(super)` | Спільна логіка між батьком і дочірнім модулем |
| `pub(crate)` | Внутрішній API бібліотеки, не призначений для зовнішніх користувачів |
| `pub` | Публічний API, контракт з користувачами |

Типовий паттерн для бібліотеки:
- `pub` — для типів і функцій, які мають бачити користувачі
- `pub(crate)` — для внутрішніх утиліт, спільних між модулями
- приватне — для деталей реалізації

---

## 13.6 Модулі у файлах

До цього всі модулі були "inline" — оголошені прямо у файлі через `mod name { ... }`. Але для реального проєкту потрібно розділити код на файли.

### Один модуль — один файл

Найпростіший випадок: модуль в окремому файлі з тією ж назвою.

Структура проєкту:
```
src/
├── main.rs
└── agent.rs
```

У `main.rs` ми оголошуємо модуль без тіла — Rust автоматично шукає файл `agent.rs`:

**main.rs:**
```rust
// Оголошуємо модуль — Rust шукатиме agent.rs
mod agent;

fn main() {
    let a = agent::Agent::new("SCOUT-001");
    println!("Агент: {}", a.id());
}
```

**agent.rs:**
```rust
pub struct Agent {
    id: String,
    battery: u8,
}

impl Agent {
    pub fn new(id: &str) -> Self {
        Self {
            id: id.to_string(),
            battery: 100,
        }
    }
    
    pub fn id(&self) -> &str {
        &self.id
    }
    
    pub fn battery(&self) -> u8 {
        self.battery
    }
}
```

Зверніть увагу: у `agent.rs` немає `mod agent { ... }`. Сам файл і є модулем. Вміст файлу — це вміст модуля.

### Модуль з підмодулями (папка)

Коли модуль росте, його варто розбити на підмодулі. Для цього створюємо папку з файлом `mod.rs`:

Структура:
```
src/
├── main.rs
└── agent/
    ├── mod.rs      # "Точка входу" модуля agent
    ├── state.rs    # Підмодуль state
    └── commands.rs # Підмодуль commands
```

**main.rs:**
```rust
mod agent;

use agent::Agent;
use agent::state::AgentState;
use agent::commands::Command;

fn main() {
    let mut agent = Agent::new("SCOUT-001");
    agent.execute(Command::Activate);
    println!("Стан: {:?}", agent.state());
}
```

**agent/mod.rs:**
```rust
// Оголошуємо підмодулі
pub mod state;
pub mod commands;

// Імпортуємо для внутрішнього використання
use state::AgentState;
use commands::Command;

pub struct Agent {
    id: String,
    state: AgentState,
    battery: u8,
}

impl Agent {
    pub fn new(id: &str) -> Self {
        Self {
            id: id.to_string(),
            state: AgentState::Idle,
            battery: 100,
        }
    }
    
    pub fn state(&self) -> &AgentState {
        &self.state
    }
    
    pub fn execute(&mut self, cmd: Command) {
        match cmd {
            Command::Activate => {
                self.state = AgentState::Active;
            }
            Command::Deactivate => {
                self.state = AgentState::Idle;
            }
            Command::MoveTo { x, y, z } => {
                self.state = AgentState::Moving { 
                    target: (x, y, z) 
                };
            }
        }
    }
}
```

**agent/state.rs:**
```rust
#[derive(Debug, Clone)]
pub enum AgentState {
    Idle,
    Active,
    Moving { target: (f64, f64, f64) },
    Charging { percent: u8 },
    Error(String),
}
```

**agent/commands.rs:**
```rust
#[derive(Debug)]
pub enum Command {
    Activate,
    Deactivate,
    MoveTo { x: f64, y: f64, z: f64 },
}
```

Тепер модуль `agent` має чітку структуру: головний тип `Agent` у `mod.rs`, стани в `state.rs`, команди в `commands.rs`.

### Альтернативний синтаксис

Починаючи з Rust 2018, замість `папка/mod.rs` можна використовувати `папка.rs` поруч із папкою:

```
src/
├── main.rs
├── agent.rs        # Замість agent/mod.rs
└── agent/
    ├── state.rs
    └── commands.rs
```

**agent.rs** містить те саме, що містив би `agent/mod.rs`:
```rust
pub mod state;
pub mod commands;

// Решта коду...
```

Обидва підходи еквівалентні. Вибирайте той, що вам зручніше.

---

## 13.7 Library crate та Binary crate

Rust-проєкт може бути одним з трьох типів:

**Binary crate** — виконуваний файл. Має `main.rs` з функцією `main()`.

**Library crate** — бібліотека. Має `lib.rs`. Надає код для використання іншими.

**Обидва разом** — бібліотека з виконуваним файлом для демонстрації чи CLI.

### Чиста бібліотека

Структура:
```
src/
└── lib.rs
```

**lib.rs:**
```rust
//! Бібліотека для керування агентами БПЛА
//! 
//! # Приклад
//! ```
//! use my_agent_lib::Agent;
//! let agent = Agent::new("SCOUT-001");
//! ```

pub mod agent;
pub mod sensors;
pub mod navigation;

// Re-export для зручності користувачів
pub use agent::Agent;
pub use sensors::SensorReading;
```

Користувачі бібліотеки імпортують її за назвою з `Cargo.toml`:
```rust
use my_agent_lib::Agent;
use my_agent_lib::sensors::gps;

fn main() {
    let agent = Agent::new("SCOUT-001");
}
```

### Бібліотека + виконуваний файл

Найчастіший варіант для серйозних проєктів:

```
src/
├── lib.rs    # Бібліотека
├── main.rs   # Виконуваний файл
├── agent.rs
└── ...
```

**lib.rs:**
```rust
pub mod agent;
pub mod sensors;

pub use agent::Agent;
```

**main.rs:**
```rust
// Імпортуємо з власної бібліотеки за назвою з Cargo.toml
use my_project::Agent;

fn main() {
    let agent = Agent::new("SCOUT-001");
    println!("Запущено: {}", agent.id());
}
```

Переваги такого підходу:
- Бібліотеку можна тестувати окремо (`cargo test`)
- Бібліотеку можна використовувати в інших проєктах
- Можна мати кілька виконуваних файлів для однієї бібліотеки

---

## 13.8 Повна організація проєкту агента

Застосуємо все вивчене до нашого проєкту автономного агента БПЛА.

### Фінальна структура

```
agent_system/
├── Cargo.toml
└── src/
    ├── lib.rs              # Публічний API бібліотеки
    ├── main.rs             # Точка входу
    ├── agent/
    │   ├── mod.rs          # Головний тип Agent
    │   ├── state.rs        # AgentState enum
    │   └── commands.rs     # Command enum
    ├── sensors/
    │   ├── mod.rs          # Модуль сенсорів
    │   ├── gps.rs          # GPS сенсор
    │   └── battery.rs      # Батарея
    └── navigation/
        ├── mod.rs          # Модуль навігації
        └── position.rs     # Position, Velocity
```

### Реалізація

**src/lib.rs:**
```rust
//! # Agent System
//! 
//! Бібліотека для керування автономними агентами БПЛА.
//! 
//! ## Модулі
//! - `agent` — головний тип агента та його стани
//! - `sensors` — сенсори (GPS, батарея)
//! - `navigation` — навігаційні типи (Position, Velocity)

pub mod agent;
pub mod sensors;
pub mod navigation;

// Зручні re-exports для користувачів
pub use agent::Agent;
pub use agent::state::AgentState;
pub use agent::commands::Command;
pub use navigation::position::Position;
```

**src/navigation/position.rs:**
```rust
/// Позиція у 3D просторі
#[derive(Debug, Clone, Copy, Default, PartialEq)]
pub struct Position {
    pub x: f64,
    pub y: f64,
    pub z: f64,
}

impl Position {
    /// Створює нову позицію
    pub fn new(x: f64, y: f64, z: f64) -> Self {
        Self { x, y, z }
    }
    
    /// Повертає початок координат
    pub fn origin() -> Self {
        Self::default()
    }
    
    /// Обчислює відстань до іншої позиції
    pub fn distance_to(&self, other: &Position) -> f64 {
        let dx = self.x - other.x;
        let dy = self.y - other.y;
        let dz = self.z - other.z;
        (dx*dx + dy*dy + dz*dz).sqrt()
    }
}

/// Швидкість у 3D просторі
#[derive(Debug, Clone, Copy, Default)]
pub struct Velocity {
    pub vx: f64,
    pub vy: f64,
    pub vz: f64,
}

impl Velocity {
    /// Створює нову швидкість
    pub fn new(vx: f64, vy: f64, vz: f64) -> Self {
        Self { vx, vy, vz }
    }
    
    /// Повертає нульову швидкість
    pub fn zero() -> Self {
        Self::default()
    }
    
    /// Обчислює швидкість (модуль вектора)
    pub fn speed(&self) -> f64 {
        (self.vx*self.vx + self.vy*self.vy + self.vz*self.vz).sqrt()
    }
}
```

**src/navigation/mod.rs:**
```rust
//! Модуль навігації
//! 
//! Містить типи для представлення позиції та швидкості.

pub mod position;

pub use position::{Position, Velocity};
```

**src/sensors/battery.rs:**
```rust
/// Рівень заряду батареї
#[derive(Debug)]
pub struct BatteryLevel {
    level: u8,
    max_capacity: u8,
}

impl BatteryLevel {
    /// Створює повністю заряджену батарею
    pub fn new() -> Self {
        Self {
            level: 100,
            max_capacity: 100,
        }
    }
    
    /// Повертає поточний рівень у відсотках
    pub fn percentage(&self) -> u8 {
        self.level
    }
    
    /// Перевіряє, чи критично низький рівень (< 20%)
    pub fn is_critical(&self) -> bool {
        self.level < 20
    }
    
    /// Перевіряє, чи низький рівень (< 40%)
    pub fn is_low(&self) -> bool {
        self.level < 40
    }
    
    /// Витрачає заряд
    pub fn drain(&mut self, amount: u8) {
        self.level = self.level.saturating_sub(amount);
    }
    
    /// Заряджає батарею
    pub fn charge(&mut self, amount: u8) {
        self.level = (self.level + amount).min(self.max_capacity);
    }
    
    /// Повністю заряджає батарею
    pub fn full_charge(&mut self) {
        self.level = self.max_capacity;
    }
}

impl Default for BatteryLevel {
    fn default() -> Self {
        Self::new()
    }
}
```

**src/sensors/gps.rs:**
```rust
use crate::navigation::Position;

/// GPS сенсор для визначення позиції
#[derive(Debug)]
pub struct GpsSensor {
    last_reading: Option<Position>,
    accuracy: f64,
}

impl GpsSensor {
    /// Створює новий GPS сенсор
    pub fn new() -> Self {
        Self {
            last_reading: None,
            accuracy: 5.0,  // метри
        }
    }
    
    /// Читає поточну позицію
    pub fn read(&mut self, actual_position: &Position) -> Position {
        // У реальності тут була б похибка
        let reading = *actual_position;
        self.last_reading = Some(reading);
        reading
    }
    
    /// Повертає останнє читання
    pub fn last_reading(&self) -> Option<&Position> {
        self.last_reading.as_ref()
    }
    
    /// Повертає точність сенсора
    pub fn accuracy(&self) -> f64 {
        self.accuracy
    }
}

impl Default for GpsSensor {
    fn default() -> Self {
        Self::new()
    }
}
```

**src/sensors/mod.rs:**
```rust
//! Модуль сенсорів
//! 
//! Містить різні сенсори для агента.

pub mod battery;
pub mod gps;

pub use battery::BatteryLevel;
pub use gps::GpsSensor;
```

**src/agent/state.rs:**
```rust
use crate::navigation::Position;

/// Стан агента
#[derive(Debug, Clone)]
pub enum AgentState {
    /// Очікування команд
    Idle,
    
    /// Рух до цілі
    Moving { 
        target: Position,
        speed: f64,
    },
    
    /// Сканування території
    Scanning {
        radius: f64,
        progress: u8,
    },
    
    /// Зарядка
    Charging {
        current: u8,
        target_level: u8,
    },
    
    /// Повернення на базу
    Returning {
        base: Position,
    },
    
    /// Аварійний стан
    Emergency(String),
}
```

**src/agent/commands.rs:**
```rust
use crate::navigation::Position;

/// Команда для агента
#[derive(Debug, Clone)]
pub enum Command {
    /// Рух до точки
    MoveTo(Position),
    
    /// Сканування з радіусом
    Scan { radius: f64 },
    
    /// Повернення на базу
    Return,
    
    /// Зарядка
    Charge,
    
    /// Аварійна зупинка
    EmergencyStop(String),
}
```

**src/agent/mod.rs:**
```rust
//! Модуль агента
//! 
//! Головний тип Agent та пов'язані типи.

pub mod state;
pub mod commands;

use crate::navigation::Position;
use crate::sensors::{BatteryLevel, GpsSensor};
use state::AgentState;
use commands::Command;

/// Автономний агент БПЛА
#[derive(Debug)]
pub struct Agent {
    id: String,
    position: Position,
    state: AgentState,
    battery: BatteryLevel,
    gps: GpsSensor,
}

impl Agent {
    /// Створює нового агента
    pub fn new(id: &str) -> Self {
        Self {
            id: id.to_string(),
            position: Position::origin(),
            state: AgentState::Idle,
            battery: BatteryLevel::new(),
            gps: GpsSensor::new(),
        }
    }
    
    /// Повертає ідентифікатор
    pub fn id(&self) -> &str {
        &self.id
    }
    
    /// Повертає поточну позицію
    pub fn position(&self) -> &Position {
        &self.position
    }
    
    /// Повертає поточний стан
    pub fn state(&self) -> &AgentState {
        &self.state
    }
    
    /// Повертає рівень батареї
    pub fn battery_level(&self) -> u8 {
        self.battery.percentage()
    }
    
    /// Виконує команду
    pub fn execute(&mut self, cmd: Command) {
        // Перевірка: з аварійного стану виходу немає
        if matches!(self.state, AgentState::Emergency(_)) {
            println!("  [{}] В аварійному стані, команда ігнорується", self.id);
            return;
        }
        
        match cmd {
            Command::MoveTo(target) => {
                self.state = AgentState::Moving { target, speed: 10.0 };
            }
            Command::Scan { radius } => {
                self.state = AgentState::Scanning { radius, progress: 0 };
            }
            Command::Return => {
                self.state = AgentState::Returning { 
                    base: Position::origin() 
                };
            }
            Command::Charge => {
                self.state = AgentState::Charging { 
                    current: self.battery.percentage(),
                    target_level: 100,
                };
            }
            Command::EmergencyStop(reason) => {
                self.state = AgentState::Emergency(reason);
            }
        }
    }
    
    /// Оновлює стан агента (виклик кожен тік)
    pub fn update(&mut self) {
        match &mut self.state {
            AgentState::Idle => {}
            
            AgentState::Moving { target, speed } => {
                let dist = self.position.distance_to(target);
                if dist < *speed {
                    self.position = *target;
                    self.state = AgentState::Idle;
                } else {
                    let ratio = *speed / dist;
                    self.position.x += (target.x - self.position.x) * ratio;
                    self.position.y += (target.y - self.position.y) * ratio;
                    self.position.z += (target.z - self.position.z) * ratio;
                }
                self.battery.drain(2);
            }
            
            AgentState::Scanning { progress, .. } => {
                *progress = (*progress + 20).min(100);
                if *progress >= 100 {
                    self.state = AgentState::Idle;
                }
                self.battery.drain(3);
            }
            
            AgentState::Charging { current, target_level } => {
                *current = (*current + 10).min(*target_level);
                self.battery.charge(10);
                if *current >= *target_level {
                    self.state = AgentState::Idle;
                }
            }
            
            AgentState::Returning { base } => {
                let dist = self.position.distance_to(base);
                if dist < 5.0 {
                    self.position = *base;
                    self.state = AgentState::Idle;
                }
                self.battery.drain(2);
            }
            
            AgentState::Emergency(_) => {}
        }
    }
}
```

**src/main.rs:**
```rust
use agent_system::{Agent, Command, Position};

fn main() {
    println!("=== Система керування агентами ===\n");
    
    let mut scout = Agent::new("SCOUT-001");
    
    println!("Створено агента: {}", scout.id());
    println!("Позиція: {:?}", scout.position());
    println!("Стан: {:?}", scout.state());
    println!("Батарея: {}%\n", scout.battery_level());
    
    // Виконуємо місію
    println!("--- Відправляємо в точку (100, 50, 30) ---");
    scout.execute(Command::MoveTo(Position::new(100.0, 50.0, 30.0)));
    
    for i in 1..=12 {
        scout.update();
        println!("Тік {}: позиція ({:.1}, {:.1}, {:.1}), батарея {}%",
                 i, 
                 scout.position().x, 
                 scout.position().y, 
                 scout.position().z,
                 scout.battery_level());
        
        if matches!(scout.state(), agent_system::AgentState::Idle) {
            println!("Прибули!\n");
            break;
        }
    }
    
    println!("--- Сканування ---");
    scout.execute(Command::Scan { radius: 50.0 });
    while !matches!(scout.state(), agent_system::AgentState::Idle) {
        scout.update();
        println!("Сканування... батарея {}%", scout.battery_level());
    }
    println!("Сканування завершено!\n");
    
    println!("Фінальний стан: {:?}", scout.state());
    println!("Фінальна батарея: {}%", scout.battery_level());
}
```

---

## 13.9 Лабораторна робота

Створіть модульну структуру для системи розумного будинку.

**Завдання:**

1. Створіть проєкт `smart_home` з такою структурою:
   ```
   src/
   ├── lib.rs
   ├── main.rs
   ├── devices/
   │   ├── mod.rs
   │   ├── light.rs      # Лампочка
   │   └── thermostat.rs # Термостат
   └── rooms/
       ├── mod.rs
       └── room.rs       # Кімната з пристроями
   ```

2. Реалізуйте:
   - `Light` з методами `turn_on()`, `turn_off()`, `set_brightness(u8)`
   - `Thermostat` з методами `set_temperature(f64)`, `current_temperature()`
   - `Room` що містить вектор пристроїв

3. У `lib.rs` зробіть зручні re-exports

4. У `main.rs` продемонструйте роботу системи

**Критерії оцінювання:**

| Критерій | Бали |
|----------|------|
| Правильна структура файлів | 2 |
| Модуль devices з Light і Thermostat | 3 |
| Модуль rooms з Room | 2 |
| Re-exports у lib.rs | 1 |
| Демонстрація в main.rs | 2 |
| **Максимум** | **10** |

---

## Підсумок

Модульна система Rust — це не просто спосіб розділити код на файли. Це інструмент архітектури, що дозволяє контролювати складність, приховувати деталі реалізації, і будувати надійні API.

Ви навчились створювати модулі через `mod`, керувати видимістю через `pub` різних рівнів, імпортувати елементи через `use`, і організовувати файлову структуру проєкту.

Ключові принципи:
- **Все приватне за замовчуванням** — явно публікуйте тільки необхідне
- **Один модуль — одна відповідальність** — не змішуйте різні концепції
- **Re-export для зручності** — внутрішня структура не повинна диктувати публічний API
- **lib.rs для бібліотеки, main.rs для виконання** — розділяйте логіку і точку входу

---

## Зв'язок з наступним розділом

Ваш код тепер елегантно організований у модулі. Але як переконатись, що він працює правильно? У **Розділі 14: Модульне тестування** ви навчитесь писати тести, що перевіряють інваріанти, виловлюють регресії, і служать документацією до коду.
