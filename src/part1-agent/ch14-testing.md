# Розділ 14: Модульне тестування

## Вступ

"Код працює!" — радісно каже розробник після ручного тестування. Через тиждень він змінює одну функцію, і система ламається в п'яти різних місцях. Ще через місяць новий член команди боїться торкатися коду — ніхто не знає, що зламається від найменшої зміни.

Автоматичні тести — це страхова сітка для вашого коду. Вони документують очікувану поведінку, миттєво виявляють регресії (баги, що з'явилися після змін), і дають впевненість при рефакторингу. Замість ручної перевірки "працює чи ні" ви запускаєте одну команду і отримуєте відповідь за секунди.

Rust має вбудовану підтримку тестування: атрибут `#[test]`, команда `cargo test`, уся необхідна інфраструктура. Не потрібно встановлювати зовнішні бібліотеки чи налаштовувати конфігурацію — все готово з коробки.

Цей розділ особливо важливий перед переходом до паралелізму та асинхронності. Коли код виконується в кількох потоках одночасно, традиційний дебаг з println! стає практично неможливим. Тести — це ваші очі в темряві конкурентного виконання.

---

## 14.1 Чому тести — це не марнування часу

Розглянемо реальний сценарій. Ось код агента БПЛА:

```rust
impl Agent {
    pub fn can_execute_mission(&self) -> bool {
        self.battery > 20 && self.is_active
    }
}
```

Функція перевіряє, чи може агент виконати місію: батарея має бути вище 20% І агент має бути активний. Код простий, очевидний, працює.

Через місяць хтось "оптимізує" код або випадково видаляє частину умови:

```rust
pub fn can_execute_mission(&self) -> bool {
    self.battery > 20  // Забув && self.is_active
}
```

Без тестів цей баг непомітний на етапі компіляції. Код скомпілюється успішно. Можливо, баг виявиться через тиждень у продакшені, коли неактивний агент спробує злетіти.

Але якщо у вас є тест:

```rust
#[test]
fn inactive_agent_cannot_execute_mission() {
    let agent = Agent {
        battery: 100,      // Повна батарея
        is_active: false,  // Але неактивний!
    };
    
    // Неактивний агент НЕ МОЖЕ виконувати місію
    assert!(!agent.can_execute_mission());
}
```

Цей тест впаде миттєво після помилкової "оптимізації". Розробник побачить:

```
test inactive_agent_cannot_execute_mission ... FAILED

assertion failed: !agent.can_execute_mission()
```

Баг знайдено за секунди, а не за тижні. Ніхто не постраждав.

---

## 14.2 Анатомія тесту в Rust

### Найпростіший тест

Тест у Rust — це звичайна функція з атрибутом `#[test]`:

```rust
// Функція, яку ми тестуємо
fn add(a: i32, b: i32) -> i32 {
    a + b
}

// Тестова функція
#[test]
fn test_add() {
    let result = add(2, 3);
    assert_eq!(result, 5);
}
```

Щоб запустити тести, виконайте команду:

```bash
cargo test
```

Ви побачите вивід:

```
running 1 test
test test_add ... ok

test result: ok. 1 passed; 0 failed; 0 ignored
```

Що тут відбувається? Атрибут `#[test]` каже компілятору: "це не звичайна функція, це тест". При запуску `cargo test` Rust компілює спеціальну версію програми, знаходить усі функції з `#[test]`, і виконує їх по черзі.

### Коли тест успішний

Тест вважається успішним, якщо функція завершується без паніки. Паніка — це аварійне завершення, яке виникає, наприклад, при невиконанні `assert!`:

```rust
#[test]
fn passing_test() {
    let x = 2 + 2;
    assert_eq!(x, 4);  // 4 == 4 — true — тест пройшов
}

#[test]
fn failing_test() {
    let x = 2 + 2;
    assert_eq!(x, 5);  // 4 != 5 — ПАНІКА — тест провалено
}
```

Коли `assert_eq!(4, 5)` виконується, Rust бачить, що значення не рівні, і викликає `panic!`. Тестовий раннер ловить цю паніку і позначає тест як провалений.

### Тест без assert теж може пройти

Тест без жодного assert пройде успішно, бо функція просто завершиться:

```rust
#[test]
fn empty_test() {
    // Нічого не перевіряємо — тест пройде
}
```

Такий тест безглуздий, але легальний. Корисно знати: успіх тесту = відсутність паніки.

---

## 14.3 Макроси assertions

Rust надає кілька макросів для перевірки умов у тестах.

### assert! — перевірка булевого виразу

Найпростіший макрос. Перевіряє, що вираз повертає `true`:

```rust
#[test]
fn test_boolean_conditions() {
    let battery = 85;
    let is_active = true;
    
    // Прості перевірки
    assert!(battery > 0);
    assert!(battery <= 100);
    assert!(is_active);
    assert!(!false);  // !false == true
    
    // З кастомним повідомленням про помилку
    assert!(battery >= 20, "Батарея занадто низька: {}%", battery);
}
```

Якщо умова `false`, тест падає з повідомленням. Другий аргумент — опціональне кастомне повідомлення, яке буде виведено при падінні.

### assert_eq! та assert_ne! — порівняння значень

Коли потрібно порівняти два значення, `assert_eq!` зручніший за `assert!(a == b)`, бо показує обидва значення при помилці:

```rust
#[test]
fn test_equality() {
    let expected = 42;
    let actual = 40 + 2;
    
    assert_eq!(actual, expected);  // actual == expected
    assert_ne!(actual, 0);         // actual != 0
    
    // З кастомним повідомленням
    assert_eq!(actual, expected, 
               "Розрахунок невірний: отримано {}, очікувалось {}", 
               actual, expected);
}
```

Коли `assert_eq!` падає, ви бачите чітке повідомлення:

```
thread 'test_equality' panicked at 'assertion failed: `(left == right)`
  left: `42`,
 right: `43`'
```

Це набагато інформативніше, ніж просто "assertion failed".

### Порівняння складних типів

Щоб `assert_eq!` працював з вашими структурами, вони мають реалізовувати два traits: `PartialEq` (для порівняння) та `Debug` (для виводу при помилці). Найпростіше — використати derive:

```rust
#[derive(Debug, PartialEq)]  // Обов'язково для assert_eq!
struct Position {
    x: f64,
    y: f64,
}

#[test]
fn test_position_equality() {
    let p1 = Position { x: 1.0, y: 2.0 };
    let p2 = Position { x: 1.0, y: 2.0 };
    let p3 = Position { x: 3.0, y: 4.0 };
    
    assert_eq!(p1, p2);  // Однакові координати — рівні
    assert_ne!(p1, p3);  // Різні координати — не рівні
}
```

### Обережно з плаваючою точкою

Числа з плаваючою точкою мають обмежену точність. Класична проблема:

```rust
#[test]
fn test_float_danger() {
    let result = 0.1 + 0.2;
    
    // ❌ Цей тест може провалитись!
    // assert_eq!(result, 0.3);
    
    // Насправді result ≈ 0.30000000000000004
    println!("0.1 + 0.2 = {:.20}", result);
}
```

Правильний спосіб — порівнювати з допустимою похибкою (epsilon):

```rust
fn approx_eq(a: f64, b: f64, epsilon: f64) -> bool {
    (a - b).abs() < epsilon
}

#[test]
fn test_float_correct() {
    let result = 0.1 + 0.2;
    let epsilon = 1e-10;
    
    assert!(approx_eq(result, 0.3, epsilon));
    // Або напряму:
    assert!((result - 0.3).abs() < epsilon);
}
```

---

## 14.4 Тестування паніки

Іноді функція ПОВИННА панікувати за певних умов. Наприклад, ділення на нуль:

```rust
fn divide(a: i32, b: i32) -> i32 {
    if b == 0 {
        panic!("Ділення на нуль!");
    }
    a / b
}
```

Як протестувати, що функція справді панікує? Атрибут `#[should_panic]`:

```rust
#[test]
#[should_panic]
fn test_divide_by_zero_panics() {
    divide(10, 0);  // Має викликати panic!
}
```

Тест пройде, якщо функція запанікує. Якщо функція завершиться нормально — тест провалиться.

### Перевірка повідомлення паніки

Можна переконатись, що паніка містить конкретний текст:

```rust
#[test]
#[should_panic(expected = "Ділення на нуль")]
fn test_panic_message() {
    divide(10, 0);  // Паніка має містити "Ділення на нуль"
}
```

Якщо повідомлення паніки не містить "Ділення на нуль", тест провалиться. Це корисно, коли функція може панікувати з різних причин — ви перевіряєте саме очікувану.

### Альтернатива: Result у тестах

Замість паніки тест може повертати `Result`:

```rust
#[test]
fn test_with_result() -> Result<(), String> {
    let result = some_operation();
    
    if result != 42 {
        return Err(format!("Очікувалось 42, отримано {}", result));
    }
    
    Ok(())
}
```

Тест успішний, якщо повертає `Ok(())`. Якщо повертає `Err`, тест провалено з відповідним повідомленням.

---

## 14.5 Організація тестів у модулі

### Стандартний паттерн: модуль tests

Домовленість у Rust — тримати тести в модулі `tests` внизу того ж файлу:

```rust
// src/battery.rs

pub struct Battery {
    level: u8,
}

impl Battery {
    pub fn new() -> Self {
        Self { level: 100 }
    }
    
    pub fn drain(&mut self, amount: u8) {
        self.level = self.level.saturating_sub(amount);
    }
    
    pub fn level(&self) -> u8 {
        self.level
    }
    
    pub fn is_critical(&self) -> bool {
        self.level < 20
    }
}

// ═══════════════════════════════════════════════════════════════
// ТЕСТИ
// ═══════════════════════════════════════════════════════════════

#[cfg(test)]
mod tests {
    use super::*;  // Імпортуємо все з батьківського модуля
    
    #[test]
    fn new_battery_is_full() {
        let battery = Battery::new();
        assert_eq!(battery.level(), 100);
    }
    
    #[test]
    fn drain_reduces_level() {
        let mut battery = Battery::new();
        battery.drain(30);
        assert_eq!(battery.level(), 70);
    }
    
    #[test]
    fn drain_does_not_go_negative() {
        let mut battery = Battery::new();
        battery.drain(150);  // Більше ніж є
        assert_eq!(battery.level(), 0);
    }
    
    #[test]
    fn critical_below_20() {
        let mut battery = Battery::new();
        
        battery.drain(80);  // 20%
        assert!(!battery.is_critical());  // 20% — ще не критично
        
        battery.drain(1);  // 19%
        assert!(battery.is_critical());  // 19% — критично
    }
}
```

### Навіщо #[cfg(test)]

Атрибут `#[cfg(test)]` каже компілятору: "цей код тільки для тестів". Без нього тестовий модуль потрапив би у фінальний бінарник, збільшуючи його розмір без потреби.

```rust
#[cfg(test)]  // Компілюється ТІЛЬКИ при cargo test
mod tests {
    // Тести тут
}
```

При звичайному `cargo build` цей модуль повністю ігнорується.

### use super::*

Рядок `use super::*` імпортує все з батьківського модуля (того, де оголошено `mod tests`). Це дає доступ до всіх функцій і типів, які ми хочемо тестувати.

Важливий нюанс: `use super::*` імпортує і приватні елементи! Тести в тому ж файлі можуть тестувати приватні функції:

```rust
mod my_module {
    pub fn public_fn() -> i32 { 42 }
    fn private_fn() -> i32 { 13 }  // Приватна!
    
    #[cfg(test)]
    mod tests {
        use super::*;
        
        #[test]
        fn test_public() {
            assert_eq!(public_fn(), 42);
        }
        
        #[test]
        fn test_private() {
            // Можемо тестувати приватні функції!
            assert_eq!(private_fn(), 13);
        }
    }
}
```

---

## 14.6 Чи тестувати приватні функції?

Це дискусійне питання в спільноті розробників. Є два протилежних підходи.

### Підхід 1: Тестувати тільки публічне API

Аргументи:
- Приватні функції — деталі реалізації
- Якщо публічний API працює правильно, приватні функції теж працюють
- Можна вільно рефакторити приватний код, не ламаючи тести

### Підхід 2: Тестувати і приватне

Аргументи:
- Легше локалізувати джерело багу
- Важлива бізнес-логіка може бути в приватних функціях
- Тести документують очікувану поведінку для майбутніх розробників

### Практична рекомендація

Якщо приватна функція містить складну логіку, яку важко протестувати через публічний API — тестуйте її напряму. Якщо вона тривіальна — достатньо тестів публічного API.

Приклад складної приватної функції, яку варто тестувати:

```rust
// Приватна функція з нетривіальною логікою
fn calculate_checksum(data: &[u8]) -> u32 {
    data.iter()
        .enumerate()
        .map(|(i, &b)| (b as u32) * (i as u32 + 1))
        .sum()
}

// Публічна функція
pub fn validate_message(data: &[u8], expected: u32) -> bool {
    calculate_checksum(data) == expected
}

#[cfg(test)]
mod tests {
    use super::*;
    
    // Тестуємо приватну функцію напряму — легше дебажити
    #[test]
    fn checksum_empty_data() {
        assert_eq!(calculate_checksum(&[]), 0);
    }
    
    #[test]
    fn checksum_known_values() {
        // [1] -> 1*1 = 1
        // [1, 2] -> 1*1 + 2*2 = 5
        // [1, 2, 3] -> 1*1 + 2*2 + 3*3 = 14
        assert_eq!(calculate_checksum(&[1]), 1);
        assert_eq!(calculate_checksum(&[1, 2]), 5);
        assert_eq!(calculate_checksum(&[1, 2, 3]), 14);
    }
    
    // Тестуємо публічну функцію теж
    #[test]
    fn validation_works() {
        let data = vec![1, 2, 3];
        assert!(validate_message(&data, 14));
        assert!(!validate_message(&data, 15));
    }
}
```

---

## 14.7 Запуск і фільтрація тестів

### Базові команди

```bash
# Запустити всі тести
cargo test

# Показувати вивід println! (за замовчуванням приховується)
cargo test -- --nocapture

# Запустити конкретний тест за повною назвою
cargo test test_battery_drain

# Запустити тести, що містять "battery" в назві
cargo test battery

# Запустити в одному потоці (для дебагу race conditions)
cargo test -- --test-threads=1

# Показати список усіх тестів без запуску
cargo test -- --list
```

### Фільтрація за назвою

Якщо у вас тести:
- `test_battery_drain`
- `test_battery_charge`
- `test_position_distance`

Тоді:
```bash
cargo test battery      # запустить test_battery_drain і test_battery_charge
cargo test drain        # запустить тільки test_battery_drain
cargo test position     # запустить тільки test_position_distance
```

### Ігнорування повільних тестів

Деякі тести можуть бути повільними (наприклад, інтеграційні з мережею). Позначте їх `#[ignore]`:

```rust
#[test]
#[ignore]  // Не запускається за замовчуванням
fn slow_integration_test() {
    // Повільний тест
}

#[test]
#[ignore = "потребує підключення до мережі"]
fn network_test() {
    // Тест з мережею
}
```

Команди:
```bash
cargo test              # Запустити всі, КРІМ ignored
cargo test -- --ignored # Запустити ТІЛЬКИ ignored
cargo test -- --include-ignored  # Запустити ВСІ
```

---

## 14.8 Паттерн Arrange-Act-Assert

Класичний паттерн організації тесту — AAA (Arrange-Act-Assert):

```rust
#[test]
fn test_agent_completes_mission() {
    // ═══════════════════════════════════════════
    // ARRANGE — підготовка тестових даних
    // ═══════════════════════════════════════════
    let mut agent = Agent::new("TEST-001");
    let target = Position::new(10.0, 0.0, 0.0);
    agent.start_mission(target).unwrap();
    
    // ═══════════════════════════════════════════
    // ACT — виконання дії, яку тестуємо
    // ═══════════════════════════════════════════
    for _ in 0..20 {
        agent.update();
    }
    
    // ═══════════════════════════════════════════
    // ASSERT — перевірка результату
    // ═══════════════════════════════════════════
    assert_eq!(agent.state(), &AgentState::Idle);
    assert_eq!(agent.position(), &target);
}
```

Чітке розділення на три частини робить тест читабельним: що налаштовано, що виконано, що перевірено.

---

## 14.9 Тести граничних значень

Баги часто ховаються на межах: нуль, максимум, "рівно на порозі". Тестуйте ці випадки явно:

```rust
#[cfg(test)]
mod boundary_tests {
    use super::*;
    
    #[test]
    fn battery_at_zero() {
        let mut battery = Battery::new();
        battery.drain(100);
        
        assert_eq!(battery.level(), 0);
        assert!(battery.is_critical());
    }
    
    #[test]
    fn battery_cannot_go_negative() {
        let mut battery = Battery::new();
        battery.drain(150);  // Витрачаємо більше, ніж є
        
        assert_eq!(battery.level(), 0);  // Не -50!
    }
    
    #[test]
    fn battery_at_critical_threshold() {
        let mut battery = Battery::new();
        battery.drain(80);  // Рівно 20%
        
        // 20% — це критично чи ні?
        // Тест документує точну поведінку
        assert!(!battery.is_critical());  // За нашою логікою: < 20, не <=
    }
    
    #[test]
    fn battery_just_below_critical() {
        let mut battery = Battery::new();
        battery.drain(81);  // 19%
        
        assert!(battery.is_critical());
    }
}
```

---

## 14.10 Повний приклад: тестування агента БПЛА

Зберемо все разом і напишемо повний набір тестів для агента.

### Тестування Position

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
pub struct Position {
    pub x: f64,
    pub y: f64,
    pub z: f64,
}

impl Position {
    pub fn new(x: f64, y: f64, z: f64) -> Self {
        Self { x, y, z }
    }
    
    pub fn origin() -> Self {
        Self::new(0.0, 0.0, 0.0)
    }
    
    pub fn distance_to(&self, other: &Position) -> f64 {
        let dx = self.x - other.x;
        let dy = self.y - other.y;
        let dz = self.z - other.z;
        (dx*dx + dy*dy + dz*dz).sqrt()
    }
}

#[cfg(test)]
mod position_tests {
    use super::*;
    
    fn approx_eq(a: f64, b: f64) -> bool {
        (a - b).abs() < 1e-9
    }
    
    #[test]
    fn new_creates_position_with_coordinates() {
        let pos = Position::new(1.0, 2.0, 3.0);
        
        assert!(approx_eq(pos.x, 1.0));
        assert!(approx_eq(pos.y, 2.0));
        assert!(approx_eq(pos.z, 3.0));
    }
    
    #[test]
    fn origin_is_at_zero() {
        let pos = Position::origin();
        
        assert_eq!(pos, Position::new(0.0, 0.0, 0.0));
    }
    
    #[test]
    fn distance_to_self_is_zero() {
        let pos = Position::new(5.0, 5.0, 5.0);
        
        assert!(approx_eq(pos.distance_to(&pos), 0.0));
    }
    
    #[test]
    fn distance_to_known_points() {
        let p1 = Position::origin();
        let p2 = Position::new(3.0, 4.0, 0.0);
        
        // Трикутник 3-4-5: відстань = 5
        assert!(approx_eq(p1.distance_to(&p2), 5.0));
    }
    
    #[test]
    fn distance_is_symmetric() {
        let p1 = Position::new(1.0, 2.0, 3.0);
        let p2 = Position::new(4.0, 5.0, 6.0);
        
        // d(p1, p2) == d(p2, p1)
        assert!(approx_eq(p1.distance_to(&p2), p2.distance_to(&p1)));
    }
}
```

### Тестування AgentState

```rust
#[derive(Debug, Clone, PartialEq)]
pub enum AgentState {
    Idle,
    Moving { target: Position },
    Scanning { progress: u8 },
    Charging,
    Emergency(String),
}

impl AgentState {
    pub fn is_active(&self) -> bool {
        matches!(self, AgentState::Moving { .. } | AgentState::Scanning { .. })
    }
    
    pub fn can_receive_commands(&self) -> bool {
        !matches!(self, AgentState::Emergency(_))
    }
}

#[cfg(test)]
mod state_tests {
    use super::*;
    
    #[test]
    fn idle_is_not_active() {
        assert!(!AgentState::Idle.is_active());
    }
    
    #[test]
    fn moving_is_active() {
        let state = AgentState::Moving { 
            target: Position::origin() 
        };
        assert!(state.is_active());
    }
    
    #[test]
    fn scanning_is_active() {
        let state = AgentState::Scanning { progress: 50 };
        assert!(state.is_active());
    }
    
    #[test]
    fn charging_is_not_active() {
        assert!(!AgentState::Charging.is_active());
    }
    
    #[test]
    fn normal_states_accept_commands() {
        assert!(AgentState::Idle.can_receive_commands());
        assert!(AgentState::Charging.can_receive_commands());
        assert!(AgentState::Moving { target: Position::origin() }.can_receive_commands());
    }
    
    #[test]
    fn emergency_blocks_commands() {
        let state = AgentState::Emergency("Тест".to_string());
        assert!(!state.can_receive_commands());
    }
}
```

### Тестування Agent

```rust
pub struct Agent {
    id: String,
    position: Position,
    battery: u8,
    state: AgentState,
}

impl Agent {
    pub fn new(id: &str) -> Self {
        Self {
            id: id.to_string(),
            position: Position::origin(),
            battery: 100,
            state: AgentState::Idle,
        }
    }
    
    pub fn battery(&self) -> u8 {
        self.battery
    }
    
    pub fn state(&self) -> &AgentState {
        &self.state
    }
    
    pub fn is_operational(&self) -> bool {
        self.battery > 20 && self.state.can_receive_commands()
    }
    
    pub fn drain_battery(&mut self, amount: u8) {
        self.battery = self.battery.saturating_sub(amount);
    }
    
    pub fn start_mission(&mut self, target: Position) -> Result<(), String> {
        if !self.is_operational() {
            return Err("Агент не операційний".to_string());
        }
        
        self.state = AgentState::Moving { target };
        Ok(())
    }
}

#[cfg(test)]
mod agent_tests {
    use super::*;
    
    // ═══════════════════════════════════════════
    // Тести створення
    // ═══════════════════════════════════════════
    
    #[test]
    fn new_agent_has_full_battery() {
        let agent = Agent::new("TEST-001");
        assert_eq!(agent.battery(), 100);
    }
    
    #[test]
    fn new_agent_is_idle() {
        let agent = Agent::new("TEST-001");
        assert_eq!(agent.state(), &AgentState::Idle);
    }
    
    #[test]
    fn new_agent_is_operational() {
        let agent = Agent::new("TEST-001");
        assert!(agent.is_operational());
    }
    
    // ═══════════════════════════════════════════
    // Тести батареї
    // ═══════════════════════════════════════════
    
    #[test]
    fn drain_reduces_battery() {
        let mut agent = Agent::new("TEST-001");
        
        agent.drain_battery(30);
        
        assert_eq!(agent.battery(), 70);
    }
    
    #[test]
    fn drain_does_not_go_negative() {
        let mut agent = Agent::new("TEST-001");
        
        agent.drain_battery(150);
        
        assert_eq!(agent.battery(), 0);
    }
    
    #[test]
    fn low_battery_makes_non_operational() {
        let mut agent = Agent::new("TEST-001");
        
        agent.drain_battery(85);  // Залишилось 15%
        
        assert!(!agent.is_operational());
    }
    
    #[test]
    fn boundary_21_percent_is_operational() {
        let mut agent = Agent::new("TEST-001");
        
        agent.drain_battery(79);  // Залишилось 21%
        
        assert!(agent.is_operational());
    }
    
    #[test]
    fn boundary_20_percent_is_not_operational() {
        let mut agent = Agent::new("TEST-001");
        
        agent.drain_battery(80);  // Залишилось 20%
        
        // Умова: battery > 20, тому 20 — НЕ операційний
        assert!(!agent.is_operational());
    }
    
    // ═══════════════════════════════════════════
    // Тести місії
    // ═══════════════════════════════════════════
    
    #[test]
    fn operational_agent_starts_mission() {
        let mut agent = Agent::new("TEST-001");
        let target = Position::new(100.0, 50.0, 30.0);
        
        let result = agent.start_mission(target);
        
        assert!(result.is_ok());
        assert_eq!(agent.state(), &AgentState::Moving { target });
    }
    
    #[test]
    fn low_battery_agent_rejects_mission() {
        let mut agent = Agent::new("TEST-001");
        agent.drain_battery(90);  // 10%
        
        let result = agent.start_mission(Position::origin());
        
        assert!(result.is_err());
        assert_eq!(agent.state(), &AgentState::Idle);  // Стан не змінився
    }
    
    #[test]
    fn emergency_agent_rejects_mission() {
        let mut agent = Agent::new("TEST-001");
        agent.state = AgentState::Emergency("Тест".to_string());
        
        let result = agent.start_mission(Position::origin());
        
        assert!(result.is_err());
    }
}
```

---

## 14.11 Лабораторна робота

Створіть повний набір тестів для системи керування температурою.

**Завдання:**

1. Реалізуйте структуру `Thermostat`:
   ```rust
   pub struct Thermostat {
       current_temp: f64,
       target_temp: f64,
       is_heating: bool,
       is_cooling: bool,
   }
   ```

2. Додайте методи:
   - `new(initial_temp: f64)` — конструктор
   - `set_target(&mut self, temp: f64)` — встановлення цільової температури
   - `update(&mut self)` — оновлення стану (вмикає нагрів/охолодження)
   - `status(&self) -> &str` — повертає "heating", "cooling", або "idle"

3. Напишіть тести:
   - Тест конструктора
   - Тести граничних значень температури
   - Тести логіки нагріву/охолодження
   - Тест переходів між станами

**Критерії оцінювання:**

| Критерій | Бали |
|----------|------|
| Структура Thermostat | 2 |
| Методи | 3 |
| Тести конструктора | 1 |
| Тести граничних значень | 2 |
| Тести логіки переходів | 2 |
| **Максимум** | **10** |

---

## Підсумок

Тестування — це не додаткова робота, а інвестиція в майбутнє. Тести:

- **Документують** очікувану поведінку краще за коментарі
- **Захищають** від регресій при змінах коду
- **Прискорюють** розробку через швидкий зворотній зв'язок
- **Дозволяють** рефакторинг без страху все зламати

У Rust тестування вбудовано в мову. `#[test]` позначає тестову функцію, `#[cfg(test)]` ізолює тестовий код, `cargo test` запускає все однією командою.

Пишіть тести для граничних значень. Використовуйте паттерн Arrange-Act-Assert. Тестуйте і успішні сценарії, і помилки. І найголовніше — пишіть тести ДО того, як баг потрапить у продакшн.

---

## Зв'язок з наступним розділом

Тепер у вас є всі інструменти для створення надійного коду: структури, enum'и, модулі, тести. У **Розділі 15: Практикум — Автономний агент v1.0** ви об'єднаєте все в цілісний проєкт: повноцінного автономного агента з машиною станів, модульною архітектурою та повним покриттям тестами.
