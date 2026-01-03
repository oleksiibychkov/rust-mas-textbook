## Структурно-логічна схема вивчення дисципліни «Програмування мультиагентних систем»


### **Блон Всуп: Основи мови Rust**

| № | Тема лекції | Ключові поняття | Розділи підручника | Лекція |
|---|-------------|-----------------|-------------------|---------|
| 1 | **Інструменти. Основи синтаксису** | `Cargo`, `let`, `if`, `loop`, `while` | 1-4 | [Переглянути](#лекція-1)
| 2 | **Структури даних у Rust** | `[T; N]`, `(T1, T2, ...)`, `&[T]`, `struct` | 5 |
| 3 | **Система володіння — серце Rust** | `Move`, `Copy`, `Clone`, `Drop`, `Borrowing` | 7 |
 
 
### Лекція 1
<iframe src="./lectures/1_Rust.pdf" width="100%" height="700px"></iframe>

### Лекція 2
<iframe src="./lectures/1_Rust.pdf" width="100%" height="700px"></iframe>

### Лекція 3
<iframe src="./lectures/1_Rust.pdf" width="100%" height="700px"></iframe>

### **Блок A: Колекції та обробка даних**

| № | Тема лекції | Ключові поняття | Розділи підручника |
|---|-------------|-----------------|-------------------|
| 4 | **Колекції: Vec та динамічні дані** | `Vec<T>`, `push`, `pop`, `iter()`, `into_iter()`, heap allocation | 16 |
| 5 | **Колекції: HashMap та HashSet** | `HashMap<K,V>`, `HashSet<T>`, `entry()`, `insert`, `get` | 17 |
| 6 | **Обробка помилок: Option та Result** | `Option<T>`, `Result<T,E>`, `?`, `unwrap`, `map`, `and_then` | 18-19 |

### Лекція 4
<iframe src="./lectures/1_Rust.pdf" width="100%" height="700px"></iframe>

### Лекція 5
<iframe src="./lectures/1_Rust.pdf" width="100%" height="700px"></iframe>

### Лекція 6
<iframe src="./lectures/1_Rust.pdf" width="100%" height="700px"></iframe>

### **Блок B: Трейти та узагальнення**

| № | Тема лекції | Ключові поняття | Розділи підручника |
|---|-------------|-----------------|-------------------|
| 7 | **Traits: Поліморфізм у Rust** | `trait`, `impl Trait for`, `derive`, `Display`, `Debug`, `Clone` | 22-23 |
| 8 | **Generics та Trait Bounds** | `<T>`, `where`, `impl Trait`, монорфізація | 24-25 |

### Лекція 7
<iframe src="./lectures/1_Rust.pdf" width="100%" height="700px"></iframe>

### Лекція 8
<iframe src="./lectures/1_Rust.pdf" width="100%" height="700px"></iframe>


### **Блок C: Smart Pointers**

| № | Тема лекції | Ключові поняття | Розділи підручника |
|---|-------------|-----------------|-------------------|
| 9 | **Box та Rc: Розумні вказівники** | `Box<T>`, `Rc<T>`, heap vs stack, shared ownership | 28-29 |
| 10 | **RefCell та Interior Mutability** | `RefCell<T>`, `Cell<T>`, runtime borrow checking | 30-31 |
| 11 | **Lifetimes: Час життя посилань** | `'a`, lifetime annotations, `'static` | 32 |

### Лекція 9
<iframe src="./lectures/1_Rust.pdf" width="100%" height="700px"></iframe>

### Лекція 10
<iframe src="./lectures/1_Rust.pdf" width="100%" height="700px"></iframe>

### Лекція 11
<iframe src="./lectures/1_Rust.pdf" width="100%" height="700px"></iframe>

### **Блок D: Багатопотоковість (ключовий для МАС)**

| № | Тема лекції | Ключові поняття | Розділи підручника |
|---|-------------|-----------------|-------------------|
| 12 | **Потоки: Базова багатопотоковість** | `std::thread`, `spawn`, `join`, `move` closures, `Send` trait | 33 |
| 13 | **Arc та Mutex: Спільний стан** | `Arc<T>`, `Mutex<T>`, `RwLock`, `MutexGuard`, deadlocks, `Sync` | 34 |
| 14 | **Channels: Message Passing** | `mpsc::channel`, `Sender`, `Receiver`, `send`, `recv`, `try_recv` | 35 |
| 15 | **Практикум: Локальний рій агентів** | Інтеграція потоків, Arc, Mutex, channels | 36 |

### Лекція 12
<iframe src="./lectures/1_Rust.pdf" width="100%" height="700px"></iframe>

### Лекція 13
<iframe src="./lectures/1_Rust.pdf" width="100%" height="700px"></iframe>

### Лекція 14
<iframe src="./lectures/1_Rust.pdf" width="100%" height="700px"></iframe>

### Лекція 15
<iframe src="./lectures/1_Rust.pdf" width="100%" height="700px"></iframe>


### **Блок E: Асинхронність (масштабування МАС)**

| № | Тема лекції | Ключові поняття | Розділи підручника |
|---|-------------|-----------------|-------------------|
| 16 | **Async/Await: Концепції** | `async fn`, `Future`, `.await`, state machines | 37 |
| 17 | **Tokio: Async Runtime** | `#[tokio::main]`, `tokio::spawn`, `select!`, таймери | 38 |
| 18 | **Async Channels та синхронізація** | `tokio::sync::mpsc`, `broadcast`, async `Mutex` | 39 |
| 19 | **Streams: Async ітератори** | `Stream`, `StreamExt`, `map`, `filter`, `merge` | 40 |

### Лекція 16
<iframe src="./lectures/1_Rust.pdf" width="100%" height="700px"></iframe>

### Лекція 17
<iframe src="./lectures/1_Rust.pdf" width="100%" height="700px"></iframe>

### Лекція 18
<iframe src="./lectures/1_Rust.pdf" width="100%" height="700px"></iframe>

### Лекція 19
<iframe src="./lectures/1_Rust.pdf" width="100%" height="700px"></iframe>

### **Блок F: Архітектури МАС**

| № | Тема лекції | Ключові поняття | Розділи підручника |
|---|-------------|-----------------|-------------------|
| 20 | **Actor Model: Концепції** | Актор, mailbox, supervision, location transparency | 41 |
| 21 | **Actor Model: Реалізація на Tokio** | Актор як task + channel, graceful shutdown, oneshot | 42 |
| 22 | **Практикум: Асинхронний рій** | Інтеграція Tokio, actors, async patterns | 44 |
| 23 | **ECS: Entity-Component-System** | Entity, Component, System, `bevy_ecs`, queries | 48 |
| 24 | **BDI: Beliefs-Desires-Intentions** | BDI цикл, планування, goal-directed behavior | 49 |

### Лекція 20
<iframe src="./lectures/1_Rust.pdf" width="100%" height="700px"></iframe>

### Лекція 21
<iframe src="./lectures/1_Rust.pdf" width="100%" height="700px"></iframe>

### Лекція 22
<iframe src="./lectures/1_Rust.pdf" width="100%" height="700px"></iframe>

### Лекція 23
<iframe src="./lectures/1_Rust.pdf" width="100%" height="700px"></iframe>

### Лекція 24
<iframe src="./lectures/1_Rust.pdf" width="100%" height="700px"></iframe>

## Візуалізація вивчення МАС

```
            Лекція 0 МАС (мотиваційна)
                     │
                     ▼
    ┌────────────────────────────────────┐
    │  Блок A: Колекції (Л.4-6)          │
    │  Vec, HashMap, Option, Result      │
    └────────────────┬───────────────────┘
                     ▼
    ┌────────────────────────────────────┐
    │  Блок B: Трейти (Л.7-8)            │
    │  trait, generics, bounds           │
    └────────────────┬───────────────────┘
                     ▼
    ┌────────────────────────────────────┐
    │  Блок C: Smart Pointers (Л.9-11)   │
    │  Box, Rc, RefCell, Lifetimes       │
    └────────────────┬───────────────────┘
                     ▼
    ┌────────────────────────────────────┐
    │  Блок D: Потоки (Л.12-15)          │  
    │  thread, Arc, Mutex, channels      │  
    └────────────────┬───────────────────┘
                     ▼
    ┌────────────────────────────────────┐
    │  Блок E: Async (Л.16-19)           │  
    │  async/await, Tokio, Streams       │  
    └────────────────┬───────────────────┘
                     ▼
    ┌────────────────────────────────────┐
    │  Блок F: Архітектури (Л.20-24)     │  
    │  Actor, ECS, BDI                   │  
    └────────────────┬───────────────────┘
                     ▼
         Фінальний проєкт: Рій БПЛА
```

---

