# ЧАСТИНА II: РОБАСТНІСТЬ, ДАНІ ТА ІНСТРУМЕНТАРІЙ

## Вступ до Частини II

У Частині I ви створили автономного агента, який демонструє всі базові концепції Rust: ownership, borrowing, структури, enum, модулі, тести. Агент працює — він злітає, рухається, сканує, повертається на базу.

Але він поки що живе в ідеальному світі. Фіксована кількість даних. Немає обробки помилок. Примітивне логування. Немає збереження стану між запусками.

**Частина II готує агента до реального світу:**

- **Колекції** (розділи 16-17) — динамічні дані, історія переміщень, карта світу
- **Обробка помилок** (розділи 18-20) — graceful degradation, діагностика
- **Логування** (розділ 21) — моніторинг та аудит
- **Traits і Generics** (розділи 22-25) — абстракції, розширюваність
- **Серіалізація** (розділ 26) — обмін даними, збереження стану
- **Практикум v2.0** (розділ 27) — інтеграція всього

Після Частини II ваш агент зможе зберігати історію польоту, обробляти помилки без паніки, логувати свої дії, і зберігати стан у файл.

---

# Розділ 16: Колекції — Vec та динамічні дані

## Вступ

До цього моменту наш агент працював з фіксованою кількістю даних: одна позиція, один стан, один рівень батареї. Але уявіть реальний політ БПЛА. Скільки точок він пролетить? Невідомо заздалегідь — може десять, може тисячу. Скільки команд отримає? Залежить від місії. Скільки об'єктів виявить? Ніхто не знає.

Масив фіксованого розміру `[Position; 1000]` — погане рішення. Якщо точок більше тисячі — програма зламається. Якщо менше — марнуємо пам'ять. І як взагалі знати наперед?

`Vec<T>` — це динамічний масив, основна колекція Rust для послідовностей елементів. На відміну від масивів фіксованого розміру, вектор росте та скорочується за потребою. Він автоматично виділяє пам'ять на heap, розширюється при додаванні елементів, і звільняє пам'ять при видаленні.

У цьому розділі ви навчитесь створювати вектори, маніпулювати елементами, ітерувати різними способами, і застосуєте це для збереження історії руху агента та черги команд.

---

## 16.1 Що таке Vec і як він працює

### Анатомія вектора

Вектор `Vec<T>` складається з трьох частин:
- **Вказівник** на дані в heap
- **Довжина (length)** — скільки елементів фактично є
- **Ємність (capacity)** — скільки елементів може вміститись без реалокації

```
Stack:                 Heap:
┌─────────────┐       ┌───┬───┬───┬───┬───┬───┐
│ ptr ────────┼──────►│ 1 │ 2 │ 3 │   │   │   │
│ length: 3   │       └───┴───┴───┴───┴───┴───┘
│ capacity: 6 │        ▲           ▲
└─────────────┘        │           │
                       └─ 3 елементи (length)
                                   └─ 6 слотів (capacity)
```

Коли ви додаєте елемент і length == capacity, вектор виділяє новий, більший блок пам'яті (зазвичай подвоює ємність), копіює старі дані, і звільняє старий блок.

### Створення порожнього вектора

Найпростіший спосіб — `Vec::new()`. Але є нюанс: компілятор має знати тип елементів.

```rust
fn main() {
    // Спосіб 1: явна анотація типу
    let numbers: Vec<i32> = Vec::new();
    
    // Спосіб 2: компілятор виводить тип з контексту
    let mut values = Vec::new();
    values.push(42);  // Тепер компілятор знає: Vec<i32>
    
    println!("numbers: {:?}", numbers);  // []
    println!("values: {:?}", values);    // [42]
}
```

У першому випадку ми явно вказали `Vec<i32>`. У другому — компілятор побачив `push(42)` і зрозумів, що це вектор цілих чисел.

### Макрос vec! — швидке створення

Коли потрібен вектор з початковими значеннями, макрос `vec!` зручніший за послідовність `push`:

```rust
fn main() {
    // Список значень
    let numbers = vec![1, 2, 3, 4, 5];
    println!("numbers: {:?}", numbers);  // [1, 2, 3, 4, 5]
    
    // Повторення значення: vec![значення; кількість]
    let zeros = vec![0; 10];
    println!("zeros: {:?}", zeros);  // [0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
    
    // Для структур теж працює
    let points = vec![
        Point { x: 0.0, y: 0.0 },
        Point { x: 1.0, y: 2.0 },
        Point { x: 3.0, y: 4.0 },
    ];
    println!("points count: {}", points.len());  // 3
}

#[derive(Debug)]
struct Point { x: f64, y: f64 }
```

Синтаксис `vec![0; 10]` вимагає, щоб тип реалізовував trait `Clone` — адже значення потрібно скопіювати 10 разів.

### Vec::with_capacity — оптимізація

Якщо ви знаєте приблизну кількість елементів наперед, `with_capacity` уникає багаторазових реалокацій:

```rust
fn main() {
    // Виділяємо місце для ~100 елементів заздалегідь
    let mut readings = Vec::with_capacity(100);
    
    println!("Початок:");
    println!("  Length: {}", readings.len());        // 0
    println!("  Capacity: {}", readings.capacity()); // 100
    
    // Додаємо 100 елементів — жодної реалокації
    for i in 0..100 {
        readings.push(i);
    }
    
    println!("\nПісля 100 push:");
    println!("  Length: {}", readings.len());        // 100
    println!("  Capacity: {}", readings.capacity()); // 100 (без змін)
    
    // 101-й елемент — тепер реалокація
    readings.push(100);
    println!("\nПісля 101-го push:");
    println!("  Capacity: {}", readings.capacity()); // ~200 (подвоєння)
}
```

Коли length досягає capacity, вектор виділяє новий блок (зазвичай вдвічі більший), копіює дані, і звільняє старий. Це O(n) операція. Якщо заздалегідь виділити достатньо місця — уникаємо цих витрат.

### Конвертація з інших типів

Вектори можна створювати з масивів, діапазонів, рядків:

```rust
fn main() {
    // З масиву
    let array = [1, 2, 3];
    let vec_from_array: Vec<i32> = array.into();
    println!("from array: {:?}", vec_from_array);  // [1, 2, 3]
    
    // З діапазону через collect()
    let vec_from_range: Vec<i32> = (1..=5).collect();
    println!("from range: {:?}", vec_from_range);  // [1, 2, 3, 4, 5]
    
    // З рядка — символи
    let chars: Vec<char> = "hello".chars().collect();
    println!("chars: {:?}", chars);  // ['h', 'e', 'l', 'l', 'o']
    
    // З рядка — байти
    let bytes: Vec<u8> = "hello".bytes().collect();
    println!("bytes: {:?}", bytes);  // [104, 101, 108, 108, 111]
}
```

Метод `collect()` — це "магічний" метод ітераторів, що збирає елементи в колекцію. Тип колекції визначається з контексту (анотація або використання).

---

## 16.2 Додавання та видалення елементів

### push і pop — робота з кінцем

Найчастіші операції — додавання в кінець і видалення з кінця. Вони O(1) — найшвидші:

```rust
fn main() {
    let mut stack = vec![1, 2, 3];
    
    // push — додати в кінець
    stack.push(4);
    stack.push(5);
    println!("Після push: {:?}", stack);  // [1, 2, 3, 4, 5]
    
    // pop — видалити з кінця, повертає Option<T>
    let last = stack.pop();  // Some(5)
    let also_last = stack.pop();  // Some(4)
    println!("Popped: {:?}, {:?}", last, also_last);
    println!("Після pop: {:?}", stack);  // [1, 2, 3]
    
    // pop з порожнього вектора — None
    let mut empty: Vec<i32> = vec![];
    let nothing = empty.pop();
    println!("Pop empty: {:?}", nothing);  // None
}
```

`pop()` повертає `Option<T>` — `Some(value)` якщо є що видалити, `None` якщо вектор порожній. Це безпечніше за паніку.

### insert і remove — робота з серединою

Вставка та видалення за індексом потребують зсуву елементів, тому це O(n):

```rust
fn main() {
    let mut vec = vec![1, 2, 3, 4, 5];
    
    // insert(index, value) — вставити за індексом
    vec.insert(0, 0);     // На початок
    println!("Після insert(0, 0): {:?}", vec);  // [0, 1, 2, 3, 4, 5]
    
    vec.insert(3, 100);   // В середину
    println!("Після insert(3, 100): {:?}", vec);  // [0, 1, 2, 100, 3, 4, 5]
    
    // remove(index) — видалити за індексом, повертає елемент
    let removed = vec.remove(3);  // Видаляємо 100
    println!("Removed: {}", removed);  // 100
    println!("Після remove: {:?}", vec);  // [0, 1, 2, 3, 4, 5]
    
    // remove з невалідним індексом — ПАНІКА!
    // vec.remove(100);  // panic: index out of bounds
}
```

Важливо: `insert` і `remove` панікують, якщо індекс виходить за межі. Завжди перевіряйте!

### swap_remove — швидке видалення без збереження порядку

Якщо порядок елементів не важливий, `swap_remove` видаляє за O(1) — він замінює елемент останнім і робить pop:

```rust
fn main() {
    let mut vec = vec!["a", "b", "c", "d", "e"];
    
    // Звичайний remove(1) зсуває всі елементи після "b"
    // swap_remove(1) — заміняє "b" на "e" і видаляє "e"
    let removed = vec.swap_remove(1);
    
    println!("Removed: {}", removed);  // b
    println!("Vec: {:?}", vec);        // ["a", "e", "c", "d"]
    //                                       ↑
    //                              "e" тепер на місці "b"
}
```

Це корисно для списків, де порядок не має значення (наприклад, список виявлених об'єктів).

### extend і append — масове додавання

```rust
fn main() {
    let mut vec = vec![1, 2, 3];
    
    // extend — додає елементи з ітератора
    vec.extend([4, 5, 6]);  // З масиву
    vec.extend(7..=9);      // З діапазону
    println!("Після extend: {:?}", vec);  // [1, 2, 3, 4, 5, 6, 7, 8, 9]
    
    // append — переміщує всі елементи з іншого вектора
    let mut other = vec![10, 11, 12];
    vec.append(&mut other);
    
    println!("Після append: {:?}", vec);    // [..., 10, 11, 12]
    println!("other тепер: {:?}", other);   // [] — порожній!
}
```

Різниця: `extend` копіює або переміщує елементи з ітератора, `append` переміщує всі елементи з іншого вектора (який стає порожнім).

### retain — фільтрація на місці

```rust
fn main() {
    let mut numbers = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
    
    // Залишити тільки парні
    numbers.retain(|&x| x % 2 == 0);
    println!("Тільки парні: {:?}", numbers);  // [2, 4, 6, 8, 10]
    
    // Залишити тільки більші за 5
    numbers.retain(|&x| x > 5);
    println!("Більші за 5: {:?}", numbers);  // [6, 8, 10]
}
```

`retain` модифікує вектор на місці, видаляючи елементи, для яких предикат повертає `false`.

### clear — видалити все

```rust
fn main() {
    let mut vec = vec![1, 2, 3, 4, 5];
    
    println!("До clear: len={}, capacity={}", vec.len(), vec.capacity());
    
    vec.clear();
    
    println!("Після clear: len={}, capacity={}", vec.len(), vec.capacity());
    // len=0, capacity залишається!
}
```

`clear()` видаляє елементи, але не звільняє виділену пам'ять. Capacity залишається для можливого повторного використання.

---

## 16.3 Доступ до елементів

### Індексація — швидко, але небезпечно

Найпростіший спосіб — квадратні дужки:

```rust
fn main() {
    let vec = vec![10, 20, 30, 40, 50];
    
    let first = vec[0];   // 10
    let third = vec[2];   // 30
    
    println!("first={}, third={}", first, third);
    
    // Але невалідний індекс — ПАНІКА!
    // let invalid = vec[100];  // panic: index out of bounds
}
```

Індексація паралізує програму, якщо індекс виходить за межі. Це прийнятно, якщо ви на 100% впевнені в індексі.

### get — безпечний доступ через Option

Якщо індекс може бути невалідним, використовуйте `get`:

```rust
fn main() {
    let vec = vec![10, 20, 30, 40, 50];
    
    // get повертає Option<&T>
    let third = vec.get(2);
    println!("vec.get(2) = {:?}", third);  // Some(&30)
    
    let invalid = vec.get(100);
    println!("vec.get(100) = {:?}", invalid);  // None
    
    // Зручно з if let або match
    if let Some(value) = vec.get(2) {
        println!("Елемент існує: {}", value);
    }
    
    // Або з unwrap_or для значення за замовчуванням
    let value = vec.get(100).unwrap_or(&0);
    println!("Значення або 0: {}", value);  // 0
}
```

### first і last — спеціальні випадки

```rust
fn main() {
    let vec = vec![10, 20, 30];
    
    println!("first: {:?}", vec.first());  // Some(&10)
    println!("last: {:?}", vec.last());    // Some(&30)
    
    let empty: Vec<i32> = vec![];
    println!("empty.first: {:?}", empty.first());  // None
    println!("empty.last: {:?}", empty.last());    // None
}
```

Повертають `Option<&T>` — безпечно працюють з порожнім вектором.

### Слайси — вікна в дані

Слайс `&[T]` — це посилання на частину вектора:

```rust
fn main() {
    let vec = vec![0, 1, 2, 3, 4, 5, 6, 7, 8, 9];
    
    // Повний діапазон
    let slice = &vec[2..6];  // [2, 3, 4, 5]
    println!("vec[2..6]: {:?}", slice);
    
    // Від початку
    let from_start = &vec[..4];  // [0, 1, 2, 3]
    println!("vec[..4]: {:?}", from_start);
    
    // До кінця
    let to_end = &vec[7..];  // [7, 8, 9]
    println!("vec[7..]: {:?}", to_end);
    
    // Весь вектор як слайс
    let all = &vec[..];
    println!("vec[..]: {:?}", all);
}
```

Слайси не володіють даними — це лише "вікно" в існуючий вектор.

---

## 16.4 Ітерація — серце роботи з колекціями

Rust надає три види ітераторів для векторів, кожен з різним ownership.

### iter() — читання без зміни

`iter()` повертає ітератор іммутабельних посилань `&T`:

```rust
fn main() {
    let names = vec!["Аліса", "Боб", "Чарлі"];
    
    // iter() — позичаємо елементи для читання
    for name in names.iter() {
        println!("Привіт, {}!", name);  // name: &str
    }
    
    // Вектор залишається доступним
    println!("Всього імен: {}", names.len());
}
```

Після ітерації вектор все ще ваш — ви тільки "подивились" на елементи.

### iter_mut() — модифікація елементів

`iter_mut()` повертає ітератор мутабельних посилань `&mut T`:

```rust
fn main() {
    let mut numbers = vec![1, 2, 3, 4, 5];
    
    // iter_mut() — позичаємо елементи для зміни
    for num in numbers.iter_mut() {
        *num *= 2;  // Подвоюємо кожен елемент
    }
    
    println!("Подвоєні: {:?}", numbers);  // [2, 4, 6, 8, 10]
}
```

Зверніть увагу на `*num *= 2` — ми розіменовуємо посилання, щоб змінити значення.

### into_iter() — споживання вектора

`into_iter()` переміщує елементи з вектора, споживаючи його:

```rust
fn main() {
    let names = vec![
        String::from("Аліса"),
        String::from("Боб"),
        String::from("Чарлі"),
    ];
    
    // into_iter() — забираємо ownership елементів
    for name in names.into_iter() {
        println!("Оброблено: {}", name);  // name: String (owned)
        // name звільняється в кінці ітерації
    }
    
    // names більше не існує!
    // println!("{:?}", names);  // Помилка компіляції
}
```

Це корисно, коли потрібно передати елементи кудись або трансформувати їх.

### Скорочений синтаксис у for

Rust автоматично викликає потрібний ітератор залежно від синтаксису:

```rust
fn main() {
    let vec = vec![1, 2, 3];
    
    // &vec — викликає iter()
    for x in &vec {
        println!("{}", x);  // x: &i32
    }
    
    // &mut vec — викликає iter_mut()
    let mut vec2 = vec![1, 2, 3];
    for x in &mut vec2 {
        *x += 10;  // x: &mut i32
    }
    
    // vec — викликає into_iter()
    let vec3 = vec![1, 2, 3];
    for x in vec3 {
        println!("{}", x);  // x: i32 (owned)
    }
    // vec3 більше не доступний
}
```

### Методи ітераторів — функціональний стиль

Ітератори мають багато корисних методів для трансформації та агрегації:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
    
    // map — трансформувати кожен елемент
    let squared: Vec<i32> = numbers.iter()
        .map(|x| x * x)
        .collect();
    println!("Квадрати: {:?}", squared);  // [1, 4, 9, 16, ...]
    
    // filter — відфільтрувати елементи
    let evens: Vec<i32> = numbers.iter()
        .filter(|x| *x % 2 == 0)
        .copied()  // &i32 -> i32
        .collect();
    println!("Парні: {:?}", evens);  // [2, 4, 6, 8, 10]
    
    // fold — агрегувати в одне значення
    let sum: i32 = numbers.iter().fold(0, |acc, x| acc + x);
    println!("Сума: {}", sum);  // 55
    
    // Або простіше — sum()
    let sum2: i32 = numbers.iter().sum();
    println!("Сума (sum): {}", sum2);  // 55
    
    // find — знайти перший елемент за умовою
    let first_even = numbers.iter().find(|x| *x % 2 == 0);
    println!("Перше парне: {:?}", first_even);  // Some(&2)
    
    // any — чи є хоча б один
    let has_negative = numbers.iter().any(|x| *x < 0);
    println!("Є від'ємні: {}", has_negative);  // false
    
    // all — чи всі задовольняють
    let all_positive = numbers.iter().all(|x| *x > 0);
    println!("Всі додатні: {}", all_positive);  // true
    
    // enumerate — додати індекси
    for (i, x) in numbers.iter().enumerate().take(3) {
        println!("numbers[{}] = {}", i, x);
    }
    
    // take / skip — взяти перші / пропустити перші
    let first_three: Vec<i32> = numbers.iter().take(3).copied().collect();
    let after_five: Vec<i32> = numbers.iter().skip(5).copied().collect();
    println!("Перші 3: {:?}", first_three);  // [1, 2, 3]
    println!("Після 5: {:?}", after_five);   // [6, 7, 8, 9, 10]
}
```

### Ланцюжки ітераторів

Методи можна комбінувати в ланцюжки:

```rust
fn main() {
    let data = vec![
        ("Аліса", 85),
        ("Боб", 92),
        ("Чарлі", 78),
        ("Діана", 95),
        ("Єва", 88),
    ];
    
    // Знайти імена студентів з оцінкою > 85
    let top_students: Vec<&str> = data.iter()
        .filter(|(_, score)| *score > 85)  // Тільки > 85
        .map(|(name, _)| *name)            // Взяти тільки ім'я
        .collect();
    
    println!("Топ студенти: {:?}", top_students);  // ["Боб", "Діана", "Єва"]
    
    // Середня оцінка
    let total: i32 = data.iter().map(|(_, score)| *score).sum();
    let average = total as f64 / data.len() as f64;
    println!("Середня оцінка: {:.1}", average);  // 87.6
}
```

---

## 16.5 Ownership та Vec

Вектор володіє своїми елементами. Це має важливі наслідки.

### Не можна "вийняти" елемент за індексом

```rust
fn main() {
    let names = vec![
        String::from("Аліса"),
        String::from("Боб"),
    ];
    
    // Це НЕ працює:
    // let first = names[0];  // Помилка: cannot move out of index
    
    // Чому? Якби це працювало, names[0] стало б "дірою" —
    // вектор володів би недійсним значенням.
    
    // Правильні способи:
    
    // 1. Клонування — створюємо копію
    let first_clone = names[0].clone();
    println!("Клон: {}", first_clone);
    
    // 2. Позичання — посилання на елемент
    let first_ref = &names[0];
    println!("Посилання: {}", first_ref);
    
    // 3. remove — видаляємо з вектора і отримуємо ownership
    let mut names_mut = names;
    let removed = names_mut.remove(0);
    println!("Видалено: {}", removed);
    println!("Залишилось: {:?}", names_mut);  // ["Боб"]
}
```

### drain — вийняти діапазон елементів

```rust
fn main() {
    let mut vec = vec![0, 1, 2, 3, 4, 5, 6, 7, 8, 9];
    
    // drain повертає ітератор, що забирає ownership
    let middle: Vec<i32> = vec.drain(3..7).collect();
    
    println!("Вийняте: {:?}", middle);  // [3, 4, 5, 6]
    println!("Залишилось: {:?}", vec);  // [0, 1, 2, 7, 8, 9]
}
```

### Правила borrowing з Vec

Компілятор Rust захищає вас від помилок пам'яті:

```rust
fn main() {
    let mut vec = vec![1, 2, 3, 4, 5];
    
    // ❌ Помилка: push може реалокувати вектор,
    // інвалідуючи посилання first
    // let first = &vec[0];
    // vec.push(6);  // Помилка!
    // println!("{}", first);
    
    // ✅ Правильно: закінчити використання посилання перед мутацією
    let first = &vec[0];
    println!("Перший: {}", first);
    // first більше не використовується
    vec.push(6);  // OK
    
    // ❌ Помилка: два мутабельних посилання
    // let a = &mut vec[0];
    // let b = &mut vec[1];  // Компілятор не може довести, що це безпечно
    
    // ✅ Правильно: split_at_mut
    let (left, right) = vec.split_at_mut(2);
    // left = [1, 2], right = [3, 4, 5, 6]
    left[0] = 100;
    right[0] = 200;
    println!("Vec: {:?}", vec);  // [100, 2, 200, 4, 5, 6]
}
```

Чому `push` може інвалідувати посилання? Якщо вектор заповнений, `push` виділяє новий блок пам'яті, копіює дані, і звільняє старий. Старе посилання тепер вказує на звільнену пам'ять — use-after-free!

---

## 16.6 Практичне застосування: FlightRecorder

Створимо систему запису історії польоту агента. Ця програма демонструє практичне використання Vec для зберігання динамічних даних з методами аналізу.

```rust
use std::time::Instant;

#[derive(Debug, Clone, Copy)]
struct Position {
    x: f64,
    y: f64,
    z: f64,
}

impl Position {
    fn new(x: f64, y: f64, z: f64) -> Self {
        Self { x, y, z }
    }
    
    fn distance_to(&self, other: &Position) -> f64 {
        let dx = self.x - other.x;
        let dy = self.y - other.y;
        let dz = self.z - other.z;
        (dx*dx + dy*dy + dz*dz).sqrt()
    }
}

/// Запис позиції з часовою міткою
#[derive(Debug, Clone)]
struct PositionRecord {
    position: Position,
    timestamp: f64,  // Секунди від початку місії
}

/// Система запису історії польоту
struct FlightRecorder {
    records: Vec<PositionRecord>,
    mission_start: Instant,
}

impl FlightRecorder {
    /// Створює новий записувач
    fn new() -> Self {
        Self {
            records: Vec::new(),
            mission_start: Instant::now(),
        }
    }
    
    /// Записує поточну позицію
    fn record(&mut self, position: Position) {
        let timestamp = self.mission_start.elapsed().as_secs_f64();
        self.records.push(PositionRecord { position, timestamp });
    }
    
    /// Кількість записів
    fn count(&self) -> usize {
        self.records.len()
    }
    
    /// Загальна відстань польоту
    fn total_distance(&self) -> f64 {
        // windows(2) — ковзне вікно з 2 елементів
        // Для [A, B, C, D] дає: [A,B], [B,C], [C,D]
        self.records.windows(2)
            .map(|pair| pair[0].position.distance_to(&pair[1].position))
            .sum()
    }
    
    /// Максимальна висота за весь політ
    fn max_altitude(&self) -> Option<f64> {
        self.records.iter()
            .map(|r| r.position.z)
            .fold(None, |max, z| {
                Some(max.map_or(z, |m: f64| m.max(z)))
            })
    }
    
    /// Середня швидкість
    fn average_speed(&self) -> Option<f64> {
        if self.records.len() < 2 {
            return None;
        }
        
        let first = self.records.first()?;
        let last = self.records.last()?;
        let duration = last.timestamp - first.timestamp;
        
        if duration > 0.0 {
            Some(self.total_distance() / duration)
        } else {
            None
        }
    }
    
    /// Останні N позицій (для відображення треку)
    fn last_positions(&self, n: usize) -> Vec<Position> {
        self.records.iter()
            .rev()           // Від кінця
            .take(n)         // Взяти N
            .map(|r| r.position)
            .collect()
    }
    
    /// Позиції вище заданої висоти
    fn positions_above(&self, altitude: f64) -> Vec<&Position> {
        self.records.iter()
            .filter(|r| r.position.z > altitude)
            .map(|r| &r.position)
            .collect()
    }
}

fn main() {
    let mut recorder = FlightRecorder::new();
    
    // Симуляція польоту
    let path = vec![
        Position::new(0.0, 0.0, 0.0),      // Старт
        Position::new(0.0, 0.0, 50.0),     // Зліт
        Position::new(100.0, 0.0, 50.0),   // Рух на схід
        Position::new(100.0, 100.0, 80.0), // Рух на північ + підйом
        Position::new(0.0, 100.0, 60.0),   // Рух на захід
        Position::new(0.0, 0.0, 30.0),     // Повернення
        Position::new(0.0, 0.0, 0.0),      // Посадка
    ];
    
    for pos in path {
        recorder.record(pos);
        std::thread::sleep(std::time::Duration::from_millis(100));
    }
    
    println!("╔══════════════════════════════════════╗");
    println!("║       СТАТИСТИКА ПОЛЬОТУ             ║");
    println!("╚══════════════════════════════════════╝");
    println!("  Записів: {}", recorder.count());
    println!("  Загальна відстань: {:.1} м", recorder.total_distance());
    
    if let Some(max_alt) = recorder.max_altitude() {
        println!("  Максимальна висота: {:.1} м", max_alt);
    }
    
    if let Some(speed) = recorder.average_speed() {
        println!("  Середня швидкість: {:.1} м/с", speed);
    }
    
    println!("\nПозиції вище 50м: {}", recorder.positions_above(50.0).len());
    
    println!("\nОстанні 3 позиції:");
    for pos in recorder.last_positions(3) {
        println!("  ({:.0}, {:.0}, {:.0})", pos.x, pos.y, pos.z);
    }
}
```

Результат виконання:
```
╔══════════════════════════════════════╗
║       СТАТИСТИКА ПОЛЬОТУ             ║
╚══════════════════════════════════════╝
  Записів: 7
  Загальна відстань: 411.4 м
  Максимальна висота: 80.0 м
  Середня швидкість: 685.7 м/с
  
Позиції вище 50м: 3

Останні 3 позиції:
  (0, 0, 0)
  (0, 0, 30)
  (0, 100, 60)
```

---

## 16.7 Практичне застосування: CommandQueue

Тепер створимо чергу команд для агента. Команди надходять ззовні, агент виконує їх по черзі. Деякі команди мають вищий пріоритет (наприклад, ReturnToBase).

```rust
#[derive(Debug, Clone)]
struct Position {
    x: f64,
    y: f64,
    z: f64,
}

impl Position {
    fn new(x: f64, y: f64, z: f64) -> Self {
        Self { x, y, z }
    }
}

/// Команда для агента
#[derive(Debug, Clone)]
enum Command {
    TakeOff { altitude: f64 },
    MoveTo { target: Position },
    Hover { duration_secs: f64 },
    Scan { radius: f64 },
    Land,
    ReturnToBase,
}

impl Command {
    /// Пріоритет команди (більше = важливіше)
    fn priority(&self) -> u8 {
        match self {
            Command::ReturnToBase => 10,   // Найвищий
            Command::Land => 8,
            Command::TakeOff { .. } => 5,
            Command::MoveTo { .. } => 3,
            Command::Scan { .. } => 2,
            Command::Hover { .. } => 1,    // Найнижчий
        }
    }
    
    /// Назва команди для логування
    fn name(&self) -> &'static str {
        match self {
            Command::TakeOff { .. } => "TakeOff",
            Command::MoveTo { .. } => "MoveTo",
            Command::Hover { .. } => "Hover",
            Command::Scan { .. } => "Scan",
            Command::Land => "Land",
            Command::ReturnToBase => "ReturnToBase",
        }
    }
}

/// Черга команд з пріоритетами
struct CommandQueue {
    commands: Vec<Command>,
    max_size: usize,
}

impl CommandQueue {
    fn new(max_size: usize) -> Self {
        Self {
            commands: Vec::with_capacity(max_size),
            max_size,
        }
    }
    
    /// Додає команду в кінець черги
    fn enqueue(&mut self, command: Command) -> Result<(), &'static str> {
        if self.commands.len() >= self.max_size {
            return Err("Черга переповнена");
        }
        self.commands.push(command);
        Ok(())
    }
    
    /// Додає команду з урахуванням пріоритету
    fn enqueue_priority(&mut self, command: Command) -> Result<(), &'static str> {
        if self.commands.len() >= self.max_size {
            return Err("Черга переповнена");
        }
        
        let priority = command.priority();
        
        // Знаходимо позицію для вставки:
        // перший елемент з нижчим пріоритетом
        let insert_pos = self.commands.iter()
            .position(|c| c.priority() < priority)
            .unwrap_or(self.commands.len());
        
        self.commands.insert(insert_pos, command);
        Ok(())
    }
    
    /// Забирає наступну команду з черги
    fn dequeue(&mut self) -> Option<Command> {
        if self.commands.is_empty() {
            None
        } else {
            Some(self.commands.remove(0))
        }
    }
    
    /// Переглядає наступну команду без видалення
    fn peek(&self) -> Option<&Command> {
        self.commands.first()
    }
    
    /// Кількість команд у черзі
    fn len(&self) -> usize {
        self.commands.len()
    }
    
    /// Чи черга порожня
    fn is_empty(&self) -> bool {
        self.commands.is_empty()
    }
    
    /// Очищає чергу
    fn clear(&mut self) {
        self.commands.clear();
    }
    
    /// Видаляє всі команди певного типу
    fn remove_by_name(&mut self, name: &str) -> usize {
        let initial = self.commands.len();
        self.commands.retain(|c| c.name() != name);
        initial - self.commands.len()
    }
}

fn main() {
    let mut queue = CommandQueue::new(10);
    
    println!("╔══════════════════════════════════════╗");
    println!("║       ЧЕРГА КОМАНД АГЕНТА            ║");
    println!("╚══════════════════════════════════════╝\n");
    
    // Додаємо команди в звичайному порядку
    queue.enqueue(Command::TakeOff { altitude: 50.0 }).unwrap();
    queue.enqueue(Command::MoveTo { 
        target: Position::new(100.0, 0.0, 50.0) 
    }).unwrap();
    queue.enqueue(Command::Scan { radius: 30.0 }).unwrap();
    queue.enqueue(Command::Land).unwrap();
    
    println!("Черга після звичайного enqueue:");
    for (i, cmd) in queue.commands.iter().enumerate() {
        println!("  {}: {} (пріоритет: {})", i, cmd.name(), cmd.priority());
    }
    
    // Екстрене повернення — вставляємо з пріоритетом
    println!("\n--- Екстрене повернення на базу ---");
    queue.enqueue_priority(Command::ReturnToBase).unwrap();
    
    println!("\nЧерга після пріоритетного enqueue:");
    for (i, cmd) in queue.commands.iter().enumerate() {
        println!("  {}: {} (пріоритет: {})", i, cmd.name(), cmd.priority());
    }
    
    // Виконуємо команди
    println!("\n--- Виконання команд ---");
    while let Some(cmd) = queue.dequeue() {
        println!("Виконую: {:?}", cmd);
    }
    
    println!("\nЧерга порожня: {}", queue.is_empty());
}
```

Результат виконання:
```
╔══════════════════════════════════════╗
║       ЧЕРГА КОМАНД АГЕНТА            ║
╚══════════════════════════════════════╝

Черга після звичайного enqueue:
  0: TakeOff (пріоритет: 5)
  1: MoveTo (пріоритет: 3)
  2: Scan (пріоритет: 2)
  3: Land (пріоритет: 8)

--- Екстрене повернення на базу ---

Черга після пріоритетного enqueue:
  0: ReturnToBase (пріоритет: 10)
  1: TakeOff (пріоритет: 5)
  2: MoveTo (пріоритет: 3)
  3: Scan (пріоритет: 2)
  4: Land (пріоритет: 8)

--- Виконання команд ---
Виконую: ReturnToBase
Виконую: TakeOff { altitude: 50.0 }
Виконую: MoveTo { target: Position { x: 100.0, y: 0.0, z: 50.0 } }
Виконую: Scan { radius: 30.0 }
Виконую: Land

Черга порожня: true
```

---

## 16.8 Лабораторна робота

Створіть систему відстеження об'єктів для агента БПЛА.

**Завдання:**

1. Створіть структуру `DetectedObject`:
   - `id: u32` — унікальний ідентифікатор
   - `object_type: ObjectType` — enum (Vehicle, Person, Building, Unknown)
   - `position: Position`
   - `confidence: f64` — впевненість (0.0-1.0)
   - `timestamp: f64` — час виявлення

2. Створіть структуру `ObjectTracker`:
   - `objects: Vec<DetectedObject>`
   - Метод `register()` — додає новий об'єкт або оновлює існуючий
   - Метод `by_type()` — повертає об'єкти певного типу
   - Метод `in_radius()` — об'єкти в радіусі від точки
   - Метод `remove_stale()` — видаляє об'єкти старші за поріг

3. Напишіть тести для всіх методів

**Критерії оцінювання:**

| Критерій | Бали |
|----------|------|
| Структури DetectedObject та ObjectType | 2 |
| Базовий ObjectTracker | 2 |
| Метод register з оновленням | 2 |
| Методи фільтрації | 2 |
| Тести | 2 |
| **Максимум** | **10** |

---

## Підсумок

`Vec<T>` — це робоча конячка колекцій Rust. Ви навчились:

- **Створювати** вектори: `Vec::new()`, `vec![]`, `Vec::with_capacity()`
- **Модифікувати**: `push`, `pop`, `insert`, `remove`, `retain`, `clear`
- **Отримувати доступ**: індексація, `get`, `first`, `last`, слайси
- **Ітерувати**: `iter()`, `iter_mut()`, `into_iter()`, та їх відмінності
- **Трансформувати**: `map`, `filter`, `fold`, `collect` та інші методи ітераторів
- **Розуміти ownership**: вектор володіє елементами, правила borrowing

Ключові моменти:
- Вектор росте автоматично, але `with_capacity` уникає реалокацій
- `get()` безпечніший за індексацію — повертає Option
- Три види ітераторів для різних потреб ownership
- `windows()` — ковзне вікно для попарної обробки

---

## Зв'язок з наступним розділом

Вектор зберігає послідовність елементів. Але що, якщо потрібен швидкий пошук за ключем? Наприклад, знайти об'єкт за ID, або перевірити, чи клітинка карти зайнята?

У **Розділі 17: HashMap та просторові структури** ви навчитесь використовувати асоціативні масиви для швидкого пошуку і побудуєте карту світу агента.
