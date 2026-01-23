# Практичне заняття 8: Структури та Enum

## Мета заняття

Після цього заняття ви зможете:
- Створювати власні типи даних за допомогою структур (struct)
- Додавати методи до структур через impl блоки
- Використовувати enum для представлення варіантів
- Застосовувати pattern matching для обробки enum
- Працювати зі стандартними типами Option та Result

---

## Теоретичний вступ

### Навіщо потрібні власні типи?

До цього ми використовували примітивні типи та кортежі:

```rust
// Незручно та неочевидно
let drone = ("Alpha", 100, 50, true);  // Що означає кожне поле?
```

Структури дозволяють створити **іменований тип** з **іменованими полями**:

```rust
struct Drone {
    name: String,
    x: i32,
    y: i32,
    battery: i32,
    active: bool,
}
```

Тепер код читається зрозуміліше, а компілятор допомагає уникнути помилок.

---

## Структури (Structs)

### Оголошення структури

```rust
struct Drone {
    name: String,
    x: i32,
    y: i32,
    battery: i32,
    active: bool,
}
```

**Синтаксис:**
- `struct` — ключове слово
- `Drone` — назва типу (PascalCase)
- `{ }` — блок з полями
- `name: String` — ім'я поля та його тип

### Створення екземпляра

```rust
fn main() {
    let drone = Drone {
        name: String::from("Alpha"),
        x: 0,
        y: 0,
        battery: 100,
        active: true,
    };
    
    println!("Дрон: {}", drone.name);
}
```

**Важливо:** всі поля мають бути ініціалізовані при створенні.

### Доступ до полів

```rust
fn main() {
    let drone = Drone {
        name: String::from("Beta"),
        x: 10,
        y: 20,
        battery: 85,
        active: false,
    };
    
    println!("Ім'я: {}", drone.name);
    println!("Позиція: ({}, {})", drone.x, drone.y);
    println!("Батарея: {}%", drone.battery);
    println!("Активний: {}", drone.active);
}
```

### Мутабельні структури

Щоб змінювати поля, структура має бути `mut`:

```rust
fn main() {
    let mut drone = Drone {
        name: String::from("Gamma"),
        x: 0,
        y: 0,
        battery: 100,
        active: true,
    };
    
    // Змінюємо поля
    drone.x = 50;
    drone.y = 30;
    drone.battery -= 10;
    
    println!("Нова позиція: ({}, {})", drone.x, drone.y);
}
```

**Примітка:** не можна зробити окремі поля мутабельними — вся структура або `mut`, або ні.

### Скорочений синтаксис ініціалізації

Якщо змінна має те саме ім'я, що й поле:

```rust
fn create_drone(name: String, x: i32, y: i32) -> Drone {
    Drone {
        name,     // Замість name: name
        x,        // Замість x: x
        y,        // Замість y: y
        battery: 100,
        active: true,
    }
}
```

### Оновлення структури (struct update syntax)

Створення нової структури на основі існуючої:

```rust
fn main() {
    let drone1 = Drone {
        name: String::from("Original"),
        x: 10,
        y: 20,
        battery: 80,
        active: true,
    };
    
    // Копіюємо всі поля, крім зазначених
    let drone2 = Drone {
        name: String::from("Clone"),
        ..drone1  // Решта полів з drone1
    };
    
    // Увага: drone1.name переміщено, але drone1.x, drone1.y, etc. скопійовані (Copy типи)
}
```

---

## Методи (impl блоки)

### Що таке методи?

Методи — це функції, пов'язані зі структурою. Вони визначаються в блоці `impl`:

```rust
struct Drone {
    name: String,
    x: i32,
    y: i32,
    battery: i32,
}

impl Drone {
    // Метод — перший параметр &self або &mut self
    fn status(&self) -> String {
        format!("Дрон '{}' на ({}, {}), батарея: {}%", 
                self.name, self.x, self.y, self.battery)
    }
}

fn main() {
    let drone = Drone {
        name: String::from("Alpha"),
        x: 10,
        y: 20,
        battery: 85,
    };
    
    println!("{}", drone.status());  // Виклик методу
}
```

### self, &self, &mut self

```rust
impl Drone {
    // &self — іммутабельне посилання (читання)
    fn get_position(&self) -> (i32, i32) {
        (self.x, self.y)
    }
    
    // &mut self — мутабельне посилання (зміна)
    fn move_to(&mut self, x: i32, y: i32) {
        self.x = x;
        self.y = y;
    }
    
    // self — забирає власність (рідко використовується)
    fn destroy(self) -> String {
        format!("Дрон '{}' знищено", self.name)
        // self більше не доступний після виклику
    }
}
```

### Асоційовані функції (конструктори)

Функції без `self` — це асоційовані функції. Часто використовуються як конструктори:

```rust
impl Drone {
    // Асоційована функція — немає self
    fn new(name: String) -> Drone {
        Drone {
            name,
            x: 0,
            y: 0,
            battery: 100,
        }
    }
    
    fn with_position(name: String, x: i32, y: i32) -> Drone {
        Drone {
            name,
            x,
            y,
            battery: 100,
        }
    }
}

fn main() {
    // Виклик через ::
    let drone1 = Drone::new(String::from("Alpha"));
    let drone2 = Drone::with_position(String::from("Beta"), 50, 100);
}
```

### Кілька impl блоків

Можна мати кілька `impl` блоків для однієї структури:

```rust
impl Drone {
    fn new(name: String) -> Drone { /* ... */ }
}

impl Drone {
    fn status(&self) -> String { /* ... */ }
}
```

Це корисно для організації коду або при використанні traits (пізніше).

---

## Tuple Structs та Unit Structs

### Tuple Struct

Структура без іменованих полів:

```rust
struct Point(i32, i32, i32);
struct Color(u8, u8, u8);

fn main() {
    let pos = Point(10, 20, 50);
    let red = Color(255, 0, 0);
    
    println!("X: {}", pos.0);
    println!("Red: {}", red.0);
}
```

### Unit Struct

Структура без полів (маркер):

```rust
struct Marker;

fn main() {
    let _m = Marker;
}
```

---

## Enum (Перелічення)

### Що таке Enum?

Enum дозволяє визначити тип, який може бути **одним з кількох варіантів**:

```rust
enum DroneState {
    Idle,
    Flying,
    Charging,
    Emergency,
}

fn main() {
    let state = DroneState::Flying;
}
```

### Enum з даними

Кожен варіант може містити різні дані:

```rust
enum DroneCommand {
    TakeOff,                          // Без даних
    MoveTo(i32, i32),                 // Tuple-like
    SetAltitude { height: i32 },      // Struct-like
    Land,
    EmergencyStop(String),            // З повідомленням
}

fn main() {
    let cmd1 = DroneCommand::TakeOff;
    let cmd2 = DroneCommand::MoveTo(100, 200);
    let cmd3 = DroneCommand::SetAltitude { height: 50 };
    let cmd4 = DroneCommand::EmergencyStop(String::from("Obstacle detected"));
}
```

### Методи для Enum

```rust
impl DroneCommand {
    fn describe(&self) -> String {
        match self {
            DroneCommand::TakeOff => String::from("Злет"),
            DroneCommand::MoveTo(x, y) => format!("Рух до ({}, {})", x, y),
            DroneCommand::SetAltitude { height } => format!("Висота: {} м", height),
            DroneCommand::Land => String::from("Посадка"),
            DroneCommand::EmergencyStop(msg) => format!("СТОП: {}", msg),
        }
    }
}
```

---

## Pattern Matching з match

### Базовий match

```rust
fn process_state(state: DroneState) {
    match state {
        DroneState::Idle => println!("Дрон очікує"),
        DroneState::Flying => println!("Дрон летить"),
        DroneState::Charging => println!("Дрон заряджається"),
        DroneState::Emergency => println!("АВАРІЯ!"),
    }
}
```

**Важливо:** `match` має бути **вичерпним** — покривати всі варіанти.

### match з витягуванням даних

```rust
fn execute_command(cmd: DroneCommand) {
    match cmd {
        DroneCommand::TakeOff => {
            println!("Запуск двигунів...");
        },
        DroneCommand::MoveTo(x, y) => {
            println!("Переміщення до точки ({}, {})", x, y);
        },
        DroneCommand::SetAltitude { height } => {
            println!("Зміна висоти на {} метрів", height);
        },
        DroneCommand::Land => {
            println!("Процедура посадки...");
        },
        DroneCommand::EmergencyStop(reason) => {
            println!("АВАРІЙНА ЗУПИНКА: {}", reason);
        },
    }
}
```

### Wildcard pattern (_)

```rust
fn is_emergency(state: DroneState) -> bool {
    match state {
        DroneState::Emergency => true,
        _ => false,  // Всі інші варіанти
    }
}
```

### match як вираз

```rust
fn state_priority(state: DroneState) -> i32 {
    match state {
        DroneState::Emergency => 0,   // Найвищий пріоритет
        DroneState::Flying => 1,
        DroneState::Charging => 2,
        DroneState::Idle => 3,
    }
}
```

---

## Option<T> — обробка відсутніх значень

### Що таке Option?

`Option<T>` — стандартний enum для представлення "може бути значення, а може ні":

```rust
enum Option<T> {
    Some(T),
    None,
}
```

### Використання Option

```rust
fn find_drone(drones: &[Drone], name: &str) -> Option<&Drone> {
    for drone in drones {
        if drone.name == name {
            return Some(drone);
        }
    }
    None
}

fn main() {
    let drones = vec![
        Drone::new(String::from("Alpha")),
        Drone::new(String::from("Beta")),
    ];
    
    match find_drone(&drones, "Alpha") {
        Some(drone) => println!("Знайдено: {}", drone.name),
        None => println!("Не знайдено"),
    }
}
```

### Методи Option

```rust
fn main() {
    let some_value: Option<i32> = Some(42);
    let no_value: Option<i32> = None;
    
    // unwrap — паніка якщо None
    let x = some_value.unwrap();  // 42
    
    // unwrap_or — значення за замовчуванням
    let y = no_value.unwrap_or(0);  // 0
    
    // is_some / is_none
    println!("Has value: {}", some_value.is_some());  // true
    
    // map — трансформація
    let doubled = some_value.map(|v| v * 2);  // Some(84)
}
```

---

## if let та while let

### if let — спрощений match

Замість:
```rust
match some_option {
    Some(value) => println!("Got: {}", value),
    None => {},
}
```

Можна написати:
```rust
if let Some(value) = some_option {
    println!("Got: {}", value);
}
```

### while let — цикл з pattern matching

```rust
fn main() {
    let mut stack = vec![1, 2, 3, 4, 5];
    
    while let Some(top) = stack.pop() {
        println!("Елемент: {}", top);
    }
}
```

---

## Типові помилки

### Помилка 1: Не всі поля ініціалізовані

```rust
let drone = Drone {
    name: String::from("Alpha"),
    x: 0,
    // Забули y, battery, active!
};
```

**Виправлення:** Ініціалізуйте всі поля або використайте `..Default::default()` (якщо реалізовано Default).

### Помилка 2: Зміна незмінної структури

```rust
let drone = Drone::new(String::from("Alpha"));
drone.x = 10;  // ПОМИЛКА! drone не mut
```

**Виправлення:** `let mut drone = ...`

### Помилка 3: Невичерпний match

```rust
match state {
    DroneState::Idle => println!("Idle"),
    DroneState::Flying => println!("Flying"),
    // Забули Charging та Emergency!
}
```

**Виправлення:** Додайте всі варіанти або `_ => ...`

### Помилка 4: Використання Option без перевірки

```rust
let value: Option<i32> = None;
let x = value.unwrap();  // ПАНІКА!
```

**Виправлення:** Використайте `match`, `if let`, або `unwrap_or`.

---

## Практичні задачі

### Задача 1: Базова структура дрона

**Умова:** Створіть структуру `Drone` з полями: name (String), x (i32), y (i32), altitude (i32), battery (i32), active (bool). Реалізуйте методи: `new`, `status`, `distance_from_origin`.

**Розв'язання:**

```rust
struct Drone {
    name: String,
    x: i32,
    y: i32,
    altitude: i32,
    battery: i32,
    active: bool,
}

impl Drone {
    // Конструктор
    fn new(name: String) -> Drone {
        Drone {
            name,
            x: 0,
            y: 0,
            altitude: 0,
            battery: 100,
            active: false,
        }
    }
    
    // Конструктор з позицією
    fn with_position(name: String, x: i32, y: i32, altitude: i32) -> Drone {
        Drone {
            name,
            x,
            y,
            altitude,
            battery: 100,
            active: true,
        }
    }
    
    // Статус дрона
    fn status(&self) -> String {
        let state = if self.active { "АКТИВНИЙ" } else { "НЕАКТИВНИЙ" };
        format!(
            "Дрон '{}' [{}]\n  Позиція: ({}, {}, {})\n  Батарея: {}%",
            self.name, state, self.x, self.y, self.altitude, self.battery
        )
    }
    
    // Відстань від початку координат
    fn distance_from_origin(&self) -> f64 {
        let x = self.x as f64;
        let y = self.y as f64;
        let z = self.altitude as f64;
        (x*x + y*y + z*z).sqrt()
    }
}

fn main() {
    println!("=== Базова структура дрона ===\n");
    
    // Створення дронів
    let drone1 = Drone::new(String::from("Alpha"));
    let drone2 = Drone::with_position(String::from("Beta"), 30, 40, 50);
    
    // Виведення статусу
    println!("{}\n", drone1.status());
    println!("{}\n", drone2.status());
    
    // Відстань
    println!("Відстань Alpha від бази: {:.2}", drone1.distance_from_origin());
    println!("Відстань Beta від бази: {:.2}", drone2.distance_from_origin());
}
```

**Пояснення:**

1. `struct Drone { }` — визначення структури з полями
2. `impl Drone { }` — блок з методами
3. `fn new(name: String) -> Drone` — конструктор (асоційована функція)
4. `fn status(&self)` — метод, що читає дані
5. `self.x` — доступ до полів через self

**Результат:**
```
=== Базова структура дрона ===

Дрон 'Alpha' [НЕАКТИВНИЙ]
  Позиція: (0, 0, 0)
  Батарея: 100%

Дрон 'Beta' [АКТИВНИЙ]
  Позиція: (30, 40, 50)
  Батарея: 100%

Відстань Alpha від бази: 0.00
Відстань Beta від бази: 70.71
```

---

### Задача 2: Методи модифікації

**Умова:** Додайте до структури `Drone` методи: `activate`, `deactivate`, `move_to`, `change_altitude`, `consume_battery`. Всі методи модифікації мають приймати `&mut self`.

**Розв'язання:**

```rust
struct Drone {
    name: String,
    x: i32,
    y: i32,
    altitude: i32,
    battery: i32,
    active: bool,
}

impl Drone {
    fn new(name: String) -> Drone {
        Drone {
            name,
            x: 0,
            y: 0,
            altitude: 0,
            battery: 100,
            active: false,
        }
    }
    
    fn activate(&mut self) {
        if self.battery > 10 {
            self.active = true;
            println!("[{}] Активовано", self.name);
        } else {
            println!("[{}] Недостатньо батареї для активації!", self.name);
        }
    }
    
    fn deactivate(&mut self) {
        self.active = false;
        self.altitude = 0;  // Посадка
        println!("[{}] Деактивовано", self.name);
    }
    
    fn move_to(&mut self, x: i32, y: i32) -> bool {
        if !self.active {
            println!("[{}] Помилка: дрон неактивний!", self.name);
            return false;
        }
        
        let distance = (((x - self.x).pow(2) + (y - self.y).pow(2)) as f64).sqrt();
        let cost = (distance / 10.0) as i32 + 1;
        
        if self.battery < cost {
            println!("[{}] Недостатньо батареї для переміщення!", self.name);
            return false;
        }
        
        println!("[{}] Переміщення ({}, {}) → ({}, {}), витрата: {}%", 
                 self.name, self.x, self.y, x, y, cost);
        
        self.x = x;
        self.y = y;
        self.battery -= cost;
        true
    }
    
    fn change_altitude(&mut self, delta: i32) -> bool {
        if !self.active {
            println!("[{}] Помилка: дрон неактивний!", self.name);
            return false;
        }
        
        let new_altitude = self.altitude + delta;
        
        if new_altitude < 0 {
            println!("[{}] Помилка: висота не може бути від'ємною!", self.name);
            return false;
        }
        
        let cost = delta.abs() / 10 + 1;
        
        if self.battery < cost {
            println!("[{}] Недостатньо батареї!", self.name);
            return false;
        }
        
        println!("[{}] Зміна висоти: {} → {}, витрата: {}%", 
                 self.name, self.altitude, new_altitude, cost);
        
        self.altitude = new_altitude;
        self.battery -= cost;
        true
    }
    
    fn charge(&mut self, amount: i32) {
        let old = self.battery;
        self.battery = (self.battery + amount).min(100);
        println!("[{}] Зарядка: {}% → {}%", self.name, old, self.battery);
    }
    
    fn status(&self) -> String {
        let state = if self.active { "●" } else { "○" };
        format!("{} {} | ({:>3},{:>3},{:>3}) | 🔋{}%", 
                state, self.name, self.x, self.y, self.altitude, self.battery)
    }
}

fn main() {
    println!("=== Методи модифікації ===\n");
    
    let mut drone = Drone::new(String::from("Explorer"));
    
    println!("Початок: {}\n", drone.status());
    
    // Спроба рухатись без активації
    drone.move_to(10, 10);
    
    // Активація та рух
    drone.activate();
    drone.change_altitude(50);
    drone.move_to(30, 40);
    drone.move_to(100, 100);
    
    println!("\nПоточний стан: {}\n", drone.status());
    
    // Зарядка та ще рух
    drone.charge(50);
    drone.move_to(0, 0);
    drone.change_altitude(-50);
    drone.deactivate();
    
    println!("\nФінал: {}", drone.status());
}
```

**Пояснення:**

1. `&mut self` — метод може змінювати поля структури
2. Кожен метод перевіряє умови (активність, батарея) перед дією
3. Методи повертають `bool` для індикації успіху
4. `self.battery = (self.battery + amount).min(100)` — обмеження максимуму

**Результат:**
```
=== Методи модифікації ===

Початок: ○ Explorer | (  0,  0,  0) | 🔋100%

[Explorer] Помилка: дрон неактивний!
[Explorer] Активовано
[Explorer] Зміна висоти: 0 → 50, витрата: 6%
[Explorer] Переміщення (0, 0) → (30, 40), витрата: 6%
[Explorer] Переміщення (30, 40) → (100, 100), витрата: 10%

Поточний стан: ● Explorer | (100,100, 50) | 🔋78%

[Explorer] Зарядка: 78% → 100%
[Explorer] Переміщення (100, 100) → (0, 0), витрата: 15%
[Explorer] Зміна висоти: 50 → 0, витрата: 6%
[Explorer] Деактивовано

Фінал: ○ Explorer | (  0,  0,  0) | 🔋79%
```

---

### Задача 3: Enum станів дрона

**Умова:** Створіть enum `DroneState` з варіантами: Idle, Flying { altitude: i32 }, Charging { percent: i32 }, Emergency(String). Реалізуйте методи `describe` та `priority`.

**Розв'язання:**

```rust
#[derive(Debug, Clone)]
enum DroneState {
    Idle,
    Flying { altitude: i32, speed: i32 },
    Charging { current: i32, target: i32 },
    Returning { x: i32, y: i32 },
    Emergency(String),
}

impl DroneState {
    fn describe(&self) -> String {
        match self {
            DroneState::Idle => {
                String::from("⏸️  Очікування — дрон готовий до команд")
            },
            DroneState::Flying { altitude, speed } => {
                format!("🚁 Політ — висота {} м, швидкість {} м/с", altitude, speed)
            },
            DroneState::Charging { current, target } => {
                let progress = ((*current as f64 / *target as f64) * 100.0) as i32;
                format!("🔋 Зарядка — {}% з {}% (прогрес: {}%)", current, target, progress)
            },
            DroneState::Returning { x, y } => {
                format!("🏠 Повернення на базу — ціль ({}, {})", x, y)
            },
            DroneState::Emergency(reason) => {
                format!("🚨 АВАРІЯ — {}", reason)
            },
        }
    }
    
    fn priority(&self) -> i32 {
        match self {
            DroneState::Emergency(_) => 0,  // Найвищий
            DroneState::Returning { .. } => 1,
            DroneState::Flying { .. } => 2,
            DroneState::Charging { .. } => 3,
            DroneState::Idle => 4,  // Найнижчий
        }
    }
    
    fn is_operational(&self) -> bool {
        match self {
            DroneState::Emergency(_) => false,
            _ => true,
        }
    }
    
    fn can_accept_commands(&self) -> bool {
        match self {
            DroneState::Idle => true,
            DroneState::Flying { .. } => true,
            _ => false,
        }
    }
}

fn main() {
    println!("=== Enum станів дрона ===\n");
    
    let states = vec![
        DroneState::Idle,
        DroneState::Flying { altitude: 100, speed: 15 },
        DroneState::Charging { current: 45, target: 100 },
        DroneState::Returning { x: 0, y: 0 },
        DroneState::Emergency(String::from("Втрата GPS сигналу")),
    ];
    
    println!("Всі стани:\n");
    for state in &states {
        println!("Пріоритет {}: {}", state.priority(), state.describe());
        println!("  Операційний: {}, Приймає команди: {}\n", 
                 state.is_operational(), state.can_accept_commands());
    }
    
    // Сортування за пріоритетом
    let mut sorted = states.clone();
    sorted.sort_by_key(|s| s.priority());
    
    println!("--- Відсортовано за пріоритетом ---\n");
    for state in &sorted {
        println!("[P{}] {}", state.priority(), state.describe());
    }
}
```

**Пояснення:**

1. `enum DroneState { }` — визначення enum з різними варіантами
2. Варіанти можуть мати дані: tuple-style або struct-style
3. `match self { }` — обробка кожного варіанту
4. `DroneState::Flying { altitude, speed }` — деструктуризація в match
5. `#[derive(Debug, Clone)]` — автоматична реалізація traits

**Результат:**
```
=== Enum станів дрона ===

Всі стани:

Пріоритет 4: ⏸️  Очікування — дрон готовий до команд
  Операційний: true, Приймає команди: true

Пріоритет 2: 🚁 Політ — висота 100 м, швидкість 15 м/с
  Операційний: true, Приймає команди: true

Пріоритет 3: 🔋 Зарядка — 45% з 100% (прогрес: 45%)
  Операційний: true, Приймає команди: false

Пріоритет 1: 🏠 Повернення на базу — ціль (0, 0)
  Операційний: true, Приймає команди: false

Пріоритет 0: 🚨 АВАРІЯ — Втрата GPS сигналу
  Операційний: false, Приймає команди: false

--- Відсортовано за пріоритетом ---

[P0] 🚨 АВАРІЯ — Втрата GPS сигналу
[P1] 🏠 Повернення на базу — ціль (0, 0)
[P2] 🚁 Політ — висота 100 м, швидкість 15 м/с
[P3] 🔋 Зарядка — 45% з 100% (прогрес: 45%)
[P4] ⏸️  Очікування — дрон готовий до команд
```

---

### Задача 4: Повноцінна модель дрона

**Умова:** Об'єднайте структуру `Drone` з enum `DroneState`. Дрон має стан, і його методи мають змінювати стан відповідно до дій.

**Розв'язання:**

```rust
#[derive(Debug, Clone)]
enum DroneState {
    Idle,
    Flying { altitude: i32 },
    Charging,
    Emergency(String),
}

struct Drone {
    name: String,
    x: i32,
    y: i32,
    battery: i32,
    state: DroneState,
}

impl Drone {
    fn new(name: String) -> Drone {
        Drone {
            name,
            x: 0,
            y: 0,
            battery: 100,
            state: DroneState::Idle,
        }
    }
    
    fn take_off(&mut self, altitude: i32) -> Result<(), String> {
        match &self.state {
            DroneState::Idle => {
                if self.battery < 20 {
                    return Err(String::from("Недостатньо батареї для зльоту"));
                }
                self.state = DroneState::Flying { altitude };
                self.battery -= 5;
                Ok(())
            },
            DroneState::Emergency(reason) => {
                Err(format!("Неможливо злетіти: аварія ({})", reason))
            },
            _ => Err(String::from("Неможливо злетіти з поточного стану")),
        }
    }
    
    fn land(&mut self) -> Result<(), String> {
        match &self.state {
            DroneState::Flying { .. } => {
                self.state = DroneState::Idle;
                self.battery -= 3;
                Ok(())
            },
            _ => Err(String::from("Дрон не в польоті")),
        }
    }
    
    fn move_to(&mut self, x: i32, y: i32) -> Result<(), String> {
        match &self.state {
            DroneState::Flying { altitude } => {
                let distance = (((x - self.x).pow(2) + (y - self.y).pow(2)) as f64).sqrt();
                let cost = (distance / 10.0) as i32 + 1;
                
                if self.battery < cost + 10 {  // Резерв для посадки
                    self.state = DroneState::Emergency(String::from("Критичний рівень батареї"));
                    return Err(String::from("Недостатньо батареї!"));
                }
                
                self.x = x;
                self.y = y;
                self.battery -= cost;
                
                // Зберігаємо поточну висоту
                self.state = DroneState::Flying { altitude: *altitude };
                Ok(())
            },
            _ => Err(String::from("Дрон не в польоті")),
        }
    }
    
    fn start_charging(&mut self) -> Result<(), String> {
        match &self.state {
            DroneState::Idle => {
                self.state = DroneState::Charging;
                Ok(())
            },
            _ => Err(String::from("Можна заряджати тільки в режимі очікування")),
        }
    }
    
    fn charge_tick(&mut self) -> Result<bool, String> {
        match &self.state {
            DroneState::Charging => {
                self.battery = (self.battery + 10).min(100);
                if self.battery == 100 {
                    self.state = DroneState::Idle;
                    Ok(true)  // Зарядка завершена
                } else {
                    Ok(false)  // Ще заряджається
                }
            },
            _ => Err(String::from("Дрон не заряджається")),
        }
    }
    
    fn emergency(&mut self, reason: String) {
        self.state = DroneState::Emergency(reason);
    }
    
    fn reset(&mut self) -> Result<(), String> {
        match &self.state {
            DroneState::Emergency(_) => {
                self.state = DroneState::Idle;
                Ok(())
            },
            _ => Err(String::from("Скидання можливе тільки в аварійному режимі")),
        }
    }
    
    fn status(&self) -> String {
        let state_str = match &self.state {
            DroneState::Idle => String::from("⏸️  Очікування"),
            DroneState::Flying { altitude } => format!("🚁 Політ ({}м)", altitude),
            DroneState::Charging => format!("🔋 Зарядка ({}%)", self.battery),
            DroneState::Emergency(reason) => format!("🚨 {}", reason),
        };
        
        format!("[{}] {} | ({}, {}) | 🔋{}%", 
                self.name, state_str, self.x, self.y, self.battery)
    }
}

fn main() {
    println!("=== Повноцінна модель дрона ===\n");
    
    let mut drone = Drone::new(String::from("Falcon"));
    
    println!("Початок: {}\n", drone.status());
    
    // Місія
    println!("--- Виконання місії ---\n");
    
    // Зліт
    match drone.take_off(50) {
        Ok(()) => println!("✓ Зліт успішний"),
        Err(e) => println!("✗ Помилка: {}", e),
    }
    println!("  {}\n", drone.status());
    
    // Переміщення
    for (x, y) in [(100, 0), (100, 100), (50, 150)] {
        match drone.move_to(x, y) {
            Ok(()) => println!("✓ Переміщення до ({}, {})", x, y),
            Err(e) => println!("✗ Помилка: {}", e),
        }
        println!("  {}\n", drone.status());
    }
    
    // Посадка
    match drone.land() {
        Ok(()) => println!("✓ Посадка"),
        Err(e) => println!("✗ Помилка: {}", e),
    }
    println!("  {}\n", drone.status());
    
    // Зарядка
    println!("--- Зарядка ---\n");
    
    if let Err(e) = drone.start_charging() {
        println!("✗ Помилка: {}", e);
    }
    
    loop {
        match drone.charge_tick() {
            Ok(true) => {
                println!("✓ Зарядка завершена!");
                break;
            },
            Ok(false) => {
                println!("  Заряджається... {}%", drone.battery);
            },
            Err(e) => {
                println!("✗ Помилка: {}", e);
                break;
            }
        }
    }
    
    println!("\nФінал: {}", drone.status());
}
```

**Пояснення:**

1. `state: DroneState` — поле структури типу enum
2. `match &self.state { }` — перевірка поточного стану
3. `Result<(), String>` — повертаємо Ok або Err з повідомленням
4. Методи змінюють `self.state` залежно від дії
5. `if let Err(e) = ...` — обробка тільки помилки

**Результат:**
```
=== Повноцінна модель дрона ===

Початок: [Falcon] ⏸️  Очікування | (0, 0) | 🔋100%

--- Виконання місії ---

✓ Зліт успішний
  [Falcon] 🚁 Політ (50м) | (0, 0) | 🔋95%

✓ Переміщення до (100, 0)
  [Falcon] 🚁 Політ (50м) | (100, 0) | 🔋84%

✓ Переміщення до (100, 100)
  [Falcon] 🚁 Політ (50м) | (100, 100) | 🔋73%

✓ Переміщення до (50, 150)
  [Falcon] 🚁 Політ (50м) | (50, 150) | 🔋65%

✓ Посадка
  [Falcon] ⏸️  Очікування | (50, 150) | 🔋62%

--- Зарядка ---

  Заряджається... 72%
  Заряджається... 82%
  Заряджається... 92%
✓ Зарядка завершена!

Фінал: [Falcon] ⏸️  Очікування | (50, 150) | 🔋100%
```

---

## Домашнє завдання

### Завдання 1: Структура місії

**Умова:** Створіть структуру `Mission` з полями: name, waypoints (Vec точок), completed (bool), current_waypoint (usize). Реалізуйте методи для додавання точок, отримання наступної точки, відмітки прогресу.

**Розв'язання:**

```rust
#[derive(Debug, Clone)]
struct Waypoint {
    x: i32,
    y: i32,
    altitude: i32,
    name: String,
}

struct Mission {
    name: String,
    waypoints: Vec<Waypoint>,
    current_index: usize,
    completed: bool,
}

impl Waypoint {
    fn new(name: &str, x: i32, y: i32, altitude: i32) -> Waypoint {
        Waypoint {
            x,
            y,
            altitude,
            name: String::from(name),
        }
    }
}

impl Mission {
    fn new(name: String) -> Mission {
        Mission {
            name,
            waypoints: Vec::new(),
            current_index: 0,
            completed: false,
        }
    }
    
    fn add_waypoint(&mut self, waypoint: Waypoint) {
        self.waypoints.push(waypoint);
    }
    
    fn current_waypoint(&self) -> Option<&Waypoint> {
        if self.completed {
            return None;
        }
        self.waypoints.get(self.current_index)
    }
    
    fn advance(&mut self) -> bool {
        if self.completed {
            return false;
        }
        
        self.current_index += 1;
        
        if self.current_index >= self.waypoints.len() {
            self.completed = true;
        }
        
        true
    }
    
    fn progress(&self) -> f64 {
        if self.waypoints.is_empty() {
            return 0.0;
        }
        (self.current_index as f64 / self.waypoints.len() as f64) * 100.0
    }
    
    fn status(&self) -> String {
        let progress = self.progress();
        let state = if self.completed { "ЗАВЕРШЕНО" } else { "В ПРОЦЕСІ" };
        
        format!("Місія '{}' [{}] - {:.0}% ({}/{})", 
                self.name, state, progress, 
                self.current_index, self.waypoints.len())
    }
}

fn main() {
    println!("=== Структура місії ===\n");
    
    let mut mission = Mission::new(String::from("Патрулювання периметру"));
    
    // Додаємо точки маршруту
    mission.add_waypoint(Waypoint::new("Старт", 0, 0, 50));
    mission.add_waypoint(Waypoint::new("Північний кут", 0, 100, 75));
    mission.add_waypoint(Waypoint::new("Північно-східний кут", 100, 100, 75));
    mission.add_waypoint(Waypoint::new("Південно-східний кут", 100, 0, 50));
    mission.add_waypoint(Waypoint::new("Повернення", 0, 0, 25));
    
    println!("{}\n", mission.status());
    
    // Симуляція виконання
    while let Some(wp) = mission.current_waypoint() {
        println!("→ Рухаємось до: {} ({}, {}, {})", 
                 wp.name, wp.x, wp.y, wp.altitude);
        mission.advance();
        println!("  {}\n", mission.status());
    }
    
    println!("Місія завершена!");
}
```

---

### Завдання 2: Enum команд з обробкою

**Умова:** Створіть enum `Command` з варіантами різних команд дрона. Реалізуйте функцію `execute_command`, яка обробляє кожну команду та повертає Result.

**Розв'язання:**

```rust
enum Command {
    TakeOff { altitude: i32 },
    Land,
    MoveTo { x: i32, y: i32 },
    Rotate { degrees: i32 },
    TakePhoto { resolution: String },
    ReturnHome,
    EmergencyLand,
}

impl Command {
    fn validate(&self) -> Result<(), String> {
        match self {
            Command::TakeOff { altitude } => {
                if *altitude < 1 || *altitude > 500 {
                    Err(format!("Недопустима висота: {} (1-500)", altitude))
                } else {
                    Ok(())
                }
            },
            Command::MoveTo { x, y } => {
                if x.abs() > 1000 || y.abs() > 1000 {
                    Err(String::from("Координати за межами зони"))
                } else {
                    Ok(())
                }
            },
            Command::Rotate { degrees } => {
                if *degrees < -180 || *degrees > 180 {
                    Err(format!("Недопустимий кут: {} (-180 до 180)", degrees))
                } else {
                    Ok(())
                }
            },
            _ => Ok(()),
        }
    }
    
    fn describe(&self) -> String {
        match self {
            Command::TakeOff { altitude } => format!("Зліт на {} м", altitude),
            Command::Land => String::from("Посадка"),
            Command::MoveTo { x, y } => format!("Рух до ({}, {})", x, y),
            Command::Rotate { degrees } => format!("Поворот на {}°", degrees),
            Command::TakePhoto { resolution } => format!("Фото {}", resolution),
            Command::ReturnHome => String::from("Повернення на базу"),
            Command::EmergencyLand => String::from("Аварійна посадка"),
        }
    }
    
    fn priority(&self) -> i32 {
        match self {
            Command::EmergencyLand => 0,
            Command::ReturnHome => 1,
            Command::Land => 2,
            _ => 3,
        }
    }
}

fn execute_command(cmd: &Command) -> Result<String, String> {
    cmd.validate()?;
    
    let result = match cmd {
        Command::TakeOff { altitude } => {
            format!("✓ Злетіли на висоту {} м", altitude)
        },
        Command::Land => {
            String::from("✓ Успішна посадка")
        },
        Command::MoveTo { x, y } => {
            format!("✓ Прибули в точку ({}, {})", x, y)
        },
        Command::Rotate { degrees } => {
            let direction = if *degrees > 0 { "праворуч" } else { "ліворуч" };
            format!("✓ Поворот {} на {}°", direction, degrees.abs())
        },
        Command::TakePhoto { resolution } => {
            format!("✓ Фото збережено ({})", resolution)
        },
        Command::ReturnHome => {
            String::from("✓ Повернулись на базу")
        },
        Command::EmergencyLand => {
            String::from("✓ Аварійна посадка виконана")
        },
    };
    
    Ok(result)
}

fn main() {
    println!("=== Enum команд з обробкою ===\n");
    
    let commands = vec![
        Command::TakeOff { altitude: 50 },
        Command::MoveTo { x: 100, y: 200 },
        Command::Rotate { degrees: 90 },
        Command::TakePhoto { resolution: String::from("4K") },
        Command::MoveTo { x: 0, y: 0 },
        Command::Land,
        // Невалідні команди
        Command::TakeOff { altitude: 1000 },
        Command::Rotate { degrees: 360 },
    ];
    
    for cmd in &commands {
        println!("Команда: {}", cmd.describe());
        match execute_command(cmd) {
            Ok(msg) => println!("  {}\n", msg),
            Err(e) => println!("  ✗ Помилка: {}\n", e),
        }
    }
}
```

---

### Завдання 3: Option для пошуку

**Умова:** Створіть функції пошуку, що повертають Option: пошук дрона за ім'ям, пошук найближчого дрона до точки, пошук дрона з найбільшим зарядом.

**Розв'язання:**

```rust
struct Drone {
    name: String,
    x: i32,
    y: i32,
    battery: i32,
}

impl Drone {
    fn new(name: &str, x: i32, y: i32, battery: i32) -> Drone {
        Drone {
            name: String::from(name),
            x,
            y,
            battery,
        }
    }
    
    fn distance_to(&self, x: i32, y: i32) -> f64 {
        (((self.x - x).pow(2) + (self.y - y).pow(2)) as f64).sqrt()
    }
}

fn find_by_name<'a>(drones: &'a [Drone], name: &str) -> Option<&'a Drone> {
    drones.iter().find(|d| d.name == name)
}

fn find_nearest<'a>(drones: &'a [Drone], x: i32, y: i32) -> Option<&'a Drone> {
    drones.iter().min_by(|a, b| {
        a.distance_to(x, y)
            .partial_cmp(&b.distance_to(x, y))
            .unwrap()
    })
}

fn find_best_battery<'a>(drones: &'a [Drone]) -> Option<&'a Drone> {
    drones.iter().max_by_key(|d| d.battery)
}

fn find_available<'a>(drones: &'a [Drone], min_battery: i32) -> Vec<&'a Drone> {
    drones.iter().filter(|d| d.battery >= min_battery).collect()
}

fn main() {
    println!("=== Option для пошуку ===\n");
    
    let fleet = vec![
        Drone::new("Alpha", 10, 20, 85),
        Drone::new("Beta", 50, 60, 45),
        Drone::new("Gamma", 100, 100, 92),
        Drone::new("Delta", 30, 40, 30),
    ];
    
    // Пошук за ім'ям
    println!("--- Пошук за ім'ям ---");
    for name in ["Alpha", "Omega"] {
        match find_by_name(&fleet, name) {
            Some(drone) => println!("✓ '{}' знайдено: ({}, {}), {}%", 
                                   drone.name, drone.x, drone.y, drone.battery),
            None => println!("✗ '{}' не знайдено", name),
        }
    }
    
    // Найближчий до точки
    println!("\n--- Найближчий до (0, 0) ---");
    if let Some(drone) = find_nearest(&fleet, 0, 0) {
        let dist = drone.distance_to(0, 0);
        println!("✓ '{}' на відстані {:.1}", drone.name, dist);
    }
    
    // Найбільший заряд
    println!("\n--- Найбільший заряд ---");
    if let Some(drone) = find_best_battery(&fleet) {
        println!("✓ '{}' має {}%", drone.name, drone.battery);
    }
    
    // Доступні для місії
    println!("\n--- Доступні (батарея >= 50%) ---");
    let available = find_available(&fleet, 50);
    if available.is_empty() {
        println!("✗ Немає доступних дронів");
    } else {
        for drone in available {
            println!("  • {} ({}%)", drone.name, drone.battery);
        }
    }
}
```

---

### Завдання 4: Машина станів дрона

**Умова:** Реалізуйте повну машину станів дрона, де переходи між станами контролюються методами, і неможливі переходи повертають помилку.

**Розв'язання:**

```rust
#[derive(Debug, Clone, PartialEq)]
enum State {
    Off,
    Idle,
    Preflight,
    Flying,
    Hovering,
    Landing,
    Emergency,
}

impl State {
    fn name(&self) -> &str {
        match self {
            State::Off => "Вимкнено",
            State::Idle => "Очікування",
            State::Preflight => "Підготовка",
            State::Flying => "Політ",
            State::Hovering => "Зависання",
            State::Landing => "Посадка",
            State::Emergency => "Аварія",
        }
    }
}

struct DroneFSM {
    name: String,
    state: State,
    battery: i32,
    history: Vec<State>,
}

impl DroneFSM {
    fn new(name: &str) -> DroneFSM {
        DroneFSM {
            name: String::from(name),
            state: State::Off,
            battery: 100,
            history: vec![State::Off],
        }
    }
    
    fn transition(&mut self, new_state: State) {
        self.history.push(new_state.clone());
        self.state = new_state;
    }
    
    fn power_on(&mut self) -> Result<(), String> {
        match self.state {
            State::Off => {
                self.transition(State::Idle);
                Ok(())
            },
            _ => Err(format!("Неможливо увімкнути зі стану '{}'", self.state.name())),
        }
    }
    
    fn power_off(&mut self) -> Result<(), String> {
        match self.state {
            State::Idle => {
                self.transition(State::Off);
                Ok(())
            },
            _ => Err(format!("Можна вимкнути тільки в режимі очікування")),
        }
    }
    
    fn start_preflight(&mut self) -> Result<(), String> {
        match self.state {
            State::Idle => {
                if self.battery < 20 {
                    return Err(String::from("Недостатньо заряду для польоту"));
                }
                self.transition(State::Preflight);
                Ok(())
            },
            _ => Err(format!("Неможливо почати підготовку зі стану '{}'", self.state.name())),
        }
    }
    
    fn take_off(&mut self) -> Result<(), String> {
        match self.state {
            State::Preflight => {
                self.transition(State::Flying);
                self.battery -= 5;
                Ok(())
            },
            _ => Err(format!("Зліт можливий тільки після підготовки")),
        }
    }
    
    fn hover(&mut self) -> Result<(), String> {
        match self.state {
            State::Flying => {
                self.transition(State::Hovering);
                Ok(())
            },
            _ => Err(format!("Зависання можливе тільки під час польоту")),
        }
    }
    
    fn resume_flight(&mut self) -> Result<(), String> {
        match self.state {
            State::Hovering => {
                self.transition(State::Flying);
                Ok(())
            },
            _ => Err(format!("Продовження польоту можливе тільки із зависання")),
        }
    }
    
    fn start_landing(&mut self) -> Result<(), String> {
        match self.state {
            State::Flying | State::Hovering => {
                self.transition(State::Landing);
                Ok(())
            },
            _ => Err(format!("Посадка можлива тільки в повітрі")),
        }
    }
    
    fn complete_landing(&mut self) -> Result<(), String> {
        match self.state {
            State::Landing => {
                self.transition(State::Idle);
                self.battery -= 3;
                Ok(())
            },
            _ => Err(format!("Завершення посадки можливе тільки під час посадки")),
        }
    }
    
    fn emergency(&mut self) {
        self.transition(State::Emergency);
    }
    
    fn reset_from_emergency(&mut self) -> Result<(), String> {
        match self.state {
            State::Emergency => {
                self.transition(State::Idle);
                Ok(())
            },
            _ => Err(String::from("Скидання можливе тільки в аварійному режимі")),
        }
    }
    
    fn status(&self) -> String {
        format!("[{}] Стан: {} | Батарея: {}%", 
                self.name, self.state.name(), self.battery)
    }
    
    fn print_history(&self) {
        println!("Історія переходів:");
        for (i, state) in self.history.iter().enumerate() {
            let arrow = if i < self.history.len() - 1 { " →" } else { " (поточний)" };
            println!("  {}. {}{}", i + 1, state.name(), arrow);
        }
    }
}

fn main() {
    println!("=== Машина станів дрона ===\n");
    
    let mut drone = DroneFSM::new("Phoenix");
    
    println!("{}\n", drone.status());
    
    // Успішний цикл польоту
    let operations: Vec<(&str, Box<dyn Fn(&mut DroneFSM) -> Result<(), String>>)> = vec![
        ("Увімкнення", Box::new(|d| d.power_on())),
        ("Підготовка", Box::new(|d| d.start_preflight())),
        ("Зліт", Box::new(|d| d.take_off())),
        ("Зависання", Box::new(|d| d.hover())),
        ("Продовження", Box::new(|d| d.resume_flight())),
        ("Посадка", Box::new(|d| d.start_landing())),
        ("Завершення", Box::new(|d| d.complete_landing())),
        ("Вимкнення", Box::new(|d| d.power_off())),
    ];
    
    for (name, op) in operations {
        match op(&mut drone) {
            Ok(()) => println!("✓ {}: успіх", name),
            Err(e) => println!("✗ {}: {}", name, e),
        }
        println!("  {}\n", drone.status());
    }
    
    println!("\n{}", "=".repeat(40));
    drone.print_history();
}
```

---

## Підсумок заняття

На цьому занятті ви навчились:

1. **Створювати структури** з іменованими полями
2. **Додавати методи** через impl блоки
3. **Використовувати конструктори** (асоційовані функції)
4. **Визначати enum** з варіантами та даними
5. **Застосовувати pattern matching** для обробки enum
6. **Працювати з Option** для відсутніх значень
7. **Використовувати if let** для спрощеного matching

---

## Перевірте себе

1. Як створити структуру з полем `name: String`?
2. Чим відрізняється `&self` від `&mut self` в методах?
3. Як викликати асоційовану функцію (конструктор)?
4. Що означає `Some(value)` та `None`?
5. Як обробити всі варіанти enum у match?
6. Коли використовувати `if let` замість `match`?

**Відповіді:**
1. `struct MyStruct { name: String }`
2. `&self` — читання, `&mut self` — зміна полів
3. `TypeName::function_name()` (через `::`)
4. `Some(value)` — є значення, `None` — значення відсутнє
5. Перелічити всі варіанти або використати `_ => ...`
6. Коли цікавить тільки один варіант

---

## Завершення курсу

🎉 **Вітаємо! Ви завершили базовий курс Rust!**

Тепер ви володієте фундаментальними знаннями:
- Змінні, типи, функції
- Умовні оператори та цикли
- Кортежі, масиви, слайси
- Ownership та Borrowing
- Структури та Enum

**Наступні кроки:**
- Вивчіть колекції: `Vec`, `HashMap`, `String`
- Опануйте обробку помилок: `Result`, `?` оператор
- Дізнайтесь про traits та generics
- Спробуйте паралелізм та async/await

Продовжуйте практикуватись і створювати проєкти! 🚀
