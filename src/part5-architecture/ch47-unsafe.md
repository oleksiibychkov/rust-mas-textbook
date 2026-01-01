# Розділ 47: Unsafe Rust — Межі безпеки

---

## 📋 Анотація

Протягом усього курсу ви працювали в **safe Rust** — середовищі, де компілятор гарантує відсутність певних класів помилок: data races, use-after-free, buffer overflows. Це потужна перевага Rust над C/C++. Але іноді ці гарантії **заважають** зробити те, що необхідно.

Розглянемо сценарії:
- Ви пишете драйвер сенсора для БПЛА. Доступ до hardware registers — це звернення до фіксованих адрес пам'яті. Як це зробити, якщо Rust не дозволяє довільні вказівники?
- Ви інтегруєте бібліотеку планування шляхів на C. Як викликати C-код з Rust?
- Ви реалізуєте високопродуктивну структуру даних. Стандартні абстракції занадто повільні, потрібен прямий доступ до пам'яті.

Для цих випадків існує **unsafe Rust** — контрольований режим, де **ви** гарантуєте безпеку замість компілятора. Unsafe — це не "вимкнути всі перевірки", а "взяти відповідальність за конкретні операції".

Цей розділ пояснює **філософію** unsafe, **п'ять конкретних операцій**, які він дозволяє, **небезпеки** (Undefined Behavior), та **стратегії** мінімізації unsafe у вашому коді.

---

## 🎯 Цілі навчання

Після завершення цього розділу ви зможете:

1. **Пояснити** філософію unsafe та його місце в системі безпеки Rust
2. **Перелічити** п'ять "суперсил" unsafe та пояснити кожну
3. **Розуміти** Undefined Behavior та чому він критично небезпечний
4. **Застосовувати** патерн "safe API над unsafe деталями"
5. **Використовувати** unsafe для FFI та hardware access
6. **Використовувати** інструменти перевірки (Miri)

---

## 📚 Ключові терміни

| Термін | Визначення |
|--------|------------|
| **Unsafe** | Режим Rust, де програміст бере відповідальність за безпеку певних операцій |
| **Safe Rust** | Код без unsafe блоків, де компілятор гарантує memory safety |
| **Raw pointer** | Вказівник без гарантій валідності: `*const T`, `*mut T` |
| **Undefined Behavior (UB)** | Поведінка, не визначена мовою; компілятор може робити що завгодно |
| **FFI** | Foreign Function Interface — виклик коду з інших мов |
| **Unsafe surface** | Площа коду, де потенційно можливий UB |
| **Soundness** | Властивість: safe API не може викликати UB |
| **Invariant** | Умова, що завжди має виконуватись для коректності |

---

## 💡 Мотиваційний кейс: Драйвер сенсора

Уявімо: ви розробляєте firmware для БПЛА. Потрібно читати дані з акселерометра, підключеного через memory-mapped I/O. Регістр статусу знаходиться за адресою `0x4000_0000`, регістр даних — `0x4000_0004`.

```rust
// Як це зробити в safe Rust?
fn read_accelerometer() -> i16 {
    // ???
    // Немає способу звернутись до довільної адреси пам'яті!
}
```

У safe Rust — ніяк. Rust не дозволяє створювати вказівники на довільні адреси та дереференсити їх. Це **правильно** для більшості програм — такий код зазвичай є помилкою.

Але для embedded — це легітимна потреба. Unsafe дозволяє це зробити:

```rust
fn read_accelerometer() -> Option<i16> {
    const STATUS_REG: *const u32 = 0x4000_0000 as *const u32;
    const DATA_REG: *const i16 = 0x4000_0004 as *const i16;
    
    unsafe {
        // Volatile read — компілятор не оптимізує
        let status = std::ptr::read_volatile(STATUS_REG);
        if status & 0x01 != 0 {  // Дані готові
            Some(std::ptr::read_volatile(DATA_REG))
        } else {
            None
        }
    }
}
```

Unsafe не означає "небезпечний код". Він означає "я, програміст, гарантую, що цей код безпечний, бо знаю контекст, який компілятор не може перевірити" (наприклад, що ця адреса справді існує на цьому hardware).

---

## 47.1 ФІЛОСОФІЯ UNSAFE

### 47.1.1 Що означає "безпечний"?

**Safe Rust** гарантує відсутність:
- **Memory safety violations**: use-after-free, double-free, buffer overflows
- **Data races**: неконтрольований одночасний доступ до даних
- **Null pointer dereferences**: гарантовано через Option<T>
- **Uninitialized memory access**: компілятор перевіряє ініціалізацію

Це досягається через систему ownership, borrowing, та типів. Компілятор **статично** (під час компіляції) перевіряє, що ці помилки неможливі.

### 47.1.2 Що означає "unsafe"?

**Unsafe** — це **не** "вимкнути всі перевірки". Це **контракт**:

> "Компілятору: я роблю операцію, яку ти не можеш перевірити. Я гарантую, що вона безпечна."

Unsafe **не вимикає**:
- Перевірку типів
- Borrow checker
- Lifetime перевірки
- Bounds checking (крім явних unchecked операцій)

Unsafe **дозволяє** лише п'ять конкретних операцій (див. наступний розділ).

### 47.1.3 Візуалізація

```text
┌─────────────────────────────────────────────────────────────────────┐
│                    SAFE RUST                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │                                                               │ │
│  │   Компілятор гарантує:                                       │ │
│  │   • Memory safety                                             │ │
│  │   • Thread safety (через Send/Sync)                          │ │
│  │   • No null dereferences                                      │ │
│  │   • No data races                                             │ │
│  │                                                               │ │
│  │   Ви пишете код, компілятор перевіряє                        │ │
│  │                                                               │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                    UNSAFE RUST                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │                                                               │ │
│  │   Компілятор НЕ перевіряє:                                   │ │
│  │   • Валідність raw pointers                                   │ │
│  │   • Preconditions unsafe функцій                              │ │
│  │   • Правильність доступу до static mut                       │ │
│  │   • Коректність unsafe trait реалізацій                      │ │
│  │   • Безпеку доступу до union полів                           │ │
│  │                                                               │ │
│  │   ВИ гарантуєте безпеку цих операцій                         │ │
│  │                                                               │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  Все інше (типи, borrowing, lifetimes) — ВСЕ ЩЕ ПЕРЕВІРЯЄТЬСЯ!    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 47.2 П'ЯТЬ СУПЕРСИЛ UNSAFE

Unsafe дозволяє рівно **п'ять** операцій, неможливих у safe Rust. Лише п'ять — і жодної більше.

### 47.2.1 Огляд суперсил

```text
┌─────────────────────────────────────────────────────────────────────┐
│                 П'ЯТЬ СУПЕРСИЛ UNSAFE                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. ДЕРЕФЕРЕНС RAW POINTERS                                         │
│     *const T, *mut T → читання/запис за адресою                     │
│                                                                     │
│  2. ВИКЛИК UNSAFE ФУНКЦІЙ                                           │
│     unsafe fn dangerous() → функції з preconditions                 │
│                                                                     │
│  3. ДОСТУП ДО MUTABLE STATIC                                        │
│     static mut GLOBAL: T → глобальні мутабельні змінні              │
│                                                                     │
│  4. РЕАЛІЗАЦІЯ UNSAFE TRAITS                                        │
│     unsafe impl Send for T → контракти, які компілятор              │
│                               не може перевірити                    │
│                                                                     │
│  5. ДОСТУП ДО ПОЛІВ UNION                                           │
│     union U { a: T, b: V } → переінтерпретація даних                │
│                                                                     │
│  ВСЕ ІНШЕ — НЕ ПОТРЕБУЄ UNSAFE!                                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 47.2.2 Суперсила 1: Дереференс raw pointers

**Що таке raw pointers?**

Raw pointers — це "сирі" вказівники без гарантій Rust:

```rust
let x: i32 = 42;

// Посилання (safe) — з гарантіями
let ref_x: &i32 = &x;

// Raw pointer (unsafe для дереференсу) — без гарантій
let ptr_x: *const i32 = &x as *const i32;
```

**Важливо:** *створити* raw pointer можна в safe Rust. *Дереференсити* (прочитати значення за адресою) — тільки в unsafe.

**Постановка задачі:** Продемонструємо створення та дереференс raw pointer.

```rust
fn main() {
    let mut value: i32 = 100;
    
    // Створення raw pointers — safe!
    let ptr_const: *const i32 = &value;       // Вказівник для читання
    let ptr_mut: *mut i32 = &mut value;       // Вказівник для запису
    
    // Дереференс — unsafe!
    unsafe {
        println!("Через *const: {}", *ptr_const);  // Читання
        *ptr_mut = 200;                            // Запис
        println!("Після запису: {}", *ptr_const);  // 200
    }
}
```

**Чому unsafe?**

Компілятор не може гарантувати, що raw pointer:
- Вказує на валідну пам'ять
- Не є null
- Не є dangling (об'єкт ще існує)
- Не порушує aliasing rules

**Властивості raw pointers:**

| Властивість | Посилання (&T, &mut T) | Raw pointer (*const T, *mut T) |
|-------------|------------------------|--------------------------------|
| Гарантія валідності | ✅ Так | ❌ Ні |
| Гарантія non-null | ✅ Так | ❌ Ні |
| Lifetime tracking | ✅ Так | ❌ Ні |
| Aliasing rules | ✅ Enforced | ❌ Не перевіряється |
| Множинні `*mut T` | ❌ Заборонено | ✅ Можливо |

### 47.2.3 Суперсила 2: Виклик unsafe функцій

Деякі функції мають **preconditions** (передумови), які компілятор не може перевірити. Такі функції позначаються як `unsafe fn`.

**Постановка задачі:** Створимо unsafe функцію з precondition та покажемо її використання.

```rust
/// Повертає елемент за індексом БЕЗ перевірки меж.
/// 
/// # Safety
/// 
/// Викликач ГАРАНТУЄ, що:
/// - `index < slice.len()`
/// 
/// Порушення цього контракту — Undefined Behavior.
unsafe fn get_unchecked(slice: &[i32], index: usize) -> i32 {
    *slice.as_ptr().add(index)
}

fn main() {
    let data = [10, 20, 30, 40, 50];
    
    // Safe версія — з перевіркою
    if let Some(value) = data.get(2) {
        println!("Safe: {}", value);  // 30
    }
    
    // Unsafe версія — швидше, але відповідальність на нас
    unsafe {
        // Ми ЗНАЄМО, що 2 < 5, тому це безпечно
        let value = get_unchecked(&data, 2);
        println!("Unsafe: {}", value);  // 30
        
        // ⚠️ Це було б UB:
        // let bad = get_unchecked(&data, 100);  // Читання за межами!
    }
}
```

**Коли робити функцію unsafe?**

Функція має бути `unsafe fn`, якщо:
1. Вона має preconditions, які компілятор не може перевірити
2. Порушення цих preconditions може призвести до UB

**Документація `# Safety` — обов'язкова!** Кожна unsafe функція має документувати свої preconditions.

### 47.2.4 Суперсила 3: Доступ до mutable static

**Постановка задачі:** Продемонструємо глобальну мутабельну змінну та чому вона unsafe.

```rust
// Глобальна мутабельна змінна
static mut REQUEST_COUNTER: u64 = 0;

fn handle_request() {
    unsafe {
        REQUEST_COUNTER += 1;
        println!("Request #{}", REQUEST_COUNTER);
    }
}

fn main() {
    handle_request();  // Request #1
    handle_request();  // Request #2
    handle_request();  // Request #3
}
```

**Чому unsafe?**

У багатопотоковій програмі два потоки можуть одночасно:
- Читати `REQUEST_COUNTER`
- Писати `REQUEST_COUNTER`

Це **data race** — одна з найнебезпечніших помилок, що може призвести до UB.

**Краща альтернатива:**

```rust
use std::sync::atomic::{AtomicU64, Ordering};

// Атомарна змінна — safe!
static REQUEST_COUNTER: AtomicU64 = AtomicU64::new(0);

fn handle_request() {
    let count = REQUEST_COUNTER.fetch_add(1, Ordering::SeqCst);
    println!("Request #{}", count + 1);
}
```

`AtomicU64` гарантує атомарність операцій — data race неможливий.

### 47.2.5 Суперсила 4: Реалізація unsafe traits

Деякі traits мають **контракти**, які компілятор не може перевірити. Такі traits позначаються як `unsafe trait`.

**Найважливіші unsafe traits:**

| Trait | Контракт |
|-------|----------|
| `Send` | Тип можна безпечно передавати між потоками |
| `Sync` | Тип можна безпечно ділити між потоками через `&T` |

Компілятор автоматично реалізує Send/Sync для типів, де це безпечно. Ручна реалізація потребує unsafe:

```rust
struct MyWrapper(*mut u8);  // Містить raw pointer

// Компілятор НЕ реалізує Send автоматично для *mut T
// Ми гарантуємо, що MyWrapper можна передавати між потоками
unsafe impl Send for MyWrapper {}
```

**Застереження:** Неправильна реалізація Send/Sync може призвести до data races — одного з найнебезпечніших UB.

### 47.2.6 Суперсила 5: Доступ до полів union

`union` — тип, де всі поля займають ту саму пам'ять (як у C).

**Постановка задачі:** Продемонструємо union та чому доступ до полів unsafe.

```rust
// Усі поля займають ту саму пам'ять
union IntOrFloat {
    int_value: i32,
    float_value: f32,
}

fn main() {
    let u = IntOrFloat { int_value: 42 };
    
    unsafe {
        // Доступ до полів — unsafe
        println!("As int: {}", u.int_value);     // 42
        println!("As float: {}", u.float_value); // Переінтерпретація байтів!
    }
}
```

**Чому unsafe?**

Ви маєте **знати**, яке поле актуальне. Читання "неактивного" поля — переінтерпретація байтів, що може бути UB (наприклад, для типів з інваріантами).

---

## 47.3 UNDEFINED BEHAVIOR (UB)

### 47.3.1 Що таке UB?

**Undefined Behavior** — це код, поведінку якого **стандарт мови не визначає**. Коли програма містить UB, компілятор може робити **що завгодно**:

- Видати "правильний" результат (випадково)
- Видати неправильний результат
- Зламати "непов'язані" частини програми
- Форматувати жорсткий диск (теоретично)
- Працювати в debug, падати в release

### 47.3.2 Чому UB такий небезпечний?

Компілятор **оптимізує**, припускаючи, що UB не відбувається. Це може призвести до несподіваних наслідків:

```rust
// Приклад: компілятор може "оптимізувати" перевірку
fn check_and_use(ptr: *const i32) {
    unsafe {
        // Якщо ptr == null, це UB (null dereference)
        let value = *ptr;
        
        // Компілятор думає: "UB не відбувається, отже ptr != null"
        // Тому ця перевірка може бути видалена!
        if ptr.is_null() {
            println!("This is null!");  // Ніколи не виконається
        }
    }
}
```

### 47.3.3 Приклади UB

**UB 1: Null pointer dereference**

```rust
let ptr: *const i32 = std::ptr::null();
unsafe { *ptr }  // ❌ UB!
```

**UB 2: Buffer overflow / out-of-bounds access**

```rust
let arr = [1, 2, 3];
unsafe { *arr.as_ptr().add(100) }  // ❌ UB!
```

**UB 3: Data race**

```rust
static mut COUNTER: i32 = 0;

// Два потоки одночасно:
// Потік 1: unsafe { COUNTER += 1; }
// Потік 2: unsafe { COUNTER += 1; }
// ❌ UB!
```

**UB 4: Use-after-free**

```rust
let ptr: *const String;
{
    let s = String::from("hello");
    ptr = &s as *const String;
}  // s звільнено

unsafe { println!("{}", *ptr); }  // ❌ UB! Dangling pointer
```

**UB 5: Uninitialized memory**

```rust
let x: i32;
unsafe { std::ptr::read(&x) }  // ❌ UB!
```

**UB 6: Aliasing violation**

```rust
let mut x = 42;
let ptr1 = &mut x as *mut i32;
let ptr2 = &mut x as *mut i32;  // Два &mut на ті самі дані!

unsafe {
    *ptr1 = 10;
    *ptr2 = 20;  // ❌ Потенційно UB через порушення aliasing rules
}
```

---

## 47.4 UNSAFE БЛОКИ VS UNSAFE ФУНКЦІЇ

### 47.4.1 Unsafe блоки

Unsafe блок позначає **конкретне місце**, де ви виконуєте unsafe операції:

```rust
fn safe_function() {
    let ptr: *const i32 = &42 as *const i32;
    
    // Тільки ця ділянка unsafe
    let value = unsafe { *ptr };
    
    // Тут знову safe
    println!("Value: {}", value);
}
```

### 47.4.2 Unsafe функції

Unsafe функція **перекладає** відповідальність на викликача:

```rust
/// # Safety
/// - `ptr` має вказувати на валідний i32
/// - `ptr` не може бути null
unsafe fn read_ptr(ptr: *const i32) -> i32 {
    *ptr  // Без unsafe блоку — вся функція unsafe
}
```

### 47.4.3 Коли що використовувати

| Сценарій | Використовуйте |
|----------|----------------|
| Ви можете перевірити всі preconditions | `unsafe { }` блок |
| Preconditions залежать від викликача | `unsafe fn` |
| Обгортка над unsafe з перевірками | Safe функція з `unsafe { }` всередині |

### 47.4.4 Приклад: Safe обгортка

**Постановка задачі:** Створимо safe функцію, що інкапсулює unsafe операцію.

```rust
/// Safe функція — перевіряє bounds, потім викликає unsafe
fn get_element(slice: &[i32], index: usize) -> Option<i32> {
    if index < slice.len() {
        // Precondition виконано — unsafe безпечний
        Some(unsafe { *slice.as_ptr().add(index) })
    } else {
        None
    }
}

fn main() {
    let data = [10, 20, 30];
    
    // Safe API — неможливо викликати UB
    println!("{:?}", get_element(&data, 1));    // Some(20)
    println!("{:?}", get_element(&data, 100));  // None (не UB!)
}
```

**Ключовий патерн:** Safe API перевіряє preconditions, потім делегує unsafe.

---

## 47.5 МІНІМІЗАЦІЯ UNSAFE SURFACE

### 47.5.1 Принцип: Safe API над unsafe деталями

Найкраща практика — **ізолювати** unsafe у невеликих, добре документованих функціях, та надати **safe API** для решти коду:

```text
┌─────────────────────────────────────────────────────────────────────┐
│                    АРХІТЕКТУРА КОДУ                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │                                                               │ │
│  │               SAFE API (99% коду)                             │ │
│  │                                                               │ │
│  │   Користувачі вашого модуля працюють тут                     │ │
│  │   Неможливо викликати UB через цей API                       │ │
│  │                                                               │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                               │                                     │
│                               ▼                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │                                                               │ │
│  │          UNSAFE INTERNALS (1% коду)                          │ │
│  │                                                               │ │
│  │   Добре документовані unsafe функції                         │ │
│  │   Ретельно перевірені                                        │ │
│  │   Мінімальна площа                                           │ │
│  │                                                               │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 47.5.2 Патерни мінімізації

**Патерн 1: Перевірка перед unsafe**

```rust
// ❌ Погано: unsafe з некеруваним precondition
fn get(slice: &[i32], index: usize) -> i32 {
    unsafe { *slice.as_ptr().add(index) }
}

// ✅ Добре: перевірка + panic або Option
fn get_safe(slice: &[i32], index: usize) -> Option<i32> {
    if index < slice.len() {
        Some(unsafe { *slice.as_ptr().add(index) })
    } else {
        None
    }
}
```

**Патерн 2: Newtype для інваріантів**

```rust
/// NonNull вказівник — гарантовано не null
pub struct NonNullPtr<T>(*mut T);

impl<T> NonNullPtr<T> {
    /// Створює NonNullPtr, перевіряючи non-null
    pub fn new(ptr: *mut T) -> Option<Self> {
        if ptr.is_null() {
            None
        } else {
            Some(NonNullPtr(ptr))
        }
    }
    
    /// Safe, бо ми гарантували non-null при створенні
    pub fn as_ref(&self) -> &T {
        unsafe { &*self.0 }
    }
}
```

**Патерн 3: Інкапсуляція небезпечного стану**

```rust
/// Buffer з інваріантом: capacity >= len
pub struct Buffer {
    ptr: *mut u8,
    len: usize,
    capacity: usize,
}

impl Buffer {
    // Всі методи підтримують інваріант
    pub fn push(&mut self, byte: u8) {
        if self.len == self.capacity {
            self.grow();  // Внутрішній unsafe
        }
        unsafe {
            *self.ptr.add(self.len) = byte;
        }
        self.len += 1;
    }
}
```

---

## 47.6 UNSAFE У СТАНДАРТНІЙ БІБЛІОТЕЦІ

### 47.6.1 Vec, String, HashMap — unsafe всередині

Стандартні колекції Rust використовують unsafe внутрішньо для ефективності:

```rust
// Спрощена ідея Vec
pub struct Vec<T> {
    ptr: *mut T,       // Raw pointer на heap
    len: usize,
    capacity: usize,
}

impl<T> Vec<T> {
    pub fn push(&mut self, value: T) {
        if self.len == self.capacity {
            self.grow();  // Unsafe: realloc
        }
        unsafe {
            std::ptr::write(self.ptr.add(self.len), value);
        }
        self.len += 1;
    }
}
```

**Safe API** (`push`, `pop`, `get`) приховує unsafe деталі. Користувач не може викликати UB через публічний API.

### 47.6.2 UnsafeCell — основа interior mutability

`UnsafeCell<T>` — єдиний легальний спосіб мутувати дані через `&T`:

```rust
// Спрощена ідея Cell
pub struct Cell<T> {
    value: UnsafeCell<T>,
}

impl<T: Copy> Cell<T> {
    pub fn get(&self) -> T {
        unsafe { *self.value.get() }  // get() повертає *mut T
    }
    
    pub fn set(&self, value: T) {
        unsafe { *self.value.get() = value; }
    }
}
```

`RefCell`, `Mutex`, `RwLock` — всі побудовані на `UnsafeCell`.

---

## 47.7 FFI — ВИКЛИК C-КОДУ

### 47.7.1 Оголошення зовнішніх функцій

Rust може викликати функції з C-бібліотек через FFI:

```rust
// Оголошення C-функцій
extern "C" {
    fn abs(x: i32) -> i32;
    fn strlen(s: *const i8) -> usize;
    fn malloc(size: usize) -> *mut u8;
    fn free(ptr: *mut u8);
}

fn main() {
    unsafe {
        let result = abs(-42);
        println!("abs(-42) = {}", result);  // 42
    }
}
```

### 47.7.2 Чому FFI unsafe?

Rust не може перевірити:
- Чи функція взагалі існує
- Чи правильна сигнатура
- Чи виконуються preconditions C-функції
- Чи C-код не має UB

### 47.7.3 Safe обгортка над C

**Постановка задачі:** Створимо safe обгортку для роботи з C-рядками.

```rust
use std::ffi::CStr;

extern "C" {
    fn strlen(s: *const i8) -> usize;
}

/// Safe обгортка — приймає Rust str, конвертує в C string
pub fn c_strlen(s: &str) -> Option<usize> {
    // Перевіряємо, що рядок не містить null bytes
    if s.bytes().any(|b| b == 0) {
        return None;  // C string не може мати null всередині
    }
    
    // Створюємо null-terminated string
    let c_string = std::ffi::CString::new(s).ok()?;
    
    unsafe {
        Some(strlen(c_string.as_ptr()))
    }
}

fn main() {
    println!("Length: {:?}", c_strlen("Hello"));  // Some(5)
}
```

---

## 47.8 UNSAFE ДЛЯ АГЕНТІВ: HARDWARE ACCESS

### 47.8.1 Memory-Mapped I/O (MMIO)

Hardware registers часто доступні через фіксовані адреси пам'яті:

```text
┌─────────────────────────────────────────────────────────────────────┐
│                  MEMORY MAP (приклад)                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  0x0000_0000 - 0x0FFF_FFFF: RAM                                    │
│                                                                     │
│  0x4000_0000 - 0x4000_00FF: Sensor registers                       │
│      0x4000_0000: STATUS (read-only)                               │
│      0x4000_0004: DATA (read-only)                                 │
│      0x4000_0008: CONTROL (read-write)                             │
│                                                                     │
│  0x4001_0000 - 0x4001_00FF: Motor registers                        │
│      0x4001_0000: SPEED (read-write)                               │
│      0x4001_0004: DIRECTION (read-write)                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 47.8.2 Volatile операції

Для hardware registers потрібні **volatile** операції — компілятор не має оптимізувати/кешувати доступ:

**Постановка задачі:** Створимо safe абстракцію для читання сенсора.

```rust
use std::ptr::{read_volatile, write_volatile};

/// Адреси регістрів сенсора
const SENSOR_STATUS: *const u32 = 0x4000_0000 as *const u32;
const SENSOR_DATA: *const i16 = 0x4000_0004 as *const i16;
const SENSOR_CONTROL: *mut u32 = 0x4000_0008 as *mut u32;

/// Safe обгортка для сенсора
pub struct Sensor;

impl Sensor {
    /// Перевіряє, чи дані готові
    pub fn is_data_ready(&self) -> bool {
        unsafe {
            let status = read_volatile(SENSOR_STATUS);
            status & 0x01 != 0  // Біт 0 = data ready
        }
    }
    
    /// Читає дані сенсора (блокує до готовності)
    pub fn read(&self) -> i16 {
        while !self.is_data_ready() {
            // Spin wait
        }
        unsafe { read_volatile(SENSOR_DATA) }
    }
    
    /// Запускає нове вимірювання
    pub fn start_measurement(&mut self) {
        unsafe {
            write_volatile(SENSOR_CONTROL, 0x01);  // Start bit
        }
    }
}
```

**Чому `read_volatile` / `write_volatile`?**

Звичайне читання компілятор може:
- Оптимізувати (не читати взагалі, якщо "знає" значення)
- Кешувати (прочитати один раз, використовувати кеш)
- Перегрупувати (читати в іншому порядку)

Для hardware registers це **катастрофа** — регістр може змінюватись hardware'ом, і кожне читання має бути реальним.

### 47.8.3 Архітектура HAL

```text
┌─────────────────────────────────────────────────────────────────────┐
│                    АРХІТЕКТУРА HAL                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │           КОД АГЕНТА (100% Safe Rust)                         │ │
│  │                                                               │ │
│  │   let reading = sensor.read();                                │ │
│  │   motor.set_speed(50);                                        │ │
│  │   gps.get_position();                                         │ │
│  │                                                               │ │
│  └─────────────────────────────┬─────────────────────────────────┘ │
│                                │                                    │
│                                ▼                                    │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │           HAL — Hardware Abstraction Layer                    │ │
│  │                                                               │ │
│  │   pub struct Sensor { ... }                                   │ │
│  │   impl Sensor {                                               │ │
│  │       pub fn read(&self) -> i16 { ... }   // Safe API        │ │
│  │   }                                                           │ │
│  │                                                               │ │
│  └─────────────────────────────┬─────────────────────────────────┘ │
│                                │                                    │
│                                ▼                                    │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │           UNSAFE INTERNALS                                    │ │
│  │                                                               │ │
│  │   unsafe { read_volatile(SENSOR_DATA) }                       │ │
│  │   unsafe { write_volatile(MOTOR_SPEED, value) }               │ │
│  │                                                               │ │
│  └─────────────────────────────┬─────────────────────────────────┘ │
│                                │                                    │
│                                ▼                                    │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │                       HARDWARE                                │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 47.9 ІНСТРУМЕНТИ ПЕРЕВІРКИ

### 47.9.1 Miri — інтерпретатор для виявлення UB

**Miri** — інструмент, що виконує Rust-код в інтерпретаторі та виявляє UB:

```bash
# Встановлення
rustup +nightly component add miri

# Запуск програми
cargo +nightly miri run

# Запуск тестів
cargo +nightly miri test
```

**Що виявляє Miri:**
- Доступ до неініціалізованої пам'яті
- Use-after-free
- Buffer overflows
- Data races
- Порушення aliasing rules (stacked borrows)
- Memory leaks

**Приклад:**

```rust
fn main() {
    let x: i32;  // Неініціалізована
    unsafe {
        println!("{}", std::ptr::read(&x));  // UB!
    }
}
```

```bash
$ cargo +nightly miri run
error: Undefined Behavior: using uninitialized data
```

### 47.9.2 Address Sanitizer (ASan)

```bash
RUSTFLAGS="-Z sanitizer=address" cargo +nightly run
```

Виявляє memory errors під час виконання реальної програми.

### 47.9.3 cargo clippy

```bash
# Попереджає про unsafe без документації
cargo clippy -- -W clippy::undocumented_unsafe_blocks
```

---

## 47.10 ЛАБОРАТОРНА РОБОТА

### Мета роботи

Зрозуміти unsafe та його правильне використання через практику.

### Завдання 1: Аналіз unsafe у std (3 бали)

1. Відкрийте документацію `Vec::from_raw_parts`
2. Прочитайте секцію Safety
3. Опишіть усі preconditions
4. Поясніть, чому кожен precondition важливий

### Завдання 2: Safe обгортка (4 бали)

Напишіть safe функцію-обгортку для:

```rust
/// Unsafe: не перевіряє bounds
unsafe fn sum_unchecked(slice: &[i32], start: usize, end: usize) -> i32 {
    let mut sum = 0;
    for i in start..end {
        sum += *slice.as_ptr().add(i);
    }
    sum
}
```

Обгортка має:
- Перевіряти bounds
- Повертати Option<i32>
- Бути повністю safe для користувача

### Завдання 3: Miri (3 бали)

Встановіть Miri та запустіть на коді з навмисним UB:

```rust
fn main() {
    let ptr: *const i32;
    {
        let x = 42;
        ptr = &x;
    }  // x звільнено
    
    unsafe {
        println!("{}", *ptr);  // Dangling pointer
    }
}
```

Опишіть, що Miri виявив та як інтерпретувати повідомлення.

---

## 47.11 РЕЗЮМЕ

### Ключові концепції

**Unsafe** — контрольований режим, де програміст гарантує безпеку замість компілятора.

**П'ять суперсил:**
1. Дереференс raw pointers
2. Виклик unsafe функцій
3. Доступ до mutable static
4. Реалізація unsafe traits
5. Доступ до полів union

**Undefined Behavior** — код без визначеної поведінки; компілятор може оптимізувати припускаючи відсутність UB.

**Safe API над unsafe** — головний патерн мінімізації ризиків.

### Коли використовувати unsafe

| Сценарій | Unsafe? |
|----------|---------|
| FFI / виклик C-коду | ✅ Потрібен |
| Hardware access (MMIO) | ✅ Потрібен |
| Реалізація низькорівневих структур | ✅ Можливо |
| Performance optimization (після профілювання) | ⚠️ Обережно |
| "Обійти borrow checker" | ❌ Ніколи |
| "На всяк випадок" | ❌ Ніколи |

---

## 🔗 ЗВ'ЯЗОК З НАСТУПНИМ РОЗДІЛОМ

Unsafe дозволяє низькорівневу роботу, але для великих симуляцій МАС потрібна інша архітектура — **Entity-Component-System (ECS)**.

ECS — архітектурний патерн, популярний у геймдеві, ідеальний для симуляцій з тисячами об'єктів. Він кардинально відрізняється від ООП та Actor Model.

У **Розділі 48: Entity-Component-System** ви:
- Дізнаєтесь про філософію ECS: entities, components, systems
- Познайомитесь з bevy_ecs
- Побудуєте рій агентів в ECS стилі
- Порівняєте ECS з Actor Model для МАС

---

> **Наступний розділ:** [Розділ 48: Entity-Component-System (ECS)](./48_ECS.md)
