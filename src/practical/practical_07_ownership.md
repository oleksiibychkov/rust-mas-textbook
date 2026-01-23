# Практичне заняття 7: Ownership та Borrowing

## Мета заняття

Після цього заняття ви зможете:
- Розуміти концепцію володіння (ownership) у Rust
- Працювати з переміщенням (move) та копіюванням (copy) значень
- Використовувати іммутабельні посилання (&T) для читання даних
- Використовувати мутабельні посилання (&mut T) для зміни даних
- Розуміти правила borrowing та уникати помилок компілятора

---

## Теоретичний вступ

### Чому Ownership важливий?

Rust — мова без збирача сміття (garbage collector), але з гарантією безпеки пам'яті. Це можливо завдяки системі **ownership** — набору правил, які компілятор перевіряє на етапі компіляції.

**Проблеми, які вирішує ownership:**
- Витоки пам'яті (memory leaks)
- Використання звільненої пам'яті (use after free)
- Подвійне звільнення (double free)
- Гонки даних (data races)

У контексті дронів: уявіть, що кожен ресурс (батарея, сенсор, канал зв'язку) має **одного власника**. Якщо власник "помирає" — ресурс звільняється. Якщо хтось хоче використати ресурс — він має "позичити" його.

---

## Три правила Ownership

### Правило 1: Кожне значення має єдиного власника

```rust
fn main() {
    let s = String::from("Hello");  // s — власник рядка
    // Рядок "Hello" належить змінній s
}
```

### Правило 2: В один момент може бути лише один власник

```rust
fn main() {
    let s1 = String::from("Hello");
    let s2 = s1;  // Власність ПЕРЕМІЩЕНА до s2
    
    // println!("{}", s1);  // ПОМИЛКА! s1 більше не власник
    println!("{}", s2);     // OK
}
```

### Правило 3: Коли власник виходить зі scope — значення звільняється

```rust
fn main() {
    {
        let s = String::from("Hello");
        // s доступна тут
    }  // s виходить зі scope, пам'ять звільняється
    
    // println!("{}", s);  // ПОМИЛКА! s не існує
}
```

---

## Move (Переміщення)

### Що таке Move?

Коли ви присвоюєте значення іншій змінній, **власність переміщується**:

```rust
fn main() {
    let drone_name = String::from("Alpha");
    let new_name = drone_name;  // Move!
    
    // drone_name більше не валідна
    // println!("{}", drone_name);  // ПОМИЛКА!
    println!("{}", new_name);  // OK: "Alpha"
}
```

**Візуалізація:**

```
До move:
drone_name ──────► "Alpha" (на heap)

Після move:
drone_name         (невалідна)
new_name ────────► "Alpha" (на heap)
```

### Move при передачі у функцію

```rust
fn print_name(name: String) {
    println!("Ім'я: {}", name);
}  // name звільняється тут!

fn main() {
    let drone = String::from("Beta");
    print_name(drone);  // Move у функцію
    
    // println!("{}", drone);  // ПОМИЛКА! drone переміщено
}
```

### Move при поверненні з функції

```rust
fn create_drone() -> String {
    let name = String::from("Gamma");
    name  // Повертаємо (move назовні)
}

fn main() {
    let my_drone = create_drone();  // Отримуємо власність
    println!("{}", my_drone);  // OK
}
```

---

## Copy (Копіювання)

### Типи, що копіюються

Деякі типи **копіюються** замість переміщення — це прості типи фіксованого розміру:

```rust
fn main() {
    let x = 5;      // i32 — Copy
    let y = x;      // Копіювання, не move
    
    println!("x = {}", x);  // OK!
    println!("y = {}", y);  // OK!
}
```

**Copy типи:**
- Всі цілі числа: `i8`, `i32`, `u64`, тощо
- Всі дійсні числа: `f32`, `f64`
- Булеві: `bool`
- Символи: `char`
- Кортежі, якщо всі елементи Copy: `(i32, f64)`
- Масиви Copy типів: `[i32; 5]`

**Не Copy типи:**
- `String`
- `Vec<T>`
- Будь-що, що виділяє пам'ять на heap

### Чому String не Copy?

`String` зберігає дані на **heap** (купі). Копіювання означало б дублювання цих даних, що дорого. Rust змушує вас бути явними: або `clone()`, або передайте посилання.

```rust
fn main() {
    let s1 = String::from("Hello");
    let s2 = s1.clone();  // Явне глибоке копіювання
    
    println!("s1 = {}", s1);  // OK — s1 не переміщено
    println!("s2 = {}", s2);  // OK — s2 — окрема копія
}
```

---

## Borrowing (Позичання)

### Проблема: як використати значення без втрати власності?

```rust
fn calculate_length(s: String) -> usize {
    s.len()
}  // s звільняється!

fn main() {
    let name = String::from("Alpha");
    let len = calculate_length(name);  // Move
    
    // Хочемо ще використати name, але не можемо!
    // println!("{}", name);  // ПОМИЛКА!
}
```

### Рішення: посилання (&T)

Посилання дозволяє "позичити" значення без передачі власності:

```rust
fn calculate_length(s: &String) -> usize {  // Приймає посилання
    s.len()
}  // s (посилання) виходить зі scope, але дані залишаються

fn main() {
    let name = String::from("Alpha");
    let len = calculate_length(&name);  // Передаємо посилання
    
    println!("'{}' має довжину {}", name, len);  // OK!
}
```

**Візуалізація:**

```
name ──────────► "Alpha" (на heap)
      ▲
      │
      └── &name (посилання, не володіє)
```

### Іммутабельні посилання (&T)

Посилання `&T` дозволяє **читати** дані, але не змінювати:

```rust
fn main() {
    let drone = String::from("Delta");
    let r = &drone;  // Іммутабельне посилання
    
    println!("{}", r);  // OK — читання
    // r.push_str("!");  // ПОМИЛКА! Не можна змінювати через &T
}
```

### Множинні іммутабельні посилання

Можна мати **скільки завгодно** іммутабельних посилань одночасно:

```rust
fn main() {
    let data = String::from("Sensor data");
    
    let r1 = &data;
    let r2 = &data;
    let r3 = &data;
    
    println!("{}, {}, {}", r1, r2, r3);  // OK — всі читають
}
```

---

## Мутабельні посилання (&mut T)

### Зміна через посилання

Щоб змінити позичене значення, потрібно **мутабельне посилання**:

```rust
fn add_suffix(s: &mut String) {
    s.push_str("-modified");
}

fn main() {
    let mut name = String::from("Alpha");  // mut!
    add_suffix(&mut name);  // &mut
    
    println!("{}", name);  // "Alpha-modified"
}
```

**Важливо:** і змінна, і посилання мають бути `mut`.

### Обмеження мутабельних посилань

**Правило: лише ОДНЕ мутабельне посилання в один момент часу:**

```rust
fn main() {
    let mut data = String::from("Hello");
    
    let r1 = &mut data;
    // let r2 = &mut data;  // ПОМИЛКА! Друге &mut
    
    println!("{}", r1);
}
```

**Чому?** Це запобігає гонкам даних (data races). Якби два &mut існували одночасно, вони могли б конфліктувати.

### Не можна мішати &T і &mut T

```rust
fn main() {
    let mut data = String::from("Hello");
    
    let r1 = &data;      // Іммутабельне посилання
    let r2 = &data;      // Ще одне — OK
    // let r3 = &mut data;  // ПОМИЛКА! Вже є &T
    
    println!("{}, {}", r1, r2);
}
```

**Правило:** або багато `&T`, або один `&mut T` — ніколи разом.

---

## Non-Lexical Lifetimes (NLL)

Сучасний Rust (з 2018) розуміє, коли посилання **насправді** використовується:

```rust
fn main() {
    let mut data = String::from("Hello");
    
    let r1 = &data;
    println!("{}", r1);  // Останнє використання r1
    
    // r1 більше не використовується, тому можна &mut
    let r2 = &mut data;  // OK!
    r2.push_str(" World");
    
    println!("{}", r2);
}
```

Компілятор аналізує **фактичне використання**, а не лише scope.

---

## Правила Borrowing: Резюме

1. **Можна мати або:**
   - Скільки завгодно іммутабельних посилань `&T`
   - АБО рівно одне мутабельне посилання `&mut T`
   - Але не обидва одночасно

2. **Посилання завжди мають бути валідними:**
   - Не можна повернути посилання на локальну змінну
   - Посилання не може "пережити" дані

---

## Типові помилки компілятора

### Помилка 1: Use after move

```rust
fn main() {
    let s = String::from("hello");
    let s2 = s;
    println!("{}", s);  // ПОМИЛКА!
}
```

**Повідомлення:**
```
error[E0382]: borrow of moved value: `s`
  --> src/main.rs:4:20
   |
2  |     let s = String::from("hello");
   |         - move occurs because `s` has type `String`
3  |     let s2 = s;
   |              - value moved here
4  |     println!("{}", s);
   |                    ^ value borrowed here after move
```

**Виправлення:** Використайте `s2` або `s.clone()`.

### Помилка 2: Cannot borrow as mutable

```rust
fn main() {
    let data = String::from("hello");  // Не mut!
    let r = &mut data;  // ПОМИЛКА!
}
```

**Повідомлення:**
```
error[E0596]: cannot borrow `data` as mutable, as it is not declared as mutable
```

**Виправлення:** Додайте `mut`: `let mut data = ...`

### Помилка 3: Cannot borrow as mutable more than once

```rust
fn main() {
    let mut data = String::from("hello");
    let r1 = &mut data;
    let r2 = &mut data;  // ПОМИЛКА!
    println!("{} {}", r1, r2);
}
```

**Повідомлення:**
```
error[E0499]: cannot borrow `data` as mutable more than once at a time
```

**Виправлення:** Використайте r1 до створення r2, або реструктуруйте код.

### Помилка 4: Cannot borrow as mutable because also borrowed as immutable

```rust
fn main() {
    let mut data = String::from("hello");
    let r1 = &data;
    let r2 = &mut data;  // ПОМИЛКА!
    println!("{} {}", r1, r2);
}
```

**Повідомлення:**
```
error[E0502]: cannot borrow `data` as mutable because it is also borrowed as immutable
```

**Виправлення:** Завершіть використання r1 перед створенням r2.

---

## Практичні задачі

### Задача 1: Демонстрація Move

**Умова:** Створіть функцію `take_ownership`, яка приймає `String` і виводить її. Продемонструйте, що після виклику оригінальна змінна недоступна. Потім створіть функцію `give_back`, яка приймає `String` і повертає її назад.

**Розв'язання:**

```rust
fn take_ownership(s: String) {
    println!("Отримано: {}", s);
}  // s звільняється тут

fn give_back(s: String) -> String {
    println!("Обробляю: {}", s);
    s  // Повертаємо власність
}

fn main() {
    println!("=== Демонстрація Move ===\n");
    
    // Частина 1: Move без повернення
    let drone1 = String::from("Alpha");
    println!("Створено дрон: {}", drone1);
    
    take_ownership(drone1);
    // println!("{}", drone1);  // ПОМИЛКА! drone1 переміщено
    println!("drone1 більше не доступний\n");
    
    // Частина 2: Move з поверненням
    let drone2 = String::from("Beta");
    println!("Створено дрон: {}", drone2);
    
    let drone2 = give_back(drone2);  // Отримуємо назад
    println!("drone2 знову доступний: {}\n", drone2);
    
    // Частина 3: Copy типи не переміщуються
    let id: i32 = 42;
    let id_copy = id;
    println!("ID оригінал: {}, копія: {}", id, id_copy);  // Обидва доступні!
}
```

**Пояснення:**

1. `take_ownership(drone1)` — передаємо власність у функцію, drone1 "вмирає"
2. `give_back(drone2)` — функція повертає String, ми присвоюємо її тій же змінній
3. `i32` — Copy тип, тому копіюється, а не переміщується

**Результат:**
```
=== Демонстрація Move ===

Створено дрон: Alpha
Отримано: Alpha
drone1 більше не доступний

Створено дрон: Beta
Обробляю: Beta
drone2 знову доступний: Beta

ID оригінал: 42, копія: 42
```

---

### Задача 2: Іммутабельні посилання

**Умова:** Створіть структуру даних дрона (використайте String для імені). Напишіть функції:
- `print_status` — приймає `&String`, виводить статус
- `get_length` — приймає `&String`, повертає довжину

Продемонструйте, що після виклику цих функцій оригінальні дані доступні.

**Розв'язання:**

```rust
fn print_status(name: &String, battery: &i32, active: &bool) {
    let status = if *active { "АКТИВНИЙ" } else { "НЕАКТИВНИЙ" };
    println!("Дрон '{}': батарея {}%, статус: {}", name, battery, status);
}

fn get_name_length(name: &String) -> usize {
    name.len()
}

fn count_vowels(text: &String) -> usize {
    let vowels = ['a', 'e', 'i', 'o', 'u', 'A', 'E', 'I', 'O', 'U'];
    text.chars().filter(|c| vowels.contains(c)).count()
}

fn main() {
    println!("=== Іммутабельні посилання ===\n");
    
    // Дані дрона
    let name = String::from("Nighthawk");
    let battery = 85;
    let active = true;
    
    // Виклик функцій з посиланнями
    print_status(&name, &battery, &active);
    
    let len = get_name_length(&name);
    println!("Довжина імені: {} символів", len);
    
    let vowels = count_vowels(&name);
    println!("Голосних у імені: {}", vowels);
    
    // Оригінальні дані все ще доступні!
    println!("\n=== Дані після викликів ===");
    println!("name = '{}' (все ще доступна!)", name);
    println!("battery = {}", battery);
    println!("active = {}", active);
    
    // Множинні посилання одночасно
    println!("\n=== Множинні посилання ===");
    let r1 = &name;
    let r2 = &name;
    let r3 = &name;
    println!("r1={}, r2={}, r3={}", r1, r2, r3);
}
```

**Пояснення:**

1. `&String` — іммутабельне посилання, дозволяє читати
2. `*active` — розіменування (dereference) для отримання значення
3. Після викликів функцій `name`, `battery`, `active` все ще доступні
4. Можна створити багато `&T` одночасно

**Результат:**
```
=== Іммутабельні посилання ===

Дрон 'Nighthawk': батарея 85%, статус: АКТИВНИЙ
Довжина імені: 9 символів
Голосних у імені: 2

=== Дані після викликів ===
name = 'Nighthawk' (все ще доступна!)
battery = 85
active = true

=== Множинні посилання ===
r1=Nighthawk, r2=Nighthawk, r3=Nighthawk
```

---

### Задача 3: Мутабельні посилання

**Умова:** Напишіть функції для модифікації даних дрона:
- `rename_drone` — змінює ім'я дрона
- `charge_battery` — збільшує заряд батареї
- `toggle_status` — перемикає статус активності

Всі функції приймають `&mut` посилання.

**Розв'язання:**

```rust
fn rename_drone(name: &mut String, new_name: &str) {
    println!("Перейменування: '{}' → '{}'", name, new_name);
    name.clear();
    name.push_str(new_name);
}

fn charge_battery(battery: &mut i32, amount: i32) {
    let old = *battery;
    *battery += amount;
    if *battery > 100 {
        *battery = 100;
    }
    println!("Зарядка: {}% → {}%", old, battery);
}

fn toggle_status(active: &mut bool) {
    *active = !*active;
    let status = if *active { "УВІМКНЕНО" } else { "ВИМКНЕНО" };
    println!("Статус змінено: {}", status);
}

fn apply_damage(battery: &mut i32, damage: i32) {
    *battery -= damage;
    if *battery < 0 {
        *battery = 0;
    }
    println!("Пошкодження! Батарея: {}%", battery);
}

fn main() {
    println!("=== Мутабельні посилання ===\n");
    
    // Початкові дані (mut!)
    let mut name = String::from("Drone-001");
    let mut battery = 50;
    let mut active = false;
    
    println!("Початковий стан:");
    println!("  Ім'я: {}, Батарея: {}%, Активний: {}\n", name, battery, active);
    
    // Модифікації
    rename_drone(&mut name, "Phoenix");
    charge_battery(&mut battery, 30);
    toggle_status(&mut active);
    
    println!("\nПісля модифікацій:");
    println!("  Ім'я: {}, Батарея: {}%, Активний: {}\n", name, battery, active);
    
    // Ще модифікації
    charge_battery(&mut battery, 50);  // Перевищить 100
    apply_damage(&mut battery, 25);
    toggle_status(&mut active);
    
    println!("\nФінальний стан:");
    println!("  Ім'я: {}, Батарея: {}%, Активний: {}", name, battery, active);
}
```

**Пояснення:**

1. `&mut String` — мутабельне посилання на String
2. `*battery += amount` — розіменування для зміни значення
3. `name.clear(); name.push_str(...)` — методи String працюють через &mut self
4. Кожна функція приймає `&mut`, змінює дані, і вони залишаються зміненими

**Результат:**
```
=== Мутабельні посилання ===

Початковий стан:
  Ім'я: Drone-001, Батарея: 50%, Активний: false

Перейменування: 'Drone-001' → 'Phoenix'
Зарядка: 50% → 80%
Статус змінено: УВІМКНЕНО

Після модифікацій:
  Ім'я: Phoenix, Батарея: 80%, Активний: true

Зарядка: 80% → 100%
Пошкодження! Батарея: 75%
Статус змінено: ВИМКНЕНО

Фінальний стан:
  Ім'я: Phoenix, Батарея: 75%, Активний: false
```

---

### Задача 4: Комбінування правил Borrowing

**Умова:** Створіть симуляцію, де:
1. Функція `analyze` читає дані через `&T`
2. Функція `update` змінює дані через `&mut T`
3. Продемонструйте правильний порядок викликів, щоб уникнути конфліктів

**Розв'язання:**

```rust
fn analyze_drone(name: &String, battery: &i32) -> String {
    let status = if *battery > 50 {
        "нормальний"
    } else if *battery > 20 {
        "потребує зарядки"
    } else {
        "критичний"
    };
    
    format!("Аналіз '{}': батарея {}% ({})", name, battery, status)
}

fn update_battery(battery: &mut i32, change: i32) {
    *battery = (*battery + change).clamp(0, 100);
}

fn process_drone(name: &mut String, battery: &mut i32) {
    // Спочатку аналіз (потрібні & посилання)
    // Але ми вже маємо &mut! Тому робимо reborrow
    {
        let analysis = analyze_drone(name, battery);
        println!("{}", analysis);
    }
    
    // Тепер можемо модифікувати
    if *battery < 30 {
        println!("Автоматична зарядка...");
        update_battery(battery, 50);
    }
    
    name.push_str(" [processed]");
}

fn main() {
    println!("=== Комбінування Borrowing ===\n");
    
    let mut name = String::from("Scout-7");
    let mut battery = 25;
    
    // Демонстрація 1: послідовні виклики
    println!("--- Послідовні виклики ---");
    
    // Спочатку читаємо
    let report = analyze_drone(&name, &battery);
    println!("{}", report);
    
    // Потім модифікуємо (& вже не використовується)
    update_battery(&mut battery, 30);
    println!("Після зарядки: {}%\n", battery);
    
    // Демонстрація 2: комплексна обробка
    println!("--- Комплексна обробка ---");
    let mut drone2_name = String::from("Recon-3");
    let mut drone2_battery = 15;
    
    println!("До: '{}', {}%", drone2_name, drone2_battery);
    process_drone(&mut drone2_name, &mut drone2_battery);
    println!("Після: '{}', {}%\n", drone2_name, drone2_battery);
    
    // Демонстрація 3: NLL у дії
    println!("--- NLL у дії ---");
    let mut data = String::from("test");
    
    let r1 = &data;
    println!("Читання: {}", r1);
    // r1 більше не використовується
    
    let r2 = &mut data;  // OK завдяки NLL!
    r2.push_str("_modified");
    println!("Після модифікації: {}", r2);
}
```

**Пояснення:**

1. `analyze_drone` приймає `&` — тільки читає
2. `update_battery` приймає `&mut` — змінює
3. У `process_drone` ми маємо `&mut`, але можемо "reborrow" як `&` для читання
4. `clamp(0, 100)` — обмежує значення в діапазоні
5. NLL дозволяє створити `&mut` після того, як `&` перестало використовуватись

**Результат:**
```
=== Комбінування Borrowing ===

--- Послідовні виклики ---
Аналіз 'Scout-7': батарея 25% (потребує зарядки)
Після зарядки: 55%

--- Комплексна обробка ---
До: 'Recon-3', 15%
Аналіз 'Recon-3': батарея 15% (критичний)
Автоматична зарядка...
Після: 'Recon-3 [processed]', 65%

--- NLL у дії ---
Читання: test
Після модифікації: test_modified
```

---

## Домашнє завдання

### Завдання 1: Виправлення помилок Ownership

**Умова:** Наступний код має помилки ownership. Виправте їх, не змінюючи логіку програми.

```rust
// BROKEN CODE - виправте помилки!
fn main() {
    let message = String::from("Hello, Drone!");
    
    print_message(message);
    print_message(message);  // Помилка: message переміщено
    
    let mut data = String::from("Data");
    let r1 = &data;
    let r2 = &mut data;  // Помилка: конфлікт посилань
    println!("{} {}", r1, r2);
}

fn print_message(msg: String) {
    println!("{}", msg);
}
```

**Розв'язання:**

```rust
fn main() {
    println!("=== Виправлені помилки Ownership ===\n");
    
    // Виправлення 1: використовуємо посилання замість передачі власності
    let message = String::from("Hello, Drone!");
    
    print_message(&message);  // Передаємо посилання
    print_message(&message);  // Можемо викликати знову!
    
    println!("message все ще доступна: {}\n", message);
    
    // Виправлення 2: розділяємо використання посилань у часі
    let mut data = String::from("Data");
    
    // Спочатку використовуємо іммутабельне посилання
    let r1 = &data;
    println!("r1 = {}", r1);
    // r1 більше не використовується (NLL)
    
    // Тепер можемо створити мутабельне
    let r2 = &mut data;
    r2.push_str("-modified");
    println!("r2 = {}", r2);
    
    println!("\nФінальне значення data: {}", data);
}

fn print_message(msg: &String) {  // Приймаємо посилання!
    println!("Повідомлення: {}", msg);
}
```

**Ключові виправлення:**
1. `print_message(msg: String)` → `print_message(msg: &String)`
2. Виклики `print_message(message)` → `print_message(&message)`
3. Розділили використання `r1` і `r2` у часі

---

### Завдання 2: Підрахунок через посилання

**Умова:** Напишіть функцію `count_occurrences`, яка приймає посилання на рядок та символ, і повертає кількість входжень цього символу. Не переміщуйте дані!

**Розв'язання:**

```rust
fn count_occurrences(text: &String, target: char) -> usize {
    text.chars().filter(|&c| c == target).count()
}

fn count_words(text: &String) -> usize {
    text.split_whitespace().count()
}

fn count_digits(text: &String) -> usize {
    text.chars().filter(|c| c.is_ascii_digit()).count()
}

fn analyze_text(text: &String) {
    println!("Текст: '{}'", text);
    println!("  Довжина: {} символів", text.len());
    println!("  Слів: {}", count_words(text));
    println!("  Цифр: {}", count_digits(text));
    println!("  Літер 'a': {}", count_occurrences(text, 'a'));
    println!("  Літер 'e': {}", count_occurrences(text, 'e'));
    println!("  Пробілів: {}", count_occurrences(text, ' '));
}

fn main() {
    println!("=== Підрахунок через посилання ===\n");
    
    let text1 = String::from("Drone Alpha ready for takeoff at 0800");
    let text2 = String::from("Battery level: 85 percent");
    let text3 = String::from("a]e]i]o]u - vowels");
    
    analyze_text(&text1);
    println!();
    analyze_text(&text2);
    println!();
    analyze_text(&text3);
    
    // Всі рядки все ще доступні!
    println!("\n=== Рядки після аналізу ===");
    println!("text1: {}", text1);
    println!("text2: {}", text2);
    println!("text3: {}", text3);
}
```

**Результат:**
```
=== Підрахунок через посилання ===

Текст: 'Drone Alpha ready for takeoff at 0800'
  Довжина: 38 символів
  Слів: 7
  Цифр: 4
  Літер 'a': 4
  Літер 'e': 3
  Пробілів: 6

Текст: 'Battery level: 85 percent'
  Довжина: 25 символів
  Слів: 4
  Цифр: 2
  Літер 'a': 1
  Літер 'e': 5
  Пробілів: 3
...
```

---

### Завдання 3: Ланцюжок модифікацій

**Умова:** Створіть набір функцій, кожна з яких модифікує рядок через `&mut String`:
- `to_uppercase_first` — робить першу літеру великою
- `add_prefix` — додає префікс
- `add_suffix` — додає суфікс

Застосуйте їх послідовно до одного рядка.

**Розв'язання:**

```rust
fn to_uppercase_first(s: &mut String) {
    if let Some(first) = s.chars().next() {
        let rest: String = s.chars().skip(1).collect();
        s.clear();
        s.push(first.to_ascii_uppercase());
        s.push_str(&rest);
    }
}

fn add_prefix(s: &mut String, prefix: &str) {
    let original = s.clone();
    s.clear();
    s.push_str(prefix);
    s.push_str(&original);
}

fn add_suffix(s: &mut String, suffix: &str) {
    s.push_str(suffix);
}

fn trim_spaces(s: &mut String) {
    let trimmed = s.trim().to_string();
    s.clear();
    s.push_str(&trimmed);
}

fn main() {
    println!("=== Ланцюжок модифікацій ===\n");
    
    let mut drone_name = String::from("  alpha  ");
    
    println!("Оригінал: '{}'", drone_name);
    
    // Ланцюжок модифікацій
    trim_spaces(&mut drone_name);
    println!("Після trim: '{}'", drone_name);
    
    to_uppercase_first(&mut drone_name);
    println!("Після uppercase: '{}'", drone_name);
    
    add_prefix(&mut drone_name, "Drone-");
    println!("Після prefix: '{}'", drone_name);
    
    add_suffix(&mut drone_name, "-001");
    println!("Після suffix: '{}'", drone_name);
    
    // Ще один приклад
    println!("\n--- Інший приклад ---");
    let mut status = String::from("ready");
    
    to_uppercase_first(&mut status);
    add_prefix(&mut status, "[");
    add_suffix(&mut status, "]");
    
    println!("Статус: {}", status);
}
```

**Результат:**
```
=== Ланцюжок модифікацій ===

Оригінал: '  alpha  '
Після trim: 'alpha'
Після uppercase: 'Alpha'
Після prefix: 'Drone-Alpha'
Після suffix: 'Drone-Alpha-001'

--- Інший приклад ---
Статус: [Ready]
```

---

### Завдання 4: Симуляція команд дрона

**Умова:** Створіть структуру стану дрона (використовуючи окремі змінні) та функції для обробки команд. Кожна команда модифікує стан через `&mut`. Команди: MOVE (змінює позицію), CHARGE (збільшує батарею), RENAME (змінює ім'я).

**Розв'язання:**

```rust
fn execute_move(x: &mut i32, y: &mut i32, dx: i32, dy: i32, battery: &mut i32) -> bool {
    let cost = ((dx.abs() + dy.abs()) as f64 * 0.5) as i32;
    
    if *battery < cost {
        println!("MOVE: недостатньо батареї (потрібно {}%, є {}%)", cost, battery);
        return false;
    }
    
    *x += dx;
    *y += dy;
    *battery -= cost;
    println!("MOVE: ({}, {}) → ({}, {}), батарея: {}%", *x - dx, *y - dy, x, y, battery);
    true
}

fn execute_charge(battery: &mut i32, amount: i32) {
    let old = *battery;
    *battery = (*battery + amount).min(100);
    println!("CHARGE: {}% → {}%", old, battery);
}

fn execute_rename(name: &mut String, new_name: &str) {
    println!("RENAME: '{}' → '{}'", name, new_name);
    name.clear();
    name.push_str(new_name);
}

fn print_status(name: &String, x: &i32, y: &i32, battery: &i32) {
    println!("┌─────────────────────────────┐");
    println!("│ Дрон: {:^20} │", name);
    println!("│ Позиція: ({:>4}, {:>4})       │", x, y);
    println!("│ Батарея: {:>3}%               │", battery);
    println!("└─────────────────────────────┘");
}

fn main() {
    println!("=== Симуляція команд дрона ===\n");
    
    // Стан дрона
    let mut name = String::from("Unnamed");
    let mut x = 0;
    let mut y = 0;
    let mut battery = 50;
    
    println!("Початковий стан:");
    print_status(&name, &x, &y, &battery);
    
    println!("\n--- Виконання команд ---\n");
    
    // Команди
    execute_rename(&mut name, "Explorer");
    execute_move(&mut x, &mut y, 10, 20, &mut battery);
    execute_move(&mut x, &mut y, 5, -10, &mut battery);
    execute_charge(&mut battery, 30);
    execute_move(&mut x, &mut y, 100, 100, &mut battery);  // Може не вистачити
    execute_charge(&mut battery, 100);
    execute_move(&mut x, &mut y, 100, 100, &mut battery);
    
    println!("\n--- Фінальний стан ---\n");
    print_status(&name, &x, &y, &battery);
}
```

**Результат:**
```
=== Симуляція команд дрона ===

Початковий стан:
┌─────────────────────────────┐
│ Дрон:       Unnamed        │
│ Позиція: (   0,    0)       │
│ Батарея:  50%               │
└─────────────────────────────┘

--- Виконання команд ---

RENAME: 'Unnamed' → 'Explorer'
MOVE: (0, 0) → (10, 20), батарея: 35%
MOVE: (10, 20) → (15, 10), батарея: 28%
CHARGE: 28% → 58%
MOVE: недостатньо батареї (потрібно 100%, є 58%)
CHARGE: 58% → 100%
MOVE: (15, 10) → (115, 110), батарея: 0%

--- Фінальний стан ---

┌─────────────────────────────┐
│ Дрон:       Explorer       │
│ Позиція: ( 115,  110)       │
│ Батарея:   0%               │
└─────────────────────────────┘
```

---

## Підсумок заняття

На цьому занятті ви навчились:

1. **Розуміти три правила Ownership:**
   - Кожне значення має єдиного власника
   - Лише один власник в один момент
   - Значення звільняється, коли власник виходить зі scope

2. **Працювати з Move:**
   - String та інші heap-типи переміщуються
   - Copy типи (числа, bool, char) копіюються
   - `clone()` для явного копіювання

3. **Використовувати Borrowing:**
   - `&T` — іммутабельне посилання (читання)
   - `&mut T` — мутабельне посилання (зміна)

4. **Дотримуватись правил Borrowing:**
   - Багато `&T` АБО один `&mut T`
   - Ніколи обидва одночасно

---

## Перевірте себе

1. Що станеться з `String` після передачі у функцію, що приймає `String`?
2. Чому `i32` можна копіювати, а `String` — ні?
3. Скільки `&T` посилань можна мати одночасно?
4. Скільки `&mut T` посилань можна мати одночасно?
5. Що потрібно додати до змінної, щоб створити `&mut` посилання?
6. Що таке NLL і як воно допомагає?

**Відповіді:**
1. Власність переміщується у функцію, оригінальна змінна недоступна
2. `i32` живе на стеку, копіювання дешеве; `String` має дані на heap
3. Скільки завгодно
4. Рівно одне
5. `mut` — змінна має бути `let mut x = ...`
6. Non-Lexical Lifetimes — компілятор аналізує фактичне використання посилань

---

## Наступне заняття

На наступному занятті ми вивчимо **Структури та Enum** — як створювати власні типи даних для моделювання складних об'єктів, таких як повний стан дрона з усіма його характеристиками.
