# Розділ 30: RefCell — внутрішня мутабельність

## Вступ: коли компілятор занадто суворий

Правила позичання Rust прості та суворі:
- **Або** багато незмінних посилань `&T`
- **Або** одне змінне посилання `&mut T`
- **Ніколи** обидва одночасно

Компілятор перевіряє ці правила на етапі компіляції. Якщо код їх порушує — він просто не скомпілюється. Це чудово для безпеки, але іноді занадто жорстко для реальних сценаріїв.

**Розглянемо типову проблему:**

```rust
struct Agent {
    id: String,
    operation_count: u32,  // Хочемо рахувати операції
}

impl Agent {
    fn perform_action(&self) {  // Метод приймає &self
        // Тут ми хочемо збільшити лічильник...
        // self.operation_count += 1;  // ❌ Помилка!
        // ...але &self не дозволяє модифікувати поля
    }
}
```

Метод `perform_action` приймає `&self` — незмінне посилання. Це логічно: виконання дії не змінює "сутність" агента. Але лічильник операцій — це технічна деталь, яку хочеться оновлювати.

Можна змінити на `&mut self`, але тоді:
- Неможливо викликати метод на `Rc<Agent>` (Rc дає тільки `&T`)
- Неможливо мати кілька посилань під час виклику
- Семантично неправильно: "виконати дію" не повинно вимагати ексклюзивного доступу

**RefCell<T>** пропонує елегантне рішення: **внутрішня мутабельність** (interior mutability). Замість перевірки на етапі компіляції — перевірка **під час виконання**.

---

## 30.1 Концепція внутрішньої мутабельності

### Що таке Interior Mutability

**Interior Mutability** (внутрішня мутабельність) — це патерн у Rust, що дозволяє змінювати дані навіть при наявності лише незмінного посилання `&T`.

Це звучить як порушення правил Rust. Насправді — це **контрольоване** порушення:

```text
Звичайні правила:
┌─────────────────────────────────────────┐
│  &T      →  тільки читання              │
│  &mut T  →  читання та запис            │
└─────────────────────────────────────────┘

Interior Mutability:
┌─────────────────────────────────────────┐
│  &T      →  читання та запис            │
│             (через спеціальні типи)     │
└─────────────────────────────────────────┘
```

Ключове слово — "через спеціальні типи". RefCell, Cell, Mutex — всі вони надають interior mutability, але по-різному.

### Як RefCell реалізує це

RefCell переносить перевірку правил позичання з **compile-time** на **runtime**:

| Аспект | Звичайні посилання | RefCell |
|--------|-------------------|---------|
| **Коли перевірка** | Компіляція | Виконання |
| **Помилка** | Не компілюється | Паніка (panic!) |
| **Продуктивність** | Нульові витрати | Невеликий overhead |
| **Гнучкість** | Обмежена | Висока |

**Важливо:** Правила **ті самі**, просто перевіряються в інший момент:
- Багато `borrow()` одночасно — ✅ дозволено
- Один `borrow_mut()` без інших позичань — ✅ дозволено
- `borrow()` + `borrow_mut()` одночасно — ❌ **ПАНІКА**
- Кілька `borrow_mut()` одночасно — ❌ **ПАНІКА**

### Внутрішня структура RefCell

Концептуально RefCell виглядає так:

```rust
// Спрощена схема (не реальний код)
pub struct RefCell<T> {
    borrow_state: Cell<BorrowState>,  // Стан позичання
    value: UnsafeCell<T>,             // Дані (unsafe для мутації)
}

enum BorrowState {
    Unused,           // Ніхто не позичив
    Reading(usize),   // N активних незмінних позичань
    Writing,          // Одне активне змінне позичання
}
```

При кожному виклику `borrow()` або `borrow_mut()`:
1. Перевіряється поточний `borrow_state`
2. Якщо конфлікт — `panic!`
3. Якщо все добре — оновлюється стан, повертається guard

---

## 30.2 Базове використання RefCell

### Створення та доступ

**Постановка задачі:** Продемонструвати базові операції з RefCell — створення, незмінне позичання, змінне позичання.

```rust
use std::cell::RefCell;

fn main() {
    // Створюємо RefCell з початковим значенням
    let data = RefCell::new(5);
    
    println!("Початкове значення: {:?}", data);
    
    // === Незмінне позичання через borrow() ===
    {
        // borrow() повертає Ref<T> — "guard" для читання
        let borrowed = data.borrow();
        println!("Читаємо: {}", *borrowed);
        
        // borrowed автоматично звільняється в кінці блоку
        // (коли Ref<T> виходить зі scope)
    }
    
    // === Змінне позичання через borrow_mut() ===
    {
        // borrow_mut() повертає RefMut<T> — "guard" для запису
        let mut borrowed = data.borrow_mut();
        *borrowed += 10;
        println!("Після модифікації: {}", *borrowed);
        
        // borrowed звільняється тут
    }
    
    // Фінальний результат
    println!("Кінцеве значення: {}", *data.borrow());  // 15
}
```

**Як це працює:**

1. `RefCell::new(5)` створює RefCell з початковим значенням 5
2. `data.borrow()` перевіряє стан, повертає `Ref<i32>` для читання
3. Блок `{}` обмежує scope — коли Ref виходить, позичання завершується
4. `data.borrow_mut()` перевіряє стан (ніхто не позичив), повертає `RefMut<i32>`
5. Через `*borrowed += 10` модифікуємо значення
6. Фінальний `data.borrow()` показує результат

### Ref і RefMut — розумні обгортки

`borrow()` повертає `Ref<T>`, а `borrow_mut()` повертає `RefMut<T>`. Це **guards** (охоронці), що відстежують активні позичання.

**Постановка задачі:** Показати, що можна мати кілька незмінних позичань одночасно.

```rust
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(vec![1, 2, 3]);
    
    // Кілька незмінних позичань — ДОЗВОЛЕНО
    let r1 = data.borrow();  // Перше незмінне
    let r2 = data.borrow();  // Друге незмінне — OK!
    let r3 = data.borrow();  // Третє незмінне — теж OK!
    
    println!("r1: {:?}", *r1);
    println!("r2: {:?}", *r2);
    println!("r3: {:?}", *r3);
    
    // Всі три посилання активні одночасно — це безпечно,
    // бо всі тільки читають
    
    // Щоб отримати borrow_mut(), потрібно спочатку звільнити всі borrow()
    drop(r1);
    drop(r2);
    drop(r3);
    
    // Тепер можна мутувати
    let mut m = data.borrow_mut();
    m.push(4);
    println!("Після push: {:?}", *m);
}
```

### Паніка при конфлікті

**Постановка задачі:** Показати, що відбувається при порушенні правил позичання.

```rust
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(5);
    
    // Отримуємо незмінне позичання
    let r = data.borrow();
    println!("Прочитали: {}", *r);
    
    // Спроба отримати змінне позичання при активному незмінному
    // ЦЕ ВИКЛИЧЕ ПАНІКУ!
    // let m = data.borrow_mut();  // panic: already borrowed
    
    // Правильний підхід — спочатку звільнити незмінне
    drop(r);  // Явно звільняємо
    
    // Тепер безпечно
    let mut m = data.borrow_mut();
    *m = 10;
    println!("Нове значення: {}", *m);
}
```

**Важливо:** Паніка — це runtime помилка. Програма компілюється нормально, але падає при виконанні. Це ціна гнучкості RefCell.

### Безпечні альтернативи: try_borrow

Якщо паніка неприйнятна, використовуйте `try_borrow()` та `try_borrow_mut()`:

**Постановка задачі:** Показати безпечну перевірку можливості позичання.

```rust
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(5);
    
    // Створюємо незмінне позичання
    let _r = data.borrow();
    
    // try_borrow_mut повертає Result замість паніки
    match data.try_borrow_mut() {
        Ok(mut m) => {
            *m += 1;
            println!("Успішно модифіковано");
        }
        Err(e) => {
            println!("Не вдалось отримати змінне позичання: {}", e);
        }
    }
    
    // Вивід: Не вдалось отримати змінне позичання: already borrowed
}
```

**Коли використовувати try_borrow:**
- В бібліотечному коді, де паніка неприйнятна
- Коли ви не впевнені, чи позичання вже активне
- В складних сценаріях з багатьма RefCell

---

## 30.3 Комбінація Rc + RefCell: спільна мутабельність

### Патерн Rc<RefCell<T>>

Це найпоширеніший патерн використання RefCell. Він поєднує:
- **Rc** — спільне володіння (кілька власників)
- **RefCell** — мутабельність (зміна даних)

Разом вони дають: **кілька власників, кожен може змінювати**.

**Постановка задачі:** Показати, як кілька "власників" модифікують спільні дані.

```rust
use std::rc::Rc;
use std::cell::RefCell;

fn main() {
    // Спільні мутабельні дані
    let shared = Rc::new(RefCell::new(0));
    
    // Створюємо кілька "власників"
    let owner1 = Rc::clone(&shared);
    let owner2 = Rc::clone(&shared);
    let owner3 = Rc::clone(&shared);
    
    println!("Rc count: {}", Rc::strong_count(&shared));  // 4
    
    // Кожен власник може читати
    println!("Початково: {}", *shared.borrow());  // 0
    
    // Кожен власник може змінювати
    *owner1.borrow_mut() += 5;
    println!("Після owner1: {}", *shared.borrow());  // 5
    
    *owner2.borrow_mut() += 10;
    println!("Після owner2: {}", *shared.borrow());  // 15
    
    *owner3.borrow_mut() *= 2;
    println!("Після owner3: {}", *shared.borrow());  // 30
}
```

**Як це працює:**

1. `Rc::new(RefCell::new(0))` — створюємо Rc, що містить RefCell
2. `Rc::clone()` — створюємо додаткові посилання (дешева операція)
3. `owner1.borrow_mut()` — спочатку Rc дає `&RefCell`, потім RefCell дає `&mut T`
4. Всі власники бачать однакові дані, бо це один RefCell на heap

### Візуалізація структури

```text
       STACK                         HEAP
    ┌─────────┐                 ┌─────────────────────┐
    │ shared ─┼────────────────►│ Rc {                │
    ├─────────┤                 │   strong_count: 4   │
    │ owner1 ─┼────────────────►│   weak_count: 0     │
    ├─────────┤                 │   ┌───────────────┐ │
    │ owner2 ─┼────────────────►│   │ RefCell {     │ │
    ├─────────┤                 │   │   borrow: 0   │ │
    │ owner3 ─┼────────────────►│   │   value: 30   │ │
    └─────────┘                 │   └───────────────┘ │
                                └─────────────────────┘
                                
    Чотири вказівники → один Rc → один RefCell → одне значення
```

### Практичний приклад: спільний стан місії

**Постановка задачі:** Кілька агентів оновлюють спільний стан місії — знайдені цілі, проскановану площу, попередження.

```rust
use std::rc::Rc;
use std::cell::RefCell;

/// Стан місії — спільний для всіх агентів
struct MissionState {
    targets_found: u32,
    area_scanned: f64,
    alerts: Vec<String>,
}

impl MissionState {
    fn new() -> Self {
        Self {
            targets_found: 0,
            area_scanned: 0.0,
            alerts: Vec::new(),
        }
    }
    
    fn summary(&self) -> String {
        format!(
            "Цілей: {}, Площа: {:.1} км², Попереджень: {}",
            self.targets_found, 
            self.area_scanned,
            self.alerts.len()
        )
    }
}

// Тип-псевдонім для зручності
type SharedState = Rc<RefCell<MissionState>>;

/// Агент, що має доступ до спільного стану
struct Agent {
    id: String,
    state: SharedState,
}

impl Agent {
    fn new(id: &str, state: SharedState) -> Self {
        Self {
            id: id.to_string(),
            state,
        }
    }
    
    /// Повідомити про знайдену ціль
    fn report_target(&self) {
        let mut state = self.state.borrow_mut();
        state.targets_found += 1;
        println!("[{}] Ціль знайдено! Всього: {}", 
            self.id, state.targets_found);
    }
    
    /// Повідомити про просканувану площу
    fn report_scan(&self, area: f64) {
        let mut state = self.state.borrow_mut();
        state.area_scanned += area;
        println!("[{}] Просканувано {:.1} км². Загалом: {:.1} км²", 
            self.id, area, state.area_scanned);
    }
    
    /// Додати попередження
    fn add_alert(&self, message: &str) {
        let mut state = self.state.borrow_mut();
        state.alerts.push(format!("{}: {}", self.id, message));
        println!("[{}] ⚠️ {}", self.id, message);
    }
    
    /// Отримати поточний стан (тільки читання)
    fn get_summary(&self) -> String {
        self.state.borrow().summary()
    }
}

fn main() {
    // Створюємо спільний стан місії
    let mission = Rc::new(RefCell::new(MissionState::new()));
    
    // Три агенти ділять один стан
    let scout = Agent::new("SCOUT-01", Rc::clone(&mission));
    let recon = Agent::new("RECON-01", Rc::clone(&mission));
    let combat = Agent::new("COMBAT-01", Rc::clone(&mission));
    
    println!("=== Початок місії ===\n");
    
    // Scout працює
    scout.report_scan(25.0);
    scout.report_target();
    
    println!();
    
    // Recon працює
    recon.report_scan(40.0);
    recon.add_alert("Виявлено рух на сході");
    
    println!();
    
    // Combat реагує
    combat.report_target();
    combat.add_alert("Ціль нейтралізовано");
    
    println!("\n=== Підсумок місії ===");
    println!("{}", scout.get_summary());
    
    // Перевіряємо alerts
    println!("\nВсі попередження:");
    for alert in &mission.borrow().alerts {
        println!("  • {}", alert);
    }
}
```

**Вивід:**
```text
=== Початок місії ===

[SCOUT-01] Просканувано 25.0 км². Загалом: 25.0 км²
[SCOUT-01] Ціль знайдено! Всього: 1

[RECON-01] Просканувано 40.0 км². Загалом: 65.0 км²
[RECON-01] ⚠️ Виявлено рух на сході

[COMBAT-01] Ціль знайдено! Всього: 2
[COMBAT-01] ⚠️ Ціль нейтралізовано

=== Підсумок місії ===
Цілей: 2, Площа: 65.0 км², Попереджень: 2

Всі попередження:
  • RECON-01: Виявлено рух на сході
  • COMBAT-01: Ціль нейтралізовано
```

---

## 30.4 Коли використовувати RefCell

### Типові сценарії

| Сценарій | Рішення | Пояснення |
|----------|---------|-----------|
| Лічильник у методі з `&self` | `RefCell<u32>` | Не хочемо `&mut self` для простого лічильника |
| Кеш обчислень | `RefCell<HashMap>` | Кешування — "невидима" зміна стану |
| Спільний змінний стан | `Rc<RefCell<T>>` | Кілька власників, кожен може змінювати |
| Mock-об'єкти в тестах | `RefCell<Vec>` | Записуємо виклики для перевірки |
| Observer pattern | `RefCell<Vec<Observer>>` | Додаємо/видаляємо спостерігачів |

### Коли НЕ використовувати RefCell

**1. Можна перепроєктувати з `&mut self`:**

```rust
// ❌ Не найкращий варіант
struct Counter {
    value: RefCell<u32>,
}
impl Counter {
    fn increment(&self) {
        *self.value.borrow_mut() += 1;
    }
}

// ✅ Краще — просто &mut self
struct Counter {
    value: u32,
}
impl Counter {
    fn increment(&mut self) {
        self.value += 1;
    }
}
```

**2. Багатопотоковий код:**

RefCell **НЕ потокобезпечний**. Для потоків використовуйте `Mutex<T>` (Розділ 34).

**3. Критичний до продуктивності код:**

RefCell має невеликий overhead через runtime перевірки.

### Порівняння підходів до мутабельності

```rust
// Підхід 1: Звичайна мутабельність (найкращий, якщо можливо)
struct AgentV1 {
    count: u32,
}
impl AgentV1 {
    fn action(&mut self) {  // Вимагає &mut self
        self.count += 1;
    }
}

// Підхід 2: RefCell (коли &mut неможливий)
use std::cell::RefCell;

struct AgentV2 {
    count: RefCell<u32>,
}
impl AgentV2 {
    fn action(&self) {  // Працює з &self
        *self.count.borrow_mut() += 1;
    }
}

// Підхід 3: Rc<RefCell> (спільне володіння + мутабельність)
use std::rc::Rc;

struct AgentV3 {
    shared_count: Rc<RefCell<u32>>,
}
impl AgentV3 {
    fn action(&self) {
        *self.shared_count.borrow_mut() += 1;
    }
}
```

**Правило вибору:**
1. Спробуйте `&mut self` — якщо працює, це найкраще
2. Якщо потрібен `&self` — використовуйте RefCell
3. Якщо потрібно кілька власників — `Rc<RefCell<T>>`
4. Якщо багатопотоковість — `Arc<Mutex<T>>`

---

## 30.5 Уникнення паніки: найкращі практики

### Золоте правило: мінімізуйте час позичання

Чим коротший час активного позичання — тим менший ризик конфлікту.

**Постановка задачі:** Показати різницю між довгим і коротким позичанням.

```rust
use std::cell::RefCell;

fn expensive_calculation() -> i32 {
    // Уявімо, що це довга операція
    42
}

fn main() {
    let data = RefCell::new(0);
    
    // ❌ ПОГАНО: довге позичання
    // let mut guard = data.borrow_mut();
    // let result = expensive_calculation();  // guard ще активний!
    // *guard = result;
    
    // ✅ ДОБРЕ: коротке позичання
    let result = expensive_calculation();
    *data.borrow_mut() = result;  // Миттєве позичання
    
    println!("Результат: {}", *data.borrow());
}
```

### Використовуйте блоки для обмеження scope

**Постановка задачі:** Показати, як блоки допомагають керувати часом життя guards.

```rust
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(vec![1, 2, 3]);
    
    // Читання в окремому блоці
    {
        let borrowed = data.borrow();
        println!("Довжина: {}", borrowed.len());
        println!("Елементи: {:?}", *borrowed);
        // borrowed звільняється ТУТ
    }
    
    // Тепер безпечно мутувати
    {
        let mut borrowed = data.borrow_mut();
        borrowed.push(4);
        borrowed.push(5);
        // borrowed звільняється ТУТ
    }
    
    // Знову читаємо
    println!("Після змін: {:?}", *data.borrow());
}
```

### Шаблон "позичити, обробити, звільнити"

**Постановка задачі:** Показати безпечний патерн роботи з RefCell.

```rust
use std::cell::RefCell;

struct Agent {
    name: String,
    stats: RefCell<AgentStats>,
}

struct AgentStats {
    moves: u32,
    scans: u32,
    errors: u32,
}

impl Agent {
    fn new(name: &str) -> Self {
        Self {
            name: name.to_string(),
            stats: RefCell::new(AgentStats {
                moves: 0,
                scans: 0,
                errors: 0,
            }),
        }
    }
    
    // ✅ Правильний патерн: коротке позичання для кожної операції
    fn record_move(&self) {
        self.stats.borrow_mut().moves += 1;
    }
    
    fn record_scan(&self) {
        self.stats.borrow_mut().scans += 1;
    }
    
    fn record_error(&self) {
        self.stats.borrow_mut().errors += 1;
    }
    
    // ✅ Правильно: читаємо, форматуємо, звільняємо
    fn get_report(&self) -> String {
        let stats = self.stats.borrow();
        format!(
            "{}: moves={}, scans={}, errors={}",
            self.name, stats.moves, stats.scans, stats.errors
        )
    }
}

fn main() {
    let agent = Agent::new("SCOUT-01");
    
    agent.record_move();
    agent.record_scan();
    agent.record_move();
    agent.record_error();
    agent.record_move();
    agent.record_scan();
    
    println!("{}", agent.get_report());
}
```

---

## 30.6 Практичні приклади для агентів

### Кешування обчислень

**Постановка задачі:** Агент обчислює шлях до цілі. Обчислення дороге, тому кешуємо результат. Але метод `get_path` повинен приймати `&self`.

```rust
use std::cell::RefCell;
use std::collections::HashMap;

type Position = (i32, i32);
type Path = Vec<Position>;

struct Navigator {
    position: Position,
    path_cache: RefCell<HashMap<(Position, Position), Path>>,
}

impl Navigator {
    fn new(x: i32, y: i32) -> Self {
        Self {
            position: (x, y),
            path_cache: RefCell::new(HashMap::new()),
        }
    }
    
    /// Отримати шлях до цілі (з кешуванням)
    fn get_path(&self, target: Position) -> Path {
        let key = (self.position, target);
        
        // Перевіряємо кеш
        if let Some(path) = self.path_cache.borrow().get(&key) {
            println!("  [cache hit]");
            return path.clone();
        }
        
        // Обчислюємо шлях
        println!("  [calculating...]");
        let path = self.calculate_path(target);
        
        // Зберігаємо в кеш
        self.path_cache.borrow_mut().insert(key, path.clone());
        
        path
    }
    
    fn calculate_path(&self, target: Position) -> Path {
        vec![self.position, target]
    }
}

fn main() {
    let nav = Navigator::new(0, 0);
    
    println!("Запит шляху до (5, 3):");
    let _ = nav.get_path((5, 3));
    
    println!("Запит того ж шляху:");
    let _ = nav.get_path((5, 3));  // cache hit
    
    println!("Запит іншого шляху:");
    let _ = nav.get_path((2, 4));
}
```

### Observer Pattern

**Постановка задачі:** Агент сповіщає підписників про свої дії.

```rust
use std::cell::RefCell;

type Observer = Box<dyn Fn(&str, &str)>;

struct Agent {
    id: String,
    observers: RefCell<Vec<Observer>>,
}

impl Agent {
    fn new(id: &str) -> Self {
        Self {
            id: id.to_string(),
            observers: RefCell::new(Vec::new()),
        }
    }
    
    fn add_observer(&self, observer: Observer) {
        self.observers.borrow_mut().push(observer);
    }
    
    fn notify(&self, event: &str) {
        for observer in self.observers.borrow().iter() {
            observer(&self.id, event);
        }
    }
    
    fn do_action(&self) {
        println!("[{}] Виконую дію", self.id);
        self.notify("action_performed");
    }
}

fn main() {
    let agent = Agent::new("SCOUT-01");
    
    agent.add_observer(Box::new(|id, event| {
        println!("  Logger: [{}] {}", id, event);
    }));
    
    agent.do_action();
}
```

---

## 30.7 Лабораторна робота

**Мета:** Застосувати RefCell для внутрішньої мутабельності.

### Частина 1: Логер з лічильником (3 бали)

Реалізуйте логер через `&self`.

### Частина 2: Спільний журнал місії (4 бали)

Використайте `Rc<RefCell<MissionLog>>`.

### Частина 3: Безпечне позичання (3 бали)

Створіть обгортку з `try_borrow_mut()`.

---

## Підсумок

`RefCell<T>` — інструмент для **внутрішньої мутабельності**:

| Метод | Повертає | При конфлікті |
|-------|----------|---------------|
| `borrow()` | `Ref<T>` | panic! |
| `borrow_mut()` | `RefMut<T>` | panic! |
| `try_borrow()` | `Result<Ref<T>, _>` | Err |
| `try_borrow_mut()` | `Result<RefMut<T>, _>` | Err |

**Ключові правила:**

- ✅ RefCell для мутації через `&self`
- ✅ `Rc<RefCell<T>>` для спільної мутабельності
- ✅ Мінімізуйте час позичання
- ⚠️ Порушення правил = **паніка**
- ⚠️ RefCell **НЕ потокобезпечний**

---

## Зв'язок з наступним розділом

RefCell має overhead через runtime перевірки. Для простих випадків є **легша альтернатива**.

У **Розділі 31: Cell — легка альтернатива** ви дізнаєтесь:
- Коли `Cell<T>` краще за `RefCell<T>`
- `OnceCell` та `LazyCell` для одноразової ініціалізації
- Як обирати між різними типами interior mutability
