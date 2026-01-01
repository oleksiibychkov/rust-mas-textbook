# Розділ 42: Actor Model — Реалізація на Tokio

---

## 📋 Анотація

У попередньому розділі ви зрозуміли філософію Actor Model: ізольовані актори, що спілкуються через повідомлення, послідовна обробка без гонок даних, supervision для відмовостійкості. Це були концепції. Тепер час втілити їх у **працюючий Rust-код**.

Ми **свідомо не використовуємо** готові фреймворки на кшталт Actix чи Bastion. Натомість побудуємо актора "з нуля" на примітивах Tokio, які ви вже знаєте: `tokio::spawn` для задач, `mpsc` для повідомлень, `oneshot` для відповідей. Чому? По-перше, ви побачите, що Actor Model — це **архітектурний патерн**, а не бібліотека. Будь-яка система з async задачею та каналом уже є потенційним актором. По-друге, глибоке розуміння механіки дозволить вам адаптувати патерн під конкретні потреби, а не боротись з обмеженнями фреймворку.

Цей розділ — міст від теорії до практики. Ви побудуєте повноцінного агента-актора, готового стати частиною рою БПЛА: з обробкою команд, запитами стану, graceful shutdown та внутрішніми таймерами.

---

## 🎯 Цілі навчання

Після завершення цього розділу ви зможете:

1. **Реалізувати** актора через tokio::spawn та mpsc channel
2. **Створювати** ActorHandle для безпечної взаємодії з актором ззовні
3. **Застосовувати** патерн request-response через oneshot канал
4. **Реалізувати** graceful shutdown актора
5. **Додавати** внутрішні таймери через select!
6. **Структурувати** агента БПЛА як повноцінного актора

---

## 📚 Ключові терміни

| Термін | Визначення |
|--------|------------|
| **Actor Task** | Tokio задача (`JoinHandle`), що виконує нескінченний цикл обробки повідомлень |
| **ActorHandle** | Структура-фасад для зовнішньої взаємодії; містить Sender, приховує деталі |
| **Message Loop** | Цикл `while let Some(msg) = rx.recv().await`, серце актора |
| **Oneshot Response** | Канал `oneshot::Sender` у повідомленні для отримання відповіді |
| **Graceful Shutdown** | Коректне завершення: обробити залишкові повідомлення, звільнити ресурси |
| **Actor Tick** | Періодична дія актора (таймер), незалежна від вхідних повідомлень |

---

## 💡 Мотиваційний кейс: Від концепції до робочого коду

Повернімось до агента-БПЛА з попередніх розділів. У термінах Actor Model:

```text
┌─────────────────────────────────────────────────────────────────────┐
│                        БПЛА-АГЕНТ ЯК АКТОР                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Actor Model концепція          Rust реалізація                    │
│  ═══════════════════           ════════════════                    │
│                                                                     │
│  Mailbox (черга)        →      mpsc::Receiver<AgentMessage>        │
│                                                                     │
│  State (стан)           →      struct DroneState {                 │
│                                    position: (f64, f64),           │
│                                    battery: u8,                    │
│                                    mode: DroneMode,                │
│                                }                                   │
│                                                                     │
│  Behavior (поведінка)   →      impl DroneActor {                   │
│                                    fn handle(&mut self, msg) {...} │
│                                }                                   │
│                                                                     │
│  Address (адреса)       →      DroneHandle {                       │
│                                    tx: mpsc::Sender<AgentMessage>  │
│                                }                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

Виявляється, для реалізації актора достатньо **трьох речей**, які ви вже знаєте:
1. **Структура** для стану
2. **Enum** для типів повідомлень
3. **Async функція** з циклом обробки

Ніякої магії, ніяких макросів — чистий Rust та Tokio.

---

## 42.1 АРХІТЕКТУРА АКТОРА НА TOKIO

### 42.1.1 Три складові частини

Кожен актор на Tokio складається з трьох логічних компонентів:

```text
┌─────────────────────────────────────────────────────────────────────┐
│                    АНАТОМІЯ АКТОРА НА TOKIO                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ 1. MESSAGES (enum)                                             │ │
│  │                                                                │ │
│  │    enum ActorMessage {                                         │ │
│  │        Command { ... },          // вхідні команди            │ │
│  │        Query { respond_to },     // запити з відповіддю       │ │
│  │        Shutdown,                 // сигнал завершення         │ │
│  │    }                                                          │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                              │                                      │
│                              ▼                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ 2. ACTOR (struct + impl)                                       │ │
│  │                                                                │ │
│  │    struct MyActor {                                            │ │
│  │        state: MyState,           // приватний стан            │ │
│  │    }                                                          │ │
│  │                                                                │ │
│  │    impl MyActor {                                              │ │
│  │        fn handle(&mut self, msg) { ... }  // поведінка        │ │
│  │        async fn run(self, rx) { ... }     // цикл обробки     │ │
│  │    }                                                          │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                              │                                      │
│                              ▼                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ 3. HANDLE (struct)                                             │ │
│  │                                                                │ │
│  │    struct ActorHandle {                                        │ │
│  │        tx: mpsc::Sender<ActorMessage>,   // для надсилання    │ │
│  │    }                                                          │ │
│  │                                                                │ │
│  │    impl ActorHandle {                                          │ │
│  │        fn new() -> Self { ... }          // spawn + channel   │ │
│  │        async fn command(&self) { ... }   // зручні методи     │ │
│  │    }                                                          │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**1. Messages (Повідомлення)**

Enum, що визначає всі типи повідомлень, які актор може обробляти. Це контракт: актор гарантує обробку цих типів і нічого іншого.

**2. Actor (Стан + Поведінка)**

Структура з приватним станом та методами обробки. Ключовий метод — `run()` — нескінченний async цикл, що читає повідомлення з каналу та викликає обробники.

**3. Handle (Фасад)**

Структура, що містить `Sender` каналу. Це "публічний API" актора — зовнішній код взаємодіє тільки через handle. Handle приховує деталі реалізації та надає зручні методи.

### 42.1.2 Чому саме така архітектура?

Ця архітектура забезпечує всі гарантії Actor Model:

**Ізоляція стану**: Struct актора "споживається" при запуску (`run(self, ...)`). Після цього тільки задача актора має доступ до стану. Зовнішній код має тільки handle з Sender.

**Послідовна обробка**: Цикл `while let Some(msg) = rx.recv().await` обробляє повідомлення одне за одним. Неможливо мати два одночасні виклики `handle()`.

**Асинхронність**: `recv().await` не блокує потік. Коли повідомлень немає — задача поступається, worker виконує інші задачі.

**Інкапсуляція**: Handle надає типобезпечний інтерфейс. Неможливо надіслати "невідомий" тип повідомлення — компілятор не дозволить.

### 42.1.3 Життєвий цикл актора

```text
┌─────────────────────────────────────────────────────────────────────┐
│                    ЖИТТЄВИЙ ЦИКЛ АКТОРА                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Створення            Робота                        Завершення      │
│  ══════════            ══════                        ══════════      │
│                                                                     │
│  ActorHandle::new()                                                 │
│       │                                                             │
│       ├──► mpsc::channel()                                          │
│       │                                                             │
│       ├──► Actor::new()                                             │
│       │                                                             │
│       ├──► tokio::spawn(actor.run(rx))                             │
│       │         │                                                   │
│       │         ▼                                                   │
│       │    ┌─────────────────────────────────────────┐             │
│       │    │  while let Some(msg) = rx.recv().await  │ ◄─ Loop     │
│       │    │  {                                      │             │
│       │    │      self.handle(msg);                  │             │
│       │    │      if shutdown { break; }             │             │
│       │    │  }                                      │             │
│       │    └─────────────────────────────────────────┘             │
│       │                   │                                         │
│       └──► return Handle  │                                         │
│                           │                                         │
│  Використання:            │                                         │
│  handle.command().await ──┼──► msg до rx ──► обробка               │
│  handle.query().await ────┼──► msg до rx ──► обробка ──► відповідь │
│                           │                                         │
│  Завершення:              │                                         │
│  handle.shutdown().await ─┼──► Shutdown msg                        │
│                           │           │                             │
│                           │           ▼                             │
│                           │      break з циклу                     │
│                           │           │                             │
│  drop(handle) ────────────┼───────────┤                            │
│                           │           │                             │
│  (всі Sender знищені)     │           ▼                             │
│                           │      rx.recv() → None                  │
│                           │           │                             │
│                           │           ▼                             │
│                           └──────► цикл завершується               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 42.2 БАЗОВИЙ АКТОР: ПОКРОКОВА РЕАЛІЗАЦІЯ

### 42.2.1 Крок 1: Визначення повідомлень

Почнемо з найпростішого актора — лічильника. Спочатку визначимо, які повідомлення він може обробляти.

**Постановка задачі:** Створимо enum для повідомлень лічильника: збільшити, зменшити, вивести значення.

```rust
/// Повідомлення для актора-лічильника
/// Кожен варіант enum — це тип повідомлення, яке актор розуміє
enum CounterMessage {
    /// Збільшити лічильник на 1
    Increment,
    
    /// Зменшити лічильник на 1
    Decrement,
    
    /// Вивести поточне значення в консоль
    Print,
}
```

**Що важливо розуміти:** Enum повідомлень — це контракт між актором і зовнішнім світом. Актор гарантує, що обробить кожен варіант цього enum. Компілятор Rust перевірить, що ви обробили всі варіанти в `match`.

### 42.2.2 Крок 2: Структура актора зі станом

**Постановка задачі:** Створимо структуру актора з приватним станом — значенням лічильника.

```rust
/// Актор-лічильник
/// Структура містить весь стан актора — у цьому випадку одне число
struct CounterActor {
    /// Поточне значення лічильника
    /// Це ПРИВАТНИЙ стан — недоступний ззовні
    value: i32,
    
    /// Ім'я актора для логування
    name: String,
}

impl CounterActor {
    /// Створення нового актора з початковим значенням 0
    fn new(name: String) -> Self {
        println!("[{}] Актор створений", name);
        Self { value: 0, name }
    }
}
```

**Що важливо розуміти:** Поля структури приватні. Єдиний спосіб взаємодії зі станом — через повідомлення. Це і є ізоляція стану в Actor Model.

### 42.2.3 Крок 3: Метод обробки повідомлень

**Постановка задачі:** Реалізуємо логіку обробки кожного типу повідомлення.

```rust
impl CounterActor {
    // ... new() ...
    
    /// Обробка одного повідомлення
    /// Цей метод викликається для кожного повідомлення з черги
    fn handle_message(&mut self, msg: CounterMessage) {
        match msg {
            CounterMessage::Increment => {
                self.value += 1;
                println!("[{}] Increment → {}", self.name, self.value);
            }
            CounterMessage::Decrement => {
                self.value -= 1;
                println!("[{}] Decrement → {}", self.name, self.value);
            }
            CounterMessage::Print => {
                println!("[{}] Поточне значення: {}", self.name, self.value);
            }
        }
    }
}
```

**Що важливо розуміти:** Метод приймає `&mut self` — він може змінювати стан. Але оскільки цикл обробки викликає цей метод послідовно, гонок даних бути не може.

### 42.2.4 Крок 4: Цикл обробки — серце актора

**Постановка задачі:** Реалізуємо нескінченний async цикл, що читає повідомлення з каналу та обробляє їх.

```rust
use tokio::sync::mpsc;

impl CounterActor {
    // ... new(), handle_message() ...
    
    /// Головний цикл актора
    /// Ця функція "споживає" актора (self, не &self) — 
    /// після виклику тільки ця задача має доступ до стану
    async fn run(mut self, mut rx: mpsc::Receiver<CounterMessage>) {
        println!("[{}] Цикл обробки запущено", self.name);
        
        // Нескінченний цикл обробки повідомлень
        // while let Some(msg) = rx.recv().await означає:
        // - rx.recv().await чекає наступного повідомлення
        // - Повертає Some(msg) якщо повідомлення є
        // - Повертає None якщо канал закрито (всі Sender знищені)
        while let Some(msg) = rx.recv().await {
            self.handle_message(msg);
        }
        
        // Цикл завершився — канал закрито
        println!("[{}] Цикл обробки завершено", self.name);
    }
}
```

**Розглянемо, що відбувається в цьому циклі:**

1. `rx.recv().await` — асинхронно чекає повідомлення. Якщо повідомлень немає, задача "поступається", і worker виконує інші задачі. Це НЕ блокування потоку!

2. `Some(msg)` — повідомлення отримано, обробляємо його через `handle_message()`.

3. `None` — канал закрито. Це відбувається, коли всі `Sender` (тобто всі Handle) знищені. Цикл завершується природно.

4. `mut self` — актор "споживається". Після виклику `run()` оригінальна змінна недоступна. Тільки ця async задача має доступ до стану.

### 42.2.5 Крок 5: Handle — публічний інтерфейс

**Постановка задачі:** Створимо структуру-фасад, через яку зовнішній код взаємодіє з актором.

```rust
/// Handle для взаємодії з актором-лічильником
/// Це "публічний API" актора — єдиний спосіб взаємодії ззовні
#[derive(Clone)]  // Handle можна клонувати — кілька частин коду можуть взаємодіяти
struct CounterHandle {
    /// Канал для надсилання повідомлень актору
    tx: mpsc::Sender<CounterMessage>,
}

impl CounterHandle {
    /// Створення нового актора та отримання handle
    /// Ця функція:
    /// 1. Створює канал
    /// 2. Створює актора
    /// 3. Запускає актора в окремій задачі
    /// 4. Повертає handle для взаємодії
    fn new(name: &str) -> Self {
        // Створюємо канал з буфером на 32 повідомлення
        let (tx, rx) = mpsc::channel(32);
        
        // Створюємо актора
        let actor = CounterActor::new(name.to_string());
        
        // Запускаємо актора в окремій Tokio задачі
        // spawn "від'єднує" актора — він працює незалежно
        tokio::spawn(actor.run(rx));
        
        // Повертаємо handle з Sender
        Self { tx }
    }
    
    /// Збільшити лічильник
    /// async бо send може чекати, якщо буфер повний
    pub async fn increment(&self) {
        // Ігноруємо помилку (актор міг завершитись)
        let _ = self.tx.send(CounterMessage::Increment).await;
    }
    
    /// Зменшити лічильник
    pub async fn decrement(&self) {
        let _ = self.tx.send(CounterMessage::Decrement).await;
    }
    
    /// Вивести значення
    pub async fn print(&self) {
        let _ = self.tx.send(CounterMessage::Print).await;
    }
}
```

**Що важливо розуміти про Handle:**

1. **Інкапсуляція**: Користувач handle не знає, як актор влаштований всередині. Він просто викликає методи `increment()`, `decrement()`, `print()`.

2. **Clone**: Handle можна клонувати. Кожен клон має свій `Sender`, але всі вони пишуть у той самий канал. Це дозволяє кільком частинам коду взаємодіяти з одним актором.

3. **Помилки ігноруються**: `let _ = self.tx.send(...).await` — якщо актор завершився (канал закрито), send поверне помилку. Ми її ігноруємо. В production коді варто обробляти.

### 42.2.6 Повний приклад з використанням

**Постановка задачі:** Зберемо все разом і продемонструємо роботу актора.

```rust
use tokio::sync::mpsc;
use tokio::time::{sleep, Duration};

// === ПОВІДОМЛЕННЯ ===
enum CounterMessage {
    Increment,
    Decrement,
    Print,
}

// === АКТОР ===
struct CounterActor {
    value: i32,
    name: String,
}

impl CounterActor {
    fn new(name: String) -> Self {
        println!("[{}] Актор створений", name);
        Self { value: 0, name }
    }
    
    fn handle_message(&mut self, msg: CounterMessage) {
        match msg {
            CounterMessage::Increment => {
                self.value += 1;
                println!("[{}] Increment → {}", self.name, self.value);
            }
            CounterMessage::Decrement => {
                self.value -= 1;
                println!("[{}] Decrement → {}", self.name, self.value);
            }
            CounterMessage::Print => {
                println!("[{}] Поточне значення: {}", self.name, self.value);
            }
        }
    }
    
    async fn run(mut self, mut rx: mpsc::Receiver<CounterMessage>) {
        println!("[{}] Цикл обробки запущено", self.name);
        
        while let Some(msg) = rx.recv().await {
            self.handle_message(msg);
        }
        
        println!("[{}] Цикл обробки завершено", self.name);
    }
}

// === HANDLE ===
#[derive(Clone)]
struct CounterHandle {
    tx: mpsc::Sender<CounterMessage>,
}

impl CounterHandle {
    fn new(name: &str) -> Self {
        let (tx, rx) = mpsc::channel(32);
        let actor = CounterActor::new(name.to_string());
        tokio::spawn(actor.run(rx));
        Self { tx }
    }
    
    pub async fn increment(&self) {
        let _ = self.tx.send(CounterMessage::Increment).await;
    }
    
    pub async fn decrement(&self) {
        let _ = self.tx.send(CounterMessage::Decrement).await;
    }
    
    pub async fn print(&self) {
        let _ = self.tx.send(CounterMessage::Print).await;
    }
}

// === ВИКОРИСТАННЯ ===
#[tokio::main]
async fn main() {
    // Створюємо актора (автоматично запускається)
    let counter = CounterHandle::new("Counter-1");
    
    // Взаємодіємо через handle
    counter.increment().await;
    counter.increment().await;
    counter.increment().await;
    counter.print().await;
    
    counter.decrement().await;
    counter.print().await;
    
    // Даємо час на обробку останніх повідомлень
    sleep(Duration::from_millis(100)).await;
    
    // Коли counter виходить зі scope, Sender знищується,
    // канал закривається, і актор завершується
    println!("[main] Завершення програми");
}
```

**Очікуваний вивід:**

```text
[Counter-1] Актор створений
[Counter-1] Цикл обробки запущено
[Counter-1] Increment → 1
[Counter-1] Increment → 2
[Counter-1] Increment → 3
[Counter-1] Поточне значення: 3
[Counter-1] Decrement → 2
[Counter-1] Поточне значення: 2
[main] Завершення програми
[Counter-1] Цикл обробки завершено
```

---

## 42.3 REQUEST-RESPONSE: ОТРИМАННЯ ВІДПОВІДІ ВІД АКТОРА

### 42.3.1 Проблема fire-and-forget

Базовий патерн з попереднього розділу має обмеження: ми надсилаємо повідомлення і не отримуємо відповіді. Методи `increment()` та `decrement()` нічого не повертають. Але що, якщо нам потрібно **запитати** поточне значення лічильника?

```text
Проблема:
─────────

Handle                           Actor
   │                               │
   │  Increment                    │
   │ ─────────────────────────────►│
   │                               │
   │  GetValue ???                 │
   │ ─────────────────────────────►│
   │                               │
   │  Як отримати відповідь?       │
   │◄─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│
   │                               │
```

### 42.3.2 Рішення: oneshot канал у повідомленні

Ідея проста: включаємо `oneshot::Sender` прямо в повідомлення. Актор обробляє запит і надсилає відповідь через цей канал:

```text
Рішення:
────────

Handle                                Actor
   │                                    │
   │  GetValue { respond_to: tx }       │
   │ ──────────────────────────────────►│
   │                                    │
   │        oneshot::Receiver           │ process
   │◄ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│
   │                                    │
   │  await response                    │
   │◄──────────────────────────────────┤
   │        value: 42                   │
   │                                    │
```

### 42.3.3 Реалізація request-response

**Постановка задачі:** Модифікуємо лічильник, щоб можна було запитувати поточне значення.

```rust
use tokio::sync::{mpsc, oneshot};

/// Повідомлення з підтримкою запитів
enum CounterMessage {
    Increment,
    Decrement,
    
    /// Запит поточного значення
    /// respond_to — канал для надсилання відповіді
    GetValue {
        respond_to: oneshot::Sender<i32>,
    },
}

struct CounterActor {
    value: i32,
    name: String,
}

impl CounterActor {
    fn new(name: String) -> Self {
        Self { value: 0, name }
    }
    
    fn handle_message(&mut self, msg: CounterMessage) {
        match msg {
            CounterMessage::Increment => {
                self.value += 1;
            }
            CounterMessage::Decrement => {
                self.value -= 1;
            }
            CounterMessage::GetValue { respond_to } => {
                // Надсилаємо відповідь через oneshot канал
                // Ігноруємо помилку (запитувач міг "піти")
                let _ = respond_to.send(self.value);
            }
        }
    }
    
    async fn run(mut self, mut rx: mpsc::Receiver<CounterMessage>) {
        while let Some(msg) = rx.recv().await {
            self.handle_message(msg);
        }
    }
}

#[derive(Clone)]
struct CounterHandle {
    tx: mpsc::Sender<CounterMessage>,
}

impl CounterHandle {
    fn new(name: &str) -> Self {
        let (tx, rx) = mpsc::channel(32);
        let actor = CounterActor::new(name.to_string());
        tokio::spawn(actor.run(rx));
        Self { tx }
    }
    
    pub async fn increment(&self) {
        let _ = self.tx.send(CounterMessage::Increment).await;
    }
    
    pub async fn decrement(&self) {
        let _ = self.tx.send(CounterMessage::Decrement).await;
    }
    
    /// Запит поточного значення
    /// Повертає Option<i32> — None якщо актор завершився
    pub async fn get_value(&self) -> Option<i32> {
        // Створюємо oneshot канал для відповіді
        let (tx, rx) = oneshot::channel();
        
        // Надсилаємо запит з каналом відповіді
        self.tx.send(CounterMessage::GetValue { respond_to: tx }).await.ok()?;
        
        // Чекаємо відповідь
        rx.await.ok()
    }
}

#[tokio::main]
async fn main() {
    let counter = CounterHandle::new("Counter");
    
    counter.increment().await;
    counter.increment().await;
    counter.increment().await;
    
    // Запитуємо значення
    if let Some(value) = counter.get_value().await {
        println!("Поточне значення: {}", value);
    }
    
    counter.decrement().await;
    
    if let Some(value) = counter.get_value().await {
        println!("Після decrement: {}", value);
    }
}
```

**Розглянемо детально метод `get_value()`:**

1. **Створення oneshot каналу**: `let (tx, rx) = oneshot::channel()` створює пару — Sender для актора, Receiver для нас.

2. **Відправка запиту**: Включаємо `tx` у повідомлення. Тепер актор має канал для відповіді.

3. **Очікування відповіді**: `rx.await` чекає, поки актор не надішле значення через `tx.send()`.

4. **Обробка помилок**: `ok()?` перетворює помилки на `None`. Помилка може виникнути, якщо актор завершився до обробки запиту.

---

## 42.4 GRACEFUL SHUTDOWN: КОРЕКТНЕ ЗАВЕРШЕННЯ

### 42.4.1 Два способи завершення актора

Актор може завершитись двома способами:

**1. Автоматично (drop всіх handle):**

Коли всі `CounterHandle` знищуються (виходять зі scope або явно drop), всі `Sender` знищуються. Канал закривається. `rx.recv().await` повертає `None`. Цикл завершується.

```text
handle1 ─┐
handle2 ─┼──► tx клони ──► канал ──► rx ──► актор
handle3 ─┘

drop(handle1)
drop(handle2)
drop(handle3)
         │
         ▼
    канал закрито
         │
         ▼
    rx.recv() → None
         │
         ▼
    цикл завершено
```

**2. Явно (повідомлення Shutdown):**

Іноді потрібно завершити актора явно, не чекаючи drop всіх handle. Для цього додаємо спеціальне повідомлення.

### 42.4.2 Реалізація явного shutdown

**Постановка задачі:** Додамо можливість явного завершення актора та очікування його завершення.

```rust
use tokio::sync::{mpsc, oneshot};
use tokio::task::JoinHandle;

enum CounterMessage {
    Increment,
    Decrement,
    GetValue { respond_to: oneshot::Sender<i32> },
    
    /// Сигнал завершення роботи
    Shutdown,
}

struct CounterActor {
    value: i32,
}

impl CounterActor {
    fn new() -> Self {
        Self { value: 0 }
    }
    
    /// Обробка повідомлення
    /// Повертає false якщо потрібно завершити цикл
    fn handle_message(&mut self, msg: CounterMessage) -> bool {
        match msg {
            CounterMessage::Increment => {
                self.value += 1;
                true  // продовжуємо
            }
            CounterMessage::Decrement => {
                self.value -= 1;
                true  // продовжуємо
            }
            CounterMessage::GetValue { respond_to } => {
                let _ = respond_to.send(self.value);
                true  // продовжуємо
            }
            CounterMessage::Shutdown => {
                println!("[Actor] Отримано Shutdown, завершую...");
                false  // завершуємо цикл
            }
        }
    }
    
    async fn run(mut self, mut rx: mpsc::Receiver<CounterMessage>) {
        println!("[Actor] Запущено");
        
        while let Some(msg) = rx.recv().await {
            if !self.handle_message(msg) {
                break;  // Shutdown
            }
        }
        
        println!("[Actor] Завершено, фінальне значення: {}", self.value);
    }
}

struct CounterHandle {
    tx: mpsc::Sender<CounterMessage>,
    /// JoinHandle для очікування завершення актора
    join_handle: JoinHandle<()>,
}

impl CounterHandle {
    fn new() -> Self {
        let (tx, rx) = mpsc::channel(32);
        let actor = CounterActor::new();
        
        // Зберігаємо JoinHandle
        let join_handle = tokio::spawn(actor.run(rx));
        
        Self { tx, join_handle }
    }
    
    pub async fn increment(&self) {
        let _ = self.tx.send(CounterMessage::Increment).await;
    }
    
    pub async fn get_value(&self) -> Option<i32> {
        let (tx, rx) = oneshot::channel();
        self.tx.send(CounterMessage::GetValue { respond_to: tx }).await.ok()?;
        rx.await.ok()
    }
    
    /// Завершити актора і дочекатись його завершення
    pub async fn shutdown(self) {
        // Надсилаємо Shutdown
        let _ = self.tx.send(CounterMessage::Shutdown).await;
        
        // Чекаємо завершення задачі
        let _ = self.join_handle.await;
        
        println!("[Handle] Актор завершено");
    }
}

#[tokio::main]
async fn main() {
    let counter = CounterHandle::new();
    
    counter.increment().await;
    counter.increment().await;
    
    println!("Значення: {:?}", counter.get_value().await);
    
    // Явне завершення
    counter.shutdown().await;
    
    println!("[main] Програма завершена");
}
```

**Вивід:**

```text
[Actor] Запущено
Значення: Some(2)
[Actor] Отримано Shutdown, завершую...
[Actor] Завершено, фінальне значення: 2
[Handle] Актор завершено
[main] Програма завершена
```

**Ключові моменти:**

1. **JoinHandle**: Зберігаємо handle задачі, щоб можна було дочекатись її завершення.

2. **shutdown(self)**: Метод споживає handle — після shutdown взаємодія неможлива.

3. **join_handle.await**: Чекаємо, поки задача актора завершиться. Гарантуємо, що всі cleanup операції виконані.

---

## 42.5 ACTOR TICK: ПЕРІОДИЧНІ ДІЇ

### 42.5.1 Проблема: актор, що робить щось періодично

Іноді актор має виконувати дії не тільки у відповідь на повідомлення, а й **періодично**. Наприклад:
- БПЛА-агент зменшує батарею кожні 100мс
- Сенсор перевіряє показання кожну секунду
- Heartbeat кожні 5 секунд

Як реалізувати періодичні дії, якщо цикл актора блокується на `rx.recv().await`?

### 42.5.2 Рішення: select! для кількох джерел подій

`tokio::select!` дозволяє чекати на кілька futures одночасно і реагувати на той, що завершився першим:

```rust
use tokio::sync::mpsc;
use tokio::time::{interval, Duration, Interval};

enum AgentMessage {
    Command(String),
    GetStatus { respond_to: tokio::sync::oneshot::Sender<String> },
    Shutdown,
}

struct AgentActor {
    id: u32,
    battery: u8,
    tick_count: u32,
}

impl AgentActor {
    fn new(id: u32) -> Self {
        Self {
            id,
            battery: 100,
            tick_count: 0,
        }
    }
    
    fn handle_message(&mut self, msg: AgentMessage) -> bool {
        match msg {
            AgentMessage::Command(cmd) => {
                println!("[Agent {}] Команда: {}", self.id, cmd);
                true
            }
            AgentMessage::GetStatus { respond_to } => {
                let status = format!(
                    "Agent {}: battery={}%, ticks={}",
                    self.id, self.battery, self.tick_count
                );
                let _ = respond_to.send(status);
                true
            }
            AgentMessage::Shutdown => {
                println!("[Agent {}] Shutdown", self.id);
                false
            }
        }
    }
    
    fn handle_tick(&mut self) {
        self.tick_count += 1;
        
        // Зменшуємо батарею
        if self.battery > 0 {
            self.battery = self.battery.saturating_sub(1);
        }
        
        // Попередження при низькому заряді
        if self.battery == 10 {
            println!("[Agent {}] ⚠️ Низький заряд батареї!", self.id);
        }
    }
    
    /// Цикл з підтримкою tick
    async fn run(mut self, mut rx: mpsc::Receiver<AgentMessage>) {
        // Таймер для tick кожні 500мс
        let mut tick_interval = interval(Duration::from_millis(500));
        
        println!("[Agent {}] Запущено", self.id);
        
        loop {
            // select! чекає на ОБИДВА джерела одночасно
            // Реагуємо на те, що готове першим
            tokio::select! {
                // Гілка 1: Прийшло повідомлення
                msg = rx.recv() => {
                    match msg {
                        Some(m) => {
                            if !self.handle_message(m) {
                                break; // Shutdown
                            }
                        }
                        None => {
                            // Канал закрито
                            println!("[Agent {}] Канал закрито", self.id);
                            break;
                        }
                    }
                }
                
                // Гілка 2: Спрацював таймер
                _ = tick_interval.tick() => {
                    self.handle_tick();
                }
            }
        }
        
        println!("[Agent {}] Завершено", self.id);
    }
}
```

**Розглянемо, як працює select! у цьому контексті:**

1. **Два джерела подій**: `rx.recv()` (повідомлення) та `tick_interval.tick()` (таймер).

2. **Конкурентне очікування**: `select!` чекає на обидва одночасно.

3. **Перша готова гілка виграє**: Якщо прийшло повідомлення — обробляємо повідомлення. Якщо спрацював таймер — обробляємо tick.

4. **Після обробки — знову select!**: Цикл повертається до очікування.

### 42.5.3 Повний приклад з tick

**Постановка задачі:** Агент зменшує батарею кожну секунду та обробляє команди.

```rust
use tokio::sync::{mpsc, oneshot};
use tokio::time::{interval, sleep, Duration};

enum AgentMessage {
    Command(String),
    GetStatus { respond_to: oneshot::Sender<String> },
    Shutdown,
}

struct AgentActor {
    id: u32,
    battery: u8,
}

impl AgentActor {
    fn new(id: u32) -> Self {
        Self { id, battery: 100 }
    }
    
    fn handle_message(&mut self, msg: AgentMessage) -> bool {
        match msg {
            AgentMessage::Command(cmd) => {
                println!("[Agent {}] Виконую: {}", self.id, cmd);
                // Команди витрачають батарею
                self.battery = self.battery.saturating_sub(5);
                true
            }
            AgentMessage::GetStatus { respond_to } => {
                let _ = respond_to.send(format!("Battery: {}%", self.battery));
                true
            }
            AgentMessage::Shutdown => false,
        }
    }
    
    fn tick(&mut self) {
        // Пасивне споживання батареї
        self.battery = self.battery.saturating_sub(1);
        println!("[Agent {}] Tick, battery: {}%", self.id, self.battery);
        
        if self.battery == 0 {
            println!("[Agent {}] ⚠️ БАТАРЕЯ РОЗРЯДЖЕНА!", self.id);
        }
    }
    
    async fn run(mut self, mut rx: mpsc::Receiver<AgentMessage>) {
        let mut tick_timer = interval(Duration::from_secs(1));
        
        loop {
            tokio::select! {
                msg = rx.recv() => {
                    match msg {
                        Some(m) if self.handle_message(m) => {}
                        _ => break,
                    }
                }
                _ = tick_timer.tick() => {
                    self.tick();
                }
            }
        }
    }
}

#[derive(Clone)]
struct AgentHandle {
    tx: mpsc::Sender<AgentMessage>,
}

impl AgentHandle {
    fn spawn(id: u32) -> Self {
        let (tx, rx) = mpsc::channel(32);
        tokio::spawn(AgentActor::new(id).run(rx));
        Self { tx }
    }
    
    pub async fn command(&self, cmd: &str) {
        let _ = self.tx.send(AgentMessage::Command(cmd.to_string())).await;
    }
    
    pub async fn get_status(&self) -> Option<String> {
        let (tx, rx) = oneshot::channel();
        self.tx.send(AgentMessage::GetStatus { respond_to: tx }).await.ok()?;
        rx.await.ok()
    }
    
    pub async fn shutdown(&self) {
        let _ = self.tx.send(AgentMessage::Shutdown).await;
    }
}

#[tokio::main]
async fn main() {
    let agent = AgentHandle::spawn(1);
    
    // Надсилаємо кілька команд
    agent.command("SCAN").await;
    sleep(Duration::from_millis(500)).await;
    
    agent.command("MOVE").await;
    sleep(Duration::from_millis(500)).await;
    
    // Чекаємо, спостерігаємо tick'и
    sleep(Duration::from_secs(3)).await;
    
    // Перевіряємо статус
    if let Some(status) = agent.get_status().await {
        println!("Статус: {}", status);
    }
    
    // Завершуємо
    agent.shutdown().await;
    
    // Даємо час на завершення
    sleep(Duration::from_millis(100)).await;
}
```

---

## 42.6 АГЕНТ БПЛА ЯК АКТОР: ПОВНА РЕАЛІЗАЦІЯ

### 42.6.1 Структура агента

**Постановка задачі:** Реалізуємо повноцінного агента БПЛА як актора з усіма вивченими патернами.

```rust
use tokio::sync::{mpsc, oneshot};
use tokio::time::{interval, Duration};

// === ТИПИ ДАНИХ ===

#[derive(Debug, Clone, Copy)]
struct Position {
    x: f64,
    y: f64,
}

#[derive(Debug, Clone, Copy, PartialEq)]
enum DroneMode {
    Idle,
    Patrolling,
    Returning,
    Emergency,
}

#[derive(Debug, Clone)]
struct DroneStatus {
    id: u32,
    position: Position,
    battery: u8,
    mode: DroneMode,
}

// === ПОВІДОМЛЕННЯ ===

enum DroneMessage {
    /// Команда руху до точки
    MoveTo { x: f64, y: f64 },
    
    /// Почати патрулювання
    StartPatrol,
    
    /// Повернутись на базу
    ReturnToBase,
    
    /// Запит статусу
    GetStatus { respond_to: oneshot::Sender<DroneStatus> },
    
    /// Завершення роботи
    Shutdown,
}

// === АКТОР ===

struct DroneActor {
    id: u32,
    position: Position,
    battery: u8,
    mode: DroneMode,
    base_position: Position,
}

impl DroneActor {
    fn new(id: u32) -> Self {
        Self {
            id,
            position: Position { x: 0.0, y: 0.0 },
            battery: 100,
            mode: DroneMode::Idle,
            base_position: Position { x: 0.0, y: 0.0 },
        }
    }
    
    fn handle_message(&mut self, msg: DroneMessage) -> bool {
        match msg {
            DroneMessage::MoveTo { x, y } => {
                if self.mode != DroneMode::Emergency {
                    println!("[Drone {}] Рухаюсь до ({:.1}, {:.1})", self.id, x, y);
                    self.position = Position { x, y };
                    self.consume_battery(5);
                }
                true
            }
            
            DroneMessage::StartPatrol => {
                if self.battery > 20 {
                    println!("[Drone {}] Починаю патрулювання", self.id);
                    self.mode = DroneMode::Patrolling;
                } else {
                    println!("[Drone {}] Недостатньо батареї для патрулювання", self.id);
                }
                true
            }
            
            DroneMessage::ReturnToBase => {
                println!("[Drone {}] Повертаюсь на базу", self.id);
                self.mode = DroneMode::Returning;
                self.position = self.base_position;
                true
            }
            
            DroneMessage::GetStatus { respond_to } => {
                let status = DroneStatus {
                    id: self.id,
                    position: self.position,
                    battery: self.battery,
                    mode: self.mode,
                };
                let _ = respond_to.send(status);
                true
            }
            
            DroneMessage::Shutdown => {
                println!("[Drone {}] Отримано команду завершення", self.id);
                false
            }
        }
    }
    
    fn tick(&mut self) {
        // Пасивне споживання
        self.consume_battery(1);
        
        // Автоматичне повернення при низькому заряді
        if self.battery <= 15 && self.mode == DroneMode::Patrolling {
            println!("[Drone {}] ⚠️ Низький заряд, повертаюсь на базу", self.id);
            self.mode = DroneMode::Returning;
        }
        
        // Emergency при критичному рівні
        if self.battery <= 5 && self.mode != DroneMode::Emergency {
            println!("[Drone {}] 🚨 КРИТИЧНИЙ РІВЕНЬ БАТАРЕЇ!", self.id);
            self.mode = DroneMode::Emergency;
        }
    }
    
    fn consume_battery(&mut self, amount: u8) {
        self.battery = self.battery.saturating_sub(amount);
    }
    
    async fn run(mut self, mut rx: mpsc::Receiver<DroneMessage>) {
        let mut tick_timer = interval(Duration::from_millis(500));
        
        println!("[Drone {}] Актор запущено", self.id);
        
        loop {
            tokio::select! {
                msg = rx.recv() => {
                    match msg {
                        Some(m) if self.handle_message(m) => {}
                        _ => break,
                    }
                }
                _ = tick_timer.tick() => {
                    self.tick();
                }
            }
        }
        
        println!("[Drone {}] Актор завершено", self.id);
    }
}

// === HANDLE ===

#[derive(Clone)]
struct DroneHandle {
    tx: mpsc::Sender<DroneMessage>,
}

impl DroneHandle {
    fn spawn(id: u32) -> Self {
        let (tx, rx) = mpsc::channel(64);
        let actor = DroneActor::new(id);
        tokio::spawn(actor.run(rx));
        Self { tx }
    }
    
    pub async fn move_to(&self, x: f64, y: f64) {
        let _ = self.tx.send(DroneMessage::MoveTo { x, y }).await;
    }
    
    pub async fn start_patrol(&self) {
        let _ = self.tx.send(DroneMessage::StartPatrol).await;
    }
    
    pub async fn return_to_base(&self) {
        let _ = self.tx.send(DroneMessage::ReturnToBase).await;
    }
    
    pub async fn get_status(&self) -> Option<DroneStatus> {
        let (tx, rx) = oneshot::channel();
        self.tx.send(DroneMessage::GetStatus { respond_to: tx }).await.ok()?;
        rx.await.ok()
    }
    
    pub async fn shutdown(&self) {
        let _ = self.tx.send(DroneMessage::Shutdown).await;
    }
}
```

### 42.6.2 Демонстрація роботи агента

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    println!("=== Демонстрація агента-актора ===\n");
    
    // Створюємо агента
    let drone = DroneHandle::spawn(1);
    
    // Початковий статус
    if let Some(status) = drone.get_status().await {
        println!("Початковий статус: {:?}\n", status);
    }
    
    // Починаємо патрулювання
    drone.start_patrol().await;
    
    // Виконуємо кілька переміщень
    drone.move_to(10.0, 20.0).await;
    sleep(Duration::from_secs(1)).await;
    
    drone.move_to(30.0, 40.0).await;
    sleep(Duration::from_secs(1)).await;
    
    // Перевіряємо статус
    if let Some(status) = drone.get_status().await {
        println!("\nПоточний статус: {:?}\n", status);
    }
    
    // Чекаємо, поки батарея знизиться
    println!("Чекаємо розряду батареї...\n");
    sleep(Duration::from_secs(10)).await;
    
    // Фінальний статус
    if let Some(status) = drone.get_status().await {
        println!("\nФінальний статус: {:?}\n", status);
    }
    
    // Завершуємо
    drone.shutdown().await;
    sleep(Duration::from_millis(100)).await;
    
    println!("\n=== Демонстрація завершена ===");
}
```

---

## 42.7 ЛАБОРАТОРНА РОБОТА

### Мета роботи

Закріпити практичні навички реалізації акторів на Tokio.

### Завдання 1: Актор-термометр (3 бали)

Реалізуйте актора "Термометр" з наступними можливостями:
- Повідомлення: `SetTemperature(f64)`, `GetTemperature`, `GetAverage`
- Стан: поточна температура та історія останніх 10 вимірювань
- Handle з відповідними методами

### Завдання 2: Агент з автономною поведінкою (4 бали)

Модифікуйте DroneActor:
- Додайте поле `target: Option<Position>`
- У tick: якщо є target — рухатись до нього (на 1 одиницю за tick)
- Коли досягнуто target — вивести повідомлення та очистити target

### Завдання 3: Координатор рою (3 бали)

Створіть `CoordinatorActor`:
- Зберігає `Vec<DroneHandle>` для 5 агентів
- Повідомлення: `BroadcastCommand(String)`, `CollectStatuses`
- `CollectStatuses` повертає `Vec<DroneStatus>` від усіх агентів

---

## 42.8 ТИПОВІ ПОМИЛКИ ТА TROUBLESHOOTING

### 42.8.1 Актор не обробляє повідомлення

**Симптом:** Надсилаємо повідомлення, але нічого не відбувається.

**Можливі причини:**
1. Забули `tokio::spawn(actor.run(rx))`
2. Канал закрито до надсилання
3. Помилка в handle_message (panic)

**Діагностика:** Додайте логування на початку `run()` та в `handle_message()`.

### 42.8.2 Handle метод "зависає"

**Симптом:** Виклик `get_status().await` ніколи не повертається.

**Можливі причини:**
1. Актор завершився до обробки
2. Deadlock через oneshot
3. Актор обробляє повідомлення занадто довго

**Рішення:** Додайте timeout:

```rust
pub async fn get_status_timeout(&self, timeout: Duration) -> Option<DroneStatus> {
    let (tx, rx) = oneshot::channel();
    self.tx.send(DroneMessage::GetStatus { respond_to: tx }).await.ok()?;
    tokio::time::timeout(timeout, rx).await.ok()?.ok()
}
```

### 42.8.3 Пам'ять росте без обмежень

**Симптом:** Програма споживає все більше пам'яті.

**Причина:** Unbounded канал або занадто великий буфер при швидкому producer.

**Рішення:**
1. Використовуйте bounded канали (ми використовуємо `channel(32)`)
2. Відстежуйте backpressure
3. Додайте метрики розміру черги

---

## 42.9 РЕЗЮМЕ

### Ключові концепції

**Актор на Tokio** = `tokio::spawn` + `mpsc::channel` + struct зі станом.

**Три компоненти:**
1. **Message enum** — типи повідомлень
2. **Actor struct** — стан + `handle_message()` + `run()`
3. **Handle struct** — Sender + зручні методи

**Request-Response:** Включаємо `oneshot::Sender` у повідомлення для отримання відповіді.

**Graceful Shutdown:** Повідомлення `Shutdown` + `break` з циклу + `JoinHandle::await`.

**Actor Tick:** `tokio::select!` для одночасного очікування повідомлень та таймера.

### Патерн для запам'ятовування

```rust
// 1. Повідомлення
enum Msg {
    Command { data },
    Query { respond_to: oneshot::Sender<T> },
    Shutdown,
}

// 2. Актор
struct Actor { state }
impl Actor {
    fn handle(&mut self, msg: Msg) -> bool { ... }
    async fn run(mut self, mut rx: Receiver<Msg>) {
        while let Some(msg) = rx.recv().await {
            if !self.handle(msg) { break; }
        }
    }
}

// 3. Handle
struct Handle { tx: Sender<Msg> }
impl Handle {
    fn new() -> Self {
        let (tx, rx) = channel(32);
        tokio::spawn(Actor::new().run(rx));
        Self { tx }
    }
}
```

---

## 🔗 ЗВ'ЯЗОК З НАСТУПНИМ РОЗДІЛОМ

Ви опанували Actor Model для **async** коду — операцій з очікуванням (мережа, таймери, канали). Але що робити з **CPU-bound** обчисленнями? Наприклад, обробка зображень з камери дрона, розрахунок оптимального шляху для всього рою, аналіз великих масивів даних?

Async не допомагає, коли процесор завантажений на 100% — тут потрібен **паралелізм на рівні CPU ядер**.

У **Розділі 43: Rayon — Data Parallelism** ви:
- Дізнаєтесь про data parallelism та його відмінність від async
- Навчитесь використовувати `par_iter()` для паралельної обробки
- Зрозумієте, коли Rayon, а коли Tokio
- Інтегруєте Rayon з async кодом через `spawn_blocking`

---

> **Наступний розділ:** [Розділ 43: Rayon — Data Parallelism](./43_Rayon.md)
