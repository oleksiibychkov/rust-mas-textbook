# Практичне заняття 9: Колекції (Vec, HashMap, String)

## Мета заняття

Після цього заняття ви зможете:
- Створювати та використовувати динамічні вектори `Vec<T>`
- Працювати з асоціативними масивами `HashMap<K, V>`
- Ефективно маніпулювати рядками `String`
- Обирати правильну колекцію для конкретної задачі
- Ітерувати по колекціях різними способами

---

## Теоретичний вступ

### Чому колекції важливі?

Масиви мають фіксований розмір — це незручно для динамічних даних. Колекції вирішують цю проблему:

| Колекція | Призначення | Аналогія |
|----------|-------------|----------|
| `Vec<T>` | Динамічний список | Журнал польотів |
| `HashMap<K, V>` | Словник ключ-значення | Реєстр дронів за ID |
| `String` | Динамічний рядок | Назва місії |

У контексті дронів:
- `Vec<Position>` — маршрут з довільною кількістю точок
- `HashMap<String, Drone>` — флот дронів за іменами
- `String` — динамічні повідомлення та логи

---

## Vec<T> — Динамічний вектор

### Створення вектора

```rust
fn main() {
    // Порожній вектор з явним типом
    let mut numbers: Vec<i32> = Vec::new();
    
    // Макрос vec! — зручніше
    let values = vec![1, 2, 3, 4, 5];
    
    // Вектор з повторюваних значень
    let zeros = vec![0; 10];  // [0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
    
    // З ємністю (оптимізація)
    let mut with_capacity: Vec<i32> = Vec::with_capacity(100);
    
    println!("values: {:?}", values);
    println!("zeros: {:?}", zeros);
}
```

### Додавання елементів

```rust
fn main() {
    let mut drones = Vec::new();
    
    // push — додати в кінець
    drones.push("Alpha");
    drones.push("Beta");
    drones.push("Gamma");
    
    println!("Дрони: {:?}", drones);  // ["Alpha", "Beta", "Gamma"]
    
    // insert — вставити за індексом
    drones.insert(1, "Delta");  // Вставляємо на позицію 1
    println!("Після insert: {:?}", drones);  // ["Alpha", "Delta", "Beta", "Gamma"]
    
    // extend — додати кілька елементів
    drones.extend(["Epsilon", "Zeta"]);
    println!("Після extend: {:?}", drones);
}
```

### Видалення елементів

```rust
fn main() {
    let mut items = vec![1, 2, 3, 4, 5];
    
    // pop — видалити останній, повертає Option
    let last = items.pop();  // Some(5)
    println!("Видалено: {:?}, залишилось: {:?}", last, items);
    
    // remove — видалити за індексом
    let removed = items.remove(1);  // Видаляє елемент з індексом 1
    println!("Видалено: {}, залишилось: {:?}", removed, items);
    
    // retain — залишити тільки елементи, що відповідають умові
    items.retain(|&x| x > 2);
    println!("Після retain (>2): {:?}", items);
    
    // clear — очистити все
    items.clear();
    println!("Після clear: {:?}", items);
}
```

### Доступ до елементів

```rust
fn main() {
    let readings = vec![25.5, 26.0, 24.8, 25.2];
    
    // Індексація — паніка при виході за межі!
    let first = readings[0];
    println!("Перший: {}", first);
    
    // get — безпечний доступ, повертає Option
    match readings.get(10) {
        Some(value) => println!("Значення: {}", value),
        None => println!("Індекс за межами"),
    }
    
    // first / last
    if let Some(f) = readings.first() {
        println!("Перший: {}", f);
    }
    if let Some(l) = readings.last() {
        println!("Останній: {}", l);
    }
    
    // Слайси
    let middle = &readings[1..3];
    println!("Середина: {:?}", middle);
}
```

### Ітерація по вектору

```rust
fn main() {
    let drones = vec!["Alpha", "Beta", "Gamma"];
    
    // Іммутабельна ітерація
    for drone in &drones {
        println!("Дрон: {}", drone);
    }
    
    // Мутабельна ітерація
    let mut batteries = vec![50, 75, 30];
    for battery in &mut batteries {
        *battery += 10;  // Збільшуємо кожен на 10
    }
    println!("Після зарядки: {:?}", batteries);
    
    // З індексом
    for (i, drone) in drones.iter().enumerate() {
        println!("{}. {}", i + 1, drone);
    }
    
    // Споживаюча ітерація (into_iter)
    for drone in drones {  // drones переміщується!
        println!("Обробка: {}", drone);
    }
    // drones більше не доступний
}
```

### Корисні методи Vec

```rust
fn main() {
    let mut v = vec![3, 1, 4, 1, 5, 9, 2, 6];
    
    // Довжина та ємність
    println!("len: {}, capacity: {}", v.len(), v.capacity());
    
    // Перевірки
    println!("is_empty: {}", v.is_empty());
    println!("contains 4: {}", v.contains(&4));
    
    // Сортування
    v.sort();
    println!("Sorted: {:?}", v);
    
    // Реверс
    v.reverse();
    println!("Reversed: {:?}", v);
    
    // Dedupe (видаляє послідовні дублікати, потрібен sorted)
    let mut duplicates = vec![1, 1, 2, 2, 2, 3, 3];
    duplicates.dedup();
    println!("Deduped: {:?}", duplicates);
    
    // Пошук
    let numbers = vec![10, 20, 30, 40, 50];
    if let Some(pos) = numbers.iter().position(|&x| x == 30) {
        println!("30 знайдено на позиції {}", pos);
    }
}
```

---

## HashMap<K, V> — Асоціативний масив

### Створення HashMap

```rust
use std::collections::HashMap;

fn main() {
    // Порожній HashMap
    let mut scores: HashMap<String, i32> = HashMap::new();
    
    // Вставка
    scores.insert(String::from("Alpha"), 100);
    scores.insert(String::from("Beta"), 85);
    scores.insert(String::from("Gamma"), 92);
    
    println!("{:?}", scores);
    
    // Створення з ітератора
    let keys = vec!["A", "B", "C"];
    let values = vec![1, 2, 3];
    let map: HashMap<_, _> = keys.into_iter().zip(values).collect();
    println!("{:?}", map);
}
```

### Доступ до значень

```rust
use std::collections::HashMap;

fn main() {
    let mut drones: HashMap<String, i32> = HashMap::new();
    drones.insert(String::from("Alpha"), 85);
    drones.insert(String::from("Beta"), 60);
    
    // get — повертає Option<&V>
    match drones.get("Alpha") {
        Some(battery) => println!("Alpha: {}%", battery),
        None => println!("Alpha не знайдено"),
    }
    
    // get з unwrap_or
    let gamma_battery = drones.get("Gamma").unwrap_or(&0);
    println!("Gamma: {}%", gamma_battery);
    
    // Індексація — паніка якщо ключа немає!
    // let value = drones["Unknown"];  // ПАНІКА!
    
    // contains_key
    if drones.contains_key("Beta") {
        println!("Beta існує");
    }
}
```

### Оновлення значень

```rust
use std::collections::HashMap;

fn main() {
    let mut scores: HashMap<String, i32> = HashMap::new();
    
    // Просте перезаписування
    scores.insert(String::from("Alpha"), 100);
    scores.insert(String::from("Alpha"), 150);  // Перезаписує!
    
    // entry — вставити тільки якщо ключа немає
    scores.entry(String::from("Beta")).or_insert(50);
    scores.entry(String::from("Beta")).or_insert(999);  // Не змінить!
    
    println!("{:?}", scores);  // Alpha: 150, Beta: 50
    
    // entry з модифікацією
    let mut word_count: HashMap<String, i32> = HashMap::new();
    let text = "hello world hello rust hello";
    
    for word in text.split_whitespace() {
        let count = word_count.entry(String::from(word)).or_insert(0);
        *count += 1;
    }
    
    println!("Слова: {:?}", word_count);
}
```

### Видалення та ітерація

```rust
use std::collections::HashMap;

fn main() {
    let mut map: HashMap<String, i32> = HashMap::new();
    map.insert(String::from("A"), 1);
    map.insert(String::from("B"), 2);
    map.insert(String::from("C"), 3);
    
    // Видалення
    let removed = map.remove("B");
    println!("Видалено: {:?}", removed);  // Some(2)
    
    // Ітерація по ключах
    for key in map.keys() {
        println!("Ключ: {}", key);
    }
    
    // Ітерація по значеннях
    for value in map.values() {
        println!("Значення: {}", value);
    }
    
    // Ітерація по парах
    for (key, value) in &map {
        println!("{} -> {}", key, value);
    }
    
    // Мутабельна ітерація по значеннях
    for value in map.values_mut() {
        *value *= 10;
    }
    println!("Після множення: {:?}", map);
}
```

### Entry API — потужний патерн

```rust
use std::collections::HashMap;

fn main() {
    let mut fleet: HashMap<String, Vec<String>> = HashMap::new();
    
    // or_insert_with — ліниве створення
    fleet.entry(String::from("Squad-A"))
        .or_insert_with(Vec::new)
        .push(String::from("Alpha"));
    
    fleet.entry(String::from("Squad-A"))
        .or_insert_with(Vec::new)
        .push(String::from("Beta"));
    
    fleet.entry(String::from("Squad-B"))
        .or_insert_with(Vec::new)
        .push(String::from("Gamma"));
    
    println!("{:#?}", fleet);
    
    // and_modify + or_insert
    let mut counters: HashMap<String, i32> = HashMap::new();
    
    for event in ["click", "scroll", "click", "click", "scroll"] {
        counters.entry(String::from(event))
            .and_modify(|c| *c += 1)
            .or_insert(1);
    }
    
    println!("Лічильники: {:?}", counters);
}
```

---

## String — Динамічний рядок

### String vs &str

```rust
fn main() {
    // &str — рядковий слайс (незмінний, зазвичай літерал)
    let literal: &str = "Hello";
    
    // String — володіючий, динамічний рядок
    let owned: String = String::from("Hello");
    
    // Перетворення
    let s1: String = literal.to_string();
    let s2: String = String::from(literal);
    let s3: &str = &owned;  // Deref coercion
    
    println!("literal: {}, owned: {}", literal, owned);
}
```

### Створення String

```rust
fn main() {
    // Різні способи
    let s1 = String::new();                    // Порожній
    let s2 = String::from("Hello");            // З літерала
    let s3 = "World".to_string();              // to_string()
    let s4 = String::with_capacity(100);       // З ємністю
    let s5 = format!("{} {}", s2, s3);         // format!
    
    println!("s5: {}", s5);  // "Hello World"
}
```

### Модифікація String

```rust
fn main() {
    let mut s = String::from("Hello");
    
    // push_str — додати рядок
    s.push_str(", ");
    s.push_str("World");
    
    // push — додати символ
    s.push('!');
    
    println!("{}", s);  // "Hello, World!"
    
    // Конкатенація через +
    let s1 = String::from("Hello");
    let s2 = String::from("World");
    let s3 = s1 + " " + &s2;  // s1 переміщено!
    // println!("{}", s1);  // ПОМИЛКА! s1 переміщено
    println!("{}", s3);  // "Hello World"
    
    // format! — не переміщує
    let a = String::from("A");
    let b = String::from("B");
    let c = String::from("C");
    let result = format!("{}-{}-{}", a, b, c);
    println!("{}, {}, {}, {}", a, b, c, result);  // Всі доступні!
}
```

### Доступ до символів

```rust
fn main() {
    let s = String::from("Привіт");  // UTF-8!
    
    // НЕ можна індексувати напряму!
    // let c = s[0];  // ПОМИЛКА!
    
    // Ітерація по символах
    for c in s.chars() {
        println!("Символ: {}", c);
    }
    
    // Ітерація по байтах
    for b in s.bytes() {
        println!("Байт: {}", b);
    }
    
    // Отримати n-й символ
    if let Some(c) = s.chars().nth(2) {
        println!("3-й символ: {}", c);
    }
    
    // Слайс (обережно з UTF-8!)
    let hello = &s[0..12];  // "Привіт" — 12 байтів у UTF-8!
    println!("Слайс: {}", hello);
}
```

### Корисні методи String

```rust
fn main() {
    let s = String::from("  Hello, World!  ");
    
    // Довжина
    println!("len (bytes): {}", s.len());
    println!("chars count: {}", s.chars().count());
    
    // Перевірки
    println!("is_empty: {}", s.is_empty());
    println!("contains 'World': {}", s.contains("World"));
    println!("starts_with '  H': {}", s.starts_with("  H"));
    println!("ends_with '!  ': {}", s.ends_with("!  "));
    
    // Трансформації (повертають новий String)
    println!("trim: '{}'", s.trim());
    println!("to_uppercase: {}", s.to_uppercase());
    println!("to_lowercase: {}", s.to_lowercase());
    println!("replace: {}", s.replace("World", "Rust"));
    
    // Розбиття
    let parts: Vec<&str> = "a,b,c,d".split(',').collect();
    println!("split: {:?}", parts);
    
    let words: Vec<&str> = "hello world rust".split_whitespace().collect();
    println!("words: {:?}", words);
    
    // Об'єднання
    let joined = ["Alpha", "Beta", "Gamma"].join(", ");
    println!("joined: {}", joined);
}
```

### Парсинг та конвертація

```rust
fn main() {
    // String -> число
    let num_str = "42";
    let num: i32 = num_str.parse().expect("Не число!");
    println!("Parsed: {}", num);
    
    // Безпечний парсинг
    match "not_a_number".parse::<i32>() {
        Ok(n) => println!("Число: {}", n),
        Err(e) => println!("Помилка: {}", e),
    }
    
    // Число -> String
    let n = 123;
    let s = n.to_string();
    println!("String: {}", s);
    
    // format! для складних конвертацій
    let formatted = format!("ID: {:05}, Value: {:.2}", 42, 3.14159);
    println!("{}", formatted);  // "ID: 00042, Value: 3.14"
}
```

---

## Вибір колекції

| Задача | Колекція | Причина |
|--------|----------|---------|
| Список елементів | `Vec<T>` | Швидкий доступ за індексом |
| Черга/стек | `Vec<T>` | push/pop ефективні |
| Пошук за ключем | `HashMap<K, V>` | O(1) пошук |
| Унікальні елементи | `HashSet<T>` | Автоматична дедуплікація |
| Відсортований список | `BTreeMap/BTreeSet` | Впорядкованість |
| Текст | `String` | Динамічний, володіючий |
| Посилання на текст | `&str` | Легкий, незмінний |

---

## Типові помилки

### Помилка 1: Модифікація під час ітерації

```rust
fn main() {
    let mut v = vec![1, 2, 3, 4, 5];
    
    // ПОМИЛКА! Не можна модифікувати під час ітерації
    // for x in &v {
    //     if *x == 3 {
    //         v.push(10);  // Borrow conflict!
    //     }
    // }
    
    // Виправлення: збираємо індекси, потім модифікуємо
    let to_add: Vec<i32> = v.iter().filter(|&&x| x == 3).map(|_| 10).collect();
    v.extend(to_add);
}
```

### Помилка 2: Ownership при ітерації

```rust
fn main() {
    let names = vec![String::from("Alpha"), String::from("Beta")];
    
    // into_iter() — споживає вектор!
    for name in names {
        println!("{}", name);
    }
    // println!("{:?}", names);  // ПОМИЛКА! names переміщено
    
    // Виправлення: використовуйте &names або .iter()
    let names2 = vec![String::from("Gamma"), String::from("Delta")];
    for name in &names2 {
        println!("{}", name);
    }
    println!("Все ще доступно: {:?}", names2);
}
```

### Помилка 3: Індексація String

```rust
fn main() {
    let s = String::from("Привіт");
    
    // ПОМИЛКА! String не підтримує індексацію
    // let c = s[0];
    
    // Виправлення: chars().nth()
    if let Some(c) = s.chars().nth(0) {
        println!("Перший символ: {}", c);
    }
}
```

### Помилка 4: HashMap з некоректним ключем

```rust
use std::collections::HashMap;

fn main() {
    // f64 НЕ може бути ключем (не реалізує Hash)!
    // let mut map: HashMap<f64, String> = HashMap::new();  // ПОМИЛКА!
    
    // Виправлення: використовуйте інший тип або обгортку
    let mut map: HashMap<i32, String> = HashMap::new();
    map.insert(42, String::from("value"));
}
```

---

## Практичні задачі

### Задача 1: Журнал польотів (Vec)

**Умова:** Створіть структуру `FlightLog`, яка зберігає записи про польоти у векторі. Реалізуйте методи: `add_entry`, `get_entries_by_drone`, `total_flight_time`, `last_entries(n)`.

**Розв'язання:**

```rust
#[derive(Debug, Clone)]
struct FlightEntry {
    drone_name: String,
    timestamp: String,
    duration_minutes: u32,
    distance_km: f64,
    mission_type: String,
}

struct FlightLog {
    entries: Vec<FlightEntry>,
}

impl FlightEntry {
    fn new(drone: &str, timestamp: &str, duration: u32, distance: f64, mission: &str) -> Self {
        FlightEntry {
            drone_name: String::from(drone),
            timestamp: String::from(timestamp),
            duration_minutes: duration,
            distance_km: distance,
            mission_type: String::from(mission),
        }
    }
}

impl FlightLog {
    fn new() -> Self {
        FlightLog { entries: Vec::new() }
    }
    
    fn add_entry(&mut self, entry: FlightEntry) {
        println!("📝 Додано запис: {} - {}", entry.drone_name, entry.mission_type);
        self.entries.push(entry);
    }
    
    fn get_entries_by_drone(&self, drone_name: &str) -> Vec<&FlightEntry> {
        self.entries
            .iter()
            .filter(|e| e.drone_name == drone_name)
            .collect()
    }
    
    fn total_flight_time(&self) -> u32 {
        self.entries.iter().map(|e| e.duration_minutes).sum()
    }
    
    fn total_flight_time_by_drone(&self, drone_name: &str) -> u32 {
        self.entries
            .iter()
            .filter(|e| e.drone_name == drone_name)
            .map(|e| e.duration_minutes)
            .sum()
    }
    
    fn last_entries(&self, n: usize) -> Vec<&FlightEntry> {
        let start = if self.entries.len() > n {
            self.entries.len() - n
        } else {
            0
        };
        self.entries[start..].iter().collect()
    }
    
    fn total_distance(&self) -> f64 {
        self.entries.iter().map(|e| e.distance_km).sum()
    }
    
    fn count(&self) -> usize {
        self.entries.len()
    }
    
    fn print_summary(&self) {
        println!("\n=================== ЖУРНАЛ ПОЛЬОТІВ ===================");
        println!("Всього записів: {}", self.count());
        println!("Загальний час: {} хв ({:.1} год)", 
                 self.total_flight_time(),
                 self.total_flight_time() as f64 / 60.0);
        println!("Загальна відстань: {:.1} км", self.total_distance());
    }
}

fn main() {
    println!("=== Журнал польотів (Vec) ===\n");
    
    let mut log = FlightLog::new();
    
    // Додаємо записи
    log.add_entry(FlightEntry::new("Alpha", "2024-01-15 09:00", 45, 12.5, "Патрулювання"));
    log.add_entry(FlightEntry::new("Beta", "2024-01-15 10:30", 30, 8.0, "Розвідка"));
    log.add_entry(FlightEntry::new("Alpha", "2024-01-15 14:00", 60, 20.0, "Доставка"));
    log.add_entry(FlightEntry::new("Gamma", "2024-01-15 16:00", 25, 5.5, "Тестування"));
    log.add_entry(FlightEntry::new("Alpha", "2024-01-16 08:00", 90, 35.0, "Картографування"));
    
    // Загальна статистика
    log.print_summary();
    
    // Польоти Alpha
    println!("\n--- Польоти дрона Alpha ---");
    let alpha_flights = log.get_entries_by_drone("Alpha");
    for entry in &alpha_flights {
        println!("  {} | {} хв | {:.1} км | {}", 
                 entry.timestamp, entry.duration_minutes, 
                 entry.distance_km, entry.mission_type);
    }
    println!("Загальний час Alpha: {} хв", log.total_flight_time_by_drone("Alpha"));
    
    // Останні 3 записи
    println!("\n--- Останні 3 записи ---");
    for entry in log.last_entries(3) {
        println!("  {} - {} ({})", entry.drone_name, entry.mission_type, entry.timestamp);
    }
}
```

**Результат:**
```
=== Журнал польотів (Vec) ===

📝 Додано запис: Alpha - Патрулювання
📝 Додано запис: Beta - Розвідка
📝 Додано запис: Alpha - Доставка
📝 Додано запис: Gamma - Тестування
📝 Додано запис: Alpha - Картографування

=================== ЖУРНАЛ ПОЛЬОТІВ ===================
Всього записів: 5
Загальний час: 250 хв (4.2 год)
Загальна відстань: 81.0 км

--- Польоти дрона Alpha ---
  2024-01-15 09:00 | 45 хв | 12.5 км | Патрулювання
  2024-01-15 14:00 | 60 хв | 20.0 км | Доставка
  2024-01-16 08:00 | 90 хв | 35.0 км | Картографування
Загальний час Alpha: 195 хв

--- Останні 3 записи ---
  Alpha - Доставка (2024-01-15 14:00)
  Gamma - Тестування (2024-01-15 16:00)
  Alpha - Картографування (2024-01-16 08:00)
```

---

### Задача 2: Реєстр дронів (HashMap)

**Умова:** Створіть `DroneRegistry`, що зберігає дрони за унікальним ID. Реалізуйте методи: `register`, `unregister`, `get_by_id`, `update_battery`, `find_available`, `statistics`.

**Розв'язання:**

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
struct DroneInfo {
    name: String,
    model: String,
    battery: i32,
    status: String,
    missions_completed: u32,
}

struct DroneRegistry {
    drones: HashMap<String, DroneInfo>,
}

impl DroneInfo {
    fn new(name: &str, model: &str) -> Self {
        DroneInfo {
            name: String::from(name),
            model: String::from(model),
            battery: 100,
            status: String::from("idle"),
            missions_completed: 0,
        }
    }
    
    fn is_available(&self) -> bool {
        self.status == "idle" && self.battery >= 20
    }
}

impl DroneRegistry {
    fn new() -> Self {
        DroneRegistry {
            drones: HashMap::new(),
        }
    }
    
    fn register(&mut self, id: &str, drone: DroneInfo) -> Result<(), String> {
        if self.drones.contains_key(id) {
            return Err(format!("ID '{}' вже зареєстровано", id));
        }
        
        println!("✓ Зареєстровано: {} ({})", drone.name, id);
        self.drones.insert(String::from(id), drone);
        Ok(())
    }
    
    fn unregister(&mut self, id: &str) -> Result<DroneInfo, String> {
        match self.drones.remove(id) {
            Some(drone) => {
                println!("✓ Видалено з реєстру: {} ({})", drone.name, id);
                Ok(drone)
            },
            None => Err(format!("ID '{}' не знайдено", id)),
        }
    }
    
    fn get_by_id(&self, id: &str) -> Option<&DroneInfo> {
        self.drones.get(id)
    }
    
    fn update_battery(&mut self, id: &str, battery: i32) -> Result<(), String> {
        match self.drones.get_mut(id) {
            Some(drone) => {
                let old = drone.battery;
                drone.battery = battery.clamp(0, 100);
                println!("🔋 {}: {}% → {}%", drone.name, old, drone.battery);
                Ok(())
            },
            None => Err(format!("ID '{}' не знайдено", id)),
        }
    }
    
    fn set_status(&mut self, id: &str, status: &str) -> Result<(), String> {
        match self.drones.get_mut(id) {
            Some(drone) => {
                println!("📊 {}: {} → {}", drone.name, drone.status, status);
                drone.status = String::from(status);
                Ok(())
            },
            None => Err(format!("ID '{}' не знайдено", id)),
        }
    }
    
    fn find_available(&self) -> Vec<(&String, &DroneInfo)> {
        self.drones
            .iter()
            .filter(|(_, drone)| drone.is_available())
            .collect()
    }
    
    fn find_by_model(&self, model: &str) -> Vec<(&String, &DroneInfo)> {
        self.drones
            .iter()
            .filter(|(_, drone)| drone.model == model)
            .collect()
    }
    
    fn statistics(&self) -> (usize, usize, f64) {
        let total = self.drones.len();
        let available = self.find_available().len();
        let avg_battery: f64 = if total > 0 {
            self.drones.values().map(|d| d.battery as f64).sum::<f64>() / total as f64
        } else {
            0.0
        };
        (total, available, avg_battery)
    }
    
    fn print_all(&self) {
        println!("\n{:═<60}", "");
        println!("{:^60}", "РЕЄСТР ДРОНІВ");
        println!("{:═<60}", "");
        println!("{:<8} {:<12} {:<10} {:<8} {:<10} {}", 
                 "ID", "Ім'я", "Модель", "Батарея", "Статус", "Місій");
        println!("{:-<60}", "");
        
        for (id, drone) in &self.drones {
            let battery_icon = if drone.battery >= 50 { "🟢" } 
                              else if drone.battery >= 20 { "🟡" } 
                              else { "🔴" };
            println!("{:<8} {:<12} {:<10} {} {:>3}%   {:<10} {}", 
                     id, drone.name, drone.model, battery_icon,
                     drone.battery, drone.status, drone.missions_completed);
        }
        
        let (total, available, avg_bat) = self.statistics();
        println!("{:-<60}", "");
        println!("Всього: {} | Доступно: {} | Сер. батарея: {:.1}%", 
                 total, available, avg_bat);
    }
}

fn main() {
    println!("=== Реєстр дронів (HashMap) ===\n");
    
    let mut registry = DroneRegistry::new();
    
    // Реєстрація
    registry.register("DRN-001", DroneInfo::new("Alpha", "Scout-X")).unwrap();
    registry.register("DRN-002", DroneInfo::new("Beta", "Scout-X")).unwrap();
    registry.register("DRN-003", DroneInfo::new("Gamma", "Heavy-Y")).unwrap();
    registry.register("DRN-004", DroneInfo::new("Delta", "Scout-X")).unwrap();
    
    // Спроба дублікату
    if let Err(e) = registry.register("DRN-001", DroneInfo::new("Fake", "X")) {
        println!("✗ Помилка: {}", e);
    }
    
    registry.print_all();
    
    // Оновлення
    println!("\n--- Оновлення стану ---");
    registry.update_battery("DRN-002", 45).unwrap();
    registry.update_battery("DRN-004", 15).unwrap();
    registry.set_status("DRN-001", "flying").unwrap();
    registry.set_status("DRN-003", "charging").unwrap();
    
    registry.print_all();
    
    // Пошук доступних
    println!("\n--- Доступні для місії ---");
    for (id, drone) in registry.find_available() {
        println!("  {} - {} ({}%)", id, drone.name, drone.battery);
    }
    
    // Пошук за моделлю
    println!("\n--- Дрони Scout-X ---");
    for (id, drone) in registry.find_by_model("Scout-X") {
        println!("  {} - {}", id, drone.name);
    }
}
```

---

### Задача 3: Генератор повідомлень (String)

**Умова:** Створіть систему генерації повідомлень для дронів. Реалізуйте функції: `format_alert`, `parse_command`, `build_report`, `sanitize_name`.

**Розв'язання:**

```rust
#[derive(Debug)]
enum AlertLevel {
    Info,
    Warning,
    Critical,
}

#[derive(Debug)]
struct ParsedCommand {
    action: String,
    target: String,
    params: Vec<String>,
}

fn format_alert(level: AlertLevel, drone: &str, message: &str) -> String {
    let (icon, prefix) = match level {
        AlertLevel::Info => ("ℹ️", "INFO"),
        AlertLevel::Warning => ("⚠️", "WARN"),
        AlertLevel::Critical => ("🚨", "CRIT"),
    };
    
    let timestamp = "2024-01-15 14:30:00";  // В реальності — справжній час
    
    format!("[{}] {} {} | {} | {}", timestamp, icon, prefix, drone, message)
}

fn parse_command(input: &str) -> Result<ParsedCommand, String> {
    let parts: Vec<&str> = input.trim().split_whitespace().collect();
    
    if parts.is_empty() {
        return Err(String::from("Порожня команда"));
    }
    
    let action = parts[0].to_uppercase();
    
    let valid_actions = ["MOVE", "LAND", "TAKEOFF", "SCAN", "RETURN", "STATUS"];
    if !valid_actions.contains(&action.as_str()) {
        return Err(format!("Невідома дія: {}", action));
    }
    
    let target = if parts.len() > 1 {
        parts[1].to_string()
    } else {
        String::from("*")  // Всі дрони
    };
    
    let params: Vec<String> = parts.iter().skip(2).map(|s| s.to_string()).collect();
    
    Ok(ParsedCommand {
        action,
        target,
        params,
    })
}

fn build_report(drone_name: &str, stats: &[(String, String)]) -> String {
    let mut report = String::new();
    let width = 42;
    
    report.push_str(&format!("╔{:═<width$}╗\n", "", width = width));
    report.push_str(&format!("║{:^width$}║\n", format!("ЗВІТ: {}", drone_name), width = width));
    report.push_str(&format!("╠{:═<width$}╣\n", "", width = width));
    
    for (key, value) in stats {
        let line = format!(" {:<18}: {:>18} ", key, value);
        report.push_str(&format!("║{}║\n", line));
    }
    
    report.push_str(&format!("╚{:═<width$}╝", "", width = width));
    report
}

fn sanitize_name(name: &str) -> String {
    name.chars()
        .filter(|c| c.is_alphanumeric() || *c == '-' || *c == '_')
        .collect::<String>()
        .to_uppercase()
        .chars()
        .take(20)
        .collect()
}

fn truncate_message(msg: &str, max_len: usize) -> String {
    if msg.chars().count() <= max_len {
        msg.to_string()
    } else {
        let truncated: String = msg.chars().take(max_len - 3).collect();
        format!("{}...", truncated)
    }
}

fn main() {
    println!("=== Генератор повідомлень (String) ===\n");
    
    // Форматування алертів
    println!("--- Алерти ---");
    println!("{}", format_alert(AlertLevel::Info, "Alpha", "Місію завершено"));
    println!("{}", format_alert(AlertLevel::Warning, "Beta", "Низький заряд батареї"));
    println!("{}", format_alert(AlertLevel::Critical, "Gamma", "Втрата GPS сигналу"));
    
    // Парсинг команд
    println!("\n--- Парсинг команд ---");
    let commands = [
        "move Alpha 100 200",
        "takeoff Beta",
        "status",
        "dance Alpha",
        "",
    ];
    
    for cmd in commands {
        print!("'{}' → ", cmd);
        match parse_command(cmd) {
            Ok(parsed) => println!("{:?}", parsed),
            Err(e) => println!("Помилка: {}", e),
        }
    }
    
    // Генерація звіту
    println!("\n--- Звіт ---");
    let stats = vec![
        (String::from("Статус"), String::from("Активний")),
        (String::from("Батарея"), String::from("85%")),
        (String::from("Позиція"), String::from("(100, 250)")),
        (String::from("Швидкість"), String::from("15 м/с")),
        (String::from("Місій виконано"), String::from("42")),
    ];
    println!("{}", build_report("ALPHA-001", &stats));
    
    // Санітизація імен
    println!("\n--- Санітизація імен ---");
    let names = ["drone@#$123", "  Test Drone  ", "Very-Long-Name-Test-Truncated"];
    for name in names {
        println!("'{}' → '{}'", name, sanitize_name(name));
    }
    
    // Обрізання повідомлень
    println!("\n--- Обрізання повідомлень ---");
    let long_msg = "Це дуже довге повідомлення, яке потрібно обрізати";
    println!("Оригінал: {}", long_msg);
    println!("Обрізане (20): {}", truncate_message(long_msg, 20));
}
```

---

### Задача 4: Система інвентаризації (комбінування)

**Умова:** Об'єднайте Vec, HashMap та String для створення системи інвентаризації компонентів дронів.

**Розв'язання:**

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
struct Component {
    name: String,
    category: String,
    quantity: u32,
    min_quantity: u32,
    unit_price: f64,
}

struct Inventory {
    components: HashMap<String, Component>,
    transactions: Vec<String>,
}

impl Component {
    fn new(name: &str, category: &str, quantity: u32, min_qty: u32, price: f64) -> Self {
        Component {
            name: String::from(name),
            category: String::from(category),
            quantity,
            min_quantity: min_qty,
            unit_price: price,
        }
    }
    
    fn needs_restock(&self) -> bool {
        self.quantity < self.min_quantity
    }
    
    fn total_value(&self) -> f64 {
        self.quantity as f64 * self.unit_price
    }
}

impl Inventory {
    fn new() -> Self {
        Inventory {
            components: HashMap::new(),
            transactions: Vec::new(),
        }
    }
    
    fn add_component(&mut self, id: &str, component: Component) {
        let log = format!("ADD: {} - {} шт.", component.name, component.quantity);
        self.transactions.push(log);
        self.components.insert(String::from(id), component);
    }
    
    fn restock(&mut self, id: &str, amount: u32) -> Result<(), String> {
        match self.components.get_mut(id) {
            Some(comp) => {
                comp.quantity += amount;
                let log = format!("RESTOCK: {} +{} (всього: {})", 
                                 comp.name, amount, comp.quantity);
                self.transactions.push(log);
                Ok(())
            },
            None => Err(format!("Компонент '{}' не знайдено", id)),
        }
    }
    
    fn use_component(&mut self, id: &str, amount: u32) -> Result<(), String> {
        match self.components.get_mut(id) {
            Some(comp) => {
                if comp.quantity < amount {
                    return Err(format!("Недостатньо {}: є {}, потрібно {}", 
                                      comp.name, comp.quantity, amount));
                }
                comp.quantity -= amount;
                let log = format!("USE: {} -{} (залишок: {})", 
                                 comp.name, amount, comp.quantity);
                self.transactions.push(log);
                Ok(())
            },
            None => Err(format!("Компонент '{}' не знайдено", id)),
        }
    }
    
    fn get_by_category(&self, category: &str) -> Vec<(&String, &Component)> {
        self.components
            .iter()
            .filter(|(_, comp)| comp.category == category)
            .collect()
    }
    
    fn needs_restock_list(&self) -> Vec<(&String, &Component)> {
        self.components
            .iter()
            .filter(|(_, comp)| comp.needs_restock())
            .collect()
    }
    
    fn total_inventory_value(&self) -> f64 {
        self.components.values().map(|c| c.total_value()).sum()
    }
    
    fn search(&self, query: &str) -> Vec<(&String, &Component)> {
        let query_lower = query.to_lowercase();
        self.components
            .iter()
            .filter(|(id, comp)| {
                id.to_lowercase().contains(&query_lower) ||
                comp.name.to_lowercase().contains(&query_lower)
            })
            .collect()
    }
    
    fn print_inventory(&self) {
        println!("\n{:═<70}", "");
        println!("{:^70}", "ІНВЕНТАР КОМПОНЕНТІВ");
        println!("{:═<70}", "");
        println!("{:<10} {:<20} {:<12} {:>8} {:>8} {:>10}", 
                 "ID", "Назва", "Категорія", "К-сть", "Мін", "Вартість");
        println!("{:-<70}", "");
        
        for (id, comp) in &self.components {
            let status = if comp.needs_restock() { "⚠️" } else { "  " };
            println!("{} {:<8} {:<20} {:<12} {:>6} {:>8} {:>10.2}", 
                     status, id, comp.name, comp.category, 
                     comp.quantity, comp.min_quantity, comp.total_value());
        }
        
        println!("{:-<70}", "");
        println!("{:>58} {:>10.2}", "Загальна вартість:", self.total_inventory_value());
    }
    
    fn print_transactions(&self, last_n: usize) {
        println!("\n--- Останні {} транзакцій ---", last_n);
        let start = if self.transactions.len() > last_n {
            self.transactions.len() - last_n
        } else {
            0
        };
        
        for (i, t) in self.transactions[start..].iter().enumerate() {
            println!("  {}. {}", i + 1, t);
        }
    }
}

fn main() {
    println!("=== Система інвентаризації ===\n");
    
    let mut inv = Inventory::new();
    
    // Додаємо компоненти
    inv.add_component("MTR-01", Component::new("Мотор A2212", "Двигуни", 20, 10, 15.00));
    inv.add_component("MTR-02", Component::new("Мотор 2806.5", "Двигуни", 8, 10, 25.00));
    inv.add_component("PRP-01", Component::new("Пропелер 10x4.5", "Пропелери", 50, 20, 2.50));
    inv.add_component("BAT-01", Component::new("LiPo 4S 5000mAh", "Батареї", 15, 5, 45.00));
    inv.add_component("ESC-01", Component::new("ESC 30A", "Електроніка", 12, 8, 12.00));
    inv.add_component("FC-01", Component::new("Flight Controller", "Електроніка", 5, 3, 85.00));
    
    inv.print_inventory();
    
    // Використання компонентів
    println!("\n--- Операції ---");
    inv.use_component("MTR-01", 4).unwrap();
    inv.use_component("PRP-01", 16).unwrap();
    inv.use_component("BAT-01", 2).unwrap();
    inv.use_component("MTR-02", 4).unwrap();
    
    // Поповнення
    inv.restock("MTR-02", 10).unwrap();
    
    inv.print_inventory();
    
    // Список для замовлення
    println!("\n--- Потребують поповнення ---");
    let restock = inv.needs_restock_list();
    if restock.is_empty() {
        println!("  Все в нормі!");
    } else {
        for (id, comp) in restock {
            println!("  {} - {} (є: {}, мін: {})", 
                     id, comp.name, comp.quantity, comp.min_quantity);
        }
    }
    
    // Пошук
    println!("\n--- Пошук 'мотор' ---");
    for (id, comp) in inv.search("мотор") {
        println!("  {} - {}", id, comp.name);
    }
    
    // Категорія
    println!("\n--- Категорія 'Електроніка' ---");
    for (id, comp) in inv.get_by_category("Електроніка") {
        println!("  {} - {} ({} шт.)", id, comp.name, comp.quantity);
    }
    
    // Історія
    inv.print_transactions(5);
}
```

---

## Домашнє завдання

### Завдання 1: Черга завдань з пріоритетами (Vec)

**Умова:** Реалізуйте чергу завдань для дронів з пріоритетами. Завдання з вищим пріоритетом виконуються першими. При однаковому пріоритеті — FIFO порядок.

**Розв'язання:**

```rust
#[derive(Debug, Clone)]
struct Task {
    id: u32,
    description: String,
    priority: u8,  // 0 = найвищий
    drone_id: Option<String>,
}

impl Task {
    fn new(id: u32, description: &str, priority: u8) -> Self {
        Task {
            id,
            description: String::from(description),
            priority,
            drone_id: None,
        }
    }
}

struct PriorityQueue {
    tasks: Vec<Task>,
    next_id: u32,
}

impl PriorityQueue {
    fn new() -> Self {
        PriorityQueue {
            tasks: Vec::new(),
            next_id: 1,
        }
    }
    
    fn enqueue(&mut self, description: &str, priority: u8) -> u32 {
        let id = self.next_id;
        self.next_id += 1;
        
        let task = Task::new(id, description, priority);
        
        // Знаходимо позицію для вставки (зберігаємо FIFO для однакових пріоритетів)
        let pos = self.tasks
            .iter()
            .position(|t| t.priority > priority)
            .unwrap_or(self.tasks.len());
        
        self.tasks.insert(pos, task);
        id
    }
    
    fn dequeue(&mut self) -> Option<Task> {
        if self.tasks.is_empty() {
            None
        } else {
            Some(self.tasks.remove(0))
        }
    }
    
    fn peek(&self) -> Option<&Task> {
        self.tasks.first()
    }
    
    fn assign_drone(&mut self, task_id: u32, drone_id: &str) -> bool {
        if let Some(task) = self.tasks.iter_mut().find(|t| t.id == task_id) {
            task.drone_id = Some(String::from(drone_id));
            true
        } else {
            false
        }
    }
    
    fn cancel(&mut self, task_id: u32) -> Option<Task> {
        if let Some(pos) = self.tasks.iter().position(|t| t.id == task_id) {
            Some(self.tasks.remove(pos))
        } else {
            None
        }
    }
    
    fn len(&self) -> usize {
        self.tasks.len()
    }
    
    fn get_by_priority(&self, priority: u8) -> Vec<&Task> {
        self.tasks.iter().filter(|t| t.priority == priority).collect()
    }
    
    fn print_queue(&self) {
        println!("\n=== Черга завдань ({} шт.) ===", self.len());
        if self.tasks.is_empty() {
            println!("  (порожньо)");
            return;
        }
        
        let priority_names = ["КРИТИЧНИЙ", "ВИСОКИЙ", "СЕРЕДНІЙ", "НИЗЬКИЙ"];
        
        for task in &self.tasks {
            let p_name = priority_names.get(task.priority as usize).unwrap_or(&"НЕВІДОМИЙ");
            let drone = task.drone_id.as_deref().unwrap_or("-");
            println!("  [{}] P{} ({}) | {} | Дрон: {}", 
                     task.id, task.priority, p_name, task.description, drone);
        }
    }
}

fn main() {
    println!("=== Черга завдань з пріоритетами ===\n");
    
    let mut queue = PriorityQueue::new();
    
    // Додаємо завдання різних пріоритетів
    queue.enqueue("Планова перевірка периметру", 2);
    queue.enqueue("Доставка медикаментів", 1);
    queue.enqueue("Фотозйомка території", 3);
    queue.enqueue("АВАРІЙНА ЕВАКУАЦІЯ", 0);
    queue.enqueue("Моніторинг трафіку", 2);
    queue.enqueue("Термінова розвідка", 1);
    
    queue.print_queue();
    
    // Призначаємо дрони
    println!("\n--- Призначення дронів ---");
    queue.assign_drone(4, "Alpha");  // АВАРІЙНА
    queue.assign_drone(2, "Beta");   // Доставка
    
    queue.print_queue();
    
    // Виконуємо завдання
    println!("\n--- Виконання завдань ---");
    while let Some(task) = queue.dequeue() {
        let drone = task.drone_id.as_deref().unwrap_or("не призначено");
        println!("✓ Виконую: [{}] {} (дрон: {})", task.id, task.description, drone);
    }
    
    println!("\nЧерга порожня: {}", queue.len() == 0);
}
```

---

### Завдання 2: Кеш маршрутів (HashMap)

**Умова:** Створіть кеш для збереження обчислених маршрутів між точками. Ключ — пара точок (від, до), значення — вектор проміжних точок та загальна відстань.

**Розв'язання:**

```rust
use std::collections::HashMap;

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
struct Point {
    x: i32,
    y: i32,
}

impl Point {
    fn new(x: i32, y: i32) -> Self {
        Point { x, y }
    }
    
    fn distance_to(&self, other: &Point) -> f64 {
        let dx = (self.x - other.x) as f64;
        let dy = (self.y - other.y) as f64;
        (dx * dx + dy * dy).sqrt()
    }
}

#[derive(Debug, Clone)]
struct Route {
    waypoints: Vec<Point>,
    total_distance: f64,
    computed_at: u64,  // timestamp
}

struct RouteCache {
    cache: HashMap<(Point, Point), Route>,
    hits: u32,
    misses: u32,
    max_size: usize,
}

impl RouteCache {
    fn new(max_size: usize) -> Self {
        RouteCache {
            cache: HashMap::new(),
            hits: 0,
            misses: 0,
            max_size,
        }
    }
    
    fn get(&mut self, from: Point, to: Point) -> Option<&Route> {
        let key = (from, to);
        if self.cache.contains_key(&key) {
            self.hits += 1;
            self.cache.get(&key)
        } else {
            self.misses += 1;
            None
        }
    }
    
    fn put(&mut self, from: Point, to: Point, waypoints: Vec<Point>, distance: f64) {
        // Простий LRU: видаляємо випадковий якщо переповнений
        if self.cache.len() >= self.max_size {
            if let Some(key) = self.cache.keys().next().cloned() {
                self.cache.remove(&key);
            }
        }
        
        let route = Route {
            waypoints,
            total_distance: distance,
            computed_at: 0, // Спрощено
        };
        
        self.cache.insert((from, to), route);
    }
    
    fn invalidate(&mut self, point: Point) {
        // Видаляємо всі маршрути, що проходять через точку
        self.cache.retain(|(from, to), route| {
            *from != point && *to != point && 
            !route.waypoints.contains(&point)
        });
    }
    
    fn clear(&mut self) {
        self.cache.clear();
        self.hits = 0;
        self.misses = 0;
    }
    
    fn stats(&self) -> (u32, u32, f64) {
        let total = self.hits + self.misses;
        let hit_rate = if total > 0 {
            (self.hits as f64 / total as f64) * 100.0
        } else {
            0.0
        };
        (self.hits, self.misses, hit_rate)
    }
    
    fn print_cache(&self) {
        println!("\n=== Кеш маршрутів ({}/{}) ===", self.cache.len(), self.max_size);
        for ((from, to), route) in &self.cache {
            println!("  ({},{}) → ({},{}) : {:.1} км, {} точок", 
                     from.x, from.y, to.x, to.y, 
                     route.total_distance, route.waypoints.len());
        }
        let (hits, misses, rate) = self.stats();
        println!("  Статистика: {} hits, {} misses ({:.1}% hit rate)", hits, misses, rate);
    }
}

// Функція обчислення маршруту (спрощена)
fn compute_route(from: Point, to: Point) -> (Vec<Point>, f64) {
    println!("  [Обчислення маршруту ({},{}) → ({},{})]", from.x, from.y, to.x, to.y);
    
    // Спрощено: пряма лінія з проміжними точками
    let mut waypoints = Vec::new();
    let steps = 3;
    
    for i in 1..steps {
        let t = i as f64 / steps as f64;
        let x = from.x + ((to.x - from.x) as f64 * t) as i32;
        let y = from.y + ((to.y - from.y) as f64 * t) as i32;
        waypoints.push(Point::new(x, y));
    }
    
    let distance = from.distance_to(&to);
    (waypoints, distance)
}

fn get_route_cached(cache: &mut RouteCache, from: Point, to: Point) -> Route {
    if let Some(route) = cache.get(from, to) {
        println!("  [Cache HIT]");
        return route.clone();
    }
    
    println!("  [Cache MISS]");
    let (waypoints, distance) = compute_route(from, to);
    cache.put(from, to, waypoints.clone(), distance);
    
    Route {
        waypoints,
        total_distance: distance,
        computed_at: 0,
    }
}

fn main() {
    println!("=== Кеш маршрутів ===\n");
    
    let mut cache = RouteCache::new(10);
    
    let base = Point::new(0, 0);
    let points = vec![
        Point::new(100, 50),
        Point::new(50, 100),
        Point::new(150, 150),
    ];
    
    // Перші запити — всі MISS
    println!("--- Перші запити ---");
    for p in &points {
        let route = get_route_cached(&mut cache, base, *p);
        println!("    Маршрут: {:.1} км\n", route.total_distance);
    }
    
    cache.print_cache();
    
    // Повторні запити — всі HIT
    println!("\n--- Повторні запити ---");
    for p in &points {
        let route = get_route_cached(&mut cache, base, *p);
        println!("    Маршрут: {:.1} км\n", route.total_distance);
    }
    
    cache.print_cache();
}
```

---

### Завдання 3: Парсер логів (String)

**Умова:** Напишіть парсер для рядків логів формату `[TIMESTAMP] [LEVEL] [DRONE] MESSAGE`. Витягніть всі компоненти, зберіть статистику, відфільтруйте за критеріями.

**Розв'язання:**

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
struct LogEntry {
    timestamp: String,
    level: String,
    drone: String,
    message: String,
}

impl LogEntry {
    fn parse(line: &str) -> Option<Self> {
        let line = line.trim();
        if line.is_empty() || !line.starts_with('[') {
            return None;
        }
        
        // Парсимо [TIMESTAMP] [LEVEL] [DRONE] MESSAGE
        let mut parts = Vec::new();
        let mut current = String::new();
        let mut in_bracket = false;
        let mut chars = line.chars().peekable();
        
        while let Some(c) = chars.next() {
            match c {
                '[' => {
                    in_bracket = true;
                    current.clear();
                },
                ']' => {
                    in_bracket = false;
                    parts.push(current.clone());
                    current.clear();
                },
                _ if in_bracket => {
                    current.push(c);
                },
                _ if parts.len() >= 3 => {
                    current.push(c);
                },
                _ => {}
            }
        }
        
        if parts.len() < 3 {
            return None;
        }
        
        Some(LogEntry {
            timestamp: parts[0].clone(),
            level: parts[1].clone(),
            drone: parts[2].clone(),
            message: current.trim().to_string(),
        })
    }
    
    fn is_error(&self) -> bool {
        self.level == "ERROR" || self.level == "CRITICAL"
    }
}

struct LogAnalyzer {
    entries: Vec<LogEntry>,
}

impl LogAnalyzer {
    fn new() -> Self {
        LogAnalyzer { entries: Vec::new() }
    }
    
    fn parse_logs(&mut self, text: &str) {
        for line in text.lines() {
            if let Some(entry) = LogEntry::parse(line) {
                self.entries.push(entry);
            }
        }
    }
    
    fn filter_by_level(&self, level: &str) -> Vec<&LogEntry> {
        self.entries.iter()
            .filter(|e| e.level == level)
            .collect()
    }
    
    fn filter_by_drone(&self, drone: &str) -> Vec<&LogEntry> {
        self.entries.iter()
            .filter(|e| e.drone == drone)
            .collect()
    }
    
    fn search(&self, keyword: &str) -> Vec<&LogEntry> {
        let kw = keyword.to_lowercase();
        self.entries.iter()
            .filter(|e| e.message.to_lowercase().contains(&kw))
            .collect()
    }
    
    fn count_by_level(&self) -> HashMap<String, usize> {
        let mut counts = HashMap::new();
        for entry in &self.entries {
            *counts.entry(entry.level.clone()).or_insert(0) += 1;
        }
        counts
    }
    
    fn count_by_drone(&self) -> HashMap<String, usize> {
        let mut counts = HashMap::new();
        for entry in &self.entries {
            *counts.entry(entry.drone.clone()).or_insert(0) += 1;
        }
        counts
    }
    
    fn get_errors(&self) -> Vec<&LogEntry> {
        self.entries.iter().filter(|e| e.is_error()).collect()
    }
    
    fn print_summary(&self) {
        println!("\n{:═<60}", "");
        println!("{:^60}", "АНАЛІЗ ЛОГІВ");
        println!("{:═<60}", "");
        
        println!("\nВсього записів: {}", self.entries.len());
        
        println!("\nПо рівнях:");
        let by_level = self.count_by_level();
        let levels = ["CRITICAL", "ERROR", "WARNING", "INFO", "DEBUG"];
        for level in levels {
            if let Some(count) = by_level.get(level) {
                let bar = "█".repeat((*count).min(30));
                println!("  {:10} : {:3} {}", level, count, bar);
            }
        }
        
        println!("\nПо дронах:");
        for (drone, count) in self.count_by_drone() {
            println!("  {:10} : {} записів", drone, count);
        }
        
        let errors = self.get_errors();
        if !errors.is_empty() {
            println!("\n⚠️  Помилки ({}):", errors.len());
            for e in errors.iter().take(5) {
                println!("  [{}] {} - {}", e.timestamp, e.drone, e.message);
            }
        }
    }
}

fn main() {
    println!("=== Парсер логів ===");
    
    let logs = r#"
[10:15:01] [INFO] [Alpha] System started
[10:15:02] [DEBUG] [Alpha] GPS initializing
[10:15:03] [INFO] [Alpha] GPS ready
[10:15:05] [INFO] [Beta] System started
[10:15:06] [WARNING] [Beta] Low battery: 25%
[10:15:10] [INFO] [Alpha] Taking off
[10:15:15] [ERROR] [Beta] GPS signal lost
[10:15:20] [INFO] [Gamma] System started
[10:15:25] [DEBUG] [Alpha] Altitude: 50m
[10:15:30] [WARNING] [Alpha] Wind speed high
[10:15:35] [CRITICAL] [Beta] Emergency landing initiated
[10:15:40] [INFO] [Gamma] Mission started
[10:15:45] [INFO] [Alpha] Waypoint 1 reached
[10:15:50] [ERROR] [Gamma] Sensor malfunction
[10:15:55] [INFO] [Alpha] Photo captured
[10:16:00] [INFO] [Beta] Emergency landing complete
    "#;
    
    let mut analyzer = LogAnalyzer::new();
    analyzer.parse_logs(logs);
    
    analyzer.print_summary();
    
    // Фільтрація
    println!("\n--- Логи дрона Alpha ---");
    for entry in analyzer.filter_by_drone("Alpha").iter().take(3) {
        println!("  [{}] {}", entry.level, entry.message);
    }
    
    // Пошук
    println!("\n--- Пошук 'GPS' ---");
    for entry in analyzer.search("gps") {
        println!("  [{}] [{}] {}", entry.timestamp, entry.drone, entry.message);
    }
}
```

---

### Завдання 4: Повна система управління флотом

**Умова:** Об'єднайте всі колекції для системи управління флотом: реєстр дронів (HashMap), журнал подій (Vec), обробка команд (String parsing).

**Розв'язання:**

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
struct Drone {
    id: String,
    name: String,
    battery: i32,
    status: String,
    position: (i32, i32),
}

impl Drone {
    fn new(id: &str, name: &str) -> Self {
        Drone {
            id: String::from(id),
            name: String::from(name),
            battery: 100,
            status: String::from("IDLE"),
            position: (0, 0),
        }
    }
}

#[derive(Debug)]
struct Event {
    timestamp: u32,
    drone_id: String,
    event_type: String,
    details: String,
}

#[derive(Debug)]
enum Command {
    AddDrone { id: String, name: String },
    RemoveDrone { id: String },
    MoveTo { id: String, x: i32, y: i32 },
    Charge { id: String },
    Status { id: String },
    ListAll,
    Help,
    Unknown(String),
}

impl Command {
    fn parse(input: &str) -> Self {
        let parts: Vec<&str> = input.trim().split_whitespace().collect();
        
        if parts.is_empty() {
            return Command::Unknown(String::new());
        }
        
        match parts[0].to_uppercase().as_str() {
            "ADD" if parts.len() >= 3 => {
                Command::AddDrone {
                    id: String::from(parts[1]),
                    name: parts[2..].join(" "),
                }
            },
            "REMOVE" if parts.len() >= 2 => {
                Command::RemoveDrone { id: String::from(parts[1]) }
            },
            "MOVE" if parts.len() >= 4 => {
                let x = parts[2].parse().unwrap_or(0);
                let y = parts[3].parse().unwrap_or(0);
                Command::MoveTo { id: String::from(parts[1]), x, y }
            },
            "CHARGE" if parts.len() >= 2 => {
                Command::Charge { id: String::from(parts[1]) }
            },
            "STATUS" if parts.len() >= 2 => {
                Command::Status { id: String::from(parts[1]) }
            },
            "LIST" => Command::ListAll,
            "HELP" => Command::Help,
            _ => Command::Unknown(input.to_string()),
        }
    }
}

struct FleetManager {
    drones: HashMap<String, Drone>,
    events: Vec<Event>,
    time: u32,
}

impl FleetManager {
    fn new() -> Self {
        FleetManager {
            drones: HashMap::new(),
            events: Vec::new(),
            time: 0,
        }
    }
    
    fn log_event(&mut self, drone_id: &str, event_type: &str, details: &str) {
        self.time += 1;
        self.events.push(Event {
            timestamp: self.time,
            drone_id: String::from(drone_id),
            event_type: String::from(event_type),
            details: String::from(details),
        });
    }
    
    fn execute(&mut self, cmd: Command) -> String {
        match cmd {
            Command::AddDrone { id, name } => {
                if self.drones.contains_key(&id) {
                    return format!("✗ Дрон '{}' вже існує", id);
                }
                let drone = Drone::new(&id, &name);
                self.drones.insert(id.clone(), drone);
                self.log_event(&id, "ADDED", &format!("Новий дрон: {}", name));
                format!("✓ Додано дрон '{}' ({})", id, name)
            },
            
            Command::RemoveDrone { id } => {
                match self.drones.remove(&id) {
                    Some(drone) => {
                        self.log_event(&id, "REMOVED", &drone.name);
                        format!("✓ Видалено дрон '{}' ({})", id, drone.name)
                    },
                    None => format!("✗ Дрон '{}' не знайдено", id),
                }
            },
            
            Command::MoveTo { id, x, y } => {
                match self.drones.get_mut(&id) {
                    Some(drone) => {
                        let (old_x, old_y) = drone.position;
                        drone.position = (x, y);
                        drone.battery -= 10;
                        drone.status = String::from("MOVING");
                        self.log_event(&id, "MOVE", &format!("({},{}) → ({},{})", old_x, old_y, x, y));
                        format!("✓ Дрон '{}' рухається до ({}, {})", id, x, y)
                    },
                    None => format!("✗ Дрон '{}' не знайдено", id),
                }
            },
            
            Command::Charge { id } => {
                match self.drones.get_mut(&id) {
                    Some(drone) => {
                        let old = drone.battery;
                        drone.battery = 100;
                        drone.status = String::from("CHARGING");
                        self.log_event(&id, "CHARGE", &format!("{}% → 100%", old));
                        format!("✓ Дрон '{}' заряджається ({}% → 100%)", id, old)
                    },
                    None => format!("✗ Дрон '{}' не знайдено", id),
                }
            },
            
            Command::Status { id } => {
                match self.drones.get(&id) {
                    Some(drone) => {
                        format!("═══ Дрон '{}' ═══\n  Назва: {}\n  Статус: {}\n  Батарея: {}%\n  Позиція: ({}, {})", 
                                id, drone.name, drone.status, drone.battery, 
                                drone.position.0, drone.position.1)
                    },
                    None => format!("✗ Дрон '{}' не знайдено", id),
                }
            },
            
            Command::ListAll => {
                if self.drones.is_empty() {
                    return String::from("Флот порожній");
                }
                
                let mut result = format!("═══ Флот ({} дронів) ═══\n", self.drones.len());
                for (id, drone) in &self.drones {
                    result.push_str(&format!("  {} | {} | {}% | ({},{})\n",
                        id, drone.status, drone.battery,
                        drone.position.0, drone.position.1));
                }
                result
            },
            
            Command::Help => {
                String::from(r#"Доступні команди:
  ADD <id> <name>      - додати дрон
  REMOVE <id>          - видалити дрон
  MOVE <id> <x> <y>    - перемістити дрон
  CHARGE <id>          - зарядити дрон
  STATUS <id>          - статус дрона
  LIST                 - список всіх дронів
  HELP                 - ця довідка"#)
            },
            
            Command::Unknown(s) => {
                format!("✗ Невідома команда: '{}'. Введіть HELP для довідки.", s)
            },
        }
    }
    
    fn print_events(&self, last_n: usize) {
        println!("\n═══ Останні події ═══");
        let start = self.events.len().saturating_sub(last_n);
        for event in &self.events[start..] {
            println!("  [T{}] [{}] {} - {}", 
                     event.timestamp, event.drone_id, 
                     event.event_type, event.details);
        }
    }
}

fn main() {
    println!("=== Система управління флотом ===\n");
    
    let mut fleet = FleetManager::new();
    
    // Симуляція команд
    let commands = [
        "HELP",
        "ADD D001 Alpha Scout",
        "ADD D002 Beta Transport",
        "ADD D003 Gamma Recon",
        "LIST",
        "MOVE D001 100 50",
        "MOVE D002 -50 200",
        "STATUS D001",
        "CHARGE D001",
        "STATUS D001",
        "REMOVE D003",
        "LIST",
        "MOVE D999 0 0",  // Неіснуючий дрон
    ];
    
    for cmd_str in commands {
        println!("\n> {}", cmd_str);
        let cmd = Command::parse(cmd_str);
        let result = fleet.execute(cmd);
        println!("{}", result);
    }
    
    fleet.print_events(10);
}
```

**Результат:**
```
=== Система управління флотом ===

> HELP
Доступні команди:
  ADD <id> <name>      - додати дрон
  REMOVE <id>          - видалити дрон
  ...

> ADD D001 Alpha Scout
✓ Додано дрон 'D001' (Alpha Scout)

> LIST
═══ Флот (3 дронів) ═══
  D001 | IDLE | 100% | (0,0)
  D002 | IDLE | 100% | (0,0)
  D003 | IDLE | 100% | (0,0)

> MOVE D001 100 50
✓ Дрон 'D001' рухається до (100, 50)

> STATUS D001
═══ Дрон 'D001' ═══
  Назва: Alpha Scout
  Статус: MOVING
  Батарея: 90%
  Позиція: (100, 50)
...
```

---

## Підсумок заняття

На цьому занятті ви навчились:

1. **Працювати з Vec<T>**: push, pop, ітерація, фільтрація
2. **Використовувати HashMap<K, V>**: insert, get, entry API
3. **Маніпулювати String**: створення, модифікація, парсинг
4. **Обирати правильну колекцію** для задачі
5. **Комбінувати колекції** для складних систем

---

## Перевірте себе

1. Чим `Vec::new()` відрізняється від `vec![]`?
2. Що повертає `HashMap::get()`?
3. Як додати елемент у HashMap тільки якщо ключа немає?
4. Чому не можна індексувати String як `s[0]`?
5. Як перетворити `&str` у `String`?
6. Що робить `Vec::retain()`?

**Відповіді:**
1. Нічим функціонально, `vec![]` — зручніший синтаксис
2. `Option<&V>` — посилання на значення або None
3. `map.entry(key).or_insert(value)`
4. String — UTF-8, один символ може займати кілька байтів
5. `s.to_string()` або `String::from(s)`
6. Залишає тільки елементи, для яких предикат повертає true

---

## Наступне заняття

На наступному занятті ми вивчимо **Обробку помилок**: `Result<T, E>`, оператор `?`, власні типи помилок та патерни graceful degradation.
