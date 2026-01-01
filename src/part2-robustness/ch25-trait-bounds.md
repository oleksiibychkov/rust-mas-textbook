# Розділ 25: Обмеження трейтів та Where-клаузи

## Вступ

У попередньому розділі ви створювали generic функції та структури. Але виникає питання: що можна зробити з невідомим типом `T`? Відповідь — **нічого**. Ви не можете порівняти два значення `T`, вивести `T` на екран, чи навіть скопіювати `T`. Компілятор не знає, що цей тип вміє.

```rust
// ❌ Не компілюється
fn print_twice<T>(value: T) {
    println!("{}", value);  // Помилка! T не реалізує Display
}
```

Помилка говорить: "the trait `Display` is not implemented for `T`". Компілятор правий — він не знає, чи тип `T` взагалі можна вивести на екран.

**Trait bounds** (обмеження типів) — спосіб сказати компілятору: "Цей generic тип T **має** реалізовувати певні traits". Замість "будь-який T" ви кажете "T, який можна вивести" або "T, який можна клонувати":

```rust
// ✅ Компілюється — T обмежений Display
fn print_twice<T: std::fmt::Display>(value: T) {
    println!("{}", value);
    println!("{}", value);
}
```

Тепер компілятор знає: якщо хтось викличе `print_twice(42)`, тип `i32` **точно** реалізує `Display`. Якщо спробують викликати з типом без `Display` — помилка компіляції.

---

## 25.1 Базовий синтаксис обмежень

### Синтаксис T: Trait

Обмеження записується після параметра типу через двокрапку: `T: Trait`.

**Постановка задачі:** Створити функції, що використовують різні можливості типу — вивід, клонування, створення за замовчуванням.

```rust
use std::fmt::Display;

// T має реалізовувати Display — можна виводити через {}
fn show<T: Display>(item: T) {
    println!("Значення: {}", item);
}

// T має реалізовувати Clone — можна клонувати
fn duplicate<T: Clone>(item: &T) -> (T, T) {
    (item.clone(), item.clone())
}

// T має реалізовувати Default — є значення за замовчуванням
fn create_default<T: Default>() -> T {
    T::default()
}

fn main() {
    show(42);                     // i32: Display ✓
    show("hello");                // &str: Display ✓
    
    let pair = duplicate(&String::from("test"));  // String: Clone ✓
    println!("{:?}", pair);       // ("test", "test")
    
    let num: i32 = create_default();  // i32: Default ✓
    println!("Default i32: {}", num); // 0
}
```

**Як це працює:**

1. `T: Display` — обмеження: T повинен реалізовувати Display
2. Всередині функції можна використовувати методи Display (тобто `{}` форматування)
3. При виклику компілятор перевіряє, чи аргумент відповідає обмеженню
4. Якщо ні — помилка компіляції з чітким повідомленням

### Обмеження у структурах

Можна додавати обмеження до структур, але це **рідко** потрібно.

```rust
use std::fmt::Display;

// Структура вимагає T: Display
struct Wrapper<T: Display> {
    value: T,
}

impl<T: Display> Wrapper<T> {
    fn new(value: T) -> Self {
        Self { value }
    }
    
    fn show(&self) {
        println!("Wrapped: {}", self.value);
    }
}

fn main() {
    let w = Wrapper::new(42);  // i32: Display ✓
    w.show();
    
    // ❌ Не компілюється — Vec<i32> не реалізує Display
    // let w2 = Wrapper::new(vec![1, 2, 3]);
}
```

**Важливо:** Обмеження у визначенні структури означає, що ви **не зможете** створити `Wrapper` для типу без `Display`. Це занадто обмежуюче у більшості випадків.

### Обмеження тільки в impl — гнучкіший підхід

**Краща практика:** Додавати обмеження тільки там, де вони потрібні — в `impl`, а не в структурі.

**Постановка задачі:** Створити `Container<T>`, де базові методи працюють для будь-якого T, а додаткові — тільки для T з певними можливостями.

```rust
use std::fmt::Display;

// Структура БЕЗ обмежень — максимальна гнучкість
struct Container<T> {
    value: T,
}

// Базові методи — для ВСІХ T
impl<T> Container<T> {
    fn new(value: T) -> Self {
        Self { value }
    }
    
    fn into_inner(self) -> T {
        self.value
    }
    
    fn get_ref(&self) -> &T {
        &self.value
    }
}

// Метод print() — тільки для T: Display
impl<T: Display> Container<T> {
    fn print(&self) {
        println!("{}", self.value);
    }
}

// Метод duplicate() — тільки для T: Clone
impl<T: Clone> Container<T> {
    fn duplicate(&self) -> Self {
        Self { value: self.value.clone() }
    }
}

// Метод reset() — тільки для T: Default
impl<T: Default> Container<T> {
    fn reset(&mut self) {
        self.value = T::default();
    }
}

fn main() {
    // Container з Vec<i32>
    let c1 = Container::new(vec![1, 2, 3]);
    println!("{:?}", c1.get_ref());  // Працює — базовий метод
    let c1_copy = c1.duplicate();    // Працює — Vec: Clone
    // c1.print();                   // ❌ НЕ працює — Vec: !Display
    
    // Container з i32
    let mut c2 = Container::new(42);
    c2.print();                      // Працює — i32: Display
    c2.reset();                      // Працює — i32: Default
    println!("After reset: {}", c2.into_inner());  // 0
}
```

**Як це працює:**

1. `Container<T>` не має обмежень — можна створити для будь-якого типу
2. `impl<T> Container<T>` — методи для всіх T
3. `impl<T: Display> Container<T>` — методи тільки для T з Display
4. Компілятор показує тільки методи, доступні для конкретного типу

---

## 25.2 Множинні обмеження

### Синтаксис з +

Коли потрібно кілька traits, вони комбінуються через `+`.

**Постановка задачі:** Функція, що потребує і Display (для виводу), і Debug (для налагодження), і Clone (для копіювання).

```rust
use std::fmt::{Display, Debug};

// T має реалізовувати і Display, і Debug
fn log_value<T: Display + Debug>(value: T) {
    println!("Display: {}", value);
    println!("Debug: {:?}", value);
}

// T має реалізовувати Clone, Default, і PartialEq
fn process<T: Clone + Default + PartialEq>(value: T) -> T {
    let default_val = T::default();
    if value == default_val {
        println!("Значення дорівнює default");
        value.clone()
    } else {
        println!("Значення відрізняється від default");
        default_val
    }
}

fn main() {
    // i32 реалізує всі потрібні traits
    log_value(42);
    
    let result = process(0);  // 0 == default для i32
    println!("Result: {}", result);
    
    let result2 = process(5);  // 5 != default
    println!("Result2: {}", result2);  // 0
}
```

### Типові комбінації обмежень

| Комбінація | Призначення |
|------------|-------------|
| `Clone + Copy` | Легке копіювання (Copy вимагає Clone) |
| `Debug + Display` | Логування + вивід користувачу |
| `PartialEq + Eq + Hash` | Ключ для HashMap |
| `PartialOrd + Ord` | Сортування |
| `Send + Sync` | Багатопотоковість |
| `Clone + Default` | Копії та значення за замовчуванням |
| `Serialize + Deserialize` | Серіалізація (serde) |

---

## 25.3 Where-клаузи — для читабельності

### Проблема довгих сигнатур

Коли обмежень багато, сигнатура функції стає важкочитабельною:

```rust
// ❌ Важко читати — занадто багато інформації в одному рядку
fn complex_function<T: Clone + Debug + Display + PartialEq, U: Clone + Default + Send>(
    first: T,
    second: U,
) -> T {
    // ...
}
```

### Where на допомогу

Where-клауза виносить обмеження **після** сигнатури, роблячи код читабельнішим:

```rust
// ✅ Набагато краще — сигнатура чиста, bounds окремо
fn complex_function<T, U>(first: T, second: U) -> T
where
    T: Clone + Debug + Display + PartialEq,
    U: Clone + Default + Send,
{
    // ...
}
```

### Коли використовувати where

**Використовуйте where коли:**
- Більше одного параметра типу
- Більше двох bounds для одного параметра
- Bounds містять складні типи (асоційовані типи, ітератори)
- Просто для кращої читабельності

**Inline bounds достатньо коли:**
- Один параметр з одним-двома bounds
- Простий випадок

**Постановка задачі:** Порівняти inline та where синтаксис для різних випадків.

```rust
// Простий випадок — inline достатньо
fn simple<T: Clone>(value: T) -> T {
    value.clone()
}

// Складний випадок — where краще
fn complex<T, U, V>(a: T, b: U, c: V) -> V
where
    T: Clone + Send,
    U: Default + Debug,
    V: From<T> + From<U>,
{
    // Можна конвертувати T в V
    V::from(a)
}

// Дуже складний з асоційованими типами
fn process_iterator<I, T>(iter: I) -> Vec<T>
where
    I: Iterator<Item = T>,
    T: Clone + PartialOrd,
{
    let mut result: Vec<T> = iter.collect();
    result.sort_by(|a, b| a.partial_cmp(b).unwrap());
    result
}

fn main() {
    let data = vec![3, 1, 4, 1, 5, 9, 2, 6];
    let sorted = process_iterator(data.into_iter());
    println!("{:?}", sorted);  // [1, 1, 2, 3, 4, 5, 6, 9]
}
```

### Where з асоційованими типами

Where особливо корисний для обмежень на асоційовані типи:

```rust
use std::iter::Iterator;

// Bound на асоційований тип Item
fn sum_integers<I>(iter: I) -> i32
where
    I: Iterator<Item = i32>,
{
    iter.sum()
}

// Ще складніший — Item має бути парою
fn process_pairs<I, T>(iter: I) -> Vec<T>
where
    I: Iterator<Item = (T, T)>,
    T: Clone + PartialOrd,
{
    iter.map(|(a, b)| {
        if a > b { a } else { b }  // Вибираємо більше з пари
    }).collect()
}

fn main() {
    let numbers = vec![1, 2, 3, 4, 5];
    println!("Sum: {}", sum_integers(numbers.into_iter()));  // 15
    
    let pairs = vec![(1, 5), (8, 3), (2, 9)];
    let maxes = process_pairs(pairs.into_iter());
    println!("Maxes: {:?}", maxes);  // [5, 8, 9]
}
```

---

## 25.4 impl Trait — анонімні типи

### impl Trait в аргументах

`impl Trait` — альтернативний, коротший синтаксис для простих випадків:

```rust
use std::fmt::Display;

// Варіант 1: Generic з bound
fn print_generic<T: Display>(value: T) {
    println!("{}", value);
}

// Варіант 2: impl Trait — коротше
fn print_impl(value: impl Display) {
    println!("{}", value);
}

fn main() {
    print_generic(42);  // Працює
    print_impl(42);     // Теж працює — еквівалентно
}
```

**Різниця:** `impl Trait` коротший, але ви не можете посилатись на тип `T` деінде (наприклад, якщо потрібно два аргументи одного типу).

```rust
// ❌ З impl Trait не можна гарантувати однаковий тип
fn compare(a: impl PartialEq, b: impl PartialEq) -> bool {
    // a == b  // Помилка! Можуть бути різні типи
    false
}

// ✅ Generic гарантує однаковий тип
fn compare_generic<T: PartialEq>(a: T, b: T) -> bool {
    a == b
}
```

### impl Trait у поверненні — головна перевага

Справжня сила `impl Trait` — повернення типу без його іменування:

```rust
// Повертаємо "якийсь тип, що реалізує Iterator"
fn make_counter() -> impl Iterator<Item = i32> {
    (0..10).map(|x| x * 2)
}

// Без impl Trait довелось би писати ДУЖЕ довгий тип:
// fn make_counter() -> std::iter::Map<std::ops::Range<i32>, fn(i32) -> i32>

fn main() {
    for value in make_counter() {
        println!("{}", value);  // 0, 2, 4, 6, 8, 10, 12, 14, 16, 18
    }
}
```

Це особливо корисно для **closures** — їх типи неможливо записати вручну.

### Обмеження impl Trait

`impl Trait` у поверненні означає **один конкретний тип**. Різні гілки if/else не можуть повертати різні типи:

```rust
// ❌ Не компілюється — різні типи в гілках
fn make_iterator(ascending: bool) -> impl Iterator<Item = i32> {
    if ascending {
        0..10           // Range<i32>
    } else {
        (0..10).rev()   // Rev<Range<i32>> — ІНШИЙ тип!
    }
}
```

Для таких випадків потрібен **trait object**:

```rust
// ✅ Box<dyn Trait> — динамічний поліморфізм
fn make_iterator_dynamic(ascending: bool) -> Box<dyn Iterator<Item = i32>> {
    if ascending {
        Box::new(0..10)
    } else {
        Box::new((0..10).rev())
    }
}

fn main() {
    let iter = make_iterator_dynamic(false);
    let result: Vec<_> = iter.collect();
    println!("{:?}", result);  // [9, 8, 7, 6, 5, 4, 3, 2, 1, 0]
}
```

---

## 25.5 Sized та ?Sized — робота з DST

### Неявне обмеження Sized

За замовчуванням **всі** параметри типу мають неявне обмеження `Sized`:

```rust
// Ці два варіанти ЕКВІВАЛЕНТНІ
fn foo<T>(value: T) { }
fn foo<T: Sized>(value: T) { }
```

`Sized` означає, що розмір типу відомий на етапі компіляції. Це потрібно для розміщення значення на стеку.

### Типи без фіксованого розміру (DST)

Деякі типи **не мають** фіксованого розміру:
- `str` — рядковий слайс (не `String`!)
- `[T]` — слайс (не `Vec<T>`!)
- `dyn Trait` — trait object

Ці типи називаються **DST** (Dynamically Sized Types). З ними можна працювати тільки через посилання.

### ?Sized — зняття обмеження

Щоб функція могла приймати DST (через посилання), використовується `?Sized`:

```rust
use std::fmt::Display;

// Працює тільки з Sized типами
fn print_sized<T: Display>(value: T) {
    println!("{}", value);
}

// Працює і з несized типами (через посилання)
fn print_unsized<T: Display + ?Sized>(value: &T) {
    println!("{}", value);
}

fn main() {
    // String — Sized
    let string = String::from("hello");
    print_sized(string.clone());
    print_unsized(&string);
    
    // str — НЕ Sized!
    let s: &str = "world";
    // print_sized(*s);  // ❌ Не можна — str не Sized
    print_unsized(s);    // ✅ Працює через ?Sized
    
    // Слайс — не Sized
    let slice: &[i32] = &[1, 2, 3];
    // Потрібна функція з ?Sized щоб приймати слайси
}
```

**Коли використовувати ?Sized:**
- Функція приймає `&T` або `Box<T>`
- Хочете підтримувати `str`, `[T]`, `dyn Trait`

---

## 25.6 Blanket implementations — автоматичні реалізації

### Що таке blanket implementation

**Blanket implementation** — реалізація trait для **всіх типів**, що відповідають певному обмеженню.

Найвідоміший приклад зі стандартної бібліотеки:

```rust
// Зі std (спрощено):
// ToString реалізований для ВСІХ типів, що реалізують Display
impl<T: Display> ToString for T {
    fn to_string(&self) -> String {
        format!("{}", self)
    }
}
```

Це означає: якщо ваш тип реалізує `Display`, він **автоматично** отримує `to_string()`.

**Постановка задачі:** Показати, як Display автоматично дає ToString.

```rust
use std::fmt::{Display, Formatter, Result};

struct Point {
    x: i32,
    y: i32,
}

// Реалізуємо ТІЛЬКИ Display
impl Display for Point {
    fn fmt(&self, f: &mut Formatter<'_>) -> Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}

fn main() {
    let p = Point { x: 10, y: 20 };
    
    // to_string() доступний АВТОМАТИЧНО завдяки blanket impl!
    let s: String = p.to_string();
    println!("{}", s);  // (10, 20)
}
```

### Власний blanket implementation

**Постановка задачі:** Створити trait `Loggable`, який автоматично реалізований для всіх типів з `Debug`.

```rust
use std::fmt::Debug;

// Наш trait
trait Loggable {
    fn log(&self);
    fn log_with_prefix(&self, prefix: &str);
}

// Blanket implementation — для ВСІХ T: Debug
impl<T: Debug> Loggable for T {
    fn log(&self) {
        println!("[LOG] {:?}", self);
    }
    
    fn log_with_prefix(&self, prefix: &str) {
        println!("[{}] {:?}", prefix, self);
    }
}

#[derive(Debug)]
struct Agent {
    id: String,
    battery: u8,
}

fn main() {
    let agent = Agent {
        id: "A-001".to_string(),
        battery: 85,
    };
    
    // log() доступний, бо Agent: Debug!
    agent.log();
    agent.log_with_prefix("AGENT");
    
    // Працює навіть для примітивів
    42.log();
    "hello".log();
    vec![1, 2, 3].log();
}
```

### Orphan rule — обмеження на реалізації

Rust має **orphan rule** — ви можете реалізувати trait тільки якщо:
- Trait визначений у **вашому** крейті, АБО
- Тип визначений у **вашому** крейті

```rust
// ❌ Не можна: і Display, і Vec — зі std
// impl std::fmt::Display for Vec<i32> { ... }

// ✅ Можна: MyTrait — наш
trait MyTrait {
    fn my_method(&self);
}
impl MyTrait for String {
    fn my_method(&self) { println!("MyTrait for String"); }
}

// ✅ Можна: MyType — наш
struct MyType(i32);
impl std::fmt::Display for MyType {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "MyType({})", self.0)
    }
}
```

---

## 25.7 Supertraits — ієрархія трейтів

### Trait, що вимагає інший trait

**Supertrait** — trait, який вимагає реалізації іншого trait. Якщо тип реалізує "дочірній" trait, він **обов'язково** реалізує і "батьківський".

**Постановка задачі:** Trait `Agent` вимагає `Debug` — щоб агента можна було логувати.

```rust
use std::fmt::Debug;

// Agent вимагає Debug — це supertrait
trait Agent: Debug {
    fn id(&self) -> &str;
    fn execute(&mut self);
    
    // Default метод може використовувати Debug
    fn log_state(&self) {
        println!("Agent state: {:?}", self);
    }
}

#[derive(Debug)]  // Обов'язково — Agent вимагає Debug
struct Drone {
    id: String,
    altitude: f64,
}

impl Agent for Drone {
    fn id(&self) -> &str {
        &self.id
    }
    
    fn execute(&mut self) {
        self.altitude += 10.0;
        println!("Drone {} піднявся до {}", self.id, self.altitude);
    }
}

// Функція може використовувати і Agent, і Debug методи
fn process_agent(agent: &mut impl Agent) {
    println!("Processing: {:?}", agent);  // Debug (supertrait)
    agent.execute();                       // Agent
    agent.log_state();                     // Agent (використовує Debug)
}

fn main() {
    let mut drone = Drone {
        id: "D-001".to_string(),
        altitude: 0.0,
    };
    
    process_agent(&mut drone);
}
```

### Кілька supertraits

```rust
use std::fmt::{Debug, Display};

// Printable вимагає І Debug, І Display
trait Printable: Debug + Display {
    fn print_all(&self) {
        println!("Debug: {:?}", self);
        println!("Display: {}", self);
    }
}
```

---

## 25.8 Практичний приклад: generic контейнер агентів

**Постановка задачі:** Створити `Swarm<A>` — контейнер агентів, де доступні різні методи залежно від можливостей агента.

```rust
use std::fmt::Debug;

// Trait агента з supertrait Debug + Clone
trait Agent: Debug + Clone {
    fn id(&self) -> &str;
    fn position(&self) -> (f64, f64);
    fn act(&mut self);
}

// Контейнер агентів
struct Swarm<A: Agent> {
    agents: Vec<A>,
    name: String,
}

// Базові методи — для всіх Agent
impl<A: Agent> Swarm<A> {
    fn new(name: &str) -> Self {
        Self {
            agents: Vec::new(),
            name: name.to_string(),
        }
    }
    
    fn add(&mut self, agent: A) {
        println!("Додаємо агента {} до {}", agent.id(), self.name);
        self.agents.push(agent);
    }
    
    fn count(&self) -> usize {
        self.agents.len()
    }
    
    fn step(&mut self) {
        for agent in &mut self.agents {
            agent.act();
        }
    }
}

// Метод clone_all — потрібен Agent: Clone (вже є через supertrait)
impl<A: Agent> Swarm<A> {
    fn clone_all(&self) -> Vec<A> {
        self.agents.clone()
    }
}

// Метод find_nearest — потрібна можливість порівнювати
impl<A: Agent> Swarm<A> {
    fn find_nearest(&self, pos: (f64, f64)) -> Option<&A> {
        self.agents.iter().min_by(|a, b| {
            let dist_a = distance(a.position(), pos);
            let dist_b = distance(b.position(), pos);
            dist_a.partial_cmp(&dist_b).unwrap()
        })
    }
}

fn distance(p1: (f64, f64), p2: (f64, f64)) -> f64 {
    let dx = p2.0 - p1.0;
    let dy = p2.1 - p1.1;
    (dx * dx + dy * dy).sqrt()
}

// Конкретний тип агента
#[derive(Debug, Clone)]
struct Drone {
    id: String,
    x: f64,
    y: f64,
}

impl Agent for Drone {
    fn id(&self) -> &str { &self.id }
    fn position(&self) -> (f64, f64) { (self.x, self.y) }
    fn act(&mut self) {
        self.x += 1.0;
        self.y += 1.0;
        println!("Drone {} moved to ({}, {})", self.id, self.x, self.y);
    }
}

fn main() {
    let mut swarm = Swarm::new("Alpha");
    
    swarm.add(Drone { id: "D-001".to_string(), x: 0.0, y: 0.0 });
    swarm.add(Drone { id: "D-002".to_string(), x: 10.0, y: 10.0 });
    swarm.add(Drone { id: "D-003".to_string(), x: 5.0, y: 3.0 });
    
    println!("Агентів у рої: {}", swarm.count());
    
    // Знайти найближчого до точки (4, 4)
    if let Some(nearest) = swarm.find_nearest((4.0, 4.0)) {
        println!("Найближчий: {:?}", nearest);
    }
    
    // Крок симуляції
    swarm.step();
    
    // Клонувати всіх
    let backup = swarm.clone_all();
    println!("Backup: {} агентів", backup.len());
}
```

---

## 25.9 Лабораторна робота

**Завдання:** Застосувати trait bounds для типобезпечних абстракцій.

### Частина 1: Generic функції з правильними bounds (3 бали)

```rust
// Знайти мінімум у слайсі — які bounds?
fn find_min<T: ???>(slice: &[T]) -> Option<&T> { }

// Видалити дублікати — які bounds?
fn unique<T: ???>(items: Vec<T>) -> Vec<T> { }

// Підрахувати кількість кожного елемента — які bounds?
fn count_items<T: ???>(items: &[T]) -> HashMap<T, usize> { }
```

### Частина 2: Conditional methods (4 бали)

```rust
struct Registry<K, V> {
    items: HashMap<K, V>,
}

// Базові методи — для всіх K, V
// get/insert — потрібні певні bounds для K
// find_by_value — потрібен bound для V
// debug_print — потрібні bounds для обох
```

### Частина 3: Trait з supertraits (3 бали)

```rust
trait Identifiable: ??? {
    fn id(&self) -> &str;
    fn log_id(&self);  // Default метод, що використовує supertrait
}
```

**Критерії оцінювання:**

| Критерій | Бали |
|----------|------|
| Generic функції з правильними bounds | 3 |
| Conditional methods | 4 |
| Trait з supertraits | 3 |
| **Максимум** | **10** |

---

## Підсумок

Trait bounds — механізм, що робить generic код корисним:

| Концепція | Синтаксис | Призначення |
|-----------|-----------|-------------|
| **Trait bound** | `T: Clone` | Обмеження типу |
| **Multiple bounds** | `T: Clone + Debug` | Кілька обмежень |
| **Where clause** | `where T: Clone` | Читабельність |
| **impl Trait** | `impl Clone` | Анонімний тип |
| **?Sized** | `T: ?Sized` | Робота з DST |
| **Blanket impl** | `impl<T: X> Y for T` | Автоматичні реалізації |
| **Supertrait** | `trait X: Y` | Ієрархія |

**Ключові правила:**
- Bounds у `impl` гнучкіші, ніж у структурі
- Where для складних випадків
- `impl Trait` для простих випадків і повернення
- `?Sized` для роботи зі слайсами та str

---

## Зв'язок з наступним розділом

Ви опанували traits, generics та bounds — потужні інструменти абстракції. Тепер час навчитись **зберігати та передавати** дані агента.

У **Розділі 26: Серіалізація та serde** ви дізнаєтесь:
- Як перетворювати структури на JSON/TOML
- Як завантажувати конфігурацію з файлів
- Як обмінюватись даними між агентами
- Як використовувати derive макроси serde
