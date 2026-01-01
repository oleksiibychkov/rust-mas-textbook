# Розділ 29: Rc — спільне володіння

## Вступ: коли одного власника замало

Ownership в Rust суворий: одне значення — **один власник**. Коли власник виходить зі scope — значення звільняється. Просто, передбачувано, безпечно.

Але реальні системи часто потребують **спільного доступу**. Уявіть:

- **Карта світу**, яку читають усі агенти рою
- **Конфігурація місії**, на яку посилаються різні модулі
- **База знань**, спільна для розвідників та бойових дронів

Хто "володіє" цими даними? Якщо Scout володіє картою — що робити Combat-агенту? Копіювати гігабайти даних? Це неефективно. Передати ownership? Тоді Scout втратить доступ.

**Rc<T>** (Reference Counted — з підрахунком посилань) вирішує цю проблему елегантно:

```text
Без Rc — хто володіє картою?
┌─────────────┐     ┌─────────────┐
│    Scout    │     │   Combat    │
│  map: ???   │     │  map: ???   │
└─────────────┘     └─────────────┘
    Копія?             Ще копія?
    
З Rc — один екземпляр, багато посилань:
┌─────────────┐     ┌─────────────┐
│    Scout    │     │   Combat    │
│  map: Rc ───┼─────┼──► WorldMap │
└─────────────┘     └─────────────┘
                         count: 2
```

Замість одного власника — **лічильник посилань**. Кожен `Rc::clone()` збільшує лічильник, кожен `drop` зменшує. Коли лічильник досягає нуля — дані звільняються. Без збирача сміття, без витоків.

**Важливе обмеження:** Rc працює тільки в **однопотоковому** коді. Для багатопотоковості є `Arc` (Розділ 34).

---

## 29.1 Як працює Reference Counting

### Концепція підрахунку посилань

Reference counting — одна з найстаріших технік керування пам'яттю, що використовується ще з 1960-х років. Ідея проста:

1. Кожне значення має **лічильник** — скільки "власників" на нього посилаються
2. Створення нового посилання — лічильник **+1**
3. Знищення посилання — лічильник **-1**
4. Лічильник досягає **0** — значення **звільняється**

```rust
Життєвий цикл Rc:

Rc::new(data)       → count = 1    "Народження"
Rc::clone(&rc)      → count = 2    "Хтось ще посилається"
Rc::clone(&rc)      → count = 3    "І ще хтось"
drop(rc1)           → count = 2    "Один пішов"
drop(rc2)           → count = 1    "Ще один"
drop(rc3)           → count = 0    "Останній — звільняємо!"
                         ↓
                      FREE
```

### Що зберігається на Heap

Коли ви створюєте `Rc::new(value)`, на heap виділяється блок, що містить:

```text
┌─────────────────────────────────────────┐
│            HEAP BLOCK                    │
├─────────────────────────────────────────┤
│  strong_count: usize  ← Сильні посилання │
│  weak_count: usize    ← Слабкі посилання │
│  value: T             ← Ваші дані        │
└─────────────────────────────────────────┘
```

Кожен `Rc<T>` на стеку — це просто вказівник на цей блок (8 байт на 64-bit системі). Коли ви робите `Rc::clone()`, дані **не копіюються** — лише збільшується `strong_count`.

### Візуалізація спільного володіння

```text
       ┌─────────────────────────────────────┐
       │           HEAP                       │
       │  ┌─────────────────────────────┐    │
       │  │  WorldMap { ... }           │    │
       │  │  strong_count: 3            │    │
       │  │  weak_count: 0              │    │
       │  └─────────────────────────────┘    │
       └──────────▲────▲────▲────────────────┘
                  │    │    │
       ┌──────────┼────┼────┼──────────┐
       │  STACK   │    │    │          │
       │  scout ──┘    │    │  8 байт  │
       │  combat ──────┘    │  8 байт  │
       │  recon ────────────┘  8 байт  │
       └───────────────────────────────┘
       
Три вказівники → одні дані
```

### Порівняння Box та Rc

| Характеристика | `Box<T>` | `Rc<T>` |
|----------------|----------|---------|
| **Кількість власників** | Рівно 1 | Багато |
| **Clone** | Копіює дані (дорого) | Збільшує лічильник (дешево) |
| **Мутабельність** | Так (`&mut`) | Ні (тільки читання) |
| **Overhead** | Мінімальний | Лічильник (16 байт) |
| **Потокобезпечність** | Ні | Ні |
| **Use case** | Один власник на heap | Спільне володіння |

---

## 29.2 Базове використання Rc

### Створення та клонування

**Постановка задачі:** Продемонструвати створення Rc та спостереження за лічильником посилань.

```rust
use std::rc::Rc;

fn main() {
    // Створюємо Rc — лічильник = 1
    let original = Rc::new(String::from("shared data"));
    println!("Після створення: count = {}", Rc::strong_count(&original));
    
    // Клонуємо — лічильник = 2
    // УВАГА: дані НЕ копіюються!
    let clone1 = Rc::clone(&original);
    println!("Після clone1: count = {}", Rc::strong_count(&original));
    
    // Ще один клон — лічильник = 3
    let clone2 = Rc::clone(&original);
    println!("Після clone2: count = {}", Rc::strong_count(&original));
    
    // Всі три змінні посилаються на ТІ САМІ дані
    println!("\noriginal: '{}'", original);
    println!("clone1:   '{}'", clone1);
    println!("clone2:   '{}'", clone2);
    
    // Перевіряємо, що це справді ті самі дані
    // (порівнюємо адреси вказівників)
    println!("\nЦе одні й ті самі дані: {}", 
        Rc::ptr_eq(&original, &clone1));
    
    // Drop зменшує лічильник
    drop(clone2);
    println!("\nПісля drop(clone2): count = {}", Rc::strong_count(&original));
    
    drop(clone1);
    println!("Після drop(clone1): count = {}", Rc::strong_count(&original));
    
    // Коли original вийде зі scope, count стане 0 → дані звільняться
}
```

**Вивід:**
```rust
Після створення: count = 1
Після clone1: count = 2
Після clone2: count = 3

original: 'shared data'
clone1:   'shared data'
clone2:   'shared data'

Це одні й ті самі дані: true

Після drop(clone2): count = 2
Після drop(clone1): count = 1
```

**Як це працює:**

1. `Rc::new()` виділяє блок на heap з даними та лічильником = 1
2. `Rc::clone(&original)` — **не копіює дані**, лише збільшує лічильник
3. `Rc::strong_count()` показує поточне значення лічильника
4. `Rc::ptr_eq()` перевіряє, чи два Rc вказують на той самий блок
5. `drop()` зменшує лічильник; коли він досягає 0 — дані звільняються

### Чому Rc::clone(), а не .clone()?

У Rust є конвенція: використовувати `Rc::clone(&rc)` замість `rc.clone()`.

**Постановка задачі:** Пояснити різницю між двома способами клонування.

```rust
use std::rc::Rc;

fn main() {
    let data = Rc::new(vec![1, 2, 3, 4, 5]);
    
    // Спосіб 1: Rc::clone(&data)
    // ✅ Рекомендовано — явно показує, що це ДЕШЕВА операція
    let ref1 = Rc::clone(&data);
    
    // Спосіб 2: data.clone()
    // ⚠️ Працює, але ВИГЛЯДАЄ як глибоке копіювання
    let ref2 = data.clone();
    
    // Обидва способи ІДЕНТИЧНІ — збільшують лічильник
    // Різниця лише в читабельності коду
    println!("Count: {}", Rc::strong_count(&data));  // 3
}
```

**Чому це важливо?**

Коли ви читаєте код і бачите `big_vector.clone()`, ви можете подумати: "О, тут копіюється великий вектор — це дорого!" Але якщо big_vector — це `Rc<Vec<...>>`, то `.clone()` насправді дешевий (лише +1 до лічильника).

`Rc::clone(&big_vector)` явно каже: "Це Rc-клон — дешева операція".

### Автоматичне розіменування (Deref)

Rc реалізує trait `Deref`, тому ви можете використовувати його як звичайне значення — методи викликаються напряму.

**Постановка задачі:** Показати, що Rc "прозорий" для використання.

```rust
use std::rc::Rc;

struct Agent {
    id: String,
    battery: u8,
}

impl Agent {
    fn status(&self) -> String {
        format!("{}: батарея {}%", self.id, self.battery)
    }
    
    fn is_low_battery(&self) -> bool {
        self.battery < 20
    }
}

fn main() {
    let agent = Rc::new(Agent {
        id: "SCOUT-001".to_string(),
        battery: 85,
    });
    
    // Deref дозволяє звертатись до полів напряму
    println!("ID: {}", agent.id);  // Не потрібно (*agent).id
    
    // Методи теж працюють
    println!("Status: {}", agent.status());
    println!("Low battery: {}", agent.is_low_battery());
    
    // Можна передавати у функції, що приймають &Agent
    fn print_agent(a: &Agent) {
        println!("Agent: {}", a.id);
    }
    print_agent(&agent);  // &Rc<Agent> автоматично перетворюється на &Agent
}
```

---

## 29.3 Rc та іммутабельність

### Rc дає тільки спільні посилання

Це **найважливіше** обмеження Rc: він надає тільки `&T`, ніколи `&mut T`.

**Постановка задачі:** Показати, що Rc не дозволяє мутацію.

```rust
use std::rc::Rc;

fn main() {
    let data = Rc::new(5);
    
    // ✅ Читання — працює
    println!("Value: {}", *data);
    
    // ❌ Запис — помилка компіляції!
    // *data = 10;
    // error: cannot assign to data in an `Rc`
    
    // ❌ Отримати &mut теж не можна
    // let r: &mut i32 = &mut *data;
    // error: cannot borrow as mutable
}
```

**Чому так?** Пригадайте правило Rust: **або** багато `&T`, **або** один `&mut T` — ніколи обидва. Rc за своєю природою дає багато посилань. Якби вони були мутабельними — виник би хаос: хто що змінює? В якому порядку? Data race навіть в однопотоковому коді!

### Рішення: комбінація Rc + RefCell

Коли потрібна і спільне володіння, і мутабельність — використовують `Rc<RefCell<T>>`. Це тема Розділу 30, але ось короткий огляд:

**Постановка задачі:** Показати, як отримати мутабельність через Rc.

```rust
use std::rc::Rc;
use std::cell::RefCell;

fn main() {
    // Rc для спільного володіння + RefCell для мутабельності
    let data = Rc::new(RefCell::new(5));
    
    // Створюємо кілька "власників"
    let ref1 = Rc::clone(&data);
    let ref2 = Rc::clone(&data);
    
    // Мутація через borrow_mut()
    *ref1.borrow_mut() += 10;
    
    // Читання через borrow()
    println!("Value: {}", *ref2.borrow());  // 15
    
    // Ще мутація через інше посилання
    *ref2.borrow_mut() *= 2;
    
    println!("Final: {}", *data.borrow());  // 30
}
```

RefCell переносить перевірки borrowing з compile-time на runtime. Детальніше — у Розділі 30.

---

## 29.4 Weak — уникнення циклічних посилань

### Проблема циклів

Reference counting має ахіллесову п'яту: **циклічні посилання**. Якщо A посилається на B, а B посилається на A — жоден не буде звільнений.

**Постановка задачі:** Показати, як виникає витік пам'яті через цикл.

```rust
use std::rc::Rc;
use std::cell::RefCell;

// Вузол, що може посилатись на інший вузол
struct Node {
    value: i32,
    next: Option<Rc<RefCell<Node>>>,
}

fn create_memory_leak() {
    // Створюємо два вузли
    let a = Rc::new(RefCell::new(Node { value: 1, next: None }));
    let b = Rc::new(RefCell::new(Node { value: 2, next: None }));
    
    println!("Початково:");
    println!("  a.count = {}", Rc::strong_count(&a));  // 1
    println!("  b.count = {}", Rc::strong_count(&b));  // 1
    
    // a → b
    a.borrow_mut().next = Some(Rc::clone(&b));
    println!("\nПісля a → b:");
    println!("  a.count = {}", Rc::strong_count(&a));  // 1
    println!("  b.count = {}", Rc::strong_count(&b));  // 2
    
    // b → a — СТВОРЮЄМО ЦИКЛ!
    b.borrow_mut().next = Some(Rc::clone(&a));
    println!("\nПісля b → a (ЦИКЛ!):");
    println!("  a.count = {}", Rc::strong_count(&a));  // 2
    println!("  b.count = {}", Rc::strong_count(&b));  // 2
    
    // Коли функція завершується:
    // 1. Локальні змінні a, b виходять зі scope
    // 2. a.count зменшується: 2 → 1 (бо b.next ще посилається)
    // 3. b.count зменшується: 2 → 1 (бо a.next ще посилається)
    // 4. Обидва лічильники = 1, не 0
    // 5. НІХТО НЕ ЗВІЛЬНЯЄТЬСЯ → ВИТІК ПАМ'ЯТІ!
    
    println!("\nКінець функції — очікуємо витік пам'яті!");
}

fn main() {
    create_memory_leak();
    println!("Функція завершилась, але пам'ять не звільнена.");
}
```

**Вивід:**
```rust
Початково:
  a.count = 1
  b.count = 1

Після a → b:
  a.count = 1
  b.count = 2

Після b → a (ЦИКЛ!):
  a.count = 2
  b.count = 2

Кінець функції — очікуємо витік пам'яті!
Функція завершилась, але пам'ять не звільнена.
```

### Weak — слабкі посилання

`Weak<T>` — "слабке" посилання, що **не впливає** на звільнення даних:

- `Rc::clone()` збільшує `strong_count`
- `Rc::downgrade()` збільшує `weak_count`
- Дані звільняються коли `strong_count = 0` (незалежно від `weak_count`)

**Постановка задачі:** Показати, як Weak запобігає циклам.

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

// Вузол з WEAK посиланням на попередній
struct Node {
    value: i32,
    next: Option<Rc<RefCell<Node>>>,
    prev: Option<Weak<RefCell<Node>>>,  // Weak для зворотного посилання!
}

fn no_memory_leak() {
    let a = Rc::new(RefCell::new(Node { 
        value: 1, 
        next: None, 
        prev: None 
    }));
    
    let b = Rc::new(RefCell::new(Node { 
        value: 2, 
        next: None, 
        prev: None 
    }));
    
    // a → b (сильне посилання)
    a.borrow_mut().next = Some(Rc::clone(&b));
    
    // b → a (СЛАБКЕ посилання) — НЕ створює цикл!
    b.borrow_mut().prev = Some(Rc::downgrade(&a));
    
    println!("a: strong = {}, weak = {}", 
        Rc::strong_count(&a), Rc::weak_count(&a));
    println!("b: strong = {}, weak = {}", 
        Rc::strong_count(&b), Rc::weak_count(&b));
    
    // Коли функція завершується:
    // 1. a.strong_count: 1 → 0 (Weak не рахується!)
    // 2. a звільняється
    // 3. b.strong_count: 2 → 1 → 0
    // 4. b звільняється
    // ВИТОКУ НЕМАЄ!
}

fn main() {
    no_memory_leak();
    println!("Все успішно звільнено!");
}
```

**Вивід:**
```rust
a: strong = 1, weak = 1
b: strong = 2, weak = 0
Все успішно звільнено!
```

### Використання Weak

Щоб отримати доступ до даних через Weak, потрібно "підняти" його до Rc через `upgrade()`. Це повертає `Option<Rc<T>>` — `None` якщо дані вже звільнені.

**Постановка задачі:** Показати роботу з Weak.

```rust
use std::rc::{Rc, Weak};

fn main() {
    // Створюємо сильне посилання
    let strong: Rc<String> = Rc::new(String::from("important data"));
    
    // Створюємо слабке посилання
    let weak: Weak<String> = Rc::downgrade(&strong);
    
    println!("strong_count = {}", Rc::strong_count(&strong));  // 1
    println!("weak_count = {}", Rc::weak_count(&strong));      // 1
    
    // Upgrade Weak → Option<Rc>
    println!("\nСпроба upgrade поки дані існують:");
    if let Some(rc) = weak.upgrade() {
        println!("  Успіх! Дані: '{}'", rc);
        println!("  strong_count = {}", Rc::strong_count(&strong));  // 2 (тимчасово)
    }
    // rc вийшов зі scope, strong_count знову 1
    
    // Звільняємо сильне посилання
    println!("\nЗвільняємо strong...");
    drop(strong);
    
    // Тепер upgrade повертає None
    println!("Спроба upgrade після звільнення:");
    match weak.upgrade() {
        Some(rc) => println!("  Дані: '{}'", rc),
        None => println!("  Дані вже звільнені!"),
    }
}
```

**Вивід:**
```rust
strong_count = 1
weak_count = 1

Спроба upgrade поки дані існують:
  Успіх! Дані: 'important data'
  strong_count = 2

Звільняємо strong...
Спроба upgrade після звільнення:
  Дані вже звільнені!
```

### Коли використовувати Weak

| Сценарій | Використовувати |
|----------|-----------------|
| Основне посилання (від батька до дитини) | `Rc` |
| Зворотне посилання (від дитини до батька) | `Weak` |
| Кеш, що може бути очищений | `Weak` |
| Спостерігачі (observers) | `Weak` |
| Циклічні структури (графи) | `Weak` для "зворотних" ребер |

---

## 29.5 Практичний приклад: Спільні ресурси агентів

### Спільна конфігурація місії

**Постановка задачі:** Кілька агентів повинні мати доступ до однієї конфігурації місії. Конфігурація велика — копіювати неефективно.

```rust
use std::rc::Rc;

/// Конфігурація місії — спільна для всіх агентів
struct MissionConfig {
    name: String,
    max_altitude: f64,
    max_speed: f64,
    battery_threshold: u8,
    waypoints: Vec<(f64, f64)>,
}

impl MissionConfig {
    fn new(name: &str) -> Self {
        Self {
            name: name.to_string(),
            max_altitude: 100.0,
            max_speed: 20.0,
            battery_threshold: 20,
            waypoints: vec![(0.0, 0.0), (100.0, 0.0), (100.0, 100.0), (0.0, 0.0)],
        }
    }
}

/// Агент, що посилається на спільну конфігурацію
struct Agent {
    id: String,
    config: Rc<MissionConfig>,  // Спільна конфігурація
    battery: u8,
}

impl Agent {
    fn new(id: &str, config: Rc<MissionConfig>) -> Self {
        Self {
            id: id.to_string(),
            config,
            battery: 100,
        }
    }
    
    fn should_return(&self) -> bool {
        self.battery < self.config.battery_threshold
    }
    
    fn max_altitude(&self) -> f64 {
        self.config.max_altitude
    }
    
    fn mission_name(&self) -> &str {
        &self.config.name
    }
}

fn main() {
    // Одна конфігурація для всіх
    let config = Rc::new(MissionConfig::new("Операція Схід"));
    
    println!("Конфігурація створена. Waypoints: {}", 
        config.waypoints.len());
    println!("Rc count: {}\n", Rc::strong_count(&config));
    
    // Створюємо трьох агентів — кожен отримує Rc на ту саму конфігурацію
    let scout = Agent::new("SCOUT-001", Rc::clone(&config));
    let combat = Agent::new("COMBAT-001", Rc::clone(&config));
    let recon = Agent::new("RECON-001", Rc::clone(&config));
    
    println!("Агенти створені. Rc count: {}\n", Rc::strong_count(&config));
    
    // Всі читають ту саму конфігурацію
    println!("Scout місія: '{}', max alt: {}m", 
        scout.mission_name(), scout.max_altitude());
    println!("Combat місія: '{}', max alt: {}m", 
        combat.mission_name(), combat.max_altitude());
    println!("Recon місія: '{}', max alt: {}m", 
        recon.mission_name(), recon.max_altitude());
    
    // Перевіряємо, що це справді ті самі дані
    println!("\nВсі посилаються на одні дані: {}", 
        Rc::ptr_eq(&scout.config, &combat.config) && 
        Rc::ptr_eq(&combat.config, &recon.config));
}
```

**Вивід:**
```text
Конфігурація створена. Waypoints: 4
Rc count: 1

Агенти створені. Rc count: 4

Scout місія: 'Операція Схід', max alt: 100m
Combat місія: 'Операція Схід', max alt: 100m
Recon місія: 'Операція Схід', max alt: 100m

Всі посилаються на одні дані: true
```

### Спільна карта світу з оновленням

**Постановка задачі:** Агенти досліджують територію та оновлюють спільну карту. Кожен бачить, що знайшли інші.

```rust
use std::rc::Rc;
use std::cell::RefCell;
use std::collections::HashMap;

type Position = (i32, i32);

#[derive(Clone, Debug)]
enum CellType {
    Unknown,
    Empty,
    Obstacle,
    Target,
    Base,
}

/// Спільна карта світу
struct WorldMap {
    cells: HashMap<Position, CellType>,
    width: i32,
    height: i32,
}

impl WorldMap {
    fn new(width: i32, height: i32) -> Self {
        Self {
            cells: HashMap::new(),
            width,
            height,
        }
    }
    
    fn set(&mut self, pos: Position, cell: CellType) {
        self.cells.insert(pos, cell);
    }
    
    fn get(&self, pos: Position) -> CellType {
        self.cells.get(&pos).cloned().unwrap_or(CellType::Unknown)
    }
    
    fn known_cells(&self) -> usize {
        self.cells.len()
    }
}

// Тип для спільної карти: Rc для спільного володіння, RefCell для мутабельності
type SharedMap = Rc<RefCell<WorldMap>>;

/// Агент-розвідник
struct Scout {
    id: String,
    position: Position,
    map: SharedMap,
}

impl Scout {
    fn new(id: &str, start: Position, map: SharedMap) -> Self {
        Self {
            id: id.to_string(),
            position: start,
            map,
        }
    }
    
    /// Досліджує клітинку та оновлює карту
    fn explore(&mut self, pos: Position, found: CellType) {
        self.position = pos;
        self.map.borrow_mut().set(pos, found.clone());
        println!("[{}] Знайдено {:?} на {:?}", self.id, found, pos);
    }
    
    /// Перевіряє, чи клітинка вже досліджена (іншим агентом)
    fn is_explored(&self, pos: Position) -> bool {
        !matches!(self.map.borrow().get(pos), CellType::Unknown)
    }
}

fn main() {
    // Спільна карта
    let map: SharedMap = Rc::new(RefCell::new(WorldMap::new(100, 100)));
    
    // Позначаємо базу
    map.borrow_mut().set((0, 0), CellType::Base);
    
    // Три розвідники
    let mut scout1 = Scout::new("SCOUT-1", (0, 0), Rc::clone(&map));
    let mut scout2 = Scout::new("SCOUT-2", (0, 0), Rc::clone(&map));
    let mut scout3 = Scout::new("SCOUT-3", (0, 0), Rc::clone(&map));
    
    println!("=== Початок розвідки ===\n");
    
    // Scout1 досліджує на північ
    scout1.explore((0, 1), CellType::Empty);
    scout1.explore((0, 2), CellType::Empty);
    scout1.explore((0, 3), CellType::Obstacle);
    
    println!();
    
    // Scout2 досліджує на схід
    scout2.explore((1, 0), CellType::Empty);
    scout2.explore((2, 0), CellType::Target);
    
    println!();
    
    // Scout3 перевіряє, чи (0, 2) вже досліджена
    println!("[SCOUT-3] Перевірка (0, 2): {}", 
        if scout3.is_explored((0, 2)) { 
            "вже досліджено (SCOUT-1)" 
        } else { 
            "невідомо" 
        });
    
    // Scout3 досліджує нові клітинки
    scout3.explore((1, 1), CellType::Empty);
    
    println!("\n=== Підсумок ===");
    println!("Всього досліджено клітинок: {}", map.borrow().known_cells());
    println!("Rc count: {}", Rc::strong_count(&map));  // 4 (map + 3 scouts)
}
```

**Вивід:**
```rust
=== Початок розвідки ===

[SCOUT-1] Знайдено Empty на (0, 1)
[SCOUT-1] Знайдено Empty на (0, 2)
[SCOUT-1] Знайдено Obstacle на (0, 3)

[SCOUT-2] Знайдено Empty на (1, 0)
[SCOUT-2] Знайдено Target на (2, 0)

[SCOUT-3] Перевірка (0, 2): вже досліджено (SCOUT-1)
[SCOUT-3] Знайдено Empty на (1, 1)

=== Підсумок ===
Всього досліджено клітинок: 7
Rc count: 4
```

---

## 29.6 Лабораторна робота

**Мета:** Використати Rc для спільних ресурсів між агентами.

### Частина 1: Спільна база знань (4 бали)

Реалізуйте систему, де кілька агентів мають доступ до спільної бази знань:

```rust
struct KnowledgeBase {
    facts: HashMap<String, String>,
}

struct Agent {
    id: String,
    knowledge: Rc<RefCell<KnowledgeBase>>,
}

impl Agent {
    fn learn(&self, key: &str, value: &str);
    fn recall(&self, key: &str) -> Option<String>;
}
```

### Частина 2: Дерево команд з Weak (3 бали)

Реалізуйте дерево команд, де діти посилаються на батька через Weak:

```rust
struct CommandNode {
    name: String,
    parent: Option<Weak<RefCell<CommandNode>>>,
    children: Vec<Rc<RefCell<CommandNode>>>,
}
```

### Частина 3: Моніторинг посилань (3 бали)

Створіть програму, що візуалізує життєвий цикл Rc — створення, клонування, drop.

---

## Підсумок

`Rc<T>` — smart pointer для **спільного володіння** в однопотоковому коді:

| Операція | Ефект |
|----------|-------|
| `Rc::new(value)` | Створює Rc, count = 1 |
| `Rc::clone(&rc)` | count += 1 (дані не копіюються!) |
| `drop(rc)` | count -= 1, якщо 0 → звільнення |
| `Rc::downgrade(&rc)` | Створює Weak |
| `weak.upgrade()` | `Option<Rc>` — None якщо дані звільнені |

**Ключові правила:**

- ✅ Rc для спільного **читання** даних
- ✅ `Rc::clone()` замість `.clone()` для ясності
- ✅ Weak для зворотних посилань (уникнення циклів)
- ⚠️ Rc **не дає мутабельності** — для цього `Rc<RefCell<T>>`
- ⚠️ Rc **не потокобезпечний** — для цього Arc (Розділ 34)

---

## Зв'язок з наступним розділом

Rc дає спільний доступ, але тільки для **читання**. Що робити, коли агенти мають **змінювати** спільні дані? Як отримати `&mut` через іммутабельне посилання?

У **Розділі 30: RefCell — внутрішня мутабельність** ви дізнаєтесь:
- Що таке **interior mutability**
- Як RefCell переносить перевірки на **runtime**
- Як комбінувати `Rc<RefCell<T>>` для спільної мутабельності
- Чим відрізняється `borrow()` від `borrow_mut()`
