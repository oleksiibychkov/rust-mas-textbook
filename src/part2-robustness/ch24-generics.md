# Розділ 24: Узагальнення — параметричні типи

## Вступ

Уявіть, що вам потрібна функція пошуку мінімального елемента. Для чисел — одна функція `min_i32()`, для рядків — інша `min_string()`, для позицій — ще одна `min_position()`. Три функції з майже ідентичним кодом? Це дублювання, яке важко підтримувати.

**Узагальнення** (generics) дозволяють написати **одну** функцію, що працює з **будь-яким** типом:

```rust
// Замість трьох функцій — одна
fn min<T: PartialOrd>(a: T, b: T) -> T {
    if a < b { a } else { b }
}
```

Параметр `T` — це **параметр типу**. Він означає "будь-який тип". Компілятор згенерує конкретні версії для кожного типу, з яким функція використовується — `min::<i32>()`, `min::<String>()`, `min::<Position>()`. Цей процес називається **мономорфізація**.

Головна перевага: **zero-cost abstraction**. Generic код працює так само швидко, як код, написаний вручну для кожного типу, бо компілятор генерує оптимізований код на етапі компіляції.

---

## 24.1 Generic функції

### Базовий синтаксис

Синтаксис узагальненої функції: `fn name<T>(arg: T) -> T`. Параметр типу `T` оголошується в кутових дужках `<>` після імені функції.

**Постановка задачі:** Створити функцію `identity()`, що повертає свій аргумент без змін. Функція має працювати з будь-яким типом.

```rust
// <T> — оголошення параметра типу
// T використовується як звичайний тип у сигнатурі
fn identity<T>(value: T) -> T {
    value
}

fn main() {
    // Компілятор виводить тип з контексту
    let x = identity(42);            // T = i32
    let y = identity("hello");       // T = &str
    let z = identity(vec![1, 2, 3]); // T = Vec<i32>
    
    println!("x = {}, y = {}, z = {:?}", x, y, z);
    // x = 42, y = hello, z = [1, 2, 3]
}
```

**Як це працює:**

1. `<T>` оголошує параметр типу з ім'ям `T` (можна будь-яке ім'я, але `T` — конвенція)
2. `value: T` — параметр має тип `T`
3. `-> T` — повертаємо тип `T`
4. При виклику `identity(42)` компілятор виводить `T = i32`
5. Генерується конкретна функція `identity_i32(value: i32) -> i32`

### Turbofish — явне вказання типу

Іноді компілятор не може вивести тип з контексту. Тоді потрібно вказати його явно через синтаксис **turbofish** `::<>`.

**Постановка задачі:** Функція `create_default()` створює значення за замовчуванням. Без контексту компілятор не знає, який тип створювати.

```rust
fn create_default<T: Default>() -> T {
    T::default()
}

fn main() {
    // ❌ Помилка: cannot infer type
    // let mystery = create_default();
    
    // ✅ Варіант 1: type annotation
    let num: i32 = create_default();
    
    // ✅ Варіант 2: turbofish ::<>
    let string = create_default::<String>();
    let vec = create_default::<Vec<u8>>();
    
    println!("num: {}", num);         // 0
    println!("string: '{}'", string); // ''
    println!("vec: {:?}", vec);       // []
}
```

**Коли потрібен turbofish:**
- Функція повертає generic тип, але немає контексту
- `parse::<i32>()` — конвертація рядка в число
- `collect::<Vec<_>>()` — збирання ітератора в колекцію
- `Vec::<i32>::new()` — створення порожнього вектора

### Кілька параметрів типу

Функція може мати кілька параметрів типу, якщо потрібно працювати з різними типами.

**Постановка задачі:** Створити функцію `swap()`, що міняє місцями елементи кортежу з різними типами.

```rust
// T та U — два різних параметри типу
fn swap<T, U>(pair: (T, U)) -> (U, T) {
    (pair.1, pair.0)
}

// Три параметри типу
fn combine<A, B, C>(a: A, b: B, f: fn(A, B) -> C) -> C {
    f(a, b)
}

fn main() {
    let pair = (1, "hello");
    let swapped = swap(pair);
    println!("Swapped: {:?}", swapped);  // ("hello", 1)
    
    let result = combine(10, 20, |a, b| a + b);
    println!("Combined: {}", result);  // 30
    
    // Різні типи
    let mixed = combine("Hello", 5, |s, n| format!("{} {}", s, n));
    println!("Mixed: {}", mixed);  // Hello 5
}
```

**Конвенції імен параметрів типу:**
- `T`, `U`, `V` — загальні типи
- `K`, `V` — ключ і значення (HashMap)
- `E` — тип помилки
- `R` — тип результату

### Generic функції з посиланнями

Часто generic функції працюють з посиланнями, щоб не забирати ownership.

**Постановка задачі:** Створити функцію `find()`, що шукає елемент у слайсі та повертає його індекс.

```rust
// &[T] — слайс будь-якого типу T
// &T — посилання на шуканий елемент
fn find<T: PartialEq>(slice: &[T], target: &T) -> Option<usize> {
    for (i, item) in slice.iter().enumerate() {
        if item == target {
            return Some(i);
        }
    }
    None
}

// Повертає посилання на перший елемент
fn first<T>(slice: &[T]) -> Option<&T> {
    slice.first()
}

fn main() {
    let numbers = vec![1, 2, 3, 4, 5];
    let names = vec!["Alice", "Bob", "Charlie"];
    
    println!("First number: {:?}", first(&numbers));  // Some(1)
    println!("First name: {:?}", first(&names));      // Some("Alice")
    
    println!("Index of 3: {:?}", find(&numbers, &3));      // Some(2)
    println!("Index of Bob: {:?}", find(&names, &"Bob"));  // Some(1)
    println!("Index of 10: {:?}", find(&numbers, &10));    // None
}
```

**Важливо:** `T: PartialEq` означає "T повинен реалізовувати трейт PartialEq". Це **trait bound** — обмеження на параметр типу. Детальніше в Розділі 25.

---

## 24.2 Generic структури

### Базова generic структура

Структура може мати параметр типу для полів.

**Постановка задачі:** Створити структуру `Point<T>`, де координати можуть бути будь-якого числового типу.

```rust
#[derive(Debug)]
struct Point<T> {
    x: T,
    y: T,
}

fn main() {
    // Point з цілими числами
    let int_point = Point { x: 5, y: 10 };
    
    // Point з дробовими
    let float_point = Point { x: 1.5, y: 2.5 };
    
    println!("Int point: {:?}", int_point);     // Point { x: 5, y: 10 }
    println!("Float point: {:?}", float_point); // Point { x: 1.5, y: 2.5 }
    
    // ❌ Помилка: x та y мають бути ОДНОГО типу T
    // let mixed = Point { x: 5, y: 2.5 };
}
```

**Як це працює:**

1. `struct Point<T>` — структура параметризована типом `T`
2. `x: T, y: T` — обидва поля мають тип `T`
3. `Point { x: 5, y: 10 }` — компілятор виводить `T = i32`
4. `Point<i32>` та `Point<f64>` — це **різні типи**

### Структура з кількома параметрами

Якщо поля мають різні типи, потрібно кілька параметрів.

**Постановка задачі:** Структура `Pair<T, U>`, де перший і другий елементи можуть бути різних типів.

```rust
#[derive(Debug)]
struct Pair<T, U> {
    first: T,
    second: U,
}

#[derive(Debug)]
struct KeyValue<K, V> {
    key: K,
    value: V,
}

fn main() {
    // Різні типи для елементів
    let pair = Pair {
        first: 42,
        second: "відповідь",
    };
    
    let kv = KeyValue {
        key: String::from("name"),
        value: String::from("Alice"),
    };
    
    println!("Pair: {:?}", pair);      // Pair { first: 42, second: "відповідь" }
    println!("KeyValue: {:?}", kv);    // KeyValue { key: "name", value: "Alice" }
}
```

### Методи для generic структур

Методи для generic структур визначаються через `impl<T>`.

**Постановка задачі:** Додати методи до `Point<T>` — конструктор, геттери, та спеціальний метод тільки для `f64`.

```rust
#[derive(Debug, Clone, Copy)]
struct Point<T> {
    x: T,
    y: T,
}

// Методи для БУДЬ-ЯКОГО T
impl<T> Point<T> {
    // Конструктор
    fn new(x: T, y: T) -> Self {
        Self { x, y }
    }
    
    // Геттери — повертають посилання
    fn x(&self) -> &T {
        &self.x
    }
    
    fn y(&self) -> &T {
        &self.y
    }
}

// Методи ТІЛЬКИ для f64
impl Point<f64> {
    fn distance_from_origin(&self) -> f64 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}

// Методи для типів, що підтримують додавання
impl<T: std::ops::Add<Output = T> + Copy> Point<T> {
    fn add(&self, other: &Point<T>) -> Point<T> {
        Point {
            x: self.x + other.x,
            y: self.y + other.y,
        }
    }
}

fn main() {
    let p1 = Point::new(3.0, 4.0);
    let p2 = Point::new(1.0, 2.0);
    
    println!("p1.x = {}", p1.x());  // 3
    println!("Відстань від початку: {}", p1.distance_from_origin());  // 5
    
    let p3 = p1.add(&p2);
    println!("p1 + p2 = {:?}", p3);  // Point { x: 4.0, y: 6.0 }
    
    // Integer point
    let int_p1 = Point::new(3, 4);
    let int_p2 = Point::new(1, 2);
    
    // ❌ distance_from_origin недоступний для Point<i32>!
    // int_p1.distance_from_origin();
    
    // ✅ Але add доступний (i32 підтримує Add)
    println!("int sum: {:?}", int_p1.add(&int_p2));  // Point { x: 4, y: 6 }
}
```

**Три блоки impl:**

1. `impl<T> Point<T>` — методи для **всіх** типів T
2. `impl Point<f64>` — методи **тільки** для `f64`
3. `impl<T: Add + Copy> Point<T>` — методи для типів, що реалізують потрібні трейти

---

## 24.3 Generic enum

### Option та Result — приклади зі стандартної бібліотеки

Найвідоміші generic enum'и — це `Option<T>` та `Result<T, E>`:

```rust
// Визначення зі std (спрощено)
enum Option<T> {
    Some(T),
    None,
}

enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

`Option<i32>` та `Option<String>` — різні типи, але з однаковою структурою.

### Власний generic enum

**Постановка задачі:** Створити enum `Response<T, E>` для асинхронних операцій з трьома станами: успіх, помилка, очікування.

```rust
#[derive(Debug)]
enum Response<T, E> {
    Success(T),
    Error(E),
    Pending,
}

impl<T, E> Response<T, E> {
    fn is_success(&self) -> bool {
        matches!(self, Response::Success(_))
    }
    
    fn is_error(&self) -> bool {
        matches!(self, Response::Error(_))
    }
    
    fn is_pending(&self) -> bool {
        matches!(self, Response::Pending)
    }
    
    // unwrap вимагає E: Debug для повідомлення про помилку
    fn unwrap(self) -> T
    where
        E: std::fmt::Debug,
    {
        match self {
            Response::Success(value) => value,
            Response::Error(e) => panic!("Called unwrap on Error: {:?}", e),
            Response::Pending => panic!("Called unwrap on Pending"),
        }
    }
    
    // map трансформує значення Success
    fn map<U, F: FnOnce(T) -> U>(self, f: F) -> Response<U, E> {
        match self {
            Response::Success(value) => Response::Success(f(value)),
            Response::Error(e) => Response::Error(e),
            Response::Pending => Response::Pending,
        }
    }
}

fn main() {
    let success: Response<i32, &str> = Response::Success(42);
    let error: Response<i32, &str> = Response::Error("failed");
    let pending: Response<i32, &str> = Response::Pending;
    
    println!("success.is_success(): {}", success.is_success());  // true
    println!("error.is_error(): {}", error.is_error());          // true
    println!("pending.is_pending(): {}", pending.is_pending());  // true
    
    // map трансформує значення
    let doubled = Response::Success(21).map(|x| x * 2);
    println!("doubled: {:?}", doubled);  // Success(42)
    
    // map на Error/Pending не змінює їх
    let still_error = Response::<i32, &str>::Error("oops").map(|x| x * 2);
    println!("still_error: {:?}", still_error);  // Error("oops")
}
```

---

## 24.4 Мономорфізація — як це працює

### Генерація коду для кожного типу

**Мономорфізація** — процес, коли компілятор створює конкретну версію generic коду для кожного типу, з яким він використовується.

```rust
fn print_value<T: std::fmt::Display>(value: T) {
    println!("{}", value);
}

fn main() {
    print_value(42);      // Генерується print_value_i32
    print_value(3.14);    // Генерується print_value_f64
    print_value("hello"); // Генерується print_value_str
}
```

**Що генерує компілятор (концептуально):**

```rust
fn print_value_i32(value: i32) {
    println!("{}", value);
}

fn print_value_f64(value: f64) {
    println!("{}", value);
}

fn print_value_str(value: &str) {
    println!("{}", value);
}
```

### Zero-cost abstraction

Мономорфізація забезпечує **zero-cost abstraction** — generic код працює так само швидко, як код, написаний вручну.

```rust
// Generic версія
fn sum_generic<T>(slice: &[T]) -> T
where
    T: std::ops::Add<Output = T> + Default + Copy,
{
    let mut total = T::default();
    for item in slice {
        total = total + *item;
    }
    total
}

// Вручну для i32
fn sum_i32(slice: &[i32]) -> i32 {
    let mut total = 0;
    for item in slice {
        total = total + *item;
    }
    total
}

fn main() {
    let numbers = vec![1, 2, 3, 4, 5];
    
    // Обидва виклики мають ОДНАКОВУ продуктивність
    println!("Generic sum: {}", sum_generic(&numbers));  // 15
    println!("Concrete sum: {}", sum_i32(&numbers));     // 15
}
```

### Компроміси мономорфізації

| Плюси | Мінуси |
|-------|--------|
| ✅ Немає runtime overhead | ❌ Більший розмір бінарного файлу |
| ✅ Компілятор оптимізує для кожного типу | ❌ Довша компіляція |
| ✅ Можливий inlining | ❌ Код дублюється в бінарнику |

**Приклад:** Якщо функція використовується з 10 різними типами — в бінарнику буде 10 копій коду.

---

## 24.5 Const generics — параметри-константи

### Розмір як параметр типу

Крім типів, можна параметризувати **константами**. Це корисно для масивів фіксованого розміру.

**Постановка задачі:** Створити буфер фіксованого розміру, де розмір визначається на етапі компіляції.

```rust
#[derive(Debug)]
struct FixedBuffer<T, const N: usize> {
    data: [T; N],  // Масив розміру N
    len: usize,
}

impl<T: Default + Copy, const N: usize> FixedBuffer<T, N> {
    fn new() -> Self {
        Self {
            data: [T::default(); N],
            len: 0,
        }
    }
    
    fn push(&mut self, item: T) -> Result<(), &'static str> {
        if self.len >= N {
            return Err("Буфер заповнений");
        }
        self.data[self.len] = item;
        self.len += 1;
        Ok(())
    }
    
    fn get(&self, index: usize) -> Option<&T> {
        if index < self.len {
            Some(&self.data[index])
        } else {
            None
        }
    }
    
    fn capacity(&self) -> usize {
        N
    }
    
    fn len(&self) -> usize {
        self.len
    }
}

fn main() {
    // Буфер на 5 елементів
    let mut small: FixedBuffer<i32, 5> = FixedBuffer::new();
    small.push(1).unwrap();
    small.push(2).unwrap();
    println!("Small (capacity {}): {:?}", small.capacity(), small.data);
    // Small (capacity 5): [1, 2, 0, 0, 0]
    
    // Буфер на 10 елементів
    let mut large: FixedBuffer<f64, 10> = FixedBuffer::new();
    large.push(3.14).unwrap();
    println!("Large (capacity {}): len = {}", large.capacity(), large.len());
    // Large (capacity 10): len = 1
}
```

**Як це працює:**

1. `const N: usize` — параметр-константа типу `usize`
2. `[T; N]` — масив розміру N, відомого на етапі компіляції
3. `FixedBuffer<i32, 5>` та `FixedBuffer<i32, 10>` — **різні типи**
4. Розмір масиву "вбудований" у тип

### Практичне застосування: матриці

```rust
#[derive(Debug, Clone, Copy)]
struct Matrix<T, const ROWS: usize, const COLS: usize> {
    data: [[T; COLS]; ROWS],
}

impl<T: Default + Copy, const ROWS: usize, const COLS: usize> Matrix<T, ROWS, COLS> {
    fn new() -> Self {
        Self {
            data: [[T::default(); COLS]; ROWS],
        }
    }
    
    fn from_array(data: [[T; COLS]; ROWS]) -> Self {
        Self { data }
    }
    
    fn rows(&self) -> usize { ROWS }
    fn cols(&self) -> usize { COLS }
}

// Транспонування змінює розмірність!
impl<T: Copy + Default, const ROWS: usize, const COLS: usize> Matrix<T, ROWS, COLS> {
    fn transpose(&self) -> Matrix<T, COLS, ROWS> {  // Зверніть увагу: COLS, ROWS
        let mut result = Matrix::new();
        for i in 0..ROWS {
            for j in 0..COLS {
                result.data[j][i] = self.data[i][j];
            }
        }
        result
    }
}

fn main() {
    let m: Matrix<i32, 2, 3> = Matrix::from_array([
        [1, 2, 3],
        [4, 5, 6],
    ]);
    
    println!("Оригінал {}x{}:", m.rows(), m.cols());
    println!("{:?}", m.data);
    // [[1, 2, 3], [4, 5, 6]]
    
    let t = m.transpose();  // Matrix<i32, 3, 2>
    println!("\nТранспонована {}x{}:", t.rows(), t.cols());
    println!("{:?}", t.data);
    // [[1, 4], [2, 5], [3, 6]]
}
```

---

## 24.6 PhantomData — маркер невикористаного типу

### Навіщо потрібен PhantomData

Іноді параметр типу не використовується безпосередньо в полях структури, але важливий для типової системи. `PhantomData<T>` — маркер, що каже компілятору "цей тип важливий, хоча не зберігається".

**Постановка задачі:** Створити тип `Distance<Unit>`, де Unit — одиниця виміру (метри, кілометри). Це запобігає змішуванню різних одиниць.

```rust
use std::marker::PhantomData;

// Маркери одиниць виміру (порожні типи)
struct Meters;
struct Kilometers;

// Відстань з phantom type для одиниці
struct Distance<Unit> {
    value: f64,
    _unit: PhantomData<Unit>,  // Не займає пам'яті!
}

impl<Unit> Distance<Unit> {
    fn new(value: f64) -> Self {
        Self {
            value,
            _unit: PhantomData,
        }
    }
    
    fn value(&self) -> f64 {
        self.value
    }
}

// Конвертація ТІЛЬКИ для конкретних типів
impl Distance<Meters> {
    fn to_kilometers(self) -> Distance<Kilometers> {
        Distance::new(self.value / 1000.0)
    }
}

impl Distance<Kilometers> {
    fn to_meters(self) -> Distance<Meters> {
        Distance::new(self.value * 1000.0)
    }
}

fn main() {
    let d_meters: Distance<Meters> = Distance::new(5000.0);
    let d_km = d_meters.to_kilometers();
    
    println!("Відстань: {} км", d_km.value());  // 5 км
    
    // ❌ Компілятор не дозволить змішати одиниці!
    // let wrong = add_distances(d_meters, d_km);  // Різні типи!
}
```

**Як це працює:**

1. `PhantomData<Unit>` каже компілятору, що `Unit` використовується
2. `PhantomData` не займає пам'яті (zero-sized type)
3. `Distance<Meters>` та `Distance<Kilometers>` — **різні типи**
4. Компілятор не дозволить змішати їх

### Typestate pattern — стани як типи

PhantomData дозволяє реалізувати **typestate pattern** — стани об'єкта як типи. Компілятор гарантує правильний порядок операцій.

**Постановка задачі:** З'єднання, яке можна використовувати тільки після authenticate(). Компілятор має заборонити send() до автентифікації.

```rust
use std::marker::PhantomData;

// Стани як типи
struct Disconnected;
struct Connected;
struct Authenticated;

struct Connection<State> {
    address: String,
    _state: PhantomData<State>,
}

// Методи для Disconnected
impl Connection<Disconnected> {
    fn new(address: &str) -> Self {
        println!("Створюємо з'єднання до {}", address);
        Self {
            address: address.to_string(),
            _state: PhantomData,
        }
    }
    
    // Повертає Connection<Connected>
    fn connect(self) -> Connection<Connected> {
        println!("Підключаємось...");
        Connection {
            address: self.address,
            _state: PhantomData,
        }
    }
}

// Методи для Connected
impl Connection<Connected> {
    fn authenticate(self, _password: &str) -> Connection<Authenticated> {
        println!("Автентифікація...");
        Connection {
            address: self.address,
            _state: PhantomData,
        }
    }
    
    fn disconnect(self) -> Connection<Disconnected> {
        println!("Відключаємось...");
        Connection {
            address: self.address,
            _state: PhantomData,
        }
    }
}

// Методи для Authenticated
impl Connection<Authenticated> {
    fn send(&self, data: &str) {
        println!("Відправляємо на {}: {}", self.address, data);
    }
    
    fn disconnect(self) -> Connection<Disconnected> {
        println!("Завершуємо сесію...");
        Connection {
            address: self.address,
            _state: PhantomData,
        }
    }
}

fn main() {
    // Компілятор гарантує правильний порядок!
    let conn = Connection::<Disconnected>::new("192.168.1.1");
    let conn = conn.connect();
    let conn = conn.authenticate("secret");
    conn.send("Hello!");
    let _conn = conn.disconnect();
    
    // ❌ Не скомпілюється — send недоступний для Connected!
    // let conn2 = Connection::<Disconnected>::new("10.0.0.1");
    // let conn2 = conn2.connect();
    // conn2.send("data");  // Помилка: метод не знайдено
}
```

---

## 24.7 Практичний приклад: generic контейнери для агента

**Постановка задачі:** Створити `RingBuffer<T>` — кільцевий буфер фіксованої ємності для зберігання історії даних агента.

```rust
use std::collections::VecDeque;

#[derive(Debug)]
struct RingBuffer<T> {
    data: VecDeque<T>,
    capacity: usize,
}

impl<T> RingBuffer<T> {
    fn new(capacity: usize) -> Self {
        Self {
            data: VecDeque::with_capacity(capacity),
            capacity,
        }
    }
    
    fn push(&mut self, item: T) {
        // Якщо досягли ємності — видаляємо найстаріший
        if self.data.len() == self.capacity {
            self.data.pop_front();
        }
        self.data.push_back(item);
    }
    
    fn latest(&self) -> Option<&T> {
        self.data.back()
    }
    
    fn oldest(&self) -> Option<&T> {
        self.data.front()
    }
    
    fn iter(&self) -> impl Iterator<Item = &T> {
        self.data.iter()
    }
    
    fn len(&self) -> usize {
        self.data.len()
    }
    
    fn is_full(&self) -> bool {
        self.data.len() == self.capacity
    }
}

// Типи даних агента
#[derive(Debug, Clone)]
struct Position {
    x: f64,
    y: f64,
}

#[derive(Debug, Clone)]
struct SensorReading {
    value: f64,
    timestamp: u64,
}

fn main() {
    // Історія позицій (останні 5)
    let mut position_history: RingBuffer<Position> = RingBuffer::new(5);
    position_history.push(Position { x: 0.0, y: 0.0 });
    position_history.push(Position { x: 1.0, y: 1.0 });
    position_history.push(Position { x: 2.0, y: 2.0 });
    
    println!("Історія позицій ({} елементів):", position_history.len());
    for pos in position_history.iter() {
        println!("  {:?}", pos);
    }
    
    // Історія сенсора (останні 3 показання)
    let mut sensor_history: RingBuffer<SensorReading> = RingBuffer::new(3);
    sensor_history.push(SensorReading { value: 23.5, timestamp: 1000 });
    sensor_history.push(SensorReading { value: 24.0, timestamp: 1001 });
    sensor_history.push(SensorReading { value: 23.8, timestamp: 1002 });
    sensor_history.push(SensorReading { value: 24.2, timestamp: 1003 });  // Витіснить перший!
    
    println!("\nІсторія сенсора (заповнений: {}):", sensor_history.is_full());
    for reading in sensor_history.iter() {
        println!("  {:?}", reading);
    }
    // Найстаріший — 1001, не 1000!
}
```

---

## 24.8 Лабораторна робота

**Завдання:** Створити generic структури даних для агента.

### Частина 1: Generic Cache (3 бали)

```rust
struct Cache<K, V> {
    // HashMap з обмеженим розміром
}

impl<K: Hash + Eq, V> Cache<K, V> {
    fn new(capacity: usize) -> Self;
    fn get(&self, key: &K) -> Option<&V>;
    fn insert(&mut self, key: K, value: V);
    fn len(&self) -> usize;
}
```

### Частина 2: Generic PriorityQueue (4 бали)

```rust
struct PriorityQueue<T, P: Ord> {
    // Елементи з пріоритетом
}

impl<T, P: Ord> PriorityQueue<T, P> {
    fn new() -> Self;
    fn push(&mut self, item: T, priority: P);
    fn pop(&mut self) -> Option<T>;  // Повертає елемент з найвищим пріоритетом
}
```

### Частина 3: Generic StateMachine (3 бали)

```rust
struct StateMachine<S, E> {
    // S — тип стану, E — тип події
}

impl<S: Clone, E> StateMachine<S, E> {
    fn new(initial: S) -> Self;
    fn current_state(&self) -> &S;
    fn transition<F>(&mut self, event: E, f: F)
    where F: FnOnce(&S, E) -> S;
}
```

**Критерії оцінювання:**

| Критерій | Бали |
|----------|------|
| Generic Cache | 3 |
| Generic PriorityQueue | 4 |
| Generic StateMachine | 3 |
| **Максимум** | **10** |

---

## Підсумок

Узагальнення (generics) — потужний інструмент для написання повторно використовуваного коду:

| Концепція | Синтаксис | Призначення |
|-----------|-----------|-------------|
| **Generic функція** | `fn name<T>(arg: T)` | Функція для будь-якого типу |
| **Generic структура** | `struct Name<T> { field: T }` | Структура для будь-якого типу |
| **Generic enum** | `enum Name<T> { Variant(T) }` | Enum для будь-якого типу |
| **Const generics** | `<const N: usize>` | Параметр-константа |
| **PhantomData** | `PhantomData<T>` | Маркер невикористаного типу |
| **Turbofish** | `::<Type>` | Явне вказання типу |

**Ключові правила:**
- Generics — compile-time, без runtime overhead
- Мономорфізація генерує код для кожного типу
- Turbofish коли тип не виводиться
- PhantomData для phantom types та typestate

---

## Зв'язок з наступним розділом

Generics без обмежень безкорисні — ви не можете нічого зробити з невідомим `T`. Потрібно сказати "T має реалізовувати Display" або "T має бути Clone".

У **Розділі 25: Обмеження трейтів та Where-клаузи** ви дізнаєтесь:
- Як обмежувати параметри типу
- Синтаксис trait bounds
- Where-клаузи для читабельності
- Множинні bounds
