# Розділ 44: Практикум — Асинхронний рій v4.0

---

## 📋 Анотація

Це кульмінаційний розділ Частини IV. За попередні сім розділів ви опанували потужний арсенал інструментів: async/await для конкурентності без блокування, Tokio як production-ready runtime, різні типи каналів для комунікації, streams для потокової обробки даних, Actor Model для структурування системи, Rayon для паралельних обчислень. Тепер час **об'єднати все разом** у повноцінну мультиагентну систему.

**Асинхронний рій v4.0** — це не просто "ще одна версія" проєкту. Це якісний стрибок порівняно з v3.0 (потоки + mutex):

| Характеристика | v3.0 (Потоки) | v4.0 (Async) |
|----------------|---------------|--------------|
| Кількість агентів | 10-50 | 1000+ |
| Пам'ять на агента | ~2 МБ (стек потоку) | ~2 КБ (задача) |
| Координація | Mutex + каналів | Actor Model |
| CPU-bound задачі | Ті ж потоки | Окремий Rayon пул |
| Shutdown | Складний | Елегантний (watch) |

У цьому розділі ви побудуєте систему, де **сотні агентів** патрулюють територію, виявляють цілі, координуються через центральний координатор, і все це — ефективно, масштабовано, з graceful shutdown. Це production-ready архітектура, яку можна адаптувати для реальних застосувань.

---

## 🎯 Цілі навчання

Після завершення цього розділу ви зможете:

1. **Проєктувати** архітектуру async мультиагентної системи
2. **Обирати** правильні типи каналів для різних патернів комунікації
3. **Реалізовувати** агентів як акторів на Tokio
4. **Інтегрувати** CPU-bound обчислення через spawn_blocking + Rayon
5. **Імплементувати** graceful shutdown всієї системи
6. **Оцінювати** масштабованість та оптимізувати bottlenecks

---

## 📚 Ключові терміни

| Термін | Визначення |
|--------|------------|
| **Рій (Swarm)** | Колекція автономних агентів, що координуються для досягнення спільної мети |
| **Координатор** | Центральний актор, що керує роєм: розподіляє завдання, збирає звіти |
| **Broadcast** | Патерн "один до багатьох": одне повідомлення доставляється всім підписникам |
| **Fan-in** | Патерн "багато до одного": багато джерел надсилають до одного отримувача |
| **Graceful Shutdown** | Коректне завершення: всі компоненти отримують сигнал і завершуються впорядковано |
| **Watch Channel** | Канал для broadcast одного значення з можливістю перевірки поточного стану |

---

## 💡 Мотиваційний кейс: Пошукова місія

Уявімо реальний сценарій: пошукова операція на території 10×10 км. Потрібно знайти всі об'єкти інтересу (цілі) у мінімальний час.

**Задача:**
- Територія розділена на сектори
- 100 БПЛА мають покрити всю територію
- При виявленні цілі — негайно повідомити координатора
- Координатор може перенаправити агентів до знайдених цілей
- Місія триває 30 хвилин, потім — повернення на базу

**Архітектурні вимоги:**
1. **Масштабованість**: 100 агентів не повинні "з'їдати" всі ресурси
2. **Реактивність**: Команди доставляються миттєво всім
3. **Надійність**: Система має коректно завершуватись
4. **Розширюваність**: Легко додати нові типи агентів або команд

Actor Model на Tokio ідеально підходить для цих вимог.

---

## 44.1 АРХІТЕКТУРА СИСТЕМИ

### 44.1.1 Компоненти системи

Система складається з трьох основних типів компонентів:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    АРХІТЕКТУРА РОЮ v4.0                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│                    ┌───────────────────────┐                       │
│                    │     КООРДИНАТОР       │                       │
│                    │                       │                       │
│                    │  • Керування місією   │                       │
│                    │  • Розподіл завдань   │                       │
│                    │  • Збір звітів        │                       │
│                    │  • Shutdown control   │                       │
│                    └───────────┬───────────┘                       │
│                                │                                    │
│              ┌─────────────────┼─────────────────┐                 │
│              │                 │                 │                 │
│              ▼                 ▼                 ▼                 │
│       ┌───────────┐     ┌───────────┐     ┌───────────┐           │
│       │ АГЕНТ #1  │     │ АГЕНТ #2  │     │ АГЕНТ #N  │           │
│       │           │     │           │     │           │           │
│       │ • Патруль │     │ • Патруль │     │ • Патруль │           │
│       │ • Пошук   │     │ • Пошук   │     │ • Пошук   │           │
│       │ • Звіти   │     │ • Звіти   │     │ • Звіти   │           │
│       └─────┬─────┘     └─────┬─────┘     └─────┬─────┘           │
│             │                 │                 │                  │
│             └─────────────────┼─────────────────┘                  │
│                               │                                    │
│                               ▼                                    │
│                    ┌───────────────────────┐                       │
│                    │        СВІТ           │                       │
│                    │                       │                       │
│                    │  • Територія          │                       │
│                    │  • Цілі               │                       │
│                    │  • Перешкоди          │                       │
│                    │  (Arc<RwLock<World>>) │                       │
│                    └───────────────────────┘                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Координатор (Coordinator)**

"Мозок" системи. Єдиний екземпляр, що:
- Запускає та завершує місію
- Надсилає команди всім агентам (broadcast)
- Отримує звіти від агентів (fan-in)
- Приймає стратегічні рішення
- Ініціює graceful shutdown

**Агенти (Agents)**

Автономні одиниці, кожен — окрема Tokio задача. Агент:
- Отримує команди від координатора
- Виконує патрулювання території
- Виявляє цілі та повідомляє координатора
- Керує своїм станом (батарея, позиція, режим)
- Реагує на сигнал shutdown

**Світ (World)**

Спільний стан середовища. Містить:
- Розміри території
- Позиції цілей
- Інформацію про виявлені об'єкти
- Позицію бази

Доступ до Світу — через `Arc<RwLock<World>>`. Це єдиний спільний мутабельний стан у системі.

### 44.1.2 Потоки даних

Комунікація в системі організована через канали. Важливо обрати правильний тип каналу для кожного патерну:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ПОТОКИ ДАНИХ                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│                      КООРДИНАТОР                                    │
│                           │                                         │
│         ┌─────────────────┼─────────────────┐                      │
│         │                 │                 │                      │
│         │    BROADCAST    │    WATCH        │    MPSC              │
│         │    (команди)    │    (shutdown)   │    (звіти)           │
│         │                 │                 │                      │
│         ▼                 ▼                 │                      │
│    ┌─────────┐       ┌─────────┐           │                      │
│    │         │       │         │           │                      │
│    │ Agent 1 ├───────┤ Agent 1 │───────────┤                      │
│    │         │       │         │           │                      │
│    ├─────────┤       ├─────────┤           │                      │
│    │         │       │         │           │                      │
│    │ Agent 2 ├───────┤ Agent 2 │───────────┤                      │
│    │         │       │         │           │                      │
│    ├─────────┤       ├─────────┤           │                      │
│    │   ...   │       │   ...   │           │                      │
│    │         │       │         │           │                      │
│    ├─────────┤       ├─────────┤           │                      │
│    │         │       │         │           │                      │
│    │ Agent N ├───────┤ Agent N │───────────┘                      │
│    │         │       │         │           ▲                      │
│    └─────────┘       └─────────┘           │                      │
│                                            │                      │
│         Координатор надсилає         Агенти надсилають            │
│         ОДНУ команду —               КОЖЕН свій звіт —            │
│         отримують ВСІ                отримує ОДИН                 │
│                                      (координатор)                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 44.1.3 Вибір каналів: обґрунтування

Чому саме такі типи каналів? Розглянемо кожен випадок детально.

**Команди: broadcast::channel**

Патерн: один відправник (координатор) → багато отримувачів (всі агенти).

```
Координатор: send(StartPatrol)
    │
    ├──► Agent 1: recv() → StartPatrol
    ├──► Agent 2: recv() → StartPatrol
    ├──► Agent 3: recv() → StartPatrol
    └──► Agent N: recv() → StartPatrol
```

Чому broadcast?
- Кожен агент отримує **копію** повідомлення
- Агенти можуть приєднуватись/від'єднуватись динамічно (subscribe/drop)
- Якщо агент "відстав" — старі повідомлення пропускаються (lagged)

Альтернатива (mpsc до кожного агента) потребувала б зберігати `Vec<Sender>` та ітерувати.

**Звіти: mpsc::channel**

Патерн: багато відправників (агенти) → один отримувач (координатор).

```
Agent 1: send(StatusReport) ──┐
Agent 2: send(TargetFound)  ──┼──► Координатор: recv()
Agent 3: send(StatusReport) ──┤
Agent N: send(LowBattery)   ──┘
```

Чому mpsc?
- Класичний fan-in патерн
- Sender можна клонувати для кожного агента
- Один Receiver у координатора
- Повідомлення не губляться (bounded buffer)

**Shutdown: watch::channel**

Патерн: одне значення, що змінюється, багато спостерігачів.

```
Координатор: tx.send(true)  // Сигнал shutdown
    │
    ├──► Agent 1: rx.changed().await → *rx.borrow() == true
    ├──► Agent 2: rx.changed().await → *rx.borrow() == true
    └──► Agent N: rx.changed().await → *rx.borrow() == true
```

Чому watch?
- Не потрібна черга повідомлень — тільки поточний стан
- Агент може перевірити стан без очікування
- Ефективніший за broadcast для boolean сигналів
- Гарантує, що новий агент одразу бачить поточний стан

**Порівняльна таблиця вибору каналів:**

| Патерн | Канал | Коли використовувати |
|--------|-------|----------------------|
| Один → Один | `mpsc` | Приватна комунікація actor-actor |
| Один → Багато (черга) | `broadcast` | Команди, події, сповіщення |
| Один → Багато (стан) | `watch` | Конфігурація, shutdown, прапорці |
| Багато → Один | `mpsc` | Збір звітів, fan-in |
| Запит-Відповідь | `oneshot` у повідомленні | Синхронні запити |

---

## 44.2 МОДЕЛЬ ДАНИХ

### 44.2.1 Базові типи

Перш ніж писати акторів, визначимо типи даних, якими вони оперують.

**Постановка задачі:** Створимо базові типи для представлення координат, ідентифікаторів та режимів роботи агентів.

```rust
use std::fmt;

/// Унікальний ідентифікатор агента
pub type AgentId = u32;

/// Позиція у 2D просторі
#[derive(Debug, Clone, Copy, PartialEq)]
pub struct Position {
    pub x: f64,
    pub y: f64,
}

impl Position {
    pub fn new(x: f64, y: f64) -> Self {
        Self { x, y }
    }
    
    /// Відстань до іншої точки
    pub fn distance_to(&self, other: &Position) -> f64 {
        let dx = self.x - other.x;
        let dy = self.y - other.y;
        (dx * dx + dy * dy).sqrt()
    }
    
    /// Рух до цілі з обмеженою швидкістю
    pub fn move_towards(&mut self, target: &Position, speed: f64) {
        let dist = self.distance_to(target);
        if dist < speed {
            // Досягли цілі
            *self = *target;
        } else {
            // Рухаємось у напрямку цілі
            let ratio = speed / dist;
            self.x += (target.x - self.x) * ratio;
            self.y += (target.y - self.y) * ratio;
        }
    }
}

impl fmt::Display for Position {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({:.1}, {:.1})", self.x, self.y)
    }
}

/// Режим роботи агента
#[derive(Debug, Clone, Copy, PartialEq)]
pub enum AgentMode {
    /// Очікування на базі
    Idle,
    /// Активне патрулювання
    Patrolling,
    /// Повернення на базу
    Returning,
    /// Аварійний режим
    Emergency,
}

impl fmt::Display for AgentMode {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AgentMode::Idle => write!(f, "IDLE"),
            AgentMode::Patrolling => write!(f, "PATROL"),
            AgentMode::Returning => write!(f, "RETURN"),
            AgentMode::Emergency => write!(f, "EMERG"),
        }
    }
}
```

**Розглянемо ключові рішення:**

1. **Position як Copy**: Позиція — маленький тип (16 байт), копіювання дешевше за посилання.

2. **move_towards**: Метод для плавного руху агента. Обмежує переміщення за один tick.

3. **AgentMode як enum**: Чітко визначені стани спрощують логіку переходів.

### 44.2.2 Модель світу

**Постановка задачі:** Створимо структуру для представлення середовища — території з цілями.

```rust
use std::sync::Arc;
use tokio::sync::RwLock;

/// Ціль для виявлення
#[derive(Debug, Clone)]
pub struct Target {
    pub id: u32,
    pub position: Position,
    /// Хто виявив (None якщо ще не виявлено)
    pub discovered_by: Option<AgentId>,
}

/// Світ — спільне середовище для всіх агентів
#[derive(Debug)]
pub struct World {
    pub width: f64,
    pub height: f64,
    pub base_position: Position,
    pub targets: Vec<Target>,
}

impl World {
    /// Створення нового світу
    pub fn new(width: f64, height: f64) -> Self {
        Self {
            width,
            height,
            base_position: Position::new(width / 2.0, height / 2.0),
            targets: Vec::new(),
        }
    }
    
    /// Додавання цілей у випадкових позиціях
    pub fn spawn_targets(&mut self, count: usize) {
        use rand::Rng;
        let mut rng = rand::thread_rng();
        
        for id in 0..count as u32 {
            self.targets.push(Target {
                id,
                position: Position::new(
                    rng.gen_range(0.0..self.width),
                    rng.gen_range(0.0..self.height),
                ),
                discovered_by: None,
            });
        }
    }
    
    /// Позначити ціль як виявлену
    pub fn mark_target_discovered(&mut self, target_id: u32, agent_id: AgentId) -> bool {
        if let Some(target) = self.targets.iter_mut().find(|t| t.id == target_id) {
            if target.discovered_by.is_none() {
                target.discovered_by = Some(agent_id);
                return true;  // Перше виявлення
            }
        }
        false  // Вже було виявлено або не знайдено
    }
    
    /// Підрахунок виявлених цілей
    pub fn discovered_count(&self) -> usize {
        self.targets.iter().filter(|t| t.discovered_by.is_some()).count()
    }
}

/// Тип для спільного доступу до світу
pub type SharedWorld = Arc<RwLock<World>>;
```

**Чому Arc<RwLock<World>>?**

Світ — єдиний компонент системи, що потребує спільного мутабельного доступу:
- **Arc**: Дозволяє мати кілька "власників" (кожен агент + координатор)
- **RwLock**: Async-сумісний lock. Читання паралельне, запис ексклюзивний

Альтернативи:
- `Mutex<World>`: Менш ефективний (блокує читання)
- Окремий канал до "World actor": Складніший, але уникає shared state

Для нашого випадку `Arc<RwLock<World>>` — оптимальний баланс простоти та ефективності.

### 44.2.3 Повідомлення системи

**Постановка задачі:** Визначимо типи команд (від координатора до агентів) та звітів (від агентів до координатора).

```rust
use tokio::time::Instant;

/// Команди від координатора до агентів
#[derive(Debug, Clone)]
pub enum Command {
    /// Почати патрулювання
    StartPatrol,
    
    /// Рухатись до вказаної позиції
    MoveTo { position: Position },
    
    /// Повернутись на базу
    ReturnToBase,
    
    /// Зупинити всі дії
    Halt,
}

/// Звіти від агентів до координатора
#[derive(Debug, Clone)]
pub enum Report {
    /// Періодичний звіт про стан
    Status {
        agent_id: AgentId,
        position: Position,
        battery: u8,
        mode: AgentMode,
        timestamp: Instant,
    },
    
    /// Виявлено ціль
    TargetDiscovered {
        agent_id: AgentId,
        target_id: u32,
        position: Position,
    },
    
    /// Низький заряд батареї
    LowBattery {
        agent_id: AgentId,
        battery: u8,
    },
    
    /// Агент повернувся на базу
    ReturnedToBase {
        agent_id: AgentId,
    },
}
```

**Що важливо розуміти:**

1. **Command — Clone**: Broadcast канал вимагає Clone, бо кожен підписник отримує копію.

2. **Report містить agent_id**: Координатор має знати, від якого агента прийшов звіт.

3. **Timestamp в Status**: Дозволяє координатору відстежувати "свіжість" даних.

---

## 44.3 АГЕНТ-АКТОР

### 44.3.1 Структура агента

Агент — це актор з трьома каналами: вхідні команди (broadcast), вихідні звіти (mpsc), сигнал shutdown (watch).

**Постановка задачі:** Створимо структуру агента з усіма необхідними полями та каналами.

```rust
use tokio::sync::{broadcast, mpsc, watch};
use tokio::time::{interval, Duration, Instant};

/// Конфігурація агента
pub struct AgentConfig {
    /// Швидкість руху (одиниць за tick)
    pub speed: f64,
    /// Радіус виявлення цілей
    pub scan_radius: f64,
    /// Інтервал tick'ів
    pub tick_interval: Duration,
    /// Поріг низької батареї
    pub low_battery_threshold: u8,
}

impl Default for AgentConfig {
    fn default() -> Self {
        Self {
            speed: 2.0,
            scan_radius: 10.0,
            tick_interval: Duration::from_millis(100),
            low_battery_threshold: 20,
        }
    }
}

/// Агент — автономна одиниця рою
pub struct Agent {
    // Ідентифікація
    id: AgentId,
    
    // Стан
    position: Position,
    battery: u8,
    mode: AgentMode,
    target_position: Option<Position>,
    
    // Канали
    command_rx: broadcast::Receiver<Command>,
    report_tx: mpsc::Sender<Report>,
    shutdown_rx: watch::Receiver<bool>,
    
    // Спільний стан
    world: SharedWorld,
    
    // Конфігурація
    config: AgentConfig,
    
    // Лічильники
    tick_count: u64,
}
```

**Розглянемо кожне поле:**

**Ідентифікація:**
- `id` — унікальний номер агента для логування та звітів

**Стан:**
- `position` — поточне місцезнаходження
- `battery` — заряд батареї (0-100%)
- `mode` — поточний режим роботи
- `target_position` — куди рухається (якщо є ціль)

**Канали:**
- `command_rx` — приймач broadcast для команд
- `report_tx` — відправник mpsc для звітів
- `shutdown_rx` — приймач watch для сигналу завершення

**Спільний стан:**
- `world` — доступ до середовища (цілі, територія)

### 44.3.2 Конструктор агента

**Постановка задачі:** Створимо функцію для ініціалізації агента з усіма залежностями.

```rust
impl Agent {
    /// Створення нового агента
    pub fn new(
        id: AgentId,
        start_position: Position,
        command_rx: broadcast::Receiver<Command>,
        report_tx: mpsc::Sender<Report>,
        shutdown_rx: watch::Receiver<bool>,
        world: SharedWorld,
    ) -> Self {
        Self {
            id,
            position: start_position,
            battery: 100,
            mode: AgentMode::Idle,
            target_position: None,
            command_rx,
            report_tx,
            shutdown_rx,
            world,
            config: AgentConfig::default(),
            tick_count: 0,
        }
    }
}
```

**Що важливо:** Агент не створює канали сам — він отримує їх ззовні. Це дозволяє координатору контролювати топологію комунікації.

### 44.3.3 Головний цикл агента

Серце актора — цикл обробки подій. Використовуємо `tokio::select!` для одночасного очікування на кілька джерел.

**Постановка задачі:** Реалізуємо цикл, що реагує на shutdown, команди та періодичні tick'и.

```rust
impl Agent {
    /// Головний цикл роботи агента
    pub async fn run(mut self) {
        println!("[Agent {}] Запущено на позиції {}", self.id, self.position);
        
        // Таймер для періодичних оновлень
        let mut tick_timer = interval(self.config.tick_interval);
        
        loop {
            tokio::select! {
                // Пріоритет 1: Перевірка shutdown
                // biased гарантує, що ця гілка перевіряється першою
                biased;
                
                _ = self.shutdown_rx.changed() => {
                    if *self.shutdown_rx.borrow() {
                        println!("[Agent {}] Отримано сигнал shutdown", self.id);
                        break;
                    }
                }
                
                // Пріоритет 2: Вхідні команди
                result = self.command_rx.recv() => {
                    match result {
                        Ok(command) => self.handle_command(command).await,
                        Err(broadcast::error::RecvError::Closed) => {
                            println!("[Agent {}] Канал команд закрито", self.id);
                            break;
                        }
                        Err(broadcast::error::RecvError::Lagged(n)) => {
                            // Агент "відстав" — пропустив n повідомлень
                            println!("[Agent {}] Пропущено {} команд", self.id, n);
                        }
                    }
                }
                
                // Пріоритет 3: Періодичний tick
                _ = tick_timer.tick() => {
                    self.tick().await;
                }
            }
        }
        
        // Cleanup перед завершенням
        self.send_report(Report::ReturnedToBase { agent_id: self.id }).await;
        println!("[Agent {}] Завершено", self.id);
    }
}
```

**Розглянемо механіку цього циклу:**

1. **`biased`**: Гарантує, що гілки перевіряються в порядку оголошення. Без цього `select!` обирає випадково, що може затримати shutdown.

2. **Shutdown через watch**: `changed()` спрацьовує при зміні значення. Перевіряємо `*borrow()` чи це true.

3. **Команди через broadcast**: `recv()` може повернути помилку `Lagged` якщо агент не встигає обробляти. Це не критично — просто логуємо.

4. **Tick через interval**: Періодичні оновлення стану, незалежно від команд.

5. **Cleanup**: Перед завершенням надсилаємо фінальний звіт.

### 44.3.4 Обробка команд

**Постановка задачі:** Реалізуємо реакцію агента на різні типи команд.

```rust
impl Agent {
    /// Обробка вхідної команди
    async fn handle_command(&mut self, command: Command) {
        match command {
            Command::StartPatrol => {
                if self.battery > self.config.low_battery_threshold {
                    println!("[Agent {}] Починаю патрулювання", self.id);
                    self.mode = AgentMode::Patrolling;
                    self.pick_patrol_target().await;
                } else {
                    println!("[Agent {}] Недостатньо батареї для патрулювання", self.id);
                }
            }
            
            Command::MoveTo { position } => {
                println!("[Agent {}] Рухаюсь до {}", self.id, position);
                self.target_position = Some(position);
            }
            
            Command::ReturnToBase => {
                println!("[Agent {}] Повертаюсь на базу", self.id);
                self.mode = AgentMode::Returning;
                let base = self.world.read().await.base_position;
                self.target_position = Some(base);
            }
            
            Command::Halt => {
                println!("[Agent {}] Зупинка", self.id);
                self.mode = AgentMode::Idle;
                self.target_position = None;
            }
        }
    }
    
    /// Вибір випадкової точки для патрулювання
    async fn pick_patrol_target(&mut self) {
        use rand::Rng;
        let mut rng = rand::thread_rng();
        
        let world = self.world.read().await;
        self.target_position = Some(Position::new(
            rng.gen_range(0.0..world.width),
            rng.gen_range(0.0..world.height),
        ));
    }
}
```

### 44.3.5 Логіка tick'а

Tick — це один "крок часу" симуляції. Агент оновлює свій стан, рухається, скануює територію.

**Постановка задачі:** Реалізуємо періодичне оновлення стану агента.

```rust
impl Agent {
    /// Один tick симуляції
    async fn tick(&mut self) {
        self.tick_count += 1;
        
        // Споживання батареї
        if self.mode == AgentMode::Patrolling {
            self.battery = self.battery.saturating_sub(1);
        }
        
        // Перевірка батареї
        if self.battery <= self.config.low_battery_threshold 
           && self.mode == AgentMode::Patrolling 
        {
            self.send_report(Report::LowBattery {
                agent_id: self.id,
                battery: self.battery,
            }).await;
            
            // Автоматичне повернення
            self.mode = AgentMode::Returning;
            let base = self.world.read().await.base_position;
            self.target_position = Some(base);
        }
        
        // Рух до цілі
        if let Some(target) = self.target_position {
            self.position.move_towards(&target, self.config.speed);
            
            // Досягли цілі?
            if self.position.distance_to(&target) < 0.1 {
                self.on_target_reached().await;
            }
        }
        
        // Сканування території (тільки при патрулюванні)
        if self.mode == AgentMode::Patrolling {
            self.scan_for_targets().await;
        }
        
        // Періодичний звіт (кожні 10 tick'ів)
        if self.tick_count % 10 == 0 {
            self.send_status_report().await;
        }
    }
    
    /// Дія при досягненні цільової точки
    async fn on_target_reached(&mut self) {
        self.target_position = None;
        
        match self.mode {
            AgentMode::Patrolling => {
                // Обираємо нову точку патрулювання
                self.pick_patrol_target().await;
            }
            AgentMode::Returning => {
                // Повернулись на базу
                self.mode = AgentMode::Idle;
                self.send_report(Report::ReturnedToBase { agent_id: self.id }).await;
                
                // Заряджаємо батарею
                self.battery = 100;
            }
            _ => {}
        }
    }
    
    /// Сканування території на цілі
    async fn scan_for_targets(&mut self) {
        // Читаємо список цілей
        let targets: Vec<(u32, Position)> = {
            let world = self.world.read().await;
            world.targets
                .iter()
                .filter(|t| t.discovered_by.is_none())  // Тільки невиявлені
                .map(|t| (t.id, t.position))
                .collect()
        };
        
        // Перевіряємо відстань до кожної цілі
        for (target_id, target_pos) in targets {
            if self.position.distance_to(&target_pos) <= self.config.scan_radius {
                // Ціль у зоні виявлення!
                let is_first = {
                    let mut world = self.world.write().await;
                    world.mark_target_discovered(target_id, self.id)
                };
                
                if is_first {
                    println!("[Agent {}] 🎯 Виявлено ціль #{} на {}", 
                        self.id, target_id, target_pos);
                    
                    self.send_report(Report::TargetDiscovered {
                        agent_id: self.id,
                        target_id,
                        position: target_pos,
                    }).await;
                }
            }
        }
    }
    
    /// Надсилання звіту координатору
    async fn send_report(&self, report: Report) {
        if let Err(e) = self.report_tx.send(report).await {
            println!("[Agent {}] Помилка надсилання звіту: {}", self.id, e);
        }
    }
    
    /// Надсилання звіту про стан
    async fn send_status_report(&self) {
        self.send_report(Report::Status {
            agent_id: self.id,
            position: self.position,
            battery: self.battery,
            mode: self.mode,
            timestamp: Instant::now(),
        }).await;
    }
}
```

**Ключові аспекти логіки tick'а:**

1. **Споживання батареї**: Тільки при активному патрулюванні.

2. **Автоматичне повернення**: При низькому заряді агент сам повертається на базу.

3. **Двофазний lock на world**: Спочатку читаємо (read), потім, якщо потрібно — пишемо (write). Мінімізуємо час утримання lock'а.

4. **Періодичні звіти**: Кожен 10-й tick, щоб не перевантажувати канал.

---

## 44.4 КООРДИНАТОР-АКТОР

### 44.4.1 Роль координатора

Координатор — "мозок" системи. Він:
- Запускає місію, надсилаючи StartPatrol
- Збирає звіти від агентів
- Приймає рішення (перенаправлення агентів, завершення)
- Ініціює graceful shutdown

### 44.4.2 Структура координатора

**Постановка задачі:** Створимо структуру координатора з каналами та статистикою місії.

```rust
/// Статистика місії
#[derive(Debug, Default)]
pub struct MissionStats {
    pub targets_discovered: usize,
    pub total_targets: usize,
    pub reports_received: u64,
    pub low_battery_events: u64,
}

/// Координатор рою
pub struct Coordinator {
    // Канали
    report_rx: mpsc::Receiver<Report>,
    command_tx: broadcast::Sender<Command>,
    shutdown_tx: watch::Sender<bool>,
    
    // Спільний стан
    world: SharedWorld,
    
    // Статистика
    stats: MissionStats,
    agent_count: usize,
    
    // Стан агентів (останні відомі позиції)
    agent_positions: std::collections::HashMap<AgentId, Position>,
}

impl Coordinator {
    pub fn new(
        agent_count: usize,
        report_rx: mpsc::Receiver<Report>,
        command_tx: broadcast::Sender<Command>,
        shutdown_tx: watch::Sender<bool>,
        world: SharedWorld,
    ) -> Self {
        Self {
            report_rx,
            command_tx,
            shutdown_tx,
            world,
            stats: MissionStats::default(),
            agent_count,
            agent_positions: std::collections::HashMap::new(),
        }
    }
}
```

### 44.4.3 Головний цикл координатора

**Постановка задачі:** Реалізуємо цикл координатора з таймером місії та обробкою звітів.

```rust
use tokio::time::{timeout, Duration};

impl Coordinator {
    /// Запуск координатора на визначений час
    pub async fn run(mut self, mission_duration: Duration) {
        println!("\n========================================");
        println!("    КООРДИНАТОР: Місія розпочата");
        println!("    Тривалість: {:?}", mission_duration);
        println!("    Агентів: {}", self.agent_count);
        println!("========================================\n");
        
        // Ініціалізація статистики
        {
            let world = self.world.read().await;
            self.stats.total_targets = world.targets.len();
        }
        
        // Команда старту
        self.broadcast_command(Command::StartPatrol).await;
        
        // Основний цикл з таймаутом місії
        let mission_result = timeout(
            mission_duration,
            self.mission_loop()
        ).await;
        
        match mission_result {
            Ok(_) => println!("\n[Coordinator] Місія завершена достроково"),
            Err(_) => println!("\n[Coordinator] Час місії вичерпано"),
        }
        
        // Graceful shutdown
        self.initiate_shutdown().await;
    }
    
    /// Основний цикл обробки звітів
    async fn mission_loop(&mut self) {
        loop {
            // Чекаємо звіт з таймаутом
            match timeout(Duration::from_secs(1), self.report_rx.recv()).await {
                Ok(Some(report)) => {
                    self.handle_report(report).await;
                    
                    // Перевірка умови завершення
                    if self.stats.targets_discovered >= self.stats.total_targets {
                        println!("\n🎉 Всі цілі виявлено!");
                        break;
                    }
                }
                Ok(None) => {
                    // Канал закрито — всі агенти завершились
                    println!("[Coordinator] Канал звітів закрито");
                    break;
                }
                Err(_) => {
                    // Таймаут — друкуємо статус
                    self.print_status();
                }
            }
        }
    }
    
    /// Обробка звіту від агента
    async fn handle_report(&mut self, report: Report) {
        self.stats.reports_received += 1;
        
        match report {
            Report::Status { agent_id, position, battery, mode, .. } => {
                self.agent_positions.insert(agent_id, position);
                
                // Можна додати логіку моніторингу
                if battery < 10 && mode != AgentMode::Returning {
                    // Критично низький заряд — примусове повернення
                    // (насправді агент сам повернеться, але можна й так)
                }
            }
            
            Report::TargetDiscovered { agent_id, target_id, position } => {
                self.stats.targets_discovered += 1;
                println!(
                    "📍 Ціль #{} виявлена агентом {} на {}  [{}/{}]",
                    target_id, agent_id, position,
                    self.stats.targets_discovered, self.stats.total_targets
                );
            }
            
            Report::LowBattery { agent_id, battery } => {
                self.stats.low_battery_events += 1;
                println!("🔋 Agent {} — низький заряд: {}%", agent_id, battery);
            }
            
            Report::ReturnedToBase { agent_id } => {
                println!("🏠 Agent {} повернувся на базу", agent_id);
            }
        }
    }
    
    /// Надсилання команди всім агентам
    async fn broadcast_command(&self, command: Command) {
        if let Err(e) = self.command_tx.send(command.clone()) {
            println!("[Coordinator] Помилка broadcast: {}", e);
        }
        println!("[Coordinator] Broadcast: {:?}", command);
    }
    
    /// Друк поточного статусу
    fn print_status(&self) {
        println!(
            "📊 Статус: цілей {}/{}, звітів {}, low_battery {}",
            self.stats.targets_discovered,
            self.stats.total_targets,
            self.stats.reports_received,
            self.stats.low_battery_events
        );
    }
    
    /// Ініціація graceful shutdown
    async fn initiate_shutdown(&mut self) {
        println!("\n[Coordinator] Ініціюю shutdown...");
        
        // Команда повернення
        self.broadcast_command(Command::ReturnToBase).await;
        
        // Даємо час на обробку
        tokio::time::sleep(Duration::from_millis(500)).await;
        
        // Сигнал shutdown
        if let Err(e) = self.shutdown_tx.send(true) {
            println!("[Coordinator] Помилка shutdown: {}", e);
        }
        
        // Чекаємо завершення агентів
        tokio::time::sleep(Duration::from_secs(1)).await;
        
        // Фінальна статистика
        self.print_final_stats();
    }
    
    /// Фінальна статистика місії
    fn print_final_stats(&self) {
        println!("\n========================================");
        println!("         РЕЗУЛЬТАТИ МІСІЇ");
        println!("========================================");
        println!("  Цілей виявлено: {}/{}", 
            self.stats.targets_discovered, self.stats.total_targets);
        println!("  Звітів отримано: {}", self.stats.reports_received);
        println!("  Подій низької батареї: {}", self.stats.low_battery_events);
        println!("========================================\n");
    }
}
```

---

## 44.5 ІНТЕГРАЦІЯ RAYON ДЛЯ CPU-BOUND ЗАДАЧ

### 44.5.1 Коли потрібен Rayon

У реальній системі координатор може мати CPU-bound задачі:
- Розрахунок оптимальних шляхів для всіх агентів
- Аналіз покриття території
- Кластеризація виявлених цілей
- Обробка зображень з камер

Ці задачі не можна виконувати в async контексті напряму — вони заблокують worker threads.

### 44.5.2 Приклад: паралельне планування шляхів

**Постановка задачі:** Координатор має обчислити нові цільові позиції для всіх агентів на основі їх поточних позицій та невиявлених цілей.

```rust
use rayon::prelude::*;
use tokio::task;

/// Результат планування для одного агента
#[derive(Debug, Clone)]
struct PlannedPath {
    agent_id: AgentId,
    target: Position,
    estimated_time: f64,
}

impl Coordinator {
    /// Планування шляхів для всіх агентів
    /// CPU-bound операція — використовуємо spawn_blocking + Rayon
    async fn plan_paths_for_all_agents(&self) -> Vec<PlannedPath> {
        // Збираємо дані для планування
        let agent_positions: Vec<(AgentId, Position)> = 
            self.agent_positions.iter()
                .map(|(&id, &pos)| (id, pos))
                .collect();
        
        let undiscovered_targets: Vec<Position> = {
            let world = self.world.read().await;
            world.targets.iter()
                .filter(|t| t.discovered_by.is_none())
                .map(|t| t.position)
                .collect()
        };
        
        // CPU-bound обчислення в окремому пулі
        let paths = task::spawn_blocking(move || {
            agent_positions.par_iter()
                .map(|(agent_id, position)| {
                    // Знаходимо найближчу невиявлену ціль
                    let nearest = undiscovered_targets.iter()
                        .min_by(|a, b| {
                            let dist_a = position.distance_to(a);
                            let dist_b = position.distance_to(b);
                            dist_a.partial_cmp(&dist_b).unwrap()
                        })
                        .cloned()
                        .unwrap_or(Position::new(50.0, 50.0));
                    
                    // Симуляція "важких" обчислень (A*, RRT, тощо)
                    std::thread::sleep(std::time::Duration::from_micros(100));
                    
                    PlannedPath {
                        agent_id: *agent_id,
                        target: nearest,
                        estimated_time: position.distance_to(&nearest) / 2.0,
                    }
                })
                .collect::<Vec<_>>()
        }).await.expect("Planning failed");
        
        paths
    }
}
```

**Архітектура гібридної обробки:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ГІБРИДНА ОБРОБКА                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Координатор (async)                                                │
│       │                                                             │
│       │ 1. Збір даних (async read)                                 │
│       ▼                                                             │
│  ┌─────────────────────────────────────────┐                       │
│  │ world.read().await                      │                       │
│  │ → agent_positions, undiscovered_targets │                       │
│  └─────────────────────────────────────────┘                       │
│       │                                                             │
│       │ 2. spawn_blocking                                          │
│       ▼                                                             │
│  ┌─────────────────────────────────────────┐                       │
│  │ Blocking Thread Pool                    │                       │
│  │                                         │                       │
│  │   Rayon par_iter()                      │                       │
│  │   ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐     │                       │
│  │   │Path1│ │Path2│ │Path3│ │PathN│     │                       │
│  │   │calc │ │calc │ │calc │ │calc │     │                       │
│  │   └─────┘ └─────┘ └─────┘ └─────┘     │                       │
│  │         (паралельно на всіх ядрах)      │                       │
│  └─────────────────────────────────────────┘                       │
│       │                                                             │
│       │ 3. .await результат                                        │
│       ▼                                                             │
│  ┌─────────────────────────────────────────┐                       │
│  │ Vec<PlannedPath>                        │                       │
│  │ → broadcast MoveTo commands             │                       │
│  └─────────────────────────────────────────┘                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 44.6 ЗАПУСК СИСТЕМИ

### 44.6.1 Функція ініціалізації

**Постановка задачі:** Створимо головну функцію, що ініціалізує всі компоненти та запускає систему.

```rust
use std::sync::Arc;
use tokio::sync::{broadcast, mpsc, watch, RwLock};
use tokio::time::Duration;

/// Запуск рою агентів
pub async fn run_swarm(agent_count: usize, target_count: usize, mission_duration: Duration) {
    println!("\n╔════════════════════════════════════════╗");
    println!("║     ASYNC SWARM v4.0                   ║");
    println!("╠════════════════════════════════════════╣");
    println!("║  Агентів: {:4}                         ║", agent_count);
    println!("║  Цілей:   {:4}                         ║", target_count);
    println!("║  Час:     {:4?}                       ║", mission_duration);
    println!("╚════════════════════════════════════════╝\n");
    
    // 1. Створення світу
    let mut world = World::new(100.0, 100.0);
    world.spawn_targets(target_count);
    let world = Arc::new(RwLock::new(world));
    
    // 2. Створення каналів
    let (command_tx, _) = broadcast::channel::<Command>(64);
    let (report_tx, report_rx) = mpsc::channel::<Report>(agent_count * 10);
    let (shutdown_tx, shutdown_rx) = watch::channel(false);
    
    // 3. Запуск агентів
    let base_position = world.read().await.base_position;
    
    for id in 0..agent_count as u32 {
        let agent = Agent::new(
            id,
            base_position,
            command_tx.subscribe(),  // Кожен агент отримує свій Receiver
            report_tx.clone(),       // Клонуємо Sender
            shutdown_rx.clone(),     // Клонуємо Receiver
            world.clone(),           // Клонуємо Arc
        );
        
        // Запускаємо агента як окрему задачу
        tokio::spawn(agent.run());
    }
    
    // 4. Створення та запуск координатора
    let coordinator = Coordinator::new(
        agent_count,
        report_rx,
        command_tx,
        shutdown_tx,
        world.clone(),
    );
    
    // Координатор працює у головній задачі
    coordinator.run(mission_duration).await;
    
    // 5. Очікування завершення всіх задач
    tokio::time::sleep(Duration::from_secs(2)).await;
    
    println!("╔════════════════════════════════════════╗");
    println!("║     СИСТЕМА ЗАВЕРШЕНА                  ║");
    println!("╚════════════════════════════════════════╝\n");
}
```

**Розглянемо послідовність ініціалізації:**

1. **Створення світу**: `World` з територією та цілями, обгорнутий в `Arc<RwLock<>>`.

2. **Створення каналів**: 
   - `broadcast` для команд (один sender, багато receivers)
   - `mpsc` для звітів (багато senders, один receiver)
   - `watch` для shutdown (один sender, багато receivers)

3. **Запуск агентів**: Кожен агент створюється з клонами каналів і запускається через `tokio::spawn`.

4. **Запуск координатора**: Координатор отримує "оригінали" каналів і запускається в головній задачі.

### 44.6.2 Точка входу

```rust
#[tokio::main]
async fn main() {
    // Запуск рою: 50 агентів, 20 цілей, 30 секунд
    run_swarm(50, 20, Duration::from_secs(30)).await;
}
```

---

## 44.7 МАСШТАБУВАННЯ ТА ОПТИМІЗАЦІЯ

### 44.7.1 Тестування масштабованості

Проведемо теоретичну оцінку масштабованості системи:

| Агентів | Пам'ять | CPU (idle) | Повідомлень/сек | Примітки |
|---------|---------|------------|-----------------|----------|
| 10 | ~5 MB | <1% | ~100 | Базовий тест |
| 100 | ~15 MB | ~5% | ~1000 | Комфортно |
| 1000 | ~100 MB | ~15% | ~10000 | Добре працює |
| 10000 | ~500 MB | ~50% | ~100000 | Потребує оптимізації |

**Ключові спостереження:**

1. **Пам'ять**: Кожна async задача займає ~2-5 KB (на відміну від ~2 MB для потоку).

2. **CPU**: Більшість часу система простоює (idle), чекаючи на таймери.

3. **Bottleneck**: При великій кількості агентів — канал звітів (fan-in).

### 44.7.2 Оптимізації для великих рої

**1. Зменшення частоти звітів:**

```rust
// Замість кожного tick — кожен 50-й
if self.tick_count % 50 == 0 {
    self.send_status_report().await;
}
```

**2. Батчинг звітів у координаторі:**

```rust
// Обробка пачками замість по одному
let mut batch = Vec::with_capacity(100);
while let Ok(report) = self.report_rx.try_recv() {
    batch.push(report);
    if batch.len() >= 100 { break; }
}
// Обробка всієї пачки
for report in batch {
    self.handle_report(report).await;
}
```

**3. Sharding для дуже великих роїв:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SHARDED ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│                    ┌───────────────────┐                           │
│                    │  ГОЛОВНИЙ         │                           │
│                    │  КООРДИНАТОР      │                           │
│                    └─────────┬─────────┘                           │
│                              │                                      │
│         ┌────────────────────┼────────────────────┐                │
│         │                    │                    │                │
│         ▼                    ▼                    ▼                │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐        │
│  │ Sector A    │      │ Sector B    │      │ Sector C    │        │
│  │ Coordinator │      │ Coordinator │      │ Coordinator │        │
│  └──────┬──────┘      └──────┬──────┘      └──────┬──────┘        │
│         │                    │                    │                │
│    ┌────┴────┐          ┌────┴────┐          ┌────┴────┐          │
│    ▼         ▼          ▼         ▼          ▼         ▼          │
│ [Agents] [Agents]    [Agents] [Agents]    [Agents] [Agents]       │
│  1-100    101-200     201-300  301-400     401-500  501-600       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 44.8 ПОРІВНЯННЯ V3.0 ТА V4.0

### 44.8.1 Архітектурні зміни

| Аспект | v3.0 (Потоки) | v4.0 (Async) |
|--------|---------------|--------------|
| **Примітив конкурентності** | `std::thread` | `tokio::spawn` |
| **Синхронізація стану** | `Arc<Mutex<T>>` | `Arc<RwLock<T>>` |
| **Комунікація** | `std::sync::mpsc` | `tokio::sync::*` |
| **Shutdown** | Атомарний прапорець | `watch::channel` |
| **Tick** | `thread::sleep` | `tokio::time::interval` |
| **CPU-bound** | Ті ж потоки | `spawn_blocking` + Rayon |

### 44.8.2 Що залишилось незмінним

Попри архітектурні зміни, концептуальна модель агента та рою залишилась:
- Агент = автономна одиниця зі станом та поведінкою
- Координатор = центральний керуючий елемент
- Комунікація = через повідомлення
- Graceful shutdown = впорядковане завершення

Це демонструє, що **Actor Model** — абстракція, незалежна від реалізації (потоки чи async).

---

## 44.9 ЛАБОРАТОРНА РОБОТА

### Мета роботи

Побудувати власну версію async рою з нуля, застосовуючи всі вивчені концепції.

### Завдання 1: Базовий рій (4 бали)

Реалізуйте мінімальний рій:
- 10 агентів
- Координатор з командою StartPatrol
- Звіт StatusUpdate кожні 10 tick'ів
- Graceful shutdown через 10 секунд

### Завдання 2: Виявлення цілей (3 бали)

Розширте базовий рій:
- 10 цілей у випадкових позиціях
- Логіка виявлення (радіус 10 одиниць)
- Звіт TargetDiscovered
- Статистика знайдених цілей

### Завдання 3: Паралельні обчислення (3 бали)

Додайте CPU-bound задачу:
- Функція "планування шляху" (симуляція через `thread::sleep(100μs)`)
- Координатор викликає її для всіх агентів
- Використайте `spawn_blocking` + `par_iter`
- Виміряйте прискорення порівняно з послідовним виконанням

---

## 44.10 РЕЗЮМЕ

### Архітектурні рішення v4.0

**Actor Model**: Кожен агент — ізольований актор з власним станом та циклом обробки.

**Канали за призначенням:**
- `broadcast` — команди (один до багатьох)
- `mpsc` — звіти (багато до одного)
- `watch` — shutdown (стан для всіх)

**Гібридний паралелізм:**
- Tokio — для I/O-bound (канали, таймери)
- Rayon — для CPU-bound (обчислення шляхів)

**Graceful Shutdown:**
- watch канал для сигналу
- Поетапне завершення (команда → сигнал → очікування)

### Ключові навички Частини IV

✅ Async/await та Future'и
✅ Tokio runtime та примітиви
✅ Різні типи каналів
✅ Streams для потокових даних
✅ Actor Model на Tokio
✅ Rayon для паралелізму даних
✅ Інтеграція async та CPU-bound

---

## 🔗 ЗВ'ЯЗОК З НАСТУПНОЮ ЧАСТИНОЮ

**Частина IV завершена!** 🎉

Ви побудували масштабовану async мультиагентну систему. Але попереду ще багато цікавого.

У **Частині V: Архітектури МАС та екосистема** ви:
- Опануєте **макроси** для створення DSL
- Дізнаєтесь про **unsafe Rust** та його межі
- Вивчите **ECS архітектуру** для великих симуляцій
- Реалізуєте **BDI-агентів** (Beliefs-Desires-Intentions)
- Додасте **мережеву комунікацію** для розподілених систем
- Побудуєте **WebAssembly візуалізацію** у браузері
- Завершите **фінальний проєкт** — повноцінний автономний рій БПЛА!

---

> **Наступна частина:** [Частина V: Архітектури МАС та екосистема](./Part_V_Overview.md)
