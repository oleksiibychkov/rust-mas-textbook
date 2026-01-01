# Розділ 21: Логування та tracing

## Вступ

Уявіть: ваш агент БПЛА виконує місію. Раптом він зникає з радарів. Що сталось? Де він був у той момент? Яка була батарея? Чи працював GPS? Без записів про роботу системи — ви ніколи не дізнаєтесь.

**Printf-дебаг** — перше, що спадає на думку новачкам:

```rust
println!("Starting mission");
println!("Battery: {}", agent.battery);
println!("Position: {:?}", agent.position);
// ... 500 рядків пізніше
println!("Something happened");  // Що? Де? Коли?
```

Проблеми цього підходу:
- **Немає рівнів** — всі повідомлення однакові, не відфільтруєш важливе від шуму
- **Немає структури** — парсити рядки для аналізу — кошмар
- **Немає контексту** — "Something happened" нічого не каже
- **Немає часу** — коли це сталось?
- **Завжди увімкнено** — в production весь цей шум заважає

**Логування** — систематичний запис подій. Кожен запис має:
- **Рівень** (trace, debug, info, warn, error)
- **Час**
- **Контекст** (де в коді, в якій операції)
- **Структуровані дані** (поля-ключі, не просто текст)

**tracing** — сучасний крейт для Rust, що йде далі за просте логування. Він надає **spans** — проміжки часу з контекстом. Замість "сталась помилка" ви бачите "помилка в місії Alpha, під час waypoint 3, при читанні GPS, через timeout".

---

## 21.1 П'ять рівнів логування

### Філософія рівнів

Кожен рівень має своє призначення. Вибір правильного рівня — це мистецтво:

| Рівень | Призначення | Приклади для агента БПЛА |
|--------|------------|--------------------------|
| **TRACE** | Найдетальніший, кожна дрібниця | Координати кожні 100мс, raw дані сенсорів |
| **DEBUG** | Для розробників під час дебагу | Обчислений шлях, зміна стану FSM |
| **INFO** | Нормальні події роботи | Старт місії, досягнення waypoint, посадка |
| **WARN** | Потенційні проблеми | Батарея < 30%, retry з'єднання, fallback |
| **ERROR** | Збої, щось пішло не так | Втрата GPS, помилка мотора, неможливість продовжити |

### Практичне правило

- **Production:** зазвичай INFO і вище (warn, error)
- **Development:** DEBUG і вище
- **Глибокий дебаг:** TRACE і вище

Це контролюється через змінну середовища `RUST_LOG`:

```bash
# Production — тільки важливе
RUST_LOG=info cargo run

# Development — більше деталей
RUST_LOG=debug cargo run

# Глибокий дебаг конкретного модуля
RUST_LOG=my_agent::navigation=trace cargo run
```

---

## 21.2 Налаштування tracing

### Залежності

```toml
# Cargo.toml
[dependencies]
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

**tracing** — макроси та API для логування.
**tracing-subscriber** — компонент, що обробляє логи (виводить в консоль, файл, тощо).

### Базова ініціалізація

Найпростіший варіант — один рядок для початку роботи. Ця програма демонструє мінімальне налаштування tracing.

```rust
use tracing::{info, debug, warn, error};

fn main() {
    // Найпростіша ініціалізація — все з defaults
    tracing_subscriber::fmt::init();
    
    // Тепер можна логувати
    info!("Застосунок запущено");
    debug!("Це debug повідомлення");
    warn!("Це попередження");
    error!("Це помилка");
}
```

**Як це працює:**

- `tracing_subscriber::fmt::init()` створює "підписника" (subscriber), що виводить логи в stdout
- За замовчуванням показує INFO і вище
- DEBUG і TRACE не покажуться без зміни рівня

### Налаштування через builder

Для production потрібен більший контроль. Ця програма демонструє налаштування формату виводу, рівня, та додаткової інформації.

```rust
use tracing::Level;
use tracing_subscriber::FmtSubscriber;

fn main() {
    let subscriber = FmtSubscriber::builder()
        // Мінімальний рівень логів
        .with_max_level(Level::DEBUG)
        // Формат часу — час від старту програми
        .with_timer(tracing_subscriber::fmt::time::uptime())
        // Показувати рівень (INFO, DEBUG, тощо)
        .with_level(true)
        // Показувати target (шлях модуля)
        .with_target(true)
        // Показувати номер рядка
        .with_line_number(true)
        // Показувати назву файлу
        .with_file(true)
        // Компактний формат (один рядок)
        .compact()
        .finish();
    
    // Встановлюємо як глобальний subscriber
    tracing::subscriber::set_global_default(subscriber)
        .expect("Failed to set subscriber");
    
    tracing::info!("Subscriber налаштовано");
    tracing::debug!("Debug тепер видно");
}
```

**Опції builder:**

- `with_max_level()` — максимальний рівень, що буде оброблятись
- `with_timer()` — формат часу (uptime, local, utc)
- `with_target()` — показувати модуль (корисно для фільтрації)
- `compact()` — однорядковий формат (альтернатива: `pretty()`)

### Фільтрація через EnvFilter

Найгнучкіший спосіб — читати налаштування з `RUST_LOG`. Ця програма демонструє налаштування фільтрації з підтримкою змінної середовища.

```rust
use tracing::{info, debug};
use tracing_subscriber::{fmt, prelude::*, EnvFilter};

fn main() {
    // Читаємо RUST_LOG або використовуємо default
    tracing_subscriber::registry()
        .with(fmt::layer())
        .with(EnvFilter::from_default_env()
            .add_directive("info".parse().unwrap())  // Default: info
            .add_directive("my_agent=debug".parse().unwrap())  // my_agent: debug
        )
        .init();
    
    info!("Це видно (info)");
    debug!("Це видно тільки якщо RUST_LOG=debug або вище");
}
```

**Приклади RUST_LOG:**

```bash
# Все на рівні debug
RUST_LOG=debug cargo run

# info для всього, debug для my_agent
RUST_LOG=info,my_agent=debug cargo run

# warn для всього, trace для конкретного модуля
RUST_LOG=warn,my_agent::navigation=trace cargo run
```

---

## 21.3 Макроси логування

### Базовий синтаксис

Макроси `trace!`, `debug!`, `info!`, `warn!`, `error!` мають однаковий синтаксис. Ця програма демонструє різні способи їх використання.

```rust
use tracing::{info, warn, debug};

fn main() {
    tracing_subscriber::fmt::init();
    
    // Просте повідомлення
    info!("Місія розпочата");
    
    // З форматуванням (як println!)
    let mission_id = "ALPHA-001";
    info!("Розпочинаємо місію {}", mission_id);
    
    // Структуроване логування — з іменованими полями
    info!(mission_id = "ALPHA-001", "Розпочинаємо місію");
    
    // Кілька полів
    info!(
        mission_id = "ALPHA-001",
        pilot = "AUTO",
        waypoints = 5,
        "Параметри місії"
    );
    
    // Тільки поля, без повідомлення
    debug!(battery = 85, position_x = 100.0, position_y = 50.0);
    
    // Умовний рівень
    let battery = 25;
    if battery < 30 {
        warn!(battery, threshold = 30, "Низький заряд батареї");
    }
}
```

**Структуроване vs форматоване:**

```rust
// ❌ Форматоване — важко парсити
info!("Battery is {}% at position ({}, {})", 85, 100.0, 50.0);

// ✅ Структуроване — легко фільтрувати та аналізувати
info!(battery = 85, pos_x = 100.0, pos_y = 50.0, "Status");
```

### Типи полів: %, ?, без префікса

Як виводити значення полів? Залежить від типу:

```rust
use tracing::info;

#[derive(Debug)]
struct Position {
    x: f64,
    y: f64,
    z: f64,
}

impl std::fmt::Display for Position {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "({:.1}, {:.1}, {:.1})", self.x, self.y, self.z)
    }
}

fn main() {
    tracing_subscriber::fmt::init();
    
    let pos = Position { x: 100.0, y: 50.0, z: 30.0 };
    let name = "SCOUT-001";
    let battery: u8 = 85;
    
    // % — використовує Display trait
    info!(agent = %name, "Агент ідентифіковано");
    info!(position = %pos, "Позиція (Display)");
    
    // ? — використовує Debug trait
    info!(position = ?pos, "Позиція (Debug)");
    
    // Без префікса — для примітивів (числа, bool)
    info!(battery = battery, "Рівень батареї");
    
    // Комбінація
    info!(
        agent = %name,
        position = ?pos,
        battery = battery,
        "Статус агента"
    );
}
```

**Правило вибору:**

- `%field` — коли тип реалізує Display і ви хочете гарний вивід
- `?field` — коли потрібен Debug (складні структури)
- `field` — для примітивів (i32, f64, bool, &str)

---

## 21.4 Spans — контекст виконання

### Що таке span

**Event** (подія) — одноразовий запис: "сталось X".

**Span** — проміжок часу: "виконується операція Y". Всі події в межах span отримують його контекст.

Уявіть місію з трьома waypoints. Без spans:

```text
INFO: Moving to waypoint
INFO: Scanning area
INFO: Moving to waypoint
INFO: Scanning area
```

Який waypoint? Неясно. З spans:

```rust
INFO mission{id=ALPHA-001}:waypoint{n=1}: Moving
INFO mission{id=ALPHA-001}:waypoint{n=1}:scan: Scanning
INFO mission{id=ALPHA-001}:waypoint{n=2}: Moving
INFO mission{id=ALPHA-001}:waypoint{n=2}:scan: Scanning
```

Контекст зрозумілий.

### Створення та вхід у span

Ця програма демонструє створення spans та вкладену структуру контексту.

```rust
use tracing::{span, info, Level};

fn main() {
    tracing_subscriber::fmt::init();
    
    // Створюємо span з іменем та полями
    let mission_span = span!(Level::INFO, "mission", id = "ALPHA-001");
    
    // Входимо в span — guard живе до кінця scope
    let _enter = mission_span.enter();
    
    // Всі логи тепер мають контекст mission{id=ALPHA-001}
    info!("Preflight check");
    info!("Takeoff");
    
    // Вкладений span
    {
        let waypoint_span = span!(Level::INFO, "waypoint", number = 1);
        let _wp = waypoint_span.enter();
        
        info!("Approaching");  // mission{...}:waypoint{number=1}
        info!("Reached");
    }  // waypoint_span закривається
    
    info!("Returning to base");  // Знову тільки mission{...}
}
```

**Як це працює:**

1. `span!()` створює span, але не входить у нього
2. `.enter()` повертає guard, що тримає span "відкритим"
3. Коли guard виходить зі scope (drop), span закривається
4. Вкладені spans формують дерево контексту

### Альтернативні способи входу

```rust
use tracing::{span, info, Level};

fn main() {
    tracing_subscriber::fmt::init();
    
    // Спосіб 1: enter() — окремі кроки
    let span = span!(Level::INFO, "operation");
    let _guard = span.enter();
    info!("Inside span");
    
    // Спосіб 2: entered() — створення + вхід одразу
    let _span = span!(Level::INFO, "another").entered();
    info!("Inside another span");
    
    // Спосіб 3: in_scope() — для closure
    span!(Level::INFO, "scoped").in_scope(|| {
        info!("Inside via closure");
    });
}
```

---

## 21.5 #[instrument] — автоматичне трасування

### Що робить #[instrument]

Атрибут `#[instrument]` автоматично:
1. Створює span з назвою функції
2. Додає параметри як поля span
3. Входить у span на початку функції
4. Виходить при завершенні

Ця програма демонструє, що #[instrument] генерує "під капотом".

```rust
use tracing::{info, instrument};

// З атрибутом — 1 рядок
#[instrument]
fn calculate_path(from_x: f64, from_y: f64, to_x: f64, to_y: f64) -> f64 {
    info!("Calculating distance");
    ((to_x - from_x).powi(2) + (to_y - from_y).powi(2)).sqrt()
}

// Еквівалентний код без атрибута — 8+ рядків
fn calculate_path_manual(from_x: f64, from_y: f64, to_x: f64, to_y: f64) -> f64 {
    let span = tracing::span!(
        tracing::Level::INFO,
        "calculate_path_manual",
        from_x = from_x,
        from_y = from_y,
        to_x = to_x,
        to_y = to_y
    );
    let _enter = span.enter();
    
    tracing::info!("Calculating distance");
    ((to_x - from_x).powi(2) + (to_y - from_y).powi(2)).sqrt()
}

fn main() {
    tracing_subscriber::fmt::init();
    
    let dist = calculate_path(0.0, 0.0, 3.0, 4.0);
    println!("Distance: {}", dist);
}
```

### Налаштування #[instrument]

Для складніших випадків — опції атрибута:

```rust
use tracing::{info, warn, instrument, Level};

#[derive(Debug)]
struct Agent {
    id: String,
    battery: u8,
    position: (f64, f64),
}

impl Agent {
    // skip(self) — не включати self у поля
    // fields(...) — власні поля замість параметрів
    // level = "..." — рівень span
    #[instrument(
        skip(self),
        fields(agent_id = %self.id, battery = self.battery),
        level = "info"
    )]
    fn execute_command(&mut self, command: &str) {
        info!(command = command, "Executing");
        self.battery = self.battery.saturating_sub(5);
    }
    
    // name = "..." — перейменувати span
    #[instrument(name = "agent_status_check", skip(self))]
    fn check_status(&self) -> String {
        format!("Agent {} at {}%", self.id, self.battery)
    }
    
    // ret — логувати повернене значення
    #[instrument(skip(self), ret)]
    fn get_battery(&self) -> u8 {
        self.battery
    }
    
    // err — логувати помилку якщо Result::Err
    #[instrument(skip(self), err)]
    fn risky_operation(&self) -> Result<(), String> {
        if self.battery < 10 {
            Err("Battery too low".to_string())
        } else {
            Ok(())
        }
    }
}

fn main() {
    tracing_subscriber::fmt()
        .with_max_level(Level::DEBUG)
        .init();
    
    let mut agent = Agent {
        id: "SCOUT-001".to_string(),
        battery: 50,
        position: (0.0, 0.0),
    };
    
    agent.execute_command("TAKEOFF");
    agent.execute_command("SCAN");
    
    let battery = agent.get_battery();
    println!("Battery: {}", battery);
    
    let _ = agent.risky_operation();
}
```

**Опції #[instrument]:**

| Опція | Призначення |
|-------|------------|
| `skip(param)` | Не включати параметр у поля span |
| `skip_all` | Не включати жодних параметрів |
| `fields(key = value)` | Додати власні поля |
| `level = "..."` | Рівень span (trace/debug/info/warn/error) |
| `name = "..."` | Власна назва span |
| `ret` | Логувати повернене значення |
| `err` | Логувати помилку (для Result) |

---

## 21.6 Структуроване логування

### Чому структура важлива

Порівняйте два підходи до аналізу логів:

```rust
// ❌ Неструктуровано — парсити регулярками?
"Agent SCOUT-001 battery is 45% at position (100.0, 50.0, 30.0)"

// ✅ Структуровано — легко фільтрувати
{"agent_id":"SCOUT-001","battery":45,"position":{"x":100.0,"y":50.0,"z":30.0}}
```

Структуровані логи дозволяють:
- Фільтрувати: "покажи всі записи де battery < 30"
- Агрегувати: "середнє battery по агентах"
- Візуалізувати: графіки метрик у реальному часі
- Алертувати: "сповісти якщо error_count > 10/хв"

### JSON формат

Для production часто потрібен JSON:

```rust
use tracing::info;

fn main() {
    tracing_subscriber::fmt()
        .json()  // JSON формат
        .init();
    
    info!(
        agent_id = "SCOUT-001",
        battery = 85,
        position_x = 100.0,
        position_y = 50.0,
        "Status update"
    );
}
```

**Вивід:**
```json
{"timestamp":"2024-01-15T10:30:00.123Z","level":"INFO","fields":{"agent_id":"SCOUT-001","battery":85,"position_x":100.0,"position_y":50.0},"message":"Status update"}
```

---

## 21.7 Фільтрація та кілька outputs

### Розширена фільтрація

Ця програма демонструє складну фільтрацію: різні рівні для різних модулів.

```rust
use tracing::{info, debug, trace};
use tracing_subscriber::{fmt, prelude::*, EnvFilter, filter::LevelFilter};

fn main() {
    let filter = EnvFilter::builder()
        // Default для всього
        .with_default_directive(LevelFilter::INFO.into())
        // Debug для нашого застосунку
        .parse_lossy("my_app=debug")
        // Trace для конкретного модуля
        .add_directive("my_app::navigation=trace".parse().unwrap())
        // Warn для шумної бібліотеки
        .add_directive("noisy_lib=warn".parse().unwrap());
    
    tracing_subscriber::registry()
        .with(fmt::layer())
        .with(filter)
        .init();
    
    info!("Застосунок запущено");
}
```

### Кілька outputs (layers)

Ця програма демонструє виведення логів одночасно в консоль та файл.

```rust
use tracing::{info, error};
use tracing_subscriber::{fmt, prelude::*, EnvFilter};
use std::fs::File;

fn main() {
    // Layer для stdout — INFO і вище
    let stdout_layer = fmt::layer()
        .with_writer(std::io::stdout)
        .with_filter(EnvFilter::new("info"));
    
    // Layer для файлу — всі рівні, без кольорів
    let file = File::create("agent.log").expect("Failed to create log file");
    let file_layer = fmt::layer()
        .with_writer(file)
        .with_ansi(false)  // Без ANSI кольорів у файлі
        .with_filter(EnvFilter::new("debug"));
    
    tracing_subscriber::registry()
        .with(stdout_layer)
        .with(file_layer)
        .init();
    
    info!("Це піде в консоль і файл");
    tracing::debug!("Це тільки у файл");
    error!("Помилка — скрізь");
}
```

**Як це працює:**

- `registry()` створює "реєстр", до якого додаються layers
- Кожен layer може мати власний writer (stdout, stderr, file)
- Кожен layer може мати власний filter

---

## 21.8 Продуктивність логування

### Ліниве обчислення

Макроси tracing **ліниві** — якщо рівень вимкнено, вираз не обчислюється:

```rust
use tracing::{trace, debug, enabled, Level};

fn compute_expensive_stats() -> Vec<i32> {
    // Уявіть тут важкі обчислення
    println!("Computing...");  // Це не виконається якщо trace вимкнено
    (0..10000).collect()
}

fn main() {
    tracing_subscriber::fmt()
        .with_max_level(Level::INFO)  // trace/debug вимкнені
        .init();
    
    // ✅ Функція НЕ викличеться, бо trace вимкнено
    trace!(stats = ?compute_expensive_stats(), "Detailed stats");
    
    // Але якщо потрібна гарантія:
    if enabled!(Level::TRACE) {
        let stats = compute_expensive_stats();
        trace!(stats = ?stats, "Detailed stats");
    }
}
```

### % vs ? — продуктивність

```rust
use tracing::debug;

fn main() {
    tracing_subscriber::fmt::init();
    
    let value = 42;
    
    // % — Display, зазвичай швидше
    debug!(value = %value, "Using Display");
    
    // ? — Debug, може бути повільніше для складних типів
    debug!(value = ?value, "Using Debug");
    
    // Без префікса — найшвидше для примітивів
    debug!(value = value, "Direct");
}
```

---

## 21.9 Практичний приклад: логований агент

Ця програма демонструє повне логування агента БПЛА: spans для місій та waypoints, структуровані поля, різні рівні для різних ситуацій.

```rust
use tracing::{info, warn, error, debug, trace, instrument, span, Level};
use tracing_subscriber::{fmt, prelude::*, EnvFilter};

// ═══════════════════════════════════════════════════════════════════════════
// ТИПИ
// ═══════════════════════════════════════════════════════════════════════════

#[derive(Debug, Clone)]
pub struct Position {
    pub x: f64,
    pub y: f64,
    pub z: f64,
}

impl std::fmt::Display for Position {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "({:.1}, {:.1}, {:.1})", self.x, self.y, self.z)
    }
}

#[derive(Debug)]
pub enum AgentState {
    Grounded,
    Flying,
    Emergency(String),
}

// ═══════════════════════════════════════════════════════════════════════════
// ЛОГОВАНИЙ АГЕНТ
// ═══════════════════════════════════════════════════════════════════════════

pub struct LoggedAgent {
    id: String,
    position: Position,
    battery: u8,
    state: AgentState,
}

impl LoggedAgent {
    pub fn new(id: &str) -> Self {
        // Логуємо створення агента
        info!(agent_id = %id, "Створено нового агента");
        
        Self {
            id: id.to_string(),
            position: Position { x: 0.0, y: 0.0, z: 0.0 },
            battery: 100,
            state: AgentState::Grounded,
        }
    }
    
    /// Зліт — демонструє #[instrument] з err
    #[instrument(skip(self), fields(agent = %self.id), err)]
    pub fn take_off(&mut self, altitude: f64) -> Result<(), String> {
        info!(target_altitude = altitude, "Ініціюємо зліт");
        
        // Перевірка батареї
        if self.battery < 20 {
            warn!(battery = self.battery, threshold = 20, "Низький заряд для зльоту");
            return Err("Недостатньо заряду батареї".to_string());
        }
        
        // Перевірка стану
        match &self.state {
            AgentState::Grounded => {
                debug!("Перехід стану: Grounded -> Flying");
                self.position.z = altitude;
                self.state = AgentState::Flying;
                self.battery = self.battery.saturating_sub(10);
                
                info!(
                    altitude = self.position.z,
                    battery = self.battery,
                    "Зліт завершено"
                );
                Ok(())
            }
            AgentState::Flying => {
                warn!("Вже в повітрі");
                Err("Агент вже летить".to_string())
            }
            AgentState::Emergency(reason) => {
                error!(reason = %reason, "Неможливо злетіти в аварійному стані");
                Err(format!("Аварійний стан: {}", reason))
            }
        }
    }
    
    /// Рух до точки — демонструє вкладені spans
    #[instrument(skip(self, target), fields(agent = %self.id, target = %target))]
    pub fn move_to(&mut self, target: Position) {
        let distance = ((target.x - self.position.x).powi(2)
                      + (target.y - self.position.y).powi(2)).sqrt();
        
        trace!(
            from = %self.position,
            distance = distance,
            "Обчислюємо шлях"
        );
        
        // Симуляція руху
        self.position = target;
        let battery_cost = (distance / 20.0) as u8;
        self.battery = self.battery.saturating_sub(battery_cost);
        
        // Попередження про низьку батарею
        if self.battery < 30 {
            warn!(
                battery = self.battery,
                threshold = 30,
                "Батарея нижче 30%, розгляньте повернення"
            );
        }
        
        debug!(
            position = %self.position,
            battery = self.battery,
            "Досягнуто точки"
        );
    }
    
    /// Виконання місії — демонструє span для місії та вкладені spans для waypoints
    #[instrument(skip(self, waypoints), fields(agent = %self.id))]
    pub fn execute_mission(&mut self, mission_name: &str, waypoints: Vec<Position>) -> Result<(), String> {
        // Створюємо span для всієї місії
        let mission_span = span!(
            Level::INFO,
            "mission",
            name = %mission_name,
            waypoints = waypoints.len()
        );
        let _mission = mission_span.enter();
        
        info!("Початок місії");
        
        // Зліт
        self.take_off(50.0)?;
        
        // Проходимо waypoints
        for (i, waypoint) in waypoints.iter().enumerate() {
            // Span для кожного waypoint
            let wp_span = span!(Level::INFO, "waypoint", number = i + 1);
            let _wp = wp_span.enter();
            
            info!(target = %waypoint, "Рухаємось до waypoint");
            self.move_to(waypoint.clone());
            
            // Перевірка критичного стану
            if self.battery < 15 {
                error!(
                    battery = self.battery,
                    waypoint = i + 1,
                    "Критичний рівень батареї! Аварійне повернення."
                );
                return Err("Батарея критична".to_string());
            }
            
            info!("Waypoint досягнуто");
        }
        
        // Повернення
        info!("Місію завершено, повертаємось на базу");
        self.move_to(Position { x: 0.0, y: 0.0, z: self.position.z });
        self.position.z = 0.0;
        self.state = AgentState::Grounded;
        
        info!(final_battery = self.battery, "Посадка завершена");
        Ok(())
    }
}

// ═══════════════════════════════════════════════════════════════════════════
// MAIN
// ═══════════════════════════════════════════════════════════════════════════

fn main() {
    // Налаштування логування
    let filter = EnvFilter::builder()
        .with_default_directive(Level::INFO.into())
        .parse_lossy("logged_agent=debug");
    
    tracing_subscriber::registry()
        .with(fmt::layer()
            .with_target(true)
            .with_level(true))
        .with(filter)
        .init();
    
    info!("╔══════════════════════════════════════════════════════════════╗");
    info!("║         ДЕМО СИСТЕМИ ЛОГУВАННЯ АГЕНТА                       ║");
    info!("╚══════════════════════════════════════════════════════════════╝");
    
    let mut agent = LoggedAgent::new("SCOUT-001");
    
    let waypoints = vec![
        Position { x: 100.0, y: 0.0, z: 50.0 },
        Position { x: 100.0, y: 100.0, z: 50.0 },
        Position { x: 0.0, y: 100.0, z: 50.0 },
    ];
    
    match agent.execute_mission("Patrol-Alpha", waypoints) {
        Ok(()) => info!("✅ Місію виконано успішно"),
        Err(e) => error!(error = %e, "❌ Місію не завершено"),
    }
}
```

**Що демонструє цей приклад:**

1. **#[instrument]** на методах з `skip(self)` та `fields(agent = ...)`
2. **Вкладені spans:** mission → waypoint → action
3. **Рівні:** trace для деталей, debug для проміжних станів, info для подій, warn для попереджень, error для збоїв
4. **Структуровані поля:** battery, position, target
5. **err на Result** — автоматичне логування помилок

---

## 21.10 Лабораторна робота

**Завдання:** Додати повне логування до агента БПЛА.

### Частина 1: Базове логування (3 бали)

- Додайте логування до всіх методів агента
- Використовуйте правильні рівні (info, warn, error)
- Структуровані поля для agent_id, battery, position

### Частина 2: Spans та #[instrument] (4 бали)

- Додайте `#[instrument]` до публічних методів
- Створіть вкладені spans: mission → waypoint → action
- Використовуйте `skip(self)`, `fields()`, `err`

### Частина 3: Фільтрація (3 бали)

- Налаштуйте EnvFilter
- Development: debug і вище
- Production: info і вище
- Продемонструйте через RUST_LOG

**Критерії оцінювання:**

| Критерій | Бали |
|----------|------|
| Базове логування з рівнями | 3 |
| Spans та #[instrument] | 4 |
| Фільтрація | 3 |
| **Максимум** | **10** |

---

## Підсумок

Професійне логування з tracing:

**Рівні:**
- TRACE — кожна дрібниця
- DEBUG — для розробників
- INFO — нормальна робота
- WARN — потенційні проблеми
- ERROR — збої

**Ключові концепції:**
- **Events** — одноразові записи (`info!`, `warn!`, тощо)
- **Spans** — проміжки часу з контекстом
- **Subscriber** — обробляє та виводить логи
- **Filter** — контролює що показувати

**Практичні поради:**
- Використовуйте структуровані поля замість форматування
- #[instrument] для автоматичного трасування функцій
- EnvFilter для гнучкої фільтрації через RUST_LOG
- % для Display, ? для Debug

---

## Зв'язок з наступним розділом

Ви опанували колекції, обробку помилок та логування. Тепер час для потужної абстракції — **traits**.

У **Розділі 22: Traits — базові концепції** ви дізнаєтесь:
- Що таке traits та навіщо вони потрібні
- Стандартні traits: Display, Debug, Clone, Default
- Як derive автоматизує реалізацію
- Як створити типобезпечні абстракції для агента
