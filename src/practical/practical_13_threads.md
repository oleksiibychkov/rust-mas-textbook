# Практичне заняття 13: Багатопотоковість (Threads, Arc, Mutex)

## Мета заняття

Після цього заняття ви зможете:
- Створювати потоки та передавати в них дані
- Використовувати `Arc<T>` для спільного володіння між потоками
- Застосовувати `Mutex<T>` для безпечного мутабельного доступу
- Уникати типових проблем: deadlocks, data races
- Використовувати канали (channels) для комунікації між потоками

---

## Теоретичний вступ

### Чому Rust безпечний для багатопотоковості?

У багатьох мовах помилки паралелізму виявляються лише під час виконання:
- **Data race** — одночасний доступ до даних без синхронізації
- **Deadlock** — взаємне блокування потоків

Rust запобігає **data races на етапі компіляції** через систему ownership:

```rust
// Data race НЕМОЖЛИВИЙ у safe Rust
let mut data = vec![1, 2, 3];
std::thread::spawn(|| {
    data.push(4);  // ПОМИЛКА: не можна захопити &mut через потоки
});
```

### Основні примітиви

| Примітив | Призначення |
|----------|-------------|
| `std::thread::spawn` | Створення потоку |
| `JoinHandle` | Очікування завершення потоку |
| `Arc<T>` | Atomic Reference Counting — спільне володіння |
| `Mutex<T>` | Взаємне виключення — один потік одночасно |
| `RwLock<T>` | Багато читачів АБО один писач |
| `mpsc::channel` | Канал для передачі повідомлень |

---

## Створення потоків

### Базовий spawn

```rust
use std::thread;
use std::time::Duration;

fn main() {
    // Створюємо новий потік
    let handle = thread::spawn(|| {
        for i in 1..=5 {
            println!("Потік: {}", i);
            thread::sleep(Duration::from_millis(100));
        }
    });
    
    // Головний потік продовжує роботу
    for i in 1..=3 {
        println!("Головний: {}", i);
        thread::sleep(Duration::from_millis(150));
    }
    
    // Чекаємо завершення дочірнього потоку
    handle.join().unwrap();
    println!("Обидва потоки завершились");
}
```

### Move closure — передача даних у потік

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3];
    
    // move переміщує ownership у потік
    let handle = thread::spawn(move || {
        println!("Дані в потоці: {:?}", data);
        data.iter().sum::<i32>()
    });
    
    // data більше недоступна тут!
    // println!("{:?}", data);  // ПОМИЛКА
    
    let result = handle.join().unwrap();
    println!("Сума: {}", result);
}
```

### Кілька потоків

```rust
use std::thread;

fn main() {
    let mut handles = vec![];
    
    for i in 0..5 {
        let handle = thread::spawn(move || {
            println!("Потік {} стартував", i);
            thread::sleep(std::time::Duration::from_millis(100 * i as u64));
            println!("Потік {} завершився", i);
            i * 10
        });
        handles.push(handle);
    }
    
    // Збираємо результати
    let results: Vec<i32> = handles
        .into_iter()
        .map(|h| h.join().unwrap())
        .collect();
    
    println!("Результати: {:?}", results);
}
```

---

## Arc — Atomic Reference Counting

### Проблема: спільні дані

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3];
    
    // ПОМИЛКА: не можна перемістити data в кілька потоків
    let h1 = thread::spawn(move || println!("{:?}", data));
    let h2 = thread::spawn(move || println!("{:?}", data));  // data вже moved!
}
```

### Рішення: Arc

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let data = Arc::new(vec![1, 2, 3]);
    
    let data1 = Arc::clone(&data);  // Збільшуємо лічильник
    let data2 = Arc::clone(&data);
    
    let h1 = thread::spawn(move || {
        println!("Потік 1: {:?}", data1);
    });
    
    let h2 = thread::spawn(move || {
        println!("Потік 2: {:?}", data2);
    });
    
    h1.join().unwrap();
    h2.join().unwrap();
    
    println!("Головний: {:?}", data);
}
```

### Arc vs Rc

| `Rc<T>` | `Arc<T>` |
|---------|----------|
| НЕ thread-safe | Thread-safe |
| Швидший (не atomic) | Повільніший (atomic operations) |
| Для single-threaded | Для multi-threaded |
| НЕ реалізує `Send` | Реалізує `Send` |

---

## Mutex — Mutual Exclusion

### Проблема: мутабельний спільний доступ

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let counter = Arc::new(0);  // Просто число
    
    let counter1 = Arc::clone(&counter);
    let h1 = thread::spawn(move || {
        // *counter1 += 1;  // ПОМИЛКА! Arc дає тільки read-only доступ
    });
}
```

### Рішення: Arc<Mutex<T>>

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];
    
    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            // lock() блокує до отримання доступу
            let mut num = counter.lock().unwrap();
            *num += 1;
            // MutexGuard автоматично розблоковується при drop
        });
        handles.push(handle);
    }
    
    for handle in handles {
        handle.join().unwrap();
    }
    
    println!("Результат: {}", *counter.lock().unwrap());
}
```

### Як працює Mutex

```rust
use std::sync::Mutex;

fn main() {
    let m = Mutex::new(5);
    
    {
        // lock() повертає MutexGuard
        let mut guard = m.lock().unwrap();
        *guard = 10;
        println!("Inside lock: {}", *guard);
        // guard dropped тут — mutex розблоковано
    }
    
    // Можна знову заблокувати
    let guard = m.lock().unwrap();
    println!("After unlock: {}", *guard);
}
```

### try_lock — неблокуючий доступ

```rust
use std::sync::Mutex;

fn main() {
    let m = Mutex::new(5);
    
    {
        let _guard = m.lock().unwrap();
        
        // try_lock не блокує — повертає Err якщо зайнято
        match m.try_lock() {
            Ok(guard) => println!("Got lock: {}", *guard),
            Err(_) => println!("Mutex is locked"),
        }
    }
    
    // Тепер mutex вільний
    match m.try_lock() {
        Ok(guard) => println!("Got lock: {}", *guard),
        Err(_) => println!("Mutex is locked"),
    }
}
```

---

## RwLock — Read-Write Lock

```rust
use std::sync::{Arc, RwLock};
use std::thread;

fn main() {
    let data = Arc::new(RwLock::new(vec![1, 2, 3]));
    let mut handles = vec![];
    
    // Кілька читачів одночасно
    for i in 0..3 {
        let data = Arc::clone(&data);
        handles.push(thread::spawn(move || {
            let reader = data.read().unwrap();
            println!("Reader {}: {:?}", i, *reader);
        }));
    }
    
    // Один писач (блокує всіх)
    {
        let data = Arc::clone(&data);
        handles.push(thread::spawn(move || {
            let mut writer = data.write().unwrap();
            writer.push(4);
            println!("Writer: {:?}", *writer);
        }));
    }
    
    for h in handles {
        h.join().unwrap();
    }
    
    println!("Final: {:?}", *data.read().unwrap());
}
```

---

## Channels — Message Passing

### mpsc: Multiple Producer, Single Consumer

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    // Створюємо канал
    let (tx, rx) = mpsc::channel();
    
    thread::spawn(move || {
        tx.send("Hello from thread!").unwrap();
        tx.send("Another message").unwrap();
    });
    
    // recv() блокує до отримання
    println!("Received: {}", rx.recv().unwrap());
    println!("Received: {}", rx.recv().unwrap());
}
```

### Кілька відправників

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();
    
    for i in 0..3 {
        let tx = tx.clone();  // Клонуємо sender
        thread::spawn(move || {
            tx.send(format!("Message from thread {}", i)).unwrap();
        });
    }
    
    drop(tx);  // Закриваємо оригінальний sender
    
    // Отримуємо всі повідомлення
    for msg in rx {
        println!("Received: {}", msg);
    }
}
```

### Bounded channel (sync_channel)

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    // Буфер на 2 повідомлення
    let (tx, rx) = mpsc::sync_channel(2);
    
    thread::spawn(move || {
        for i in 0..5 {
            println!("Sending {}", i);
            tx.send(i).unwrap();  // Блокується якщо буфер повний
            println!("Sent {}", i);
        }
    });
    
    thread::sleep(std::time::Duration::from_millis(500));
    
    for msg in rx {
        println!("Received: {}", msg);
        thread::sleep(std::time::Duration::from_millis(100));
    }
}
```

---

## Типові проблеми та рішення

### Deadlock

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let a = Arc::new(Mutex::new(1));
    let b = Arc::new(Mutex::new(2));
    
    let a1 = Arc::clone(&a);
    let b1 = Arc::clone(&b);
    
    let h1 = thread::spawn(move || {
        let _a = a1.lock().unwrap();
        thread::sleep(std::time::Duration::from_millis(100));
        let _b = b1.lock().unwrap();  // Чекає на b, тримаючи a
        println!("Thread 1");
    });
    
    let a2 = Arc::clone(&a);
    let b2 = Arc::clone(&b);
    
    let h2 = thread::spawn(move || {
        let _b = b2.lock().unwrap();
        thread::sleep(std::time::Duration::from_millis(100));
        let _a = a2.lock().unwrap();  // Чекає на a, тримаючи b
        println!("Thread 2");
    });
    
    // DEADLOCK! Обидва потоки чекають один на одного
    // h1.join().unwrap();
    // h2.join().unwrap();
}
```

**Рішення:** Завжди блокувати в однаковому порядку.

### Poison — mutex після паніки

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let data = Arc::new(Mutex::new(vec![1, 2, 3]));
    
    let data2 = Arc::clone(&data);
    let h = thread::spawn(move || {
        let mut guard = data2.lock().unwrap();
        guard.push(4);
        panic!("Oops!");  // Mutex стає "отруєним"
    });
    
    let _ = h.join();  // Ігноруємо паніку
    
    // Mutex отруєний, але дані можна відновити
    match data.lock() {
        Ok(guard) => println!("Data: {:?}", *guard),
        Err(poisoned) => {
            println!("Mutex was poisoned, recovering...");
            let guard = poisoned.into_inner();
            println!("Recovered data: {:?}", *guard);
        }
    }
}
```

---

## Практичні задачі

### Задача 1: Паралельний пошук

**Умова:** Реалізуйте паралельний пошук елемента в масиві. Розділіть масив на частини та шукайте в кожній паралельно.

**Розв'язання:**

```rust
use std::sync::Arc;
use std::thread;

fn parallel_find<T>(data: &[T], target: &T, num_threads: usize) -> Option<usize>
where
    T: PartialEq + Send + Sync,
{
    let data = Arc::new(data.to_vec());
    let target = Arc::new(target.clone());
    let chunk_size = (data.len() + num_threads - 1) / num_threads;
    
    let mut handles = vec![];
    
    for i in 0..num_threads {
        let data = Arc::clone(&data);
        let target = Arc::clone(&target);
        let start = i * chunk_size;
        
        let handle = thread::spawn(move || {
            let end = (start + chunk_size).min(data.len());
            for j in start..end {
                if data[j] == *target {
                    return Some(j);
                }
            }
            None
        });
        
        handles.push(handle);
    }
    
    for handle in handles {
        if let Some(idx) = handle.join().unwrap() {
            return Some(idx);
        }
    }
    
    None
}

fn parallel_find_all<T>(data: &[T], predicate: fn(&T) -> bool, num_threads: usize) -> Vec<usize>
where
    T: Send + Sync + Clone,
{
    let data = Arc::new(data.to_vec());
    let chunk_size = (data.len() + num_threads - 1) / num_threads;
    
    let mut handles = vec![];
    
    for i in 0..num_threads {
        let data = Arc::clone(&data);
        let start = i * chunk_size;
        
        let handle = thread::spawn(move || {
            let end = (start + chunk_size).min(data.len());
            let mut results = vec![];
            for j in start..end {
                if predicate(&data[j]) {
                    results.push(j);
                }
            }
            results
        });
        
        handles.push(handle);
    }
    
    let mut all_results = vec![];
    for handle in handles {
        all_results.extend(handle.join().unwrap());
    }
    all_results.sort();
    all_results
}

fn main() {
    println!("=== Паралельний пошук ===\n");
    
    let data: Vec<i32> = (0..1000).collect();
    
    // Пошук одного елемента
    println!("Searching for 500...");
    if let Some(idx) = parallel_find(&data, &500, 4) {
        println!("Found at index: {}", idx);
    }
    
    println!("\nSearching for 9999...");
    match parallel_find(&data, &9999, 4) {
        Some(idx) => println!("Found at index: {}", idx),
        None => println!("Not found"),
    }
    
    // Пошук всіх елементів за умовою
    println!("\n--- Find all multiples of 100 ---");
    let multiples = parallel_find_all(&data, |x| x % 100 == 0, 4);
    println!("Found {} elements: {:?}", multiples.len(), multiples);
    
    // Пошук парних чисел більших за 990
    println!("\n--- Even numbers > 990 ---");
    let large_evens = parallel_find_all(&data, |x| *x > 990 && x % 2 == 0, 4);
    println!("Found: {:?}", large_evens);
}
```

---

### Задача 2: Спільний лічильник

**Умова:** Створіть систему з кількома "воркерами", що паралельно обробляють завдання та оновлюють спільні лічильники (оброблено, помилок, в процесі).

**Розв'язання:**

```rust
use std::sync::{Arc, Mutex};
use std::thread;
use std::time::Duration;

#[derive(Debug, Default)]
struct Stats {
    processed: u32,
    errors: u32,
    in_progress: u32,
}

struct SharedStats {
    stats: Mutex<Stats>,
}

impl SharedStats {
    fn new() -> Self {
        SharedStats {
            stats: Mutex::new(Stats::default()),
        }
    }
    
    fn start_task(&self) {
        let mut stats = self.stats.lock().unwrap();
        stats.in_progress += 1;
    }
    
    fn complete_task(&self, success: bool) {
        let mut stats = self.stats.lock().unwrap();
        stats.in_progress -= 1;
        if success {
            stats.processed += 1;
        } else {
            stats.errors += 1;
        }
    }
    
    fn get_stats(&self) -> Stats {
        let stats = self.stats.lock().unwrap();
        Stats {
            processed: stats.processed,
            errors: stats.errors,
            in_progress: stats.in_progress,
        }
    }
}

fn process_task(task_id: u32) -> bool {
    // Симуляція обробки
    thread::sleep(Duration::from_millis(50 + (task_id % 100) as u64));
    // 20% завдань "провалюються"
    task_id % 5 != 0
}

fn main() {
    println!("=== Спільний лічильник ===\n");
    
    let stats = Arc::new(SharedStats::new());
    let tasks: Vec<u32> = (1..=20).collect();
    let num_workers = 4;
    let chunk_size = (tasks.len() + num_workers - 1) / num_workers;
    
    let mut handles = vec![];
    
    for worker_id in 0..num_workers {
        let stats = Arc::clone(&stats);
        let start = worker_id * chunk_size;
        let worker_tasks: Vec<u32> = tasks[start..]
            .iter()
            .take(chunk_size)
            .copied()
            .collect();
        
        let handle = thread::spawn(move || {
            println!("Worker {} started with {} tasks", worker_id, worker_tasks.len());
            
            for task_id in worker_tasks {
                stats.start_task();
                let success = process_task(task_id);
                stats.complete_task(success);
                
                let current = stats.get_stats();
                println!(
                    "  Worker {}: Task {} {} (done: {}, errors: {}, active: {})",
                    worker_id,
                    task_id,
                    if success { "✓" } else { "✗" },
                    current.processed,
                    current.errors,
                    current.in_progress
                );
            }
            
            println!("Worker {} finished", worker_id);
        });
        
        handles.push(handle);
    }
    
    // Моніторинг прогресу
    let stats_monitor = Arc::clone(&stats);
    let monitor = thread::spawn(move || {
        loop {
            thread::sleep(Duration::from_millis(200));
            let current = stats_monitor.get_stats();
            println!(
                "\n📊 Progress: {} done, {} errors, {} active\n",
                current.processed, current.errors, current.in_progress
            );
            
            if current.in_progress == 0 && (current.processed + current.errors) > 0 {
                break;
            }
        }
    });
    
    for handle in handles {
        handle.join().unwrap();
    }
    
    monitor.join().unwrap();
    
    let final_stats = stats.get_stats();
    println!("\n=== Final Stats ===");
    println!("Processed: {}", final_stats.processed);
    println!("Errors: {}", final_stats.errors);
    println!("Success rate: {:.1}%", 
             final_stats.processed as f64 / (final_stats.processed + final_stats.errors) as f64 * 100.0);
}
```

---

### Задача 3: Producer-Consumer з каналами

**Умова:** Реалізуйте патерн Producer-Consumer: кілька продюсерів генерують завдання, кілька консьюмерів їх обробляють. Використовуйте канали для комунікації.

**Розв'язання:**

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

#[derive(Debug)]
enum Task {
    Process(u32),
    Shutdown,
}

#[derive(Debug)]
struct Result {
    task_id: u32,
    worker_id: u32,
    output: String,
}

fn main() {
    println!("=== Producer-Consumer ===\n");
    
    let (task_tx, task_rx) = mpsc::channel::<Task>();
    let (result_tx, result_rx) = mpsc::channel::<Result>();
    
    let num_producers = 2;
    let num_consumers = 3;
    let tasks_per_producer = 5;
    
    // Запускаємо консьюмерів
    let task_rx = std::sync::Arc::new(std::sync::Mutex::new(task_rx));
    let mut consumer_handles = vec![];
    
    for worker_id in 0..num_consumers {
        let task_rx = std::sync::Arc::clone(&task_rx);
        let result_tx = result_tx.clone();
        
        let handle = thread::spawn(move || {
            loop {
                let task = {
                    let rx = task_rx.lock().unwrap();
                    rx.recv()
                };
                
                match task {
                    Ok(Task::Process(task_id)) => {
                        // Симуляція обробки
                        thread::sleep(Duration::from_millis(100));
                        
                        let result = Result {
                            task_id,
                            worker_id,
                            output: format!("Processed task {} by worker {}", task_id, worker_id),
                        };
                        
                        result_tx.send(result).unwrap();
                    }
                    Ok(Task::Shutdown) | Err(_) => {
                        println!("Consumer {} shutting down", worker_id);
                        break;
                    }
                }
            }
        });
        
        consumer_handles.push(handle);
    }
    
    // Запускаємо продюсерів
    let mut producer_handles = vec![];
    
    for producer_id in 0..num_producers {
        let task_tx = task_tx.clone();
        
        let handle = thread::spawn(move || {
            for i in 0..tasks_per_producer {
                let task_id = producer_id * 100 + i;
                println!("Producer {} created task {}", producer_id, task_id);
                task_tx.send(Task::Process(task_id)).unwrap();
                thread::sleep(Duration::from_millis(50));
            }
            println!("Producer {} finished", producer_id);
        });
        
        producer_handles.push(handle);
    }
    
    // Чекаємо завершення продюсерів
    for handle in producer_handles {
        handle.join().unwrap();
    }
    
    // Відправляємо сигнали завершення
    for _ in 0..num_consumers {
        task_tx.send(Task::Shutdown).unwrap();
    }
    
    drop(task_tx);  // Закриваємо канал
    
    // Збираємо результати
    drop(result_tx);  // Закриваємо наш sender
    
    println!("\n--- Results ---");
    let mut results = vec![];
    for result in result_rx {
        println!("  {:?}", result);
        results.push(result);
    }
    
    // Чекаємо завершення консьюмерів
    for handle in consumer_handles {
        handle.join().unwrap();
    }
    
    println!("\n=== Summary ===");
    println!("Total tasks processed: {}", results.len());
    
    // Статистика по воркерах
    for worker_id in 0..num_consumers {
        let count = results.iter().filter(|r| r.worker_id == worker_id).count();
        println!("Worker {} processed {} tasks", worker_id, count);
    }
}
```

---

### Задача 4: Thread-safe кеш

**Умова:** Реалізуйте thread-safe LRU кеш з використанням `RwLock`. Читання не блокує інших читачів, запис блокує всіх.

**Розв'язання:**

```rust
use std::collections::HashMap;
use std::sync::{Arc, RwLock};
use std::thread;
use std::time::Duration;

struct LruCache<K, V> {
    data: RwLock<HashMap<K, V>>,
    order: RwLock<Vec<K>>,
    capacity: usize,
}

impl<K, V> LruCache<K, V>
where
    K: std::hash::Hash + Eq + Clone + std::fmt::Debug,
    V: Clone + std::fmt::Debug,
{
    fn new(capacity: usize) -> Self {
        LruCache {
            data: RwLock::new(HashMap::new()),
            order: RwLock::new(Vec::new()),
            capacity,
        }
    }
    
    fn get(&self, key: &K) -> Option<V> {
        // Читання — не блокує інших читачів
        let data = self.data.read().unwrap();
        if let Some(value) = data.get(key) {
            // Оновлюємо порядок (потребує write lock)
            drop(data);
            let mut order = self.order.write().unwrap();
            order.retain(|k| k != key);
            order.push(key.clone());
            
            let data = self.data.read().unwrap();
            return data.get(key).cloned();
        }
        None
    }
    
    fn put(&self, key: K, value: V) {
        let mut data = self.data.write().unwrap();
        let mut order = self.order.write().unwrap();
        
        // Видаляємо найстаріший якщо переповнення
        if data.len() >= self.capacity && !data.contains_key(&key) {
            if let Some(oldest) = order.first().cloned() {
                data.remove(&oldest);
                order.remove(0);
            }
        }
        
        order.retain(|k| k != &key);
        order.push(key.clone());
        data.insert(key, value);
    }
    
    fn remove(&self, key: &K) -> Option<V> {
        let mut data = self.data.write().unwrap();
        let mut order = self.order.write().unwrap();
        
        order.retain(|k| k != key);
        data.remove(key)
    }
    
    fn len(&self) -> usize {
        self.data.read().unwrap().len()
    }
    
    fn keys(&self) -> Vec<K> {
        self.order.read().unwrap().clone()
    }
}

fn main() {
    println!("=== Thread-safe LRU Cache ===\n");
    
    let cache = Arc::new(LruCache::<String, i32>::new(3));
    let mut handles = vec![];
    
    // Потоки-писачі
    for i in 0..3 {
        let cache = Arc::clone(&cache);
        handles.push(thread::spawn(move || {
            for j in 0..5 {
                let key = format!("key-{}-{}", i, j);
                let value = i * 100 + j;
                cache.put(key.clone(), value);
                println!("Writer {}: put {}={}", i, key, value);
                thread::sleep(Duration::from_millis(50));
            }
        }));
    }
    
    // Потоки-читачі
    for i in 0..2 {
        let cache = Arc::clone(&cache);
        handles.push(thread::spawn(move || {
            thread::sleep(Duration::from_millis(25));
            for j in 0..10 {
                let key = format!("key-{}-{}", j % 3, j % 5);
                match cache.get(&key) {
                    Some(v) => println!("Reader {}: get {}={}", i, key, v),
                    None => println!("Reader {}: miss {}", i, key),
                }
                thread::sleep(Duration::from_millis(30));
            }
        }));
    }
    
    for handle in handles {
        handle.join().unwrap();
    }
    
    println!("\n=== Final Cache State ===");
    println!("Size: {}", cache.len());
    println!("Keys (LRU order): {:?}", cache.keys());
    
    // Перевірка значень
    for key in cache.keys() {
        if let Some(value) = cache.get(&key) {
            println!("  {} = {}", key, value);
        }
    }
}
```

---

## Домашнє завдання

### Завдання 1: Паралельний map

Реалізуйте функцію `parallel_map<T, U, F>(data: &[T], f: F, num_threads: usize) -> Vec<U>`, що застосовує функцію до кожного елемента паралельно.

### Завдання 2: Thread pool

Створіть простий пул потоків з фіксованою кількістю воркерів, що виконують завдання з черги.

### Завдання 3: Читачі-писачі з пріоритетами

Реалізуйте примітив синхронізації, де писачі мають пріоритет над читачами.

### Завдання 4: Distributed counter

Створіть "розподілений" лічильник, де кожен потік має локальний лічильник, а загальна сума обчислюється періодично.

---

## Підсумок заняття

На цьому занятті ви навчились:

1. **Створювати потоки**: `thread::spawn`, `JoinHandle`, `move`
2. **Спільно володіти даними**: `Arc<T>`
3. **Синхронізувати доступ**: `Mutex<T>`, `RwLock<T>`
4. **Передавати повідомлення**: `mpsc::channel`
5. **Уникати проблем**: deadlocks, poisoning

---

## Перевірте себе

1. Чим `Arc` відрізняється від `Rc`?
2. Що таке `MutexGuard`?
3. Коли використовувати `RwLock` замість `Mutex`?
4. Що таке "отруєний" mutex?
5. Як створити bounded channel?

**Відповіді:**
1. `Arc` — thread-safe (atomic), `Rc` — ні
2. RAII guard, що автоматично розблоковує mutex при drop
3. Коли багато читачів і мало писачів
4. Mutex, де потік запанікував тримаючи lock
5. `mpsc::sync_channel(capacity)`

---

## Наступне заняття

На наступному занятті ми вивчимо **Async/Await та Tokio** — як писати асинхронний код для ефективної обробки I/O без блокування потоків.
