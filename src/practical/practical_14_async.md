# Практичне заняття 14: Async/Await та Tokio

## Мета заняття

Після цього заняття ви зможете:
- Розуміти різницю між синхронним та асинхронним кодом
- Використовувати `async`/`await` для написання неблокуючого коду
- Працювати з Tokio runtime
- Створювати concurrent завдання з `tokio::spawn`
- Використовувати асинхронні канали та примітиви синхронізації

---

## Теоретичний вступ

### Синхронний vs Асинхронний код

**Синхронний код** — операції виконуються послідовно, блокуючи потік:

```rust
fn sync_fetch_data() -> String {
    // Потік заблокований, поки чекаємо відповідь
    let response = blocking_http_get("https://api.example.com/data");
    response
}
```

**Асинхронний код** — операції можуть "призупинятись", звільняючи потік:

```rust
async fn async_fetch_data() -> String {
    // Потік НЕ заблокований — може робити інше
    let response = http_get("https://api.example.com/data").await;
    response
}
```

### Коли використовувати async?

| Використовуйте async | Використовуйте threads |
|---------------------|------------------------|
| I/O-bound операції | CPU-bound обчислення |
| Багато одночасних з'єднань | Паралельні обчислення |
| Мережеві сервери | Обробка великих даних |
| Низький overhead | Простіший код |

### Як працює async/await

```rust
// async fn повертає Future, не виконується одразу
async fn hello() {
    println!("Hello!");
}

// .await "чекає" результат, може призупинити виконання
async fn greet(name: &str) {
    hello().await;  // Виконується hello()
    println!("Greetings, {}!", name);
}
```

**Future** — це значення, яке *може* бути готове в майбутньому. Викликана async функція створює Future, але не виконує код до `.await`.

---

## Tokio — Async Runtime

### Що таке Runtime?

Rust не має вбудованого async runtime. **Tokio** — найпопулярніший runtime, що:
- Планує виконання futures
- Керує пулом потоків
- Надає async I/O, таймери, канали

### Налаштування проєкту

```toml
# Cargo.toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

### Базовий приклад

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    println!("Starting...");
    
    sleep(Duration::from_secs(1)).await;
    
    println!("Done after 1 second!");
}
```

### Що робить #[tokio::main]?

```rust
#[tokio::main]
async fn main() {
    // async код
}

// Розгортається в:
fn main() {
    tokio::runtime::Runtime::new()
        .unwrap()
        .block_on(async {
            // async код
        })
}
```

---

## Concurrent виконання

### tokio::spawn — незалежні задачі

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    // Запускаємо задачу у фоні
    let handle = tokio::spawn(async {
        sleep(Duration::from_millis(500)).await;
        println!("Background task done!");
        42
    });
    
    println!("Main continues...");
    
    // Чекаємо результат
    let result = handle.await.unwrap();
    println!("Result: {}", result);
}
```

### tokio::join! — паралельне очікування

```rust
use tokio::time::{sleep, Duration};

async fn fetch_user() -> String {
    sleep(Duration::from_millis(200)).await;
    "User data".to_string()
}

async fn fetch_posts() -> Vec<String> {
    sleep(Duration::from_millis(300)).await;
    vec!["Post 1".to_string(), "Post 2".to_string()]
}

#[tokio::main]
async fn main() {
    // Виконуються ПАРАЛЕЛЬНО, загальний час ~300ms
    let (user, posts) = tokio::join!(
        fetch_user(),
        fetch_posts()
    );
    
    println!("User: {}", user);
    println!("Posts: {:?}", posts);
}
```

### tokio::select! — перша завершена задача

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    tokio::select! {
        _ = sleep(Duration::from_secs(1)) => {
            println!("Timeout!");
        }
        result = async_operation() => {
            println!("Got result: {}", result);
        }
    }
}

async fn async_operation() -> i32 {
    sleep(Duration::from_millis(500)).await;
    42
}
```

---

## Async Channels

### mpsc — Multiple Producer, Single Consumer

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    // Створюємо канал з буфером 32
    let (tx, mut rx) = mpsc::channel(32);
    
    // Продюсер
    let tx2 = tx.clone();
    tokio::spawn(async move {
        tx2.send("Hello").await.unwrap();
        tx2.send("World").await.unwrap();
    });
    
    tokio::spawn(async move {
        tx.send("From another task").await.unwrap();
    });
    
    // Консьюмер
    while let Some(msg) = rx.recv().await {
        println!("Received: {}", msg);
    }
}
```

### oneshot — Single Producer, Single Consumer (одне значення)

```rust
use tokio::sync::oneshot;

#[tokio::main]
async fn main() {
    let (tx, rx) = oneshot::channel();
    
    tokio::spawn(async move {
        // Виконуємо роботу
        let result = expensive_computation().await;
        tx.send(result).unwrap();
    });
    
    // Чекаємо результат
    let result = rx.await.unwrap();
    println!("Result: {}", result);
}

async fn expensive_computation() -> i32 {
    tokio::time::sleep(tokio::time::Duration::from_secs(1)).await;
    42
}
```

### broadcast — Multiple Producer, Multiple Consumer

```rust
use tokio::sync::broadcast;

#[tokio::main]
async fn main() {
    let (tx, mut rx1) = broadcast::channel(16);
    let mut rx2 = tx.subscribe();
    
    tokio::spawn(async move {
        while let Ok(msg) = rx1.recv().await {
            println!("Receiver 1: {}", msg);
        }
    });
    
    tokio::spawn(async move {
        while let Ok(msg) = rx2.recv().await {
            println!("Receiver 2: {}", msg);
        }
    });
    
    tx.send("Hello everyone!").unwrap();
    tx.send("Goodbye!").unwrap();
    
    tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
}
```

### watch — Single value, multiple observers

```rust
use tokio::sync::watch;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = watch::channel("initial");
    
    tokio::spawn(async move {
        loop {
            // Чекаємо зміни
            rx.changed().await.unwrap();
            println!("Value changed to: {}", *rx.borrow());
        }
    });
    
    tx.send("first update").unwrap();
    tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
    
    tx.send("second update").unwrap();
    tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
}
```

---

## Async Mutex та RwLock

### tokio::sync::Mutex

```rust
use std::sync::Arc;
use tokio::sync::Mutex;

#[tokio::main]
async fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];
    
    for i in 0..10 {
        let counter = Arc::clone(&counter);
        handles.push(tokio::spawn(async move {
            let mut lock = counter.lock().await;  // .await замість блокування!
            *lock += 1;
            println!("Task {}: counter = {}", i, *lock);
        }));
    }
    
    for handle in handles {
        handle.await.unwrap();
    }
    
    println!("Final: {}", *counter.lock().await);
}
```

### Коли використовувати std::sync::Mutex vs tokio::sync::Mutex?

| `std::sync::Mutex` | `tokio::sync::Mutex` |
|--------------------|----------------------|
| Lock швидкий, не тримати довго | Lock може тримати через .await |
| Блокує потік | Не блокує потік |
| Для CPU-bound операцій | Для async операцій під lock |

---

## Timeouts та Intervals

### Timeout

```rust
use tokio::time::{timeout, Duration};

#[tokio::main]
async fn main() {
    match timeout(Duration::from_secs(1), slow_operation()).await {
        Ok(result) => println!("Got result: {}", result),
        Err(_) => println!("Operation timed out!"),
    }
}

async fn slow_operation() -> i32 {
    tokio::time::sleep(Duration::from_secs(5)).await;
    42
}
```

### Interval

```rust
use tokio::time::{interval, Duration};

#[tokio::main]
async fn main() {
    let mut interval = interval(Duration::from_millis(500));
    
    for i in 0..5 {
        interval.tick().await;
        println!("Tick {}", i);
    }
}
```

---

## Практичні задачі

### Задача 1: Паралельні HTTP запити (симуляція)

**Умова:** Симулюйте паралельні HTTP запити до кількох "серверів". Кожен запит має різний час відповіді. Порівняйте послідовне та паралельне виконання.

**Розв'язання:**

```rust
use tokio::time::{sleep, Duration, Instant};

// Симуляція HTTP запиту
async fn fetch_url(url: &str, delay_ms: u64) -> Result<String, String> {
    println!("  Fetching {}...", url);
    sleep(Duration::from_millis(delay_ms)).await;
    
    if url.contains("error") {
        Err(format!("Failed to fetch {}", url))
    } else {
        Ok(format!("Response from {} ({}ms)", url, delay_ms))
    }
}

// Послідовне виконання
async fn sequential_fetch(urls: &[(&str, u64)]) -> Vec<Result<String, String>> {
    let mut results = vec![];
    for (url, delay) in urls {
        results.push(fetch_url(url, *delay).await);
    }
    results
}

// Паралельне виконання
async fn parallel_fetch(urls: &[(&str, u64)]) -> Vec<Result<String, String>> {
    let futures: Vec<_> = urls
        .iter()
        .map(|(url, delay)| fetch_url(url, *delay))
        .collect();
    
    futures::future::join_all(futures).await
}

// Паралельне з обмеженням одночасних запитів
async fn parallel_fetch_limited(
    urls: &[(&str, u64)],
    max_concurrent: usize,
) -> Vec<Result<String, String>> {
    use tokio::sync::Semaphore;
    use std::sync::Arc;
    
    let semaphore = Arc::new(Semaphore::new(max_concurrent));
    let mut handles = vec![];
    
    for (url, delay) in urls.iter() {
        let permit = semaphore.clone().acquire_owned().await.unwrap();
        let url = url.to_string();
        let delay = *delay;
        
        handles.push(tokio::spawn(async move {
            let result = fetch_url(&url, delay).await;
            drop(permit);  // Звільняємо permit
            result
        }));
    }
    
    let mut results = vec![];
    for handle in handles {
        results.push(handle.await.unwrap());
    }
    results
}

#[tokio::main]
async fn main() {
    println!("=== Паралельні HTTP запити ===\n");
    
    let urls = vec![
        ("https://api1.example.com", 200u64),
        ("https://api2.example.com", 150),
        ("https://api3.example.com", 300),
        ("https://api4.example.com", 100),
        ("https://api5.example.com", 250),
    ];
    
    // Послідовно
    println!("--- Sequential ---");
    let start = Instant::now();
    let results = sequential_fetch(&urls).await;
    println!("Time: {:?}", start.elapsed());
    for r in &results {
        match r {
            Ok(s) => println!("  ✓ {}", s),
            Err(e) => println!("  ✗ {}", e),
        }
    }
    
    // Паралельно
    println!("\n--- Parallel ---");
    let start = Instant::now();
    let results = parallel_fetch(&urls).await;
    println!("Time: {:?}", start.elapsed());
    for r in &results {
        match r {
            Ok(s) => println!("  ✓ {}", s),
            Err(e) => println!("  ✗ {}", e),
        }
    }
    
    // Паралельно з обмеженням
    println!("\n--- Parallel (max 2 concurrent) ---");
    let start = Instant::now();
    let results = parallel_fetch_limited(&urls, 2).await;
    println!("Time: {:?}", start.elapsed());
    for r in &results {
        match r {
            Ok(s) => println!("  ✓ {}", s),
            Err(e) => println!("  ✗ {}", e),
        }
    }
}
```

**Результат:**
```
--- Sequential ---
  Fetching https://api1.example.com...
  Fetching https://api2.example.com...
  ...
Time: 1.000s (сума всіх затримок)

--- Parallel ---
  Fetching https://api1.example.com...
  Fetching https://api2.example.com...
  ... (одночасно)
Time: 300ms (максимальна затримка)

--- Parallel (max 2 concurrent) ---
Time: ~550ms
```

---

### Задача 2: Async Event Loop з каналами

**Умова:** Створіть систему обробки подій для дронів: події надходять через канал, обробник виконує різні дії залежно від типу події.

**Розв'язання:**

```rust
use tokio::sync::mpsc;
use tokio::time::{sleep, Duration, Instant};

#[derive(Debug, Clone)]
enum DroneEvent {
    TakeOff { drone_id: String },
    Move { drone_id: String, x: f64, y: f64 },
    Land { drone_id: String },
    EmergencyStop { drone_id: String },
    StatusRequest { drone_id: String },
    Shutdown,
}

#[derive(Debug)]
struct DroneState {
    id: String,
    x: f64,
    y: f64,
    flying: bool,
}

impl DroneState {
    fn new(id: &str) -> Self {
        DroneState {
            id: id.to_string(),
            x: 0.0,
            y: 0.0,
            flying: false,
        }
    }
}

async fn process_event(state: &mut DroneState, event: DroneEvent) -> Option<String> {
    match event {
        DroneEvent::TakeOff { drone_id } if drone_id == state.id => {
            if state.flying {
                return Some(format!("{} already flying", state.id));
            }
            // Симуляція часу зльоту
            sleep(Duration::from_millis(100)).await;
            state.flying = true;
            Some(format!("{} took off", state.id))
        }
        
        DroneEvent::Move { drone_id, x, y } if drone_id == state.id => {
            if !state.flying {
                return Some(format!("{} must take off first", state.id));
            }
            // Симуляція руху
            let distance = ((x - state.x).powi(2) + (y - state.y).powi(2)).sqrt();
            let time = (distance * 10.0) as u64;
            sleep(Duration::from_millis(time.max(50))).await;
            state.x = x;
            state.y = y;
            Some(format!("{} moved to ({:.1}, {:.1})", state.id, x, y))
        }
        
        DroneEvent::Land { drone_id } if drone_id == state.id => {
            if !state.flying {
                return Some(format!("{} already landed", state.id));
            }
            sleep(Duration::from_millis(100)).await;
            state.flying = false;
            Some(format!("{} landed at ({:.1}, {:.1})", state.id, state.x, state.y))
        }
        
        DroneEvent::EmergencyStop { drone_id } if drone_id == state.id => {
            state.flying = false;
            Some(format!("⚠️ {} EMERGENCY STOP", state.id))
        }
        
        DroneEvent::StatusRequest { drone_id } if drone_id == state.id => {
            let status = if state.flying { "flying" } else { "grounded" };
            Some(format!("{}: {} at ({:.1}, {:.1})", state.id, status, state.x, state.y))
        }
        
        _ => None,  // Подія для іншого дрона
    }
}

async fn drone_controller(
    id: String,
    mut event_rx: mpsc::Receiver<DroneEvent>,
    log_tx: mpsc::Sender<String>,
) {
    let mut state = DroneState::new(&id);
    log_tx.send(format!("[{}] Controller started", id)).await.ok();
    
    while let Some(event) = event_rx.recv().await {
        if matches!(event, DroneEvent::Shutdown) {
            log_tx.send(format!("[{}] Shutting down", id)).await.ok();
            break;
        }
        
        if let Some(msg) = process_event(&mut state, event).await {
            log_tx.send(format!("[{}] {}", id, msg)).await.ok();
        }
    }
}

#[tokio::main]
async fn main() {
    println!("=== Async Event Loop ===\n");
    
    // Канал для логів
    let (log_tx, mut log_rx) = mpsc::channel::<String>(100);
    
    // Канали для дронів
    let (alpha_tx, alpha_rx) = mpsc::channel::<DroneEvent>(32);
    let (beta_tx, beta_rx) = mpsc::channel::<DroneEvent>(32);
    
    // Запускаємо контролери
    let log_tx1 = log_tx.clone();
    tokio::spawn(drone_controller("Alpha".to_string(), alpha_rx, log_tx1));
    
    let log_tx2 = log_tx.clone();
    tokio::spawn(drone_controller("Beta".to_string(), beta_rx, log_tx2));
    
    // Обробник логів
    let log_handle = tokio::spawn(async move {
        while let Some(msg) = log_rx.recv().await {
            println!("{}", msg);
        }
    });
    
    // Генератор подій
    let start = Instant::now();
    
    // Послідовність подій для Alpha
    alpha_tx.send(DroneEvent::TakeOff { drone_id: "Alpha".into() }).await.ok();
    alpha_tx.send(DroneEvent::Move { drone_id: "Alpha".into(), x: 100.0, y: 50.0 }).await.ok();
    
    // Послідовність для Beta (паралельно)
    beta_tx.send(DroneEvent::TakeOff { drone_id: "Beta".into() }).await.ok();
    beta_tx.send(DroneEvent::Move { drone_id: "Beta".into(), x: -50.0, y: 100.0 }).await.ok();
    
    sleep(Duration::from_millis(500)).await;
    
    // Більше подій
    alpha_tx.send(DroneEvent::Move { drone_id: "Alpha".into(), x: 200.0, y: 200.0 }).await.ok();
    beta_tx.send(DroneEvent::StatusRequest { drone_id: "Beta".into() }).await.ok();
    
    sleep(Duration::from_millis(300)).await;
    
    // Завершення
    alpha_tx.send(DroneEvent::Land { drone_id: "Alpha".into() }).await.ok();
    beta_tx.send(DroneEvent::Land { drone_id: "Beta".into() }).await.ok();
    
    sleep(Duration::from_millis(200)).await;
    
    // Shutdown
    alpha_tx.send(DroneEvent::Shutdown).await.ok();
    beta_tx.send(DroneEvent::Shutdown).await.ok();
    
    drop(log_tx);  // Закриваємо канал логів
    
    log_handle.await.ok();
    
    println!("\nTotal time: {:?}", start.elapsed());
}
```

---

### Задача 3: Async з timeout та retry

**Умова:** Реалізуйте функцію, що виконує async операцію з timeout та автоматичним retry при помилках. Використовуйте експоненційний backoff.

**Розв'язання:**

```rust
use tokio::time::{sleep, timeout, Duration, Instant};
use std::future::Future;

#[derive(Debug)]
struct RetryConfig {
    max_attempts: u32,
    initial_delay: Duration,
    max_delay: Duration,
    timeout: Duration,
}

impl Default for RetryConfig {
    fn default() -> Self {
        RetryConfig {
            max_attempts: 3,
            initial_delay: Duration::from_millis(100),
            max_delay: Duration::from_secs(5),
            timeout: Duration::from_secs(10),
        }
    }
}

#[derive(Debug)]
enum RetryError<E> {
    Timeout,
    MaxAttemptsReached(Vec<E>),
}

async fn with_retry<T, E, Fut, F>(
    config: &RetryConfig,
    mut operation: F,
) -> Result<T, RetryError<E>>
where
    F: FnMut() -> Fut,
    Fut: Future<Output = Result<T, E>>,
    E: std::fmt::Debug,
{
    let mut errors = Vec::new();
    let mut delay = config.initial_delay;
    
    for attempt in 1..=config.max_attempts {
        println!("  Attempt {}/{}...", attempt, config.max_attempts);
        
        match timeout(config.timeout, operation()).await {
            Ok(Ok(result)) => {
                println!("  ✓ Success on attempt {}", attempt);
                return Ok(result);
            }
            Ok(Err(e)) => {
                println!("  ✗ Error: {:?}", e);
                errors.push(e);
            }
            Err(_) => {
                println!("  ✗ Timeout after {:?}", config.timeout);
                // Продовжуємо retry
            }
        }
        
        if attempt < config.max_attempts {
            println!("  Waiting {:?} before retry...", delay);
            sleep(delay).await;
            delay = (delay * 2).min(config.max_delay);
        }
    }
    
    Err(RetryError::MaxAttemptsReached(errors))
}

// Симуляція нестабільної операції
struct UnstableService {
    fail_count: std::sync::atomic::AtomicU32,
    fail_until: u32,
}

impl UnstableService {
    fn new(fail_until: u32) -> Self {
        UnstableService {
            fail_count: std::sync::atomic::AtomicU32::new(0),
            fail_until,
        }
    }
    
    async fn call(&self) -> Result<String, String> {
        let count = self.fail_count.fetch_add(1, std::sync::atomic::Ordering::SeqCst);
        
        // Симуляція затримки
        sleep(Duration::from_millis(50 + (count * 20) as u64)).await;
        
        if count < self.fail_until {
            Err(format!("Service unavailable (attempt {})", count + 1))
        } else {
            Ok(format!("Success after {} failures", count))
        }
    }
}

#[tokio::main]
async fn main() {
    println!("=== Async з timeout та retry ===\n");
    
    let config = RetryConfig {
        max_attempts: 5,
        initial_delay: Duration::from_millis(100),
        max_delay: Duration::from_secs(1),
        timeout: Duration::from_secs(2),
    };
    
    // Тест 1: Успіх на 3-й спробі
    println!("--- Test 1: Success on 3rd attempt ---");
    let service = UnstableService::new(2);
    let start = Instant::now();
    
    match with_retry(&config, || service.call()).await {
        Ok(result) => println!("Result: {}", result),
        Err(e) => println!("Failed: {:?}", e),
    }
    println!("Time: {:?}\n", start.elapsed());
    
    // Тест 2: Успіх одразу
    println!("--- Test 2: Immediate success ---");
    let service = UnstableService::new(0);
    let start = Instant::now();
    
    match with_retry(&config, || service.call()).await {
        Ok(result) => println!("Result: {}", result),
        Err(e) => println!("Failed: {:?}", e),
    }
    println!("Time: {:?}\n", start.elapsed());
    
    // Тест 3: Всі спроби провалились
    println!("--- Test 3: All attempts fail ---");
    let service = UnstableService::new(10);
    let start = Instant::now();
    
    match with_retry(&config, || service.call()).await {
        Ok(result) => println!("Result: {}", result),
        Err(e) => println!("Failed: {:?}", e),
    }
    println!("Time: {:?}", start.elapsed());
}
```

---

### Задача 4: Async Actor Model

**Умова:** Реалізуйте простий Actor на Tokio: actor = spawn + channel. Actor обробляє повідомлення послідовно, зберігає стан, відповідає через oneshot channel.

**Розв'язання:**

```rust
use tokio::sync::{mpsc, oneshot};

// Повідомлення для актора
enum DroneMessage {
    GetStatus {
        respond_to: oneshot::Sender<DroneStatus>,
    },
    Move {
        x: f64,
        y: f64,
        respond_to: oneshot::Sender<Result<(), String>>,
    },
    UpdateBattery {
        level: i32,
    },
    Shutdown,
}

#[derive(Debug, Clone)]
struct DroneStatus {
    id: String,
    x: f64,
    y: f64,
    battery: i32,
    active: bool,
}

// Actor
struct DroneActor {
    id: String,
    x: f64,
    y: f64,
    battery: i32,
    receiver: mpsc::Receiver<DroneMessage>,
}

impl DroneActor {
    fn new(id: String, receiver: mpsc::Receiver<DroneMessage>) -> Self {
        DroneActor {
            id,
            x: 0.0,
            y: 0.0,
            battery: 100,
            receiver,
        }
    }
    
    async fn run(mut self) {
        println!("[{}] Actor started", self.id);
        
        while let Some(msg) = self.receiver.recv().await {
            match msg {
                DroneMessage::GetStatus { respond_to } => {
                    let status = DroneStatus {
                        id: self.id.clone(),
                        x: self.x,
                        y: self.y,
                        battery: self.battery,
                        active: true,
                    };
                    let _ = respond_to.send(status);
                }
                
                DroneMessage::Move { x, y, respond_to } => {
                    if self.battery < 10 {
                        let _ = respond_to.send(Err("Battery too low".into()));
                        continue;
                    }
                    
                    // Симуляція руху
                    let distance = ((x - self.x).powi(2) + (y - self.y).powi(2)).sqrt();
                    let cost = (distance / 10.0) as i32;
                    
                    tokio::time::sleep(tokio::time::Duration::from_millis(
                        (distance * 5.0) as u64
                    )).await;
                    
                    self.x = x;
                    self.y = y;
                    self.battery -= cost;
                    
                    println!("[{}] Moved to ({:.1}, {:.1}), battery: {}%", 
                             self.id, self.x, self.y, self.battery);
                    
                    let _ = respond_to.send(Ok(()));
                }
                
                DroneMessage::UpdateBattery { level } => {
                    self.battery = level.min(100).max(0);
                    println!("[{}] Battery updated to {}%", self.id, self.battery);
                }
                
                DroneMessage::Shutdown => {
                    println!("[{}] Shutting down", self.id);
                    break;
                }
            }
        }
        
        println!("[{}] Actor stopped", self.id);
    }
}

// Handle для взаємодії з актором
#[derive(Clone)]
struct DroneHandle {
    sender: mpsc::Sender<DroneMessage>,
}

impl DroneHandle {
    fn new(id: &str) -> Self {
        let (sender, receiver) = mpsc::channel(32);
        let actor = DroneActor::new(id.to_string(), receiver);
        
        tokio::spawn(actor.run());
        
        DroneHandle { sender }
    }
    
    async fn get_status(&self) -> Option<DroneStatus> {
        let (tx, rx) = oneshot::channel();
        self.sender.send(DroneMessage::GetStatus { respond_to: tx }).await.ok()?;
        rx.await.ok()
    }
    
    async fn move_to(&self, x: f64, y: f64) -> Result<(), String> {
        let (tx, rx) = oneshot::channel();
        self.sender
            .send(DroneMessage::Move { x, y, respond_to: tx })
            .await
            .map_err(|_| "Actor unavailable")?;
        rx.await.map_err(|_| "No response")?
    }
    
    async fn update_battery(&self, level: i32) {
        let _ = self.sender.send(DroneMessage::UpdateBattery { level }).await;
    }
    
    async fn shutdown(&self) {
        let _ = self.sender.send(DroneMessage::Shutdown).await;
    }
}

#[tokio::main]
async fn main() {
    println!("=== Async Actor Model ===\n");
    
    // Створюємо акторів
    let alpha = DroneHandle::new("Alpha");
    let beta = DroneHandle::new("Beta");
    
    // Паралельні операції
    let alpha2 = alpha.clone();
    let move1 = tokio::spawn(async move {
        alpha2.move_to(100.0, 50.0).await
    });
    
    let beta2 = beta.clone();
    let move2 = tokio::spawn(async move {
        beta2.move_to(50.0, 100.0).await
    });
    
    // Чекаємо завершення
    let (r1, r2) = tokio::join!(move1, move2);
    println!("\nMove results: {:?}, {:?}", r1.unwrap(), r2.unwrap());
    
    // Перевіряємо статус
    println!("\n--- Status check ---");
    if let Some(status) = alpha.get_status().await {
        println!("Alpha: {:?}", status);
    }
    if let Some(status) = beta.get_status().await {
        println!("Beta: {:?}", status);
    }
    
    // Більше операцій
    println!("\n--- More operations ---");
    alpha.move_to(200.0, 200.0).await.ok();
    alpha.move_to(300.0, 300.0).await.ok();
    
    // Оновлюємо батарею
    alpha.update_battery(5).await;
    
    // Спроба руху з низькою батареєю
    match alpha.move_to(400.0, 400.0).await {
        Ok(()) => println!("Move succeeded"),
        Err(e) => println!("Move failed: {}", e),
    }
    
    // Shutdown
    println!("\n--- Shutdown ---");
    alpha.shutdown().await;
    beta.shutdown().await;
    
    tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
}
```

---

## Домашнє завдання

### Завдання 1: Rate Limiter

Реалізуйте async rate limiter, що дозволяє не більше N операцій за M секунд.

### Завдання 2: Async Pipeline

Створіть pipeline з кількох async етапів: читання → обробка → запис. Дані передаються через канали.

### Завдання 3: Graceful Shutdown

Реалізуйте систему з graceful shutdown: при отриманні сигналу всі задачі мають коректно завершитись.

### Завдання 4: Connection Pool

Створіть async пул з'єднань з обмеженою кількістю, що видає з'єднання через .await.

---

## Підсумок заняття

На цьому занятті ви навчились:

1. **Розуміти async/await**: Future, .await, non-blocking
2. **Використовувати Tokio**: runtime, spawn, join!, select!
3. **Працювати з каналами**: mpsc, oneshot, broadcast, watch
4. **Синхронізувати async код**: Mutex, RwLock, Semaphore
5. **Застосовувати патерни**: timeout, retry, actor model

---

## Перевірте себе

1. Чим async відрізняється від threads?
2. Що робить `.await`?
3. Коли використовувати `tokio::spawn`?
4. Чим `tokio::sync::Mutex` відрізняється від `std::sync::Mutex`?
5. Що таке `select!`?

**Відповіді:**
1. Async — cooperative multitasking на одному потоці (або кількох), threads — preemptive на рівні ОС
2. Призупиняє виконання async функції до готовності Future
3. Коли потрібно виконати задачу незалежно/паралельно
4. tokio::sync::Mutex дозволяє тримати lock через .await без блокування потоку
5. Макрос, що чекає першу завершену з кількох futures

---

## Наступне заняття

Це було останнє заняття базового курсу! Далі рекомендуємо вивчати:
- Мережеве програмування з Tokio (TCP/UDP)
- Web frameworks (Axum, Actix)
- Серіалізацію (serde)
- Бази даних (sqlx, diesel)
