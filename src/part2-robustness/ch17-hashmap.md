# Розділ 17: Колекції — HashMap та просторові структури

## Вступ

У попередньому розділі ви опанували Vec — динамічний масив для послідовностей елементів. Вектор чудово підходить, коли потрібно зберігати елементи в порядку або коли доступ відбувається за індексом. Але уявіть іншу задачу.

Агент БПЛА досліджує територію. Він сканує сектори, класифікує їх (ліс, вода, будівля, перешкода), і потім йому потрібно швидко відповісти на питання: "Що знаходиться в секторі з координатами (5, 3)?". Або: "Чи є перешкода на шляху до цілі?". Або: "Знайди всі сектори з водою поблизу".

Якщо зберігати дані у векторі як пари ((x, y), тип_поверхні), то для кожного запиту доведеться перебирати ВСІ елементи. При 10,000 секторів — це 10,000 порівнянь для кожного запиту. Занадто повільно для системи реального часу.

`HashMap<K, V>` вирішує цю проблему. Це колекція пар "ключ-значення" з доступом за ключем за O(1) у середньому. Замість перебору всіх елементів, HashMap обчислює позицію значення безпосередньо з ключа.

---

## 17.1 Як працює HashMap

### Ідея хешування

Уявіть бібліотеку з мільйоном книг. Є два способи знайти потрібну книгу:

**Спосіб 1 (Vec):** Перебирати всі книги по черзі, поки не знайдете потрібну. При мільйоні книг — в середньому 500,000 перевірок.

**Спосіб 2 (HashMap):** Книги розкладені по полицях за першою літерою автора. Щоб знайти Шевченка — йдете одразу до полиці "Ш", а там вже значно менше книг для перебору.

HashMap працює за схожим принципом, але набагато розумніше. Він використовує **хеш-функцію** — математичну функцію, що перетворює ключ (будь-який!) на число. Це число визначає "полицю" (bucket), де зберігається значення.

```text
Ключ "hello" → хеш-функція → число 12345 → bucket[12345 % capacity]
Ключ "world" → хеш-функція → число 67890 → bucket[67890 % capacity]
```

Коли ви шукаєте значення за ключем, HashMap обчислює той самий хеш і одразу знає, де шукати — без перебору всіх елементів.

### Колізії

Що станеться, якщо два різних ключі дадуть однаковий хеш (або потраплять в один bucket)? Це називається **колізія**. HashMap обробляє колізії, зберігаючи в кожному bucket ланцюжок пар (ключ, значення) і порівнюючи ключі при пошуку.

При добре спроектованій хеш-функції та достатній ємності колізії трапляються рідко, і середня складність залишається O(1).

### Вимоги до ключів

Щоб тип міг бути ключем HashMap, він повинен реалізувати два traits:

1. **`Hash`** — для обчислення хеш-значення
2. **`Eq`** — для порівняння ключів на рівність (при колізіях)

Стандартні типи (числа, рядки, кортежі з хешованих типів) вже реалізують ці traits. Для власних структур можна використати `#[derive(Hash, Eq, PartialEq)]`.

**Важливо:** `f64` та `f32` **не можуть** бути ключами HashMap! Причина — значення `NaN` (Not a Number) не дорівнює самому собі за стандартом IEEE 754:

```rust
let nan = f64::NAN;
println!("{}", nan == nan);  // false!
```

Це порушує контракт `Eq`, тому компілятор забороняє використовувати float як ключ.

---

## 17.2 Створення HashMap

### Порожня HashMap

Для використання HashMap потрібно імпортувати його з стандартної бібліотеки:

```rust
use std::collections::HashMap;

fn main() {
    // Спосіб 1: явна анотація типів
    let scores: HashMap<String, i32> = HashMap::new();
    println!("Scores: {:?}", scores);  // {}
    
    // Спосіб 2: компілятор виведе типи з контексту
    let mut ages = HashMap::new();
    ages.insert(String::from("Alice"), 30);
    // Тепер ages: HashMap<String, i32>
    
    println!("Ages: {:?}", ages);  // {"Alice": 30}
}
```

У першому випадку ми явно вказали типи ключа та значення. У другому — компілятор побачив `insert(String, i32)` і вивів типи автоматично.

### Створення з початковими даними

Часто потрібно створити HashMap з відомими значеннями. Є кілька способів:

```rust
use std::collections::HashMap;

fn main() {
    // Спосіб 1: послідовні insert
    let mut map1 = HashMap::new();
    map1.insert("one", 1);
    map1.insert("two", 2);
    map1.insert("three", 3);
    
    // Спосіб 2: з масиву пар через collect()
    let map2: HashMap<&str, i32> = [
        ("one", 1),
        ("two", 2),
        ("three", 3),
    ].into_iter().collect();
    
    // Спосіб 3: HashMap::from (Rust 1.56+)
    let map3 = HashMap::from([
        ("one", 1),
        ("two", 2),
        ("three", 3),
    ]);
    
    println!("map1: {:?}", map1);
    println!("map2: {:?}", map2);
    println!("map3: {:?}", map3);
}
```

Метод `collect()` — універсальний спосіб перетворення ітератора в колекцію. Тип результату визначається з контексту або анотації.

### Створення з двох векторів

Іноді ключі та значення приходять окремо. Метод `zip` об'єднує два ітератори в ітератор пар:

```rust
use std::collections::HashMap;

fn main() {
    let keys = vec!["alpha", "beta", "gamma"];
    let values = vec![1, 2, 3];
    
    // zip об'єднує: ("alpha", 1), ("beta", 2), ("gamma", 3)
    let map: HashMap<&str, i32> = keys.into_iter()
        .zip(values.into_iter())
        .collect();
    
    println!("{:?}", map);  // {"alpha": 1, "beta": 2, "gamma": 3}
}
```

### with_capacity — оптимізація

Якщо ви знаєте приблизну кількість елементів, `with_capacity` уникає перевиділення пам'яті:

```rust
use std::collections::HashMap;

fn main() {
    // Очікуємо ~100 сенсорів
    let mut sensors: HashMap<String, f64> = HashMap::with_capacity(100);
    
    for i in 0..100 {
        sensors.insert(format!("sensor_{}", i), i as f64 * 0.1);
    }
    
    println!("Length: {}", sensors.len());        // 100
    println!("Capacity: {}", sensors.capacity()); // >= 100
}
```

---

## 17.3 Базові операції

### Вставка — insert

Метод `insert` додає пару ключ-значення. Якщо ключ вже існує — значення замінюється, а старе повертається:

```rust
use std::collections::HashMap;

fn main() {
    let mut scores: HashMap<String, i32> = HashMap::new();
    
    // Вставляємо нові ключі
    scores.insert(String::from("Blue"), 10);
    scores.insert(String::from("Red"), 50);
    println!("Після insert: {:?}", scores);
    
    // Замінюємо існуючий ключ
    let old_value = scores.insert(String::from("Blue"), 25);
    println!("Старе значення Blue: {:?}", old_value);  // Some(10)
    println!("Нове значення Blue: {:?}", scores.get("Blue"));  // Some(&25)
    
    // Вставка нового ключа повертає None
    let was_there = scores.insert(String::from("Green"), 30);
    println!("Green раніше: {:?}", was_there);  // None
}
```

Зверніть увагу: `insert` **переміщує** ключ і значення у HashMap. Якщо ключ — `String`, він більше не доступний після вставки (ownership передано).

### Отримання значення — get

Метод `get` повертає `Option<&V>` — посилання на значення або `None`:

```rust
use std::collections::HashMap;

fn main() {
    let mut scores = HashMap::new();
    scores.insert("Blue", 10);
    scores.insert("Red", 50);
    
    // get повертає Option<&V>
    let blue_score = scores.get("Blue");
    println!("Blue: {:?}", blue_score);  // Some(&10)
    
    let green_score = scores.get("Green");
    println!("Green: {:?}", green_score);  // None
    
    // Зручні комбінації з Option:
    
    // unwrap_or — значення за замовчуванням
    let score = scores.get("Green").unwrap_or(&0);
    println!("Green (default 0): {}", score);  // 0
    
    // copied() — копіює значення (якщо Copy)
    let score: i32 = scores.get("Blue").copied().unwrap_or(0);
    println!("Blue copied: {}", score);  // 10
    
    // Індексація — ПАНІКА якщо ключа немає!
    let red = scores["Red"];
    println!("Red via []: {}", red);  // 50
    // let green = scores["Green"];  // PANIC!
}
```

**Правило:** використовуйте `get()` для безпечного доступу, індексацію `[]` — тільки коли на 100% впевнені, що ключ існує.

### Модифікація значення — get_mut

Для зміни значення потрібне мутабельне посилання:

```rust
use std::collections::HashMap;

fn main() {
    let mut scores = HashMap::new();
    scores.insert("Blue", 10);
    
    // get_mut повертає Option<&mut V>
    if let Some(score) = scores.get_mut("Blue") {
        *score += 100;  // Додаємо 100 до Blue
    }
    
    println!("Blue after +100: {:?}", scores.get("Blue"));  // Some(&110)
}
```

### Видалення — remove

```rust
use std::collections::HashMap;

fn main() {
    let mut map = HashMap::new();
    map.insert("a", 1);
    map.insert("b", 2);
    map.insert("c", 3);
    
    // remove — видаляє і повертає значення
    let removed = map.remove("b");
    println!("Removed: {:?}", removed);  // Some(2)
    println!("Map: {:?}", map);  // {"a": 1, "c": 3}
    
    // remove неіснуючого ключа
    let nothing = map.remove("z");
    println!("Remove 'z': {:?}", nothing);  // None
    
    // remove_entry — повертає і ключ, і значення
    let entry = map.remove_entry("a");
    println!("Removed entry: {:?}", entry);  // Some(("a", 1))
}
```

### Фільтрація — retain

Метод `retain` залишає тільки елементи, що задовольняють умову:

```rust
use std::collections::HashMap;

fn main() {
    let mut scores: HashMap<&str, i32> = HashMap::from([
        ("Alice", 95),
        ("Bob", 45),
        ("Charlie", 78),
        ("Diana", 32),
    ]);
    
    // Залишаємо тільки тих, хто склав (>= 60)
    scores.retain(|_name, &mut score| score >= 60);
    
    println!("Passed: {:?}", scores);  // {"Alice": 95, "Charlie": 78}
}
```

### Перевірка наявності

```rust
use std::collections::HashMap;

fn main() {
    let map = HashMap::from([("a", 1), ("b", 2)]);
    
    println!("len: {}", map.len());           // 2
    println!("is_empty: {}", map.is_empty()); // false
    println!("contains 'a': {}", map.contains_key("a"));  // true
    println!("contains 'z': {}", map.contains_key("z"));  // false
}
```

---

## 17.4 Entry API — потужний інтерфейс

Entry API — це елегантний спосіб виконувати умовні операції: "вставити, якщо немає", "оновити, якщо є", "вставити або оновити".

### Проблема без Entry API

Уявіть задачу: підрахувати частоту слів у тексті. Без Entry API код виглядає громіздко:

```rust
use std::collections::HashMap;

fn main() {
    let text = "hello world hello rust world rust rust";
    let mut word_count: HashMap<String, i32> = HashMap::new();
    
    // ❌ Громіздкий підхід
    for word in text.split_whitespace() {
        if word_count.contains_key(word) {
            // Ключ є — збільшуємо лічильник
            let count = word_count.get_mut(word).unwrap();
            *count += 1;
        } else {
            // Ключа немає — створюємо
            word_count.insert(word.to_string(), 1);
        }
    }
    
    println!("{:?}", word_count);
}
```

Це працює, але код надмірний: перевірка, потім get_mut або insert.

### Рішення з Entry API

```rust
use std::collections::HashMap;

fn main() {
    let text = "hello world hello rust world rust rust";
    let mut word_count: HashMap<String, i32> = HashMap::new();
    
    // ✅ Елегантний підхід з Entry
    for word in text.split_whitespace() {
        let count = word_count.entry(word.to_string()).or_insert(0);
        *count += 1;
    }
    
    println!("{:?}", word_count);
    // {"hello": 2, "world": 2, "rust": 3}
}
```

Що тут відбувається?

1. `entry(key)` повертає `Entry` — enum з варіантами `Occupied` (ключ є) або `Vacant` (ключа немає)
2. `or_insert(0)` — якщо ключа немає, вставляє 0 і повертає `&mut V`; якщо є — просто повертає `&mut V`
3. Результат — мутабельне посилання на значення, яке ми збільшуємо

### Варіанти Entry API

**or_insert** — вставляє константу:
```rust
let count = map.entry(key).or_insert(0);
```

**or_insert_with** — ліниве обчислення (функція викликається тільки якщо ключа немає):
```rust
use std::collections::HashMap;

fn main() {
    let mut cache: HashMap<String, Vec<i32>> = HashMap::new();
    
    // expensive_compute викликається тільки для нових ключів
    let data = cache
        .entry("data".to_string())
        .or_insert_with(|| {
            println!("Computing expensive data...");
            vec![1, 2, 3, 4, 5]
        });
    
    println!("Data: {:?}", data);
    
    // Другий раз — не обчислює
    let data2 = cache
        .entry("data".to_string())
        .or_insert_with(|| {
            println!("This won't print!");
            vec![]
        });
    
    println!("Data2: {:?}", data2);  // Той самий вектор
}
```

**or_default** — вставляє `Default::default()`:
```rust
use std::collections::HashMap;

fn main() {
    let mut counts: HashMap<char, i32> = HashMap::new();
    
    for c in "hello".chars() {
        // i32::default() = 0
        *counts.entry(c).or_default() += 1;
    }
    
    println!("{:?}", counts);  // {'h': 1, 'e': 1, 'l': 2, 'o': 1}
}
```

**and_modify** — модифікує існуюче значення:
```rust
use std::collections::HashMap;

fn main() {
    let mut scores: HashMap<&str, i32> = HashMap::new();
    scores.insert("player1", 100);
    
    // Якщо player1 є — додаємо бонус, інакше — стартове значення
    scores.entry("player1")
        .and_modify(|s| *s += 50)
        .or_insert(0);
    
    // player2 немає — and_modify не виконується
    scores.entry("player2")
        .and_modify(|s| *s += 50)
        .or_insert(0);
    
    println!("{:?}", scores);  // {"player1": 150, "player2": 0}
}
```

### Entry як enum

Під капотом Entry — це enum:

```rust
use std::collections::HashMap;
use std::collections::hash_map::Entry;

fn main() {
    let mut map: HashMap<&str, i32> = HashMap::new();
    map.insert("existing", 42);
    
    match map.entry("existing") {
        Entry::Occupied(entry) => {
            println!("Ключ існує, значення: {}", entry.get());
        }
        Entry::Vacant(entry) => {
            println!("Ключа немає, вставляємо...");
            entry.insert(0);
        }
    }
    
    match map.entry("new_key") {
        Entry::Occupied(entry) => {
            println!("Існує: {}", entry.get());
        }
        Entry::Vacant(entry) => {
            println!("Вставляємо new_key");
            entry.insert(100);
        }
    }
    
    println!("{:?}", map);
}
```

---

## 17.5 Ітерація по HashMap

HashMap надає кілька способів ітерації.

### Ітерація по парах

```rust
use std::collections::HashMap;

fn main() {
    let scores = HashMap::from([
        ("Alice", 100),
        ("Bob", 85),
        ("Charlie", 92),
    ]);
    
    // &scores — ітерація по &(&K, &V)
    println!("Всі результати:");
    for (name, score) in &scores {
        println!("  {}: {}", name, score);
    }
    
    // Мутабельна ітерація — можна змінювати значення
    let mut scores_mut = scores.clone();
    for (_, score) in &mut scores_mut {
        *score += 10;  // Бонус всім
    }
    println!("\nПісля бонусу: {:?}", scores_mut);
    
    // into_iter — споживає HashMap
    for (name, score) in scores {
        println!("Consumed: {} = {}", name, score);
    }
    // scores більше не доступний
}
```

**Важливо:** порядок ітерації по HashMap **не гарантований**! Елементи можуть йти в будь-якому порядку, і порядок може змінюватись між запусками.

### Окремо ключі та значення

```rust
use std::collections::HashMap;

fn main() {
    let scores = HashMap::from([
        ("Alice", 100),
        ("Bob", 85),
        ("Charlie", 92),
    ]);
    
    // keys() — ітератор по ключах
    println!("Гравці:");
    for name in scores.keys() {
        println!("  {}", name);
    }
    
    // values() — ітератор по значеннях
    let total: i32 = scores.values().sum();
    let average = total as f64 / scores.len() as f64;
    println!("Середній бал: {:.1}", average);
    
    // values_mut() — мутабельний ітератор значень
    let mut scores_mut = scores.clone();
    for score in scores_mut.values_mut() {
        *score = (*score as f64 * 1.1) as i32;  // +10%
    }
    println!("Після підвищення: {:?}", scores_mut);
    
    // Збір ключів у вектор
    let names: Vec<_> = scores.keys().collect();
    println!("Імена: {:?}", names);
}
```

### Фільтрація та трансформація

```rust
use std::collections::HashMap;

fn main() {
    let scores = HashMap::from([
        ("Alice", 100),
        ("Bob", 65),
        ("Charlie", 92),
        ("Diana", 45),
    ]);
    
    // Фільтрація — нова HashMap з тих, хто склав
    let passed: HashMap<_, _> = scores.iter()
        .filter(|(_, &score)| score >= 70)
        .map(|(&name, &score)| (name, score))
        .collect();
    
    println!("Склали: {:?}", passed);
    
    // Трансформація — оцінки в літери
    let grades: HashMap<_, _> = scores.iter()
        .map(|(&name, &score)| {
            let grade = match score {
                90..=100 => "A",
                80..=89 => "B",
                70..=79 => "C",
                60..=69 => "D",
                _ => "F",
            };
            (name, grade)
        })
        .collect();
    
    println!("Оцінки: {:?}", grades);
}
```

---

## 17.6 HashSet — множина унікальних елементів

`HashSet<T>` — це, по суті, `HashMap<T, ()>` — зберігає тільки ключі, без значень. Ідеально для:
- Перевірки "чи елемент у множині" за O(1)
- Операцій над множинами (об'єднання, перетин, різниця)
- Видалення дублікатів

### Базове використання

```rust
use std::collections::HashSet;

fn main() {
    let mut visited: HashSet<(i32, i32)> = HashSet::new();
    
    // insert — додає елемент, повертає true якщо елемент новий
    println!("Insert (0,0): {}", visited.insert((0, 0)));  // true
    println!("Insert (1,0): {}", visited.insert((1, 0)));  // true
    println!("Insert (0,0): {}", visited.insert((0, 0)));  // false — дублікат
    
    println!("Кількість: {}", visited.len());  // 2
    
    // contains — перевірка наявності
    println!("Є (1,0): {}", visited.contains(&(1, 0)));  // true
    println!("Є (5,5): {}", visited.contains(&(5, 5)));  // false
    
    // remove — видаляє елемент
    visited.remove(&(1, 0));
    println!("Після remove: {:?}", visited);
}
```

### Операції над множинами

HashSet підтримує класичні операції теорії множин:

```rust
use std::collections::HashSet;

fn main() {
    let a: HashSet<i32> = [1, 2, 3, 4, 5].into_iter().collect();
    let b: HashSet<i32> = [4, 5, 6, 7, 8].into_iter().collect();
    
    println!("A = {:?}", a);
    println!("B = {:?}", b);
    
    // Об'єднання: A ∪ B — всі елементи з обох
    let union: HashSet<_> = a.union(&b).copied().collect();
    println!("A ∪ B = {:?}", union);  // {1,2,3,4,5,6,7,8}
    
    // Перетин: A ∩ B — спільні елементи
    let intersection: HashSet<_> = a.intersection(&b).copied().collect();
    println!("A ∩ B = {:?}", intersection);  // {4, 5}
    
    // Різниця: A - B — елементи в A, яких немає в B
    let diff: HashSet<_> = a.difference(&b).copied().collect();
    println!("A - B = {:?}", diff);  // {1, 2, 3}
    
    // Симетрична різниця: A △ B — елементи, що є тільки в одній множині
    let sym_diff: HashSet<_> = a.symmetric_difference(&b).copied().collect();
    println!("A △ B = {:?}", sym_diff);  // {1,2,3,6,7,8}
    
    // Перевірки відносин
    let subset: HashSet<i32> = [1, 2].into_iter().collect();
    println!("{:?} ⊆ A: {}", subset, subset.is_subset(&a));  // true
    println!("A disjoint B: {}", a.is_disjoint(&b));  // false (є спільні)
}
```

### HashSet для агента — відстеження відвіданих клітинок

```rust
use std::collections::HashSet;

/// Координати клітинки на сітці
#[derive(Debug, Clone, Copy, Hash, Eq, PartialEq)]
struct Cell {
    x: i32,
    y: i32,
}

impl Cell {
    fn new(x: i32, y: i32) -> Self {
        Self { x, y }
    }
    
    /// Сусідні клітинки (4 напрямки)
    fn neighbors(&self) -> Vec<Cell> {
        vec![
            Cell::new(self.x - 1, self.y),
            Cell::new(self.x + 1, self.y),
            Cell::new(self.x, self.y - 1),
            Cell::new(self.x, self.y + 1),
        ]
    }
}

/// Трекер дослідження території
struct ExplorationTracker {
    visited: HashSet<Cell>,
    obstacles: HashSet<Cell>,
}

impl ExplorationTracker {
    fn new() -> Self {
        Self {
            visited: HashSet::new(),
            obstacles: HashSet::new(),
        }
    }
    
    fn visit(&mut self, cell: Cell) {
        self.visited.insert(cell);
    }
    
    fn mark_obstacle(&mut self, cell: Cell) {
        self.obstacles.insert(cell);
    }
    
    fn is_visited(&self, cell: &Cell) -> bool {
        self.visited.contains(cell)
    }
    
    /// Невідвідані прохідні сусіди — кандидати для дослідження
    fn unvisited_neighbors(&self, cell: &Cell) -> Vec<Cell> {
        cell.neighbors()
            .into_iter()
            .filter(|c| !self.visited.contains(c) && !self.obstacles.contains(c))
            .collect()
    }
    
    fn progress(&self, total_area: usize) -> f64 {
        self.visited.len() as f64 / total_area as f64 * 100.0
    }
}

fn main() {
    let mut tracker = ExplorationTracker::new();
    
    // Позначаємо перешкоди
    tracker.mark_obstacle(Cell::new(2, 0));
    tracker.mark_obstacle(Cell::new(2, 1));
    
    // Симуляція руху
    let path = [
        Cell::new(0, 0),
        Cell::new(1, 0),
        Cell::new(1, 1),
        Cell::new(0, 1),
    ];
    
    for cell in path {
        tracker.visit(cell);
        let candidates = tracker.unvisited_neighbors(&cell);
        println!("At {:?}, можна піти до: {:?}", cell, candidates);
    }
    
    println!("\nВідвідано: {} клітинок", tracker.visited.len());
    println!("Перешкод: {}", tracker.obstacles.len());
}
```

---

## 17.7 Власні типи як ключі

Щоб використовувати власну структуру як ключ HashMap, потрібно реалізувати `Hash` та `Eq`.

### Автоматична реалізація через derive

Найпростіший спосіб — `#[derive(Hash, Eq, PartialEq)]`:

```rust
use std::collections::HashMap;

#[derive(Debug, Clone, Copy, Hash, Eq, PartialEq)]
struct SectorCoord {
    x: i32,
    y: i32,
}

impl SectorCoord {
    fn new(x: i32, y: i32) -> Self {
        Self { x, y }
    }
}

fn main() {
    let mut terrain: HashMap<SectorCoord, &str> = HashMap::new();
    
    terrain.insert(SectorCoord::new(0, 0), "grass");
    terrain.insert(SectorCoord::new(1, 0), "water");
    terrain.insert(SectorCoord::new(0, 1), "forest");
    
    let coord = SectorCoord::new(1, 0);
    println!("At {:?}: {:?}", coord, terrain.get(&coord));
}
```

**Важливо:** всі поля структури повинні реалізовувати `Hash` та `Eq`. Якщо є `f64` поле — derive не спрацює.

### Складені ключі

Ключем може бути складена структура:

```rust
use std::collections::HashMap;

#[derive(Debug, Clone, Hash, Eq, PartialEq)]
struct EventKey {
    agent_id: String,
    timestamp: u64,
}

#[derive(Debug)]
struct Event {
    description: String,
    severity: u8,
}

fn main() {
    let mut events: HashMap<EventKey, Event> = HashMap::new();
    
    let key = EventKey {
        agent_id: "SCOUT-001".to_string(),
        timestamp: 1000,
    };
    
    events.insert(key.clone(), Event {
        description: "Takeoff completed".to_string(),
        severity: 1,
    });
    
    println!("Event: {:?}", events.get(&key));
}
```

---

## 17.8 Практичне застосування: WorldMap

Тепер побудуємо карту світу для агента БПЛА. Ця програма демонструє комплексне використання HashMap та HashSet для зберігання та аналізу просторових даних.

**Постановка задачі:** Агент досліджує територію, поділену на сектори. Кожен сектор має тип поверхні (ліс, вода, будівля тощо). Потрібно:
- Швидко отримувати інформацію про будь-який сектор
- Відстежувати досліджені та недосліджені області
- Знаходити прохідні шляхи
- Позначати зони загрози

```rust
use std::collections::{HashMap, HashSet};

/// Координати сектора
#[derive(Debug, Clone, Copy, Hash, Eq, PartialEq)]
pub struct SectorCoord {
    pub x: i32,
    pub y: i32,
}

impl SectorCoord {
    pub fn new(x: i32, y: i32) -> Self {
        Self { x, y }
    }
    
    /// Сусідні сектори (4 напрямки)
    pub fn neighbors(&self) -> Vec<SectorCoord> {
        vec![
            SectorCoord::new(self.x - 1, self.y),
            SectorCoord::new(self.x + 1, self.y),
            SectorCoord::new(self.x, self.y - 1),
            SectorCoord::new(self.x, self.y + 1),
        ]
    }
    
    /// Манхеттенська відстань до іншого сектора
    pub fn distance_to(&self, other: &SectorCoord) -> i32 {
        (self.x - other.x).abs() + (self.y - other.y).abs()
    }
}

/// Тип поверхні сектора
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Terrain {
    Unknown,     // Не досліджено
    Clear,       // Вільний простір
    Water,       // Вода
    Forest,      // Ліс
    Building,    // Будівля
    NoFlyZone,   // Заборонена зона
}

impl Terrain {
    /// Чи можна пролетіти через цей тип
    pub fn is_flyable(&self) -> bool {
        matches!(self, Terrain::Clear | Terrain::Water | Terrain::Forest | Terrain::Unknown)
    }
    
    /// Вартість проходження для pathfinding
    pub fn cost(&self) -> f64 {
        match self {
            Terrain::Clear => 1.0,
            Terrain::Water => 1.2,
            Terrain::Forest => 1.5,
            Terrain::Unknown => 2.0,  // Невідоме — обережніше
            _ => f64::INFINITY,        // Непрохідне
        }
    }
}

impl Default for Terrain {
    fn default() -> Self {
        Terrain::Unknown
    }
}

/// Інформація про сектор
#[derive(Debug, Clone)]
pub struct SectorInfo {
    pub terrain: Terrain,
    pub threat_level: u8,  // 0-10
    pub scanned_at: Option<f64>,
}

impl SectorInfo {
    pub fn new(terrain: Terrain) -> Self {
        Self {
            terrain,
            threat_level: 0,
            scanned_at: None,
        }
    }
    
    pub fn with_scan_time(mut self, time: f64) -> Self {
        self.scanned_at = Some(time);
        self
    }
}

impl Default for SectorInfo {
    fn default() -> Self {
        Self::new(Terrain::Unknown)
    }
}

/// Карта світу агента
pub struct WorldMap {
    sectors: HashMap<SectorCoord, SectorInfo>,
    explored: HashSet<SectorCoord>,
    sector_size: f64,  // метрів на сектор
}

impl WorldMap {
    pub fn new(sector_size: f64) -> Self {
        Self {
            sectors: HashMap::new(),
            explored: HashSet::new(),
            sector_size,
        }
    }
    
    /// Конвертує світові координати в координати сектора
    pub fn world_to_sector(&self, x: f64, y: f64) -> SectorCoord {
        SectorCoord::new(
            (x / self.sector_size).floor() as i32,
            (y / self.sector_size).floor() as i32,
        )
    }
    
    /// Отримує тип поверхні сектора
    pub fn get_terrain(&self, coord: &SectorCoord) -> Terrain {
        self.sectors.get(coord)
            .map(|s| s.terrain)
            .unwrap_or(Terrain::Unknown)
    }
    
    /// Сканує сектор — оновлює інформацію
    pub fn scan(&mut self, coord: SectorCoord, terrain: Terrain, time: f64) {
        let info = SectorInfo::new(terrain).with_scan_time(time);
        self.sectors.insert(coord, info);
        self.explored.insert(coord);
    }
    
    /// Позначає рівень загрози
    pub fn mark_threat(&mut self, coord: SectorCoord, level: u8) {
        self.sectors
            .entry(coord)
            .or_default()
            .threat_level = level.min(10);
    }
    
    /// Чи досліджено сектор
    pub fn is_explored(&self, coord: &SectorCoord) -> bool {
        self.explored.contains(coord)
    }
    
    /// Чи прохідний сектор
    pub fn is_passable(&self, coord: &SectorCoord) -> bool {
        self.get_terrain(coord).is_flyable()
    }
    
    /// Прохідні сусіди для планування маршруту
    pub fn passable_neighbors(&self, coord: &SectorCoord) -> Vec<SectorCoord> {
        coord.neighbors()
            .into_iter()
            .filter(|c| self.is_passable(c))
            .collect()
    }
    
    /// Frontier — недосліджені сектори на межі відомої території
    pub fn frontier(&self) -> Vec<SectorCoord> {
        let mut result = HashSet::new();
        
        for explored in &self.explored {
            for neighbor in explored.neighbors() {
                if !self.is_explored(&neighbor) {
                    result.insert(neighbor);
                }
            }
        }
        
        result.into_iter().collect()
    }
    
    /// Сектори з загрозою вище порогу
    pub fn threat_zones(&self, min_level: u8) -> Vec<SectorCoord> {
        self.sectors.iter()
            .filter(|(_, info)| info.threat_level >= min_level)
            .map(|(coord, _)| *coord)
            .collect()
    }
    
    /// Статистика карти
    pub fn stats(&self) -> MapStats {
        let mut stats = MapStats::default();
        
        stats.total = self.sectors.len();
        stats.explored = self.explored.len();
        
        for info in self.sectors.values() {
            match info.terrain {
                Terrain::Clear => stats.clear += 1,
                Terrain::Water => stats.water += 1,
                Terrain::Forest => stats.forest += 1,
                Terrain::Building => stats.buildings += 1,
                Terrain::NoFlyZone => stats.no_fly += 1,
                Terrain::Unknown => stats.unknown += 1,
            }
        }
        
        stats
    }
}

#[derive(Debug, Default)]
pub struct MapStats {
    pub total: usize,
    pub explored: usize,
    pub clear: usize,
    pub water: usize,
    pub forest: usize,
    pub buildings: usize,
    pub no_fly: usize,
    pub unknown: usize,
}

fn main() {
    println!("╔══════════════════════════════════════╗");
    println!("║       КАРТА СВІТУ АГЕНТА             ║");
    println!("╚══════════════════════════════════════╝\n");
    
    let mut world = WorldMap::new(10.0);  // 10м на сектор
    let time = 0.0;
    
    // Симуляція сканування
    let scan_data = [
        (SectorCoord::new(0, 0), Terrain::Clear),
        (SectorCoord::new(1, 0), Terrain::Clear),
        (SectorCoord::new(2, 0), Terrain::Water),
        (SectorCoord::new(0, 1), Terrain::Forest),
        (SectorCoord::new(1, 1), Terrain::Clear),
        (SectorCoord::new(2, 1), Terrain::Building),
        (SectorCoord::new(0, 2), Terrain::Clear),
        (SectorCoord::new(1, 2), Terrain::NoFlyZone),
        (SectorCoord::new(2, 2), Terrain::Clear),
    ];
    
    println!("📡 Сканування...");
    for (coord, terrain) in scan_data {
        world.scan(coord, terrain, time);
        println!("  {:?} → {:?}", coord, terrain);
    }
    
    // Позначаємо загрозу
    world.mark_threat(SectorCoord::new(2, 1), 7);
    
    // Статистика
    println!("\n📊 Статистика:");
    let stats = world.stats();
    println!("  Всього секторів: {}", stats.total);
    println!("  Clear: {}, Water: {}, Forest: {}", 
             stats.clear, stats.water, stats.forest);
    println!("  Buildings: {}, No-fly: {}", 
             stats.buildings, stats.no_fly);
    
    // Frontier
    println!("\n🔍 Frontier (недосліджені сусіди):");
    for coord in world.frontier().iter().take(5) {
        println!("  {:?}", coord);
    }
    
    // Прохідні сусіди
    println!("\n🛤️ Прохідні шляхи від (1,1):");
    let center = SectorCoord::new(1, 1);
    for neighbor in world.passable_neighbors(&center) {
        let terrain = world.get_terrain(&neighbor);
        println!("  {:?} — {:?}", neighbor, terrain);
    }
    
    // Зони загрози
    println!("\n⚠️ Зони загрози (рівень >= 5):");
    for coord in world.threat_zones(5) {
        println!("  {:?}", coord);
    }
}
```

**Як працює ця програма:**

1. **SectorCoord** — координати сектора з derive(Hash, Eq, PartialEq), може бути ключем HashMap
2. **HashMap<SectorCoord, SectorInfo>** — швидкий доступ до інформації про сектор за O(1)
3. **HashSet<SectorCoord>** — множина досліджених секторів, швидка перевірка за O(1)
4. **Entry API** (`or_default`) — для mark_threat, якщо сектора ще немає
5. **frontier()** — використовує HashSet для збору унікальних недосліджених сусідів
6. **Ітерація з фільтрацією** — threat_zones використовує iter().filter()

---

## 17.9 Лабораторна робота

**Завдання:** Створіть реєстр об'єктів для агента БПЛА.

Агент виявляє об'єкти (транспорт, люди, будівлі) і потребує:
- Швидкого пошуку за ID
- Фільтрації за категорією
- Пошуку в радіусі від точки
- Видалення застарілих записів

**Структури:**
```rust
struct ObjectId(u32);
enum Category { Vehicle, Person, Building, Unknown }
struct TrackedObject {
    id: ObjectId,
    category: Category,
    position: (f64, f64, f64),
    last_seen: f64,
    confidence: f64,
}
struct ObjectRegistry { /* ... */ }
```

**Методи ObjectRegistry:**
- `register()` — додає новий об'єкт
- `get()` — отримує за ID
- `by_category()` — всі об'єкти категорії
- `in_radius()` — об'єкти в радіусі
- `remove_stale()` — видаляє застарілі

**Критерії оцінювання:**

| Критерій | Бали |
|----------|------|
| Структури та enum | 2 |
| HashMap для зберігання | 2 |
| Метод register | 1 |
| Метод by_category | 1 |
| Метод in_radius | 2 |
| Метод remove_stale | 2 |
| **Максимум** | **10** |

---

## Підсумок

`HashMap<K, V>` — це словник з доступом за ключем за O(1). Ви навчились:

- **Створювати** HashMap різними способами
- **Маніпулювати** елементами: insert, get, remove, retain
- **Використовувати Entry API** для елегантних умовних операцій
- **Ітерувати** по ключах, значеннях, парах
- **Працювати з HashSet** для множин унікальних елементів
- **Створювати власні ключі** через derive(Hash, Eq, PartialEq)
- **Будувати карту світу** агента

**Ключові моменти:**
- HashMap не гарантує порядок елементів
- Ключ повинен реалізувати Hash + Eq
- `f64` не може бути ключем (через NaN)
- Entry API — найелегантніший спосіб для "insert or update"
- HashSet — HashMap без значень, для множин

---

## Зв'язок з наступним розділом

Тепер ваш агент має динамічні колекції: Vec для історії, HashMap для карти світу. Але що робити, коли щось йде не так? Сенсор не відповідає. Файл не знайдено. Команда невалідна.

У **Розділі 18: Обробка помилок — Option та Result** ви навчитесь елегантно обробляти відсутність даних та помилки операцій, перетворюючи агента з "оптимістичного" на "робастного".
