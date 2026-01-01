# Розділ 48: Entity-Component-System (ECS)

---

## 📋 Анотація

До цього ви будували агентів як **об'єкти** — структури з полями та методами, що інкапсулюють стан і поведінку. Це класичний об'єктно-орієнтований підхід (OOP), який добре працює для невеликої кількості сутностей. Але що, якщо у вас 10,000 агентів? 100,000? Мільйон частинок у симуляції?

OOP стає вузьким місцем:
- **Продуктивність**: об'єкти розкидані по пам'яті, cache miss на кожному доступі
- **Гнучкість**: додавання нової поведінки потребує зміни класу
- **Паралелізм**: інкапсульований стан ускладнює безпечний паралелізм

**Entity-Component-System (ECS)** — архітектурний патерн, що радикально відрізняється від OOP. Замість "об'єктів з поведінкою" маємо чітке розділення: **дані окремо (Components), логіка окремо (Systems), ідентифікатори окремо (Entities)**. Це дає кращу cache locality, простішу паралелізацію та гнучкішу композицію.

ECS — стандарт у геймдеві (Unity DOTS, Unreal Mass, Bevy), але ідеально підходить і для симуляцій мультиагентних систем. Цей розділ пояснює філософію ECS та показує, як застосувати її до рою агентів.

---

## 🎯 Цілі навчання

Після завершення цього розділу ви зможете:

1. **Пояснити** філософію ECS та її відмінність від OOP
2. **Описати** три стовпи ECS: Entity, Component, System
3. **Розуміти** Archetypes та чому ECS швидкий
4. **Порівняти** ECS з Actor Model для МАС
5. **Використовувати** bevy_ecs для базових операцій
6. **Моделювати** рій агентів у ECS-стилі

---

## 📚 Ключові терміни

| Термін | Визначення |
|--------|------------|
| **Entity** | Унікальний ідентифікатор сутності (число, не об'єкт) |
| **Component** | Чисті дані без логіки (struct без методів) |
| **System** | Функція, що обробляє entities з певними компонентами |
| **Query** | Запит на entities, що мають вказані компоненти |
| **World** | Контейнер для всіх entities, компонентів та ресурсів |
| **Archetype** | Група entities з однаковим набором компонентів |
| **Resource** | Глобальні дані, не прив'язані до entity |
| **Event** | Повідомлення для комунікації між systems |

---

## 💡 Мотиваційний кейс: Симуляція хмари дронів

Уявімо симуляцію хмари з 100,000 мікро-дронів:

```rust
// OOP підхід — кожен дрон це об'єкт
struct Drone {
    id: u32,
    position: (f64, f64, f64),
    velocity: (f64, f64, f64),
    battery: f32,
    target: Option<(f64, f64, f64)>,
    mode: DroneMode,
    sensors: SensorData,
    // ... ще 20 полів
}

impl Drone {
    fn update(&mut self, dt: f32) {
        // Оновлення позиції
        // Перевірка батареї
        // Оновлення сенсорів
        // ... вся логіка в одному місці
    }
}

// 100,000 об'єктів
let drones: Vec<Drone> = create_drones(100_000);

// Оновлення — послідовно по об'єктах
for drone in &mut drones {
    drone.update(dt);  // Cache miss на кожному об'єкті!
}
```

**Проблеми OOP підходу:**

1. **Cache inefficiency**: Кожен `Drone` це великий блок пам'яті (100+ байт). При ітерації по `Vec<Drone>` CPU кеш постійно промахується.

2. **Все або нічого**: Метод `update` чіпає всі поля, навіть якщо конкретна логіка використовує лише деякі.

3. **Складна паралелізація**: `&mut self` на весь об'єкт — не можна паралельно оновлювати позицію та батарею.

**ECS підхід:**

```rust
// Компоненти — окремі маленькі структури
struct Position(f64, f64, f64);
struct Velocity(f64, f64, f64);
struct Battery(f32);

// Systems — окремі функції
fn movement_system(positions: &mut [Position], velocities: &[Velocity], dt: f32) {
    // Тільки Position та Velocity — cache-friendly!
    for (pos, vel) in positions.iter_mut().zip(velocities) {
        pos.0 += vel.0 * dt;
        pos.1 += vel.1 * dt;
        pos.2 += vel.2 * dt;
    }
}

fn battery_system(batteries: &mut [Battery], dt: f32) {
    // Тільки Battery — паралельно з movement!
    for bat in batteries {
        bat.0 -= 0.01 * dt;
    }
}
```

---

## 48.1 ФІЛОСОФІЯ ECS

### 48.1.1 OOP vs ECS: фундаментальна різниця

```text
┌─────────────────────────────────────────────────────────────────────┐
│                    OOP: Об'єкти з поведінкою                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │
│  │     Drone 1     │  │     Drone 2     │  │     Drone 3     │     │
│  ├─────────────────┤  ├─────────────────┤  ├─────────────────┤     │
│  │ position        │  │ position        │  │ position        │     │
│  │ velocity        │  │ velocity        │  │ velocity        │     │
│  │ battery         │  │ battery         │  │ battery         │     │
│  │ mode            │  │ mode            │  │ mode            │     │
│  ├─────────────────┤  ├─────────────────┤  ├─────────────────┤     │
│  │ fn update()     │  │ fn update()     │  │ fn update()     │     │
│  │ fn move()       │  │ fn move()       │  │ fn move()       │     │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘     │
│                                                                     │
│  • Дані + поведінка інкапсульовані разом                           │
│  • Об'єкти розкидані по пам'яті                                    │
│  • Зміна поведінки = зміна класу                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                    ECS: Дані окремо, логіка окремо                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ENTITIES (просто ID):     [1]  [2]  [3]  [4]  [5]  ...            │
│                                                                     │
│  COMPONENTS (таблиці даних):                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Position: [(x1,y1), (x2,y2), (x3,y3), (x4,y4), (x5,y5)...]  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Velocity: [(vx1,vy1), (vx2,vy2), (vx3,vy3), ...]            │   │
│  └─────────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Battery:  [100.0, 95.0, 87.5, 100.0, 42.0, ...]             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  SYSTEMS (функції над даними):                                     │
│  ┌──────────────────┐  ┌──────────────────┐                        │
│  │ MovementSystem   │  │ BatterySystem    │                        │
│  │ Query: Pos, Vel  │  │ Query: Battery   │                        │
│  └──────────────────┘  └──────────────────┘                        │
│                                                                     │
│  • Дані в послідовних масивах (cache-friendly)                     │
│  • Логіка окремо від даних                                         │
│  • Легко додати нові компоненти/systems                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 48.1.2 Три стовпи ECS

**Entity (Сутність)**
- Просто **унікальний ідентифікатор** (зазвичай число)
- Не містить даних
- Не містить логіки
- Це "ім'я" для набору компонентів

**Component (Компонент)**
- **Чисті дані** без логіки
- Маленька структура з одним призначенням
- Приклади: Position, Velocity, Health, Name
- Немає методів (окрім new, тривіальних getters)

**System (Система)**
- **Логіка**, що працює над компонентами
- Функція, а не метод
- Отримує Query: "дай мені всі entities з Position та Velocity"
- Трансформує дані

### 48.1.3 Аналогія з базою даних

ECS можна розуміти як **реляційну базу даних**:

| SQL концепт | ECS концепт |
|-------------|-------------|
| Row ID | Entity |
| Column | Component type |
| Cell value | Component instance |
| Table | Archetype |
| Query | System's Query |
| Stored Procedure | System |

```sql
-- SQL аналогія ECS Query
SELECT position, velocity 
FROM entities 
WHERE has_component(Position) AND has_component(Velocity);
```

---

## 48.2 COMPONENTS: ДАНІ БЕЗ ЛОГІКИ

### 48.2.1 Принцип Single Responsibility

Кожен компонент відповідає за **одну** річ:

```rust
// ✅ Добре: маленькі, спеціалізовані компоненти
struct Position { x: f32, y: f32 }
struct Velocity { dx: f32, dy: f32 }
struct Health { current: f32, max: f32 }
struct Name(String);
struct Battery { level: f32, drain_rate: f32 }

// ❌ Погано: "god component" з усім підряд
struct DroneData {
    x: f32, y: f32,
    dx: f32, dy: f32,
    health: f32,
    name: String,
    battery: f32,
    target_x: f32, target_y: f32,
    mode: u8,
    // ... ще 20 полів
}
```

**Чому маленькі компоненти краще?**

1. **Cache efficiency**: System, що працює тільки з Position, не тягне зайві дані в кеш
2. **Гнучкість**: Entity може мати Position без Velocity (статичний об'єкт)
3. **Паралелізм**: Різні systems можуть паралельно працювати з різними компонентами

### 48.2.2 Marker components

Іноді компонент не має даних — він просто **позначає** entity:

```rust
// Marker components — zero-sized types
struct Player;      // Позначає entity гравця
struct Enemy;       // Позначає ворога
struct Destroyed;   // Позначає для видалення

// Використання в Query:
// "Дай всі entities, що є Player І мають Position"
// Query<&Position, With<Player>>
```

### 48.2.3 Композиція замість ієрархії

В OOP: складна ієрархія класів

```text
GameObject
  └── Actor
        └── Character
              ├── Player
              └── NPC
                    └── Enemy
```

В ECS: плоска композиція компонентів

```rust
// Player = Position + Velocity + Health + PlayerInput + Sprite
// Enemy = Position + Velocity + Health + AIBehavior + Sprite
// Obstacle = Position + Collider + Sprite
// Collectible = Position + Sprite + OnPickup

// Легко створити нову сутність — просто комбінуй компоненти!
// FlyingEnemy = Position + Velocity + Health + AIBehavior + Flying + Sprite
```

---

## 48.3 ENTITIES: ПРОСТО ІДЕНТИФІКАТОРИ

### 48.3.1 Entity — це число

```rust
// Типове представлення Entity
struct Entity {
    id: u32,         // Унікальний ідентифікатор
    generation: u32, // Для recycling (див. нижче)
}

// Entity = 42
// Це НЕ об'єкт. Це номер рядка в таблиці.
```

### 48.3.2 Створення та знищення

**Постановка задачі:** Продемонструємо життєвий цикл entity.

```rust
// Створення — World видає новий ID
let entity = world.spawn();  // entity = Entity(1)

// Додавання компонентів
world.insert(entity, Position { x: 0.0, y: 0.0 });
world.insert(entity, Velocity { dx: 1.0, dy: 0.0 });

// Або разом при spawn
let entity = world.spawn((
    Position { x: 0.0, y: 0.0 },
    Velocity { dx: 1.0, dy: 0.0 },
    Battery { level: 100.0, drain_rate: 0.1 },
));

// Знищення
world.despawn(entity);
```

### 48.3.3 Generation для recycling

Коли entity знищується, її ID може бути перевикористаний. Але як відрізнити стару entity від нової?

```rust
// Entity з generation
struct Entity {
    id: u32,
    generation: u32,
}

// Entity(42, gen=1) знищено
// Новий spawn може отримати id=42, але gen=2
// Entity(42, gen=1) != Entity(42, gen=2)
```

Це запобігає "dangling references" — посилання на стару entity не працюватиме з новою.

---

## 48.4 SYSTEMS: ЛОГІКА НАД ДАНИМИ

### 48.4.1 System — це функція, не метод

```rust
// OOP: метод об'єкта
impl Drone {
    fn update(&mut self, dt: f32) {
        self.position.x += self.velocity.dx * dt;
        self.position.y += self.velocity.dy * dt;
    }
}

// ECS: функція над компонентами
fn movement_system(query: Query<(&mut Position, &Velocity)>, dt: f32) {
    for (mut pos, vel) in query.iter_mut() {
        pos.x += vel.dx * dt;
        pos.y += vel.dy * dt;
    }
}
```

### 48.4.2 Query — запит на компоненти

Query описує, які entities потрібні системі:

```rust
// Базовий Query: entities з Position та Velocity
Query<(&Position, &Velocity)>

// Мутабельний доступ
Query<(&mut Position, &Velocity)>  // Position можна змінювати

// З фільтрами
Query<&Position, With<Player>>     // Тільки з компонентом Player
Query<&Position, Without<Dead>>    // Без компонента Dead

// Optional компоненти
Query<(&Position, Option<&Target>)>  // Target може бути відсутнім
```

### 48.4.3 Автоматична паралелізація

Ключова перевага ECS — **автоматичний паралелізм**:

```rust
// Ці системи можуть виконуватись паралельно!
fn movement_system(query: Query<(&mut Position, &Velocity)>) { ... }
fn battery_system(query: Query<&mut Battery>) { ... }

// Чому? Вони працюють з РІЗНИМИ компонентами.
// Немає конфлікту даних → безпечний паралелізм.
```

ECS runtime автоматично визначає, які systems можуть бути паралельними:

```text
┌─────────────────────────────────────────────────────────────────────┐
│                    АВТОМАТИЧНИЙ ПАРАЛЕЛІЗМ                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  movement_system: Query<(&mut Position, &Velocity)>                 │
│  battery_system:  Query<&mut Battery>                               │
│  ai_system:       Query<(&mut AIState, &Position)>                  │
│                                                                     │
│  Аналіз конфліктів:                                                │
│  • movement читає Velocity, пише Position                          │
│  • battery пише Battery                                             │
│  • ai пише AIState, читає Position                                  │
│                                                                     │
│  Граф залежностей:                                                  │
│                                                                     │
│  movement ─────┬───── battery (паралельно!)                        │
│       │        │                                                    │
│       └────────┼───── ai (конфлікт по Position)                    │
│                │      ai чекає movement                             │
│                                                                     │
│  Результат: movement || battery, потім ai                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 48.4.4 System ordering

Іноді порядок важливий:

```rust
// Спочатку рух, потім перевірка колізій
app
    .add_system(movement_system)
    .add_system(collision_system.after(movement_system))
    .add_system(cleanup_system.after(collision_system));
```

---

## 48.5 ARCHETYPES: ОПТИМІЗАЦІЯ ПАМ'ЯТІ

### 48.5.1 Що таке Archetype

**Archetype** — група entities з **однаковим набором компонентів**.

```text
Entity 1: [Position, Velocity, Battery]    ─┐
Entity 2: [Position, Velocity, Battery]     ├── Archetype A
Entity 3: [Position, Velocity, Battery]    ─┘

Entity 4: [Position, Velocity]             ─┐
Entity 5: [Position, Velocity]              ├── Archetype B
Entity 6: [Position, Velocity]             ─┘

Entity 7: [Position, Health]               ─── Archetype C
```

### 48.5.2 Чому це швидко

```text
┌─────────────────────────────────────────────────────────────────────┐
│                    ARCHETYPE STORAGE                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Archetype A: [Position, Velocity, Battery]                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Entities: [1, 2, 3]                                        │   │
│  │                                                             │   │
│  │  Position[]:  [(x1,y1), (x2,y2), (x3,y3)]  ← Послідовно!   │   │
│  │  Velocity[]:  [(vx1,vy1), (vx2,vy2), (vx3,vy3)]            │   │
│  │  Battery[]:   [100.0, 95.0, 87.5]                          │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Query<(&mut Position, &Velocity)> для Archetype A:                │
│                                                                     │
│  for i in 0..3 {                                                   │
│      position[i].x += velocity[i].dx;  // Cache-friendly!          │
│      position[i].y += velocity[i].dy;                              │
│  }                                                                 │
│                                                                     │
│  Дані в пам'яті послідовно → CPU prefetcher працює ефективно       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Порівняння з OOP:**

| Аспект | OOP (Vec<Drone>) | ECS (Archetypes) |
|--------|------------------|------------------|
| Розташування | Об'єкти розкидані | Дані послідовно |
| Cache hits | Низький % | Високий % |
| Prefetching | Неефективне | Дуже ефективне |
| Ітерація 100K | ~10ms | ~1ms |

### 48.5.3 Archetype changes

Додавання або видалення компонента = **зміна archetype**:

```text
Entity 5 в Archetype A: [Position, Velocity]
          │
          │ insert(Battery)
          ▼
Entity 5 в Archetype B: [Position, Velocity, Battery]
```

Це операція "переїзду" — дані копіюються з одного archetype в інший. Тому **часте додавання/видалення компонентів — дорога операція**.

**Практичні наслідки:**
- Не додавайте/видаляйте компоненти кожен кадр
- Використовуйте Option<T> всередині компонента замість видалення
- Або marker components, що рідко змінюються

---

## 48.6 ECS VS ACTOR MODEL

### 48.6.1 Порівняльна таблиця

| Аспект | Actor Model | ECS |
|--------|-------------|-----|
| **Одиниця** | Актор (об'єкт з mailbox) | Entity (ID) + Components |
| **Стан** | Приватний, інкапсульований | Відкритий, у таблицях |
| **Логіка** | Методи актора | Глобальні Systems |
| **Комунікація** | Повідомлення (async) | Через спільні дані / Events |
| **Паралелізм** | Актори паралельні | Systems паралельні |
| **Масштабування** | Тисячі-десятки тисяч | Мільйони entities |
| **Когнітивна модель** | "Об'єкти спілкуються" | "Дані трансформуються" |
| **Memory layout** | Об'єкти розкидані | Data-oriented, cache-friendly |

### 48.6.2 Коли що обирати

**Actor Model краще для:**
- Distributed systems (актори на різних машинах)
- Мережеві агенти з складною комунікацією
- Системи, де важлива інкапсуляція стану
- Гетерогенні агенти з різною логікою

**ECS краще для:**
- Симуляції з величезною кількістю entities
- Ігрові сцени (rendering, physics)
- Коли критична продуктивність
- Гомогенні сутності з подібною поведінкою

### 48.6.3 Гібридний підхід

Часто найкращий варіант — комбінувати:

```text
┌─────────────────────────────────────────────────────────────────────┐
│                    ГІБРИДНА АРХІТЕКТУРА                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │               ECS WORLD (симуляція)                           │ │
│  │                                                               │ │
│  │   100,000 entities з Position, Velocity, Battery              │ │
│  │   MovementSystem, BatterySystem, CollisionSystem              │ │
│  │   Надшвидка обробка фізики та базової логіки                  │ │
│  │                                                               │ │
│  └─────────────────────────────┬─────────────────────────────────┘ │
│                                │                                    │
│                                │ Events                             │
│                                ▼                                    │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │               ACTOR SYSTEM (координація)                      │ │
│  │                                                               │ │
│  │   Coordinator Actor: приймає рішення на рівні рою            │ │
│  │   Network Actor: комунікація з зовнішнім світом              │ │
│  │   AI Director Actor: high-level стратегія                    │ │
│  │                                                               │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  ECS для "фізики", Actors для "інтелекту"                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 48.7 SCHEDULING ТА СТАДІЇ ВИКОНАННЯ

### 48.7.1 Концепція Scheduling

У реальній симуляції systems мають виконуватись у певному порядку. **Scheduler** визначає:
- Порядок виконання
- Які systems можуть бути паралельними
- Стадії оновлення

### 48.7.2 Стадії (Stages)

```text
┌─────────────────────────────────────────────────────────────────────┐
│                    ТИПОВІ СТАДІЇ FRAME                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. PRE_UPDATE                                                      │
│     - input_system: читання команд                                  │
│     - spawn_system: створення нових entities                       │
│                                                                     │
│  2. UPDATE                                                          │
│     - ai_system: прийняття рішень                                   │
│     - movement_system: оновлення позицій                            │
│     - collision_system: перевірка зіткнень                          │
│                                                                     │
│  3. POST_UPDATE                                                     │
│     - damage_system: обробка пошкоджень                             │
│     - death_system: перевірка "смерті"                              │
│                                                                     │
│  4. CLEANUP                                                         │
│     - despawn_system: видалення "мертвих"                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 48.7.3 Явне впорядкування

```rust
// bevy_ecs синтаксис
app
    .add_systems(Update, (
        ai_system,
        movement_system.after(ai_system),
        collision_system.after(movement_system),
    ))
    .add_systems(PostUpdate, (
        despawn_dead_system,
    ));
```

---

## 48.8 ECS ФРЕЙМВОРКИ В RUST

### 48.8.1 Огляд екосистеми

| Бібліотека | Опис | Особливості |
|------------|------|-------------|
| **bevy_ecs** | ECS з Bevy engine | Найпопулярніший, ergonomic |
| **hecs** | Мінімалістичний | Швидкий, простий API |
| **legion** | Потужний, гнучкий | Складніший, більше features |
| **specs** | Класичний | Зрілий, стабільний |

### 48.8.2 Чому bevy_ecs

Ми використовуватимемо **bevy_ecs** з кількох причин:

1. **Найактивніша розробка** — Bevy швидко розвивається
2. **Ergonomic API** — зручний для навчання
3. **Автоматична паралелізація** — з коробки
4. **Можна використовувати окремо** — без решти Bevy
5. **Хороша документація** — багато прикладів

### 48.8.3 Обмеження bevy_ecs

Важливо знати обмеження:

- **Archetype fragmentation**: багато різних комбінацій компонентів = багато маленьких archetypes = менша ефективність
- **Compile times**: макроси derive можуть сповільнювати компіляцію
- **Learning curve**: ECS мислення відрізняється від OOP

---

## 48.9 BEVY_ECS: ПРАКТИКА

### 48.9.1 Налаштування проєкту

```toml
[dependencies]
bevy_ecs = "0.12"
```

### 48.9.2 Визначення компонентів

**Постановка задачі:** Створимо базові компоненти для агента БПЛА.

```rust
use bevy_ecs::prelude::*;

/// Позиція в 2D просторі
#[derive(Component, Debug)]
struct Position {
    x: f32,
    y: f32,
}

/// Вектор швидкості
#[derive(Component, Debug)]
struct Velocity {
    dx: f32,
    dy: f32,
}

/// Рівень заряду батареї
#[derive(Component, Debug)]
struct Battery {
    level: f32,
    drain_rate: f32,
}

/// Marker: це агент (а не перешкода чи ціль)
#[derive(Component)]
struct Agent;

/// Marker: агент активний
#[derive(Component)]
struct Active;
```

**Розглянемо ключові моменти:**

1. **`#[derive(Component)]`** — макрос bevy_ecs, що робить struct компонентом
2. **Маленькі структури** — кожен компонент відповідає за одне
3. **Marker components** — `Agent`, `Active` не мають полів

### 48.9.3 Створення World та entities

**Постановка задачі:** Створимо світ та кілька агентів.

```rust
fn main() {
    // World — контейнер для всього
    let mut world = World::new();
    
    // Spawn entity з компонентами
    let agent1 = world.spawn((
        Agent,                              // Marker
        Active,                             // Активний
        Position { x: 0.0, y: 0.0 },
        Velocity { dx: 1.0, dy: 0.5 },
        Battery { level: 100.0, drain_rate: 0.1 },
    )).id();
    
    let agent2 = world.spawn((
        Agent,
        Active,
        Position { x: 10.0, y: 10.0 },
        Velocity { dx: -0.5, dy: 1.0 },
        Battery { level: 80.0, drain_rate: 0.15 },
    )).id();
    
    // Ціль (без Velocity — статична)
    let target = world.spawn((
        Position { x: 50.0, y: 50.0 },
    )).id();
    
    println!("Created agents: {:?}, {:?}", agent1, agent2);
    println!("Created target: {:?}", target);
}
```

**Розглянемо як працює spawn:**

1. **`world.spawn()`** приймає tuple компонентів
2. **Повертає EntityWorldMut** — builder для entity
3. **`.id()`** повертає Entity (ID)
4. Entity автоматично потрапляє в правильний archetype

### 48.9.4 Написання Systems

**Постановка задачі:** Створимо системи для руху та витрачання батареї.

```rust
/// System руху: оновлює Position на основі Velocity
fn movement_system(mut query: Query<(&mut Position, &Velocity)>) {
    for (mut pos, vel) in query.iter_mut() {
        pos.x += vel.dx;
        pos.y += vel.dy;
    }
}

/// System витрачання батареї
fn battery_drain_system(mut query: Query<&mut Battery, With<Active>>) {
    for mut battery in query.iter_mut() {
        battery.level -= battery.drain_rate;
        if battery.level < 0.0 {
            battery.level = 0.0;
        }
    }
}

/// System логування позицій агентів
fn debug_system(query: Query<(Entity, &Position, &Battery), With<Agent>>) {
    for (entity, pos, bat) in query.iter() {
        println!(
            "Agent {:?}: pos=({:.1}, {:.1}), battery={:.1}%",
            entity, pos.x, pos.y, bat.level
        );
    }
}
```

**Розглянемо Query синтаксис:**

1. **`Query<(&mut Position, &Velocity)>`** — entities з обома компонентами
2. **`&mut Position`** — мутабельний доступ до Position
3. **`With<Active>`** — фільтр: тільки з компонентом Active
4. **`Entity`** в tuple — отримати ID entity

### 48.9.5 Запуск симуляції

**Постановка задачі:** Об'єднаємо все в працюючу симуляцію.

```rust
use bevy_ecs::prelude::*;
use bevy_ecs::schedule::Schedule;

fn main() {
    // Створюємо World
    let mut world = World::new();
    
    // Створюємо агентів
    for i in 0..5 {
        world.spawn((
            Agent,
            Active,
            Position { x: i as f32 * 10.0, y: 0.0 },
            Velocity { dx: 1.0, dy: 0.5 },
            Battery { level: 100.0, drain_rate: 0.5 },
        ));
    }
    
    // Створюємо Schedule (набір систем)
    let mut schedule = Schedule::default();
    schedule.add_systems((
        movement_system,
        battery_drain_system,
        debug_system,
    ));
    
    // Симуляція: 10 кроків
    for step in 0..10 {
        println!("\n=== Step {} ===", step);
        schedule.run(&mut world);
    }
}
```

---

## 48.10 РІЙ АГЕНТІВ В ECS

### 48.10.1 Розширені компоненти

**Постановка задачі:** Створимо повний набір компонентів для рою БПЛА.

```rust
use bevy_ecs::prelude::*;

/// Позиція в 2D
#[derive(Component, Clone, Copy, Debug)]
struct Position { x: f32, y: f32 }

/// Швидкість
#[derive(Component, Clone, Copy, Debug)]
struct Velocity { dx: f32, dy: f32 }

/// Батарея
#[derive(Component, Debug)]
struct Battery { level: f32 }

/// Поточна ціль руху
#[derive(Component, Debug)]
struct Target { x: f32, y: f32 }

/// Режим агента
#[derive(Component, Debug, PartialEq)]
enum AgentMode {
    Idle,
    Patrolling,
    Returning,
    Charging,
}

/// Marker: це агент-дрон
#[derive(Component)]
struct Drone;

/// База для повернення
#[derive(Component)]
struct HomeBase { x: f32, y: f32 }
```

### 48.10.2 Systems для рою

**Постановка задачі:** Реалізуємо логіку поведінки рою.

```rust
/// System: рух до цілі
fn seek_target_system(
    mut query: Query<(&mut Position, &mut Velocity, &Target), With<Drone>>
) {
    let speed = 2.0;
    
    for (mut pos, mut vel, target) in query.iter_mut() {
        let dx = target.x - pos.x;
        let dy = target.y - pos.y;
        let distance = (dx * dx + dy * dy).sqrt();
        
        if distance > 1.0 {
            vel.dx = (dx / distance) * speed;
            vel.dy = (dy / distance) * speed;
        } else {
            vel.dx = 0.0;
            vel.dy = 0.0;
        }
    }
}

/// System: перевірка батареї
fn battery_check_system(
    mut query: Query<(&Battery, &mut AgentMode), With<Drone>>
) {
    for (battery, mut mode) in query.iter_mut() {
        if battery.level < 20.0 && *mode != AgentMode::Returning {
            *mode = AgentMode::Returning;
            println!("Low battery! Returning to base.");
        }
    }
}

/// System: повернення на базу
fn return_to_base_system(
    mut query: Query<(&HomeBase, &mut Target, &AgentMode), With<Drone>>
) {
    for (home, mut target, mode) in query.iter_mut() {
        if *mode == AgentMode::Returning {
            target.x = home.x;
            target.y = home.y;
        }
    }
}

/// System: застосування швидкості
fn apply_velocity_system(
    mut query: Query<(&mut Position, &Velocity)>
) {
    for (mut pos, vel) in query.iter_mut() {
        pos.x += vel.dx;
        pos.y += vel.dy;
    }
}

/// System: витрата батареї при русі
fn battery_drain_system(
    mut query: Query<(&mut Battery, &Velocity), With<Drone>>
) {
    for (mut battery, vel) in query.iter_mut() {
        let movement = (vel.dx * vel.dx + vel.dy * vel.dy).sqrt();
        battery.level -= movement * 0.1;  // Витрата пропорційна швидкості
    }
}
```

### 48.10.3 Архітектура рою

```text
┌─────────────────────────────────────────────────────────────────────┐
│                    ECS РІЙ БПЛА                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  COMPONENTS:                                                        │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐              │
│  │ Position │ │ Velocity │ │ Battery  │ │  Target  │              │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                           │
│  │AgentMode │ │  Drone   │ │ HomeBase │                           │
│  └──────────┘ └──────────┘ └──────────┘                           │
│                                                                     │
│  SYSTEMS (порядок виконання):                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 1. battery_check_system     (перевірка заряду)              │   │
│  │ 2. return_to_base_system    (якщо низький заряд)            │   │
│  │ 3. seek_target_system       (обчислення напрямку)           │   │
│  │ 4. apply_velocity_system    (оновлення позиції)             │   │
│  │ 5. battery_drain_system     (витрата енергії)               │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ENTITIES:                                                         │
│  Drone 1: [Position, Velocity, Battery, Target, AgentMode, ...]   │
│  Drone 2: [Position, Velocity, Battery, Target, AgentMode, ...]   │
│  ...                                                               │
│  Drone N: [Position, Velocity, Battery, Target, AgentMode, ...]   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 48.11 EVENTS ТА RESOURCES

### 48.11.1 Events — комунікація між systems

Events дозволяють systems комунікувати без прямих залежностей:

```rust
/// Event: ціль виявлено
#[derive(Event)]
struct TargetDiscovered {
    discoverer: Entity,
    target_position: Position,
}

/// System: виявлення цілей
fn discovery_system(
    query: Query<(Entity, &Position), With<Drone>>,
    targets: Query<&Position, With<DiscoverableTarget>>,
    mut events: EventWriter<TargetDiscovered>,
) {
    for (drone_entity, drone_pos) in query.iter() {
        for target_pos in targets.iter() {
            let distance = /* обчислення */;
            if distance < 10.0 {
                events.send(TargetDiscovered {
                    discoverer: drone_entity,
                    target_position: *target_pos,
                });
            }
        }
    }
}

/// System: реакція на виявлення
fn on_discovery_system(
    mut events: EventReader<TargetDiscovered>,
) {
    for event in events.read() {
        println!("Drone {:?} discovered target at ({}, {})", 
                 event.discoverer, 
                 event.target_position.x, 
                 event.target_position.y);
    }
}
```

### 48.11.2 Resources — глобальний стан

Resources — дані, не прив'язані до entity:

```rust
/// Час симуляції
#[derive(Resource)]
struct SimulationTime {
    elapsed: f32,
    delta: f32,
}

/// Глобальна карта
#[derive(Resource)]
struct WorldMap {
    width: f32,
    height: f32,
    obstacles: Vec<Position>,
}

/// System з доступом до ресурсу
fn time_aware_system(
    time: Res<SimulationTime>,
    mut query: Query<&mut Position>,
) {
    for mut pos in query.iter_mut() {
        pos.x += 1.0 * time.delta;  // Використовуємо delta time
    }
}
```

---

## 48.12 ЛАБОРАТОРНА РОБОТА

### Мета роботи

Опанувати ECS архітектуру для моделювання рою агентів.

### Завдання 1: Базові компоненти (3 бали)

Створіть компоненти для простого агента:
- Position, Velocity, Health
- Marker компоненти: Player, Enemy

Створіть 10 entities з різними комбінаціями.

### Завдання 2: Movement System (4 бали)

Реалізуйте:
- movement_system: застосування velocity
- bounds_check_system: агенти не виходять за межі карти

### Завдання 3: Рій з патрулюванням (3 бали)

Створіть рій з 5 агентів, що:
- Рухаються до випадкових точок
- Змінюють ціль при досягненні
- Витрачають батарею

---

## 48.13 РЕЗЮМЕ

### Ключові концепції

**ECS** — архітектурний патерн: Entity (ID) + Component (дані) + System (логіка).

**Data-oriented design** — дані в центрі, логіка працює над даними.

**Archetypes** — групи entities з однаковими компонентами; cache-friendly.

**Автоматичний паралелізм** — systems без конфліктів виконуються паралельно.

### Коли використовувати ECS

| Сценарій | ECS? |
|----------|------|
| Симуляція 100,000+ entities | ✅ Ідеально |
| Гомогенні сутності | ✅ Добре |
| Критична продуктивність | ✅ Добре |
| Distributed система | ⚠️ Actor Model краще |
| Складна комунікація | ⚠️ Actor Model краще |
| Маленький проєкт | ❌ Overkill |

---

## 🔗 ЗВ'ЯЗОК З НАСТУПНИМ РОЗДІЛОМ

ECS чудово підходить для **реактивних** агентів — тих, що реагують на середовище. Але для **когнітивних** агентів, що планують і міркують, потрібна інша модель.

**BDI (Beliefs-Desires-Intentions)** — класична архітектура для раціональних агентів з AI. Агент має переконання про світ, бажання (цілі), та наміри (плани).

У **Розділі 49: BDI-агенти — Beliefs-Desires-Intentions** ви:
- Познайомитесь з BDI моделлю раціональних агентів
- Зрозумієте цикл deliberation (обмірковування)
- Реалізуєте BDI-агента на Rust
- Порівняєте BDI з реактивними агентами

---

> **Наступний розділ:** [Розділ 49: BDI-агенти — Beliefs-Desires-Intentions](./49_BDI_Agents.md)
