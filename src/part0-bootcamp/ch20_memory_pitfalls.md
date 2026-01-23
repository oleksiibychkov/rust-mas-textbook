# Підступні задачі з пам'яттю та ресурсами

---

## 📋 Анотація

Цей розділ розкриває неочевидні аспекти керування пам'яттю та ресурсами, які можуть призвести до повільної деградації системи, витоків пам'яті та вичерпання ресурсів. Ви дізнаєтесь, чому Rust гарантує відсутність use-after-free, але не захищає від витоків пам'яті, як циклічні посилання через Rc обходять систему ownership, та чому закриття файлу може не відбутись навіть з RAII. У контексті рою БПЛА ці знання критичні: система керування працює безперервно, і навіть мінімальний витік пам'яті за годину перетвориться на гігабайти за тиждень польоту.

---

## 🎯 Цілі навчання

Після завершення цього розділу ви зможете:

1. **Пояснити** чому Rust дозволяє витоки пам'яті
2. **Розпізнавати** циклічні посилання через Rc/Arc
3. **Використовувати** Weak для розриву циклів
4. **Уникати** витоків через замикання та канали
5. **Розуміти** різницю між пам'яттю та іншими ресурсами
6. **Правильно звільняти** ресурси в складних сценаріях
7. **Профілювати** програми для виявлення витоків

---

## 📚 Ключові терміни

| Термін | Визначення |
|--------|------------|
| **витік пам'яті (memory leak)** | Пам'ять виділена, але недоступна і не звільняється |
| **Rc<T>** | Reference Counted — розумний вказівник зі спільним володінням |
| **Weak<T>** | Слабке посилання, що не запобігає звільненню |
| **циклічне посилання** | Структура, де A посилається на B, а B на A |
| **RAII** | Resource Acquisition Is Initialization — патерн автоматичного звільнення |
| **Drop** | Трейт деструктора в Rust |
| **mem::forget** | Функція, що запобігає виклику Drop |

---

## 💡 Мотиваційний кейс: Сервер, що "товстішав"

Команда розробляла сервер керування роєм дронів. Система працювала стабільно — жодних crash-ів, жодних помилок у логах. Але кожні кілька днів сервер потрібно було перезапускати, бо він споживав всю доступну пам'ять.

Профілювання показало дивну картину: мільйони об'єктів `DroneSession` залишались у пам'яті, хоча відповідні дрони давно відключились.

Причина була в архітектурі:

```rust
struct DroneSession {
    id: DroneId,
    swarm: Rc<RefCell<Swarm>>,  // Посилання на рій
    // ...
}

struct Swarm {
    drones: HashMap<DroneId, Rc<RefCell<DroneSession>>>,  // Посилання на сесії
    // ...
}
```

Класичний цикл: `DroneSession` → `Swarm` → `DroneSession`. Коли дрон відключався, його видаляли з HashMap. Але Swarm все ще мав Rc на сесію (через інші структури), а сесія мала Rc на Swarm. Reference count ніколи не досягав нуля.

Rust скомпілював цей код без застережень. Borrow checker не бачить проблеми — всі правила дотримані. Але пам'ять витікала.

Виправлення зайняло тиждень рефакторингу: заміна деяких Rc на Weak, перепроектування зв'язків між структурами, додавання явного "від'єднання" при закритті сесії.

---

## ТЕОРІЯ: ЧОМУ RUST ДОЗВОЛЯЄ ВИТОКИ ПАМ'ЯТІ

### Гарантії Rust

Rust гарантує memory safety — відсутність:
- Use-after-free (використання звільненої пам'яті)
- Double free (подвійне звільнення)
- Data races (гонки даних)
- Null pointer dereference (розіменування null)

Rust НЕ гарантує відсутність:
- Memory leaks (витоків пам'яті)
- Resource leaks (витоків ресурсів)
- Logical errors (логічних помилок)

### Чому витоки дозволені

Витік пам'яті — це не порушення memory safety. Пам'ять просто не звільняється, але ніхто не намагається її використати після звільнення. Це "безпечно" з точки зору undefined behavior.

Більше того, заборона витоків зробила б неможливими деякі корисні патерни:

```rust
// Box::leak — навмисний "витік" для статичних даних
let config: &'static Config = Box::leak(Box::new(load_config()));

// mem::forget — запобігання Drop для передачі ownership в C
let ptr = Box::into_raw(Box::new(data));
// Тепер C-код відповідає за звільнення
```

### Safe Rust може створювати витоки

Це важливо розуміти: витоки можливі без unsafe коду.

```rust
use std::rc::Rc;
use std::cell::RefCell;

// Повністю safe код, що створює витік
let a = Rc::new(RefCell::new(None));
let b = Rc::new(RefCell::new(None));

*a.borrow_mut() = Some(Rc::clone(&b));
*b.borrow_mut() = Some(Rc::clone(&a));

// a і b виходять зі scope, але пам'ять не звільняється!
// Reference count: a=1 (від b), b=1 (від a)
```

---

## ТЕОРІЯ: RC-ЦИКЛИ — ГОЛОВНЕ ДЖЕРЕЛО ВИТОКІВ

### Як працює Rc

`Rc<T>` (Reference Counted) дозволяє множинне володіння одними даними:

```rust
use std::rc::Rc;

let a = Rc::new(42);
let b = Rc::clone(&a);  // Не копіює дані, збільшує лічильник

println!("Count: {}", Rc::strong_count(&a));  // 2

drop(b);  // Лічильник зменшується до 1
drop(a);  // Лічильник = 0, пам'ять звільняється
```

Проблема виникає, коли лічильник ніколи не досягає нуля.

### Класичний цикл: двозв'язний список

```rust
use std::rc::Rc;
use std::cell::RefCell;

struct Node {
    value: i32,
    next: Option<Rc<RefCell<Node>>>,
    prev: Option<Rc<RefCell<Node>>>,  // Цикл!
}

fn create_cycle() {
    let a = Rc::new(RefCell::new(Node {
        value: 1,
        next: None,
        prev: None,
    }));
    
    let b = Rc::new(RefCell::new(Node {
        value: 2,
        next: None,
        prev: Some(Rc::clone(&a)),  // b → a
    }));
    
    a.borrow_mut().next = Some(Rc::clone(&b));  // a → b
    
    // Цикл: a → b → a
    // При виході зі scope:
    // - a має count=2 (локальна + від b.prev)
    // - b має count=2 (локальна + від a.next)
    // Після drop локальних:
    // - a має count=1 (від b.prev)
    // - b має count=1 (від a.next)
    // Ніколи не досягне 0!
}
```

### Менш очевидні цикли

**Parent-Child з зворотним посиланням**:

```rust
struct Parent {
    children: Vec<Rc<RefCell<Child>>>,
}

struct Child {
    parent: Rc<RefCell<Parent>>,  // Цикл!
}
```

**Observer pattern**:

```rust
struct Subject {
    observers: Vec<Rc<RefCell<dyn Observer>>>,
}

struct ConcreteObserver {
    subject: Rc<RefCell<Subject>>,  // Цикл!
}
```

**Graph структури**:

```rust
struct GraphNode {
    edges: Vec<Rc<RefCell<GraphNode>>>,  // Можливі цикли в графі
}
```

### Як виявити цикл

```rust
use std::rc::Rc;

fn check_for_leaks<T>(rc: &Rc<T>, expected_count: usize) {
    let actual = Rc::strong_count(rc);
    if actual != expected_count {
        eprintln!(
            "Warning: expected {} references, found {}. Possible cycle?",
            expected_count, actual
        );
    }
}
```

---

## ТЕОРІЯ: WEAK — РІШЕННЯ ДЛЯ ЦИКЛІВ

### Що таке Weak

`Weak<T>` — це "слабке" посилання, що не впливає на strong_count:

```rust
use std::rc::{Rc, Weak};

let strong = Rc::new(42);
let weak: Weak<i32> = Rc::downgrade(&strong);

println!("Strong: {}", Rc::strong_count(&strong));  // 1
println!("Weak: {}", Rc::weak_count(&strong));      // 1

// Weak не запобігає звільненню
drop(strong);

// Спроба отримати значення через weak
match weak.upgrade() {
    Some(rc) => println!("Value: {}", rc),
    None => println!("Value was dropped"),  // Цей варіант
}
```

### Розрив циклу через Weak

**Правило**: посилання "вниз" (parent → child) — Rc, посилання "вгору" (child → parent) — Weak.

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

struct Parent {
    children: Vec<Rc<RefCell<Child>>>,  // Strong: parent володіє children
}

struct Child {
    parent: Weak<RefCell<Parent>>,  // Weak: child не володіє parent
}

impl Child {
    fn get_parent(&self) -> Option<Rc<RefCell<Parent>>> {
        self.parent.upgrade()  // Може повернути None!
    }
}
```

### Двозв'язний список з Weak

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

struct Node {
    value: i32,
    next: Option<Rc<RefCell<Node>>>,   // Strong: володіємо наступним
    prev: Option<Weak<RefCell<Node>>>, // Weak: не володіємо попереднім
}

fn create_list() {
    let a = Rc::new(RefCell::new(Node {
        value: 1,
        next: None,
        prev: None,
    }));
    
    let b = Rc::new(RefCell::new(Node {
        value: 2,
        next: None,
        prev: Some(Rc::downgrade(&a)),  // Weak!
    }));
    
    a.borrow_mut().next = Some(Rc::clone(&b));
    
    // При виході:
    // - a: strong=1 (локальна), weak=1 (від b.prev)
    // - b: strong=2 (локальна + від a.next)
    // drop(b): b.strong=1
    // drop(a): a.strong=0 → звільняється → b.strong=0 → звільняється
    // Немає витоку!
}
```

### Observer з Weak

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

trait Observer {
    fn update(&self, value: i32);
}

struct Subject {
    observers: Vec<Weak<dyn Observer>>,  // Weak!
    value: i32,
}

impl Subject {
    fn notify(&mut self) {
        // Видаляємо "мертві" weak посилання
        self.observers.retain(|weak| weak.strong_count() > 0);
        
        for weak in &self.observers {
            if let Some(observer) = weak.upgrade() {
                observer.update(self.value);
            }
        }
    }
    
    fn subscribe(&mut self, observer: &Rc<dyn Observer>) {
        self.observers.push(Rc::downgrade(observer));
    }
}
```

---

## ТЕОРІЯ: ARC І ЦИКЛИ В БАГАТОПОТОКОВОМУ КОДІ

### Arc має ті ж проблеми

`Arc<T>` — це thread-safe версія Rc, але вразливість до циклів ідентична:

```rust
use std::sync::{Arc, Mutex, Weak};

struct ThreadSafeNode {
    value: i32,
    // Цикли можливі так само!
    neighbors: Vec<Arc<Mutex<ThreadSafeNode>>>,
}
```

### Arc<Mutex<T>> цикли особливо небезпечні

Цикли з Arc<Mutex<T>> не тільки витікають, але можуть спричинити deadlock при спробі звільнення:

```rust
use std::sync::{Arc, Mutex};

struct A {
    b: Option<Arc<Mutex<B>>>,
}

struct B {
    a: Option<Arc<Mutex<A>>>,
}

impl Drop for A {
    fn drop(&mut self) {
        if let Some(b) = &self.b {
            let _guard = b.lock().unwrap();  // Може deadlock!
        }
    }
}
```

### Рішення: Arc<T> + Weak<T>

```rust
use std::sync::{Arc, Weak, Mutex};

struct Worker {
    id: usize,
    pool: Weak<Mutex<WorkerPool>>,  // Weak до пулу
}

struct WorkerPool {
    workers: Vec<Arc<Worker>>,  // Strong до воркерів
}
```

---

## ТЕОРІЯ: ІНШІ ДЖЕРЕЛА ВИТОКІВ

### mem::forget

Функція `mem::forget` запобігає виклику Drop:

```rust
use std::mem;

let v = vec![1, 2, 3];
mem::forget(v);  // Drop не викликається, пам'ять не звільняється
```

Це safe функція! Використовується для:
- Передачі ownership в C-код
- Запобігання Drop у специфічних сценаріях

Але необережне використання створює витоки.

### Box::leak

```rust
let leaked: &'static str = Box::leak(String::from("hello").into_boxed_str());
// leaked живе "вічно" — це навмисний витік
```

### ManuallyDrop

```rust
use std::mem::ManuallyDrop;

let mut data = ManuallyDrop::new(vec![1, 2, 3]);
// Drop не викликається автоматично

// Потрібно явно:
// unsafe { ManuallyDrop::drop(&mut data); }
```

### Unbounded channels

```rust
use std::sync::mpsc;

let (tx, rx) = mpsc::channel();

// Producer швидший за consumer
loop {
    tx.send(large_data.clone()).unwrap();
    // Пам'ять зростає необмежено!
}
```

Використовуйте bounded channels:

```rust
let (tx, rx) = mpsc::sync_channel(1000);  // Максимум 1000 повідомлень
```

### Замикання, що захоплюють

```rust
let large_data = vec![0u8; 1_000_000];

let closure = move || {
    // large_data захоплено!
    println!("Hello");
};

// closure живе, large_data теж живе
// Якщо closure зберігається довго — large_data не звільняється
```

### Глобальний стан

```rust
use std::sync::Mutex;
use once_cell::sync::Lazy;

static CACHE: Lazy<Mutex<Vec<LargeData>>> = Lazy::new(|| Mutex::new(Vec::new()));

fn cache_data(data: LargeData) {
    CACHE.lock().unwrap().push(data);
    // Дані ніколи не видаляються!
}
```

---

## ТЕОРІЯ: РЕСУРСИ ≠ ПАМ'ЯТЬ

### Що таке ресурси

Окрім пам'яті, програма використовує інші обмежені ресурси:
- File descriptors (файли, сокети)
- Database connections
- Thread handles
- GPU buffers
- Locks (mutex guards)

### RAII: Resource Acquisition Is Initialization

Rust використовує RAII: ресурс звільняється при виході зі scope:

```rust
{
    let file = File::open("data.txt")?;
    // file descriptor відкритий
    
    // ... робота з файлом ...
    
}  // file виходить зі scope, Drop закриває descriptor
```

### Коли RAII не працює

**Цикли з ресурсами**:

```rust
struct Connection {
    socket: TcpStream,
    peer: Option<Rc<RefCell<Connection>>>,  // Цикл!
}
// TcpStream ніколи не закриється при циклі
```

**Паніка перед закриттям**:

```rust
fn process_file(path: &str) -> Result<(), Error> {
    let mut file = File::create(path)?;
    
    write!(file, "data")?;
    
    dangerous_operation()?;  // Якщо паніка — file.flush() не викликано!
    
    file.flush()?;
    Ok(())
}
```

**Забутий guard**:

```rust
let guard = mutex.lock().unwrap();
mem::forget(guard);  // Mutex залишається заблокованим назавжди!
```

### File descriptor exhaustion

```rust
fn bad_function() {
    for i in 0..100_000 {
        let file = File::open(format!("file_{}.txt", i)).unwrap();
        // file не використовується, але descriptor зайнятий
        // На багатьох системах ліміт ~1024 descriptors
        mem::forget(file);  // Витік!
    }
    // Error: Too many open files
}
```

---

## ТЕОРІЯ: DROP І ПОРЯДОК ЗВІЛЬНЕННЯ

### Порядок Drop у структурах

Поля структури звільняються в порядку оголошення:

```rust
struct Data {
    first: String,   // Drop викликається першим
    second: String,  // Drop викликається другим
}
```

### Порядок Drop локальних змінних

Локальні змінні звільняються у зворотному порядку створення (LIFO):

```rust
fn example() {
    let a = String::from("a");  // Створено першим
    let b = String::from("b");  // Створено другим
    let c = String::from("c");  // Створено третім
}
// Drop: c, потім b, потім a
```

### Коли порядок важливий

```rust
struct Logger {
    file: File,
}

struct App {
    logger: Logger,
    database: Database,  // Може логувати при закритті!
}

// Якщо database.drop() хоче логувати, а logger вже закритий?
```

Рішення: явний порядок через Option або ManuallyDrop:

```rust
struct App {
    database: Option<Database>,  // Закриваємо першим
    logger: Logger,
}

impl Drop for App {
    fn drop(&mut self) {
        // Явно закриваємо database поки logger ще живий
        if let Some(db) = self.database.take() {
            db.close();  // Може логувати
        }
        // Тепер logger закриється автоматично
    }
}
```

### Drop і паніка

Якщо Drop панікує під час розгортання іншої паніки — програма abort:

```rust
struct PanickyDrop;

impl Drop for PanickyDrop {
    fn drop(&mut self) {
        panic!("Panic in drop!");
    }
}

fn main() {
    let _p = PanickyDrop;
    panic!("Original panic");
    // Double panic → abort!
}
```

---

## ТЕОРІЯ: ПРОФІЛЮВАННЯ ТА ВИЯВЛЕННЯ ВИТОКІВ

### Valgrind (Linux)

```bash
valgrind --leak-check=full ./target/debug/my_program
```

Показує:
- Definitely lost: точні витоки
- Indirectly lost: витоки через втрачені вказівники
- Still reachable: пам'ять доступна, але не звільнена

### Heaptrack (Linux)

```bash
heaptrack ./target/debug/my_program
heaptrack_gui heaptrack.my_program.*.gz
```

Візуалізує:
- Графік використання пам'яті з часом
- Flamegraph алокацій
- Місця витоків

### Instruments (macOS)

Xcode Instruments → Leaks template

### Метрики в коді

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

static ALLOCATIONS: AtomicUsize = AtomicUsize::new(0);
static DEALLOCATIONS: AtomicUsize = AtomicUsize::new(0);

struct Tracked<T> {
    inner: T,
}

impl<T> Tracked<T> {
    fn new(value: T) -> Self {
        ALLOCATIONS.fetch_add(1, Ordering::Relaxed);
        Self { inner: value }
    }
}

impl<T> Drop for Tracked<T> {
    fn drop(&mut self) {
        DEALLOCATIONS.fetch_add(1, Ordering::Relaxed);
    }
}

fn check_leaks() {
    let allocs = ALLOCATIONS.load(Ordering::Relaxed);
    let deallocs = DEALLOCATIONS.load(Ordering::Relaxed);
    if allocs != deallocs {
        eprintln!("Leak detected: {} allocations, {} deallocations", allocs, deallocs);
    }
}
```

### Rc/Arc лічильники

```rust
use std::rc::Rc;

fn debug_rc<T>(name: &str, rc: &Rc<T>) {
    println!("{}: strong={}, weak={}", 
             name, 
             Rc::strong_count(rc), 
             Rc::weak_count(rc));
}
```

---

## ПРАКТИЧНІ РЕКОМЕНДАЦІЇ

### Правило 1: Уникайте Rc/Arc циклів — використовуйте Weak

```rust
// Погано: цикл
struct Node {
    parent: Option<Rc<Node>>,
    children: Vec<Rc<Node>>,
}

// Добре: Weak для зворотних посилань
struct Node {
    parent: Option<Weak<Node>>,  // Weak вгору
    children: Vec<Rc<Node>>,     // Strong вниз
}
```

### Правило 2: Перевіряйте strong_count у тестах

```rust
#[test]
fn test_no_leak() {
    let root = create_complex_structure();
    
    // Має бути лише одне посилання (наше)
    assert_eq!(Rc::strong_count(&root), 1);
    
    drop(root);
    // Якщо тест пройшов — циклів немає
}
```

### Правило 3: Bounded channels для backpressure

```rust
// Погано: unbounded
let (tx, rx) = mpsc::channel();

// Добре: bounded
let (tx, rx) = mpsc::sync_channel(1000);
```

### Правило 4: Явно очищайте глобальний стан

```rust
static CACHE: Lazy<Mutex<HashMap<Key, Value>>> = Lazy::new(|| Mutex::new(HashMap::new()));

fn clear_cache() {
    CACHE.lock().unwrap().clear();
}

// Періодично викликайте clear_cache() або використовуйте LRU cache
```

### Правило 5: Не зловживайте mem::forget та Box::leak

```rust
// Погано: leak без причини
let data = Box::leak(Box::new(compute_data()));

// Добре: явний lifetime
let data = Box::new(compute_data());
use_data(&data);
// data автоматично звільниться
```

### Правило 6: Явний Drop для важливих ресурсів

```rust
struct Connection {
    socket: TcpStream,
}

impl Connection {
    fn close(self) -> io::Result<()> {
        self.socket.shutdown(Shutdown::Both)?;
        Ok(())
        // Drop все одно викличеться
    }
}

// Використання
let conn = Connection::connect(addr)?;
// ... робота ...
conn.close()?;  // Явне закриття з обробкою помилок
```

---

## Застосування до рою БПЛА

### Архітектура без циклів

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

/// Рій володіє дронами (strong)
pub struct Swarm {
    id: SwarmId,
    drones: Vec<Rc<RefCell<Drone>>>,
    config: SwarmConfig,
}

/// Дрон має слабке посилання на рій (weak)
pub struct Drone {
    id: DroneId,
    swarm: Weak<RefCell<Swarm>>,  // Weak — немає циклу!
    state: DroneState,
}

impl Drone {
    pub fn get_swarm_config(&self) -> Option<SwarmConfig> {
        self.swarm.upgrade().map(|s| s.borrow().config.clone())
    }
    
    pub fn is_swarm_alive(&self) -> bool {
        self.swarm.strong_count() > 0
    }
}

impl Swarm {
    pub fn add_drone(&mut self, this: &Rc<RefCell<Self>>) -> Rc<RefCell<Drone>> {
        let drone = Rc::new(RefCell::new(Drone {
            id: DroneId::new(),
            swarm: Rc::downgrade(this),  // Weak посилання
            state: DroneState::default(),
        }));
        self.drones.push(Rc::clone(&drone));
        drone
    }
    
    pub fn remove_drone(&mut self, id: DroneId) {
        self.drones.retain(|d| d.borrow().id != id);
    }
}
```

### Обмежений буфер телеметрії

```rust
use std::collections::VecDeque;

pub struct TelemetryBuffer {
    buffer: VecDeque<TelemetryPoint>,
    max_size: usize,
}

impl TelemetryBuffer {
    pub fn new(max_size: usize) -> Self {
        Self {
            buffer: VecDeque::with_capacity(max_size),
            max_size,
        }
    }
    
    pub fn push(&mut self, point: TelemetryPoint) {
        if self.buffer.len() >= self.max_size {
            self.buffer.pop_front();  // Видаляємо найстаріше
        }
        self.buffer.push_back(point);
    }
    
    pub fn memory_usage(&self) -> usize {
        self.buffer.len() * std::mem::size_of::<TelemetryPoint>()
    }
}
```

### Кеш з автоматичним очищенням

```rust
use std::collections::HashMap;
use std::time::{Duration, Instant};

pub struct ExpiringCache<K, V> {
    entries: HashMap<K, (V, Instant)>,
    ttl: Duration,
    max_entries: usize,
}

impl<K: Eq + std::hash::Hash, V> ExpiringCache<K, V> {
    pub fn new(ttl: Duration, max_entries: usize) -> Self {
        Self {
            entries: HashMap::new(),
            ttl,
            max_entries,
        }
    }
    
    pub fn insert(&mut self, key: K, value: V) {
        self.cleanup_expired();
        
        // Якщо переповнення — видаляємо найстаріший
        if self.entries.len() >= self.max_entries {
            if let Some(oldest_key) = self.find_oldest() {
                self.entries.remove(&oldest_key);
            }
        }
        
        self.entries.insert(key, (value, Instant::now()));
    }
    
    pub fn get(&self, key: &K) -> Option<&V> {
        self.entries.get(key).and_then(|(v, created)| {
            if created.elapsed() < self.ttl {
                Some(v)
            } else {
                None
            }
        })
    }
    
    fn cleanup_expired(&mut self) {
        self.entries.retain(|_, (_, created)| created.elapsed() < self.ttl);
    }
    
    fn find_oldest(&self) -> Option<K> 
    where 
        K: Clone 
    {
        self.entries.iter()
            .min_by_key(|(_, (_, created))| *created)
            .map(|(k, _)| k.clone())
    }
}
```

### Моніторинг ресурсів

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

pub struct ResourceMonitor {
    active_connections: AtomicUsize,
    active_tasks: AtomicUsize,
    peak_memory: AtomicUsize,
}

impl ResourceMonitor {
    pub const fn new() -> Self {
        Self {
            active_connections: AtomicUsize::new(0),
            active_tasks: AtomicUsize::new(0),
            peak_memory: AtomicUsize::new(0),
        }
    }
    
    pub fn connection_opened(&self) {
        self.active_connections.fetch_add(1, Ordering::Relaxed);
    }
    
    pub fn connection_closed(&self) {
        self.active_connections.fetch_sub(1, Ordering::Relaxed);
    }
    
    pub fn task_started(&self) {
        self.active_tasks.fetch_add(1, Ordering::Relaxed);
    }
    
    pub fn task_finished(&self) {
        self.active_tasks.fetch_sub(1, Ordering::Relaxed);
    }
    
    pub fn report(&self) -> ResourceReport {
        ResourceReport {
            active_connections: self.active_connections.load(Ordering::Relaxed),
            active_tasks: self.active_tasks.load(Ordering::Relaxed),
        }
    }
    
    pub fn check_for_leaks(&self) -> Vec<String> {
        let mut warnings = Vec::new();
        
        let conns = self.active_connections.load(Ordering::Relaxed);
        if conns > 100 {
            warnings.push(format!("High connection count: {}", conns));
        }
        
        let tasks = self.active_tasks.load(Ordering::Relaxed);
        if tasks > 1000 {
            warnings.push(format!("High task count: {}", tasks));
        }
        
        warnings
    }
}

// RAII guard для автоматичного обліку
pub struct ConnectionGuard<'a> {
    monitor: &'a ResourceMonitor,
}

impl<'a> ConnectionGuard<'a> {
    pub fn new(monitor: &'a ResourceMonitor) -> Self {
        monitor.connection_opened();
        Self { monitor }
    }
}

impl Drop for ConnectionGuard<'_> {
    fn drop(&mut self) {
        self.monitor.connection_closed();
    }
}
```

### Безпечне завершення з очищенням

```rust
use tokio::sync::broadcast;

pub struct SwarmController {
    drones: Vec<Rc<RefCell<Drone>>>,
    shutdown_tx: broadcast::Sender<()>,
    resource_monitor: ResourceMonitor,
}

impl SwarmController {
    pub async fn shutdown(&mut self) {
        // Сигнал всім компонентам
        let _ = self.shutdown_tx.send(());
        
        // Очікуємо завершення задач
        tokio::time::sleep(Duration::from_secs(5)).await;
        
        // Примусове очищення
        self.drones.clear();
        
        // Перевірка на витоки
        let report = self.resource_monitor.report();
        if report.active_connections > 0 {
            log::warn!("Shutdown with {} active connections", report.active_connections);
        }
        if report.active_tasks > 0 {
            log::warn!("Shutdown with {} active tasks", report.active_tasks);
        }
    }
}
```

---

## Резюме

У цьому розділі ми розглянули підступні аспекти керування пам'яттю та ресурсами.

**Rust дозволяє витоки**: memory safety ≠ відсутність витоків. Safe код може створювати витоки через Rc-цикли, mem::forget, unbounded channels.

**Rc/Arc цикли**: головне джерело витоків. Виникають у двозв'язних списках, parent-child зв'язках, observer pattern, графах. Рішення — Weak для зворотних посилань.

**Weak<T>**: слабке посилання, що не запобігає звільненню. Правило: strong вниз (parent → child), weak вгору (child → parent).

**Інші джерела витоків**: mem::forget, Box::leak, unbounded channels, замикання з захопленням, глобальний стан без очищення.

**Ресурси ≠ пам'ять**: file descriptors, connections, locks теж можуть "витікати". RAII допомагає, але не вирішує проблему циклів.

**Drop і порядок**: поля — в порядку оголошення, локальні змінні — LIFO. Паніка в Drop під час іншої паніки → abort.

**Профілювання**: Valgrind, heaptrack, власні метрики, перевірка strong_count у тестах.

---

## 🔗 Зв'язок з наступним матеріалом

Опанувавши підступності пам'яті та ресурсів, ви завершили серію розділів про типові пастки в Rust. Ці знання — фундамент для створення надійних систем, що працюють безперервно без деградації продуктивності та вичерпання ресурсів.
