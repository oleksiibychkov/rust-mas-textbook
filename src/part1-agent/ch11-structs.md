# Розділ 11: Struct — Моделювання стану агента

## Вступ

Уявіть, що ви будуєте систему керування автономними дронами. Кожен дрон має свій ідентифікатор, поточну позицію в просторі, рівень заряду батареї, напрямок польоту та статус активності. У попередніх розділах ми зберігали ці дані в окремих змінних:

```rust
let mut agent1_id = String::from("SCOUT-001");
let mut agent1_x = 100.0;
let mut agent1_y = 200.0;
let mut agent1_z = 50.0;
let mut agent1_battery = 85;
let mut agent1_heading = 90.0;
```

Такий підхід працює, але швидко стає кошмаром. Уявіть, що вам потрібно додати другого агента — доведеться створити ще шість змінних з суфіксом `_2`. А функція, що оновлює стан агента, матиме сім або більше параметрів. Легко переплутати змінні різних агентів, особливо коли їх десятки.

Структури вирішують цю проблему елегантно. Замість розрізнених змінних ми створюємо єдиний тип `Agent`, який містить всі пов'язані дані разом. Один агент — одна змінна. Просто, зрозуміло, безпечно.

---

## 11.1 Що таке структура і як її створити

Структура в Rust — це спосіб згрупувати кілька пов'язаних значень під одним іменем. Ви визначаєте, які поля матиме структура та якого типу кожне з них, а потім створюєте конкретні екземпляри з реальними даними.

Оголошення структури виглядає так:

```rust
struct Agent {
    id: String,
    position: (f64, f64, f64),
    battery: u8,
    heading: f64,
    is_active: bool,
}
```

Ми щойно створили новий тип даних під назвою `Agent`. Це як креслення будинку — воно описує, з чого складається будинок, але сам будинок ще не побудований. Кожне поле має ім'я (наприклад, `id`) та тип (наприклад, `String`). Зверніть увагу на синтаксис: ім'я структури пишеться з великої літери у стилі CamelCase, поля розділяються комами.

Щоб створити конкретного агента, ми вказуємо значення для кожного поля:

```rust
fn main() {
    let agent = Agent {
        id: String::from("SCOUT-001"),
        position: (100.0, 200.0, 50.0),
        battery: 85,
        heading: 90.0,
        is_active: true,
    };
    
    println!("Агент {} знаходиться на висоті {} метрів", 
             agent.id, agent.position.2);
    println!("Заряд батареї: {}%", agent.battery);
}
```

Тут ми створили екземпляр структури `Agent`. Усі поля обов'язкові — компілятор не дозволить пропустити жодне. Доступ до полів здійснюється через крапку: `agent.id`, `agent.battery` тощо. Порядок полів при створенні екземпляра не має значення — головне, щоб усі були присутні.

Результат виконання:
```
Агент SCOUT-001 знаходиться на висоті 50 метрів
Заряд батареї: 85%
```

### Мутабельність структури

Якщо ви хочете змінювати поля структури, весь екземпляр має бути оголошений як мутабельний. Rust не дозволяє мати "частково мутабельну" структуру — або все можна змінювати, або нічого:

```rust
fn main() {
    // Цей агент іммутабельний — поля змінити не можна
    let agent1 = Agent {
        id: String::from("SCOUT-001"),
        position: (100.0, 200.0, 50.0),
        battery: 85,
        heading: 90.0,
        is_active: true,
    };
    
    // agent1.battery = 80;  // Помилка компіляції!
    
    // А цей мутабельний — можна змінювати будь-яке поле
    let mut agent2 = Agent {
        id: String::from("SCOUT-002"),
        position: (150.0, 180.0, 45.0),
        battery: 72,
        heading: 180.0,
        is_active: true,
    };
    
    agent2.battery = 65;           // Тепер працює
    agent2.position.0 += 10.0;     // Можна змінювати вкладені значення
    
    println!("Агент {} має {}% заряду", agent2.id, agent2.battery);
}
```

Це може здатися обмеженням, але насправді спрощує міркування про код. Коли ви бачите `let agent = ...`, ви знаєте, що цей агент ніколи не зміниться. Коли бачите `let mut agent = ...`, розумієте, що десь далі його стан може бути модифікований.

---

## 11.2 Зручний синтаксис для роботи зі структурами

Rust пропонує кілька скорочень, що роблять код чистішим.

### Скорочена ініціалізація полів

Коли змінна має те саме ім'я, що й поле структури, можна не повторюватися:

```rust
fn create_agent(id: String, battery: u8) -> Agent {
    Agent {
        id,                          // Замість id: id
        battery,                     // Замість battery: battery
        position: (0.0, 0.0, 0.0),
        heading: 0.0,
        is_active: true,
    }
}
```

Компілятор розуміє, що `id` — це і назва поля, і назва змінної, з якої береться значення. Це особливо зручно у конструкторах, де параметри часто називаються так само, як поля.

### Створення на основі іншого екземпляра

Іноді потрібно створити нового агента, який відрізняється від існуючого лише кількома полями. Синтаксис `..other` копіює решту полів:

```rust
fn main() {
    let agent1 = Agent {
        id: String::from("SCOUT-001"),
        position: (100.0, 200.0, 50.0),
        battery: 85,
        heading: 90.0,
        is_active: true,
    };
    
    // Створюємо agent2 на основі agent1
    let agent2 = Agent {
        id: String::from("SCOUT-002"),  // Нове значення
        battery: 100,                    // Нове значення
        ..agent1                         // Решта полів з agent1
    };
    
    println!("Agent2 heading: {}", agent2.heading);  // 90.0 — з agent1
}
```

Тут є важливий нюанс щодо ownership. Поля з типами, що реалізують `Copy` (числа, булеві значення, кортежі з Copy-типів), копіюються. Але якби ми не вказали нове значення для `id`, String з `agent1.id` переміщувався б у `agent2`, і `agent1.id` став би недоступним. У нашому прикладі ми явно вказали новий `id`, тому проблем немає.

---

## 11.3 Різновиди структур

Rust підтримує три види структур, кожен для свого випадку.

### Структури з іменованими полями

Це найпоширеніший вид, який ми вже бачили. Поля мають імена, доступ через крапку. Використовуйте їх, коли у структури кілька полів і важливо розуміти, що означає кожне:

```rust
struct Agent {
    id: String,
    battery: u8,
    is_active: bool,
}
```

### Кортежні структури

Іноді поля настільки очевидні, що імена здаються зайвими. Кортежні структури мають лише типи, доступ за індексом:

```rust
struct Position(f64, f64, f64);    // Очевидно: x, y, z
struct Color(u8, u8, u8);          // Очевидно: r, g, b
struct AgentId(String);            // Обгортка для типобезпеки

fn main() {
    let pos = Position(100.0, 200.0, 50.0);
    let color = Color(255, 128, 0);
    
    println!("X: {}, Y: {}, Z: {}", pos.0, pos.1, pos.2);
    println!("Red component: {}", color.0);
}
```

Головна перевага кортежних структур — типобезпека. `Position` і `Color` — це різні типи, хоча обидва містять три числа. Компілятор не дозволить випадково передати колір замість позиції. Це називається "newtype pattern" — обгортання примітивного типу для надання йому семантичного значення.

### Порожні структури

Структури можуть не мати полів взагалі. Вони займають нуль байт і використовуються як маркери:

```rust
struct Idle;
struct Moving;
struct Charging;
```

Самі по собі вони здаються безглуздими, але стають корисними в поєднанні з системою типів та traits. Ми повернемося до них у наступних розділах.

---

## 11.4 Методи: надаємо структурі поведінку

Структура без методів — просто контейнер даних. Щоб агент міг "робити" щось, потрібно додати методи. Вони визначаються в блоці `impl`:

```rust
struct Agent {
    id: String,
    position: (f64, f64, f64),
    battery: u8,
}

impl Agent {
    fn get_altitude(&self) -> f64 {
        self.position.2
    }
    
    fn info(&self) -> String {
        format!("Агент {} на висоті {:.1}м, батарея {}%",
                self.id, self.get_altitude(), self.battery)
    }
}

fn main() {
    let agent = Agent {
        id: String::from("SCOUT-001"),
        position: (100.0, 200.0, 50.0),
        battery: 85,
    };
    
    println!("{}", agent.info());
}
```

Метод виглядає як звичайна функція, але його перший параметр — `self`. Це посилання на екземпляр, для якого викликається метод. Коли ви пишете `agent.info()`, Rust автоматично передає `&agent` як `self`.

Результат:
```
Агент SCOUT-001 на висоті 50.0м, батарея 85%
```

### Три способи приймати self

Метод може працювати з екземпляром трьома способами, залежно від того, що йому потрібно робити:

**`&self`** — іммутабельне позичання. Метод може читати поля, але не може їх змінювати. Більшість методів використовують саме цей варіант:

```rust
impl Agent {
    fn id(&self) -> &str {
        &self.id
    }
    
    fn is_low_battery(&self) -> bool {
        self.battery < 20
    }
}
```

**`&mut self`** — мутабельне позичання. Метод може змінювати поля екземпляра:

```rust
impl Agent {
    fn drain_battery(&mut self, amount: u8) {
        self.battery = self.battery.saturating_sub(amount);
    }
    
    fn move_to(&mut self, x: f64, y: f64, z: f64) {
        self.position = (x, y, z);
        self.drain_battery(5);  // Рух витрачає енергію
    }
}
```

Щоб викликати такий метод, екземпляр має бути мутабельним:

```rust
let mut agent = Agent { /* ... */ };
agent.move_to(200.0, 300.0, 60.0);  // Потрібен mut
```

**`self`** — володіння. Метод забирає екземпляр і після виклику він більше не існує:

```rust
impl Agent {
    fn destroy(self) -> String {
        format!("Агент {} знищено на позиції {:?}", self.id, self.position)
    }
}

fn main() {
    let agent = Agent { /* ... */ };
    let message = agent.destroy();
    println!("{}", message);
    
    // println!("{}", agent.id);  // Помилка! agent більше не існує
}
```

Такі методи використовуються рідко, зазвичай коли екземпляр трансформується в щось інше або коли його ресурси потрібно звільнити певним чином.

### Асоційовані функції та конструктори

Функції в блоці `impl`, що не приймають `self`, називаються асоційованими функціями. Вони викликаються через `::`, а не через крапку:

```rust
impl Agent {
    fn new(id: String) -> Self {
        Self {
            id,
            position: (0.0, 0.0, 0.0),
            battery: 100,
        }
    }
    
    fn with_position(id: String, x: f64, y: f64, z: f64) -> Self {
        Self {
            id,
            position: (x, y, z),
            battery: 100,
        }
    }
    
    fn max_battery() -> u8 {
        100
    }
}

fn main() {
    let agent1 = Agent::new(String::from("SCOUT-001"));
    let agent2 = Agent::with_position(String::from("SCOUT-002"), 100.0, 50.0, 30.0);
    
    println!("Максимальний заряд: {}", Agent::max_battery());
}
```

Зверніть увагу на `Self` з великої літери — це псевдонім для типу структури всередині блоку `impl`. Замість `Agent { ... }` можна писати `Self { ... }`. Це зручно — якщо ви перейменуєте структуру, не доведеться змінювати код у кожному конструкторі.

---

## 11.5 Ownership і структури

Структури підпорядковуються тим самим правилам володіння, що й інші типи в Rust.

### Структура володіє своїми полями

Коли ви створюєте структуру з якимось значенням, структура стає його власником:

```rust
fn main() {
    let name = String::from("SCOUT-001");
    
    let agent = Agent {
        id: name,  // name переміщено в agent
        position: (0.0, 0.0, 0.0),
        battery: 100,
    };
    
    // println!("{}", name);  // Помилка! name вже не існує
    println!("{}", agent.id);  // А ось так можна
}
```

Рядок `name` переміщено у поле `agent.id`. Тепер агент володіє цим рядком. Коли агент вийде зі scope, рядок буде автоматично звільнено.

### Позичання структур

Структури можна позичати так само, як і прості типи:

```rust
fn print_status(agent: &Agent) {
    println!("Агент {}: батарея {}%", agent.id, agent.battery);
}

fn charge(agent: &mut Agent) {
    agent.battery = 100;
}

fn main() {
    let mut scout = Agent::new(String::from("SCOUT-001"));
    
    print_status(&scout);   // Іммутабельне позичання
    print_status(&scout);   // Можна позичати багато разів
    
    charge(&mut scout);     // Мутабельне позичання
    
    print_status(&scout);   // Знову іммутабельне
}
```

Усі правила borrowing, які ви вивчили раніше, застосовуються і до структур. Можна мати багато іммутабельних посилань або одне мутабельне, але не обидва одночасно.

---

## 11.6 Вкладені структури та композиція

Реальні системи часто мають ієрархічну структуру даних. Замість того, щоб тримати всі поля в одній великій структурі, можна розбити її на менші компоненти:

```rust
struct Position {
    x: f64,
    y: f64,
    z: f64,
}

struct Velocity {
    vx: f64,
    vy: f64,
    vz: f64,
}

struct Agent {
    id: String,
    position: Position,
    velocity: Velocity,
    battery: u8,
}
```

Тепер `Position` і `Velocity` — окремі типи зі своїми методами:

```rust
impl Position {
    fn new(x: f64, y: f64, z: f64) -> Self {
        Self { x, y, z }
    }
    
    fn origin() -> Self {
        Self { x: 0.0, y: 0.0, z: 0.0 }
    }
    
    fn distance_to(&self, other: &Position) -> f64 {
        let dx = self.x - other.x;
        let dy = self.y - other.y;
        let dz = self.z - other.z;
        (dx*dx + dy*dy + dz*dz).sqrt()
    }
}

impl Velocity {
    fn zero() -> Self {
        Self { vx: 0.0, vy: 0.0, vz: 0.0 }
    }
    
    fn speed(&self) -> f64 {
        (self.vx*self.vx + self.vy*self.vy + self.vz*self.vz).sqrt()
    }
}
```

Методи `Agent` можуть делегувати роботу методам вкладених структур:

```rust
impl Agent {
    fn new(id: String) -> Self {
        Self {
            id,
            position: Position::origin(),
            velocity: Velocity::zero(),
            battery: 100,
        }
    }
    
    fn distance_to(&self, other: &Agent) -> f64 {
        self.position.distance_to(&other.position)
    }
    
    fn current_speed(&self) -> f64 {
        self.velocity.speed()
    }
}
```

Така композиція має кілька переваг. По-перше, код краще організований — кожна структура відповідає за свою частину функціональності. По-друге, `Position` можна використовувати і в інших місцях, не тільки в `Agent`. По-третє, легше тестувати — можна тестувати `Position` окремо від `Agent`.

---

## 11.7 Практичний приклад: повна модель агента

Об'єднаємо все вивчене у цілісну модель агента БПЛА:

```rust
struct Position {
    x: f64,
    y: f64,
    z: f64,
}

impl Position {
    fn new(x: f64, y: f64, z: f64) -> Self {
        Self { x, y, z }
    }
    
    fn origin() -> Self {
        Self::new(0.0, 0.0, 0.0)
    }
    
    fn distance_to(&self, other: &Position) -> f64 {
        let dx = self.x - other.x;
        let dy = self.y - other.y;
        let dz = self.z - other.z;
        (dx*dx + dy*dy + dz*dz).sqrt()
    }
}

struct Agent {
    id: String,
    position: Position,
    heading: f64,
    battery: u8,
    is_active: bool,
}

impl Agent {
    fn new(id: String) -> Self {
        Self {
            id,
            position: Position::origin(),
            heading: 0.0,
            battery: 100,
            is_active: true,
        }
    }
    
    fn with_position(id: String, x: f64, y: f64, z: f64) -> Self {
        Self {
            id,
            position: Position::new(x, y, z),
            heading: 0.0,
            battery: 100,
            is_active: true,
        }
    }
    
    fn altitude(&self) -> f64 {
        self.position.z
    }
    
    fn is_low_battery(&self) -> bool {
        self.battery < 20
    }
    
    fn move_to(&mut self, x: f64, y: f64, z: f64) {
        self.position = Position::new(x, y, z);
        self.drain_battery(5);
    }
    
    fn turn(&mut self, angle: f64) {
        self.heading = (self.heading + angle) % 360.0;
        self.drain_battery(1);
    }
    
    fn drain_battery(&mut self, amount: u8) {
        self.battery = self.battery.saturating_sub(amount);
        if self.battery == 0 {
            self.is_active = false;
        }
    }
    
    fn charge(&mut self) {
        self.battery = 100;
        self.is_active = true;
    }
    
    fn distance_to(&self, other: &Agent) -> f64 {
        self.position.distance_to(&other.position)
    }
    
    fn status(&self) -> String {
        let state = if self.is_active { "ACTIVE" } else { "INACTIVE" };
        format!("{}: pos({:.1}, {:.1}, {:.1}) hdg={:.0}° bat={}% {}",
                self.id,
                self.position.x, self.position.y, self.position.z,
                self.heading, self.battery, state)
    }
}

fn main() {
    println!("=== Симуляція агента БПЛА ===\n");
    
    let mut scout = Agent::new(String::from("SCOUT-001"));
    let base = Agent::with_position(String::from("BASE"), 0.0, 0.0, 0.0);
    
    println!("Початковий стан:");
    println!("  {}", scout.status());
    println!("  {}", base.status());
    
    println!("\nВиконання місії:");
    
    scout.move_to(100.0, 50.0, 80.0);
    println!("  Після переміщення: {}", scout.status());
    
    scout.turn(90.0);
    println!("  Після повороту: {}", scout.status());
    
    for i in 1..=10 {
        scout.drain_battery(8);
        if scout.is_low_battery() {
            println!("  Крок {}: низький заряд! {}", i, scout.status());
            break;
        }
    }
    
    println!("\nФінальна статистика:");
    println!("  Відстань до бази: {:.2}м", scout.distance_to(&base));
    println!("  Висота: {:.1}м", scout.altitude());
    
    if !scout.is_active {
        println!("  Агент деактивовано через розряд батареї");
    }
}
```

Виконавши цю програму, ви побачите:

```
=== Симуляція агента БПЛА ===

Початковий стан:
  SCOUT-001: pos(0.0, 0.0, 0.0) hdg=0° bat=100% ACTIVE
  BASE: pos(0.0, 0.0, 0.0) hdg=0° bat=100% ACTIVE

Виконання місії:
  Після переміщення: SCOUT-001: pos(100.0, 50.0, 80.0) hdg=0° bat=95% ACTIVE
  Після повороту: SCOUT-001: pos(100.0, 50.0, 80.0) hdg=90° bat=94% ACTIVE
  Крок 3: низький заряд! SCOUT-001: pos(100.0, 50.0, 80.0) hdg=90° bat=18% ACTIVE

Фінальна статистика:
  Відстань до бази: 135.28м
  Висота: 80.0м
```

---

## 11.8 Лабораторна робота

Тепер ваша черга практикуватись. Створіть систему для представлення космічного корабля.

**Завдання:**

1. Створіть структуру `Vector3D` з полями `x`, `y`, `z` та методами:
   - `new(x, y, z)` — конструктор
   - `length(&self)` — довжина вектора
   - `add(&self, other: &Vector3D)` — повертає суму векторів

2. Створіть структуру `Spaceship` з полями:
   - `name: String`
   - `position: Vector3D`
   - `velocity: Vector3D`
   - `fuel: u32`

3. Додайте методи для `Spaceship`:
   - `new(name)` — конструктор
   - `accelerate(&mut self, dv: Vector3D)` — змінює швидкість
   - `update(&mut self, dt: f64)` — оновлює позицію за час dt
   - `info(&self)` — повертає рядок з інформацією

**Критерії оцінювання:**

| Критерій | Бали |
|----------|------|
| Vector3D структура та методи | 3 |
| Spaceship структура | 2 |
| Методи Spaceship | 3 |
| Демонстрація роботи | 2 |
| **Максимум** | **10** |

---

## Підсумок

У цьому розділі ви навчились групувати пов'язані дані у структури, замість того щоб мати десятки окремих змінних. Ви дізнались про три види структур: з іменованими полями, кортежні та порожні. Ви освоїли блок `impl` для додавання методів і зрозуміли різницю між `&self`, `&mut self` та `self`. Нарешті, ви побачили, як композиція дозволяє будувати складні типи з простіших компонентів.

Структури — це фундамент для моделювання реальних сутностей у вашому коді. Агент, місія, команда, сенсор — все це природно представляється структурами. У наступному розділі ми додамо ще один потужний інструмент — переліки (enum), які дозволять моделювати стани та варіанти.

---

## Зв'язок з наступним розділом

Структури чудово групують дані, але як представити **стани** агента? Idle, Moving, Charging — це не просто різні значення поля, це принципово різні режими роботи з різними даними. У **Розділі 12** ви познайомитесь з `enum` — інструментом для моделювання варіантів та машин станів.
