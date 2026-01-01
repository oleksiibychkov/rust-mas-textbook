# ДОДАТОК В: Міграція з C на Rust

## Практичний посібник для розробників вбудованих систем та БПЛА

---

## В.1. Навіщо мігрувати?

### В.1.1. Реальна ціна помилок у C

У 2021 році дослідження безпеки популярної утиліти cURL (написаної на C) показало, що **53 із 95 відомих вразливостей** могли б бути усунені використанням Rust. Це не теоретичні міркування — це реальні CVE (Common Vulnerabilities and Exposures), що призвели до компрометації систем по всьому світу.

Для систем керування БПЛА та IoT-пристроїв ціна помилки ще вища:

| Тип помилки | Наслідки в IoT/БПЛА |
|-------------|---------------------|
| Buffer overflow | Втрата контролю над апаратом |
| Use-after-free | Непередбачувана поведінка в польоті |
| Data race | Конфлікт між підсистемами навігації |
| Null pointer | Аварійне завершення польотного контролера |
| Memory leak | Деградація продуктивності, відмова місії |

### В.1.2. Що дає Rust для вбудованих систем

**Без runtime-overhead:**
- Відсутність garbage collector
- Передбачуваний час виконання (важливо для real-time систем)
- Контроль над розміщенням у пам'яті

**Гарантії на етапі компіляції:**
- Відсутність data races
- Відсутність use-after-free
- Відсутність null pointer dereference
- Безпечна робота з пам'яттю

**Сучасна екосистема:**
- Cargo для керування залежностями
- Вбудоване тестування
- Крос-компіляція "з коробки"
- Активна спільнота embedded Rust

---

## В.2. Стратегії міграції

### В.2.1. Три підходи

```text
┌─────────────────────────────────────────────────────────────────┐
│                    СТРАТЕГІЇ МІГРАЦІЇ                           │
├─────────────────┬─────────────────────┬─────────────────────────┤
│   ПОВНИЙ        │    ПОСТУПОВИЙ       │    ГІБРИДНИЙ            │
│   ПЕРЕПИС       │    ПЕРЕПИС          │    (FFI)                │
├─────────────────┼─────────────────────┼─────────────────────────┤
│ Переписати все  │ Модуль за модулем   │ Rust + C через FFI      │
│ з нуля на Rust  │ замінювати C→Rust   │ в одному проєкті        │
├─────────────────┼─────────────────────┼─────────────────────────┤
│ ✓ Найчистіший   │ ✓ Поступовий ризик  │ ✓ Швидкий старт         │
│   результат     │ ✓ Зберігає робочу   │ ✓ Використання          │
│ ✗ Найбільший    │   систему           │   існуючих бібліотек    │
│   початковий    │ ✗ Складність        │ ✗ unsafe на границях    │
│   ризик         │   інтеграції        │ ✗ Два toolchain'и       │
└─────────────────┴─────────────────────┴─────────────────────────┘
```

### В.2.2. Рекомендований підхід для БПЛА та IoT

Для систем керування БПЛА рекомендується **поступовий перепис** із такою пріоритизацією:

1. **Спочатку:** модулі обробки даних (телеметрія, логування)
2. **Потім:** бізнес-логіка (планування місій, прийняття рішень)
3. **Далі:** комунікаційний рівень (протоколи, серіалізація)
4. **Останнім:** низькорівневі драйвери (HAL)

Причина: найбільший виграш безпеки при найменшому ризику для критичних систем.

---

## В.3. Таблиці відповідності синтаксису C → Rust

### В.3.1. Базові типи даних

| C | Rust | Примітки |
|---|------|----------|
| `char` | `i8` або `u8` | Залежить від signed/unsigned за замовчуванням |
| `signed char` | `i8` | 8-bit signed |
| `unsigned char` | `u8` | 8-bit unsigned |
| `short` | `i16` | 16-bit signed |
| `unsigned short` | `u16` | 16-bit unsigned |
| `int` | `i32` | 32-bit signed |
| `unsigned int` | `u32` | 32-bit unsigned |
| `long` | `i32` або `i64` | Платформозалежний у C, явний у Rust |
| `unsigned long` | `u32` або `u64` | Платформозалежний у C |
| `long long` | `i64` | 64-bit signed |
| `unsigned long long` | `u64` | 64-bit unsigned |
| `float` | `f32` | 32-bit floating point |
| `double` | `f64` | 64-bit floating point |
| `_Bool` / `bool` | `bool` | `true` / `false` |
| `size_t` | `usize` | Розмір вказівника платформи |
| `ssize_t` | `isize` | Signed розмір вказівника |
| `intptr_t` | `isize` | Signed pointer-sized |
| `uintptr_t` | `usize` | Unsigned pointer-sized |
| `int8_t` | `i8` | Точна відповідність |
| `uint8_t` | `u8` | Точна відповідність |
| `int16_t` | `i16` | Точна відповідність |
| `uint16_t` | `u16` | Точна відповідність |
| `int32_t` | `i32` | Точна відповідність |
| `uint32_t` | `u32` | Точна відповідність |
| `int64_t` | `i64` | Точна відповідність |
| `uint64_t` | `u64` | Точна відповідність |
| `void` | `()` | Unit type |
| `void*` | `*mut c_void` / `*const c_void` | Для FFI |

### В.3.2. Вказівники та посилання

| C | Rust | Коли використовувати |
|---|------|---------------------|
| `T*` (читання) | `&T` | Незмінне посилання (безпечне) |
| `T*` (запис) | `&mut T` | Змінне посилання (безпечне) |
| `T*` (nullable) | `Option<&T>` | Може бути NULL |
| `T*` (nullable, запис) | `Option<&mut T>` | Може бути NULL, змінне |
| `T*` (володіння) | `Box<T>` | Heap-allocated, один власник |
| `T*` (shared) | `Rc<T>` | Reference counted (однопотоковий) |
| `T*` (shared, threadsafe) | `Arc<T>` | Atomic reference counted |
| `T*` (raw, FFI) | `*const T` | Сирий вказівник для FFI |
| `T*` (raw, mutable, FFI) | `*mut T` | Сирий змінний вказівник |
| `T**` | `&mut &T` або `Box<Box<T>>` | Залежить від семантики |
| `const T*` | `&T` або `*const T` | Незмінний |
| `T* const` | `&T` | Вказівник сам константний |
| `T* restrict` | `&mut T` | Rust гарантує унікальність |
| `NULL` | `None` (для Option) | Або `std::ptr::null()` |
| `&variable` | `&variable` / `&mut variable` | Взяття посилання |
| `*ptr` | `*ptr` (unsafe) або `ptr.as_ref()` | Розіменування |

### В.3.3. Масиви та колекції

| C | Rust | Примітки |
|---|------|----------|
| `T arr[N]` | `[T; N]` | Масив фіксованого розміру |
| `T arr[]` (параметр) | `&[T]` | Слайс (посилання на масив) |
| `T arr[]` (параметр, mut) | `&mut [T]` | Змінний слайс |
| `T* arr` (динамічний) | `Vec<T>` | Динамічний масив |
| `arr[i]` | `arr[i]` | Bounds checking у Rust |
| `arr[i]` (без перевірки) | `arr.get_unchecked(i)` | unsafe |
| `sizeof(arr)/sizeof(arr[0])` | `arr.len()` | Довжина масиву |
| `memset(arr, 0, size)` | `arr.fill(0)` або `vec![0; n]` | Заповнення |
| `memcpy(dst, src, size)` | `dst.copy_from_slice(src)` | Копіювання |
| `memmove(dst, src, size)` | `dst.copy_from_slice(src)` | Безпечне в Rust |
| `char str[]` | `&str` або `String` | UTF-8 рядки |
| `char* str` (динамічний) | `String` | Owned UTF-8 рядок |
| `wchar_t*` | `String` | Rust використовує UTF-8 |

### В.3.4. Структури та типи

| C | Rust | Примітки |
|---|------|----------|
| `struct S { ... };` | `struct S { ... }` | Без крапки з комою |
| `typedef struct { ... } S;` | `struct S { ... }` | typedef не потрібен |
| `struct S s;` | `let s: S;` або `let s = S { ... };` | Ініціалізація обов'язкова |
| `struct S s = {0};` | `let s = S::default();` | Потребує `#[derive(Default)]` |
| `s.field` | `s.field` | Однаково |
| `ptr->field` | `ptr.field` або `(*ptr).field` | Автоматичний deref |
| `union U { ... };` | `union U { ... }` | unsafe для читання |
| `enum E { A, B, C };` | `enum E { A, B, C }` | Без значень |
| `enum E { A=0, B=1 };` | `enum E { A = 0, B = 1 }` | З явними значеннями |
| `enum` (з даними) | `enum E { A(i32), B(String) }` | Tagged union |
| `typedef T NewT;` | `type NewT = T;` | Type alias |
| `#define CONST 42` | `const CONST: i32 = 42;` | Типізована константа |

### В.3.5. Функції

| C | Rust | Примітки |
|---|------|----------|
| `void foo(void)` | `fn foo()` | Без параметрів |
| `int foo(int x)` | `fn foo(x: i32) -> i32` | З параметром і поверненням |
| `void foo(int* x)` | `fn foo(x: &mut i32)` | Передача за посиланням |
| `int foo(const int* x)` | `fn foo(x: &i32) -> i32` | Незмінне посилання |
| `T* foo(void)` | `fn foo() -> Option<Box<T>>` | Nullable повернення |
| `static int foo()` | `fn foo()` (без `pub`) | Приватна за замовчуванням |
| `inline int foo()` | `#[inline] fn foo()` | Inline hint |
| `extern int foo()` | `extern "C" { fn foo(); }` | FFI декларація |
| `__attribute__((noreturn))` | `fn foo() -> !` | Never type |
| `int (*fptr)(int)` | `fn(i32) -> i32` | Function pointer type |
| `int (*fptr)(int) = &foo` | `let fptr: fn(i32) -> i32 = foo` | Function pointer |
| Variadic `...` | Не підтримується (safe) | Використовуйте макроси |

### В.3.6. Оператори

| C | Rust | Примітки |
|---|------|----------|
| `=` | `=` | Присвоєння (може бути move) |
| `==`, `!=` | `==`, `!=` | Порівняння (потребує `PartialEq`) |
| `<`, `>`, `<=`, `>=` | `<`, `>`, `<=`, `>=` | Порівняння (потребує `PartialOrd`) |
| `+`, `-`, `*`, `/` | `+`, `-`, `*`, `/` | Арифметика |
| `%` | `%` | Остача від ділення |
| `++x` | `x += 1` | Інкремент (не вираз) |
| `x++` | `{ let t = x; x += 1; t }` | Постфіксний інкремент |
| `--x` | `x -= 1` | Декремент |
| `&&`, `\|\|` | `&&`, `\|\|` | Логічні оператори |
| `!` | `!` | Логічне заперечення |
| `&`, `\|`, `^` | `&`, `\|`, `^` | Побітові оператори |
| `~` | `!` | Побітове заперечення |
| `<<`, `>>` | `<<`, `>>` | Зсуви |
| `? :` | `if ... { } else { }` | Тернарний → if expression |
| `(T)x` | `x as T` | Приведення типу |
| `sizeof(T)` | `std::mem::size_of::<T>()` | Розмір типу |
| `sizeof(x)` | `std::mem::size_of_val(&x)` | Розмір значення |
| `&x` | `&x` / `&mut x` | Взяття посилання |
| `*p` | `*p` | Розіменування (unsafe для raw) |
| `a[i]` | `a[i]` | Індексація |
| `a.b` | `a.b` | Доступ до поля |
| `a->b` | `a.b` | Автоматичний deref |
| `,` | `;` або `{ ...; ... }` | Послідовність |

### В.3.7. Керування потоком

| C | Rust | Примітки |
|---|------|----------|
| `if (cond) { }` | `if cond { }` | Без дужок навколо умови |
| `if (cond) { } else { }` | `if cond { } else { }` | Однаково |
| `else if` | `else if` | Однаково |
| `switch (x) { case 1: ... }` | `match x { 1 => ... }` | Pattern matching |
| `case 1: case 2:` | `1 \| 2 =>` | Об'єднання варіантів |
| `default:` | `_ =>` | Wildcard pattern |
| `break;` (у switch) | Автоматично | Немає fall-through |
| `for (i=0; i<n; i++)` | `for i in 0..n` | Range iterator |
| `for (;;)` | `loop { }` | Нескінченний цикл |
| `while (cond)` | `while cond { }` | Однаково |
| `do { } while (cond)` | `loop { ... if !cond { break; } }` | Немає прямого аналога |
| `break` | `break` | Вихід з циклу |
| `continue` | `continue` | Наступна ітерація |
| `return x;` | `return x;` або просто `x` | Expression-based |
| `goto label;` | Немає | Використовуйте loop labels |
| `label:` | `'label: loop { }` | Мітки для циклів |

### В.3.8. Препроцесор → Rust аналоги

| C препроцесор | Rust | Примітки |
|---------------|------|----------|
| `#include "file.h"` | `mod file;` або `use crate::file;` | Модульна система |
| `#include <stdio.h>` | `use std::io;` | Стандартна бібліотека |
| `#define CONST 42` | `const CONST: i32 = 42;` | Типізована константа |
| `#define MACRO(x) ((x)*2)` | `fn macro_fn(x: i32) -> i32` | Функція або macro_rules! |
| `#define MACRO(x) ...` | `macro_rules! macro_name { }` | Декларативний макрос |
| `#ifdef DEBUG` | `#[cfg(debug_assertions)]` | Умовна компіляція |
| `#ifndef HEADER_H` | Не потрібно | Модулі не дублюються |
| `#if PLATFORM == 1` | `#[cfg(target_os = "linux")]` | Platform-specific |
| `#pragma once` | Не потрібно | Автоматично |
| `#error "msg"` | `compile_error!("msg")` | Помилка компіляції |
| `__FILE__` | `file!()` | Ім'я файлу |
| `__LINE__` | `line!()` | Номер рядка |
| `__func__` | Немає стандартного | Можна через макрос |
| `assert(cond)` | `assert!(cond)` або `debug_assert!` | Assertion |
| `static_assert` | `const _: () = assert!(...)` | Compile-time assert |

### В.3.9. Керування пам'яттю

| C | Rust | Примітки |
|---|------|----------|
| `malloc(size)` | `Box::new(value)` | Один об'єкт |
| `malloc(n * sizeof(T))` | `Vec::with_capacity(n)` | Масив |
| `calloc(n, size)` | `vec![0; n]` | Ініціалізований нулями |
| `realloc(ptr, size)` | `vec.reserve(additional)` | Зміна розміру |
| `free(ptr)` | Автоматично (Drop) | RAII |
| `alloca(size)` | Stack array або `Vec` | Немає прямого аналога |
| Stack allocation | `let x = T { };` | За замовчуванням на стеку |
| Heap allocation | `let x = Box::new(T { });` | Явно на heap |

### В.3.10. Обробка помилок

| C | Rust | Примітки |
|---|------|----------|
| `return -1;` (error code) | `return Err(Error::...);` | Result type |
| `return NULL;` (not found) | `return None;` | Option type |
| `if (ptr == NULL)` | `if let None = opt` або `opt.is_none()` | Pattern matching |
| `errno` | `std::io::Error::last_os_error()` | OS помилки |
| `perror("msg")` | `eprintln!("{}", err)` | Вивід помилки |
| `exit(1)` | `std::process::exit(1)` | Завершення програми |
| `abort()` | `std::process::abort()` | Аварійне завершення |
| `setjmp/longjmp` | `panic!/catch_unwind` | Не рекомендується |

### В.3.11. Багатопотоковість

| C (pthreads) | Rust | Примітки |
|--------------|------|----------|
| `pthread_create` | `std::thread::spawn` | Створення потоку |
| `pthread_join` | `handle.join()` | Очікування завершення |
| `pthread_mutex_t` | `std::sync::Mutex<T>` | Mutex із даними |
| `pthread_mutex_lock` | `mutex.lock()` | Автоматичний unlock |
| `pthread_mutex_unlock` | Автоматично (Drop) | RAII guard |
| `pthread_rwlock_t` | `std::sync::RwLock<T>` | Read-write lock |
| `pthread_cond_t` | `std::sync::Condvar` | Condition variable |
| `pthread_key_t` | `thread_local! { }` | Thread-local storage |
| `volatile` | `std::sync::atomic::*` | Atomic types |
| `_Atomic int` | `AtomicI32` | Atomic integer |

### В.3.12. Введення/виведення

| C | Rust | Примітки |
|---|------|----------|
| `printf("fmt", ...)` | `print!("fmt", ...)` | Без newline |
| `printf("fmt\n", ...)` | `println!("fmt", ...)` | З newline |
| `fprintf(stderr, ...)` | `eprint!` / `eprintln!` | До stderr |
| `sprintf(buf, ...)` | `format!("fmt", ...)` | До String |
| `snprintf` | `write!(&mut buf, ...)` | До буфера |
| `scanf` | `std::io::stdin().read_line()` | Читання рядка |
| `fopen` | `std::fs::File::open()` | Відкриття файлу |
| `fclose` | Автоматично (Drop) | RAII |
| `fread` | `file.read(&mut buf)` | Читання |
| `fwrite` | `file.write(&buf)` | Запис |
| `fgets` | `BufRead::read_line()` | Читання рядка |
| `fputs` | `write!` / `writeln!` | Запис рядка |
| `fseek` | `file.seek(SeekFrom::...)` | Позиціонування |
| `ftell` | `file.stream_position()` | Поточна позиція |
| `feof` | Перевірка `read() == 0` | Кінець файлу |
| `ferror` | `Result::Err` | Помилка читання |

### В.3.13. Приклад комплексної міграції

**C-код:**

```c
// C: Типова функція обробки даних БПЛА
typedef struct {
    float x, y, z;
} Vec3;

typedef struct {
    int id;
    Vec3 position;
    float battery;
    int* sensor_data;
    size_t sensor_count;
} Drone;

Drone* create_drone(int id, size_t num_sensors) {
    Drone* d = (Drone*)malloc(sizeof(Drone));
    if (d == NULL) return NULL;
    
    d->id = id;
    d->position = (Vec3){0.0f, 0.0f, 0.0f};
    d->battery = 100.0f;
    d->sensor_count = num_sensors;
    d->sensor_data = (int*)calloc(num_sensors, sizeof(int));
    
    if (d->sensor_data == NULL) {
        free(d);
        return NULL;
    }
    
    return d;
}

void destroy_drone(Drone* d) {
    if (d != NULL) {
        free(d->sensor_data);
        free(d);
    }
}

float get_distance(const Drone* a, const Drone* b) {
    if (a == NULL || b == NULL) return -1.0f;
    
    float dx = a->position.x - b->position.x;
    float dy = a->position.y - b->position.y;
    float dz = a->position.z - b->position.z;
    
    return sqrtf(dx*dx + dy*dy + dz*dz);
}

int update_sensor(Drone* d, size_t index, int value) {
    if (d == NULL || index >= d->sensor_count) {
        return -1;
    }
    d->sensor_data[index] = value;
    return 0;
}
```

**Еквівалентний Rust-код:**

```rust
/// 3D вектор
#[derive(Debug, Clone, Copy, Default)]
pub struct Vec3 {
    pub x: f32,
    pub y: f32,
    pub z: f32,
}

impl Vec3 {
    pub fn new(x: f32, y: f32, z: f32) -> Self {
        Self { x, y, z }
    }
    
    pub fn distance_to(&self, other: &Vec3) -> f32 {
        let dx = self.x - other.x;
        let dy = self.y - other.y;
        let dz = self.z - other.z;
        (dx*dx + dy*dy + dz*dz).sqrt()
    }
}

/// Помилки роботи з дроном
#[derive(Debug, thiserror::Error)]
pub enum DroneError {
    #[error("Sensor index {0} out of bounds (max: {1})")]
    SensorIndexOutOfBounds(usize, usize),
}

/// Дрон із автоматичним керуванням пам'яттю
#[derive(Debug)]
pub struct Drone {
    pub id: i32,
    pub position: Vec3,
    pub battery: f32,
    sensor_data: Vec<i32>,  // Vec замість raw pointer + count
}

impl Drone {
    /// Створює новий дрон із заданою кількістю сенсорів.
    /// 
    /// # Приклади
    /// ```
    /// let drone = Drone::new(1, 8);
    /// assert_eq!(drone.sensor_count(), 8);
    /// ```rust
    pub fn new(id: i32, num_sensors: usize) -> Self {
        Self {
            id,
            position: Vec3::default(),
            battery: 100.0,
            sensor_data: vec![0; num_sensors],  // calloc еквівалент
        }
    }
    
    /// Кількість сенсорів.
    pub fn sensor_count(&self) -> usize {
        self.sensor_data.len()
    }
    
    /// Оновлює значення сенсора.
    pub fn update_sensor(&mut self, index: usize, value: i32) -> Result<(), DroneError> {
        let sensor = self.sensor_data
            .get_mut(index)
            .ok_or(DroneError::SensorIndexOutOfBounds(index, self.sensor_data.len()))?;
        
        *sensor = value;
        Ok(())
    }
    
    /// Отримує значення сенсора.
    pub fn get_sensor(&self, index: usize) -> Option<i32> {
        self.sensor_data.get(index).copied()
    }
    
    /// Відстань до іншого дрона.
    pub fn distance_to(&self, other: &Drone) -> f32 {
        self.position.distance_to(&other.position)
    }
}

// Drop реалізовано автоматично — пам'ять звільняється
// destroy_drone не потрібен!

// Використання
fn main() {
    // Створення — завжди успішне (panic при OOM, не NULL)
    let mut drone = Drone::new(1, 8);
    
    // Оновлення — з обробкою помилок
    match drone.update_sensor(0, 42) {
        Ok(()) => println!("Sensor updated"),
        Err(e) => eprintln!("Error: {}", e),
    }
    
    // Або з ? оператором
    drone.update_sensor(0, 42)?;
    
    // Читання з перевіркою
    if let Some(value) = drone.get_sensor(0) {
        println!("Sensor 0: {}", value);
    }
    
    // Відстань
    let other = Drone::new(2, 4);
    println!("Distance: {}", drone.distance_to(&other));
    
    // Пам'ять звільняється автоматично тут
}
```

**Ключові відмінності:**

| Аспект | C | Rust |
|--------|---|------|
| Алокація | `malloc` + перевірка NULL | `new()` завжди успішний |
| Звільнення | Явний `free()` | Автоматичний Drop |
| Помилки | Return codes (-1, NULL) | `Result<T, E>`, `Option<T>` |
| Bounds check | Ручна перевірка | Автоматична (паніка) |
| Документація | Коментарі | Doc comments + приклади |
| Null safety | Ручні перевірки | Компілятор гарантує |

---

## В.4. Практичні патерни міграції

### В.4.1. Вказівники → Посилання та Option

**C — типовий код БПЛА:**

```c
// C: Пошук найближчого дрона
Drone* find_nearest_drone(Drone* drones, int count, Position* target) {
    if (drones == NULL || target == NULL || count <= 0) {
        return NULL;
    }
    
    Drone* nearest = NULL;
    float min_distance = FLT_MAX;
    
    for (int i = 0; i < count; i++) {
        float dist = calculate_distance(&drones[i].position, target);
        if (dist < min_distance) {
            min_distance = dist;
            nearest = &drones[i];
        }
    }
    
    return nearest;  // Може бути NULL
}

// Використання (НЕБЕЗПЕЧНО!)
Drone* drone = find_nearest_drone(swarm, swarm_size, &target);
printf("Nearest: %s\n", drone->id);  // Потенційний NULL dereference!
```

**Проблеми C-версії:**
- Компілятор не перевіряє, чи обробляється NULL
- Легко забути перевірку
- Немає інформації про lifetime вказівника

**Rust — безпечна версія:**

```rust
/// Пошук найближчого дрона до цільової позиції.
/// 
/// Повертає `Some(&Drone)` якщо знайдено, `None` якщо список порожній.
fn find_nearest_drone<'a>(
    drones: &'a [Drone],
    target: &Position,
) -> Option<&'a Drone> {
    drones
        .iter()
        .min_by(|a, b| {
            let dist_a = a.position.distance_to(target);
            let dist_b = b.position.distance_to(target);
            dist_a.partial_cmp(&dist_b).unwrap_or(std::cmp::Ordering::Equal)
        })
}

// Використання (БЕЗПЕЧНО!)
match find_nearest_drone(&swarm, &target) {
    Some(drone) => println!("Nearest: {}", drone.id),
    None => println!("No drones in swarm"),
}

// Або компактніше:
if let Some(drone) = find_nearest_drone(&swarm, &target) {
    drone.execute_command(Command::Approach(target.clone()));
}
```

**Що змінилося:**
- `Option<&Drone>` явно виражає можливість відсутності результату
- Компілятор **змушує** обробити обидва випадки
- Lifetime `'a` гарантує, що посилання валідне
- Слайс `&[Drone]` замість вказівника + розмір

### В.4.2. malloc/free → Box, Vec, власні типи

**C — динамічна пам'ять:**

```c
// C: Створення маршруту
Waypoint* create_route(int num_points) {
    Waypoint* route = malloc(num_points * sizeof(Waypoint));
    if (route == NULL) {
        return NULL;  // Помилка алокації
    }
    
    for (int i = 0; i < num_points; i++) {
        route[i].reached = false;
        route[i].position = (Position){0, 0, 0};
    }
    
    return route;
}

void process_mission(Drone* drone) {
    Waypoint* route = create_route(10);
    if (route == NULL) {
        log_error("Failed to allocate route");
        return;
    }
    
    // ... використання route ...
    
    free(route);  // Легко забути!
}

// НЕБЕЗПЕЧНО: подвійне звільнення
void dangerous_code() {
    Waypoint* route = create_route(5);
    Waypoint* alias = route;
    
    free(route);
    free(alias);  // Double free! Undefined behavior
}
```

**Rust — автоматичне керування пам'яттю:**

```rust
/// Точка маршруту
#[derive(Debug, Clone, Default)]
struct Waypoint {
    position: Position,
    reached: bool,
}

/// Створення маршруту із заданою кількістю точок.
fn create_route(num_points: usize) -> Vec<Waypoint> {
    vec![Waypoint::default(); num_points]
}

fn process_mission(drone: &mut Drone) {
    let route = create_route(10);
    
    for waypoint in &route {
        drone.navigate_to(&waypoint.position);
    }
    
    // Пам'ять звільняється автоматично при виході з функції
}

// НЕМОЖЛИВО: подвійне звільнення
fn safe_code() {
    let route = create_route(5);
    let alias = route;  // route переміщено в alias
    
    // route більше не існує — компілятор не дозволить використати
    // println!("{:?}", route);  // ПОМИЛКА КОМПІЛЯЦІЇ!
    
    println!("{:?}", alias);  // OK
    // alias звільняється один раз при виході
}
```

**Ключові відмінності:**
- `Vec<T>` автоматично керує пам'яттю
- Move semantics запобігають подвійному звільненню
- Неможливо забути `free()` — RAII гарантує очищення
- Немає NULL — `Vec::new()` завжди успішний

### В.4.3. Глобальні змінні → Явна передача стану

**C — глобальний стан (антипатерн):**

```c
// C: Типовий legacy-код
static SwarmState g_swarm_state;
static bool g_initialized = false;
static pthread_mutex_t g_mutex = PTHREAD_MUTEX_INITIALIZER;

void init_swarm(int num_drones) {
    pthread_mutex_lock(&g_mutex);
    g_swarm_state.num_drones = num_drones;
    g_swarm_state.drones = malloc(num_drones * sizeof(Drone));
    g_initialized = true;
    pthread_mutex_unlock(&g_mutex);
}

Drone* get_drone(int id) {
    // Хто гарантує, що init_swarm викликано?
    // Хто гарантує thread-safety?
    if (!g_initialized || id >= g_swarm_state.num_drones) {
        return NULL;
    }
    return &g_swarm_state.drones[id];
}

void update_drone_position(int id, Position pos) {
    // Data race можливий!
    Drone* drone = get_drone(id);
    if (drone) {
        drone->position = pos;
    }
}
```

**Rust — явний стан із гарантіями:**

```rust
use std::sync::{Arc, RwLock};

/// Стан рою з thread-safe доступом
pub struct SwarmState {
    drones: Vec<Drone>,
}

impl SwarmState {
    /// Створює новий рій із заданою кількістю дронів.
    pub fn new(num_drones: usize) -> Self {
        Self {
            drones: (0..num_drones)
                .map(|id| Drone::new(id))
                .collect(),
        }
    }
    
    /// Отримує незмінне посилання на дрон.
    pub fn get_drone(&self, id: usize) -> Option<&Drone> {
        self.drones.get(id)
    }
    
    /// Отримує змінне посилання на дрон.
    pub fn get_drone_mut(&mut self, id: usize) -> Option<&mut Drone> {
        self.drones.get_mut(id)
    }
}

/// Thread-safe обгортка для рою
pub struct SharedSwarm {
    inner: Arc<RwLock<SwarmState>>,
}

impl SharedSwarm {
    pub fn new(num_drones: usize) -> Self {
        Self {
            inner: Arc::new(RwLock::new(SwarmState::new(num_drones))),
        }
    }
    
    /// Оновлює позицію дрона (thread-safe).
    pub fn update_drone_position(&self, id: usize, pos: Position) -> Result<(), SwarmError> {
        let mut state = self.inner.write()
            .map_err(|_| SwarmError::LockPoisoned)?;
        
        let drone = state.get_drone_mut(id)
            .ok_or(SwarmError::DroneNotFound(id))?;
        
        drone.position = pos;
        Ok(())
    }
    
    /// Читає позиції всіх дронів (thread-safe).
    pub fn get_all_positions(&self) -> Result<Vec<Position>, SwarmError> {
        let state = self.inner.read()
            .map_err(|_| SwarmError::LockPoisoned)?;
        
        Ok(state.drones.iter().map(|d| d.position.clone()).collect())
    }
}

// Використання
fn main() {
    let swarm = SharedSwarm::new(10);
    
    // Можна безпечно клонувати для передачі в інші потоки
    let swarm_clone = swarm.inner.clone();
    
    std::thread::spawn(move || {
        // Thread-safe доступ гарантований типами
        let positions = swarm_clone.read().unwrap();
        // ...
    });
}
```

**Переваги Rust-підходу:**
- Стан явно передається, а не глобальний
- `Arc<RwLock<T>>` гарантує thread-safety на рівні типів
- Компілятор перевіряє коректність доступу
- Неможливо забути про синхронізацію

### В.4.4. Callback-функції → Замикання та trait-об'єкти

**C — callbacks для подій:**

```c
// C: Типовий callback-механізм
typedef void (*TelemetryCallback)(Drone* drone, TelemetryData* data, void* user_data);

typedef struct {
    TelemetryCallback callback;
    void* user_data;  // Небезпечний void*
} TelemetrySubscriber;

static TelemetrySubscriber g_subscribers[MAX_SUBSCRIBERS];
static int g_num_subscribers = 0;

void subscribe_telemetry(TelemetryCallback cb, void* user_data) {
    if (g_num_subscribers < MAX_SUBSCRIBERS) {
        g_subscribers[g_num_subscribers].callback = cb;
        g_subscribers[g_num_subscribers].user_data = user_data;
        g_num_subscribers++;
    }
}

void emit_telemetry(Drone* drone, TelemetryData* data) {
    for (int i = 0; i < g_num_subscribers; i++) {
        // user_data може бути невалідним!
        g_subscribers[i].callback(drone, data, g_subscribers[i].user_data);
    }
}

// Використання — легко помилитися з типами
void my_handler(Drone* drone, TelemetryData* data, void* user_data) {
    int* counter = (int*)user_data;  // Небезпечний cast
    (*counter)++;
}
```

**Rust — типобезпечні замикання:**

```rust
use std::sync::{Arc, Mutex};

/// Тип для обробника телеметрії
pub type TelemetryHandler = Box<dyn Fn(&Drone, &TelemetryData) + Send + Sync>;

/// Менеджер підписок на телеметрію
pub struct TelemetryDispatcher {
    subscribers: Vec<TelemetryHandler>,
}

impl TelemetryDispatcher {
    pub fn new() -> Self {
        Self { subscribers: Vec::new() }
    }
    
    /// Підписка на телеметрію з типобезпечним замиканням.
    pub fn subscribe<F>(&mut self, handler: F)
    where
        F: Fn(&Drone, &TelemetryData) + Send + Sync + 'static,
    {
        self.subscribers.push(Box::new(handler));
    }
    
    /// Розсилка телеметрії всім підписникам.
    pub fn emit(&self, drone: &Drone, data: &TelemetryData) {
        for handler in &self.subscribers {
            handler(drone, data);
        }
    }
}

// Використання — типобезпечно та елегантно
fn main() {
    let mut dispatcher = TelemetryDispatcher::new();
    
    // Лічильник у замиканні — типобезпечно!
    let counter = Arc::new(Mutex::new(0u32));
    let counter_clone = counter.clone();
    
    dispatcher.subscribe(move |drone, data| {
        let mut count = counter_clone.lock().unwrap();
        *count += 1;
        println!("Drone {}: altitude = {}", drone.id, data.altitude);
    });
    
    // Ще один обробник
    dispatcher.subscribe(|drone, data| {
        if data.battery_percent < 20 {
            println!("WARNING: Drone {} low battery!", drone.id);
        }
    });
    
    // Використання
    let drone = Drone::new(1);
    let telemetry = TelemetryData { altitude: 100.0, battery_percent: 15 };
    dispatcher.emit(&drone, &telemetry);
    
    println!("Total events: {}", *counter.lock().unwrap());
}
```

**Переваги:**
- Замикання захоплюють контекст типобезпечно
- Немає `void*` та небезпечних cast'ів
- `Send + Sync` гарантують thread-safety
- Компілятор перевіряє lifetime захоплених змінних

### В.4.5. Масиви фіксованого розміру → Слайси та ітератори

**C — робота з масивами:**

```c
// C: Обробка масиву сенсорів
#define MAX_SENSORS 16

typedef struct {
    SensorReading readings[MAX_SENSORS];
    int num_readings;
} SensorArray;

float calculate_average(SensorArray* arr) {
    if (arr == NULL || arr->num_readings == 0) {
        return 0.0f;
    }
    
    float sum = 0.0f;
    for (int i = 0; i < arr->num_readings; i++) {
        // Потенційний buffer overflow якщо num_readings > MAX_SENSORS
        sum += arr->readings[i].value;
    }
    
    return sum / arr->num_readings;
}

// Фільтрація — складно та error-prone
int filter_valid_readings(SensorArray* input, SensorArray* output) {
    output->num_readings = 0;
    
    for (int i = 0; i < input->num_readings; i++) {
        if (input->readings[i].is_valid) {
            if (output->num_readings >= MAX_SENSORS) {
                return -1;  // Overflow
            }
            output->readings[output->num_readings++] = input->readings[i];
        }
    }
    
    return output->num_readings;
}
```

**Rust — ітератори та функціональний стиль:**

```rust
/// Показання сенсора
#[derive(Debug, Clone)]
struct SensorReading {
    value: f32,
    is_valid: bool,
    timestamp: u64,
}

/// Обчислення середнього значення.
fn calculate_average(readings: &[SensorReading]) -> Option<f32> {
    if readings.is_empty() {
        return None;
    }
    
    let sum: f32 = readings.iter().map(|r| r.value).sum();
    Some(sum / readings.len() as f32)
}

/// Фільтрація валідних показань.
fn filter_valid_readings(readings: &[SensorReading]) -> Vec<SensorReading> {
    readings
        .iter()
        .filter(|r| r.is_valid)
        .cloned()
        .collect()
}

/// Розширена обробка з ітераторами
fn process_sensor_data(readings: &[SensorReading]) -> SensorStats {
    let valid_readings: Vec<_> = readings
        .iter()
        .filter(|r| r.is_valid)
        .collect();
    
    let values: Vec<f32> = valid_readings.iter().map(|r| r.value).collect();
    
    SensorStats {
        count: valid_readings.len(),
        average: values.iter().sum::<f32>() / values.len().max(1) as f32,
        min: values.iter().cloned().fold(f32::INFINITY, f32::min),
        max: values.iter().cloned().fold(f32::NEG_INFINITY, f32::max),
        latest_timestamp: valid_readings.last().map(|r| r.timestamp),
    }
}

#[derive(Debug)]
struct SensorStats {
    count: usize,
    average: f32,
    min: f32,
    max: f32,
    latest_timestamp: Option<u64>,
}

// Використання
fn main() {
    let readings = vec![
        SensorReading { value: 10.0, is_valid: true, timestamp: 1000 },
        SensorReading { value: 0.0, is_valid: false, timestamp: 1001 },
        SensorReading { value: 20.0, is_valid: true, timestamp: 1002 },
        SensorReading { value: 15.0, is_valid: true, timestamp: 1003 },
    ];
    
    // Слайс — немає копіювання
    let stats = process_sensor_data(&readings);
    println!("{:?}", stats);
    
    // Середнє з Option
    if let Some(avg) = calculate_average(&readings) {
        println!("Average: {}", avg);
    }
}
```

**Переваги Rust:**
- Bounds checking автоматичний (паніка при виході за межі)
- Ітератори — zero-cost abstraction
- Функціональний стиль зменшує помилки
- `Option` явно виражає можливість відсутності результату

---

## В.5. FFI: Інтеграція Rust із існуючим C-кодом

### В.5.1. Коли використовувати FFI

FFI (Foreign Function Interface) доцільний, коли:
- Є велика кодова база C, яку неможливо переписати відразу
- Потрібно використовувати C-бібліотеки (MAVLink, FreeRTOS API)
- Критичний до часу код вже оптимізований на C
- Драйвери hardware надаються виробником лише на C

### В.5.2. Виклик C з Rust

**C-бібліотека (mavlink_wrapper.h):**

```c
// mavlink_wrapper.h
#ifndef MAVLINK_WRAPPER_H
#define MAVLINK_WRAPPER_H

#include <stdint.h>
#include <stdbool.h>

typedef struct {
    double latitude;
    double longitude;
    float altitude;
} GPSPosition;

typedef struct {
    uint8_t system_id;
    uint8_t component_id;
    GPSPosition position;
    float battery_voltage;
    uint8_t battery_remaining;
    bool is_armed;
} DroneStatus;

// Ініціалізація MAVLink-з'єднання
int mavlink_init(const char* port, int baudrate);

// Отримання статусу дрона
int mavlink_get_status(uint8_t system_id, DroneStatus* status);

// Надсилання команди
int mavlink_send_goto(uint8_t system_id, GPSPosition* target);

// Завершення з'єднання
void mavlink_close(void);

#endif
```

**Rust-обгортка:**

```rust
//! Безпечна обгортка над C-бібліотекою MAVLink
//!
//! Цей модуль надає типобезпечний інтерфейс до низькорівневої
//! C-реалізації протоколу MAVLink.

use std::ffi::CString;
use std::os::raw::{c_char, c_int};

/// GPS-позиція (відповідає C-структурі)
#[repr(C)]
#[derive(Debug, Clone, Copy, Default)]
pub struct GPSPosition {
    pub latitude: f64,
    pub longitude: f64,
    pub altitude: f32,
}

/// Статус дрона (відповідає C-структурі)
#[repr(C)]
#[derive(Debug, Clone, Copy)]
pub struct DroneStatus {
    pub system_id: u8,
    pub component_id: u8,
    pub position: GPSPosition,
    pub battery_voltage: f32,
    pub battery_remaining: u8,
    pub is_armed: bool,
}

impl Default for DroneStatus {
    fn default() -> Self {
        Self {
            system_id: 0,
            component_id: 0,
            position: GPSPosition::default(),
            battery_voltage: 0.0,
            battery_remaining: 0,
            is_armed: false,
        }
    }
}

// Оголошення зовнішніх C-функцій
extern "C" {
    fn mavlink_init(port: *const c_char, baudrate: c_int) -> c_int;
    fn mavlink_get_status(system_id: u8, status: *mut DroneStatus) -> c_int;
    fn mavlink_send_goto(system_id: u8, target: *const GPSPosition) -> c_int;
    fn mavlink_close();
}

/// Помилки роботи з MAVLink
#[derive(Debug, thiserror::Error)]
pub enum MavlinkError {
    #[error("Не вдалося відкрити порт: {0}")]
    ConnectionFailed(String),
    
    #[error("Помилка отримання статусу для system_id={0}")]
    StatusError(u8),
    
    #[error("Помилка надсилання команди для system_id={0}")]
    CommandError(u8),
    
    #[error("Невалідний рядок порту")]
    InvalidPort,
}

/// Безпечна обгортка над MAVLink-з'єднанням.
///
/// Автоматично закриває з'єднання при виході зі scope (RAII).
pub struct MavlinkConnection {
    _private: (), // Запобігає створенню без init()
}

impl MavlinkConnection {
    /// Створює нове MAVLink-з'єднання.
    ///
    /// # Приклади
    ///
    /// ```no_run
    /// let conn = MavlinkConnection::new("/dev/ttyUSB0", 57600)?;
    /// ```rust
    pub fn new(port: &str, baudrate: i32) -> Result<Self, MavlinkError> {
        let c_port = CString::new(port)
            .map_err(|_| MavlinkError::InvalidPort)?;
        
        // SAFETY: mavlink_init очікує валідний C-рядок,
        // CString гарантує null-termination
        let result = unsafe { mavlink_init(c_port.as_ptr(), baudrate) };
        
        if result < 0 {
            return Err(MavlinkError::ConnectionFailed(port.to_string()));
        }
        
        Ok(Self { _private: () })
    }
    
    /// Отримує статус дрона за його system_id.
    pub fn get_status(&self, system_id: u8) -> Result<DroneStatus, MavlinkError> {
        let mut status = DroneStatus::default();
        
        // SAFETY: status — валідний вказівник на ініціалізовану структуру
        let result = unsafe { mavlink_get_status(system_id, &mut status) };
        
        if result < 0 {
            return Err(MavlinkError::StatusError(system_id));
        }
        
        Ok(status)
    }
    
    /// Надсилає команду переміщення до цільової позиції.
    pub fn send_goto(&self, system_id: u8, target: &GPSPosition) -> Result<(), MavlinkError> {
        // SAFETY: target — валідне посилання, яке передається як const pointer
        let result = unsafe { mavlink_send_goto(system_id, target) };
        
        if result < 0 {
            return Err(MavlinkError::CommandError(system_id));
        }
        
        Ok(())
    }
}

impl Drop for MavlinkConnection {
    fn drop(&mut self) {
        // SAFETY: викликається лише один раз при знищенні об'єкта
        unsafe { mavlink_close() };
    }
}

// Використання
fn main() -> Result<(), Box<dyn std::error::Error>> {
    // RAII: з'єднання закриється автоматично
    let mavlink = MavlinkConnection::new("/dev/ttyUSB0", 57600)?;
    
    // Отримання статусу — типобезпечно
    let status = mavlink.get_status(1)?;
    println!("Drone {} at {:?}", status.system_id, status.position);
    
    // Надсилання команди
    let target = GPSPosition {
        latitude: 50.4501,
        longitude: 30.5234,
        altitude: 100.0,
    };
    mavlink.send_goto(1, &target)?;
    
    Ok(())
    // mavlink_close() викликається автоматично тут
}
```

### В.5.3. Виклик Rust з C

**Rust-бібліотека для C:**

```rust
//! Експорт Rust-функцій для виклику з C
//!
//! Компілюється як статична або динамічна бібліотека.

use std::ffi::{CStr, CString};
use std::os::raw::c_char;
use std::ptr;

/// Результат операції (для C API)
#[repr(C)]
pub struct SwarmResult {
    pub success: bool,
    pub error_code: i32,
    pub message: *mut c_char,
}

impl SwarmResult {
    fn ok() -> Self {
        Self {
            success: true,
            error_code: 0,
            message: ptr::null_mut(),
        }
    }
    
    fn error(code: i32, msg: &str) -> Self {
        let c_msg = CString::new(msg).unwrap_or_default();
        Self {
            success: false,
            error_code: code,
            message: c_msg.into_raw(),
        }
    }
}

/// Внутрішній стан рою (непрозорий для C)
pub struct SwarmController {
    drones: Vec<DroneState>,
    mission_active: bool,
}

#[derive(Default)]
struct DroneState {
    id: u32,
    position: (f64, f64, f32),
    battery: u8,
}

// Глобальний стан (для C API без ООП)
static mut SWARM: Option<SwarmController> = None;

/// Ініціалізація рою.
///
/// # Safety
/// Має викликатися перед будь-якими іншими функціями.
#[no_mangle]
pub unsafe extern "C" fn swarm_init(num_drones: u32) -> SwarmResult {
    let controller = SwarmController {
        drones: (0..num_drones)
            .map(|id| DroneState { id, ..Default::default() })
            .collect(),
        mission_active: false,
    };
    
    SWARM = Some(controller);
    SwarmResult::ok()
}

/// Отримання кількості дронів.
#[no_mangle]
pub unsafe extern "C" fn swarm_get_drone_count() -> i32 {
    match &SWARM {
        Some(swarm) => swarm.drones.len() as i32,
        None => -1,
    }
}

/// Оновлення позиції дрона.
#[no_mangle]
pub unsafe extern "C" fn swarm_update_position(
    drone_id: u32,
    lat: f64,
    lon: f64,
    alt: f32,
) -> SwarmResult {
    let swarm = match &mut SWARM {
        Some(s) => s,
        None => return SwarmResult::error(1, "Swarm not initialized"),
    };
    
    let drone = match swarm.drones.iter_mut().find(|d| d.id == drone_id) {
        Some(d) => d,
        None => return SwarmResult::error(2, "Drone not found"),
    };
    
    drone.position = (lat, lon, alt);
    SwarmResult::ok()
}

/// Запуск місії.
#[no_mangle]
pub unsafe extern "C" fn swarm_start_mission(mission_name: *const c_char) -> SwarmResult {
    let swarm = match &mut SWARM {
        Some(s) => s,
        None => return SwarmResult::error(1, "Swarm not initialized"),
    };
    
    if mission_name.is_null() {
        return SwarmResult::error(3, "Mission name is null");
    }
    
    let name = match CStr::from_ptr(mission_name).to_str() {
        Ok(s) => s,
        Err(_) => return SwarmResult::error(4, "Invalid UTF-8 in mission name"),
    };
    
    println!("Starting mission: {}", name);
    swarm.mission_active = true;
    
    SwarmResult::ok()
}

/// Звільнення пам'яті повідомлення про помилку.
#[no_mangle]
pub unsafe extern "C" fn swarm_free_message(msg: *mut c_char) {
    if !msg.is_null() {
        drop(CString::from_raw(msg));
    }
}

/// Завершення роботи з роєм.
#[no_mangle]
pub unsafe extern "C" fn swarm_shutdown() {
    SWARM = None;
}
```

**C-header (генерується або пишеться вручну):**

```c
// swarm_controller.h
#ifndef SWARM_CONTROLLER_H
#define SWARM_CONTROLLER_H

#include <stdint.h>
#include <stdbool.h>

typedef struct {
    bool success;
    int32_t error_code;
    char* message;
} SwarmResult;

// Ініціалізація
SwarmResult swarm_init(uint32_t num_drones);

// Отримання кількості дронів
int32_t swarm_get_drone_count(void);

// Оновлення позиції
SwarmResult swarm_update_position(uint32_t drone_id, double lat, double lon, float alt);

// Запуск місії
SwarmResult swarm_start_mission(const char* mission_name);

// Звільнення повідомлення
void swarm_free_message(char* msg);

// Завершення
void swarm_shutdown(void);

#endif
```

**Використання в C:**

```c
#include "swarm_controller.h"
#include <stdio.h>

int main() {
    SwarmResult result = swarm_init(10);
    if (!result.success) {
        printf("Error: %s\n", result.message);
        swarm_free_message(result.message);
        return 1;
    }
    
    printf("Drone count: %d\n", swarm_get_drone_count());
    
    result = swarm_update_position(0, 50.4501, 30.5234, 100.0f);
    if (!result.success) {
        printf("Update error: %s\n", result.message);
        swarm_free_message(result.message);
    }
    
    result = swarm_start_mission("patrol_mission");
    
    swarm_shutdown();
    return 0;
}
```

### В.5.4. Cargo.toml для FFI-бібліотеки

```toml
[package]
name = "swarm_controller"
version = "0.1.0"
edition = "2021"

[lib]
name = "swarm_controller"
# Статична бібліотека для вбудовування
crate-type = ["staticlib", "cdylib"]

[dependencies]
thiserror = "1.0"

[build-dependencies]
cbindgen = "0.26"  # Автогенерація C-header
```

---

## В.6. Embedded Rust: Особливості для мікроконтролерів

### В.6.1. no_std середовище

Для мікроконтролерів (ESP32, STM32, Arduino) стандартна бібліотека недоступна:

```rust
//! Приклад no_std програми для ESP32

#![no_std]  // Без стандартної бібліотеки
#![no_main] // Без стандартного main()

use core::panic::PanicInfo;
use esp_hal::prelude::*;

/// Обробник паніки (обов'язковий для no_std)
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}

/// Точка входу
#[entry]
fn main() -> ! {
    let peripherals = esp_hal::init(esp_hal::Config::default());
    
    let mut led = peripherals.GPIO2.into_push_pull_output();
    let mut delay = esp_hal::delay::Delay::new();
    
    loop {
        led.toggle();
        delay.delay_ms(500u32);
    }
}
```

### В.6.2. Альтернативи std у no_std

| std | no_std альтернатива | Примітка |
|-----|---------------------|----------|
| `std::vec::Vec` | `heapless::Vec` | Фіксований розмір на стеку |
| `std::string::String` | `heapless::String` | Без алокації |
| `std::collections::HashMap` | `heapless::FnvIndexMap` | Статичний розмір |
| `std::sync::Mutex` | `critical_section::Mutex` | Для embedded |
| `std::println!` | `defmt::info!` | Ефективне логування |

```rust
#![no_std]

use heapless::Vec;
use heapless::String;

// Вектор максимум на 8 елементів, без heap
let mut sensors: Vec<u16, 8> = Vec::new();
sensors.push(1024).unwrap();
sensors.push(2048).unwrap();

// Рядок максимум 32 символи
let mut name: String<32> = String::new();
name.push_str("Drone-001").unwrap();
```

### В.6.3. Embedded HAL — абстракція над hardware

```rust
//! Драйвер сенсора, що працює з будь-яким мікроконтролером

use embedded_hal::i2c::I2c;

/// Драйвер барометра BMP280
pub struct Bmp280<I2C> {
    i2c: I2C,
    address: u8,
}

impl<I2C, E> Bmp280<I2C>
where
    I2C: I2c<Error = E>,
{
    pub fn new(i2c: I2C, address: u8) -> Self {
        Self { i2c, address }
    }
    
    pub fn read_pressure(&mut self) -> Result<f32, E> {
        let mut buffer = [0u8; 3];
        self.i2c.write_read(self.address, &[0xF7], &mut buffer)?;
        
        let raw = ((buffer[0] as u32) << 12)
            | ((buffer[1] as u32) << 4)
            | ((buffer[2] as u32) >> 4);
        
        // Спрощена конвертація
        Ok(raw as f32 / 256.0)
    }
    
    pub fn read_altitude(&mut self) -> Result<f32, E> {
        let pressure = self.read_pressure()?;
        // Барометрична формула
        Ok(44330.0 * (1.0 - (pressure / 1013.25_f32).powf(0.1903)))
    }
}

// Використання на ESP32
fn example_esp32() {
    let i2c = esp_hal::i2c::I2C::new(/* ... */);
    let mut baro = Bmp280::new(i2c, 0x76);
    
    if let Ok(alt) = baro.read_altitude() {
        defmt::info!("Altitude: {} m", alt);
    }
}

// Той самий код на STM32
fn example_stm32() {
    let i2c = stm32_hal::i2c::I2c::new(/* ... */);
    let mut baro = Bmp280::new(i2c, 0x76);
    
    if let Ok(alt) = baro.read_altitude() {
        defmt::info!("Altitude: {} m", alt);
    }
}
```

---

## В.7. Типові проблеми міграції та їх вирішення

### В.7.1. Проблема: Циклічні залежності даних

**C-код із циклічними посиланнями:**

```c
// C: Двозв'язний список дронів у формації
typedef struct FormationNode {
    Drone* drone;
    struct FormationNode* next;
    struct FormationNode* prev;  // Циклічне посилання
} FormationNode;
```

**Rust — рішення через індекси:**

```rust
/// Формація як arena-based структура
pub struct Formation {
    nodes: Vec<FormationNode>,
}

struct FormationNode {
    drone_id: usize,
    next: Option<usize>,  // Індекс замість посилання
    prev: Option<usize>,
}

impl Formation {
    pub fn new() -> Self {
        Self { nodes: Vec::new() }
    }
    
    pub fn add_drone(&mut self, drone_id: usize) -> usize {
        let index = self.nodes.len();
        
        // Зв'язуємо з останнім елементом
        let prev_index = if index > 0 {
            let last = index - 1;
            self.nodes[last].next = Some(index);
            Some(last)
        } else {
            None
        };
        
        self.nodes.push(FormationNode {
            drone_id,
            next: None,
            prev: prev_index,
        });
        
        index
    }
    
    pub fn iter(&self) -> impl Iterator<Item = usize> + '_ {
        self.nodes.iter().map(|n| n.drone_id)
    }
}
```

### В.7.2. Проблема: Глибоке вкладення вказівників

**C-код:**

```c
// C: Drone*** — три рівні вказівників
Drone*** allocate_3d_swarm(int x, int y, int z);
```

**Rust — використання вкладених Vec:**

```rust
/// 3D-сітка дронів
pub struct Swarm3D {
    grid: Vec<Vec<Vec<Option<Drone>>>>,
    dimensions: (usize, usize, usize),
}

impl Swarm3D {
    pub fn new(x: usize, y: usize, z: usize) -> Self {
        Self {
            grid: vec![vec![vec![None; z]; y]; x],
            dimensions: (x, y, z),
        }
    }
    
    pub fn get(&self, x: usize, y: usize, z: usize) -> Option<&Drone> {
        self.grid.get(x)?.get(y)?.get(z)?.as_ref()
    }
    
    pub fn set(&mut self, x: usize, y: usize, z: usize, drone: Drone) {
        if let Some(cell) = self.grid
            .get_mut(x)
            .and_then(|plane| plane.get_mut(y))
            .and_then(|row| row.get_mut(z))
        {
            *cell = Some(drone);
        }
    }
}
```

### В.7.3. Проблема: Union для різних типів даних

**C-код:**

```c
// C: Повідомлення різних типів
typedef enum { MSG_POSITION, MSG_COMMAND, MSG_TELEMETRY } MessageType;

typedef union {
    Position position;
    Command command;
    Telemetry telemetry;
} MessageData;

typedef struct {
    MessageType type;
    MessageData data;
} Message;
```

**Rust — enum із даними:**

```rust
/// Повідомлення з типобезпечними варіантами
#[derive(Debug, Clone)]
pub enum Message {
    Position(Position),
    Command(Command),
    Telemetry(Telemetry),
}

impl Message {
    pub fn as_position(&self) -> Option<&Position> {
        match self {
            Message::Position(p) => Some(p),
            _ => None,
        }
    }
    
    pub fn message_type(&self) -> &'static str {
        match self {
            Message::Position(_) => "position",
            Message::Command(_) => "command",
            Message::Telemetry(_) => "telemetry",
        }
    }
}

// Використання — компілятор гарантує обробку всіх варіантів
fn handle_message(msg: Message) {
    match msg {
        Message::Position(pos) => {
            println!("Position: {:?}", pos);
        }
        Message::Command(cmd) => {
            execute_command(cmd);
        }
        Message::Telemetry(tel) => {
            log_telemetry(tel);
        }
    }
}
```

### В.7.4. Проблема: Макроси препроцесора

**C-код:**

```c
// C: Макроси для конфігурації
#define MAX_DRONES 100
#define ENABLE_LOGGING 1

#if ENABLE_LOGGING
    #define LOG(fmt, ...) printf("[LOG] " fmt "\n", ##__VA_ARGS__)
#else
    #define LOG(fmt, ...)
#endif

#define CLAMP(x, min, max) ((x) < (min) ? (min) : ((x) > (max) ? (max) : (x)))
```

**Rust — константи, feature flags, функції:**

```rust
/// Конфігурація (константи замість #define)
pub const MAX_DRONES: usize = 100;

/// Умовна компіляція через feature flags (Cargo.toml)
#[cfg(feature = "logging")]
macro_rules! log {
    ($($arg:tt)*) => {
        println!("[LOG] {}", format!($($arg)*));
    };
}

#[cfg(not(feature = "logging"))]
macro_rules! log {
    ($($arg:tt)*) => {};
}

/// Generics замість макро-функцій
fn clamp<T: PartialOrd>(value: T, min: T, max: T) -> T {
    if value < min {
        min
    } else if value > max {
        max
    } else {
        value
    }
}

// Або використання std (Rust 1.50+)
fn clamp_std(value: f32, min: f32, max: f32) -> f32 {
    value.clamp(min, max)
}
```

---

## В.8. Чек-лист міграції

### В.8.1. Перед початком

- [ ] Визначити scope міграції (весь проєкт / критичні модулі)
- [ ] Створити тести для існуючого C-коду
- [ ] Налаштувати CI/CD для обох мов
- [ ] Обрати стратегію (повний перепис / поступовий / FFI)

### В.8.2. Аналіз C-коду

- [ ] Ідентифікувати глобальні змінні
- [ ] Знайти всі malloc/free та перевірити парність
- [ ] Виявити callback-механізми
- [ ] Документувати lifetime очікування для вказівників
- [ ] Знайти потенційні data races

### В.8.3. Процес міграції

- [ ] Почати з leaf-модулів (без залежностей)
- [ ] Конвертувати структури даних
- [ ] Замінити вказівники на Option/посилання
- [ ] Замінити malloc/free на Vec/Box
- [ ] Додати обробку помилок через Result
- [ ] Написати unit-тести для кожного модуля

### В.8.4. Інтеграція

- [ ] Створити FFI-обгортки якщо потрібно
- [ ] Перевірити memory layout (#[repr(C)])
- [ ] Протестувати взаємодію C ↔ Rust
- [ ] Профілювати продуктивність
- [ ] Оновити документацію

### В.8.5. Завершення

- [ ] Видалити старий C-код (якщо повна міграція)
- [ ] Оновити build-систему
- [ ] Навчити команду основам Rust
- [ ] Встановити code review процес для Rust

---

## В.9. Ресурси для подальшого вивчення

### Документація

- **The Embedded Rust Book:** https://docs.rust-embedded.org/book/
- **Rust FFI Omnibus:** http://jakegoulding.com/rust-ffi-omnibus/
- **The Rustonomicon (unsafe Rust):** https://doc.rust-lang.org/nomicon/

### Крейти для embedded

| Крейт | Призначення |
|-------|-------------|
| `embedded-hal` | Абстракція над hardware |
| `esp-hal` | HAL для ESP32 |
| `stm32-hal` | HAL для STM32 |
| `heapless` | Колекції без heap |
| `defmt` | Ефективне логування |
| `embassy` | Async для embedded |
| `probe-rs` | Debugging |

### Крейти для БПЛА та робототехніки

| Крейт | Призначення |
|-------|-------------|
| `mavlink` | Протокол MAVLink |
| `rosrust` | ROS інтеграція |
| `nalgebra` | Лінійна алгебра |
| `uom` | Одиниці вимірювання |
| `pid` | PID-контролер |

### Приклади проєктів

- **Drone:** https://github.com/nickel-org/drone
- **PX4 Rust:** https://github.com/PX4/px4-ros2-interface-lib
- **Embedded Rust examples:** https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials

---

## В.10. Підсумки

Міграція з C на Rust — це інвестиція в надійність та безпеку системи. Для розробників БПЛА та IoT ця інвестиція особливо виправдана:

**Що ви отримуєте:**
- Усунення цілих класів помилок (buffer overflow, use-after-free, data races)
- Безпечну багатопотоковість без runtime-overhead
- Сучасний toolchain (Cargo, clippy, rustfmt)
- Активну спільноту та екосистему

**Що потрібно врахувати:**
- Крива навчання для команди
- Час на переписування коду
- Можлива необхідність FFI для legacy-бібліотек
- Обмежена підтримка деяких платформ (покращується)

**Рекомендація:** Починайте з некритичних модулів, набувайте досвіду, поступово розширюйте scope міграції. Rust — це не лише мова, це новий спосіб мислення про безпеку коду.

---

*"Помилка в польотному контролері — це не баг у JIRA. Це падіння апарату."*

*Rust допомагає писати код, якому можна довіряти.*
