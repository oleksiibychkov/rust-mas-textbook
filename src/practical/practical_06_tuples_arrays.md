# Практичне заняття 6: Кортежі та масиви

## Мета заняття

Після цього заняття ви зможете:
- Створювати та використовувати кортежі для групування різнотипних даних
- Працювати з масивами фіксованого розміру
- Застосовувати деструктуризацію для розпакування значень
- Розуміти слайси як "вікна" в дані
- Відрізняти масиви від векторів

---

## Теоретичний вступ

### Навіщо групувати дані?

Часто дані логічно пов'язані між собою. Координата — це не просто два числа, а пара (x, y). Показання сенсора — це значення та час. Групування даних робить код зрозумілішим та безпечнішим.

У Rust є кілька способів групування:
- **Кортежі (tuples)** — для фіксованої кількості різнотипних значень
- **Масиви (arrays)** — для фіксованої кількості однотипних значень
- **Структури (structs)** — для іменованих полів (вивчимо пізніше)

---

## Кортежі (Tuples)

### Що таке кортеж?

Кортеж — це впорядкована колекція фіксованого розміру, де кожен елемент може мати **різний тип**.

```rust
fn main() {
    // Кортеж з трьох елементів різних типів
    let drone_info: (i32, &str, bool) = (1, "Alpha", true);
    
    println!("ID: {}", drone_info.0);       // 1
    println!("Назва: {}", drone_info.1);    // Alpha
    println!("Активний: {}", drone_info.2); // true
}
```

### Створення кортежів

```rust
fn main() {
    // Явне зазначення типу
    let point: (i32, i32) = (10, 20);
    
    // Тип виводиться автоматично
    let mixed = (42, "hello", 3.14, true);
    
    // Порожній кортеж (unit type)
    let empty: () = ();
    
    // Кортеж з одного елемента (потрібна кома!)
    let single = (42,);  // Це кортеж
    let not_tuple = (42); // Це просто число в дужках
}
```

### Доступ до елементів

Доступ через крапку та індекс (починаючи з 0):

```rust
fn main() {
    let position = (100, 250, 50);
    
    let x = position.0;  // 100
    let y = position.1;  // 250
    let z = position.2;  // 50
    
    println!("Позиція: x={}, y={}, z={}", x, y, z);
}
```

**Важливо:** Індекси мають бути відомі на етапі компіляції. Не можна писати `position.i` де `i` — змінна.

### Деструктуризація кортежів

Елегантний спосіб "розпакувати" кортеж:

```rust
fn main() {
    let position = (100, 250, 50);
    
    // Деструктуризація — створюємо три змінні одразу
    let (x, y, z) = position;
    
    println!("x={}, y={}, z={}", x, y, z);
}
```

### Ігнорування елементів при деструктуризації

Якщо деякі елементи не потрібні:

```rust
fn main() {
    let data = (1, "Alpha", 85.5, true);
    
    // Ігноруємо непотрібні елементи через _
    let (id, _, battery, _) = data;
    
    println!("ID: {}, Батарея: {}%", id, battery);
}
```

### Кортежі у функціях

Кортежі часто використовують для повернення кількох значень:

```rust
fn get_min_max(numbers: &[i32]) -> (i32, i32) {
    let mut min = numbers[0];
    let mut max = numbers[0];
    
    for &n in numbers {
        if n < min { min = n; }
        if n > max { max = n; }
    }
    
    (min, max)  // Повертаємо кортеж
}

fn main() {
    let values = [5, 2, 9, 1, 7, 3];
    
    let (min, max) = get_min_max(&values);
    
    println!("Мін: {}, Макс: {}", min, max);
}
```

### Вкладені кортежі

```rust
fn main() {
    // Позиція та швидкість
    let state: ((i32, i32), (f64, f64)) = ((100, 200), (5.5, -2.3));
    
    let position = state.0;
    let velocity = state.1;
    
    println!("Позиція: ({}, {})", position.0, position.1);
    println!("Швидкість: ({}, {})", velocity.0, velocity.1);
    
    // Або повна деструктуризація
    let ((x, y), (vx, vy)) = state;
    println!("x={}, y={}, vx={}, vy={}", x, y, vx, vy);
}
```

---

## Масиви (Arrays)

### Що таке масив?

Масив — це колекція **фіксованого розміру** з елементами **одного типу**. Розмір масиву — частина його типу!

```rust
fn main() {
    // Масив з 5 елементів типу i32
    let numbers: [i32; 5] = [1, 2, 3, 4, 5];
    
    println!("Перший: {}", numbers[0]);
    println!("Останній: {}", numbers[4]);
    println!("Довжина: {}", numbers.len());
}
```

### Синтаксис типу масиву

```
[ТипЕлемента; Розмір]
```

Приклади:
- `[i32; 5]` — масив з 5 цілих чисел
- `[f64; 3]` — масив з 3 дійсних чисел
- `[bool; 10]` — масив з 10 булевих значень
- `[&str; 4]` — масив з 4 рядків

### Створення масивів

```rust
fn main() {
    // Явне перелічення
    let days = ["Пн", "Вт", "Ср", "Чт", "Пт", "Сб", "Нд"];
    
    // Заповнення однаковим значенням
    let zeros = [0; 10];  // [0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
    
    // Масив за замовчуванням
    let flags = [false; 5];  // [false, false, false, false, false]
    
    println!("Дні: {:?}", days);
    println!("Нулі: {:?}", zeros);
    println!("Прапорці: {:?}", flags);
}
```

### Доступ до елементів

```rust
fn main() {
    let sensors = [25.5, 26.0, 24.8, 25.2, 25.7];
    
    // Доступ за індексом
    let first = sensors[0];
    let third = sensors[2];
    
    // ПОМИЛКА під час виконання! Вихід за межі
    // let invalid = sensors[10];
    
    // Безпечний доступ
    match sensors.get(10) {
        Some(value) => println!("Значення: {}", value),
        None => println!("Індекс за межами масиву"),
    }
}
```

### Ітерація по масиву

```rust
fn main() {
    let readings = [10, 20, 30, 40, 50];
    
    // Ітерація по значеннях
    for value in readings {
        println!("Значення: {}", value);
    }
    
    // Ітерація з індексом
    for (i, value) in readings.iter().enumerate() {
        println!("[{}] = {}", i, value);
    }
}
```

### Мутабельні масиви

```rust
fn main() {
    let mut counters = [0; 5];
    
    counters[0] = 10;
    counters[2] = 30;
    counters[4] += 1;
    
    println!("Лічильники: {:?}", counters);
}
```

### Масиви vs Вектори

| Характеристика | Масив `[T; N]` | Вектор `Vec<T>` |
|----------------|----------------|-----------------|
| Розмір | Фіксований, відомий при компіляції | Динамічний |
| Пам'ять | На стеку | На купі (heap) |
| Зміна розміру | Неможлива | push, pop, etc. |
| Швидкість | Швидший | Трохи повільніший |
| Використання | Коли розмір відомий | Коли розмір змінюється |

```rust
fn main() {
    // Масив — розмір фіксований
    let array = [1, 2, 3];
    // array.push(4);  // ПОМИЛКА! Неможливо
    
    // Вектор — розмір динамічний
    let mut vector = vec![1, 2, 3];
    vector.push(4);  // ОК!
    
    println!("Вектор: {:?}", vector);
}
```

---

## Слайси (Slices)

### Що таке слайс?

Слайс — це "вікно" або "вид" на послідовність елементів. Слайс не володіє даними, а лише посилається на них.

```rust
fn main() {
    let array = [1, 2, 3, 4, 5];
    
    // Слайс на весь масив
    let all: &[i32] = &array;
    
    // Слайс на частину масиву
    let middle: &[i32] = &array[1..4];  // [2, 3, 4]
    
    println!("Весь: {:?}", all);
    println!("Середина: {:?}", middle);
}
```

### Синтаксис слайсів

```rust
fn main() {
    let arr = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9];
    
    let a = &arr[2..5];   // [2, 3, 4] — з 2 до 5 (не включаючи)
    let b = &arr[..3];    // [0, 1, 2] — з початку до 3
    let c = &arr[7..];    // [7, 8, 9] — з 7 до кінця
    let d = &arr[..];     // Весь масив
    let e = &arr[2..=5];  // [2, 3, 4, 5] — з 2 до 5 (включаючи)
    
    println!("a: {:?}", a);
    println!("b: {:?}", b);
    println!("c: {:?}", c);
    println!("d: {:?}", d);
    println!("e: {:?}", e);
}
```

### Слайси у функціях

Слайси дозволяють писати функції, що працюють і з масивами, і з векторами:

```rust
fn sum(numbers: &[i32]) -> i32 {
    let mut total = 0;
    for n in numbers {
        total += n;
    }
    total
}

fn main() {
    let array = [1, 2, 3, 4, 5];
    let vector = vec![10, 20, 30];
    
    // Обидва працюють!
    println!("Сума масиву: {}", sum(&array));
    println!("Сума вектора: {}", sum(&vector));
    
    // Також можна передати частину
    println!("Сума [1..3]: {}", sum(&array[1..3]));
}
```

### Мутабельні слайси

```rust
fn double_all(numbers: &mut [i32]) {
    for n in numbers {
        *n *= 2;
    }
}

fn main() {
    let mut values = [1, 2, 3, 4, 5];
    
    println!("До: {:?}", values);
    double_all(&mut values);
    println!("Після: {:?}", values);
}
```

---

## Практичні патерни

### Патерн: Позиція як кортеж

```rust
type Position = (i32, i32, i32);  // Псевдонім типу

fn distance(p1: Position, p2: Position) -> f64 {
    let (x1, y1, z1) = p1;
    let (x2, y2, z2) = p2;
    
    let dx = (x2 - x1) as f64;
    let dy = (y2 - y1) as f64;
    let dz = (z2 - z1) as f64;
    
    (dx*dx + dy*dy + dz*dz).sqrt()
}

fn main() {
    let drone = (100, 200, 50);
    let target = (150, 250, 50);
    
    println!("Відстань: {:.2}", distance(drone, target));
}
```

### Патерн: Буфер фіксованого розміру

```rust
fn main() {
    const BUFFER_SIZE: usize = 5;
    let mut buffer: [i32; BUFFER_SIZE] = [0; BUFFER_SIZE];
    let mut index = 0;
    
    // Симуляція додавання даних у циклічний буфер
    for value in [10, 20, 30, 40, 50, 60, 70] {
        buffer[index % BUFFER_SIZE] = value;
        index += 1;
        println!("Буфер: {:?}", buffer);
    }
}
```

### Патерн: Знаходження в масиві

```rust
fn find_index(arr: &[i32], target: i32) -> Option<usize> {
    for (i, &value) in arr.iter().enumerate() {
        if value == target {
            return Some(i);
        }
    }
    None
}

fn main() {
    let data = [5, 10, 15, 20, 25];
    
    match find_index(&data, 15) {
        Some(i) => println!("Знайдено на позиції {}", i),
        None => println!("Не знайдено"),
    }
}
```

---

## Типові помилки та їх виправлення

### Помилка 1: Вихід за межі масиву

```rust
fn main() {
    let arr = [1, 2, 3];
    let x = arr[5];  // ПАНІКА під час виконання!
}
```

**Виправлення:** Використовуйте `.get()` для безпечного доступу:
```rust
if let Some(x) = arr.get(5) {
    println!("{}", x);
}
```

### Помилка 2: Неправильний тип масиву

```rust
fn process(data: [i32; 3]) { }

fn main() {
    let arr = [1, 2, 3, 4, 5];  // [i32; 5]
    process(arr);  // ПОМИЛКА! [i32; 5] != [i32; 3]
}
```

**Виправлення:** Використовуйте слайси для гнучкості:
```rust
fn process(data: &[i32]) { }  // Приймає будь-який розмір
```

### Помилка 3: Забута кома в кортежі з одного елемента

```rust
fn main() {
    let tuple = (42);   // Це i32, не кортеж!
    let real_tuple = (42,);  // Це кортеж з одного елемента
    
    println!("Тип: {}", std::any::type_name_of_val(&tuple));
    // Виведе: i32
}
```

### Помилка 4: Спроба змінити розмір масиву

```rust
fn main() {
    let mut arr = [1, 2, 3];
    // arr.push(4);  // ПОМИЛКА! Масиви не мають push
    
    // Використовуйте вектор для динамічного розміру
    let mut vec = vec![1, 2, 3];
    vec.push(4);  // ОК
}
```

---

## Практичні задачі

### Задача 1: Позиція дрона як кортеж

**Умова:** Створіть кортеж для представлення позиції дрона (x, y, висота). Напишіть функцію `format_position`, яка форматує позицію у рядок. Напишіть функцію `move_drone`, яка повертає нову позицію після переміщення на задані величини.

**Розв'язання:**

```rust
// Псевдонім для зручності
type Position = (i32, i32, i32);

fn format_position(pos: Position) -> String {
    let (x, y, alt) = pos;
    format!("({}, {}, висота: {} м)", x, y, alt)
}

fn move_drone(pos: Position, dx: i32, dy: i32, dalt: i32) -> Position {
    let (x, y, alt) = pos;
    (x + dx, y + dy, alt + dalt)
}

fn main() {
    println!("=== Робота з позицією дрона ===\n");
    
    // Початкова позиція
    let mut position: Position = (0, 0, 50);
    println!("Старт: {}", format_position(position));
    
    // Серія переміщень
    let movements = [
        (100, 0, 0, "вправо"),
        (0, 50, 0, "вперед"),
        (0, 0, 25, "вгору"),
        (-50, 25, -10, "діагонально"),
    ];
    
    for (dx, dy, dalt, description) in movements {
        position = move_drone(position, dx, dy, dalt);
        println!("Після руху {}: {}", description, format_position(position));
    }
    
    // Фінальна позиція через деструктуризацію
    let (final_x, final_y, final_alt) = position;
    println!("\n=== Підсумок ===");
    println!("Фінальні координати: X={}, Y={}, Висота={} м", 
             final_x, final_y, final_alt);
}
```

**Пояснення:**

1. `type Position = (i32, i32, i32)` — псевдонім типу для читабельності
2. Деструктуризація `let (x, y, alt) = pos` — розпаковуємо кортеж
3. `move_drone` повертає новий кортеж (кортежі копіюються, бо всі елементи Copy)
4. Масив кортежів `movements` для зберігання послідовності рухів

**Результат:**
```
=== Робота з позицією дрона ===

Старт: (0, 0, висота: 50 м)
Після руху вправо: (100, 0, висота: 50 м)
Після руху вперед: (100, 50, висота: 50 м)
Після руху вгору: (100, 50, висота: 75 м)
Після руху діагонально: (50, 75, висота: 65 м)

=== Підсумок ===
Фінальні координати: X=50, Y=75, Висота=65 м
```

---

### Задача 2: Масив показань сенсорів

**Умова:** Дрон має 5 датчиків температури. Створіть масив з показаннями та напишіть функції для обчислення: середнього значення, мінімуму, максимуму. Визначте, чи є критичні показання (> 50°C).

**Розв'язання:**

```rust
const SENSOR_COUNT: usize = 5;
const CRITICAL_TEMP: f64 = 50.0;

fn average(readings: &[f64]) -> f64 {
    let sum: f64 = readings.iter().sum();
    sum / readings.len() as f64
}

fn min_max(readings: &[f64]) -> (f64, f64) {
    let mut min = readings[0];
    let mut max = readings[0];
    
    for &value in readings {
        if value < min { min = value; }
        if value > max { max = value; }
    }
    
    (min, max)
}

fn find_critical(readings: &[f64], threshold: f64) -> Vec<usize> {
    let mut critical = Vec::new();
    
    for (i, &value) in readings.iter().enumerate() {
        if value > threshold {
            critical.push(i);
        }
    }
    
    critical
}

fn main() {
    println!("=== Аналіз показань сенсорів ===\n");
    
    // Показання з 5 датчиків
    let readings: [f64; SENSOR_COUNT] = [42.5, 38.0, 55.2, 41.8, 52.1];
    
    println!("Показання датчиків:");
    for (i, &temp) in readings.iter().enumerate() {
        let status = if temp > CRITICAL_TEMP { " ⚠️ КРИТИЧНО!" } else { "" };
        println!("  Датчик {}: {:.1}°C{}", i + 1, temp, status);
    }
    
    // Статистика
    let avg = average(&readings);
    let (min, max) = min_max(&readings);
    
    println!("\n=== Статистика ===");
    println!("Середня температура: {:.1}°C", avg);
    println!("Мінімум: {:.1}°C", min);
    println!("Максимум: {:.1}°C", max);
    println!("Діапазон: {:.1}°C", max - min);
    
    // Критичні показання
    let critical = find_critical(&readings, CRITICAL_TEMP);
    
    if critical.is_empty() {
        println!("\n✓ Всі показання в нормі");
    } else {
        println!("\n⚠️  Критичні датчики: {:?}", critical);
        println!("   Потрібна перевірка!");
    }
}
```

**Пояснення:**

1. `const SENSOR_COUNT: usize = 5` — константа для розміру масиву
2. `readings.iter().sum()` — сума всіх елементів
3. `&[f64]` — слайс дозволяє функції приймати масив будь-якого розміру
4. `find_critical` повертає `Vec<usize>` — вектор індексів критичних датчиків

**Результат:**
```
=== Аналіз показань сенсорів ===

Показання датчиків:
  Датчик 1: 42.5°C
  Датчик 2: 38.0°C
  Датчик 3: 55.2°C ⚠️ КРИТИЧНО!
  Датчик 4: 41.8°C
  Датчик 5: 52.1°C ⚠️ КРИТИЧНО!

=== Статистика ===
Середня температура: 45.9°C
Мінімум: 38.0°C
Максимум: 55.2°C
Діапазон: 17.2°C

⚠️  Критичні датчики: [2, 4]
   Потрібна перевірка!
```

---

### Задача 3: Деструктуризація команд

**Умова:** Команди для дрона представлені кортежами: (код_команди, параметр1, параметр2). Напишіть функцію, яка інтерпретує команду та виводить опис дії. Коди: 1 = рух, 2 = поворот, 3 = зміна висоти.

**Розв'язання:**

```rust
type Command = (i32, i32, i32);

fn interpret_command(cmd: Command) -> String {
    let (code, param1, param2) = cmd;
    
    match code {
        1 => format!("РУХ: переміститись на ({}, {})", param1, param2),
        2 => {
            let direction = if param1 > 0 { "праворуч" } else { "ліворуч" };
            format!("ПОВОРОТ: {} на {}°", direction, param1.abs())
        },
        3 => {
            let direction = if param1 > 0 { "підйом" } else { "спуск" };
            format!("ВИСОТА: {} на {} м (швидкість: {} м/с)", 
                    direction, param1.abs(), param2)
        },
        0 => String::from("СТОП: зупинити всі двигуни"),
        _ => format!("НЕВІДОМА КОМАНДА: код={}", code),
    }
}

fn execute_commands(commands: &[Command]) {
    println!("Виконання {} команд:\n", commands.len());
    
    for (i, &cmd) in commands.iter().enumerate() {
        let description = interpret_command(cmd);
        println!("{}. {:?} → {}", i + 1, cmd, description);
    }
}

fn main() {
    println!("=== Інтерпретатор команд дрона ===\n");
    
    // Масив команд
    let mission: [Command; 6] = [
        (1, 100, 50),   // Рух
        (3, 25, 5),     // Підйом
        (2, 90, 0),     // Поворот праворуч
        (1, 200, 0),    // Рух
        (3, -30, 3),    // Спуск
        (0, 0, 0),      // Стоп
    ];
    
    execute_commands(&mission);
    
    // Тест невідомої команди
    println!("\n=== Тест невідомої команди ===");
    let unknown: Command = (99, 1, 2);
    println!("{:?} → {}", unknown, interpret_command(unknown));
}
```

**Пояснення:**

1. `type Command = (i32, i32, i32)` — псевдонім для читабельності
2. `match code { }` — розбір коду команди
3. `param1.abs()` — абсолютне значення (модуль) числа
4. `&[Command]` — слайс кортежів
5. `&cmd` — отримуємо копію кортежу (бо він Copy)

**Результат:**
```
=== Інтерпретатор команд дрона ===

Виконання 6 команд:

1. (1, 100, 50) → РУХ: переміститись на (100, 50)
2. (3, 25, 5) → ВИСОТА: підйом на 25 м (швидкість: 5 м/с)
3. (2, 90, 0) → ПОВОРОТ: праворуч на 90°
4. (1, 200, 0) → РУХ: переміститись на (200, 0)
5. (3, -30, 3) → ВИСОТА: спуск на 30 м (швидкість: 3 м/с)
6. (0, 0, 0) → СТОП: зупинити всі двигуни

=== Тест невідомої команди ===
(99, 1, 2) → НЕВІДОМА КОМАНДА: код=99
```

---

### Задача 4: Обчислення статистики слайсу

**Умова:** Напишіть функцію `analyze_slice`, яка приймає слайс чисел та повертає кортеж зі статистикою: (сума, середнє, кількість додатних, кількість від'ємних). Протестуйте на різних частинах масиву.

**Розв'язання:**

```rust
fn analyze_slice(numbers: &[i32]) -> (i32, f64, usize, usize) {
    if numbers.is_empty() {
        return (0, 0.0, 0, 0);
    }
    
    let mut sum = 0;
    let mut positive = 0;
    let mut negative = 0;
    
    for &n in numbers {
        sum += n;
        if n > 0 { positive += 1; }
        if n < 0 { negative += 1; }
    }
    
    let average = sum as f64 / numbers.len() as f64;
    
    (sum, average, positive, negative)
}

fn print_analysis(name: &str, numbers: &[i32]) {
    let (sum, avg, pos, neg) = analyze_slice(numbers);
    
    println!("--- {} ---", name);
    println!("Дані: {:?}", numbers);
    println!("Сума: {}", sum);
    println!("Середнє: {:.2}", avg);
    println!("Додатних: {}, Від'ємних: {}, Нулів: {}", 
             pos, neg, numbers.len() - pos - neg);
    println!();
}

fn main() {
    println!("=== Аналіз слайсів ===\n");
    
    let data = [-5, 3, 0, 8, -2, 7, 1, -3, 4, 0];
    
    // Аналіз всього масиву
    print_analysis("Весь масив", &data);
    
    // Аналіз першої половини
    print_analysis("Перша половина", &data[..5]);
    
    // Аналіз другої половини
    print_analysis("Друга половина", &data[5..]);
    
    // Аналіз середини
    print_analysis("Середина [2..8]", &data[2..8]);
    
    // Аналіз тільки додатних (перших трьох додатних)
    let positives: Vec<i32> = data.iter()
        .filter(|&&x| x > 0)
        .take(3)
        .copied()
        .collect();
    print_analysis("Перші 3 додатних", &positives);
}
```

**Пояснення:**

1. Функція повертає кортеж з 4 елементів різних типів
2. `numbers.is_empty()` — перевірка на порожній слайс
3. `&data[..5]` — слайс перших 5 елементів
4. `&data[5..]` — слайс з 6-го елемента до кінця
5. Деструктуризація `let (sum, avg, pos, neg) = ...` розпаковує результат

**Результат:**
```
=== Аналіз слайсів ===

--- Весь масив ---
Дані: [-5, 3, 0, 8, -2, 7, 1, -3, 4, 0]
Сума: 13
Середнє: 1.30
Додатних: 5, Від'ємних: 3, Нулів: 2

--- Перша половина ---
Дані: [-5, 3, 0, 8, -2]
Сума: 4
Середнє: 0.80
Додатних: 2, Від'ємних: 2, Нулів: 1

--- Друга половина ---
Дані: [7, 1, -3, 4, 0]
Сума: 9
Середнє: 1.80
Додатних: 3, Від'ємних: 1, Нулів: 1

--- Середина [2..8] ---
Дані: [0, 8, -2, 7, 1, -3]
Сума: 11
Середнє: 1.83
Додатних: 3, Від'ємних: 2, Нулів: 1

--- Перші 3 додатних ---
Дані: [3, 8, 7]
Сума: 18
Середнє: 6.00
Додатних: 3, Від'ємних: 0, Нулів: 0
```

---

## Домашнє завдання

### Завдання 1: Мінімум та максимум як кортеж

**Умова:** Напишіть функцію `find_min_max_with_indices`, яка приймає слайс чисел та повертає кортеж: ((мін_значення, мін_індекс), (макс_значення, макс_індекс)).

**Розв'язання:**

```rust
fn find_min_max_with_indices(numbers: &[i32]) -> ((i32, usize), (i32, usize)) {
    if numbers.is_empty() {
        panic!("Масив не може бути порожнім!");
    }
    
    let mut min_val = numbers[0];
    let mut min_idx = 0;
    let mut max_val = numbers[0];
    let mut max_idx = 0;
    
    for (i, &n) in numbers.iter().enumerate() {
        if n < min_val {
            min_val = n;
            min_idx = i;
        }
        if n > max_val {
            max_val = n;
            max_idx = i;
        }
    }
    
    ((min_val, min_idx), (max_val, max_idx))
}

fn main() {
    println!("=== Пошук мін/макс з індексами ===\n");
    
    let datasets = [
        vec![5, 2, 9, 1, 7, 3],
        vec![10, 10, 10],
        vec![-5, -2, -8, -1],
        vec![42],
    ];
    
    for data in &datasets {
        let ((min_v, min_i), (max_v, max_i)) = find_min_max_with_indices(data);
        
        println!("Дані: {:?}", data);
        println!("  Мінімум: {} на позиції {}", min_v, min_i);
        println!("  Максимум: {} на позиції {}", max_v, max_i);
        println!();
    }
}
```

**Результат:**
```
=== Пошук мін/макс з індексами ===

Дані: [5, 2, 9, 1, 7, 3]
  Мінімум: 1 на позиції 3
  Максимум: 9 на позиції 2

Дані: [10, 10, 10]
  Мінімум: 10 на позиції 0
  Максимум: 10 на позиції 0

Дані: [-5, -2, -8, -1]
  Мінімум: -8 на позиції 2
  Максимум: -1 на позиції 3

Дані: [42]
  Мінімум: 42 на позиції 0
  Максимум: 42 на позиції 0
```

---

### Завдання 2: Маршрут з точок

**Умова:** Маршрут дрона — це масив точок (x, y). Напишіть функцію `calculate_route_length`, яка обчислює загальну довжину маршруту (суму відстаней між послідовними точками). Також напишіть функцію `find_longest_segment`, яка знаходить найдовший сегмент маршруту.

**Розв'язання:**

```rust
type Point = (f64, f64);

fn distance(p1: Point, p2: Point) -> f64 {
    let (x1, y1) = p1;
    let (x2, y2) = p2;
    ((x2 - x1).powi(2) + (y2 - y1).powi(2)).sqrt()
}

fn calculate_route_length(waypoints: &[Point]) -> f64 {
    if waypoints.len() < 2 {
        return 0.0;
    }
    
    let mut total = 0.0;
    
    for i in 0..waypoints.len() - 1 {
        total += distance(waypoints[i], waypoints[i + 1]);
    }
    
    total
}

fn find_longest_segment(waypoints: &[Point]) -> Option<(usize, f64)> {
    if waypoints.len() < 2 {
        return None;
    }
    
    let mut max_dist = 0.0;
    let mut max_idx = 0;
    
    for i in 0..waypoints.len() - 1 {
        let d = distance(waypoints[i], waypoints[i + 1]);
        if d > max_dist {
            max_dist = d;
            max_idx = i;
        }
    }
    
    Some((max_idx, max_dist))
}

fn main() {
    println!("=== Аналіз маршруту ===\n");
    
    let route: [Point; 5] = [
        (0.0, 0.0),
        (3.0, 4.0),
        (3.0, 10.0),
        (10.0, 10.0),
        (10.0, 0.0),
    ];
    
    println!("Точки маршруту:");
    for (i, point) in route.iter().enumerate() {
        println!("  {}: ({:.1}, {:.1})", i, point.0, point.1);
    }
    
    // Загальна довжина
    let total_length = calculate_route_length(&route);
    println!("\nЗагальна довжина маршруту: {:.2} одиниць", total_length);
    
    // Довжина кожного сегмента
    println!("\nСегменти:");
    for i in 0..route.len() - 1 {
        let d = distance(route[i], route[i + 1]);
        println!("  {} → {}: {:.2}", i, i + 1, d);
    }
    
    // Найдовший сегмент
    if let Some((idx, length)) = find_longest_segment(&route) {
        println!("\nНайдовший сегмент: {} → {} ({:.2} одиниць)", 
                 idx, idx + 1, length);
    }
}
```

**Результат:**
```
=== Аналіз маршруту ===

Точки маршруту:
  0: (0.0, 0.0)
  1: (3.0, 4.0)
  2: (3.0, 10.0)
  3: (10.0, 10.0)
  4: (10.0, 0.0)

Загальна довжина маршруту: 29.00 одиниць

Сегменти:
  0 → 1: 5.00
  1 → 2: 6.00
  2 → 3: 7.00
  3 → 4: 10.00

Найдовший сегмент: 3 → 4 (10.00 одиниць)
```

---

### Завдання 3: Пошук елемента в масиві

**Умова:** Напишіть функцію `find_all_indices`, яка знаходить всі позиції заданого елемента в масиві. Поверніть вектор індексів.

**Розв'язання:**

```rust
fn find_all_indices<T: PartialEq>(arr: &[T], target: &T) -> Vec<usize> {
    let mut indices = Vec::new();
    
    for (i, item) in arr.iter().enumerate() {
        if item == target {
            indices.push(i);
        }
    }
    
    indices
}

fn main() {
    println!("=== Пошук всіх входжень ===\n");
    
    // Пошук числа
    let numbers = [1, 5, 3, 5, 7, 5, 9, 5];
    let target = 5;
    let positions = find_all_indices(&numbers, &target);
    
    println!("Масив: {:?}", numbers);
    println!("Шукаємо: {}", target);
    println!("Знайдено на позиціях: {:?}", positions);
    println!("Кількість входжень: {}\n", positions.len());
    
    // Пошук рядка
    let drones = ["Alpha", "Beta", "Alpha", "Gamma", "Alpha", "Delta"];
    let search = "Alpha";
    let drone_positions = find_all_indices(&drones, &search);
    
    println!("Дрони: {:?}", drones);
    println!("Шукаємо: {}", search);
    println!("Знайдено на позиціях: {:?}", drone_positions);
    
    // Пошук без результату
    let empty = find_all_indices(&numbers, &100);
    println!("\nПошук 100: {:?} (порожній = не знайдено)", empty);
}
```

**Результат:**
```
=== Пошук всіх входжень ===

Масив: [1, 5, 3, 5, 7, 5, 9, 5]
Шукаємо: 5
Знайдено на позиціях: [1, 3, 5, 7]
Кількість входжень: 4

Дрони: ["Alpha", "Beta", "Alpha", "Gamma", "Alpha", "Delta"]
Шукаємо: Alpha
Знайдено на позиціях: [0, 2, 4]

Пошук 100: [] (порожній = не знайдено)
```

---

### Завдання 4: Статистика координат

**Умова:** Масив позицій дронів представлено як `[(x, y, z); N]`. Напишіть функції для обчислення:
- Центру мас (середні x, y, z)
- Найвищого та найнижчого дрона
- Дрона, найближчого до заданої точки

**Розв'язання:**

```rust
type Position = (f64, f64, f64);

fn center_of_mass(positions: &[Position]) -> Position {
    if positions.is_empty() {
        return (0.0, 0.0, 0.0);
    }
    
    let mut sum_x = 0.0;
    let mut sum_y = 0.0;
    let mut sum_z = 0.0;
    
    for &(x, y, z) in positions {
        sum_x += x;
        sum_y += y;
        sum_z += z;
    }
    
    let n = positions.len() as f64;
    (sum_x / n, sum_y / n, sum_z / n)
}

fn find_highest_lowest(positions: &[Position]) -> (usize, usize) {
    let mut highest_idx = 0;
    let mut lowest_idx = 0;
    let mut max_z = positions[0].2;
    let mut min_z = positions[0].2;
    
    for (i, &(_, _, z)) in positions.iter().enumerate() {
        if z > max_z {
            max_z = z;
            highest_idx = i;
        }
        if z < min_z {
            min_z = z;
            lowest_idx = i;
        }
    }
    
    (highest_idx, lowest_idx)
}

fn distance_3d(p1: Position, p2: Position) -> f64 {
    let (x1, y1, z1) = p1;
    let (x2, y2, z2) = p2;
    ((x2-x1).powi(2) + (y2-y1).powi(2) + (z2-z1).powi(2)).sqrt()
}

fn find_nearest(positions: &[Position], target: Position) -> usize {
    let mut nearest_idx = 0;
    let mut min_dist = distance_3d(positions[0], target);
    
    for (i, &pos) in positions.iter().enumerate() {
        let d = distance_3d(pos, target);
        if d < min_dist {
            min_dist = d;
            nearest_idx = i;
        }
    }
    
    nearest_idx
}

fn main() {
    println!("=== Статистика позицій дронів ===\n");
    
    let drones: [Position; 5] = [
        (10.0, 20.0, 100.0),
        (50.0, 30.0, 150.0),
        (30.0, 60.0, 80.0),
        (70.0, 40.0, 200.0),
        (20.0, 50.0, 120.0),
    ];
    
    let names = ["Alpha", "Beta", "Gamma", "Delta", "Epsilon"];
    
    println!("Позиції дронів:");
    for (i, &(x, y, z)) in drones.iter().enumerate() {
        println!("  {}: ({:.0}, {:.0}, {:.0})", names[i], x, y, z);
    }
    
    // Центр мас
    let (cx, cy, cz) = center_of_mass(&drones);
    println!("\nЦентр мас рою: ({:.1}, {:.1}, {:.1})", cx, cy, cz);
    
    // Найвищий та найнижчий
    let (highest, lowest) = find_highest_lowest(&drones);
    println!("\nНайвищий: {} (висота {:.0} м)", names[highest], drones[highest].2);
    println!("Найнижчий: {} (висота {:.0} м)", names[lowest], drones[lowest].2);
    
    // Найближчий до бази
    let base: Position = (0.0, 0.0, 0.0);
    let nearest = find_nearest(&drones, base);
    let dist = distance_3d(drones[nearest], base);
    println!("\nНайближчий до бази: {} (відстань {:.1})", names[nearest], dist);
}
```

**Результат:**
```
=== Статистика позицій дронів ===

Позиції дронів:
  Alpha: (10, 20, 100)
  Beta: (50, 30, 150)
  Gamma: (30, 60, 80)
  Delta: (70, 40, 200)
  Epsilon: (20, 50, 120)

Центр мас рою: (36.0, 40.0, 130.0)

Найвищий: Delta (висота 200 м)
Найнижчий: Gamma (висота 80 м)

Найближчий до бази: Alpha (відстань 102.5)
```

---

## Підсумок заняття

На цьому занятті ви навчились:

1. **Створювати кортежі** для групування різнотипних даних
2. **Деструктуризувати кортежі** для зручного доступу до елементів
3. **Працювати з масивами** фіксованого розміру
4. **Використовувати слайси** для гнучкої роботи з послідовностями
5. **Ітерувати по масивах** з індексами та без
6. **Повертати кортежі з функцій** для множинних значень
7. **Розуміти різницю** між масивами та векторами

---

## Перевірте себе

1. Чим відрізняється `(42)` від `(42,)`?
2. Як отримати третій елемент кортежу `let t = (1, 2, 3, 4)`?
3. Що таке слайс і чим він відрізняється від масиву?
4. Як створити масив з 10 нулів?
5. Як передати частину масиву у функцію?
6. Чому функції краще приймати `&[T]` замість `[T; N]`?

**Відповіді:**
1. `(42)` — це число 42, `(42,)` — кортеж з одного елемента
2. `t.2` — індексація з 0
3. Слайс — це "вікно" на дані, не володіє ними; масив володіє даними
4. `let arr = [0; 10];`
5. `&array[start..end]` або `&array[..]` для всього
6. Слайс приймає масиви будь-якого розміру та вектори

---

## Наступне заняття

На наступному занятті ми вивчимо **Ownership та Borrowing** — ключові концепції Rust для безпечної роботи з пам'яттю. Ви зрозумієте, чому Rust не потребує збирача сміття та як компілятор запобігає помилкам пам'яті.
