# Розділ 32: Lifetimes — часи життя посилань

## Вступ: чому компілятор запитує про 'a?

Ви вже знаєте, що кожне посилання в Rust має **час життя** — період, протягом якого це посилання валідне. До цього моменту компілятор виводив часи життя автоматично, і ви про це навіть не замислювались.

Але раптом з'являється помилка:

```
error[E0106]: missing lifetime specifier
 --> src/main.rs:1:33
  |
1 | fn longest(x: &str, y: &str) -> &str {
  |               ----     ----     ^ expected named lifetime parameter
```

Що це за загадкові `'a`? Чому іноді потрібні, а іноді ні? Чому функція з одним посиланням компілюється, а з двома — ні?

**Ключове розуміння:** Анотації часів життя — це **не магія**. Це спосіб пояснити компілятору, як довго посилання залишатимуться валідними. Ви не змінюєте час життя — ви **описуєте** зв'язки.

---

## 32.1 Що таке час життя

### Інтуїтивне розуміння

Кожне посилання має **scope** — область, де воно валідне. Коли дані, на які вказує посилання, звільняються — посилання стає **"звислим"** (dangling).

**Постановка задачі:** Показати, як Rust запобігає звислим посиланням.

```rust
fn main() {
    let r;                      // ─────┐ 'a (час життя r)
                                //      │
    {                           //      │
        let x = 5;              // ─┐   │ 'b (час життя x)
        r = &x;                 //  │   │ r посилається на x
    }                           // ─┘   │ x звільнено!
                                //      │
    // println!("{}", r);       // ─────┘ r тепер звисле!
}
```

**Як це працює:**
1. `r` оголошено, але не ініціалізовано
2. У внутрішньому блоці `x` отримує значення 5
3. `r = &x` — тепер r посилається на x
4. Блок закінчується — `x` **звільняється**
5. `r` все ще "живе", але вказує на **звільнену пам'ять**!

Компілятор бачить: час життя `'a` (r) **довший** за `'b` (x). Використання `r` після `}` — помилка компіляції, не runtime crash.

### Чому Rust вимагає анотацій

Компілятор аналізує кожну функцію **окремо**, не знаючи, як вона буде викликана:

```rust
fn mystery(x: &str, y: &str) -> &str {
    // Компілятор бачить два посилання на вході
    // і одне на виході
    // 
    // Питання:
    // - Результат пов'язаний з x?
    // - Чи з y?
    // - Чи з обома?
    
    x  // Навіть якщо завжди повертаємо x
}
```

Без анотацій компілятор **не знає**, який зв'язок. А йому потрібно знати, щоб перевірити: чи результат використовується тільки поки валідні вхідні дані.

**Анотації — це контракт:** "Я гарантую, що результат валідний так довго".

### Важливо: анотації НЕ змінюють час життя

Анотації **описують** зв'язки, не **встановлюють** їх:

```rust
// Це НЕ "продовжує життя" посилання
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str { ... }

// Це КАЖЕ: "результат валідний, поки валідні ОБИДВА аргументи"
// (тобто поки валідний КОРОТШИЙ з них)
```

---

## 32.2 Синтаксис анотацій

### Базовий синтаксис

Анотація часу життя — це апостроф з ім'ям: `'a`, `'b`, `'lifetime`:

```rust
&i32         // Посилання (час життя виводиться автоматично)
&'a i32      // Посилання з явним часом життя 'a
&'a mut i32  // Мутабельне посилання з часом життя 'a
```

**Імена** — довільні ідентифікатори. За конвенцією використовують короткі: `'a`, `'b`, `'c`. Для читабельності можна й довші: `'input`, `'output`.

### Анотації у функціях

**Постановка задачі:** Показати синтаксис анотацій у функціях.

```rust
// Один параметр часу життя
fn first<'a>(slice: &'a [i32]) -> &'a i32 {
    &slice[0]
}

// Пояснення:
// - <'a> — оголошуємо параметр часу життя
// - slice: &'a [i32] — вхідний слайс з часом життя 'a
// - -> &'a i32 — результат теж має час життя 'a
// Контракт: результат валідний поки валідний slice

// Два параметри, результат прив'язаний до одного
fn choose<'a, 'b>(x: &'a str, _y: &'b str) -> &'a str {
    x  // Завжди повертаємо x, тому результат — 'a
}

// Два параметри, результат прив'язаний до обох
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
// Обидва мають той самий 'a — результат валідний
// поки валідні ОБИДВА (тобто поки валідний коротший)
```

### Анотації у структурах

Структура з посиланням **обов'язково** потребує анотації:

**Постановка задачі:** Показати структуру з посиланням.

```rust
// Структура зберігає посилання на текст
struct Excerpt<'a> {
    text: &'a str,
}

// <'a> — структура параметризована часом життя
// text: &'a str — поле з посиланням

impl<'a> Excerpt<'a> {
    fn new(text: &'a str) -> Self {
        Self { text }
    }
    
    fn text(&self) -> &str {
        self.text
    }
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let excerpt = Excerpt::new(&novel[..14]);
    
    println!("Excerpt: {}", excerpt.text());
    
    // excerpt не може пережити novel
    // Якщо novel звільниться — excerpt стане невалідним
}
```

### 'static — особливий час життя

`'static` означає "живе всю програму":

```rust
// Рядкові літерали — 'static
let s: &'static str = "Hello, world!";

// Константи та статичні змінні — 'static
static GREETING: &str = "Hello";
const VERSION: &str = "1.0.0";

// Чому рядкові літерали 'static?
// Бо вони вкомпільовані в бінарний файл
// і існують з початку до кінця програми
```

---

## 32.3 Правила виведення (Elision Rules)

### Коли анотації НЕ потрібні

Rust має **правила виведення** (elision rules), що дозволяють пропускати анотації у простих випадках. Це не магія — компілятор застосовує детерміновані правила.

**Постановка задачі:** Показати еквівалентні записи.

```rust
// Ці пари функцій ІДЕНТИЧНІ:

fn first(s: &str) -> &str { ... }
fn first<'a>(s: &'a str) -> &'a str { ... }

fn get(v: &Vec<i32>) -> &i32 { ... }
fn get<'a>(v: &'a Vec<i32>) -> &'a i32 { ... }

fn len(s: &String) -> usize { s.len() }  // Без посилання в результаті — без проблем
```

### Три правила виведення

Компілятор застосовує три правила **послідовно**:

**Правило 1:** Кожен вхідний параметр-посилання отримує **власний** час життя.

```rust
fn foo(x: &str, y: &str)
// Стає:
fn foo<'a, 'b>(x: &'a str, y: &'b str)

fn bar(x: &str)
// Стає:
fn bar<'a>(x: &'a str)
```

**Правило 2:** Якщо **один** вхідний параметр — результат отримує **його** час життя.

```rust
fn foo(x: &str) -> &str
// Після правила 1: fn foo<'a>(x: &'a str) -> &str
// Після правила 2: fn foo<'a>(x: &'a str) -> &'a str  ✅
```

**Правило 3:** Якщо є `&self` або `&mut self` — результат отримує час життя **self**.

```rust
impl MyStruct {
    fn method(&self, x: &str) -> &str
    // Після правила 1: fn method<'a, 'b>(&'a self, x: &'b str) -> &str
    // Правило 2 не працює (два параметри)
    // Після правила 3: fn method<'a, 'b>(&'a self, x: &'b str) -> &'a str  ✅
}
```

### Коли правила НЕ працюють

**Постановка задачі:** Показати, коли потрібні явні анотації.

```rust
// ❌ Два параметри — правило 2 не працює, правило 3 теж (немає self)
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() { x } else { y }
}

// Компілятор застосовує правила:
// Правило 1: fn longest<'a, 'b>(x: &'a str, y: &'b str) -> &str
// Правило 2: НЕ застосовується (два параметри)
// Правило 3: НЕ застосовується (немає self)
// Результат: час життя результату НЕВІДОМИЙ
// → ПОМИЛКА: "missing lifetime specifier"

// ✅ Виправлення — явна анотація
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

---

## 32.4 Практичні приклади

### Функція з одним посиланням

**Постановка задачі:** Знайти перше слово в рядку.

```rust
// Анотація НЕ потрібна — застосовується правило 2
fn first_word(s: &str) -> &str {
    match s.find(' ') {
        Some(i) => &s[..i],
        None => s,
    }
}

fn main() {
    let sentence = String::from("Hello beautiful world");
    let word = first_word(&sentence);
    println!("First word: '{}'", word);  // "Hello"
    
    // word валідний, поки валідний sentence
}
```

**Як це працює:**
1. Правило 1: `fn first_word<'a>(s: &'a str) -> &str`
2. Правило 2: один параметр → результат отримує 'a
3. Результат: `fn first_word<'a>(s: &'a str) -> &'a str`

### Функція з кількома посиланнями

**Постановка задачі:** Знайти довший з двох рядків.

```rust
// Анотація ПОТРІБНА — два параметри
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

fn main() {
    let s1 = String::from("short");
    let s2 = String::from("much longer string");
    
    let result = longest(&s1, &s2);
    println!("Longest: '{}'", result);
    
    // result валідний поки валідні ОБИДВА s1 та s2
}
```

**Що означає `'a` тут:**
- Обидва параметри мають **той самий** час життя 'a
- Результат теж має 'a
- На практиці 'a буде **коротшим** з часів життя s1 та s2

### Різні часи життя

**Постановка задачі:** Функція завжди повертає перший аргумент.

```rust
// Результат прив'язаний ТІЛЬКИ до x
fn first_of_two<'a, 'b>(x: &'a str, _y: &'b str) -> &'a str {
    x  // Завжди повертаємо x
}

fn main() {
    let outer = String::from("I live long");
    let result;
    
    {
        let inner = String::from("I die soon");
        result = first_of_two(&outer, &inner);
        // inner виходить зі scope
    }
    
    // result все ще валідний!
    // Бо прив'язаний до outer, не до inner
    println!("{}", result);
}
```

---

## 32.5 Структури з посиланнями

### Базовий приклад

**Постановка задачі:** Створити структуру, що зберігає посилання на частину тексту.

```rust
// Структура ОБОВ'ЯЗКОВО параметризується часом життя
struct TextExcerpt<'a> {
    content: &'a str,
    page: u32,
}

impl<'a> TextExcerpt<'a> {
    fn new(content: &'a str, page: u32) -> Self {
        Self { content, page }
    }
    
    fn content(&self) -> &str {
        self.content
    }
    
    fn summary(&self) -> String {
        format!("Page {}: '{}'", self.page, 
            if self.content.len() > 20 {
                format!("{}...", &self.content[..20])
            } else {
                self.content.to_string()
            }
        )
    }
}

fn main() {
    let book = String::from("It was the best of times, it was the worst of times.");
    
    let excerpt = TextExcerpt::new(&book[..22], 1);
    
    println!("{}", excerpt.summary());
    // Page 1: 'It was the best of ti...'
    
    // excerpt НЕ МОЖЕ пережити book
}
```

### Анонімний час життя '_

У багатьох випадках можна використовувати `'_` замість явного імені:

```rust
impl Agent {
    // Обидва записи еквівалентні:
    fn snapshot(&self) -> AgentSnapshot<'_> { ... }
    fn snapshot<'a>(&'a self) -> AgentSnapshot<'a> { ... }
}

// '_ каже: "виведи час життя автоматично"
```

---

## 32.6 Часи життя для агентів

### Шаблон "Знімок" (Snapshot)

**Постановка задачі:** Створити легкий "вигляд" на стан агента без копіювання.

```rust
/// Повний стан агента (володіє даними)
struct Agent {
    id: String,
    name: String,
    position: (f64, f64),
    battery: u8,
    mission: Option<String>,
    history: Vec<(f64, f64)>,
}

/// Легкий знімок (тільки посилання — без алокацій)
struct AgentSnapshot<'a> {
    id: &'a str,
    name: &'a str,
    position: (f64, f64),  // Copy тип — без посилання
    battery: u8,           // Copy тип
}

impl Agent {
    fn new(id: &str, name: &str) -> Self {
        Self {
            id: id.to_string(),
            name: name.to_string(),
            position: (0.0, 0.0),
            battery: 100,
            mission: None,
            history: Vec::new(),
        }
    }
    
    /// Створення знімка — дешева операція (без алокацій)
    fn snapshot(&self) -> AgentSnapshot<'_> {
        AgentSnapshot {
            id: &self.id,
            name: &self.name,
            position: self.position,
            battery: self.battery,
        }
    }
}

/// Функція приймає знімок — легковісна операція
fn display_status(snap: &AgentSnapshot) {
    println!("[{}] {} at {:?}, battery: {}%",
        snap.id, snap.name, snap.position, snap.battery);
}

fn main() {
    let agent = Agent::new("SCOUT-01", "Shadow");
    
    // Створюємо знімок — без копіювання String
    let snap = agent.snapshot();
    display_status(&snap);
    
    // agent все ще доступний
    println!("Agent still owns: {}", agent.id);
}
```

### Шаблон "Позичання конфігурації"

**Постановка задачі:** Виконавець місії позичає конфігурацію, не володіє нею.

```rust
struct MissionConfig {
    name: String,
    waypoints: Vec<(f64, f64)>,
    timeout_seconds: u64,
}

/// Виконавець місії — позичає конфігурацію
struct MissionExecutor<'a> {
    config: &'a MissionConfig,
    current_waypoint: usize,
    start_time: std::time::Instant,
}

impl<'a> MissionExecutor<'a> {
    fn new(config: &'a MissionConfig) -> Self {
        Self {
            config,
            current_waypoint: 0,
            start_time: std::time::Instant::now(),
        }
    }
    
    fn mission_name(&self) -> &str {
        &self.config.name
    }
    
    fn current_target(&self) -> Option<(f64, f64)> {
        self.config.waypoints.get(self.current_waypoint).copied()
    }
    
    fn advance(&mut self) -> bool {
        if self.current_waypoint < self.config.waypoints.len() - 1 {
            self.current_waypoint += 1;
            true
        } else {
            false
        }
    }
    
    fn is_timeout(&self) -> bool {
        self.start_time.elapsed().as_secs() > self.config.timeout_seconds
    }
}

fn main() {
    let config = MissionConfig {
        name: "Patrol Alpha".to_string(),
        waypoints: vec![(0.0, 0.0), (10.0, 0.0), (10.0, 10.0)],
        timeout_seconds: 3600,
    };
    
    let mut executor = MissionExecutor::new(&config);
    
    println!("Mission: {}", executor.mission_name());
    
    while let Some(target) = executor.current_target() {
        println!("Moving to {:?}", target);
        if !executor.advance() {
            println!("Mission complete!");
            break;
        }
    }
    
    // config все ще доступна
    println!("Config had {} waypoints", config.waypoints.len());
}
```

---

## 32.7 Типові помилки та їх виправлення

### Помилка: звисле посилання

```rust
// ❌ Повертаємо посилання на локальну змінну
fn dangling() -> &str {
    let s = String::from("hello");
    &s  // s звільняється тут!
}
// error: cannot return reference to local variable

// ✅ Виправлення 1: Повертаємо owned значення
fn not_dangling() -> String {
    String::from("hello")
}

// ✅ Виправлення 2: Приймаємо посилання ззовні
fn process(s: &str) -> &str {
    s  // Посилання приходить ззовні, не створюється всередині
}
```

### Помилка: неправильні зв'язки

```rust
// ❌ x та y не мають часу життя 'a
fn wrong<'a>(x: &str, y: &str) -> &'a str {
    x
}
// error: lifetime may not live long enough

// ✅ Виправлення: x має мати 'a
fn correct<'a>(x: &'a str, _y: &str) -> &'a str {
    x
}
```

### Помилка: структура переживає дані

```rust
struct Holder<'a> {
    data: &'a str,
}

fn main() {
    let holder;
    {
        let s = String::from("temporary");
        holder = Holder { data: &s };
        // ❌ s виходить зі scope
    }
    // holder.data — звисле!
}
// error: `s` does not live long enough
```

---

## 32.8 Лабораторна робота

**Мета:** Навчитись працювати з часами життя.

### Частина 1: Функція вибору (3 бали)

Напишіть функцію, що повертає довший слайс:

```rust
fn longer_slice<'a>(x: &'a [i32], y: &'a [i32]) -> &'a [i32] {
    // Реалізуйте
}
```

### Частина 2: Структура з посиланням (4 бали)

Створіть структуру цитати:

```rust
struct Quote<'a> {
    text: &'a str,
    author: &'a str,
}
```

### Частина 3: Знімок агента (3 бали)

Реалізуйте патерн Snapshot для власної структури агента.

---

## Підсумок

**Час життя** — це scope, протягом якого посилання валідне.

| Концепція | Опис |
|-----------|------|
| `'a` | Параметр часу життя |
| `'static` | Час життя на всю програму |
| Elision | Автоматичне виведення |
| Анотації | Описують зв'язки, не змінюють час |

**Ключові правила:**

- ✅ Більшість коду НЕ потребує явних анотацій
- ✅ Правила виведення покривають прості випадки
- ✅ Структури з посиланнями ЗАВЖДИ потребують анотацій
- ✅ Анотації **описують**, не **встановлюють** часи життя
- ⚠️ Уникайте надмірного використання `'static`

---

## Зв'язок з наступним розділом

Ви опанували однопотоковий Rust повністю. Тепер час для справжньої магії — **багатопотоковості**. Rust гарантує відсутність data races на етапі компіляції!

У **Розділі 33: Threads — багатопотоковість** ви дізнаєтесь:
- Як створювати **потоки** (`std::thread`)
- Що таке **Send** та **Sync** traits
- Чому Rust **безпечніший** за інші мови для паралельного коду
- Як кілька агентів можуть працювати **одночасно**
