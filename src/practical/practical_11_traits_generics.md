# Практичне заняття 11: Traits та Generics

## Мета заняття

Після цього заняття ви зможете:
- Розуміти концепцію traits як інтерфейсів поведінки
- Використовувати стандартні traits (Debug, Clone, PartialEq, Default)
- Створювати власні traits для абстрагування поведінки
- Писати узагальнений (generic) код з параметрами типів
- Застосовувати trait bounds для обмеження generic типів

---

## Теоретичний вступ

### Що таке Trait?

**Trait** — це спосіб визначити спільну поведінку для різних типів. Це схоже на:
- Інтерфейси в Java/C#
- Протоколи в Swift
- Типажі в Haskell

```rust
// Trait визначає "що тип вміє робити"
trait Flyable {
    fn fly(&self);
    fn land(&self);
}
```

### Що таке Generics?

**Generics** — це параметри типів, що дозволяють писати код, який працює з різними типами:

```rust
// Функція, що працює з будь-яким типом T
fn print_twice<T: std::fmt::Display>(value: T) {
    println!("{}", value);
    println!("{}", value);
}
```

### Зв'язок Traits і Generics

Traits і Generics працюють разом:
- **Generics** визначають "з якими типами працює код"
- **Trait bounds** обмежують "що ці типи мають вміти"

```rust
// T має реалізовувати Clone та Debug
fn process<T: Clone + Debug>(item: T) { ... }
```

---

## Стандартні Traits

### Debug — для виводу налагодження

```rust
#[derive(Debug)]
struct Drone {
    id: u32,
    name: String,
    battery: i32,
}

fn main() {
    let drone = Drone {
        id: 1,
        name: String::from("Alpha"),
        battery: 85,
    };
    
    // {:?} — Debug формат
    println!("{:?}", drone);
    
    // {:#?} — Debug з форматуванням
    println!("{:#?}", drone);
}
```

**Вивід:**
```
Drone { id: 1, name: "Alpha", battery: 85 }
Drone {
    id: 1,
    name: "Alpha",
    battery: 85,
}
```

### Clone — для явного копіювання

```rust
#[derive(Debug, Clone)]
struct Position {
    x: f64,
    y: f64,
}

fn main() {
    let pos1 = Position { x: 10.0, y: 20.0 };
    let pos2 = pos1.clone();  // Явне копіювання
    
    println!("pos1: {:?}", pos1);  // pos1 все ще доступна
    println!("pos2: {:?}", pos2);
}
```

### Copy — для автоматичного копіювання

```rust
#[derive(Debug, Clone, Copy)]  // Copy вимагає Clone
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p1 = Point { x: 10, y: 20 };
    let p2 = p1;  // Копіювання, не move!
    
    println!("p1: {:?}", p1);  // OK — p1 скопійовано
    println!("p2: {:?}", p2);
}
```

**Примітка:** Copy можна реалізувати тільки для типів, що не містять heap-даних (String, Vec тощо не можуть бути Copy).

### PartialEq та Eq — для порівняння

```rust
#[derive(Debug, PartialEq)]
struct Drone {
    id: u32,
    name: String,
}

fn main() {
    let d1 = Drone { id: 1, name: String::from("Alpha") };
    let d2 = Drone { id: 1, name: String::from("Alpha") };
    let d3 = Drone { id: 2, name: String::from("Beta") };
    
    println!("d1 == d2: {}", d1 == d2);  // true
    println!("d1 == d3: {}", d1 == d3);  // false
}
```

### PartialOrd та Ord — для впорядкування

```rust
#[derive(Debug, PartialEq, Eq, PartialOrd, Ord)]
struct Priority(u8);

fn main() {
    let low = Priority(1);
    let high = Priority(10);
    
    println!("low < high: {}", low < high);  // true
    
    let mut priorities = vec![Priority(5), Priority(1), Priority(10)];
    priorities.sort();
    println!("{:?}", priorities);  // [Priority(1), Priority(5), Priority(10)]
}
```

### Default — значення за замовчуванням

```rust
#[derive(Debug, Default)]
struct DroneConfig {
    max_altitude: i32,
    max_speed: f64,
    battery_threshold: i32,
}

fn main() {
    // Всі поля отримають значення за замовчуванням (0, 0.0, 0)
    let config = DroneConfig::default();
    println!("{:?}", config);
    
    // Частково перезаписати
    let config = DroneConfig {
        max_altitude: 100,
        ..Default::default()
    };
    println!("{:?}", config);
}
```

### Hash — для хешування

```rust
use std::collections::HashSet;

#[derive(Debug, PartialEq, Eq, Hash)]
struct DroneId(String);

fn main() {
    let mut seen: HashSet<DroneId> = HashSet::new();
    
    seen.insert(DroneId(String::from("Alpha")));
    seen.insert(DroneId(String::from("Beta")));
    seen.insert(DroneId(String::from("Alpha")));  // Дублікат
    
    println!("Унікальних: {}", seen.len());  // 2
}
```

---

## Реалізація Traits

### Власна реалізація Debug

```rust
use std::fmt;

struct Drone {
    name: String,
    battery: i32,
}

impl fmt::Debug for Drone {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "Drone[{}](🔋{}%)", self.name, self.battery)
    }
}

fn main() {
    let drone = Drone {
        name: String::from("Alpha"),
        battery: 75,
    };
    println!("{:?}", drone);  // Drone[Alpha](🔋75%)
}
```

### Реалізація Display

```rust
use std::fmt;

struct Drone {
    name: String,
    x: i32,
    y: i32,
    battery: i32,
}

impl fmt::Display for Drone {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "Дрон '{}' на позиції ({}, {}), заряд: {}%",
               self.name, self.x, self.y, self.battery)
    }
}

fn main() {
    let drone = Drone {
        name: String::from("Alpha"),
        x: 100,
        y: 50,
        battery: 80,
    };
    
    println!("{}", drone);  // Дрон 'Alpha' на позиції (100, 50), заряд: 80%
}
```

### Власна реалізація PartialEq

```rust
struct Drone {
    id: u32,
    name: String,
    battery: i32,
}

// Порівнюємо тільки за id
impl PartialEq for Drone {
    fn eq(&self, other: &Self) -> bool {
        self.id == other.id
    }
}

fn main() {
    let d1 = Drone { id: 1, name: String::from("Alpha"), battery: 100 };
    let d2 = Drone { id: 1, name: String::from("Beta"), battery: 50 };
    
    // Однакові за id, хоча name та battery різні
    println!("d1 == d2: {}", d1 == d2);  // true
}
```

---

## Створення власних Traits

### Базовий trait

```rust
trait Flyable {
    fn take_off(&mut self);
    fn land(&mut self);
    fn is_flying(&self) -> bool;
}

struct Drone {
    name: String,
    flying: bool,
}

impl Flyable for Drone {
    fn take_off(&mut self) {
        self.flying = true;
        println!("{} злітає!", self.name);
    }
    
    fn land(&mut self) {
        self.flying = false;
        println!("{} сідає!", self.name);
    }
    
    fn is_flying(&self) -> bool {
        self.flying
    }
}

fn main() {
    let mut drone = Drone {
        name: String::from("Alpha"),
        flying: false,
    };
    
    drone.take_off();
    println!("Летить: {}", drone.is_flying());
    drone.land();
}
```

### Trait з методами за замовчуванням

```rust
trait Describable {
    fn name(&self) -> &str;
    
    // Метод за замовчуванням — можна перевизначити
    fn describe(&self) -> String {
        format!("Об'єкт: {}", self.name())
    }
    
    fn short_name(&self) -> String {
        let name = self.name();
        if name.len() > 5 {
            format!("{}...", &name[..5])
        } else {
            name.to_string()
        }
    }
}

struct Drone {
    name: String,
    model: String,
}

impl Describable for Drone {
    fn name(&self) -> &str {
        &self.name
    }
    
    // Перевизначаємо describe
    fn describe(&self) -> String {
        format!("Дрон '{}' (модель: {})", self.name, self.model)
    }
    
    // short_name використовує реалізацію за замовчуванням
}

fn main() {
    let drone = Drone {
        name: String::from("Nighthawk"),
        model: String::from("X500"),
    };
    
    println!("{}", drone.describe());
    println!("Коротко: {}", drone.short_name());
}
```

### Trait з асоційованими типами

```rust
trait Container {
    type Item;  // Асоційований тип
    
    fn add(&mut self, item: Self::Item);
    fn get(&self, index: usize) -> Option<&Self::Item>;
    fn len(&self) -> usize;
}

struct DroneFleet {
    drones: Vec<String>,
}

impl Container for DroneFleet {
    type Item = String;  // Визначаємо конкретний тип
    
    fn add(&mut self, item: String) {
        self.drones.push(item);
    }
    
    fn get(&self, index: usize) -> Option<&String> {
        self.drones.get(index)
    }
    
    fn len(&self) -> usize {
        self.drones.len()
    }
}
```

### Множинна реалізація trait

```rust
trait Chargeable {
    fn charge(&mut self, amount: i32);
    fn battery_level(&self) -> i32;
}

struct Drone {
    battery: i32,
}

struct Robot {
    power: i32,
}

impl Chargeable for Drone {
    fn charge(&mut self, amount: i32) {
        self.battery = (self.battery + amount).min(100);
    }
    
    fn battery_level(&self) -> i32 {
        self.battery
    }
}

impl Chargeable for Robot {
    fn charge(&mut self, amount: i32) {
        self.power = (self.power + amount).min(200);  // Інший максимум
    }
    
    fn battery_level(&self) -> i32 {
        self.power
    }
}

// Функція працює з будь-яким Chargeable
fn charge_to_full(device: &mut impl Chargeable) {
    while device.battery_level() < 100 {
        device.charge(10);
    }
}
```

---

## Generics — параметричні типи

### Generic функції

```rust
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    
    largest
}

fn main() {
    let numbers = vec![34, 50, 25, 100, 65];
    println!("Найбільше число: {}", largest(&numbers));
    
    let chars = vec!['y', 'm', 'a', 'q'];
    println!("Найбільший символ: {}", largest(&chars));
}
```

### Generic структури

```rust
#[derive(Debug)]
struct Point<T> {
    x: T,
    y: T,
}

impl<T> Point<T> {
    fn new(x: T, y: T) -> Self {
        Point { x, y }
    }
}

// Методи тільки для Point<f64>
impl Point<f64> {
    fn distance_from_origin(&self) -> f64 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}

fn main() {
    let int_point = Point::new(5, 10);
    let float_point = Point::new(1.5, 2.5);
    
    println!("{:?}", int_point);
    println!("{:?}", float_point);
    println!("Відстань: {}", float_point.distance_from_origin());
}
```

### Кілька generic параметрів

```rust
#[derive(Debug)]
struct Pair<T, U> {
    first: T,
    second: U,
}

impl<T, U> Pair<T, U> {
    fn new(first: T, second: U) -> Self {
        Pair { first, second }
    }
    
    fn swap(self) -> Pair<U, T> {
        Pair {
            first: self.second,
            second: self.first,
        }
    }
}

fn main() {
    let pair = Pair::new("Alpha", 42);
    println!("{:?}", pair);
    
    let swapped = pair.swap();
    println!("{:?}", swapped);
}
```

### Generic enum

```rust
#[derive(Debug)]
enum Status<T, E> {
    Active(T),
    Error(E),
    Idle,
}

fn main() {
    let status1: Status<String, i32> = Status::Active(String::from("Flying"));
    let status2: Status<String, i32> = Status::Error(404);
    let status3: Status<String, i32> = Status::Idle;
    
    println!("{:?}", status1);
    println!("{:?}", status2);
    println!("{:?}", status3);
}
```

---

## Trait Bounds

### Синтаксис обмежень

```rust
// Варіант 1: у кутових дужках
fn print_info<T: Debug + Clone>(item: T) {
    println!("{:?}", item.clone());
}

// Варіант 2: impl Trait (коротше)
fn print_info2(item: impl Debug + Clone) {
    println!("{:?}", item.clone());
}

// Варіант 3: where-clause (для складних випадків)
fn print_info3<T>(item: T)
where
    T: Debug + Clone,
{
    println!("{:?}", item.clone());
}
```

### Множинні обмеження

```rust
use std::fmt::{Debug, Display};

fn complex_function<T, U>(t: T, u: U)
where
    T: Debug + Clone + PartialEq,
    U: Display + Default,
{
    println!("T: {:?}", t);
    println!("U: {}", u);
}
```

### Обмеження на impl блок

```rust
struct Container<T> {
    value: T,
}

// Методи для всіх T
impl<T> Container<T> {
    fn new(value: T) -> Self {
        Container { value }
    }
    
    fn get(&self) -> &T {
        &self.value
    }
}

// Методи тільки для T: Display
impl<T: std::fmt::Display> Container<T> {
    fn print(&self) {
        println!("Значення: {}", self.value);
    }
}

// Методи тільки для T: Clone + Default
impl<T: Clone + Default> Container<T> {
    fn reset(&mut self) {
        self.value = T::default();
    }
}
```

---

## Trait Objects (dyn Trait)

### Статичний vs динамічний поліморфізм

```rust
trait Movable {
    fn move_to(&mut self, x: i32, y: i32);
}

struct Drone { x: i32, y: i32 }
struct Robot { x: i32, y: i32 }

impl Movable for Drone {
    fn move_to(&mut self, x: i32, y: i32) {
        self.x = x;
        self.y = y;
        println!("Дрон летить до ({}, {})", x, y);
    }
}

impl Movable for Robot {
    fn move_to(&mut self, x: i32, y: i32) {
        self.x = x;
        self.y = y;
        println!("Робот їде до ({}, {})", x, y);
    }
}

// Статичний поліморфізм (generics) — визначається при компіляції
fn move_static<T: Movable>(item: &mut T, x: i32, y: i32) {
    item.move_to(x, y);
}

// Динамічний поліморфізм (trait objects) — визначається під час виконання
fn move_dynamic(item: &mut dyn Movable, x: i32, y: i32) {
    item.move_to(x, y);
}

fn main() {
    let mut drone = Drone { x: 0, y: 0 };
    let mut robot = Robot { x: 0, y: 0 };
    
    // Статичний поліморфізм
    move_static(&mut drone, 10, 20);
    move_static(&mut robot, 30, 40);
    
    // Динамічний поліморфізм
    let mut items: Vec<Box<dyn Movable>> = vec![
        Box::new(Drone { x: 0, y: 0 }),
        Box::new(Robot { x: 0, y: 0 }),
    ];
    
    for item in items.iter_mut() {
        item.move_to(100, 100);
    }
}
```

### Коли використовувати що?

| Статичний (generics) | Динамічний (dyn Trait) |
|---------------------|------------------------|
| Швидший (без vtable) | Повільніший (vtable lookup) |
| Код дублюється для кожного типу | Один код для всіх типів |
| Розмір відомий при компіляції | Потрібен Box/&dyn |
| Не можна зберігати різні типи разом | Можна зберігати різні типи |

---

## Типові помилки

### Помилка 1: Trait bound не виконується

```rust
fn print_it<T: Debug>(item: T) {
    println!("{:?}", item);
}

struct MyType;  // Не реалізує Debug!

fn main() {
    // print_it(MyType);  // ПОМИЛКА: MyType doesn't implement Debug
}

// Рішення: додати derive
#[derive(Debug)]
struct MyType2;
```

### Помилка 2: Конфлікт методів

```rust
trait A {
    fn method(&self);
}

trait B {
    fn method(&self);
}

struct MyStruct;

impl A for MyStruct {
    fn method(&self) { println!("A"); }
}

impl B for MyStruct {
    fn method(&self) { println!("B"); }
}

fn main() {
    let s = MyStruct;
    // s.method();  // ПОМИЛКА: ambiguous
    
    // Рішення: явно вказати trait
    A::method(&s);  // "A"
    B::method(&s);  // "B"
}
```

### Помилка 3: Copy для типу з heap-даними

```rust
// ПОМИЛКА: String не може бути Copy
// #[derive(Clone, Copy)]
// struct BadStruct {
//     name: String,  // String не Copy!
// }

// Рішення: тільки Clone
#[derive(Clone)]
struct GoodStruct {
    name: String,
}
```

### Помилка 4: Orphan rule

```rust
// ПОМИЛКА: не можна реалізувати чужий trait для чужого типу
// impl std::fmt::Display for Vec<i32> { ... }

// Рішення: обгорнути в свій тип
struct MyVec(Vec<i32>);

impl std::fmt::Display for MyVec {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "[{}]", self.0.iter()
            .map(|n| n.to_string())
            .collect::<Vec<_>>()
            .join(", "))
    }
}
```

---

## Практичні задачі

### Задача 1: Базові derive traits

**Умова:** Створіть структуру `Drone` з повним набором derive traits. Продемонструйте використання кожного: Debug, Clone, PartialEq, Eq, PartialOrd, Ord, Hash, Default.

**Розв'язання:**

```rust
use std::collections::{HashMap, HashSet};

#[derive(Debug, Clone, PartialEq, Eq, PartialOrd, Ord, Hash, Default)]
struct Drone {
    id: u32,
    name: String,
    priority: u8,
}

impl Drone {
    fn new(id: u32, name: &str, priority: u8) -> Self {
        Drone {
            id,
            name: String::from(name),
            priority,
        }
    }
}

fn main() {
    println!("=== Базові derive traits ===\n");
    
    // Default
    println!("--- Default ---");
    let default_drone = Drone::default();
    println!("Default: {:?}", default_drone);
    
    // Debug
    println!("\n--- Debug ---");
    let alpha = Drone::new(1, "Alpha", 5);
    println!("Debug: {:?}", alpha);
    println!("Pretty: {:#?}", alpha);
    
    // Clone
    println!("\n--- Clone ---");
    let alpha_clone = alpha.clone();
    println!("Original: {:?}", alpha);
    println!("Clone: {:?}", alpha_clone);
    
    // PartialEq / Eq
    println!("\n--- PartialEq / Eq ---");
    let alpha2 = Drone::new(1, "Alpha", 5);
    let beta = Drone::new(2, "Beta", 3);
    println!("alpha == alpha2: {}", alpha == alpha2);
    println!("alpha == beta: {}", alpha == beta);
    
    // PartialOrd / Ord
    println!("\n--- PartialOrd / Ord ---");
    let mut drones = vec![
        Drone::new(3, "Gamma", 1),
        Drone::new(1, "Alpha", 5),
        Drone::new(2, "Beta", 3),
    ];
    println!("До сортування: {:?}", drones);
    drones.sort();  // Сортує за (id, name, priority)
    println!("Після сортування: {:?}", drones);
    
    println!("alpha < beta: {}", alpha < beta);
    println!("alpha > beta: {}", alpha > beta);
    
    // Hash — для HashSet та HashMap
    println!("\n--- Hash ---");
    let mut seen: HashSet<Drone> = HashSet::new();
    seen.insert(Drone::new(1, "Alpha", 5));
    seen.insert(Drone::new(2, "Beta", 3));
    seen.insert(Drone::new(1, "Alpha", 5));  // Дублікат
    println!("Унікальних дронів: {}", seen.len());
    
    let mut drone_data: HashMap<Drone, String> = HashMap::new();
    drone_data.insert(Drone::new(1, "Alpha", 5), String::from("Активний"));
    drone_data.insert(Drone::new(2, "Beta", 3), String::from("Заряджається"));
    
    for (drone, status) in &drone_data {
        println!("  {:?} → {}", drone, status);
    }
}
```

**Результат:**
```
=== Базові derive traits ===

--- Default ---
Default: Drone { id: 0, name: "", priority: 0 }

--- Debug ---
Debug: Drone { id: 1, name: "Alpha", priority: 5 }
Pretty: Drone {
    id: 1,
    name: "Alpha",
    priority: 5,
}

--- Clone ---
Original: Drone { id: 1, name: "Alpha", priority: 5 }
Clone: Drone { id: 1, name: "Alpha", priority: 5 }

--- PartialEq / Eq ---
alpha == alpha2: true
alpha == beta: false

--- PartialOrd / Ord ---
До сортування: [Drone { id: 3, ... }, Drone { id: 1, ... }, Drone { id: 2, ... }]
Після сортування: [Drone { id: 1, ... }, Drone { id: 2, ... }, Drone { id: 3, ... }]
alpha < beta: true
...
```

---

### Задача 2: Власний trait — Agent

**Умова:** Створіть trait `Agent` з методами `perceive`, `decide`, `act`. Реалізуйте його для різних типів агентів: Drone, GroundRobot, Satellite.

**Розв'язання:**

```rust
use std::fmt;

// Спільний trait для всіх агентів
trait Agent {
    fn name(&self) -> &str;
    fn perceive(&self) -> Vec<String>;
    fn decide(&self, perceptions: &[String]) -> Option<String>;
    fn act(&mut self, action: &str) -> Result<(), String>;
    
    // Метод за замовчуванням — повний цикл
    fn cycle(&mut self) -> Result<(), String> {
        let perceptions = self.perceive();
        println!("[{}] Сприйняття: {:?}", self.name(), perceptions);
        
        if let Some(action) = self.decide(&perceptions) {
            println!("[{}] Рішення: {}", self.name(), action);
            self.act(&action)?;
            println!("[{}] Дію виконано", self.name());
        } else {
            println!("[{}] Немає дій", self.name());
        }
        
        Ok(())
    }
}

// Drone — літаючий агент
#[derive(Debug)]
struct Drone {
    name: String,
    altitude: i32,
    battery: i32,
}

impl Drone {
    fn new(name: &str) -> Self {
        Drone {
            name: String::from(name),
            altitude: 0,
            battery: 100,
        }
    }
}

impl Agent for Drone {
    fn name(&self) -> &str {
        &self.name
    }
    
    fn perceive(&self) -> Vec<String> {
        let mut perceptions = Vec::new();
        perceptions.push(format!("altitude:{}", self.altitude));
        perceptions.push(format!("battery:{}", self.battery));
        
        if self.battery < 20 {
            perceptions.push(String::from("low_battery"));
        }
        
        perceptions
    }
    
    fn decide(&self, perceptions: &[String]) -> Option<String> {
        if perceptions.contains(&String::from("low_battery")) {
            return Some(String::from("RETURN_BASE"));
        }
        
        if self.altitude == 0 {
            return Some(String::from("TAKE_OFF"));
        }
        
        Some(String::from("PATROL"))
    }
    
    fn act(&mut self, action: &str) -> Result<(), String> {
        match action {
            "TAKE_OFF" => {
                self.altitude = 50;
                self.battery -= 5;
                Ok(())
            },
            "PATROL" => {
                self.battery -= 2;
                Ok(())
            },
            "RETURN_BASE" => {
                self.altitude = 0;
                self.battery -= 3;
                Ok(())
            },
            _ => Err(format!("Невідома дія: {}", action)),
        }
    }
}

// GroundRobot — наземний агент
#[derive(Debug)]
struct GroundRobot {
    name: String,
    x: i32,
    y: i32,
    cargo: bool,
}

impl GroundRobot {
    fn new(name: &str) -> Self {
        GroundRobot {
            name: String::from(name),
            x: 0,
            y: 0,
            cargo: false,
        }
    }
}

impl Agent for GroundRobot {
    fn name(&self) -> &str {
        &self.name
    }
    
    fn perceive(&self) -> Vec<String> {
        let mut perceptions = Vec::new();
        perceptions.push(format!("position:({},{})", self.x, self.y));
        perceptions.push(format!("cargo:{}", self.cargo));
        perceptions
    }
    
    fn decide(&self, perceptions: &[String]) -> Option<String> {
        if perceptions.contains(&String::from("cargo:true")) {
            return Some(String::from("DELIVER"));
        }
        Some(String::from("PICKUP"))
    }
    
    fn act(&mut self, action: &str) -> Result<(), String> {
        match action {
            "PICKUP" => {
                self.cargo = true;
                Ok(())
            },
            "DELIVER" => {
                self.cargo = false;
                self.x += 10;
                self.y += 10;
                Ok(())
            },
            _ => Err(format!("Невідома дія: {}", action)),
        }
    }
}

// Satellite — космічний агент
#[derive(Debug)]
struct Satellite {
    name: String,
    orbit_position: u32,
    data_collected: u32,
}

impl Satellite {
    fn new(name: &str) -> Self {
        Satellite {
            name: String::from(name),
            orbit_position: 0,
            data_collected: 0,
        }
    }
}

impl Agent for Satellite {
    fn name(&self) -> &str {
        &self.name
    }
    
    fn perceive(&self) -> Vec<String> {
        vec![
            format!("orbit:{}", self.orbit_position),
            format!("data:{}", self.data_collected),
        ]
    }
    
    fn decide(&self, _perceptions: &[String]) -> Option<String> {
        if self.data_collected > 100 {
            Some(String::from("TRANSMIT"))
        } else {
            Some(String::from("SCAN"))
        }
    }
    
    fn act(&mut self, action: &str) -> Result<(), String> {
        match action {
            "SCAN" => {
                self.data_collected += 25;
                self.orbit_position = (self.orbit_position + 1) % 360;
                Ok(())
            },
            "TRANSMIT" => {
                self.data_collected = 0;
                Ok(())
            },
            _ => Err(format!("Невідома дія: {}", action)),
        }
    }
}

fn main() {
    println!("=== Власний trait Agent ===\n");
    
    // Створюємо агентів
    let mut drone = Drone::new("SkyEye");
    let mut robot = GroundRobot::new("Hauler");
    let mut satellite = Satellite::new("Observer-1");
    
    // Виконуємо цикли
    println!("--- Drone ---");
    drone.cycle().unwrap();
    drone.cycle().unwrap();
    println!("Стан: {:?}\n", drone);
    
    println!("--- GroundRobot ---");
    robot.cycle().unwrap();
    robot.cycle().unwrap();
    println!("Стан: {:?}\n", robot);
    
    println!("--- Satellite ---");
    for _ in 0..5 {
        satellite.cycle().unwrap();
    }
    println!("Стан: {:?}", satellite);
    
    // Динамічний поліморфізм
    println!("\n--- Всі агенти через dyn Agent ---");
    let agents: Vec<Box<dyn Agent>> = vec![
        Box::new(Drone::new("D1")),
        Box::new(GroundRobot::new("R1")),
        Box::new(Satellite::new("S1")),
    ];
    
    for agent in agents {
        println!("Агент: {}", agent.name());
    }
}
```

---

### Задача 3: Generic контейнер з trait bounds

**Умова:** Створіть generic структуру `Fleet<T>` для управління колекцією агентів. Використайте trait bounds для обмеження типу T.

**Розв'язання:**

```rust
use std::fmt::Debug;

// Trait для ідентифікації
trait Identifiable {
    fn id(&self) -> u32;
    fn name(&self) -> &str;
}

// Trait для статусу
trait HasStatus {
    fn is_active(&self) -> bool;
    fn battery_level(&self) -> i32;
}

// Generic Fleet з обмеженнями
struct Fleet<T>
where
    T: Identifiable + HasStatus + Debug,
{
    name: String,
    members: Vec<T>,
}

impl<T> Fleet<T>
where
    T: Identifiable + HasStatus + Debug,
{
    fn new(name: &str) -> Self {
        Fleet {
            name: String::from(name),
            members: Vec::new(),
        }
    }
    
    fn add(&mut self, member: T) {
        println!("Додано {} до флоту '{}'", member.name(), self.name);
        self.members.push(member);
    }
    
    fn remove(&mut self, id: u32) -> Option<T> {
        if let Some(pos) = self.members.iter().position(|m| m.id() == id) {
            Some(self.members.remove(pos))
        } else {
            None
        }
    }
    
    fn get(&self, id: u32) -> Option<&T> {
        self.members.iter().find(|m| m.id() == id)
    }
    
    fn count(&self) -> usize {
        self.members.len()
    }
    
    fn active_count(&self) -> usize {
        self.members.iter().filter(|m| m.is_active()).count()
    }
    
    fn average_battery(&self) -> f64 {
        if self.members.is_empty() {
            return 0.0;
        }
        let total: i32 = self.members.iter().map(|m| m.battery_level()).sum();
        total as f64 / self.members.len() as f64
    }
    
    fn low_battery(&self, threshold: i32) -> Vec<&T> {
        self.members
            .iter()
            .filter(|m| m.battery_level() < threshold)
            .collect()
    }
    
    fn list(&self) {
        println!("\n=== Флот '{}' ({} членів) ===", self.name, self.count());
        for member in &self.members {
            let status = if member.is_active() { "●" } else { "○" };
            println!(
                "  {} [{}] {} - батарея: {}%",
                status,
                member.id(),
                member.name(),
                member.battery_level()
            );
        }
        println!("Середня батарея: {:.1}%", self.average_battery());
    }
}

// Конкретний тип, що реалізує необхідні traits
#[derive(Debug)]
struct Drone {
    id: u32,
    name: String,
    battery: i32,
    active: bool,
}

impl Drone {
    fn new(id: u32, name: &str, battery: i32) -> Self {
        Drone {
            id,
            name: String::from(name),
            battery,
            active: true,
        }
    }
}

impl Identifiable for Drone {
    fn id(&self) -> u32 {
        self.id
    }
    
    fn name(&self) -> &str {
        &self.name
    }
}

impl HasStatus for Drone {
    fn is_active(&self) -> bool {
        self.active
    }
    
    fn battery_level(&self) -> i32 {
        self.battery
    }
}

fn main() {
    println!("=== Generic контейнер Fleet<T> ===\n");
    
    let mut fleet: Fleet<Drone> = Fleet::new("Alpha Squadron");
    
    // Додаємо дрони
    fleet.add(Drone::new(1, "Scout-1", 85));
    fleet.add(Drone::new(2, "Scout-2", 45));
    fleet.add(Drone::new(3, "Recon-1", 92));
    fleet.add(Drone::new(4, "Recon-2", 15));
    fleet.add(Drone::new(5, "Support", 67));
    
    fleet.list();
    
    // Пошук
    println!("\n--- Пошук ---");
    if let Some(drone) = fleet.get(3) {
        println!("Знайдено: {:?}", drone);
    }
    
    // Низький заряд
    println!("\n--- Низький заряд (<50%) ---");
    for drone in fleet.low_battery(50) {
        println!("  ⚠ {} - {}%", drone.name(), drone.battery_level());
    }
    
    // Статистика
    println!("\n--- Статистика ---");
    println!("Всього: {}", fleet.count());
    println!("Активних: {}", fleet.active_count());
    println!("Середня батарея: {:.1}%", fleet.average_battery());
}
```

---

### Задача 4: Комбінування traits та generics

**Умова:** Створіть систему команд з generic обробником. Команди мають різні типи параметрів та результатів. Використайте асоційовані типи.

**Розв'язання:**

```rust
use std::fmt::Debug;

// Trait для команд з асоційованими типами
trait Command {
    type Params;
    type Result;
    type Error;
    
    fn name(&self) -> &str;
    fn validate(&self, params: &Self::Params) -> Result<(), Self::Error>;
    fn execute(&self, params: Self::Params) -> Result<Self::Result, Self::Error>;
}

// Помилки команд
#[derive(Debug)]
enum CommandError {
    InvalidParams(String),
    ExecutionFailed(String),
}

// Команда Move
struct MoveCommand;

#[derive(Debug)]
struct MoveParams {
    x: i32,
    y: i32,
    speed: f32,
}

#[derive(Debug)]
struct MoveResult {
    distance: f64,
    time_seconds: f64,
}

impl Command for MoveCommand {
    type Params = MoveParams;
    type Result = MoveResult;
    type Error = CommandError;
    
    fn name(&self) -> &str {
        "MOVE"
    }
    
    fn validate(&self, params: &Self::Params) -> Result<(), Self::Error> {
        if params.speed <= 0.0 {
            return Err(CommandError::InvalidParams(
                String::from("Швидкість має бути додатною")
            ));
        }
        if params.x.abs() > 1000 || params.y.abs() > 1000 {
            return Err(CommandError::InvalidParams(
                String::from("Координати поза межами")
            ));
        }
        Ok(())
    }
    
    fn execute(&self, params: Self::Params) -> Result<Self::Result, Self::Error> {
        self.validate(&params)?;
        
        let distance = ((params.x.pow(2) + params.y.pow(2)) as f64).sqrt();
        let time = distance / params.speed as f64;
        
        Ok(MoveResult {
            distance,
            time_seconds: time,
        })
    }
}

// Команда Scan
struct ScanCommand;

#[derive(Debug)]
struct ScanParams {
    radius: u32,
    resolution: String,
}

#[derive(Debug)]
struct ScanResult {
    objects_found: u32,
    area_covered: f64,
}

impl Command for ScanCommand {
    type Params = ScanParams;
    type Result = ScanResult;
    type Error = CommandError;
    
    fn name(&self) -> &str {
        "SCAN"
    }
    
    fn validate(&self, params: &Self::Params) -> Result<(), Self::Error> {
        if params.radius == 0 || params.radius > 500 {
            return Err(CommandError::InvalidParams(
                String::from("Радіус має бути 1-500")
            ));
        }
        let valid_resolutions = ["LOW", "MEDIUM", "HIGH"];
        if !valid_resolutions.contains(&params.resolution.as_str()) {
            return Err(CommandError::InvalidParams(
                format!("Невідома роздільність: {}", params.resolution)
            ));
        }
        Ok(())
    }
    
    fn execute(&self, params: Self::Params) -> Result<Self::Result, Self::Error> {
        self.validate(&params)?;
        
        let area = std::f64::consts::PI * (params.radius as f64).powi(2);
        let objects = (params.radius / 10) + 1;  // Симуляція
        
        Ok(ScanResult {
            objects_found: objects,
            area_covered: area,
        })
    }
}

// Generic обробник команд
fn execute_and_log<C>(command: &C, params: C::Params) 
where
    C: Command,
    C::Params: Debug,
    C::Result: Debug,
    C::Error: Debug,
{
    println!("\n=== Виконання команди '{}' ===", command.name());
    println!("Параметри: {:?}", params);
    
    match command.execute(params) {
        Ok(result) => {
            println!("✓ Успіх: {:?}", result);
        },
        Err(error) => {
            println!("✗ Помилка: {:?}", error);
        }
    }
}

// Черга команд з trait objects
trait AnyCommand {
    fn name(&self) -> &str;
    fn execute_boxed(&self) -> Result<String, String>;
}

struct QueuedMove {
    params: MoveParams,
}

impl AnyCommand for QueuedMove {
    fn name(&self) -> &str {
        "MOVE"
    }
    
    fn execute_boxed(&self) -> Result<String, String> {
        let cmd = MoveCommand;
        match cmd.execute(MoveParams { 
            x: self.params.x, 
            y: self.params.y, 
            speed: self.params.speed 
        }) {
            Ok(r) => Ok(format!("Переміщення: {:.1}м за {:.1}с", r.distance, r.time_seconds)),
            Err(e) => Err(format!("{:?}", e)),
        }
    }
}

struct QueuedScan {
    params: ScanParams,
}

impl AnyCommand for QueuedScan {
    fn name(&self) -> &str {
        "SCAN"
    }
    
    fn execute_boxed(&self) -> Result<String, String> {
        let cmd = ScanCommand;
        match cmd.execute(ScanParams { 
            radius: self.params.radius, 
            resolution: self.params.resolution.clone()
        }) {
            Ok(r) => Ok(format!("Скановано: {:.0}м², знайдено {} об'єктів", r.area_covered, r.objects_found)),
            Err(e) => Err(format!("{:?}", e)),
        }
    }
}

fn main() {
    println!("=== Система команд з generics ===");
    
    let move_cmd = MoveCommand;
    let scan_cmd = ScanCommand;
    
    // Успішні команди
    execute_and_log(&move_cmd, MoveParams { x: 100, y: 50, speed: 10.0 });
    execute_and_log(&scan_cmd, ScanParams { radius: 200, resolution: String::from("HIGH") });
    
    // Невалідні команди
    execute_and_log(&move_cmd, MoveParams { x: 100, y: 50, speed: -5.0 });
    execute_and_log(&scan_cmd, ScanParams { radius: 0, resolution: String::from("ULTRA") });
    
    // Черга різнорідних команд
    println!("\n\n=== Черга команд (dyn AnyCommand) ===");
    let queue: Vec<Box<dyn AnyCommand>> = vec![
        Box::new(QueuedMove { params: MoveParams { x: 10, y: 20, speed: 5.0 } }),
        Box::new(QueuedScan { params: ScanParams { radius: 50, resolution: String::from("MEDIUM") } }),
        Box::new(QueuedMove { params: MoveParams { x: 200, y: 100, speed: 15.0 } }),
    ];
    
    for (i, cmd) in queue.iter().enumerate() {
        println!("\n[{}] {}", i + 1, cmd.name());
        match cmd.execute_boxed() {
            Ok(msg) => println!("  ✓ {}", msg),
            Err(msg) => println!("  ✗ {}", msg),
        }
    }
}
```

---

## Домашнє завдання

### Завдання 1: Власна реалізація Display та Debug

**Умова:** Створіть структуру `Mission` з полями: id, name, waypoints, status. Реалізуйте власні Display (гарний вивід для користувача) та Debug (детальний вивід для розробника).

**Розв'язання:**

```rust
use std::fmt;

#[derive(Clone)]
struct Waypoint {
    x: i32,
    y: i32,
    name: String,
}

enum MissionStatus {
    Pending,
    Active,
    Completed,
    Failed(String),
}

struct Mission {
    id: u32,
    name: String,
    waypoints: Vec<Waypoint>,
    status: MissionStatus,
    progress: f32,
}

impl fmt::Debug for Waypoint {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "WP{{{}:({},{})}}", self.name, self.x, self.y)
    }
}

impl fmt::Display for Waypoint {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{} ({}, {})", self.name, self.x, self.y)
    }
}

impl fmt::Debug for MissionStatus {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            MissionStatus::Pending => write!(f, "Pending"),
            MissionStatus::Active => write!(f, "Active"),
            MissionStatus::Completed => write!(f, "Completed"),
            MissionStatus::Failed(reason) => write!(f, "Failed({:?})", reason),
        }
    }
}

impl fmt::Display for MissionStatus {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            MissionStatus::Pending => write!(f, "⏳ Очікує"),
            MissionStatus::Active => write!(f, "▶️ Активна"),
            MissionStatus::Completed => write!(f, "✅ Завершена"),
            MissionStatus::Failed(reason) => write!(f, "❌ Провалена: {}", reason),
        }
    }
}

impl fmt::Debug for Mission {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_struct("Mission")
            .field("id", &self.id)
            .field("name", &self.name)
            .field("status", &self.status)
            .field("progress", &format!("{:.1}%", self.progress))
            .field("waypoints", &self.waypoints)
            .finish()
    }
}

impl fmt::Display for Mission {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        writeln!(f, "╔══════════════════════════════════════╗")?;
        writeln!(f, "║ Місія #{}: {:<24} ║", self.id, self.name)?;
        writeln!(f, "╠══════════════════════════════════════╣")?;
        writeln!(f, "║ Статус: {:<28} ║", self.status)?;
        writeln!(f, "║ Прогрес: {:<27} ║", format!("{:.1}%", self.progress))?;
        writeln!(f, "╠══════════════════════════════════════╣")?;
        writeln!(f, "║ Маршрут ({} точок):                   ║", self.waypoints.len())?;
        
        for (i, wp) in self.waypoints.iter().enumerate() {
            let marker = if (i as f32) < (self.waypoints.len() as f32 * self.progress / 100.0) {
                "✓"
            } else {
                "○"
            };
            writeln!(f, "║   {} {}. {:<28} ║", marker, i + 1, wp)?;
        }
        
        write!(f, "╚══════════════════════════════════════╝")
    }
}

fn main() {
    println!("=== Display та Debug ===\n");
    
    let mission = Mission {
        id: 42,
        name: String::from("Patrol Alpha"),
        waypoints: vec![
            Waypoint { x: 0, y: 0, name: String::from("Base") },
            Waypoint { x: 100, y: 50, name: String::from("Checkpoint A") },
            Waypoint { x: 200, y: 100, name: String::from("Target Zone") },
            Waypoint { x: 0, y: 0, name: String::from("Return") },
        ],
        status: MissionStatus::Active,
        progress: 45.0,
    };
    
    println!("--- Display (для користувача) ---");
    println!("{}", mission);
    
    println!("\n--- Debug (для розробника) ---");
    println!("{:?}", mission);
    
    println!("\n--- Pretty Debug ---");
    println!("{:#?}", mission);
}
```

---

### Завдання 2: Trait з generic методами

**Умова:** Створіть trait `Sensor` з generic методом `read<T>()`, що повертає різні типи даних залежно від сенсора. Реалізуйте для GPS, Temperature, Camera.

**Розв'язання:**

```rust
use std::fmt::Debug;

// Generic trait для сенсорів
trait Sensor {
    type Reading: Debug;
    
    fn name(&self) -> &str;
    fn read(&self) -> Result<Self::Reading, String>;
    fn is_available(&self) -> bool;
}

// GPS сенсор
#[derive(Debug)]
struct GpsReading {
    latitude: f64,
    longitude: f64,
    altitude: f64,
    accuracy: f32,
}

struct GpsSensor {
    name: String,
    available: bool,
}

impl Sensor for GpsSensor {
    type Reading = GpsReading;
    
    fn name(&self) -> &str {
        &self.name
    }
    
    fn read(&self) -> Result<Self::Reading, String> {
        if !self.available {
            return Err(String::from("GPS сигнал недоступний"));
        }
        
        Ok(GpsReading {
            latitude: 50.4501,
            longitude: 30.5234,
            altitude: 150.0,
            accuracy: 2.5,
        })
    }
    
    fn is_available(&self) -> bool {
        self.available
    }
}

// Температурний сенсор
#[derive(Debug)]
struct TemperatureReading {
    celsius: f32,
    humidity: f32,
}

struct TemperatureSensor {
    name: String,
}

impl Sensor for TemperatureSensor {
    type Reading = TemperatureReading;
    
    fn name(&self) -> &str {
        &self.name
    }
    
    fn read(&self) -> Result<Self::Reading, String> {
        Ok(TemperatureReading {
            celsius: 22.5,
            humidity: 65.0,
        })
    }
    
    fn is_available(&self) -> bool {
        true
    }
}

// Камера
#[derive(Debug)]
struct ImageData {
    width: u32,
    height: u32,
    format: String,
    size_bytes: usize,
}

struct CameraSensor {
    name: String,
    resolution: (u32, u32),
}

impl Sensor for CameraSensor {
    type Reading = ImageData;
    
    fn name(&self) -> &str {
        &self.name
    }
    
    fn read(&self) -> Result<Self::Reading, String> {
        Ok(ImageData {
            width: self.resolution.0,
            height: self.resolution.1,
            format: String::from("JPEG"),
            size_bytes: (self.resolution.0 * self.resolution.1 * 3) as usize,
        })
    }
    
    fn is_available(&self) -> bool {
        true
    }
}

// Generic функція для читання сенсора
fn read_sensor<S: Sensor>(sensor: &S) {
    println!("\n--- {} ---", sensor.name());
    println!("Доступний: {}", sensor.is_available());
    
    match sensor.read() {
        Ok(reading) => println!("Читання: {:?}", reading),
        Err(e) => println!("Помилка: {}", e),
    }
}

fn main() {
    println!("=== Trait з generic методами ===");
    
    let gps = GpsSensor {
        name: String::from("GPS-001"),
        available: true,
    };
    
    let gps_offline = GpsSensor {
        name: String::from("GPS-002"),
        available: false,
    };
    
    let temp = TemperatureSensor {
        name: String::from("Temp-001"),
    };
    
    let camera = CameraSensor {
        name: String::from("Camera-001"),
        resolution: (1920, 1080),
    };
    
    read_sensor(&gps);
    read_sensor(&gps_offline);
    read_sensor(&temp);
    read_sensor(&camera);
}
```

---

### Завдання 3: Trait bounds з where

**Умова:** Створіть generic функцію `process_batch`, яка обробляє колекцію елементів. Елементи мають підтримувати Clone, Debug та власний trait `Processable`. Використайте where-clause.

**Розв'язання:**

```rust
use std::fmt::Debug;

trait Processable {
    fn process(&self) -> Result<String, String>;
    fn priority(&self) -> u8;
}

#[derive(Debug, Clone)]
struct Task {
    id: u32,
    name: String,
    priority: u8,
}

impl Processable for Task {
    fn process(&self) -> Result<String, String> {
        if self.name.is_empty() {
            Err(String::from("Порожня назва"))
        } else {
            Ok(format!("Задача '{}' оброблена", self.name))
        }
    }
    
    fn priority(&self) -> u8 {
        self.priority
    }
}

#[derive(Debug)]
struct BatchResult<T> {
    successful: Vec<(T, String)>,
    failed: Vec<(T, String)>,
}

fn process_batch<T>(items: Vec<T>) -> BatchResult<T>
where
    T: Processable + Clone + Debug,
{
    let mut result = BatchResult {
        successful: Vec::new(),
        failed: Vec::new(),
    };
    
    // Сортуємо за пріоритетом
    let mut sorted = items;
    sorted.sort_by_key(|item| std::cmp::Reverse(item.priority()));
    
    for item in sorted {
        println!("Обробка: {:?}", item);
        
        match item.process() {
            Ok(msg) => {
                println!("  ✓ {}", msg);
                result.successful.push((item, msg));
            },
            Err(err) => {
                println!("  ✗ {}", err);
                result.failed.push((item, err));
            }
        }
    }
    
    result
}

fn summarize_batch<T>(result: &BatchResult<T>)
where
    T: Debug,
{
    let total = result.successful.len() + result.failed.len();
    let success_rate = if total > 0 {
        (result.successful.len() as f64 / total as f64) * 100.0
    } else {
        0.0
    };
    
    println!("\n=== Підсумок ===");
    println!("Всього: {}", total);
    println!("Успішно: {}", result.successful.len());
    println!("Провалено: {}", result.failed.len());
    println!("Успішність: {:.1}%", success_rate);
}

fn main() {
    println!("=== Trait bounds з where ===\n");
    
    let tasks = vec![
        Task { id: 1, name: String::from("Сканування"), priority: 5 },
        Task { id: 2, name: String::from(""), priority: 10 },  // Провалиться
        Task { id: 3, name: String::from("Патрулювання"), priority: 3 },
        Task { id: 4, name: String::from("Звіт"), priority: 8 },
        Task { id: 5, name: String::from(""), priority: 1 },  // Провалиться
    ];
    
    let result = process_batch(tasks);
    summarize_batch(&result);
}
```

---

### Завдання 4: Суперtrait та композиція

**Умова:** Створіть ієрархію traits: `Entity` (базовий) → `Movable` (рухомий) → `Controllable` (керований). Реалізуйте для Drone, що має підтримувати всі рівні.

**Розв'язання:**

```rust
use std::fmt::Debug;

// Базовий trait
trait Entity: Debug {
    fn id(&self) -> u32;
    fn name(&self) -> &str;
    fn entity_type(&self) -> &str;
}

// Supertrait — вимагає Entity
trait Movable: Entity {
    fn position(&self) -> (f64, f64);
    fn move_to(&mut self, x: f64, y: f64);
    fn speed(&self) -> f64;
    
    fn distance_to(&self, x: f64, y: f64) -> f64 {
        let (px, py) = self.position();
        ((x - px).powi(2) + (y - py).powi(2)).sqrt()
    }
}

// Supertrait — вимагає Movable (а отже і Entity)
trait Controllable: Movable {
    fn is_active(&self) -> bool;
    fn activate(&mut self);
    fn deactivate(&mut self);
    fn execute_command(&mut self, cmd: &str) -> Result<String, String>;
}

// Повна реалізація для Drone
#[derive(Debug)]
struct Drone {
    id: u32,
    name: String,
    x: f64,
    y: f64,
    speed: f64,
    active: bool,
    battery: i32,
}

impl Drone {
    fn new(id: u32, name: &str) -> Self {
        Drone {
            id,
            name: String::from(name),
            x: 0.0,
            y: 0.0,
            speed: 10.0,
            active: false,
            battery: 100,
        }
    }
}

impl Entity for Drone {
    fn id(&self) -> u32 {
        self.id
    }
    
    fn name(&self) -> &str {
        &self.name
    }
    
    fn entity_type(&self) -> &str {
        "Drone"
    }
}

impl Movable for Drone {
    fn position(&self) -> (f64, f64) {
        (self.x, self.y)
    }
    
    fn move_to(&mut self, x: f64, y: f64) {
        let dist = self.distance_to(x, y);
        self.battery -= (dist / 10.0) as i32;
        self.x = x;
        self.y = y;
    }
    
    fn speed(&self) -> f64 {
        self.speed
    }
}

impl Controllable for Drone {
    fn is_active(&self) -> bool {
        self.active
    }
    
    fn activate(&mut self) {
        self.active = true;
        println!("[{}] Активовано", self.name);
    }
    
    fn deactivate(&mut self) {
        self.active = false;
        println!("[{}] Деактивовано", self.name);
    }
    
    fn execute_command(&mut self, cmd: &str) -> Result<String, String> {
        if !self.active {
            return Err(String::from("Дрон неактивний"));
        }
        
        match cmd {
            "STATUS" => Ok(format!(
                "Позиція: ({:.1}, {:.1}), Батарея: {}%",
                self.x, self.y, self.battery
            )),
            "HOVER" => Ok(String::from("Зависання...")),
            _ => Err(format!("Невідома команда: {}", cmd)),
        }
    }
}

// Функції, що працюють з різними рівнями абстракції
fn print_entity<E: Entity>(e: &E) {
    println!("{} [{}]: {}", e.entity_type(), e.id(), e.name());
}

fn move_entity<M: Movable>(m: &mut M, x: f64, y: f64) {
    let dist = m.distance_to(x, y);
    println!("Переміщення {} на {:.1}м до ({:.1}, {:.1})", m.name(), dist, x, y);
    m.move_to(x, y);
}

fn control<C: Controllable>(c: &mut C, commands: &[&str]) {
    println!("\n=== Керування {} ===", c.name());
    c.activate();
    
    for cmd in commands {
        match c.execute_command(cmd) {
            Ok(result) => println!("  [{}] ✓ {}", cmd, result),
            Err(err) => println!("  [{}] ✗ {}", cmd, err),
        }
    }
    
    c.deactivate();
}

fn main() {
    println!("=== Supertrait ієрархія ===\n");
    
    let mut drone = Drone::new(1, "Alpha");
    
    // Використання на різних рівнях абстракції
    println!("--- Entity рівень ---");
    print_entity(&drone);
    
    println!("\n--- Movable рівень ---");
    move_entity(&mut drone, 100.0, 50.0);
    move_entity(&mut drone, 200.0, 150.0);
    
    println!("\n--- Controllable рівень ---");
    control(&mut drone, &["STATUS", "HOVER", "UNKNOWN"]);
    
    println!("\n--- Фінальний стан ---");
    println!("{:#?}", drone);
}
```

---

## Підсумок заняття

На цьому занятті ви навчились:

1. **Використовувати стандартні traits**: Debug, Clone, Copy, PartialEq, Ord, Hash, Default
2. **Створювати власні traits**: методи, методи за замовчуванням, асоційовані типи
3. **Писати generic код**: функції, структури, enum з параметрами типів
4. **Застосовувати trait bounds**: обмеження на generic типи
5. **Працювати з trait objects**: dyn Trait для динамічного поліморфізму

---

## Перевірте себе

1. Чим trait відрізняється від інтерфейсу в Java?
2. Коли можна використати `#[derive(Copy)]`?
3. Яка різниця між `impl Trait` та `dyn Trait`?
4. Навіщо потрібні trait bounds?
5. Що таке associated types?
6. Як викликати метод, якщо є конфлікт імен між traits?

**Відповіді:**
1. Traits можуть мати методи за замовчуванням, associated types; orphan rule обмежує реалізацію
2. Коли всі поля типу також Copy (примітиви, не heap-дані)
3. `impl Trait` — статичний поліморфізм (monomorphization); `dyn Trait` — динамічний (vtable)
4. Щоб обмежити, які типи можуть використовуватись з generic кодом
5. Типи, визначені всередині trait, що конкретизуються при реалізації
6. `TraitName::method(&self)` — явно вказати trait

---

## Наступне заняття

На наступному занятті ми вивчимо **Lifetime annotations** — як явно вказувати час життя посилань у складних випадках, коли компілятор не може вивести його автоматично.
