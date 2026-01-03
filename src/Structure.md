## Структурно-логічна схема вивчення дисципліни «Програмування мультиагентних систем»


### **Блон Всуп: Основи мови Rust**

| № | Тема лекції | Ключові поняття | Розділи підручника | Лекція |
|---|-------------|-----------------|-------------------|---------|
| 1 | **Інструменти. Основи синтаксису** | `Cargo`, `let`, `if`, `loop`, `while` | 1-4 |  <a href="./lectures/1.pdf" target="_blank">Переглянути</a>|
| 2 | **Структури даних у Rust** | `[T; N]`, `(T1, T2, ...)`, `&[T]`, `struct` | 5 | <a href="./lectures/2.pdf" target="_blank">Переглянути</a>|
| 3 | **Система володіння — серце Rust** | `Move`, `Copy`, `Clone`, `Drop`, `Borrowing` | 7 | <a href="./lectures/3.pdf" target="_blank">Переглянути</a>|
 
### **Блок A: Колекції та обробка даних**

| № | Тема лекції | Ключові поняття | Розділи підручника | Лекція |
|---|-------------|-----------------|-------------------|---------|
| 4 | **Колекції: Vec та динамічні дані** | `Vec<T>`, `push`, `pop`, `iter()`, `into_iter()`, heap allocation | 16 | <a href="./lectures/4.pdf" target="_blank">Переглянути</a>|
| 5 | **Колекції: HashMap та HashSet** | `HashMap<K,V>`, `HashSet<T>`, `entry()`, `insert`, `get` | 17 | <a href="./lectures/5.pdf" target="_blank">Переглянути</a>|
| 6 | **Обробка помилок: Option та Result** | `Option<T>`, `Result<T,E>`, `?`, `unwrap`, `map`, `and_then` | 18-19 | <a href="./lectures/6.pdf" target="_blank">Переглянути</a>|

### **Блок B: Трейти та узагальнення**

| № | Тема лекції | Ключові поняття | Розділи підручника | Лекція |
|---|-------------|-----------------|-------------------|---------|
| 7 | **Traits: Поліморфізм у Rust** | `trait`, `impl Trait for`, `derive`, `Display`, `Debug`, `Clone` | 22-23 | <a href="./lectures/7.pdf" target="_blank">Переглянути</a>|
| 8 | **Generics та Trait Bounds** | `<T>`, `where`, `impl Trait`, монорфізація | 24-25 |<a href="./lectures/8.pdf" target="_blank">Переглянути</a>|

### **Блок C: Smart Pointers**

| № | Тема лекції | Ключові поняття | Розділи підручника | Лекція |
|---|-------------|-----------------|-------------------|---------|
| 9 | **Box та Rc: Розумні вказівники** | `Box<T>`, `Rc<T>`, heap vs stack, shared ownership | 28-29 |<a href="./lectures/9.pdf" target="_blank">Переглянути</a>|
| 10 | **RefCell та Interior Mutability** | `RefCell<T>`, `Cell<T>`, runtime borrow checking | 30-31 |<a href="./lectures/10.pdf" target="_blank">Переглянути</a>|
| 11 | **Lifetimes: Час життя посилань** | `'a`, lifetime annotations, `'static` | 32 |<a href="./lectures/11.pdf" target="_blank">Переглянути</a>|

### **Блок D: Багатопотоковість (ключовий для МАС)**

| № | Тема лекції | Ключові поняття | Розділи підручника | Лекція |
|---|-------------|-----------------|-------------------|---------|
| 12 | **Потоки: Базова багатопотоковість** | `std::thread`, `spawn`, `join`, `move` closures, `Send` trait | 33 |<a href="./lectures/12.pdf" target="_blank">Переглянути</a>|
| 13 | **Arc та Mutex: Спільний стан** | `Arc<T>`, `Mutex<T>`, `RwLock`, `MutexGuard`, deadlocks, `Sync` | 34 |<a href="./lectures/13.pdf" target="_blank">Переглянути</a>|
| 14 | **Channels: Message Passing** | `mpsc::channel`, `Sender`, `Receiver`, `send`, `recv`, `try_recv` | 35 |<a href="./lectures/14.pdf" target="_blank">Переглянути</a>|
| 15 | **Практикум: Локальний рій агентів** | Інтеграція потоків, Arc, Mutex, channels | 36 |<a href="./lectures/15.pdf" target="_blank">Переглянути</a>|

### **Блок E: Асинхронність (масштабування МАС)**

| № | Тема лекції | Ключові поняття | Розділи підручника | Лекція |
|---|-------------|-----------------|-------------------|---------|
| 16 | **Async/Await: Концепції** | `async fn`, `Future`, `.await`, state machines | 37 |<a href="./lectures/16.pdf" target="_blank">Переглянути</a>|
| 17 | **Tokio: Async Runtime** | `#[tokio::main]`, `tokio::spawn`, `select!`, таймери | 38 |<a href="./lectures/17.pdf" target="_blank">Переглянути</a>|
| 18 | **Async Channels та синхронізація** | `tokio::sync::mpsc`, `broadcast`, async `Mutex` | 39 |<a href="./lectures/18.pdf" target="_blank">Переглянути</a>|
| 19 | **Streams: Async ітератори** | `Stream`, `StreamExt`, `map`, `filter`, `merge` | 40 |<a href="./lectures/19.pdf" target="_blank">Переглянути</a>|

### **Блок F: Архітектури МАС**

| № | Тема лекції | Ключові поняття | Розділи підручника | Лекція |
|---|-------------|-----------------|-------------------|---------|
| 20 | **Actor Model: Концепції** | Актор, mailbox, supervision, location transparency | 41 |<a href="./lectures/20.pdf" target="_blank">Переглянути</a>|
| 21 | **Actor Model: Реалізація на Tokio** | Актор як task + channel, graceful shutdown, oneshot | 42 |<a href="./lectures/21.pdf" target="_blank">Переглянути</a>|
| 22 | **Практикум: Асинхронний рій** | Інтеграція Tokio, actors, async patterns | 44 |<a href="./lectures/22.pdf" target="_blank">Переглянути</a>|
| 23 | **ECS: Entity-Component-System** | Entity, Component, System, `bevy_ecs`, queries | 48 |<a href="./lectures/23.pdf" target="_blank">Переглянути</a>|
| 24 | **BDI: Beliefs-Desires-Intentions** | BDI цикл, планування, goal-directed behavior | 49 |<a href="./lectures/24.pdf" target="_blank">Переглянути</a>|

### Фінальний проєкт: Рій БПЛА  <a href="./lectures/Final_MAS.pdf" target="_blank">Переглянути</a>

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

