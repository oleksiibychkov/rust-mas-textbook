# Розділ 22: Трейти — базові концепції

## Вступ

Уявіть, що ви розробляєте систему керування різними типами агентів: дрон-розвідник, наземний робот, підводний апарат. Всі вони можуть рухатись, але кожен по-своєму: дрон летить, робот їде, підводний апарат пливе. Як написати код, що працює з **будь-яким** агентом, не знаючи його конкретного типу?

Без абстракції доведеться писати окремі функції для кожного типу:

```rust
fn send_drone_to(drone: &Drone, target: Position) { ... }
fn send_rover_to(rover: &Rover, target: Position) { ... }
fn send_submarine_to(sub: &Submarine, target: Position) { ... }
```

Три функції, що роблять по суті те саме. А якщо завтра з'явиться новий тип агента? Ще одна функція. Це не масштабується.

**Traits** — рішення цієї проблеми. Trait визначає набір методів, які тип може реалізувати. Це як контракт: "якщо тип реалізує trait `Movable`, він гарантовано має метод `move_to()`". Тепер можна написати:

```rust
fn send_agent_to(agent: &impl Movable, target: Position) { ... }
```

Одна функція для всіх агентів. Traits — основа поліморфізму в Rust, аналог інтерфейсів в Java/C#, але з унікальними можливостями.

---

## 22.1 Що таке trait

### Визначення та реалізація

Trait — це набір сигнатур методів (і, опційно, реалізацій за замовчуванням), які тип зобов'язується надати.

**Постановка задачі:** Створити trait `Describable`, який гарантує, що тип має метод `describe()`. Потім реалізувати цей trait для структури `Product`.

```rust
// Визначення trait
trait Describable {
    // Обов'язковий метод — тільки сигнатура, без тіла
    fn describe(&self) -> String;
    
    // Метод з реалізацією за замовчуванням
    fn short_description(&self) -> String {
        format!("Об'єкт: {}", self.describe())
    }
}

// Тип, що реалізує trait
struct Product {
    name: String,
    price: f64,
}

// Реалізація trait для типу
impl Describable for Product {
    fn describe(&self) -> String {
        format!("{} (${:.2})", self.name, self.price)
    }
    // short_description — використовує реалізацію за замовчуванням
}

fn main() {
    let laptop = Product {
        name: "Ноутбук".to_string(),
        price: 999.99,
    };
    
    println!("{}", laptop.describe());         // Ноутбук ($999.99)
    println!("{}", laptop.short_description()); // Об'єкт: Ноутбук ($999.99)
}
```

**Як це працює:**

1. `trait Describable` визначає контракт — метод `describe()`
2. `impl Describable for Product` — Product зобов'язується виконати контракт
3. `describe()` — обов'язкова реалізація, бо trait не надав тіла
4. `short_description()` — використовує default реалізацію з trait

### Trait як контракт

Головна сила traits — можливість писати код, що працює з **будь-яким** типом, що реалізує потрібний trait.

**Постановка задачі:** Створити trait `Sensor` для різних типів сенсорів (температура, тиск) і функцію, що працює з будь-яким сенсором.

```rust
// Trait визначає контракт для всіх сенсорів
trait Sensor {
    fn read(&self) -> f64;
    fn name(&self) -> &str;
    fn is_online(&self) -> bool;
}

// Сенсор температури
struct TemperatureSensor {
    id: String,
    value: f64,
    online: bool,
}

impl Sensor for TemperatureSensor {
    fn read(&self) -> f64 {
        self.value
    }
    
    fn name(&self) -> &str {
        &self.id
    }
    
    fn is_online(&self) -> bool {
        self.online
    }
}

// Сенсор тиску — інша структура, той самий trait
struct PressureSensor {
    id: String,
    pressure: f64,
    connected: bool,
}

impl Sensor for PressureSensor {
    fn read(&self) -> f64 {
        self.pressure
    }
    
    fn name(&self) -> &str {
        &self.id
    }
    
    fn is_online(&self) -> bool {
        self.connected
    }
}

// Функція працює з БУДЬ-ЯКИМ сенсором
fn check_sensor(sensor: &impl Sensor) {
    if sensor.is_online() {
        println!("{}: {:.2}", sensor.name(), sensor.read());
    } else {
        println!("{}: OFFLINE", sensor.name());
    }
}

fn main() {
    let temp = TemperatureSensor {
        id: "TEMP-01".to_string(),
        value: 23.5,
        online: true,
    };
    
    let pressure = PressureSensor {
        id: "PRESS-01".to_string(),
        pressure: 1013.25,
        connected: false,
    };
    
    check_sensor(&temp);     // TEMP-01: 23.50
    check_sensor(&pressure); // PRESS-01: OFFLINE
}
```

**Як це працює:**

1. `trait Sensor` визначає три методи — контракт для всіх сенсорів
2. `TemperatureSensor` і `PressureSensor` — різні структури
3. Обидві реалізують `Sensor`, тому обидві "є сенсорами"
4. `check_sensor(sensor: &impl Sensor)` приймає **будь-що**, що реалізує `Sensor`
5. Компілятор гарантує: якщо тип переданий у функцію, він точно має потрібні методи

---

## 22.2 Trait Debug — для розробників

### Навіщо потрібен Debug

`Debug` — стандартний trait для виведення значення у форматі, зручному для налагодження. Без нього неможливо використовувати `{:?}` у `println!`.

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 10, y: 20 };
    // println!("{:?}", p);  // ❌ ПОМИЛКА: `Point` doesn't implement `Debug`
}
```

### Derive — автоматична реалізація

Rust може **автоматично** згенерувати реалізацію Debug через атрибут `#[derive]`.

**Постановка задачі:** Додати можливість виводити структури `Point` та `Rectangle` через `{:?}`.

```rust
#[derive(Debug)]  // Автоматична реалізація Debug
struct Point {
    x: i32,
    y: i32,
}

#[derive(Debug)]  // Працює і для вкладених структур
struct Rectangle {
    top_left: Point,
    bottom_right: Point,
}

fn main() {
    let p = Point { x: 10, y: 20 };
    let rect = Rectangle {
        top_left: Point { x: 0, y: 0 },
        bottom_right: Point { x: 100, y: 50 },
    };
    
    // Компактний формат {:?}
    println!("{:?}", p);  // Point { x: 10, y: 20 }
    
    // Pretty-print {:#?} — з відступами
    println!("{:#?}", rect);
    // Rectangle {
    //     top_left: Point {
    //         x: 0,
    //         y: 0,
    //     },
    //     bottom_right: Point {
    //         x: 100,
    //         y: 50,
    //     },
    // }
}
```

**Як це працює:**

1. `#[derive(Debug)]` просить компілятор згенерувати `impl Debug for Point`
2. Згенерована реалізація виводить назву структури та всі поля
3. `{:#?}` — "pretty" формат з переносами та відступами
4. Для вкладених структур теж потрібен `#[derive(Debug)]`

### Власна реалізація Debug

Іноді автоматична реалізація не підходить — наприклад, потрібно приховати секретні дані.

**Постановка задачі:** Створити структуру `SecretKey`, що при виведенні приховує реальне значення ключа.

```rust
use std::fmt;

struct SecretKey {
    key: String,
}

// Власна реалізація Debug — приховуємо реальне значення
impl fmt::Debug for SecretKey {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "SecretKey(***)")  // Виводимо зірочки замість ключа
    }
}

#[derive(Debug)]
struct Agent {
    id: String,
    api_key: SecretKey,  // Включає SecretKey
}

fn main() {
    let agent = Agent {
        id: "SCOUT-001".to_string(),
        api_key: SecretKey { 
            key: "super_secret_12345".to_string() 
        },
    };
    
    println!("{:?}", agent);
    // Agent { id: "SCOUT-001", api_key: SecretKey(***) }
    // Ключ прихований!
}
```

**Як це працює:**

1. `impl fmt::Debug for SecretKey` — власна реалізація
2. `write!(f, "SecretKey(***)")` — виводимо що хочемо, не реальне значення
3. Коли `Agent` виводиться, він викликає Debug для `SecretKey`
4. Секрет залишається секретом навіть у логах

---

## 22.3 Trait Display — для користувачів

### Display vs Debug

- **Debug** (`{:?}`) — технічний формат для розробників
- **Display** (`{}`) — читабельний формат для користувачів

**Ключова різниця:** Debug можна derive, Display — тільки вручну. Чому? Тому що "гарний" формат для користувача залежить від контексту, і компілятор не може його вгадати.

**Постановка задачі:** Створити тип `Temperature` з Debug (автоматичним) та Display (ручним) форматуванням.

```rust
use std::fmt;

#[derive(Debug)]  // Debug через derive — технічний формат
struct Temperature {
    celsius: f64,
}

// Display — тільки вручну, бо формат залежить від потреб
impl fmt::Display for Temperature {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{:.1}°C", self.celsius)
    }
}

fn main() {
    let temp = Temperature { celsius: 23.5 };
    
    // Для розробників
    println!("Debug: {:?}", temp);    // Debug: Temperature { celsius: 23.5 }
    
    // Для користувачів
    println!("Display: {}", temp);    // Display: 23.5°C
}
```

### Display для агента БПЛА

**Постановка задачі:** Створити читабельний вивід для агента зі станом, позицією та батареєю.

```rust
use std::fmt;

#[derive(Debug)]
struct Position {
    x: f64,
    y: f64,
    z: f64,
}

impl fmt::Display for Position {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({:.1}, {:.1}, {:.1})", self.x, self.y, self.z)
    }
}

#[derive(Debug)]
enum AgentState {
    Grounded,
    Flying,
    Emergency(String),
}

impl fmt::Display for AgentState {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AgentState::Grounded => write!(f, "🛬 На землі"),
            AgentState::Flying => write!(f, "✈️ В польоті"),
            AgentState::Emergency(reason) => write!(f, "🚨 Аварія: {}", reason),
        }
    }
}

#[derive(Debug)]
struct Agent {
    id: String,
    position: Position,
    battery: u8,
    state: AgentState,
}

impl fmt::Display for Agent {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(
            f,
            "[{}] {} | Позиція: {} | Батарея: {}%",
            self.id, self.state, self.position, self.battery
        )
    }
}

fn main() {
    let agent = Agent {
        id: "SCOUT-001".to_string(),
        position: Position { x: 100.0, y: 50.0, z: 30.0 },
        battery: 85,
        state: AgentState::Flying,
    };
    
    // Для користувача — читабельно
    println!("{}", agent);
    // [SCOUT-001] ✈️ В польоті | Позиція: (100.0, 50.0, 30.0) | Батарея: 85%
    
    // Для розробника — технічно
    println!("{:?}", agent);
    // Agent { id: "SCOUT-001", position: Position { ... }, battery: 85, state: Flying }
}
```

### Display дає ToString безкоштовно

Якщо тип реалізує `Display`, він автоматично отримує метод `to_string()`:

```rust
use std::fmt;

struct Greeting {
    name: String,
}

impl fmt::Display for Greeting {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "Привіт, {}!", self.name)
    }
}

fn main() {
    let greeting = Greeting { name: "Світ".to_string() };
    
    // to_string() доступний безкоштовно
    let s: String = greeting.to_string();
    println!("{}", s);  // Привіт, Світ!
}
```

---

## 22.4 Trait Clone — явне копіювання

### Що таке Clone

`Clone` дозволяє створити **глибоку копію** значення. На відміну від звичайного присвоєння (яке може переміщати ownership), `.clone()` завжди створює незалежну копію.

**Постановка задачі:** Продемонструвати, як Clone створює незалежні копії.

```rust
#[derive(Debug, Clone)]  // Clone через derive
struct Position {
    x: f64,
    y: f64,
}

fn main() {
    let pos1 = Position { x: 10.0, y: 20.0 };
    
    // clone() створює повністю незалежну копію
    let pos2 = pos1.clone();
    
    // pos1 все ще доступний!
    println!("pos1: {:?}", pos1);  // Position { x: 10.0, y: 20.0 }
    println!("pos2: {:?}", pos2);  // Position { x: 10.0, y: 20.0 }
    
    // Зміни в одному НЕ впливають на інший
    let mut pos3 = pos1.clone();
    pos3.x = 999.0;
    
    println!("pos1.x = {}, pos3.x = {}", pos1.x, pos3.x);
    // pos1.x = 10, pos3.x = 999
}
```

### Clone для складних структур

Якщо структура містить інші типи, всі вони теж повинні реалізовувати Clone:

```rust
#[derive(Debug, Clone)]
struct Waypoint {
    name: String,      // String реалізує Clone
    position: Position,
}

#[derive(Debug, Clone)]
struct Position {
    x: f64,
    y: f64,
}

#[derive(Debug, Clone)]
struct Mission {
    id: String,
    waypoints: Vec<Waypoint>,  // Vec<T> реалізує Clone, якщо T: Clone
}

fn main() {
    let mission = Mission {
        id: "ALPHA".to_string(),
        waypoints: vec![
            Waypoint { 
                name: "Старт".to_string(), 
                position: Position { x: 0.0, y: 0.0 } 
            },
            Waypoint { 
                name: "Фініш".to_string(), 
                position: Position { x: 100.0, y: 100.0 } 
            },
        ],
    };
    
    // Глибока копія — ВСІ вкладені дані копіюються
    let mission_backup = mission.clone();
    
    println!("Оригінал: {:?}", mission);
    println!("Копія: {:?}", mission_backup);
}
```

**Як це працює:**

1. `Mission::clone()` викликає `String::clone()` для id
2. `Vec::clone()` для waypoints
3. Для кожного `Waypoint` — `String::clone()` і `Position::clone()`
4. Результат — повністю незалежна копія всієї структури

---

## 22.5 Trait Copy — неявне копіювання

### Різниця між Clone і Copy

- **Clone** — явне копіювання через `.clone()`
- **Copy** — неявне копіювання при присвоєнні

**Постановка задачі:** Показати різницю поведінки Copy та не-Copy типів.

```rust
fn main() {
    // i32 реалізує Copy — копіюється автоматично
    let x: i32 = 42;
    let y = x;  // x копіюється в y
    
    println!("x = {}, y = {}", x, y);  // Обидва доступні!
    
    // String НЕ реалізує Copy — переміщується
    let s1 = String::from("hello");
    let s2 = s1;  // s1 ПЕРЕМІЩУЄТЬСЯ в s2
    
    // println!("{}", s1);  // ❌ ПОМИЛКА: s1 moved
    println!("{}", s2);  // OK
}
```

### Коли тип може бути Copy

Тип може реалізувати Copy, якщо:

1. **Всі поля реалізують Copy** — String, Vec, Box не Copy
2. **Тип не має Drop** — не можна мати і Copy, і Drop
3. **Копіювання семантично коректне** — тип "дешевий" для копіювання

```rust
// ✅ Може бути Copy — всі поля примітивні
#[derive(Debug, Clone, Copy)]
struct Point {
    x: f64,  // f64: Copy
    y: f64,  // f64: Copy
}

// ❌ НЕ може бути Copy — String не Copy
#[derive(Debug, Clone)]  // Тільки Clone, без Copy!
struct NamedPoint {
    name: String,  // String: НЕ Copy
    x: f64,
    y: f64,
}

fn main() {
    // Point — Copy
    let p1 = Point { x: 1.0, y: 2.0 };
    let p2 = p1;  // Copy — p1 залишається доступним
    println!("p1 = {:?}, p2 = {:?}", p1, p2);
    
    // NamedPoint — не Copy
    let np1 = NamedPoint { name: "A".to_string(), x: 1.0, y: 2.0 };
    let np2 = np1.clone();  // Потрібен явний clone!
    // let np3 = np1;  // Без clone — np1 переміщується
}
```

### Copy для агента — коли варто

**Правило:** Copy для маленьких, простих типів, що часто копіюються.

```rust
// ✅ Position — хороший кандидат для Copy
// Маленька (24 байти), часто передається, немає складних полів
#[derive(Debug, Clone, Copy)]
struct Position {
    x: f64,
    y: f64,
    z: f64,
}

// ❌ Agent — поганий кандидат для Copy
// Великий, має String та Vec, копіювання дороге
#[derive(Debug, Clone)]  // Тільки Clone
struct Agent {
    id: String,           // String: НЕ Copy
    position: Position,
    battery: u8,
    history: Vec<Position>,  // Vec: НЕ Copy
}

fn main() {
    let pos = Position { x: 1.0, y: 2.0, z: 3.0 };
    
    // Position копіюється автоматично
    fn process_position(p: Position) {
        println!("Позиція: {:?}", p);
    }
    
    process_position(pos);  // Copy
    process_position(pos);  // Copy — pos все ще доступний
    process_position(pos);  // Copy
}
```

---

## 22.6 Trait Default — значення за замовчуванням

### Навіщо Default

`Default` надає "розумне" значення за замовчуванням для типу. Корисно для конфігурацій, де більшість полів мають стандартні значення.

**Постановка задачі:** Створити структуру конфігурації з Default через derive.

```rust
#[derive(Debug, Default)]
struct Config {
    timeout: u32,    // Default для u32: 0
    retries: u8,     // Default для u8: 0
    enabled: bool,   // Default для bool: false
    name: String,    // Default для String: ""
}

fn main() {
    // Створення через Default::default()
    let config: Config = Default::default();
    println!("{:?}", config);
    // Config { timeout: 0, retries: 0, enabled: false, name: "" }
    
    // Або коротший синтаксис
    let config2 = Config::default();
    println!("{:?}", config2);
}
```

### Власна реалізація Default

Часто "нульові" значення — не те, що потрібно. Тоді реалізуємо Default вручну.

**Постановка задачі:** Створити конфігурацію агента з розумними значеннями за замовчуванням.

```rust
#[derive(Debug)]
struct AgentConfig {
    max_speed: f64,
    max_altitude: f64,
    battery_threshold: u8,
    name: String,
}

impl Default for AgentConfig {
    fn default() -> Self {
        Self {
            max_speed: 20.0,         // Не 0, а розумне значення
            max_altitude: 500.0,
            battery_threshold: 20,
            name: String::from("Безіменний"),
        }
    }
}

fn main() {
    let default_config = AgentConfig::default();
    println!("{:?}", default_config);
    // AgentConfig { max_speed: 20.0, max_altitude: 500.0, battery_threshold: 20, name: "Безіменний" }
}
```

### Struct update syntax з Default

Потужний патерн: задати тільки потрібні поля, решта — з Default.

```rust
#[derive(Debug)]
struct ServerConfig {
    host: String,
    port: u16,
    timeout_ms: u32,
    max_connections: usize,
    use_tls: bool,
}

impl Default for ServerConfig {
    fn default() -> Self {
        Self {
            host: "localhost".to_string(),
            port: 8080,
            timeout_ms: 30000,
            max_connections: 100,
            use_tls: false,
        }
    }
}

fn main() {
    // Задаємо тільки потрібні поля, решта — default
    let custom = ServerConfig {
        port: 3000,
        use_tls: true,
        ..Default::default()  // Заповнити решту з default
    };
    
    println!("{:?}", custom);
    // host: "localhost", port: 3000, timeout_ms: 30000, max_connections: 100, use_tls: true
}
```

---

## 22.7 Traits порівняння: PartialEq та Eq

### PartialEq — порівняння на рівність

`PartialEq` дозволяє використовувати `==` та `!=`.

```rust
#[derive(Debug, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p1 = Point { x: 10, y: 20 };
    let p2 = Point { x: 10, y: 20 };
    let p3 = Point { x: 5, y: 5 };
    
    println!("p1 == p2: {}", p1 == p2);  // true
    println!("p1 == p3: {}", p1 == p3);  // false
    println!("p1 != p3: {}", p1 != p3);  // true
}
```

### Чому "Partial"?

`PartialEq` — **часткова** рівність. Не всі значення можна порівняти. Класичний приклад — `f64::NAN`:

```rust
fn main() {
    let nan = f64::NAN;
    
    // За IEEE 754, NaN не дорівнює нічому, навіть собі!
    println!("NaN == NaN: {}", nan == nan);  // false!
    println!("NaN != NaN: {}", nan != nan);  // true!
    
    // Тому f64 реалізує PartialEq, але НЕ Eq
}
```

### Eq — повна рівність

`Eq` — marker trait (без методів), що гарантує: **a == a завжди true** (рефлексивність).

**Важливо:** Для використання типу як ключа `HashMap` потрібен `Eq`.

```rust
use std::collections::HashMap;

// Для HashMap потрібні Eq + Hash
#[derive(Debug, PartialEq, Eq, Hash)]
struct AgentId {
    id: u32,
}

fn main() {
    let mut positions: HashMap<AgentId, (f64, f64)> = HashMap::new();
    
    positions.insert(AgentId { id: 1 }, (10.0, 20.0));
    positions.insert(AgentId { id: 2 }, (30.0, 40.0));
    
    if let Some(pos) = positions.get(&AgentId { id: 1 }) {
        println!("Агент 1 на позиції {:?}", pos);
    }
}
```

### Власна реалізація PartialEq

Іноді потрібно порівнювати не всі поля.

**Постановка задачі:** Агенти рівні, якщо мають однаковий `id`, незалежно від інших полів.

```rust
#[derive(Debug)]
struct Agent {
    id: String,
    battery: u8,
    position: (f64, f64),
}

// Порівнюємо ТІЛЬКИ за id
impl PartialEq for Agent {
    fn eq(&self, other: &Self) -> bool {
        self.id == other.id
    }
}

impl Eq for Agent {}  // Marker trait — порожня реалізація

fn main() {
    let agent1 = Agent {
        id: "SCOUT-001".to_string(),
        battery: 100,
        position: (0.0, 0.0),
    };
    
    let agent2 = Agent {
        id: "SCOUT-001".to_string(),
        battery: 50,           // Інша батарея
        position: (100.0, 200.0),  // Інша позиція
    };
    
    // Рівні за id, хоча інші поля різні!
    println!("agent1 == agent2: {}", agent1 == agent2);  // true
}
```

---

## 22.8 Trait Hash — для HashMap

### Навіщо Hash

Для використання типу як ключа `HashMap` або елемента `HashSet` потрібні:
- `Eq` — для порівняння ключів
- `Hash` — для обчислення хеш-значення

```rust
use std::collections::HashMap;

// Derive для всіх трьох traits
#[derive(Debug, PartialEq, Eq, Hash)]
struct Coordinate {
    x: i32,
    y: i32,
}

fn main() {
    let mut map: HashMap<Coordinate, String> = HashMap::new();
    
    map.insert(Coordinate { x: 0, y: 0 }, "Початок".to_string());
    map.insert(Coordinate { x: 1, y: 1 }, "Діагональ".to_string());
    
    println!("{:?}", map.get(&Coordinate { x: 0, y: 0 }));  // Some("Початок")
}
```

### Критичне правило: Hash + Eq узгодженість

**ПРАВИЛО:** Якщо `a == b`, то `hash(a) == hash(b)`.

Порушення цього правила зламає HashMap!

```rust
use std::hash::{Hash, Hasher};

#[derive(Debug)]
struct Key {
    id: u32,
    name: String,
}

// ❌ ПОГАНО: PartialEq порівнює тільки id
impl PartialEq for Key {
    fn eq(&self, other: &Self) -> bool {
        self.id == other.id  // Тільки id!
    }
}

impl Eq for Key {}

// ❌ ПОГАНО: Hash використовує ОБА поля
impl Hash for Key {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.id.hash(state);
        self.name.hash(state);  // name теж!
    }
}

// Проблема: два ключі можуть бути equal (однаковий id),
// але мати різний hash (різний name)
// HashMap зламається!
```

**Правильно:**

```rust
// ✅ ПРАВИЛЬНО: Hash та PartialEq узгоджені
impl Hash for Key {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.id.hash(state);  // Тільки id — як у eq()
    }
}
```

---

## 22.9 Практичний приклад: агент з усіма traits

**Постановка задачі:** Створити повноцінний тип Agent з усіма потрібними traits для використання в колекціях, логуванні, порівнянні.

```rust
use std::fmt;
use std::hash::{Hash, Hasher};
use std::collections::{HashMap, HashSet};

// ═══════════════════════════════════════════════════════════════════════════
// POSITION — Copy, бо маленька та часто копіюється
// ═══════════════════════════════════════════════════════════════════════════

#[derive(Clone, Copy, PartialEq, Default)]
pub struct Position {
    pub x: f64,
    pub y: f64,
    pub z: f64,
}

impl Position {
    pub fn new(x: f64, y: f64, z: f64) -> Self {
        Self { x, y, z }
    }
}

impl fmt::Debug for Position {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "Pos({:.1}, {:.1}, {:.1})", self.x, self.y, self.z)
    }
}

impl fmt::Display for Position {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({:.1}, {:.1}, {:.1})", self.x, self.y, self.z)
    }
}

// ═══════════════════════════════════════════════════════════════════════════
// AGENT STATE — Clone для копіювання, PartialEq для порівняння
// ═══════════════════════════════════════════════════════════════════════════

#[derive(Debug, Clone, PartialEq, Eq, Default)]
pub enum AgentState {
    #[default]
    Grounded,
    Flying,
    Charging,
    Emergency(String),
}

impl fmt::Display for AgentState {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AgentState::Grounded => write!(f, "На землі"),
            AgentState::Flying => write!(f, "В польоті"),
            AgentState::Charging => write!(f, "Заряджається"),
            AgentState::Emergency(r) => write!(f, "Аварія: {}", r),
        }
    }
}

// ═══════════════════════════════════════════════════════════════════════════
// AGENT — Clone, власні PartialEq та Hash (тільки за id)
// ═══════════════════════════════════════════════════════════════════════════

#[derive(Clone)]
pub struct Agent {
    pub id: String,
    pub position: Position,
    pub battery: u8,
    pub state: AgentState,
}

impl Agent {
    pub fn new(id: &str) -> Self {
        Self {
            id: id.to_string(),
            position: Position::default(),
            battery: 100,
            state: AgentState::default(),
        }
    }
}

// Порівняння тільки за id
impl PartialEq for Agent {
    fn eq(&self, other: &Self) -> bool {
        self.id == other.id
    }
}

impl Eq for Agent {}

// Hash тільки за id (узгоджено з PartialEq!)
impl Hash for Agent {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.id.hash(state);
    }
}

impl Default for Agent {
    fn default() -> Self {
        Self::new("БЕЗІМЕННИЙ")
    }
}

impl fmt::Debug for Agent {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_struct("Agent")
            .field("id", &self.id)
            .field("position", &self.position)
            .field("battery", &self.battery)
            .field("state", &self.state)
            .finish()
    }
}

impl fmt::Display for Agent {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(
            f,
            "[{}] {} | Поз: {} | Бат: {}%",
            self.id, self.state, self.position, self.battery
        )
    }
}

// ═══════════════════════════════════════════════════════════════════════════
// ДЕМОНСТРАЦІЯ
// ═══════════════════════════════════════════════════════════════════════════

fn main() {
    println!("╔══════════════════════════════════════════════════════════════╗");
    println!("║           ДЕМОНСТРАЦІЯ СТАНДАРТНИХ TRAITS                     ║");
    println!("╚══════════════════════════════════════════════════════════════╝\n");
    
    // Debug та Display
    println!("=== Debug та Display ===");
    let agent = Agent::new("SCOUT-001");
    println!("Debug:   {:?}", agent);
    println!("Display: {}", agent);
    
    // Clone
    println!("\n=== Clone ===");
    let mut copy = agent.clone();
    copy.battery = 50;
    println!("Оригінал: {}", agent);
    println!("Копія:    {}", copy);
    
    // Default
    println!("\n=== Default ===");
    let default_agent: Agent = Default::default();
    println!("Default: {}", default_agent);
    
    // PartialEq (порівняння за id)
    println!("\n=== PartialEq ===");
    let agent1 = Agent::new("SCOUT-001");
    let agent2 = Agent::new("SCOUT-001");
    let agent3 = Agent::new("SCOUT-002");
    
    println!("agent1 == agent2: {}", agent1 == agent2);  // true
    println!("agent1 == agent3: {}", agent1 == agent3);  // false
    
    // Hash (для HashMap)
    println!("\n=== Hash (HashMap) ===");
    let mut missions: HashMap<Agent, String> = HashMap::new();
    missions.insert(Agent::new("SCOUT-001"), "Патруль".to_string());
    missions.insert(Agent::new("SCOUT-002"), "Розвідка".to_string());
    
    for (agent, mission) in &missions {
        println!("{}: {}", agent.id, mission);
    }
    
    // HashSet — дублікати ігноруються
    println!("\n=== HashSet ===");
    let mut active: HashSet<Agent> = HashSet::new();
    active.insert(Agent::new("SCOUT-001"));
    active.insert(Agent::new("SCOUT-002"));
    active.insert(Agent::new("SCOUT-001"));  // Дублікат!
    
    println!("Активних агентів: {}", active.len());  // 2, не 3
    
    // Position Copy
    println!("\n=== Position Copy ===");
    let pos = Position::new(10.0, 20.0, 30.0);
    let pos2 = pos;  // Copy, не move
    println!("pos: {}, pos2: {}", pos, pos2);  // Обидва доступні
}
```

---

## 22.10 Лабораторна робота

**Завдання:** Реалізувати стандартні traits для типів агента.

### Частина 1: Debug та Display (3 бали)

```rust
struct SensorReading {
    sensor_id: String,
    value: f64,
    unit: String,
    timestamp: u64,
}

// Debug: SensorReading { sensor_id: "TEMP-01", value: 23.5, ... }
// Display: [TEMP-01] 23.5 °C @ 1705123456
```

### Частина 2: Clone та Default (3 бали)

```rust
struct MissionConfig {
    name: String,
    waypoints: Vec<Position>,
    max_duration: u32,
    return_on_low_battery: bool,
}

// Default з розумними значеннями
// Clone для копіювання
```

### Частина 3: PartialEq, Eq, Hash (4 бали)

```rust
struct Command {
    id: u64,
    command_type: String,
    parameters: Vec<String>,
    timestamp: u64,
}

// Порівняння та hash ТІЛЬКИ за id
// Використання як ключ HashMap
```

**Критерії оцінювання:**

| Критерій | Бали |
|----------|------|
| Debug та Display | 3 |
| Clone та Default | 3 |
| PartialEq, Eq, Hash | 4 |
| **Максимум** | **10** |

---

## Підсумок

Стандартні traits — фундамент типової системи Rust:

| Trait | Призначення | Derive? |
|-------|------------|---------|
| **Debug** | Вивід `{:?}` для розробників | ✅ Так |
| **Display** | Вивід `{}` для користувачів | ❌ Тільки вручну |
| **Clone** | Явне копіювання `.clone()` | ✅ Так |
| **Copy** | Неявне копіювання | ✅ Якщо всі поля Copy |
| **Default** | Значення за замовчуванням | ✅ Так (нулі) |
| **PartialEq** | Порівняння `==` / `!=` | ✅ Так |
| **Eq** | Гарантія рефлексивності | ✅ Так |
| **Hash** | Для HashMap/HashSet | ✅ Так |

**Ключові правила:**
- Debug — майже завжди через derive
- Display — тільки вручну
- Copy — тільки для маленьких типів без String/Vec
- Hash + Eq мають бути узгоджені!

---

## Зв'язок з наступним розділом

Ви познайомились зі стандартними traits. Тепер час створити **власні** traits для абстрагування поведінки агентів.

У **Розділі 23: Трейти — власні абстракції** ви дізнаєтесь:
- Як створювати власні traits
- Методи за замовчуванням
- Associated types
- Trait objects та динамічний поліморфізм
