# Розділ 15: Практикум — Автономний агент v1.0

## Вступ

Ви пройшли великий шлях. Від змінних та функцій до ownership і borrowing. Від простих структур до enum з pattern matching. Від хаотичного коду до модульної архітектури з тестами. Кожен розділ додавав новий інструмент у ваш арсенал.

Тепер час зібрати все разом.

У цьому практикумі ви побудуєте повноцінного автономного агента БПЛА версії 1.0. Не навчальний приклад з десяти рядків, а реальну систему з чіткою архітектурою: модуль позиціонування, система батареї, машина станів, обробка команд, валідація, логування та повне покриття тестами.

Це ваш "checkpoint" — робочий агент, що демонструє володіння всіма концепціями Частини I. У наступних частинах ви будете розширювати його: додавати колекції, обробку помилок, багатопотоковість, async. Але фундамент закладається тут.

---

## 15.1 Огляд архітектури

Перш ніж писати код, подивимось на загальну картину. Гарна архітектура — це коли кожен модуль має чітку відповідальність, а залежності між модулями мінімальні та зрозумілі.

### Структура проєкту

```text
autonomous_agent_v1/
├── Cargo.toml
├── README.md
└── src/
    ├── lib.rs                 # Публічний API бібліотеки
    ├── main.rs                # Демонстраційна симуляція
    │
    ├── core/                  # Ядро системи — базові типи
    │   ├── mod.rs
    │   ├── position.rs        # Position, Velocity
    │   └── battery.rs         # BatterySystem
    │
    ├── agent/                 # Агент — головна сутність
    │   ├── mod.rs
    │   ├── state.rs           # AgentState enum
    │   ├── config.rs          # AgentConfig
    │   └── agent.rs           # Структура Agent
    │
    ├── commands/              # Система команд
    │   ├── mod.rs
    │   ├── command.rs         # Command enum
    │   └── result.rs          # CommandResult
    │
    └── simulation/            # Симуляція місії
        ├── mod.rs
        └── mission.rs         # Mission, Waypoint
```

### Чому саме така структура?

**Модуль `core`** містить базові типи, що використовуються скрізь: позиція у просторі, швидкість, система батареї. Ці типи не залежать від інших модулів — вони фундамент.

**Модуль `agent`** — серце системи. Тут живе головна структура Agent, її машина станів (AgentState), конфігурація (AgentConfig). Агент залежить від core (використовує Position, Battery).

**Модуль `commands`** визначає команди, які можна надсилати агенту, та результати їх виконання. Команди залежать від core (використовують Position), але не від agent.

**Модуль `simulation`** — найвищий рівень. Він використовує agent та commands для побудови симуляції місії.

### Залежності між модулями

```text
main.rs
   │
   ▼
lib.rs (публічний API)
   │
   ├────────────────────┬──────────────────┐
   ▼                    ▼                  ▼
  core              commands            agent
   │                    │                  │
   │                    │      ┌───────────┘
   │                    ▼      ▼
   └─────────────────► simulation
```

Зверніть увагу: стрілки йдуть "вниз" — вищі модулі залежать від нижчих, але не навпаки. Core не знає про agent. Commands не знає про agent. Це робить систему модульною та легкою для тестування.

---

## 15.2 Модуль Core: базові типи

Почнемо з фундаменту — типів, що представляють фізичний світ агента.

### Position — позиція у 3D просторі

Агент БПЛА літає у тривимірному просторі. Йому потрібні координати X (схід-захід), Y (північ-південь), Z (висота). Створимо структуру Position з усіма необхідними операціями.

Ця структура демонструє кілька важливих концепцій: derive-макроси для автоматичної реалізації traits, методи для обчислень, реалізацію Display та Debug для зручного виводу.

```rust
// src/core/position.rs

use std::fmt;

/// Позиція у 3D просторі (метри).
/// 
/// Координатна система:
/// - X: схід (позитивно) / захід (негативно)
/// - Y: північ (позитивно) / південь (негативно)  
/// - Z: висота над землею
#[derive(Clone, Copy, PartialEq, Default)]
pub struct Position {
    pub x: f64,
    pub y: f64,
    pub z: f64,
}
```

Derive-атрибути роблять важливу роботу:
- `Clone` та `Copy` — позиція легка, її можна копіювати без витрат
- `PartialEq` — можемо порівнювати позиції через `==`
- `Default` — `Position::default()` повертає (0, 0, 0)

Тепер додамо методи. Спочатку — конструктори:

```rust
impl Position {
    /// Створює нову позицію з координатами.
    pub fn new(x: f64, y: f64, z: f64) -> Self {
        Self { x, y, z }
    }
    
    /// Позиція в початку координат (0, 0, 0).
    pub fn origin() -> Self {
        Self::default()
    }
}
```

Метод `new` — класичний конструктор. Метод `origin` — зручний спосіб отримати точку (0, 0, 0), яка часто використовується як "база" або початкова позиція.

Далі — обчислення відстані. Це ключова операція для навігації:

```rust
impl Position {
    /// Обчислює 3D відстань до іншої позиції.
    /// 
    /// Використовує формулу: √((x₂-x₁)² + (y₂-y₁)² + (z₂-z₁)²)
    pub fn distance_to(&self, other: &Position) -> f64 {
        let dx = self.x - other.x;
        let dy = self.y - other.y;
        let dz = self.z - other.z;
        (dx * dx + dy * dy + dz * dz).sqrt()
    }
    
    /// Обчислює 2D відстань (горизонтальну), ігноруючи висоту.
    /// 
    /// Корисно для визначення, чи агент "над" ціллю.
    pub fn distance_2d(&self, other: &Position) -> f64 {
        let dx = self.x - other.x;
        let dy = self.y - other.y;
        (dx * dx + dy * dy).sqrt()
    }
    
    /// Повертає висоту (координата Z).
    pub fn altitude(&self) -> f64 {
        self.z
    }
    
    /// Перевіряє, чи позиція в межах заданого радіуса від іншої.
    pub fn is_within(&self, other: &Position, radius: f64) -> bool {
        self.distance_to(other) <= radius
    }
}
```

Метод `distance_to` обчислює евклідову відстань у 3D. Метод `distance_2d` ігнорує висоту — корисно, коли потрібно знати, чи агент "над" точкою. Метод `is_within` — зручна обгортка для перевірки "чи ми досягли цілі".

Тепер — методи руху:

```rust
impl Position {
    /// Переміщує позицію на задані дельти (мутує self).
    pub fn translate(&mut self, dx: f64, dy: f64, dz: f64) {
        self.x += dx;
        self.y += dy;
        self.z += dz;
    }
    
    /// Створює НОВУ позицію, переміщену на задані дельти (не мутує self).
    pub fn translated(&self, dx: f64, dy: f64, dz: f64) -> Position {
        Position::new(self.x + dx, self.y + dy, self.z + dz)
    }
    
    /// Рухає позицію в напрямку цілі на задану відстань.
    /// 
    /// Повертає `true`, якщо досягнуто цілі (залишкова відстань менша за крок).
    pub fn move_towards(&mut self, target: &Position, distance: f64) -> bool {
        let total_dist = self.distance_to(target);
        
        if total_dist <= distance {
            // Ми ближче до цілі, ніж розмір кроку — переміщуємось прямо на ціль
            *self = *target;
            true
        } else {
            // Обчислюємо частку шляху, яку пройдемо
            let ratio = distance / total_dist;
            self.x += (target.x - self.x) * ratio;
            self.y += (target.y - self.y) * ratio;
            self.z += (target.z - self.z) * ratio;
            false
        }
    }
}
```

Зверніть увагу на різницю між `translate` (мутує `&mut self`) та `translated` (повертає нову копію). Це типовий паттерн у Rust — надавати обидва варіанти для гнучкості.

Метод `move_towards` — ключовий для симуляції руху. Він пересуває позицію у напрямку цілі на задану відстань. Якщо ціль ближча за крок — переміщуємось прямо на неї і повертаємо `true`.

Нарешті — реалізація Display та Debug для зручного виводу:

```rust
impl fmt::Debug for Position {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "Position({:.1}, {:.1}, {:.1})", self.x, self.y, self.z)
    }
}

impl fmt::Display for Position {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({:.1}, {:.1}, {:.1})", self.x, self.y, self.z)
    }
}
```

Debug — для розробників (через `{:?}`), Display — для користувачів (через `{}`).

### Velocity — швидкість у 3D просторі

Швидкість — це вектор, що показує напрямок та швидкість руху. Структура схожа на Position, але з іншою семантикою:

```rust
/// Швидкість у 3D просторі (метри/секунду).
#[derive(Clone, Copy, PartialEq, Default)]
pub struct Velocity {
    pub vx: f64,  // Швидкість по осі X
    pub vy: f64,  // Швидкість по осі Y
    pub vz: f64,  // Швидкість по осі Z (вертикальна)
}

impl Velocity {
    pub fn new(vx: f64, vy: f64, vz: f64) -> Self {
        Self { vx, vy, vz }
    }
    
    pub fn zero() -> Self {
        Self::default()
    }
    
    /// Обчислює абсолютну швидкість (модуль вектора).
    pub fn speed(&self) -> f64 {
        (self.vx * self.vx + self.vy * self.vy + self.vz * self.vz).sqrt()
    }
    
    /// Горизонтальна швидкість (ігноруючи вертикальну).
    pub fn horizontal_speed(&self) -> f64 {
        (self.vx * self.vx + self.vy * self.vy).sqrt()
    }
    
    /// Перевіряє, чи швидкість нульова.
    pub fn is_zero(&self) -> bool {
        self.speed() < 1e-9
    }
    
    /// Створює швидкість у напрямку від однієї позиції до іншої.
    pub fn towards(from: &Position, to: &Position, speed: f64) -> Velocity {
        let dx = to.x - from.x;
        let dy = to.y - from.y;
        let dz = to.z - from.z;
        let dist = (dx * dx + dy * dy + dz * dz).sqrt();
        
        if dist < 1e-9 {
            Velocity::zero()
        } else {
            let ratio = speed / dist;
            Velocity::new(dx * ratio, dy * ratio, dz * ratio)
        }
    }
}
```

Метод `towards` особливо корисний — він створює вектор швидкості у напрямку від однієї точки до іншої з заданим модулем.

### Тести для Position

Тести — невід'ємна частина нашого коду. Вони документують очікувану поведінку:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    fn approx_eq(a: f64, b: f64) -> bool {
        (a - b).abs() < 1e-9
    }
    
    #[test]
    fn origin_is_zero() {
        let pos = Position::origin();
        assert_eq!(pos, Position::new(0.0, 0.0, 0.0));
    }
    
    #[test]
    fn distance_to_self_is_zero() {
        let pos = Position::new(5.0, 5.0, 5.0);
        assert!(approx_eq(pos.distance_to(&pos), 0.0));
    }
    
    #[test]
    fn distance_3_4_5_triangle() {
        // Класичний трикутник: сторони 3, 4, гіпотенуза 5
        let p1 = Position::origin();
        let p2 = Position::new(3.0, 4.0, 0.0);
        assert!(approx_eq(p1.distance_to(&p2), 5.0));
    }
    
    #[test]
    fn move_towards_reaches_target_when_close() {
        let mut pos = Position::origin();
        let target = Position::new(3.0, 4.0, 0.0);  // Відстань = 5
        
        // Крок 10 > відстань 5 — маємо досягти цілі
        let reached = pos.move_towards(&target, 10.0);
        
        assert!(reached);
        assert_eq!(pos, target);
    }
    
    #[test]
    fn move_towards_partial_movement() {
        let mut pos = Position::origin();
        let target = Position::new(10.0, 0.0, 0.0);
        
        // Крок 3 < відстань 10 — рухаємось частково
        let reached = pos.move_towards(&target, 3.0);
        
        assert!(!reached);
        assert!(approx_eq(pos.x, 3.0));
        assert!(approx_eq(pos.y, 0.0));
    }
}
```

---

## 15.3 BatterySystem — система батареї

Батарея — критичний ресурс для БПЛА. Вона визначає, скільки агент може літати, коли потрібно повертатись на базу, чи взагалі можна виконувати операцію.

### Статуси батареї

Спочатку визначимо рівні стану батареї:

```rust
// src/core/battery.rs

/// Рівні стану батареї.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum BatteryStatus {
    /// Повний заряд (>80%)
    Full,
    /// Нормальний рівень (40-80%)
    Normal,
    /// Низький рівень (20-40%)
    Low,
    /// Критичний рівень (<20%)
    Critical,
}
```

Це enum без даних — просто категоризація рівня заряду.

### Структура BatterySystem

```rust
/// Система батареї агента.
#[derive(Clone)]
pub struct BatterySystem {
    /// Поточний рівень заряду (0-100)
    level: u8,
    /// Максимальна ємність
    capacity: u8,
    /// Поріг критичного рівня
    critical_threshold: u8,
    /// Поріг низького рівня
    low_threshold: u8,
}
```

Поля приватні! Це важливо — ми не хочемо, щоб хтось міг встановити `level = 200`. Уся взаємодія — через методи з валідацією.

### Конструктори

```rust
impl BatterySystem {
    /// Створює нову батарею з повним зарядом.
    pub fn new() -> Self {
        Self {
            level: 100,
            capacity: 100,
            critical_threshold: 20,
            low_threshold: 40,
        }
    }
    
    /// Створює батарею з налаштованими порогами.
    pub fn with_thresholds(critical: u8, low: u8) -> Self {
        Self {
            level: 100,
            capacity: 100,
            critical_threshold: critical.min(100),
            low_threshold: low.min(100).max(critical),
        }
    }
}
```

Метод `with_thresholds` дозволяє налаштувати пороги. Зверніть увагу на валідацію: `critical` не може бути більше 100, а `low` має бути більше за `critical`.

### Методи читання

```rust
impl BatterySystem {
    /// Поточний рівень заряду (0-100).
    pub fn level(&self) -> u8 {
        self.level
    }
    
    /// Поточний статус батареї.
    pub fn status(&self) -> BatteryStatus {
        match self.level {
            l if l > 80 => BatteryStatus::Full,
            l if l > self.low_threshold => BatteryStatus::Normal,
            l if l > self.critical_threshold => BatteryStatus::Low,
            _ => BatteryStatus::Critical,
        }
    }
    
    /// Перевіряє, чи батарея в критичному стані.
    pub fn is_critical(&self) -> bool {
        self.level <= self.critical_threshold
    }
    
    /// Перевіряє, чи батарея повністю розряджена.
    pub fn is_empty(&self) -> bool {
        self.level == 0
    }
    
    /// Перевіряє, чи батарея повністю заряджена.
    pub fn is_full(&self) -> bool {
        self.level >= self.capacity
    }
    
    /// Перевіряє, чи батарея дозволяє операції.
    pub fn is_operational(&self) -> bool {
        !self.is_critical() && !self.is_empty()
    }
}
```

Метод `status()` використовує pattern matching з guards — елегантний спосіб категоризації.

### Методи модифікації

```rust
impl BatterySystem {
    /// Витрачає заряд батареї.
    pub fn drain(&mut self, amount: u8) {
        self.level = self.level.saturating_sub(amount);
    }
    
    /// Заряджає батарею на задану кількість.
    pub fn charge(&mut self, amount: u8) {
        self.level = (self.level + amount).min(self.capacity);
    }
    
    /// Повністю заряджає батарею.
    pub fn full_charge(&mut self) {
        self.level = self.capacity;
    }
    
    /// Встановлює рівень батареї (для тестування).
    pub fn set_level(&mut self, level: u8) {
        self.level = level.min(self.capacity);
    }
}
```

`saturating_sub` запобігає переповненню — якщо намагаємось відняти більше, ніж є, отримуємо 0, а не panic.

### Методи перевірки

```rust
impl BatterySystem {
    /// Перевіряє, чи достатньо заряду для операції.
    pub fn has_enough_for(&self, required: u8) -> bool {
        self.level >= required
    }
    
    /// Прогнозує залишок після операції (без фактичної витрати).
    pub fn remaining_after(&self, drain: u8) -> u8 {
        self.level.saturating_sub(drain)
    }
}
```

Метод `remaining_after` дозволяє "подивитись у майбутнє" — скільки залишиться після операції, не виконуючи її насправді.

### Тести для Battery

```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn new_battery_is_full() {
        let battery = BatterySystem::new();
        assert_eq!(battery.level(), 100);
        assert!(battery.is_full());
    }
    
    #[test]
    fn drain_cannot_go_negative() {
        let mut battery = BatterySystem::new();
        battery.drain(150);  // Більше ніж є
        assert_eq!(battery.level(), 0);  // Не -50!
    }
    
    #[test]
    fn charge_cannot_exceed_capacity() {
        let mut battery = BatterySystem::new();
        battery.drain(10);
        battery.charge(50);
        assert_eq!(battery.level(), 100);  // Не 140!
    }
    
    #[test]
    fn status_changes_with_level() {
        let mut battery = BatterySystem::new();
        
        battery.set_level(85);
        assert_eq!(battery.status(), BatteryStatus::Full);
        
        battery.set_level(50);
        assert_eq!(battery.status(), BatteryStatus::Normal);
        
        battery.set_level(30);
        assert_eq!(battery.status(), BatteryStatus::Low);
        
        battery.set_level(15);
        assert_eq!(battery.status(), BatteryStatus::Critical);
    }
    
    #[test]
    fn is_operational_depends_on_level() {
        let mut battery = BatterySystem::new();
        
        battery.set_level(50);
        assert!(battery.is_operational());
        
        battery.set_level(20);  // На порозі
        assert!(!battery.is_operational());  // <= 20 — критично
        
        battery.set_level(21);
        assert!(battery.is_operational());  // > 20 — OK
    }
}
```

---

## 15.4 Модуль Commands: система команд

Агент отримує команди ззовні. Потрібен чіткий контракт: які команди існують, які дані вони несуть, як повідомити про результат.

### Command enum

```rust
// src/commands/command.rs

use crate::core::Position;

/// Команда для агента.
#[derive(Debug, Clone, PartialEq)]
pub enum Command {
    /// Рух до вказаної позиції.
    MoveTo {
        target: Position,
        speed: Option<f64>,  // None = швидкість за замовчуванням
    },
    
    /// Зупинка та зависання на місці.
    Hover,
    
    /// Повернення на базу.
    ReturnToBase,
    
    /// Посадка на поточній позиції.
    Land,
    
    /// Зліт на вказану висоту.
    TakeOff { altitude: f64 },
    
    /// Сканування території.
    Scan { radius: f64 },
    
    /// Початок зарядки (тільки на базі).
    StartCharging,
    
    /// Екстрена зупинка.
    EmergencyStop { reason: String },
    
    /// Скидання аварійного стану.
    Reset,
}
```

Кожен варіант несе саме ті дані, які йому потрібні. `MoveTo` знає ціль і швидкість. `TakeOff` знає цільову висоту. `Hover` не потребує даних.

### Конструктори для зручності

```rust
impl Command {
    /// Створює команду руху з автоматичною швидкістю.
    pub fn move_to(x: f64, y: f64, z: f64) -> Self {
        Command::MoveTo {
            target: Position::new(x, y, z),
            speed: None,
        }
    }
    
    /// Створює команду руху з вказаною швидкістю.
    pub fn move_to_with_speed(x: f64, y: f64, z: f64, speed: f64) -> Self {
        Command::MoveTo {
            target: Position::new(x, y, z),
            speed: Some(speed),
        }
    }
    
    /// Створює команду зльоту.
    pub fn take_off(altitude: f64) -> Self {
        Command::TakeOff { altitude }
    }
    
    /// Створює команду сканування.
    pub fn scan(radius: f64) -> Self {
        Command::Scan { radius }
    }
    
    /// Створює команду екстреної зупинки.
    pub fn emergency(reason: &str) -> Self {
        Command::EmergencyStop {
            reason: reason.to_string(),
        }
    }
}
```

Ці методи спрощують створення команд. Замість:
```rust
let cmd = Command::MoveTo { 
    target: Position::new(100.0, 50.0, 30.0), 
    speed: None 
};
```

Можна писати:
```rust
let cmd = Command::move_to(100.0, 50.0, 30.0);
```

### Категоризація команд

```rust
impl Command {
    /// Перевіряє, чи команда потребує руху.
    pub fn requires_movement(&self) -> bool {
        matches!(
            self,
            Command::MoveTo { .. } | 
            Command::ReturnToBase | 
            Command::TakeOff { .. } |
            Command::Land
        )
    }
    
    /// Перевіряє, чи команда потребує бути на землі.
    pub fn requires_grounded(&self) -> bool {
        matches!(self, Command::TakeOff { .. } | Command::StartCharging)
    }
    
    /// Перевіряє, чи команда потребує бути в повітрі.
    pub fn requires_airborne(&self) -> bool {
        matches!(
            self,
            Command::MoveTo { .. } | 
            Command::Hover | 
            Command::Scan { .. } |
            Command::Land
        )
    }
    
    /// Перевіряє, чи це екстрена команда.
    pub fn is_emergency(&self) -> bool {
        matches!(self, Command::EmergencyStop { .. })
    }
    
    /// Оцінка витрати батареї для команди.
    pub fn estimated_battery_cost(&self) -> u8 {
        match self {
            Command::MoveTo { .. } => 10,
            Command::Hover => 2,
            Command::ReturnToBase => 15,
            Command::Land => 3,
            Command::TakeOff { .. } => 8,
            Command::Scan { radius } => (*radius as u8 / 10).max(5),
            Command::StartCharging => 0,
            Command::EmergencyStop { .. } => 0,
            Command::Reset => 0,
        }
    }
}
```

Ці методи дозволяють агенту перевірити, чи команда валідна в поточному стані, ще до спроби її виконати.

### CommandResult — результат виконання

```rust
// src/commands/result.rs

/// Результат виконання команди.
#[derive(Debug, Clone, PartialEq)]
pub enum CommandResult {
    /// Команда успішно прийнята до виконання.
    Accepted,
    
    /// Команда завершена успішно.
    Completed,
    
    /// Команда відхилена через недостатній заряд батареї.
    InsufficientBattery {
        required: u8,
        available: u8,
    },
    
    /// Команда відхилена через невідповідний стан агента.
    InvalidState {
        current_state: String,
        reason: String,
    },
    
    /// Команда відхилена через невалідні параметри.
    InvalidParameters { message: String },
    
    /// Агент в аварійному стані.
    EmergencyActive { reason: String },
    
    /// Загальна помилка.
    Error { message: String },
}

impl CommandResult {
    pub fn is_success(&self) -> bool {
        matches!(self, CommandResult::Accepted | CommandResult::Completed)
    }
    
    pub fn is_error(&self) -> bool {
        !self.is_success()
    }
    
    // Фабричні методи для зручності
    pub fn battery_error(required: u8, available: u8) -> Self {
        CommandResult::InsufficientBattery { required, available }
    }
    
    pub fn state_error(current_state: &str, reason: &str) -> Self {
        CommandResult::InvalidState {
            current_state: current_state.to_string(),
            reason: reason.to_string(),
        }
    }
}
```

CommandResult — це "типізований error handling" без використання Result/Option. Кожен тип помилки несе специфічну інформацію для діагностики.

---

## 15.5 AgentState — машина станів

Агент у кожен момент часу перебуває в одному з кількох станів. Стан визначає, що агент робить і які команди може приймати.

```rust
// src/agent/state.rs

use crate::core::Position;

/// Стан агента.
#[derive(Debug, Clone, PartialEq)]
pub enum AgentState {
    /// На землі, вимкнений.
    Grounded,
    
    /// Зліт на задану висоту.
    TakingOff { target_altitude: f64 },
    
    /// В повітрі, очікує команди.
    Hovering,
    
    /// Рух до цілі.
    Moving { target: Position, speed: f64 },
    
    /// Сканування території.
    Scanning { radius: f64, progress: u8 },
    
    /// Посадка.
    Landing,
    
    /// Повернення на базу.
    ReturningToBase { base: Position },
    
    /// Зарядка на базі.
    Charging,
    
    /// Аварійний стан.
    Emergency { reason: String },
}
```

Кожен стан несе специфічні дані. `Moving` знає ціль і швидкість. `Scanning` відстежує прогрес сканування. `Emergency` зберігає причину аварії.

### Методи перевірки стану

```rust
impl AgentState {
    /// Повертає назву стану для логування.
    pub fn name(&self) -> &'static str {
        match self {
            AgentState::Grounded => "Grounded",
            AgentState::TakingOff { .. } => "TakingOff",
            AgentState::Hovering => "Hovering",
            AgentState::Moving { .. } => "Moving",
            AgentState::Scanning { .. } => "Scanning",
            AgentState::Landing => "Landing",
            AgentState::ReturningToBase { .. } => "ReturningToBase",
            AgentState::Charging => "Charging",
            AgentState::Emergency { .. } => "Emergency",
        }
    }
    
    /// Перевіряє, чи агент в повітрі.
    pub fn is_airborne(&self) -> bool {
        matches!(
            self,
            AgentState::TakingOff { .. } |
            AgentState::Hovering |
            AgentState::Moving { .. } |
            AgentState::Scanning { .. } |
            AgentState::Landing |
            AgentState::ReturningToBase { .. }
        )
    }
    
    /// Перевіряє, чи агент на землі.
    pub fn is_grounded(&self) -> bool {
        matches!(self, AgentState::Grounded | AgentState::Charging)
    }
    
    /// Перевіряє, чи агент може приймати команди.
    pub fn can_accept_commands(&self) -> bool {
        !matches!(self, AgentState::Emergency { .. })
    }
    
    /// Перевіряє, чи агент активно рухається.
    pub fn is_moving(&self) -> bool {
        matches!(
            self,
            AgentState::TakingOff { .. } |
            AgentState::Moving { .. } |
            AgentState::Landing |
            AgentState::ReturningToBase { .. }
        )
    }
    
    /// Витрата батареї за тік для даного стану.
    pub fn battery_drain_per_tick(&self) -> u8 {
        match self {
            AgentState::Grounded => 0,
            AgentState::TakingOff { .. } => 3,
            AgentState::Hovering => 1,
            AgentState::Moving { .. } => 2,
            AgentState::Scanning { .. } => 2,
            AgentState::Landing => 1,
            AgentState::ReturningToBase { .. } => 2,
            AgentState::Charging => 0,
            AgentState::Emergency { .. } => 0,
        }
    }
}
```

Метод `battery_drain_per_tick` визначає, скільки батареї витрачається за одиницю часу. На землі та під час зарядки — 0. Під час польоту — від 1 до 3 залежно від активності.

---

## 15.6 Структура Agent

Тепер об'єднаємо все в головну структуру — Agent.

```rust
// src/agent/agent.rs

use crate::core::{Position, Velocity, BatterySystem};
use crate::commands::{Command, CommandResult};
use super::state::AgentState;
use super::config::AgentConfig;

/// Автономний агент БПЛА.
pub struct Agent {
    /// Унікальний ідентифікатор.
    id: String,
    /// Поточна позиція.
    position: Position,
    /// Поточна швидкість.
    velocity: Velocity,
    /// Система батареї.
    battery: BatterySystem,
    /// Поточний стан.
    state: AgentState,
    /// Конфігурація.
    config: AgentConfig,
    /// Лічильник тіків симуляції.
    tick_count: u64,
}
```

### Конструктор

```rust
impl Agent {
    /// Створює нового агента.
    pub fn new(id: &str) -> Self {
        Self {
            id: id.to_string(),
            position: Position::origin(),
            velocity: Velocity::zero(),
            battery: BatterySystem::new(),
            state: AgentState::Grounded,
            config: AgentConfig::default(),
            tick_count: 0,
        }
    }
    
    /// Створює агента з кастомною конфігурацією.
    pub fn with_config(id: &str, config: AgentConfig) -> Self {
        Self {
            id: id.to_string(),
            position: config.base_position,
            velocity: Velocity::zero(),
            battery: BatterySystem::new(),
            state: AgentState::Grounded,
            config,
            tick_count: 0,
        }
    }
}
```

### Getters — методи читання

```rust
impl Agent {
    pub fn id(&self) -> &str {
        &self.id
    }
    
    pub fn position(&self) -> &Position {
        &self.position
    }
    
    pub fn velocity(&self) -> &Velocity {
        &self.velocity
    }
    
    pub fn battery_level(&self) -> u8 {
        self.battery.level()
    }
    
    pub fn state(&self) -> &AgentState {
        &self.state
    }
    
    pub fn is_operational(&self) -> bool {
        self.battery.is_operational() && self.state.can_accept_commands()
    }
    
    pub fn tick_count(&self) -> u64 {
        self.tick_count
    }
}
```

### Обробка команд

Головний метод — `execute_command`. Він перевіряє валідність команди і переводить агента в новий стан:

```rust
impl Agent {
    /// Виконує команду.
    pub fn execute_command(&mut self, command: Command) -> CommandResult {
        // 1. Перевірка: агент в аварійному стані?
        if let AgentState::Emergency { ref reason } = self.state {
            if !matches!(command, Command::Reset) {
                return CommandResult::EmergencyActive {
                    reason: reason.clone(),
                };
            }
        }
        
        // 2. Перевірка батареї
        let cost = command.estimated_battery_cost();
        if !self.battery.has_enough_for(cost) {
            return CommandResult::battery_error(cost, self.battery.level());
        }
        
        // 3. Обробка команди
        match command {
            Command::TakeOff { altitude } => self.handle_takeoff(altitude),
            Command::Land => self.handle_land(),
            Command::MoveTo { target, speed } => self.handle_move_to(target, speed),
            Command::Hover => self.handle_hover(),
            Command::Scan { radius } => self.handle_scan(radius),
            Command::ReturnToBase => self.handle_return_to_base(),
            Command::StartCharging => self.handle_start_charging(),
            Command::EmergencyStop { reason } => self.handle_emergency(reason),
            Command::Reset => self.handle_reset(),
        }
    }
}
```

### Обробники окремих команд

```rust
impl Agent {
    fn handle_takeoff(&mut self, altitude: f64) -> CommandResult {
        // Можна злітати тільки з землі
        if !self.state.is_grounded() {
            return CommandResult::state_error(
                self.state.name(),
                "Для зльоту агент має бути на землі"
            );
        }
        
        // Валідація висоти
        let altitude = self.config.clamp_altitude(altitude);
        
        self.state = AgentState::TakingOff { target_altitude: altitude };
        CommandResult::Accepted
    }
    
    fn handle_land(&mut self) -> CommandResult {
        // Можна сідати тільки в повітрі
        if !self.state.is_airborne() {
            return CommandResult::state_error(
                self.state.name(),
                "Для посадки агент має бути в повітрі"
            );
        }
        
        self.state = AgentState::Landing;
        CommandResult::Accepted
    }
    
    fn handle_move_to(&mut self, target: Position, speed: Option<f64>) -> CommandResult {
        // Можна рухатись тільки в повітрі
        if !self.state.is_airborne() && !self.state.is_grounded() {
            return CommandResult::state_error(
                self.state.name(),
                "Для руху агент має бути в повітрі"
            );
        }
        
        // Якщо на землі — спочатку потрібно злетіти
        if self.state.is_grounded() {
            return CommandResult::state_error(
                self.state.name(),
                "Спочатку виконайте зліт (TakeOff)"
            );
        }
        
        let speed = speed.unwrap_or(self.config.default_speed);
        let speed = self.config.clamp_speed(speed);
        
        self.state = AgentState::Moving { target, speed };
        self.velocity = Velocity::towards(&self.position, &target, speed);
        
        CommandResult::Accepted
    }
    
    fn handle_hover(&mut self) -> CommandResult {
        if !self.state.is_airborne() {
            return CommandResult::state_error(
                self.state.name(),
                "Для зависання агент має бути в повітрі"
            );
        }
        
        self.state = AgentState::Hovering;
        self.velocity = Velocity::zero();
        
        CommandResult::Accepted
    }
    
    fn handle_scan(&mut self, radius: f64) -> CommandResult {
        if !self.state.is_airborne() {
            return CommandResult::state_error(
                self.state.name(),
                "Для сканування агент має бути в повітрі"
            );
        }
        
        self.state = AgentState::Scanning { radius, progress: 0 };
        self.velocity = Velocity::zero();
        
        CommandResult::Accepted
    }
    
    fn handle_return_to_base(&mut self) -> CommandResult {
        if self.state.is_grounded() && self.position.is_within(&self.config.base_position, self.config.base_radius) {
            return CommandResult::state_error(
                self.state.name(),
                "Агент вже на базі"
            );
        }
        
        // Якщо на землі не на базі — помилка
        if self.state.is_grounded() {
            return CommandResult::state_error(
                self.state.name(),
                "Спочатку виконайте зліт"
            );
        }
        
        self.state = AgentState::ReturningToBase {
            base: self.config.base_position,
        };
        self.velocity = Velocity::towards(
            &self.position,
            &self.config.base_position,
            self.config.default_speed
        );
        
        CommandResult::Accepted
    }
    
    fn handle_start_charging(&mut self) -> CommandResult {
        // Перевірка: на землі?
        if !self.state.is_grounded() {
            return CommandResult::state_error(
                self.state.name(),
                "Для зарядки агент має бути на землі"
            );
        }
        
        // Перевірка: на базі?
        if !self.position.is_within(&self.config.base_position, self.config.base_radius) {
            return CommandResult::state_error(
                self.state.name(),
                "Для зарядки агент має бути на базі"
            );
        }
        
        // Перевірка: вже повна?
        if self.battery.is_full() {
            return CommandResult::Completed;
        }
        
        self.state = AgentState::Charging;
        CommandResult::Accepted
    }
    
    fn handle_emergency(&mut self, reason: String) -> CommandResult {
        self.state = AgentState::Emergency { reason };
        self.velocity = Velocity::zero();
        CommandResult::Accepted
    }
    
    fn handle_reset(&mut self) -> CommandResult {
        if !self.state.is_emergency() {
            return CommandResult::state_error(
                self.state.name(),
                "Reset тільки з аварійного стану"
            );
        }
        
        // Якщо в повітрі — переходимо в Hovering
        // Якщо на землі — переходимо в Grounded
        if self.position.altitude() > 1.0 {
            self.state = AgentState::Hovering;
        } else {
            self.state = AgentState::Grounded;
        }
        
        CommandResult::Accepted
    }
}
```

### Метод update — серце симуляції

Метод `update` викликається кожен "тік" симуляції. Він оновлює стан агента: переміщує позицію, витрачає батарею, перевіряє завершення операцій:

```rust
impl Agent {
    /// Оновлює стан агента (один тік симуляції).
    pub fn update(&mut self) {
        self.tick_count += 1;
        
        // Витрата батареї
        let drain = self.state.battery_drain_per_tick();
        self.battery.drain(drain);
        
        // Перевірка критичної батареї
        if self.battery.is_critical() && self.state.is_airborne() {
            if !matches!(self.state, AgentState::ReturningToBase { .. } | AgentState::Landing) {
                // Автоматичне повернення на базу
                self.state = AgentState::ReturningToBase {
                    base: self.config.base_position,
                };
                return;
            }
        }
        
        // Оновлення залежно від стану
        match &mut self.state {
            AgentState::Grounded => {
                // Нічого не робимо
            }
            
            AgentState::TakingOff { target_altitude } => {
                // Піднімаємось
                self.position.z += 5.0;  // 5 м/тік
                if self.position.z >= *target_altitude {
                    self.position.z = *target_altitude;
                    self.state = AgentState::Hovering;
                }
            }
            
            AgentState::Hovering => {
                self.velocity = Velocity::zero();
            }
            
            AgentState::Moving { target, speed } => {
                let reached = self.position.move_towards(target, *speed);
                if reached {
                    self.velocity = Velocity::zero();
                    self.state = AgentState::Hovering;
                }
            }
            
            AgentState::Scanning { radius: _, progress } => {
                *progress = (*progress + self.config.scan_rate).min(100);
                if *progress >= 100 {
                    self.state = AgentState::Hovering;
                }
            }
            
            AgentState::Landing => {
                self.position.z -= 3.0;  // 3 м/тік
                if self.position.z <= 0.0 {
                    self.position.z = 0.0;
                    self.state = AgentState::Grounded;
                }
            }
            
            AgentState::ReturningToBase { base } => {
                let reached = self.position.move_towards(base, self.config.default_speed);
                if reached {
                    self.state = AgentState::Landing;
                }
            }
            
            AgentState::Charging => {
                self.battery.charge(self.config.charge_rate);
                if self.battery.is_full() {
                    self.state = AgentState::Grounded;
                }
            }
            
            AgentState::Emergency { .. } => {
                // В аварії нічого не робимо
            }
        }
    }
}
```

---

## 15.7 Демонстраційна симуляція

Нарешті — файл main.rs, що демонструє роботу всієї системи:

```rust
// src/main.rs

use autonomous_agent_v1::{Agent, Command, Position, AgentConfig};

fn main() {
    println!("╔════════════════════════════════════════════════════════════╗");
    println!("║     СИМУЛЯЦІЯ АВТОНОМНОГО АГЕНТА БПЛА v1.0                 ║");
    println!("╚════════════════════════════════════════════════════════════╝\n");
    
    // Створюємо агента
    let config = AgentConfig::new()
        .with_max_speed(15.0)
        .with_max_altitude(200.0);
    
    let mut agent = Agent::with_config("SCOUT-001", config);
    
    println!("Агент створено: {}", agent.id());
    println!("Початкова позиція: {}", agent.position());
    println!("Стан: {}", agent.state());
    println!("Батарея: {}%\n", agent.battery_level());
    
    // === ЕТАП 1: Зліт ===
    println!("═══ ЕТАП 1: Зліт на висоту 50м ═══");
    let result = agent.execute_command(Command::take_off(50.0));
    println!("Команда TakeOff: {}", result);
    
    // Симулюємо до завершення зльоту
    while agent.state().name() == "TakingOff" {
        agent.update();
        println!("  Тік {}: висота {:.1}м, батарея {}%",
                 agent.tick_count(),
                 agent.position().altitude(),
                 agent.battery_level());
    }
    println!("Стан: {}\n", agent.state());
    
    // === ЕТАП 2: Рух до точки ===
    println!("═══ ЕТАП 2: Рух до точки (100, 80, 50) ═══");
    let result = agent.execute_command(Command::move_to(100.0, 80.0, 50.0));
    println!("Команда MoveTo: {}", result);
    
    while agent.state().name() == "Moving" {
        agent.update();
        if agent.tick_count() % 3 == 0 {
            println!("  Тік {}: позиція {}, батарея {}%",
                     agent.tick_count(),
                     agent.position(),
                     agent.battery_level());
        }
    }
    println!("Прибули! Стан: {}\n", agent.state());
    
    // === ЕТАП 3: Сканування ===
    println!("═══ ЕТАП 3: Сканування радіусом 50м ═══");
    let result = agent.execute_command(Command::scan(50.0));
    println!("Команда Scan: {}", result);
    
    while agent.state().name() == "Scanning" {
        agent.update();
        if let crate::autonomous_agent_v1::AgentState::Scanning { progress, .. } = agent.state() {
            println!("  Сканування: {}%", progress);
        }
    }
    println!("Сканування завершено! Стан: {}\n", agent.state());
    
    // === ЕТАП 4: Повернення на базу ===
    println!("═══ ЕТАП 4: Повернення на базу ═══");
    let result = agent.execute_command(Command::ReturnToBase);
    println!("Команда ReturnToBase: {}", result);
    
    while agent.state().name() != "Grounded" {
        agent.update();
        if agent.tick_count() % 5 == 0 {
            println!("  Тік {}: позиція {}, стан: {}, батарея {}%",
                     agent.tick_count(),
                     agent.position(),
                     agent.state().name(),
                     agent.battery_level());
        }
    }
    
    // === ЕТАП 5: Зарядка ===
    println!("\n═══ ЕТАП 5: Зарядка ═══");
    let result = agent.execute_command(Command::StartCharging);
    println!("Команда StartCharging: {}", result);
    
    while agent.state().name() == "Charging" {
        agent.update();
        println!("  Зарядка: {}%", agent.battery_level());
    }
    
    // === Підсумок ===
    println!("\n╔════════════════════════════════════════════════════════════╗");
    println!("║                      МІСІЯ ЗАВЕРШЕНА                       ║");
    println!("╚════════════════════════════════════════════════════════════╝");
    println!("Фінальна позиція: {}", agent.position());
    println!("Фінальний стан: {}", agent.state());
    println!("Батарея: {}%", agent.battery_level());
    println!("Всього тіків: {}", agent.tick_count());
}
```

---

## 15.8 Лабораторна робота

Тепер ваша черга розширити систему.

**Завдання:**

1. **Додайте новий стан `Patrolling`:**
   - Агент патрулює між кількома точками
   - Зберігає список waypoints і поточний індекс
   - Після досягнення останньої точки — починає спочатку

2. **Додайте команду `Patrol`:**
   - Приймає вектор позицій
   - Валідує, що є хоча б 2 точки
   - Переводить агента в стан Patrolling

3. **Реалізуйте логіку патрулювання в `update()`:**
   - Рух до поточної точки
   - Перехід до наступної при досягненні
   - Цикл по завершенню списку

4. **Напишіть тести:**
   - Тест створення патрулю
   - Тест переходу між точками
   - Тест циклічності

**Критерії оцінювання:**

| Критерій | Бали |
|----------|------|
| AgentState::Patrolling з даними | 2 |
| Command::Patrol з валідацією | 2 |
| Логіка в handle_patrol | 2 |
| Логіка в update | 2 |
| Тести | 2 |
| **Максимум** | **10** |

---

## Підсумок

У цьому практикумі ви побудували повноцінну систему автономного агента з нуля. Це не просто код — це архітектура:

- **Модульність**: кожен модуль має чітку відповідальність
- **Інкапсуляція**: приватні поля, публічний API через методи
- **Type safety**: enum для станів і команд, неможливість невалідних значень
- **Тестованість**: кожен компонент покритий тестами

Ви застосували всі концепції Частини I:
- Ownership та borrowing — передача Position, &self у методах
- Struct — Agent, Position, Velocity, BatterySystem, AgentConfig
- Enum — AgentState, Command, CommandResult, BatteryStatus
- Pattern matching — у execute_command та update
- Modules — чітка файлова структура
- Testing — тести для кожного модуля

Цей агент — фундамент для наступних частин. У Частині II ви додасте колекції та обробку помилок. У Частині III — багатопотоковість. У Частині IV — асинхронність. Але базова архітектура залишиться тією ж.

---

## Зв'язок з наступною частиною

Ваш агент працює, але він поки що досить обмежений. Що, якщо потрібно зберігати історію переміщень? Або мати динамічний список цілей? Або обробляти помилки файлової системи? 

У **Частині II: Робастність, дані та інструментарій** ви додасте колекції (Vec, HashMap), навчитесь елегантно обробляти помилки (Option, Result, ?), створите власні типи помилок, і підготуєте агента до неідеального реального світу.
