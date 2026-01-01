# Розділ 39: Async Channels та синхронізація

---

## 📋 Анотація

У попередніх розділах ви навчились створювати async задачі та виконувати їх паралельно. Але паралельні задачі рідко працюють ізольовано — вони потребують **комунікації**. Агенти рою мають обмінюватись інформацією про виявлені цілі. Координатор має розсилати команди. Worker'и мають повідомляти про завершення роботи.

У Частині III ви використовували `std::sync::mpsc` для комунікації між потоками. Ці канали працюють, але мають критичний недолік в async контексті: метод `recv()` **блокує весь потік**. Коли async задача блокує потік, вона блокує не тільки себе, а й усі інші задачі на цьому worker'і. Це нівелює переваги асинхронного програмування.

**Tokio надає власні async канали**, де операції `send` та `recv` не блокують потоки. Замість блокування задача "поступається" (yield), дозволяючи іншим задачам працювати. Коли дані готові — задача продовжує виконання.

Цей розділ охоплює всю екосистему async комунікації в Tokio: канали mpsc для point-to-point зв'язку, broadcast для розсилки "один до багатьох", watch для спостереження за станом, oneshot для одноразових відповідей. Ви також дізнаєтесь про async версії Mutex та RwLock, та коли їх варто (і не варто) використовувати.

---

## 🎯 Цілі навчання

Після завершення цього розділу ви зможете:

1. **Пояснити** чому синхронні канали проблемні в async коді
2. **Обирати** правильний тип каналу для конкретної задачі
3. **Використовувати** tokio::sync::mpsc для point-to-point комунікації
4. **Застосовувати** broadcast для патерну "один до багатьох"
5. **Реалізовувати** graceful shutdown через watch канал
6. **Розуміти** різницю між tokio::sync::Mutex та std::sync::Mutex

---

## 📚 Ключові терміни

| Термін | Визначення |
|--------|------------|
| **async channel** | Канал, де операції send/recv не блокують потік ОС, а "поступаються" |
| **bounded channel** | Канал з обмеженою ємністю черги; при заповненні send чекає |
| **unbounded channel** | Канал без обмеження ємності; send ніколи не чекає |
| **backpressure** | Механізм сповільнення швидкого відправника, коли отримувач не встигає |
| **mpsc** | Multiple Producer, Single Consumer — багато відправників, один отримувач |
| **broadcast** | Канал, де кожен отримувач отримує копію кожного повідомлення |
| **watch** | Канал для спостереження за останнім (поточним) значенням |
| **oneshot** | Канал для передачі рівно одного повідомлення |
| **lagging** | Ситуація в broadcast, коли отримувач не встигає і пропускає повідомлення |

---

## 💡 Мотиваційний кейс: Масштабування комунікації в рої

Уявімо рій з 1000 БПЛА-агентів. Архітектура комунікації:

```
                    ┌─────────────────────────────────┐
                    │         КООРДИНАТОР             │
                    │                                 │
                    │  Отримує звіти від агентів     │
                    │  Розсилає команди всім         │
                    │  Слідкує за станом системи     │
                    └─────────────────────────────────┘
                              ▲           │
                              │           │
              ┌───────────────┼───────────┼───────────────┐
              │               │           │               │
         [звіти]         [звіти]     [команди]       [команди]
              │               │           │               │
              ▼               ▼           ▼               ▼
        ┌──────────┐   ┌──────────┐ ┌──────────┐   ┌──────────┐
        │ Агент 1  │   │ Агент 2  │ │ Агент 3  │   │Агент 1000│
        └──────────┘   └──────────┘ └──────────┘   └──────────┘
```

**Якби ми використовували std::sync::mpsc:**

Кожен виклик `recv()` на координаторі блокував би worker потік. При 1000 агентах, що надсилають звіти, координатор постійно блокувався б. Інші async задачі на тому ж worker'і — теж. Ефективність async зводиться нанівець.

**З tokio::sync каналами:**

`recv().await` не блокує. Коли повідомлень немає — координатор "поступається". Worker виконує інші задачі. Коли приходить звіт — координатор прокидається. Тисячі агентів комунікують через кілька worker потоків без блокувань.

Це не теоретична перевага — це різниця між системою, що працює, і системою, що "зависає".

---

## 39.1 ПРОБЛЕМА СИНХРОННИХ КАНАЛІВ В ASYNC КОНТЕКСТІ

### 39.1.1 Анатомія блокування

Щоб зрозуміти проблему, потрібно згадати, як працює Tokio. Runtime має кілька **worker потоків** (зазвичай = кількості ядер CPU). Кожен worker виконує багато async **задач**. Задачі кооперативно ділять час: коли задача чекає (мережа, таймер, канал), вона **поступається**, і worker бере іншу задачу.

Ключове слово — **поступається**. Це можливо тільки якщо операція очікування є async (повертає Future). Синхронна операція не може поступитись — вона просто блокує потік.

Розглянемо, що відбувається при використанні `std::sync::mpsc::recv()` в async:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    WORKER THREAD #1                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Час ────────────────────────────────────────────────────────────►  │
│                                                                     │
│  Task A: [працює][recv() BLOCKED ═══════════════════════════════]   │
│                   ↑                                                 │
│                   │ Потік заблокований!                             │
│                   │                                                 │
│  Task B: [готова до виконання, але...] ─── ЧЕКАЄ ────────────────   │
│  Task C: [готова до виконання, але...] ─── ЧЕКАЄ ────────────────   │
│  Task D: [готова до виконання, але...] ─── ЧЕКАЄ ────────────────   │
│                                                                     │
│  ═══════ = синхронне блокування (потік нічого не робить)           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Task A** викликала `std::sync::mpsc::recv()` і заблокувала **весь потік**. Tasks B, C, D призначені на цей потік, але не можуть виконуватись — потік зайнятий очікуванням.

Якщо у вас multi-thread runtime з 4 workers, і 4 задачі роблять синхронний `recv()` одночасно — весь runtime "заморожений". Жодна інша задача не може виконуватись.

### 39.1.2 Як async канали вирішують проблему

`tokio::sync::mpsc::recv()` повертає **Future**. При `.await` цей Future перевіряє, чи є повідомлення:
- Якщо є — одразу повертає
- Якщо немає — реєструє "waker" і повертає `Poll::Pending`

Коли Future повертає `Pending`, **задача поступається**, і worker бере іншу задачу:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    WORKER THREAD #1                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Час ────────────────────────────────────────────────────────────►  │
│                                                                     │
│  Task A: [працює][recv().await]──►Pending──►[прокинулась][працює]   │
│                                │            ▲                       │
│                                │            │ повідомлення!         │
│                                ▼            │                       │
│  Task B: ──────────────────────[працює]─────┼───────────────────    │
│                                             │                       │
│  Task C: ───────────────────────────────────[працює]────────────    │
│                                                                     │
│  Worker ЗАВЖДИ зайнятий корисною роботою!                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Task A** викликала `recv().await`, отримала `Pending`, поступилась. Worker почав виконувати Task B. Коли прийшло повідомлення для Task A — вона "прокинулась" і продовжила.

### 39.1.3 Порівняння інтерфейсів

Синтаксис дуже схожий, але поведінка принципово різна:

| Операція | std::sync::mpsc | tokio::sync::mpsc |
|----------|-----------------|-------------------|
| Створення | `channel()` | `channel(capacity)` або `unbounded_channel()` |
| Відправка | `tx.send(msg)` | `tx.send(msg).await` |
| Отримання | `rx.recv()` | `rx.recv().await` |
| Блокує потік? | ✅ Так | ❌ Ні |
| Поступається? | ❌ Ні | ✅ Так |
| Для async? | ❌ Не рекомендовано | ✅ Призначено |

---

## 39.2 TOKIO::SYNC::MPSC — ОСНОВНИЙ КАНАЛ КОМУНІКАЦІЇ

### 39.2.1 Два типи каналів: bounded та unbounded

Tokio пропонує два варіанти mpsc каналів з різними характеристиками.

**Bounded channel (з обмеженою ємністю):**

```rust
// Створення bounded каналу з ємністю 32 повідомлення
let (tx, rx) = tokio::sync::mpsc::channel::<Message>(32);
```

Коли черга заповнена (32 повідомлення), `send().await` **чекає**, поки отримувач не звільнить місце. Це називається **backpressure**.

**Unbounded channel (без обмеження):**

```rust
// Створення unbounded каналу
let (tx, rx) = tokio::sync::mpsc::unbounded_channel::<Message>();
```

`send()` ніколи не чекає — повідомлення завжди додається до черги. Черга може рости необмежено (поки є пам'ять).

### 39.2.2 Backpressure: чому це важливо

**Backpressure** — це механізм зворотного зв'язку, що сповільнює швидкого відправника, коли отримувач не встигає обробляти.

Без backpressure (unbounded):

```
┌─────────────────────────────────────────────────────────────────────┐
│                 UNBOUNDED: БЕЗ BACKPRESSURE                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Producer (швидкий):    [msg][msg][msg][msg][msg][msg][msg]...     │
│                              │                                      │
│                              ▼                                      │
│  Черга:                [░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░...]   │
│                         ↑ Росте і росте, поки не вичерпається RAM  │
│                                                                     │
│  Consumer (повільний):            [msg]........[msg]........       │
│                                                                     │
│  Результат: OutOfMemory або деградація системи                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

З backpressure (bounded):

```
┌─────────────────────────────────────────────────────────────────────┐
│                  BOUNDED: З BACKPRESSURE                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Producer:    [msg][msg][msg]────WAIT────[msg][msg]────WAIT────    │
│                              │                    │                 │
│                              ▼                    ▼                 │
│  Черга (max 3):    [░░░]  FULL!    [░░]         [░░░] FULL!        │
│                              │                    │                 │
│                              ▼                    ▼                 │
│  Consumer:         [msg]─────────[msg]───────[msg]─────────        │
│                                                                     │
│  Результат: Система в балансі, пам'ять контрольована               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Коли що обирати:**

| Сценарій | Рекомендація | Причина |
|----------|--------------|---------|
| Швидкість producer непередбачувана | Bounded | Захист від переповнення |
| Producer повільніший за consumer | Unbounded | Немає ризику переповнення |
| Критичні повідомлення без втрат | Bounded | Backpressure замість втрати |
| Fire-and-forget логування | Unbounded | Логування не має блокувати |
| Мережеві запити | Bounded | Обмежує паралелізм |

### 39.2.3 Базове використання bounded каналу

**Постановка задачі:** Створимо простий приклад producer-consumer з bounded каналом, щоб побачити backpressure в дії.

```rust
use tokio::sync::mpsc;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    // Створюємо bounded канал з ємністю 3
    // Це означає: максимум 3 повідомлення можуть чекати в черзі
    let (tx, mut rx) = mpsc::channel::<i32>(3);
    
    // Producer: надсилає 10 повідомлень швидко
    let producer = tokio::spawn(async move {
        for i in 1..=10 {
            println!("[Producer] Надсилаю повідомлення {}...", i);
            
            // send().await чекатиме, якщо черга заповнена
            tx.send(i).await.expect("Канал закрито");
            
            println!("[Producer] Повідомлення {} надіслано", i);
        }
        println!("[Producer] Завершено");
    });
    
    // Consumer: обробляє повідомлення повільно
    let consumer = tokio::spawn(async move {
        while let Some(msg) = rx.recv().await {
            println!("[Consumer] Отримано: {}", msg);
            
            // Симуляція повільної обробки
            sleep(Duration::from_millis(500)).await;
            
            println!("[Consumer] Оброблено: {}", msg);
        }
        println!("[Consumer] Канал закрито, завершую");
    });
    
    // Чекаємо обох
    let _ = tokio::join!(producer, consumer);
}
```

**Розглянемо, що відбувається під час виконання:**

1. Producer починає надсилати повідомлення 1, 2, 3 — вони потрапляють у чергу
2. Повідомлення 4 — черга заповнена (ємність 3)! Producer чекає на `send().await`
3. Consumer обробляє повідомлення 1 — звільняється місце
4. Producer надсилає 4, одразу чекає на 5 (черга знову повна)
5. Процес продовжується: producer не може "втекти" від consumer

Вивід буде приблизно такий:

```
[Producer] Надсилаю повідомлення 1...
[Producer] Повідомлення 1 надіслано
[Producer] Надсилаю повідомлення 2...
[Producer] Повідомлення 2 надіслано
[Producer] Надсилаю повідомлення 3...
[Producer] Повідомлення 3 надіслано
[Producer] Надсилаю повідомлення 4...     <- Чекає!
[Consumer] Отримано: 1
[Consumer] Оброблено: 1
[Producer] Повідомлення 4 надіслано       <- Місце звільнилось
[Producer] Надсилаю повідомлення 5...     <- Знову чекає
...
```

### 39.2.4 Обробка закриття каналу

Канал закривається, коли всі `Sender` знищені (dropped). Важливо правильно обробляти цю ситуацію.

**Постановка задачі:** Покажемо, як коректно завершувати роботу при закритті каналу.

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel::<String>(10);
    
    // Відправник у окремій задачі
    let sender_task = tokio::spawn(async move {
        for i in 1..=5 {
            let message = format!("Повідомлення #{}", i);
            
            // send() повертає Result
            // Err означає, що отримувач знищений
            match tx.send(message).await {
                Ok(()) => println!("[Sender] Надіслано #{}", i),
                Err(e) => {
                    // e.0 містить повідомлення, яке не вдалось надіслати
                    println!("[Sender] Помилка: отримувач зник. Втрачено: {}", e.0);
                    break;
                }
            }
        }
        println!("[Sender] Завершую роботу");
        // tx автоматично drop'ається тут
    });
    
    // Отримувач
    let receiver_task = tokio::spawn(async move {
        // recv() повертає Option<T>
        // None означає, що канал закрито (всі Sender знищені)
        while let Some(msg) = rx.recv().await {
            println!("[Receiver] Отримано: {}", msg);
        }
        
        // Вийшли з циклу — канал закрито
        println!("[Receiver] Канал закрито, завершую");
    });
    
    let _ = tokio::join!(sender_task, receiver_task);
}
```

**Що відбувається у цій програмі:**

1. Sender надсилає 5 повідомлень і завершує задачу
2. При завершенні задачі `tx` автоматично drop'ається
3. Канал стає "закритим" — немає активних відправників
4. Наступний `rx.recv().await` повертає `None`
5. Цикл `while let Some(...)` завершується
6. Receiver коректно завершує роботу

Цей патерн — **стандартний спосіб graceful shutdown** через канали.

### 39.2.5 Множинні відправники

MPSC — Multiple Producer, Single Consumer. Можна мати багато відправників через клонування `Sender`:

**Постановка задачі:** Покажемо, як кілька агентів надсилають звіти одному координатору.

```rust
use tokio::sync::mpsc;
use tokio::time::{sleep, Duration};

#[derive(Debug)]
struct AgentReport {
    agent_id: u32,
    message: String,
}

#[tokio::main]
async fn main() {
    // Один канал для всіх агентів
    let (tx, mut rx) = mpsc::channel::<AgentReport>(100);
    
    // Запускаємо 5 агентів, кожен отримує клон Sender
    let mut agent_handles = Vec::new();
    
    for agent_id in 1..=5 {
        // Клонуємо Sender для кожного агента
        let agent_tx = tx.clone();
        
        let handle = tokio::spawn(async move {
            for report_num in 1..=3 {
                let report = AgentReport {
                    agent_id,
                    message: format!("Звіт #{} від агента {}", report_num, agent_id),
                };
                
                // Надсилаємо звіт координатору
                if agent_tx.send(report).await.is_err() {
                    println!("[Агент {}] Координатор недоступний!", agent_id);
                    break;
                }
                
                // Пауза між звітами
                sleep(Duration::from_millis(100 * agent_id as u64)).await;
            }
            println!("[Агент {}] Завершив роботу", agent_id);
        });
        
        agent_handles.push(handle);
    }
    
    // ВАЖЛИВО: Drop оригінальний tx!
    // Інакше канал ніколи не закриється
    drop(tx);
    
    // Координатор отримує звіти від усіх агентів
    let coordinator = tokio::spawn(async move {
        let mut total_reports = 0;
        
        while let Some(report) = rx.recv().await {
            total_reports += 1;
            println!("[Координатор] Отримано: {:?}", report);
        }
        
        println!("[Координатор] Всього отримано {} звітів", total_reports);
    });
    
    // Чекаємо завершення всіх агентів та координатора
    for handle in agent_handles {
        let _ = handle.await;
    }
    let _ = coordinator.await;
}
```

**Ключові моменти цієї програми:**

1. **`tx.clone()`** — кожен агент отримує власну копію Sender. Всі вони пишуть в один канал.

2. **`drop(tx)`** — критично важливо! Ми створили оригінальний `tx`, потім зробили 5 клонів. Якщо не drop'нути оригінал, канал ніколи не закриється (завжди є хоча б один живий Sender).

3. **Порядок повідомлень** — MPSC гарантує FIFO порядок для кожного Sender, але повідомлення від різних Sender'ів можуть перемішуватись.

### 39.2.6 Unbounded канал: коли і як використовувати

**Постановка задачі:** Покажемо unbounded канал для логування, де блокування неприпустиме.

```rust
use tokio::sync::mpsc;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    // Unbounded канал — send() ніколи не чекає
    let (log_tx, mut log_rx) = mpsc::unbounded_channel::<String>();
    
    // Logger task — обробляє логи у фоні
    let logger = tokio::spawn(async move {
        while let Some(log_entry) = log_rx.recv().await {
            // Симуляція запису в файл
            println!("[LOG] {}", log_entry);
            sleep(Duration::from_millis(50)).await;
        }
    });
    
    // Основна робота — логування не має заважати
    let worker = tokio::spawn(async move {
        for i in 1..=20 {
            // send() на unbounded каналі повертає Result, але НЕ Future
            // Тобто він НІКОЛИ не чекає
            let _ = log_tx.send(format!("Робота #{} виконана", i));
            
            // Продовжуємо роботу без затримки на логування
            println!("[Worker] Виконую роботу #{}", i);
        }
        println!("[Worker] Завершено");
        // log_tx drop'ається тут
    });
    
    let _ = worker.await;
    let _ = logger.await;
}
```

**Чому unbounded підходить для логування:**

1. Логування не має блокувати основну роботу
2. Втрата логів при завершенні — прийнятна (краще ніж зависання)
3. Якщо лог-черга росте — це симптом проблеми, яку треба вирішувати (повільний I/O)

**Увага:** `unbounded_channel().send()` повертає `Result`, не `Future`. Його не потрібно await'ити — він завжди успішний (якщо receiver живий).

---

## 39.3 BROADCAST — КАНАЛ "ОДИН ДО БАГАТЬОХ"

### 39.3.1 Концепція та призначення

MPSC — це "багато до одного": багато відправників, один отримувач. Але іноді потрібен протилежний патерн: **один відправник, багато отримувачів**, де кожен отримувач отримує **кожне** повідомлення.

Типові сценарії використання:

- Координатор розсилає команди всім агентам рою
- Система сповіщень: кожен підписник отримує новину
- Розповсюдження оновленої конфігурації всім компонентам
- Трансляція ринкових даних всім торговим ботам

**Візуалізація broadcast:**

```
                    ┌─────────────────────┐
                    │      SENDER         │
                    │   send("SCAN")      │
                    └──────────┬──────────┘
                               │
            ┌──────────────────┼──────────────────┐
            │                  │                  │
            ▼                  ▼                  ▼
     ┌────────────┐     ┌────────────┐     ┌────────────┐
     │ Receiver A │     │ Receiver B │     │ Receiver C │
     │            │     │            │     │            │
     │ "SCAN" ✓   │     │ "SCAN" ✓   │     │ "SCAN" ✓   │
     └────────────┘     └────────────┘     └────────────┘
     
     Кожен отримує копію кожного повідомлення!
```

### 39.3.2 Створення та використання broadcast

**Постановка задачі:** Координатор розсилає команди групі агентів. Кожен агент має отримати кожну команду.

```rust
use tokio::sync::broadcast;
use tokio::time::{sleep, Duration};

#[derive(Clone, Debug)]
enum Command {
    Scan,
    Move { x: i32, y: i32 },
    Report,
    Shutdown,
}

#[tokio::main]
async fn main() {
    // Створюємо broadcast канал з ємністю 16
    // Ємність визначає, скільки повідомлень буферизується
    let (cmd_tx, _) = broadcast::channel::<Command>(16);
    
    // Запускаємо 3 агентів, кожен підписується на команди
    let mut agent_handles = Vec::new();
    
    for agent_id in 1..=3 {
        // subscribe() створює нового отримувача
        // ВАЖЛИВО: викликається на Sender, не на Receiver!
        let mut cmd_rx = cmd_tx.subscribe();
        
        let handle = tokio::spawn(async move {
            loop {
                match cmd_rx.recv().await {
                    Ok(cmd) => {
                        println!("[Агент {}] Отримано команду: {:?}", agent_id, cmd);
                        
                        // Обробка команди
                        match cmd {
                            Command::Shutdown => {
                                println!("[Агент {}] Завершую роботу", agent_id);
                                break;
                            }
                            Command::Scan => {
                                println!("[Агент {}] Сканую...", agent_id);
                            }
                            Command::Move { x, y } => {
                                println!("[Агент {}] Переміщуюсь до ({}, {})", agent_id, x, y);
                            }
                            Command::Report => {
                                println!("[Агент {}] Надсилаю звіт", agent_id);
                            }
                        }
                    }
                    Err(broadcast::error::RecvError::Lagged(n)) => {
                        // Пропустили n повідомлень — агент не встигав
                        println!("[Агент {}] УВАГА: пропущено {} команд!", agent_id, n);
                    }
                    Err(broadcast::error::RecvError::Closed) => {
                        println!("[Агент {}] Канал закрито", agent_id);
                        break;
                    }
                }
            }
        });
        
        agent_handles.push(handle);
    }
    
    // Координатор надсилає команди
    let coordinator = tokio::spawn(async move {
        let commands = vec![
            Command::Scan,
            Command::Move { x: 10, y: 20 },
            Command::Report,
            Command::Move { x: 5, y: 5 },
            Command::Shutdown,
        ];
        
        for cmd in commands {
            println!("[Координатор] Надсилаю: {:?}", cmd);
            
            // send() повертає кількість отримувачів
            match cmd_tx.send(cmd) {
                Ok(n) => println!("[Координатор] Отримано {} агентами", n),
                Err(_) => println!("[Координатор] Немає активних агентів!"),
            }
            
            sleep(Duration::from_millis(100)).await;
        }
    });
    
    let _ = coordinator.await;
    for handle in agent_handles {
        let _ = handle.await;
    }
}
```

**Ключові особливості broadcast каналу:**

1. **`subscribe()` на Sender** — нові отримувачі створюються з Sender, не з Receiver. Це дозволяє додавати підписників у будь-який момент.

2. **`Clone` requirement** — тип повідомлення має реалізувати `Clone`, бо кожен отримувач отримує копію.

3. **`send()` не чекає** — повідомлення надсилається миттєво. Якщо немає отримувачів — повідомлення втрачається.

4. **`Lagged` error** — якщо отримувач не встигає і буфер переповнився, старі повідомлення витісняються.

### 39.3.3 Проблема Lagging

Broadcast канал має обмежену ємність. Якщо повідомлення приходять швидше, ніж отримувач обробляє, старі повідомлення витісняються новими.

**Постановка задачі:** Продемонструємо ситуацію lagging і як її обробляти.

```rust
use tokio::sync::broadcast;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    // Маленька ємність для демонстрації
    let (tx, mut rx) = broadcast::channel::<i32>(4);
    
    // Швидкий відправник
    let sender = tokio::spawn(async move {
        for i in 1..=20 {
            let _ = tx.send(i);
            println!("[Sender] Надіслано {}", i);
        }
    });
    
    // Повільний отримувач — почне після відправника
    let receiver = tokio::spawn(async move {
        // Затримка, щоб sender встиг заповнити буфер
        sleep(Duration::from_millis(10)).await;
        
        loop {
            match rx.recv().await {
                Ok(value) => {
                    println!("[Receiver] Отримано: {}", value);
                }
                Err(broadcast::error::RecvError::Lagged(n)) => {
                    println!("[Receiver] ⚠️ ПРОПУЩЕНО {} повідомлень!", n);
                }
                Err(broadcast::error::RecvError::Closed) => {
                    println!("[Receiver] Канал закрито");
                    break;
                }
            }
        }
    });
    
    let _ = sender.await;
    let _ = receiver.await;
}
```

**Що відбувається:**

1. Sender швидко надсилає 20 повідомлень
2. Буфер має ємність 4, тому 1-16 витісняються
3. Receiver починає читати і отримує `Lagged(16)` — пропущено 16 повідомлень
4. Потім отримує 17, 18, 19, 20

**Як уникнути lagging:**
- Збільшити ємність каналу
- Прискорити обробку в отримувачах
- Використовувати окремі mpsc канали, якщо втрата неприпустима

---

## 39.4 WATCH — КАНАЛ ДЛЯ СПОСТЕРЕЖЕННЯ ЗА СТАНОМ

### 39.4.1 Концепція: тільки останнє значення

Іноді потрібно не чергу повідомлень, а **поточне значення** чогось:
- Статус системи (Running / Paused / Stopping)
- Конфігурація (яку можуть оновити)
- Прапорець завершення роботи

Watch канал зберігає **одне значення**, і отримувачі бачать **найновіше**:

```
┌─────────────────────────────────────────────────────────────────────┐
│                          WATCH CHANNEL                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Sender: [set(1)] [set(2)] [set(3)] [set(4)] [set(5)]              │
│                                              │                      │
│                                              ▼                      │
│  Channel:                              [ 5 ]  ← зберігає ТІЛЬКИ    │
│                                              │    останнє значення │
│                                              ▼                      │
│  Receiver A: ─────────────────────────── [ 5 ] (бачить останнє)    │
│  Receiver B: ─────────────────────────── [ 5 ] (теж бачить 5)      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

На відміну від broadcast, де кожен отримує **кожне** повідомлення, watch показує лише **поточний стан**.

### 39.4.2 Graceful shutdown через watch

Найпоширеніше використання watch — сигнал завершення роботи.

**Постановка задачі:** Координатор керує життєвим циклом worker'ів. При отриманні сигналу завершення всі worker'и мають коректно зупинитись.

```rust
use tokio::sync::watch;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    // Створюємо watch канал з початковим значенням false
    // false = продовжуємо працювати
    // true = завершуємо роботу
    let (shutdown_tx, shutdown_rx) = watch::channel(false);
    
    // Запускаємо кілька worker'ів
    let mut handles = Vec::new();
    
    for worker_id in 1..=3 {
        // Клонуємо receiver для кожного worker'а
        let mut rx = shutdown_rx.clone();
        
        let handle = tokio::spawn(async move {
            println!("[Worker {}] Запущено", worker_id);
            
            let mut iteration = 0;
            
            loop {
                // Перевіряємо сигнал завершення
                // borrow() повертає посилання на поточне значення
                if *rx.borrow() {
                    println!("[Worker {}] Отримано сигнал shutdown", worker_id);
                    break;
                }
                
                // Виконуємо корисну роботу
                iteration += 1;
                println!("[Worker {}] Ітерація {}", worker_id, iteration);
                
                // Пауза між ітераціями
                sleep(Duration::from_millis(200)).await;
            }
            
            // Cleanup при завершенні
            println!("[Worker {}] Завершую роботу, зроблено {} ітерацій", 
                     worker_id, iteration);
        });
        
        handles.push(handle);
    }
    
    // Координатор: даємо worker'ам попрацювати, потім зупиняємо
    tokio::spawn(async move {
        println!("[Coordinator] Система працює...");
        sleep(Duration::from_secs(1)).await;
        
        println!("[Coordinator] Надсилаю сигнал shutdown");
        shutdown_tx.send(true).expect("Receivers dropped");
        
        println!("[Coordinator] Сигнал надіслано");
    });
    
    // Чекаємо завершення всіх worker'ів
    for handle in handles {
        let _ = handle.await;
    }
    
    println!("[Main] Всі worker'и завершили роботу");
}
```

**Як працює graceful shutdown:**

1. Watch канал створюється зі значенням `false` (працюємо)
2. Кожен worker отримує клон receiver'а
3. Worker'и періодично перевіряють значення через `borrow()`
4. Координатор викликає `send(true)` — значення змінюється
5. Наступна перевірка `borrow()` у кожного worker'а поверне `true`
6. Worker'и виходять з циклу та завершуються коректно

### 39.4.3 Очікування змін через changed()

Замість постійного polling можна чекати на зміну значення:

**Постановка задачі:** Worker має реагувати на зміну конфігурації без постійного опитування.

```rust
use tokio::sync::watch;
use tokio::time::{sleep, Duration};

#[derive(Clone, Debug)]
struct Config {
    scan_interval_ms: u64,
    debug_mode: bool,
}

#[tokio::main]
async fn main() {
    let initial_config = Config {
        scan_interval_ms: 1000,
        debug_mode: false,
    };
    
    let (config_tx, config_rx) = watch::channel(initial_config);
    
    // Worker, що реагує на зміни конфігурації
    let worker_rx = config_rx.clone();
    let worker = tokio::spawn(async move {
        let mut rx = worker_rx;
        
        loop {
            // Отримуємо поточну конфігурацію
            let config = rx.borrow().clone();
            println!("[Worker] Застосовую конфігурацію: {:?}", config);
            
            // Чекаємо на зміну конфігурації
            // changed() повертає Ok(()) коли значення змінилось
            // або Err якщо sender закрито
            if rx.changed().await.is_err() {
                println!("[Worker] Канал конфігурації закрито");
                break;
            }
            
            println!("[Worker] Конфігурацію оновлено!");
        }
    });
    
    // Адміністратор змінює конфігурацію
    tokio::spawn(async move {
        sleep(Duration::from_millis(500)).await;
        
        println!("[Admin] Оновлюю конфігурацію: debug_mode = true");
        config_tx.send(Config {
            scan_interval_ms: 1000,
            debug_mode: true,
        }).unwrap();
        
        sleep(Duration::from_millis(500)).await;
        
        println!("[Admin] Оновлюю конфігурацію: scan_interval = 500");
        config_tx.send(Config {
            scan_interval_ms: 500,
            debug_mode: true,
        }).unwrap();
        
        sleep(Duration::from_millis(500)).await;
        println!("[Admin] Завершую");
        // config_tx drop'ається
    }).await.unwrap();
    
    let _ = worker.await;
}
```

**Переваги changed() над polling:**

1. Не витрачає CPU на постійні перевірки
2. Реагує миттєво на зміни
3. Автоматично завершується при закритті каналу

---

## 39.5 ONESHOT — ОДНОРАЗОВЕ ПОВІДОМЛЕННЯ

### 39.5.1 Концепція та призначення

Oneshot канал призначений для передачі **рівно одного** повідомлення. Після відправки канал закривається.

Типові сценарії:
- Request-Response патерн: надіслати запит, отримати одну відповідь
- Результат фонової операції
- Сигнал готовності

### 39.5.2 Request-Response патерн

**Постановка задачі:** Клієнт надсилає запит серверу і чекає відповідь. Кожен запит потребує окремого каналу для відповіді.

```rust
use tokio::sync::{mpsc, oneshot};

// Запит містить дані та канал для відповіді
struct Request {
    query: String,
    response_channel: oneshot::Sender<String>,
}

#[tokio::main]
async fn main() {
    // Канал для запитів до сервера
    let (request_tx, mut request_rx) = mpsc::channel::<Request>(10);
    
    // Сервер: обробляє запити
    let server = tokio::spawn(async move {
        while let Some(req) = request_rx.recv().await {
            println!("[Server] Отримано запит: {}", req.query);
            
            // Обробляємо запит
            let response = format!("Відповідь на: {}", req.query);
            
            // Надсилаємо відповідь через oneshot
            // Ігноруємо помилку (клієнт міг відключитись)
            let _ = req.response_channel.send(response);
        }
        println!("[Server] Завершую роботу");
    });
    
    // Клієнт: надсилає запити
    let client = tokio::spawn(async move {
        for i in 1..=3 {
            // Створюємо oneshot канал для цього запиту
            let (resp_tx, resp_rx) = oneshot::channel();
            
            let request = Request {
                query: format!("Запит #{}", i),
                response_channel: resp_tx,
            };
            
            println!("[Client] Надсилаю: {}", request.query);
            
            // Надсилаємо запит
            request_tx.send(request).await.expect("Server died");
            
            // Чекаємо відповідь
            match resp_rx.await {
                Ok(response) => println!("[Client] Отримано: {}", response),
                Err(_) => println!("[Client] Сервер не відповів"),
            }
        }
        
        println!("[Client] Завершую");
        // request_tx drop'ається
    });
    
    let _ = client.await;
    let _ = server.await;
}
```

**Як працює Request-Response:**

1. Клієнт створює oneshot канал `(resp_tx, resp_rx)`
2. Клієнт включає `resp_tx` у запит і надсилає серверу
3. Сервер отримує запит, обробляє, надсилає відповідь через `resp_tx`
4. Клієнт чекає на `resp_rx.await`
5. Oneshot гарантує рівно одну відповідь

---

## 39.6 ASYNC ПРИМІТИВИ СИНХРОНІЗАЦІЇ

### 39.6.1 tokio::sync::Mutex vs std::sync::Mutex

Tokio надає власну версію Mutex. Коли використовувати яку?

**std::sync::Mutex:**
- `lock()` **блокує потік** до отримання lock'а
- Швидший для коротких критичних секцій
- Підходить, коли lock тримається **без await всередині**

**tokio::sync::Mutex:**
- `lock().await` **не блокує потік**
- Потрібен, коли всередині критичної секції є `.await`
- Повільніший через async overhead

**Постановка задачі:** Порівняємо два підходи на прикладі.

```rust
use std::sync::Arc;
use tokio::time::{sleep, Duration};

// Сценарій 1: Короткі операції — std::sync::Mutex
mod short_operations {
    use std::sync::{Arc, Mutex};
    
    pub async fn example() {
        let counter = Arc::new(Mutex::new(0));
        
        let mut handles = Vec::new();
        
        for _ in 0..10 {
            let counter = counter.clone();
            handles.push(tokio::spawn(async move {
                for _ in 0..100 {
                    // Lock тримається дуже коротко
                    // Немає await всередині
                    let mut count = counter.lock().unwrap();
                    *count += 1;
                    // Lock звільняється одразу
                }
            }));
        }
        
        for h in handles {
            let _ = h.await;
        }
        
        println!("Результат: {}", *counter.lock().unwrap());
    }
}

// Сценарій 2: Async операції всередині — tokio::sync::Mutex
mod async_operations {
    use std::sync::Arc;
    use tokio::sync::Mutex;
    use tokio::time::{sleep, Duration};
    
    pub async fn example() {
        let data = Arc::new(Mutex::new(Vec::new()));
        
        let mut handles = Vec::new();
        
        for i in 0..5 {
            let data = data.clone();
            handles.push(tokio::spawn(async move {
                // Lock тримається через await!
                let mut vec = data.lock().await;
                
                // Async операція під lock
                sleep(Duration::from_millis(10)).await;
                
                vec.push(i);
                
                // Lock звільняється при drop
            }));
        }
        
        for h in handles {
            let _ = h.await;
        }
        
        println!("Результат: {:?}", *data.lock().await);
    }
}

#[tokio::main]
async fn main() {
    println!("=== Short operations (std::sync::Mutex) ===");
    short_operations::example().await;
    
    println!("\n=== Async operations (tokio::sync::Mutex) ===");
    async_operations::example().await;
}
```

**Правило вибору:**

| Ситуація | Використовуйте |
|----------|----------------|
| Немає `.await` під lock | `std::sync::Mutex` (швидше) |
| Є `.await` під lock | `tokio::sync::Mutex` (обов'язково) |
| Сумніваєтесь | `tokio::sync::Mutex` (безпечніше) |

### 39.6.2 Semaphore: обмеження паралелізму

Semaphore обмежує кількість задач, що можуть одночасно виконувати певну операцію.

**Постановка задачі:** Обмежити кількість одночасних HTTP запитів до 3.

```rust
use std::sync::Arc;
use tokio::sync::Semaphore;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    // Semaphore з 3 permits — максимум 3 одночасні операції
    let semaphore = Arc::new(Semaphore::new(3));
    
    let mut handles = Vec::new();
    
    // Запускаємо 10 "запитів"
    for i in 1..=10 {
        let sem = semaphore.clone();
        
        handles.push(tokio::spawn(async move {
            println!("[Запит {}] Чекаю на permit...", i);
            
            // acquire() чекає, поки не буде вільного permit
            let _permit = sem.acquire().await.expect("Semaphore closed");
            
            // Тепер у нас є permit — виконуємо операцію
            println!("[Запит {}] Отримано permit, виконую...", i);
            
            // Симуляція HTTP запиту
            sleep(Duration::from_millis(500)).await;
            
            println!("[Запит {}] Завершено, звільняю permit", i);
            
            // _permit drop'ається тут, звільняючи permit
        }));
    }
    
    for h in handles {
        let _ = h.await;
    }
}
```

**Що відбувається:**

1. Semaphore має 3 permits (3 "слоти")
2. Запити 1-3 отримують permits одразу
3. Запити 4-10 чекають в `acquire().await`
4. Коли запит 1 завершується — permit звільняється
5. Запит 4 отримує permit і починає виконання
6. І так далі...

Семафор гарантує: **ніколи більше 3 одночасних операцій**.

---

## 39.7 ВИБІР ПРАВИЛЬНОГО КАНАЛУ

### 39.7.1 Таблиця вибору

| Потреба | Канал | Приклад |
|---------|-------|---------|
| Багато → Один | `mpsc` | Агенти надсилають звіти координатору |
| Один → Багато (кожному кожне) | `broadcast` | Координатор розсилає команди |
| Один → Багато (тільки останнє) | `watch` | Статус системи, конфігурація |
| Одна відповідь | `oneshot` | Request-response |
| Необмежена черга | `mpsc::unbounded` | Логування |
| Обмежена черга з backpressure | `mpsc` (bounded) | Обробка запитів |

### 39.7.2 Архітектура рою з каналами

Ось як канали можуть використовуватись у рої БПЛА:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    АРХІТЕКТУРА КОМУНІКАЦІЇ РОЮ                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│                    ┌─────────────────────┐                         │
│                    │    КООРДИНАТОР      │                         │
│                    │                     │                         │
│                    │  [mpsc::Receiver]   │ ← звіти від агентів     │
│                    │  [broadcast::Sender]│ → команди всім          │
│                    │  [watch::Sender]    │ → стан системи          │
│                    │                     │                         │
│                    └──────────┬──────────┘                         │
│                               │                                     │
│         ┌─────────────────────┼─────────────────────┐              │
│         │                     │                     │              │
│         ▼                     ▼                     ▼              │
│  ┌────────────────┐   ┌────────────────┐   ┌────────────────┐     │
│  │   АГЕНТ #1     │   │   АГЕНТ #2     │   │   АГЕНТ #N     │     │
│  │                │   │                │   │                │     │
│  │ mpsc::Sender   │   │ mpsc::Sender   │   │ mpsc::Sender   │     │
│  │ broadcast::Rx  │   │ broadcast::Rx  │   │ broadcast::Rx  │     │
│  │ watch::Rx      │   │ watch::Rx      │   │ watch::Rx      │     │
│  │                │   │                │   │                │     │
│  └────────────────┘   └────────────────┘   └────────────────┘     │
│                                                                     │
│  Потоки даних:                                                     │
│  ─────────────                                                     │
│  mpsc:      Агенти → Координатор (звіти, виявлення)               │
│  broadcast: Координатор → Всі агенти (команди)                     │
│  watch:     Координатор → Всі агенти (статус: Running/Shutdown)   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 39.8 ЛАБОРАТОРНА РОБОТА

### Мета роботи

Закріпити практичні навички роботи з async каналами та примітивами синхронізації.

### Завдання 1: Producer-Consumer з backpressure (3 бали)

Створіть bounded mpsc канал (ємність 5). Producer надсилає числа 1-20 з затримкою 100мс. Consumer обробляє з затримкою 300мс.

Спостерігайте та задокументуйте:
- Коли producer починає чекати?
- Як довго триває обробка всіх повідомлень?
- Що станеться, якщо змінити ємність на 10?

### Завдання 2: Система команд через broadcast (4 бали)

Реалізуйте систему, де:
- Координатор надсилає команди: "SCAN", "MOVE", "REPORT", "SHUTDOWN"
- 3 агенти отримують та логують кожну команду
- Агенти завершуються при отриманні "SHUTDOWN"
- Кожен агент — окрема tokio::spawn задача

### Завдання 3: Graceful shutdown з watch (3 бали)

Реалізуйте систему з:
- 5 worker задач, що працюють у нескінченному циклі
- Кожен worker виводить "Heartbeat" кожні 200мс
- Watch канал для сигналу завершення
- Через 2 секунди координатор надсилає shutdown
- Всі worker'и мають коректно завершитись

---

## 39.9 ТИПОВІ ПОМИЛКИ ТА TROUBLESHOOTING

### 39.9.1 Використання std каналів в async

**Симптом:** Програма "зависає" або працює дуже повільно.

**Причина:** `std::sync::mpsc::recv()` блокує worker потік, інші задачі не можуть виконуватись.

**Рішення:** Замініть на `tokio::sync::mpsc`.

### 39.9.2 Канал ніколи не закривається

**Симптом:** `recv().await` ніколи не повертає `None`, програма зависає.

**Причина:** Існує живий Sender, який не drop'нуто.

**Рішення:** Перевірте, що всі Sender'и drop'аються. Використовуйте `drop(tx)` явно, якщо потрібно.

### 39.9.3 Lagging в broadcast

**Симптом:** Отримувачі пропускають повідомлення, отримують `RecvError::Lagged`.

**Причина:** Ємність каналу замала для швидкості генерації повідомлень.

**Рішення:**
- Збільшіть ємність каналу
- Прискоріть обробку в отримувачах
- Обробіть `Lagged` error gracefully

---

## 39.10 РЕЗЮМЕ

### Ключові концепції

**Async канали** не блокують worker потоки — задачі "поступаються" при очікуванні.

**mpsc** (bounded/unbounded) — основний канал для point-to-point комунікації. Bounded забезпечує backpressure.

**broadcast** — один відправник, багато отримувачів. Кожен отримує кожне повідомлення (якщо встигає).

**watch** — для спостереження за поточним станом. Зберігає одне значення. Ідеально для graceful shutdown.

**oneshot** — для одноразової відповіді. Request-response патерн.

### Правила вибору каналів

| Патерн | Канал |
|--------|-------|
| Багато → Один | `mpsc` |
| Один → Всі (кожному кожне) | `broadcast` |
| Один → Всі (тільки останнє) | `watch` |
| Одна відповідь | `oneshot` |
| Без блокування відправника | `mpsc::unbounded` |
| З backpressure | `mpsc` bounded |

### Правила для Mutex

| Ситуація | Mutex |
|----------|-------|
| Немає `.await` під lock | `std::sync::Mutex` |
| Є `.await` під lock | `tokio::sync::Mutex` |

---

## 🔗 ЗВ'ЯЗОК З НАСТУПНИМ РОЗДІЛОМ

Канали передають **окремі повідомлення**. Але що, якщо потрібен **потік даних** — послідовність значень, що надходять асинхронно? Наприклад, читання з мережевого сокета, де дані приходять порціями? Або періодичні показання сенсорів?

У **Розділі 40: Streams — Async ітератори** ви:
- Дізнаєтесь, що таке Stream та як він співвідноситься з Iterator
- Навчитесь обробляти потоки даних через `while let`
- Опануєте комбінатори: map, filter, take, merge
- Створите stream з показаннями сенсорів агента

---

> **Наступний розділ:** [Розділ 40: Streams — Async ітератори](./40_Streams.md)
