# Практичне заняття 12: Lifetimes (час життя посилань)

## Мета заняття

Після цього заняття ви зможете:
- Розуміти, що таке lifetime та навіщо він потрібен
- Читати та писати lifetime annotations (`'a`, `'b`, `'static`)
- Застосовувати правила lifetime elision
- Створювати структури з посиланнями
- Вирішувати типові проблеми з lifetimes

---

## Теоретичний вступ

### Проблема: dangling references

Rust гарантує відсутність **dangling references** — посилань на дані, що вже не існують:

```rust
fn main() {
    let r;                // ─────────────────┐-- 'a
    {                     //                  │
        let x = 5;        // ─┐-- 'b          │
        r = &x;           //  │               │
    }                     // ─┘ x dropped     │
    // println!("{}", r); // ПОМИЛКА! r посилається на звільнену пам'ять
}                         // ─────────────────┘
```

**Lifetime** — це область коду, протягом якої посилання є валідним.

### Що таке Lifetime?

**Lifetime** — це не час виконання програми, а **область видимості** (scope), в якій посилання гарантовано вказує на валідні дані.

```rust
fn main() {
    let x = 5;            // ─────────┐-- lifetime of x
    let r = &x;           // ─┐-- 'a  │
    println!("{}", r);    //  │       │
                          // ─┘       │
}                         // ─────────┘
```

### Коли потрібні lifetime annotations?

У більшості випадків Rust **виводить lifetimes автоматично**. Анотації потрібні, коли:

1. Функція приймає кілька посилань і повертає посилання
2. Структура містить посилання
3. Компілятор не може визначити зв'язки між lifetimes

```rust
// Компілятор НЕ ЗНАЄ, чий lifetime у результату
fn longest(x: &str, y: &str) -> &str {  // ПОМИЛКА!
    if x.len() > y.len() { x } else { y }
}

// Явно вказуємо: результат живе стільки, скільки x і y
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

---

## Синтаксис Lifetime Annotations

### Базовий синтаксис

```rust
&i32        // посилання
&'a i32     // посилання з явним lifetime 'a
&'a mut i32 // мутабельне посилання з lifetime 'a
```

### Lifetime у функціях

```rust
// Один lifetime параметр
fn first_word<'a>(s: &'a str) -> &'a str {
    s.split_whitespace().next().unwrap_or("")
}

// Кілька lifetime параметрів
fn longest<'a, 'b>(x: &'a str, y: &'b str) -> &'a str {
    x  // Повертаємо тільки x, тому результат прив'язаний до 'a
}

// Lifetime + generics
fn longest_with_announcement<'a, T>(x: &'a str, y: &'a str, ann: T) -> &'a str
where
    T: std::fmt::Display,
{
    println!("Announcement: {}", ann);
    if x.len() > y.len() { x } else { y }
}
```

### Lifetime у структурах

```rust
// Структура, що містить посилання
struct Excerpt<'a> {
    text: &'a str,
}

impl<'a> Excerpt<'a> {
    fn level(&self) -> i32 {
        3
    }
    
    // self має implicit lifetime
    fn announce(&self, message: &str) -> &str {
        println!("{}", message);
        self.text
    }
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first_sentence = novel.split('.').next().unwrap();
    
    let excerpt = Excerpt {
        text: first_sentence,
    };
    
    println!("Excerpt: {}", excerpt.text);
}
```

---

## Lifetime Elision Rules

Rust автоматично виводить lifetimes за трьома правилами:

### Правило 1: Кожне посилання-параметр отримує свій lifetime

```rust
fn foo(x: &str)           -> fn foo<'a>(x: &'a str)
fn foo(x: &str, y: &str)  -> fn foo<'a, 'b>(x: &'a str, y: &'b str)
```

### Правило 2: Якщо один вхідний lifetime — він присвоюється всім вихідним

```rust
fn foo(x: &str) -> &str   -> fn foo<'a>(x: &'a str) -> &'a str
```

### Правило 3: Якщо є &self або &mut self — їх lifetime присвоюється вихідним

```rust
impl Foo {
    fn method(&self, x: &str) -> &str
    // стає:
    fn method<'a, 'b>(&'a self, x: &'b str) -> &'a str
}
```

### Коли elision НЕ працює

```rust
// Два параметри, повертаємо посилання — неоднозначно!
fn longest(x: &str, y: &str) -> &str  // ПОМИЛКА

// Потрібна явна анотація
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str  // OK
```

---

## 'static Lifetime

`'static` — спеціальний lifetime, що означає "весь час виконання програми":

```rust
// String literals мають 'static lifetime
let s: &'static str = "Hello, world!";

// Константи
static GREETING: &str = "Hello!";

// У trait bounds
fn print_it<T: std::fmt::Display + 'static>(input: T) {
    println!("{}", input);
}
```

**Увага:** `'static` не означає "вічне життя", а "може жити вічно, якщо потрібно".

---

## Типові патерни та помилки

### Помилка 1: Повернення посилання на локальну змінну

```rust
fn create_string() -> &str {
    let s = String::from("hello");
    &s  // ПОМИЛКА! s буде знищено
}

// Рішення 1: Повертати owned тип
fn create_string() -> String {
    String::from("hello")
}

// Рішення 2: Приймати посилання ззовні
fn modify_string(s: &mut String) -> &str {
    s.push_str(" world");
    s
}
```

### Помилка 2: Конфліктуючі lifetimes

```rust
fn main() {
    let mut data = vec![1, 2, 3];
    let first = &data[0];  // immutable borrow
    data.push(4);          // ПОМИЛКА! mutable borrow
    println!("{}", first); // immutable borrow used here
}

// Рішення: обмежити scope
fn main() {
    let mut data = vec![1, 2, 3];
    {
        let first = &data[0];
        println!("{}", first);
    }  // immutable borrow ends
    data.push(4);  // OK
}
```

### Помилка 3: Структура переживає дані

```rust
struct Holder<'a> {
    data: &'a str,
}

fn main() {
    let holder;
    {
        let s = String::from("hello");
        holder = Holder { data: &s };  // ПОМИЛКА!
    }  // s dropped, але holder.data посилається на неї
    // println!("{}", holder.data);
}
```

### Патерн: Множинні lifetimes

```rust
// Різні lifetimes для різних полів
struct Context<'s, 'c> {
    config: &'s str,
    cache: &'c str,
}

// Результат прив'язаний до коротшого lifetime
fn shortest<'a, 'b>(x: &'a str, y: &'b str) -> &'a str
where
    'b: 'a,  // 'b живе не менше ніж 'a
{
    if x.len() < y.len() { x } else { x }
}
```

### Патерн: Lifetime з Option

```rust
struct MaybeRef<'a> {
    data: Option<&'a str>,
}

impl<'a> MaybeRef<'a> {
    fn new() -> Self {
        MaybeRef { data: None }
    }
    
    fn set(&mut self, data: &'a str) {
        self.data = Some(data);
    }
    
    fn get(&self) -> Option<&str> {
        self.data
    }
}
```

---

## Lifetime Bounds

### Lifetime як обмеження на generic типи

```rust
// T має жити не менше ніж 'a
fn print_ref<'a, T>(t: &'a T)
where
    T: std::fmt::Debug + 'a,
{
    println!("{:?}", t);
}

// T: 'static — T не містить non-static посилань
fn must_be_static<T: 'static>(t: T) {
    // T можна безпечно зберегти "назавжди"
}
```

### Lifetime у trait implementations

```rust
trait Parser<'a> {
    fn parse(&self, input: &'a str) -> &'a str;
}

struct SimpleParser;

impl<'a> Parser<'a> for SimpleParser {
    fn parse(&self, input: &'a str) -> &'a str {
        input.trim()
    }
}
```

---

## Практичні задачі

### Задача 1: Функції з посиланнями

**Умова:** Напишіть функції для роботи з текстом, що коректно використовують lifetimes: пошук найдовшого слова, знаходження спільного префіксу, розбиття на частини.

**Розв'язання:**

```rust
// Найдовше слово у рядку
fn longest_word<'a>(text: &'a str) -> &'a str {
    text.split_whitespace()
        .max_by_key(|word| word.len())
        .unwrap_or("")
}

// Найдовший з двох рядків
fn longer<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

// Спільний префікс
fn common_prefix<'a>(s1: &'a str, s2: &str) -> &'a str {
    let len = s1.chars()
        .zip(s2.chars())
        .take_while(|(a, b)| a == b)
        .count();
    
    &s1[..s1.chars().take(len).map(|c| c.len_utf8()).sum()]
}

// Перше слово (до пробілу)
fn first_word<'a>(s: &'a str) -> &'a str {
    match s.find(' ') {
        Some(pos) => &s[..pos],
        None => s,
    }
}

// Останнє слово
fn last_word<'a>(s: &'a str) -> &'a str {
    match s.rfind(' ') {
        Some(pos) => &s[pos + 1..],
        None => s,
    }
}

// Слово за індексом
fn word_at<'a>(text: &'a str, index: usize) -> Option<&'a str> {
    text.split_whitespace().nth(index)
}

// Підрядок між маркерами
fn between_markers<'a>(text: &'a str, start: &str, end: &str) -> Option<&'a str> {
    let start_pos = text.find(start)? + start.len();
    let end_pos = text[start_pos..].find(end)? + start_pos;
    Some(&text[start_pos..end_pos])
}

fn main() {
    println!("=== Функції з посиланнями ===\n");
    
    let text = "The quick brown fox jumps over the lazy dog";
    
    println!("Текст: '{}'", text);
    println!("Найдовше слово: '{}'", longest_word(text));
    println!("Перше слово: '{}'", first_word(text));
    println!("Останнє слово: '{}'", last_word(text));
    println!("Слово #3: {:?}", word_at(text, 3));
    
    println!("\n--- Порівняння ---");
    let s1 = "hello";
    let s2 = "hello world";
    println!("Довше з '{}' та '{}': '{}'", s1, s2, longer(s1, s2));
    
    println!("\n--- Спільний префікс ---");
    let pairs = [
        ("hello", "helicopter"),
        ("rust", "rusty"),
        ("abc", "xyz"),
    ];
    for (a, b) in pairs {
        println!("'{}' та '{}' → '{}'", a, b, common_prefix(a, b));
    }
    
    println!("\n--- Між маркерами ---");
    let html = "<title>Hello World</title>";
    if let Some(title) = between_markers(html, "<title>", "</title>") {
        println!("Title: '{}'", title);
    }
    
    // Демонстрація lifetime
    println!("\n--- Lifetime demo ---");
    let result;
    {
        let local = String::from("temporary string");
        // result = first_word(&local);  // ПОМИЛКА! local буде знищено
        println!("Local first word: '{}'", first_word(&local));
    }
    
    let permanent = "permanent string";
    result = first_word(permanent);  // OK - string literal є 'static
    println!("Result: '{}'", result);
}
```

**Результат:**
```
=== Функції з посиланнями ===

Текст: 'The quick brown fox jumps over the lazy dog'
Найдовше слово: 'quick'
Перше слово: 'The'
Останнє слово: 'dog'
Слово #3: Some("fox")

--- Порівняння ---
Довше з 'hello' та 'hello world': 'hello world'

--- Спільний префікс ---
'hello' та 'helicopter' → 'hel'
'rust' та 'rusty' → 'rust'
'abc' та 'xyz' → ''

--- Між маркерами ---
Title: 'Hello World'
```

---

### Задача 2: Структура з посиланнями

**Умова:** Створіть структуру `DroneView<'a>`, що містить посилання на частини стану дрона. Це "вікно" для читання даних без копіювання.

**Розв'язання:**

```rust
#[derive(Debug)]
struct Drone {
    id: String,
    name: String,
    status: String,
    position: (f64, f64),
    battery: i32,
    log: Vec<String>,
}

impl Drone {
    fn new(id: &str, name: &str) -> Self {
        Drone {
            id: id.to_string(),
            name: name.to_string(),
            status: "idle".to_string(),
            position: (0.0, 0.0),
            battery: 100,
            log: Vec::new(),
        }
    }
    
    fn move_to(&mut self, x: f64, y: f64) {
        self.position = (x, y);
        self.battery -= 5;
        self.status = "moving".to_string();
        self.log.push(format!("Moved to ({:.1}, {:.1})", x, y));
    }
    
    fn add_log(&mut self, message: &str) {
        self.log.push(message.to_string());
    }
}

// View — легковагий "погляд" на дані дрона
#[derive(Debug)]
struct DroneView<'a> {
    id: &'a str,
    name: &'a str,
    status: &'a str,
    position: (f64, f64),
    battery: i32,
}

impl<'a> DroneView<'a> {
    fn from_drone(drone: &'a Drone) -> Self {
        DroneView {
            id: &drone.id,
            name: &drone.name,
            status: &drone.status,
            position: drone.position,
            battery: drone.battery,
        }
    }
    
    fn summary(&self) -> String {
        format!(
            "[{}] {} - {} at ({:.1}, {:.1}), battery: {}%",
            self.id, self.name, self.status,
            self.position.0, self.position.1, self.battery
        )
    }
    
    fn is_low_battery(&self) -> bool {
        self.battery < 20
    }
}

// LogView — погляд на логи
struct LogView<'a> {
    drone_id: &'a str,
    entries: &'a [String],
}

impl<'a> LogView<'a> {
    fn from_drone(drone: &'a Drone) -> Self {
        LogView {
            drone_id: &drone.id,
            entries: &drone.log,
        }
    }
    
    fn last_n(&self, n: usize) -> &[String] {
        let start = self.entries.len().saturating_sub(n);
        &self.entries[start..]
    }
    
    fn filter_contains(&self, pattern: &str) -> Vec<&'a str> {
        self.entries
            .iter()
            .filter(|e| e.contains(pattern))
            .map(|e| e.as_str())
            .collect()
    }
    
    fn print(&self) {
        println!("=== Logs for {} ({} entries) ===", self.drone_id, self.entries.len());
        for (i, entry) in self.entries.iter().enumerate() {
            println!("  [{}] {}", i + 1, entry);
        }
    }
}

// Функція, що працює з view
fn analyze_fleet<'a>(views: &[DroneView<'a>]) {
    println!("\n=== Fleet Analysis ===");
    println!("Total drones: {}", views.len());
    
    let low_battery: Vec<_> = views.iter()
        .filter(|v| v.is_low_battery())
        .collect();
    
    if !low_battery.is_empty() {
        println!("⚠️ Low battery:");
        for v in low_battery {
            println!("  {} - {}%", v.name, v.battery);
        }
    }
    
    let avg_battery: f64 = views.iter()
        .map(|v| v.battery as f64)
        .sum::<f64>() / views.len() as f64;
    
    println!("Average battery: {:.1}%", avg_battery);
}

fn main() {
    println!("=== Структура з посиланнями ===\n");
    
    // Створюємо дрони
    let mut drone1 = Drone::new("D001", "Alpha");
    let mut drone2 = Drone::new("D002", "Beta");
    let mut drone3 = Drone::new("D003", "Gamma");
    
    // Модифікуємо
    drone1.move_to(100.0, 50.0);
    drone1.move_to(150.0, 75.0);
    drone1.add_log("Target acquired");
    
    drone2.move_to(200.0, 200.0);
    drone2.battery = 15;  // Низький заряд для тесту
    
    drone3.add_log("System check OK");
    drone3.add_log("Mission started");
    
    // Створюємо views (без копіювання даних!)
    let view1 = DroneView::from_drone(&drone1);
    let view2 = DroneView::from_drone(&drone2);
    let view3 = DroneView::from_drone(&drone3);
    
    println!("--- Drone Views ---");
    println!("{}", view1.summary());
    println!("{}", view2.summary());
    println!("{}", view3.summary());
    
    // Аналіз флоту
    let fleet_views = vec![view1, view2, view3];
    analyze_fleet(&fleet_views);
    
    // Log view
    println!();
    let log_view = LogView::from_drone(&drone1);
    log_view.print();
    
    println!("\nFiltered (contains 'Moved'):");
    for entry in log_view.filter_contains("Moved") {
        println!("  {}", entry);
    }
    
    // Демонстрація: view не може пережити дані
    println!("\n--- Lifetime Safety ---");
    let view;
    {
        let temp_drone = Drone::new("TEMP", "Temporary");
        // view = DroneView::from_drone(&temp_drone);  // ПОМИЛКА!
        let temp_view = DroneView::from_drone(&temp_drone);
        println!("Temp view: {}", temp_view.summary());
    }
    // temp_drone знищено, temp_view теж
    
    // Але можна використовувати view з довшим lifetime
    view = DroneView::from_drone(&drone1);
    println!("Persistent view: {}", view.summary());
}
```

---

### Задача 3: Ітератор з lifetime

**Умова:** Створіть власний ітератор `WordIterator`, що повертає посилання на слова у тексті. Ітератор має коректно відстежувати lifetime вхідного рядка.

**Розв'язання:**

```rust
// Ітератор по словах
struct WordIterator<'a> {
    text: &'a str,
    position: usize,
}

impl<'a> WordIterator<'a> {
    fn new(text: &'a str) -> Self {
        WordIterator { text, position: 0 }
    }
}

impl<'a> Iterator for WordIterator<'a> {
    type Item = &'a str;
    
    fn next(&mut self) -> Option<Self::Item> {
        // Пропускаємо пробіли
        let remaining = &self.text[self.position..];
        let trimmed_start = remaining.len() - remaining.trim_start().len();
        self.position += trimmed_start;
        
        if self.position >= self.text.len() {
            return None;
        }
        
        let remaining = &self.text[self.position..];
        
        // Знаходимо кінець слова
        let word_end = remaining
            .find(char::is_whitespace)
            .unwrap_or(remaining.len());
        
        if word_end == 0 {
            return None;
        }
        
        let word = &self.text[self.position..self.position + word_end];
        self.position += word_end;
        
        Some(word)
    }
}

// Ітератор по рядках з номерами
struct NumberedLines<'a> {
    lines: std::str::Lines<'a>,
    line_number: usize,
}

impl<'a> NumberedLines<'a> {
    fn new(text: &'a str) -> Self {
        NumberedLines {
            lines: text.lines(),
            line_number: 0,
        }
    }
}

impl<'a> Iterator for NumberedLines<'a> {
    type Item = (usize, &'a str);
    
    fn next(&mut self) -> Option<Self::Item> {
        self.lines.next().map(|line| {
            self.line_number += 1;
            (self.line_number, line)
        })
    }
}

// Ітератор по парах (вікно розміром 2)
struct PairIterator<'a, T> {
    slice: &'a [T],
    position: usize,
}

impl<'a, T> PairIterator<'a, T> {
    fn new(slice: &'a [T]) -> Self {
        PairIterator { slice, position: 0 }
    }
}

impl<'a, T> Iterator for PairIterator<'a, T> {
    type Item = (&'a T, &'a T);
    
    fn next(&mut self) -> Option<Self::Item> {
        if self.position + 1 < self.slice.len() {
            let pair = (&self.slice[self.position], &self.slice[self.position + 1]);
            self.position += 1;
            Some(pair)
        } else {
            None
        }
    }
}

// Extension trait для зручності
trait IteratorExt<'a> {
    fn words(&'a self) -> WordIterator<'a>;
    fn numbered_lines(&'a self) -> NumberedLines<'a>;
}

impl<'a> IteratorExt<'a> for str {
    fn words(&'a self) -> WordIterator<'a> {
        WordIterator::new(self)
    }
    
    fn numbered_lines(&'a self) -> NumberedLines<'a> {
        NumberedLines::new(self)
    }
}

trait SliceExt<'a, T> {
    fn pairs(&'a self) -> PairIterator<'a, T>;
}

impl<'a, T> SliceExt<'a, T> for [T] {
    fn pairs(&'a self) -> PairIterator<'a, T> {
        PairIterator::new(self)
    }
}

fn main() {
    println!("=== Ітератор з lifetime ===\n");
    
    // WordIterator
    let text = "The quick brown fox jumps over the lazy dog";
    println!("Text: '{}'", text);
    
    println!("\n--- Words ---");
    for (i, word) in text.words().enumerate() {
        println!("  {}: '{}'", i, word);
    }
    
    // Фільтрація та збір
    let long_words: Vec<&str> = text.words()
        .filter(|w| w.len() > 4)
        .collect();
    println!("\nWords longer than 4 chars: {:?}", long_words);
    
    // NumberedLines
    let poem = "Roses are red
Violets are blue
Rust is awesome
And so are you";
    
    println!("\n--- Numbered Lines ---");
    for (num, line) in poem.numbered_lines() {
        println!("  {:2}: {}", num, line);
    }
    
    // PairIterator
    let numbers = vec![1, 2, 3, 4, 5];
    println!("\n--- Pairs ---");
    for (a, b) in numbers.pairs() {
        println!("  ({}, {})", a, b);
    }
    
    // Комбінування з іншими методами Iterator
    println!("\n--- Chained operations ---");
    let code = "fn main() {
    let x = 5;
    println!(x);
}";
    
    let non_empty: Vec<_> = code.numbered_lines()
        .filter(|(_, line)| !line.trim().is_empty())
        .map(|(num, line)| format!("{:2}| {}", num, line))
        .collect();
    
    for line in non_empty {
        println!("{}", line);
    }
    
    // Lifetime demo
    println!("\n--- Lifetime guarantee ---");
    let words: Vec<&str>;
    {
        let local_text = String::from("hello world from rust");
        let iter = local_text.words();
        // words = iter.collect();  // ПОМИЛКА! local_text буде знищено
        let local_words: Vec<&str> = iter.collect();
        println!("Local words: {:?}", local_words);
    }
    // local_text знищено
    
    // Але з 'static текстом все OK
    let static_text = "static text lives forever";
    words = static_text.words().collect();
    println!("Static words: {:?}", words);
}
```

---

### Задача 4: Кеш з lifetime

**Умова:** Створіть структуру `Cache<'a, K, V>`, що зберігає пари ключ-значення, де значення є посиланнями. Реалізуйте методи get, set, evict з коректними lifetimes.

**Розв'язання:**

```rust
use std::collections::HashMap;
use std::hash::Hash;

// Кеш, що зберігає посилання на значення
struct Cache<'a, K, V> {
    data: HashMap<K, &'a V>,
    capacity: usize,
    access_order: Vec<K>,  // Для LRU
}

impl<'a, K, V> Cache<'a, K, V>
where
    K: Eq + Hash + Clone,
{
    fn new(capacity: usize) -> Self {
        Cache {
            data: HashMap::new(),
            capacity,
            access_order: Vec::new(),
        }
    }
    
    fn get(&mut self, key: &K) -> Option<&'a V> {
        if let Some(&value) = self.data.get(key) {
            // Оновлюємо порядок доступу
            self.access_order.retain(|k| k != key);
            self.access_order.push(key.clone());
            Some(value)
        } else {
            None
        }
    }
    
    fn set(&mut self, key: K, value: &'a V) {
        // Якщо кеш повний — видаляємо найстаріший
        if self.data.len() >= self.capacity && !self.data.contains_key(&key) {
            self.evict_oldest();
        }
        
        self.access_order.retain(|k| k != &key);
        self.access_order.push(key.clone());
        self.data.insert(key, value);
    }
    
    fn evict_oldest(&mut self) {
        if let Some(oldest_key) = self.access_order.first().cloned() {
            self.data.remove(&oldest_key);
            self.access_order.remove(0);
        }
    }
    
    fn remove(&mut self, key: &K) -> Option<&'a V> {
        self.access_order.retain(|k| k != key);
        self.data.remove(key)
    }
    
    fn contains(&self, key: &K) -> bool {
        self.data.contains_key(key)
    }
    
    fn len(&self) -> usize {
        self.data.len()
    }
    
    fn is_empty(&self) -> bool {
        self.data.is_empty()
    }
    
    fn clear(&mut self) {
        self.data.clear();
        self.access_order.clear();
    }
    
    fn keys(&self) -> impl Iterator<Item = &K> {
        self.data.keys()
    }
    
    fn values(&self) -> impl Iterator<Item = &&'a V> {
        self.data.values()
    }
}

// Приклад: кешування конфігурацій дронів
#[derive(Debug)]
struct DroneConfig {
    name: String,
    max_altitude: i32,
    max_speed: f64,
}

impl DroneConfig {
    fn new(name: &str, altitude: i32, speed: f64) -> Self {
        DroneConfig {
            name: name.to_string(),
            max_altitude: altitude,
            max_speed: speed,
        }
    }
}

// Структура для зберігання "джерела" даних
struct ConfigDatabase {
    configs: Vec<DroneConfig>,
}

impl ConfigDatabase {
    fn new() -> Self {
        ConfigDatabase { configs: Vec::new() }
    }
    
    fn add(&mut self, config: DroneConfig) -> usize {
        self.configs.push(config);
        self.configs.len() - 1
    }
    
    fn get(&self, index: usize) -> Option<&DroneConfig> {
        self.configs.get(index)
    }
}

fn main() {
    println!("=== Кеш з lifetime ===\n");
    
    // Створюємо "базу даних" конфігурацій
    let mut db = ConfigDatabase::new();
    let idx_alpha = db.add(DroneConfig::new("Alpha", 150, 25.0));
    let idx_beta = db.add(DroneConfig::new("Beta", 200, 30.0));
    let idx_gamma = db.add(DroneConfig::new("Gamma", 100, 20.0));
    let idx_delta = db.add(DroneConfig::new("Delta", 180, 28.0));
    
    // Створюємо кеш з capacity 2
    let mut cache: Cache<&str, DroneConfig> = Cache::new(2);
    
    println!("--- Заповнення кешу ---");
    
    // Кешуємо посилання на конфігурації
    if let Some(config) = db.get(idx_alpha) {
        cache.set("alpha", config);
        println!("Cached: alpha -> {:?}", config.name);
    }
    
    if let Some(config) = db.get(idx_beta) {
        cache.set("beta", config);
        println!("Cached: beta -> {:?}", config.name);
    }
    
    println!("Cache size: {}", cache.len());
    
    // Додаємо третій — найстаріший (alpha) буде видалено
    if let Some(config) = db.get(idx_gamma) {
        cache.set("gamma", config);
        println!("\nCached: gamma -> {:?}", config.name);
        println!("Cache size: {} (alpha evicted)", cache.len());
    }
    
    // Перевірка
    println!("\n--- Перевірка кешу ---");
    println!("Contains 'alpha': {}", cache.contains(&"alpha"));
    println!("Contains 'beta': {}", cache.contains(&"beta"));
    println!("Contains 'gamma': {}", cache.contains(&"gamma"));
    
    // Отримання з кешу
    println!("\n--- Отримання з кешу ---");
    if let Some(config) = cache.get(&"beta") {
        println!("Got beta: {} (alt: {}m, speed: {}m/s)", 
                 config.name, config.max_altitude, config.max_speed);
    }
    
    // Keys
    println!("\n--- Ключі в кеші ---");
    for key in cache.keys() {
        println!("  Key: {}", key);
    }
    
    // Lifetime safety: кеш не може пережити db
    // drop(db);  // Якщо розкоментувати — cache стане невалідним
    
    println!("\n--- Cache still valid (db exists) ---");
    if let Some(config) = cache.get(&"gamma") {
        println!("Gamma config: {:?}", config);
    }
}
```

---

## Домашнє завдання

### Завдання 1: Парсер з позиціями

**Умова:** Створіть структуру `Token<'a>`, що зберігає посилання на частину вхідного тексту та позицію. Напишіть функцію токенізації.

**Розв'язання:**

```rust
#[derive(Debug, Clone)]
struct Token<'a> {
    text: &'a str,
    start: usize,
    end: usize,
    kind: TokenKind,
}

#[derive(Debug, Clone, Copy, PartialEq)]
enum TokenKind {
    Word,
    Number,
    Punctuation,
    Whitespace,
}

impl<'a> Token<'a> {
    fn new(text: &'a str, start: usize, end: usize, kind: TokenKind) -> Self {
        Token { text, start, end, kind }
    }
    
    fn len(&self) -> usize {
        self.end - self.start
    }
}

struct Tokenizer<'a> {
    input: &'a str,
    position: usize,
}

impl<'a> Tokenizer<'a> {
    fn new(input: &'a str) -> Self {
        Tokenizer { input, position: 0 }
    }
    
    fn tokenize(&mut self) -> Vec<Token<'a>> {
        let mut tokens = Vec::new();
        
        while self.position < self.input.len() {
            if let Some(token) = self.next_token() {
                tokens.push(token);
            }
        }
        
        tokens
    }
    
    fn next_token(&mut self) -> Option<Token<'a>> {
        let chars: Vec<char> = self.input[self.position..].chars().collect();
        if chars.is_empty() {
            return None;
        }
        
        let first = chars[0];
        let start = self.position;
        
        let (kind, len) = if first.is_whitespace() {
            let len = chars.iter().take_while(|c| c.is_whitespace()).count();
            (TokenKind::Whitespace, len)
        } else if first.is_alphabetic() {
            let len = chars.iter().take_while(|c| c.is_alphanumeric() || **c == '_').count();
            (TokenKind::Word, len)
        } else if first.is_numeric() {
            let len = chars.iter().take_while(|c| c.is_numeric() || **c == '.').count();
            (TokenKind::Number, len)
        } else {
            (TokenKind::Punctuation, 1)
        };
        
        // Calculate byte position
        let byte_len: usize = chars[..len].iter().map(|c| c.len_utf8()).sum();
        self.position += byte_len;
        
        Some(Token::new(
            &self.input[start..start + byte_len],
            start,
            start + byte_len,
            kind,
        ))
    }
}

fn tokenize<'a>(input: &'a str) -> Vec<Token<'a>> {
    Tokenizer::new(input).tokenize()
}

fn main() {
    println!("=== Парсер з позиціями ===\n");
    
    let code = "let x = 42 + y;";
    println!("Input: '{}'", code);
    
    let tokens = tokenize(code);
    
    println!("\nTokens:");
    for token in &tokens {
        if token.kind != TokenKind::Whitespace {
            println!("  {:?} '{}' at {}..{}", 
                     token.kind, token.text, token.start, token.end);
        }
    }
    
    // Filter by kind
    let words: Vec<_> = tokens.iter()
        .filter(|t| t.kind == TokenKind::Word)
        .collect();
    println!("\nWords: {:?}", words.iter().map(|t| t.text).collect::<Vec<_>>());
}
```

---

### Завдання 2: Пул об'єктів з посиланнями

**Умова:** Створіть `ObjectPool<T>`, що дозволяє "брати" об'єкти та повертати їх. Повернутий об'єкт — це посилання, що живе не довше ніж пул.

**Розв'язання:**

```rust
struct ObjectPool<T> {
    objects: Vec<T>,
    available: Vec<bool>,
}

struct PooledObject<'a, T> {
    pool: &'a mut ObjectPool<T>,
    index: usize,
}

impl<T> ObjectPool<T> {
    fn new() -> Self {
        ObjectPool {
            objects: Vec::new(),
            available: Vec::new(),
        }
    }
    
    fn add(&mut self, obj: T) {
        self.objects.push(obj);
        self.available.push(true);
    }
    
    fn acquire(&mut self) -> Option<(usize, &mut T)> {
        for (i, &available) in self.available.iter().enumerate() {
            if available {
                self.available[i] = false;
                return Some((i, &mut self.objects[i]));
            }
        }
        None
    }
    
    fn release(&mut self, index: usize) {
        if index < self.available.len() {
            self.available[index] = true;
        }
    }
    
    fn available_count(&self) -> usize {
        self.available.iter().filter(|&&a| a).count()
    }
    
    fn total_count(&self) -> usize {
        self.objects.len()
    }
}

// RAII guard для автоматичного повернення
struct PoolGuard<'a, T> {
    pool: &'a mut ObjectPool<T>,
    index: usize,
}

impl<'a, T> Drop for PoolGuard<'a, T> {
    fn drop(&mut self) {
        self.pool.release(self.index);
    }
}

// Приклад використання
#[derive(Debug)]
struct Connection {
    id: u32,
    active: bool,
}

impl Connection {
    fn new(id: u32) -> Self {
        Connection { id, active: false }
    }
    
    fn connect(&mut self) {
        self.active = true;
        println!("Connection {} opened", self.id);
    }
    
    fn disconnect(&mut self) {
        self.active = false;
        println!("Connection {} closed", self.id);
    }
}

fn main() {
    println!("=== Пул об'єктів ===\n");
    
    let mut pool: ObjectPool<Connection> = ObjectPool::new();
    
    // Додаємо з'єднання до пулу
    for i in 1..=3 {
        pool.add(Connection::new(i));
    }
    
    println!("Pool: {}/{} available\n", pool.available_count(), pool.total_count());
    
    // Беремо з'єднання
    if let Some((idx, conn)) = pool.acquire() {
        conn.connect();
        println!("Using connection {}", conn.id);
        conn.disconnect();
        pool.release(idx);
    }
    
    println!("\nPool: {}/{} available", pool.available_count(), pool.total_count());
    
    // Беремо кілька одночасно
    println!("\n--- Multiple acquisitions ---");
    let mut handles = Vec::new();
    
    while let Some((idx, conn)) = pool.acquire() {
        conn.connect();
        handles.push(idx);
    }
    
    println!("Pool: {}/{} available", pool.available_count(), pool.total_count());
    
    // Повертаємо
    for idx in handles {
        pool.release(idx);
    }
    
    println!("After release: {}/{} available", pool.available_count(), pool.total_count());
}
```

---

### Завдання 3: Split iterator

**Умова:** Створіть ітератор `SplitKeep<'a>`, що розбиває рядок за роздільником, але зберігає роздільники як окремі елементи.

**Розв'язання:**

```rust
struct SplitKeep<'a> {
    text: &'a str,
    delimiter: char,
    position: usize,
    finished: bool,
}

impl<'a> SplitKeep<'a> {
    fn new(text: &'a str, delimiter: char) -> Self {
        SplitKeep {
            text,
            delimiter,
            position: 0,
            finished: false,
        }
    }
}

impl<'a> Iterator for SplitKeep<'a> {
    type Item = &'a str;
    
    fn next(&mut self) -> Option<Self::Item> {
        if self.finished || self.position >= self.text.len() {
            return None;
        }
        
        let remaining = &self.text[self.position..];
        
        // Якщо починається з роздільника — повертаємо його
        if remaining.starts_with(self.delimiter) {
            let delim_len = self.delimiter.len_utf8();
            let result = &self.text[self.position..self.position + delim_len];
            self.position += delim_len;
            return Some(result);
        }
        
        // Інакше шукаємо наступний роздільник
        match remaining.find(self.delimiter) {
            Some(pos) => {
                let result = &self.text[self.position..self.position + pos];
                self.position += pos;
                Some(result)
            }
            None => {
                self.finished = true;
                Some(remaining)
            }
        }
    }
}

trait SplitKeepExt {
    fn split_keep(&self, delimiter: char) -> SplitKeep<'_>;
}

impl SplitKeepExt for str {
    fn split_keep(&self, delimiter: char) -> SplitKeep<'_> {
        SplitKeep::new(self, delimiter)
    }
}

fn main() {
    println!("=== Split Keep Iterator ===\n");
    
    let text = "hello,world,rust";
    println!("Text: '{}'", text);
    println!("Split by ',':");
    for part in text.split_keep(',') {
        println!("  '{}'", part);
    }
    
    let path = "/home/user/documents";
    println!("\nPath: '{}'", path);
    println!("Split by '/':");
    for part in path.split_keep('/') {
        println!("  '{}'", part);
    }
    
    // Зберігаємо в вектор
    let parts: Vec<&str> = "a.b.c".split_keep('.').collect();
    println!("\nCollected: {:?}", parts);
}
```

---

### Завдання 4: Граф з посиланнями

**Умова:** Створіть структуру графа, де вузли зберігаються окремо, а ребра — це посилання на вузли. Продемонструйте безпечну роботу з lifetimes.

**Розв'язання:**

```rust
use std::collections::HashMap;

#[derive(Debug)]
struct Node {
    id: String,
    data: String,
}

impl Node {
    fn new(id: &str, data: &str) -> Self {
        Node {
            id: id.to_string(),
            data: data.to_string(),
        }
    }
}

struct Graph<'a> {
    nodes: &'a HashMap<String, Node>,
    edges: Vec<(&'a Node, &'a Node)>,
}

impl<'a> Graph<'a> {
    fn new(nodes: &'a HashMap<String, Node>) -> Self {
        Graph {
            nodes,
            edges: Vec::new(),
        }
    }
    
    fn add_edge(&mut self, from_id: &str, to_id: &str) -> Result<(), String> {
        let from = self.nodes.get(from_id)
            .ok_or_else(|| format!("Node '{}' not found", from_id))?;
        let to = self.nodes.get(to_id)
            .ok_or_else(|| format!("Node '{}' not found", to_id))?;
        
        self.edges.push((from, to));
        Ok(())
    }
    
    fn neighbors(&self, node_id: &str) -> Vec<&'a Node> {
        self.edges.iter()
            .filter(|(from, _)| from.id == node_id)
            .map(|(_, to)| *to)
            .collect()
    }
    
    fn print(&self) {
        println!("Graph with {} nodes and {} edges:", 
                 self.nodes.len(), self.edges.len());
        for (from, to) in &self.edges {
            println!("  {} -> {}", from.id, to.id);
        }
    }
}

fn main() {
    println!("=== Граф з посиланнями ===\n");
    
    // Вузли живуть у HashMap
    let mut nodes = HashMap::new();
    nodes.insert("A".to_string(), Node::new("A", "Start"));
    nodes.insert("B".to_string(), Node::new("B", "Middle"));
    nodes.insert("C".to_string(), Node::new("C", "End"));
    nodes.insert("D".to_string(), Node::new("D", "Branch"));
    
    // Граф посилається на вузли
    let mut graph = Graph::new(&nodes);
    
    graph.add_edge("A", "B").unwrap();
    graph.add_edge("B", "C").unwrap();
    graph.add_edge("A", "D").unwrap();
    graph.add_edge("D", "C").unwrap();
    
    graph.print();
    
    // Сусіди
    println!("\nNeighbors of A:");
    for neighbor in graph.neighbors("A") {
        println!("  {} ({})", neighbor.id, neighbor.data);
    }
    
    // Граф не може пережити nodes
    // drop(nodes);  // ПОМИЛКА - graph все ще використовує посилання
    
    println!("\nGraph is valid while nodes exist");
}
```

---

## Підсумок заняття

На цьому занятті ви навчились:

1. **Розуміти lifetimes**: що це і навіщо потрібно
2. **Писати lifetime annotations**: `'a`, `'static`, зв'язки
3. **Застосовувати elision rules**: коли анотації не потрібні
4. **Створювати структури з посиланнями**: `struct Foo<'a> { data: &'a T }`
5. **Вирішувати типові проблеми**: dangling references, конфлікти

---

## Перевірте себе

1. Що таке lifetime у Rust?
2. Коли потрібні явні lifetime annotations?
3. Що означає `'static`?
4. Які три правила elision?
5. Чому не можна повернути посилання на локальну змінну?
6. Як пов'язані lifetime структури та її полів?

**Відповіді:**
1. Область коду, в якій посилання гарантовано валідне
2. Коли кілька вхідних посилань і результат — посилання, і компілятор не може вивести зв'язок
3. Lifetime, що охоплює весь час виконання програми (string literals, static)
4. (1) Кожен вхідний отримує свій, (2) один вхідний → такий же вихідний, (3) &self → вихідний = self
5. Локальна змінна знищується при виході з функції, посилання стане dangling
6. Структура не може жити довше ніж дані, на які посилаються її поля

---

## Наступне заняття

На наступному занятті ми вивчимо **Багатопотоковість (Threads)** — як створювати потоки, передавати дані між ними та використовувати `Arc` і `Mutex` для безпечного спільного доступу.
