# Розділ 49: BDI-агенти — Beliefs-Desires-Intentions

---

## 📋 Анотація

До цього моменту наші агенти були переважно **реактивними** — вони реагували на стимули без глибокого "розуміння" ситуації. Побачив ціль — рухайся до неї. Низький заряд — повертайся на базу. Перешкода — обійди. Це працює для простих задач, але як бути зі складними місіями?

Уявімо: БПЛА має одночасно розвідати територію, доставити вантаж, уникати ворожого ППО, підтримувати зв'язок з базою — і все це з обмеженим зарядом батареї. Реактивні правила швидко стають хаотичними: що важливіше — доставка чи безпека? Коли переключитися з розвідки на повернення?

**BDI (Beliefs-Desires-Intentions)** — когнітивна архітектура, що моделює **раціонального агента**. Це не просто набір if-else правил, а формальна модель того, як агент **знає** (Beliefs), чого **хоче** (Desires), і що **планує робити** (Intentions). Ця модель походить з філософії розуму та є однією з найвпливовіших архітектур інтелектуальних агентів.

---

## 🎯 Цілі навчання

Після завершення цього розділу ви зможете:

1. **Пояснити** філософські та теоретичні основи BDI-моделі
2. **Описати** цикл deliberation та means-ends reasoning
3. **Порівняти** реактивних, гібридних та когнітивних агентів
4. **Реалізувати** базову BDI-архітектуру на Rust
5. **Застосувати** BDI для складних місій агентів БПЛА

---

## 📚 Ключові терміни

| Термін | Визначення |
|--------|------------|
| **Beliefs (переконання)** | Модель світу агента — що він "знає" про середовище |
| **Desires (бажання)** | Цілі, яких агент хотів би досягти |
| **Intentions (наміри)** | Обрані цілі, до яких агент committed |
| **Deliberation** | Процес вибору намірів з множини бажань |
| **Means-ends reasoning** | Пошук плану для досягнення цілі |
| **Plan library** | Бібліотека готових шаблонів планів |
| **Practical reasoning** | Міркування про дії (що робити), на відміну від theoretical reasoning (що істинне) |
| **Commitment** | Прихильність до наміру — небажання його змінювати |

---

## 💡 Мотиваційний кейс: Складна розвідувальна місія

Уявімо місію рою БПЛА:

**Одночасні цілі:**
1. Розвідка сектору А (пріоритет: високий)
2. Доставка медичного вантажу в точку Б (пріоритет: критичний)
3. Уникнення виявленої зони ППО
4. Підтримка стійкого зв'язку з базою
5. Збереження резерву батареї для повернення

**Динамічні обставини:**
- Раптово з'являється загроза в секторі А
- Погода погіршується — зростає витрата батареї
- База повідомляє про нову високопріоритетну ціль
- Один з дронів рою виходить з ладу

Реактивний агент з простими правилами не справиться — правила будуть конфліктувати, а поведінка стане непередбачуваною. Потрібен агент, що **розмірковує**: оцінює ситуацію, пріоритезує цілі, будує плани, адаптується до змін.

---

## 49.1 ФІЛОСОФСЬКІ ОСНОВИ BDI

### 49.1.1 Теорія практичного міркування Bratman

BDI-модель не була вигадана computer scientists "з нуля". Вона базується на **філософії розуму**, зокрема на роботах американського філософа **Michael Bratman** (книга "Intention, Plans, and Practical Reason", 1987).

Bratman досліджував фундаментальне питання: **як люди приймають рішення про дії?**

Він виділив два типи міркування:

**Теоретичне міркування (Theoretical reasoning)**
- Про що **істинне**
- "Чи йде зараз дощ?"
- Результат: переконання (beliefs)

**Практичне міркування (Practical reasoning)**
- Про що **робити**
- "Чи брати парасольку?"
- Результат: наміри (intentions)

### 49.1.2 Роль намірів у теорії Bratman

Ключове відкриття Bratman — **наміри** це не просто "сильні бажання". Наміри мають особливі властивості:

**1. Наміри обмежують майбутнє мислення**

Коли ви прийняли намір "їхати до Парижа на вихідних", ви перестаєте розглядати альтернативи "їхати до Лондона", "залишитися вдома". Це **економить когнітивні ресурси** — не потрібно щомиті переоцінювати всі варіанти.

**2. Наміри керують плануванням**

Намір стає "якорем" для подальшого планування:
- Намір: "їхати до Парижа"
- → Підціль: "купити квитки"
- → → Дія: "відкрити сайт бронювання"

**3. Наміри мають інерцію (commitment)**

Люди не змінюють наміри легковажно. Якщо ви вирішили їхати до Парижа, ви не передумаєте через дрібницю ("о, в Лондоні цікава виставка"). Потрібна **вагома причина** для перегляду.

**4. Наміри соціально значущі**

Інші можуть покладатися на ваші наміри. Якщо ви сказали "я буду о 18:00", люди планують відповідно.

### 49.1.3 Від філософії до штучного інтелекту

У 1991 році **Anand Rao** та **Michael Georgeff** адаптували ідеї Bratman для **штучних агентів**. Вони запропонували формальну модель BDI-агента:

```text
┌─────────────────────────────────────────────────────────────────────┐
│                    BDI АРХІТЕКТУРА (Rao & Georgeff)                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│                    СЕРЕДОВИЩЕ                                       │
│                         │                                           │
│                         │ perception (сприйняття)                   │
│                         ▼                                           │
│              ┌─────────────────────┐                               │
│              │      BELIEFS        │                               │
│              │  (переконання про   │                               │
│              │   стан світу)       │                               │
│              └──────────┬──────────┘                               │
│                         │                                           │
│                         │ inform (інформують)                       │
│                         ▼                                           │
│              ┌─────────────────────┐                               │
│              │      DESIRES        │                               │
│              │  (цілі, яких хоче   │                               │
│              │   досягти)          │                               │
│              └──────────┬──────────┘                               │
│                         │                                           │
│                         │ deliberation (обмірковування)             │
│                         ▼                                           │
│              ┌─────────────────────┐                               │
│              │    INTENTIONS       │                               │
│              │  (обрані цілі +     │                               │
│              │   плани дій)        │                               │
│              └──────────┬──────────┘                               │
│                         │                                           │
│                         │ execution (виконання)                     │
│                         ▼                                           │
│                    СЕРЕДОВИЩЕ                                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 49.1.4 Чому BDI, а не просто правила?

Логічне питання: навіщо така складна модель? Чому не просто if-else правила?

**Підхід на основі правил:**

```text
IF obstacle_ahead THEN avoid
IF target_visible THEN approach  
IF battery < 20% THEN return_to_base
IF threat_detected THEN evade
```

**Проблеми:**
1. Що робити, коли кілька правил активні одночасно?
2. Хто визначає пріоритет?
3. Як планувати послідовність дій?
4. Як пояснити, чому агент так вчинив?

**BDI-підхід:**
- Явно моделює **цілі** з пріоритетами (Desires)
- **Deliberation** — формальний процес вибору
- **Plans** — структуровані послідовності дій
- **Пояснюваність** — можна відповісти "чому" (belief X → desire Y → intention Z)

---

## 49.2 ДЕТАЛЬНА АРХІТЕКТУРА BDI

### 49.2.1 Beliefs: модель світу агента

**Beliefs** — це те, що агент "вважає істинним" про світ. Важливо: beliefs можуть бути **неповними** або **помилковими**.

```text
┌─────────────────────────────────────────────────────────────────────┐
│                    СТРУКТУРА BELIEFS                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  BELIEFS ПРО СЕБЕ:                                                  │
│  • Моя позиція: (45.2, 78.1)                                       │
│  • Мій заряд батареї: 67%                                          │
│  • Моя швидкість: 12 м/с                                           │
│  • Мій поточний режим: Патрулювання                                │
│                                                                     │
│  BELIEFS ПРО СЕРЕДОВИЩЕ:                                            │
│  • Відомі цілі: [(100, 50), (200, 150)]                            │
│  • Відомі загрози: [(80, 90, радіус=50)]                           │
│  • Погодні умови: Вітер 5 м/с, північний                           │
│  • Час доби: 14:35                                                 │
│                                                                     │
│  BELIEFS ПРО ІНШИХ АГЕНТІВ:                                         │
│  • Союзник #2 в позиції (30, 40)                                   │
│  • Союзник #3 повертається на базу                                 │
│  • База в позиції (0, 0)                                           │
│                                                                     │
│  МЕТАДАНІ:                                                          │
│  • Остання синхронізація: 2 секунди тому                           │
│  • Достовірність даних: Висока                                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Оновлення beliefs:**
- Через **сенсори** (perception)
- Через **комунікацію** з іншими агентами
- Через **логічний висновок** (якщо A і A→B, то B)

### 49.2.2 Desires: цілі агента

**Desires** — це стани світу, яких агент хотів би досягти. Desires можуть бути:

**Maintenance goals (підтримувальні):**
- "Підтримувати заряд > 30%"
- "Залишатися в зоні зв'язку"
- "Уникати загроз"

**Achievement goals (досягальні):**
- "Досягти точки (100, 50)"
- "Розвідати сектор А"
- "Доставити вантаж"

**Avoidance goals (уникання):**
- "Не заходити в зону ППО"
- "Не зіткнутися з перешкодами"

**Властивості desires:**
- Можуть **конфліктувати** ("досягти цілі" vs "зберегти батарею")
- Мають **пріоритети**
- Деякі **постійні**, деякі **тимчасові**

### 49.2.3 Intentions: committed цілі з планами

**Intentions** — це desires, до яких агент **committed**. Intention включає:
- Саму ціль
- **План** досягнення
- **Поточний крок** плану

```text
┌─────────────────────────────────────────────────────────────────────┐
│                    ПРИКЛАД INTENTION                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  INTENTION: Розвідати сектор А                                     │
│                                                                     │
│  ПЛАН:                                                              │
│  1. Піднятися на висоту 100м                     [✓ виконано]      │
│  2. Рухатися до точки входу (50, 0)              [✓ виконано]      │
│  3. Виконати патерн "змійка" по сектору          [→ поточний]      │
│  4. Повернутися до вихідної позиції              [ ]               │
│  5. Надіслати звіт на базу                       [ ]               │
│                                                                     │
│  УМОВИ ПЕРЕГЛЯДУ:                                                   │
│  • Якщо виявлено загрозу → перервати, евакуюватися                 │
│  • Якщо батарея < 30% → перервати, повернутися                     │
│  • Якщо нова високопріоритетна ціль → переглянути                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 49.2.4 Цикл BDI-агента

```text
┌─────────────────────────────────────────────────────────────────────┐
│                    ЦИКЛ BDI-АГЕНТА                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  while running {                                                    │
│                                                                     │
│      ┌─────────────────────────────────────────────────────────┐   │
│      │ 1. PERCEPTION (Сприйняття)                              │   │
│      │    • Отримати дані з сенсорів                           │   │
│      │    • Отримати повідомлення від інших агентів            │   │
│      └─────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│      ┌─────────────────────────────────────────────────────────┐   │
│      │ 2. BELIEF UPDATE (Оновлення переконань)                 │   │
│      │    • Інтегрувати нову інформацію в beliefs              │   │
│      │    • Видалити застарілі beliefs                         │   │
│      │    • Зробити логічні висновки                           │   │
│      └─────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│      ┌─────────────────────────────────────────────────────────┐   │
│      │ 3. OPTION GENERATION (Генерація опцій)                  │   │
│      │    • Які desires актуальні за поточних beliefs?         │   │
│      │    • Чи з'явилися нові можливості?                      │   │
│      └─────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│      ┌─────────────────────────────────────────────────────────┐   │
│      │ 4. DELIBERATION (Обмірковування)                        │   │
│      │    • Чи потрібно переглянути intentions?                │   │
│      │    • Якщо так: вибрати нові intentions з desires        │   │
│      └─────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│      ┌─────────────────────────────────────────────────────────┐   │
│      │ 5. MEANS-ENDS REASONING (Планування)                    │   │
│      │    • Для нових intentions: знайти план                  │   │
│      │    • Для існуючих: перевірити валідність плану          │   │
│      └─────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│      ┌─────────────────────────────────────────────────────────┐   │
│      │ 6. EXECUTION (Виконання)                                │   │
│      │    • Виконати наступний крок плану                      │   │
│      │    • Надіслати команди актуаторам                       │   │
│      └─────────────────────────────────────────────────────────┘   │
│                                                                     │
│  }                                                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 49.2.5 Deliberation: коли переглядати intentions?

Ключове питання: **як часто** агент має переглядати свої наміри?

**Занадто часто (Bold agent):**
- Постійно перемикається між цілями
- Ніколи не завершує почате
- Неефективний

**Занадто рідко (Cautious agent):**
- Продовжує неактуальні плани
- Не реагує на критичні зміни
- Негнучкий

**Збалансований підхід — перегляд при:**
1. **Завершенні** поточного intention
2. **Критичних змінах** beliefs (загроза, низький заряд)
3. **Неможливості** продовжити поточний план
4. **Появі** значно кращої опції

```text
┌─────────────────────────────────────────────────────────────────────┐
│                    ТРИГЕРИ DELIBERATION                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ЗАВЖДИ ПЕРЕГЛЯДАТИ:                                               │
│  • Поточне intention досягнуто                                     │
│  • Поточне intention стало неможливим                              │
│  • Виявлено критичну загрозу                                       │
│  • Батарея досягла критичного рівня                                │
│                                                                     │
│  МОЖЛИВО ПЕРЕГЛЯДАТИ:                                              │
│  • З'явилася нова ціль з вищим пріоритетом                         │
│  • Значно змінилися умови середовища                               │
│  • Отримано нову інформацію від бази                               │
│                                                                     │
│  НЕ ПЕРЕГЛЯДАТИ:                                                   │
│  • Дрібні зміни в beliefs                                          │
│  • Поточний план просувається нормально                            │
│  • Немає критичних тригерів                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 49.2.6 Means-Ends Reasoning: пошук планів

Коли агент обрав intention, йому потрібен **план** — послідовність дій для досягнення цілі.

**Підхід 1: Plan Library (бібліотека планів)**

Заздалегідь визначені шаблони планів:

```rust
ПЛАН: РозвідатиСектор(сектор)
  ПЕРЕДУМОВА: battery > 40%, немає_загроз_в(сектор)
  КРОКИ:
    1. РухатисьДо(сектор.вхідна_точка)
    2. ВиконатиПатерн("змійка", сектор.межі)
    3. ПовернутисьДо(вихідна_позиція)
    4. НадіслатиЗвіт()
```

**Підхід 2: Planning (динамічне планування)**

Генерація плану "з нуля" через алгоритми планування (A*, STRIPS, HTN).

**Порівняння:**

| Plan Library | Planning |
|--------------|----------|
| Швидкий вибір | Повільна генерація |
| Обмежена гнучкість | Висока гнучкість |
| Передбачувана поведінка | Може здивувати |
| Для типових ситуацій | Для нових ситуацій |

---

## 49.3 РЕАКТИВНІ VS BDI АГЕНТИ

### 49.3.1 Спектр архітектур агентів

```text
┌─────────────────────────────────────────────────────────────────────┐
│                    СПЕКТР АРХІТЕКТУР АГЕНТІВ                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Простота                                              Складність   │
│  Швидкість                                             Гнучкість    │
│  ◄────────────────────────────────────────────────────────────────► │
│                                                                     │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐   │
│  │ Reflex  │  │Reactive │  │ Hybrid  │  │Delibera-│  │  Full   │   │
│  │  Agent  │  │  Agent  │  │  Agent  │  │  tive   │  │  BDI    │   │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘  └─────────┘   │
│                                                                     │
│  IF-THEN      Стан +      Реактивний   Планування   Beliefs +      │
│  правила      правила     + плановий   + виконання  Desires +      │
│                           шар                       Intentions     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 49.3.2 Порівняльна таблиця

| Аспект | Реактивний агент | BDI агент |
|--------|------------------|-----------|
| **Модель світу** | Немає або мінімальна | Повна (Beliefs) |
| **Цілі** | Неявні (в правилах) | Явні (Desires) |
| **Планування** | Немає | Так (Intentions) |
| **Час реакції** | Мілісекунди | Десятки мілісекунд |
| **Пам'ять** | Мінімальна | Значна |
| **Пояснюваність** | Низька | Висока |
| **Адаптивність** | До передбачених ситуацій | До нових ситуацій |
| **Складність реалізації** | Низька | Висока |

### 49.3.3 Subsumption Architecture — альтернатива

**Rodney Brooks** (MIT, 1986) запропонував радикально інший підхід — **Subsumption Architecture**:

```text
┌─────────────────────────────────────────────────────────────────────┐
│                    SUBSUMPTION ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Рівень 3: Дослідження      ─────────────┐                         │
│  (explore new areas)                      │ inhibit                 │
│                                           ▼                         │
│  Рівень 2: Досягнення цілі  ─────────────┐                         │
│  (move to target)                         │ inhibit                 │
│                                           ▼                         │
│  Рівень 1: Уникнення        ─────────────┐                         │
│  (avoid obstacles)                        │ suppress                │
│                                           ▼                         │
│  Рівень 0: Базовий рух      ────────────────► АКТУАТОРИ            │
│  (basic locomotion)                                                │
│                                                                     │
│  Вищі рівні можуть ПРИГНІЧУВАТИ нижчі                              │
│  Кожен рівень працює паралельно                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Ідея:** Замість центрального "мозку" — паралельні поведінкові модулі, що конкурують за контроль.

**Порівняння з BDI:**

| Subsumption | BDI |
|-------------|-----|
| Паралельні поведінки | Послідовні плани |
| Немає явних цілей | Явні Desires |
| Emergent behavior | Designed behavior |
| Швидший | Повільніший |
| Менш передбачуваний | Більш передбачуваний |

### 49.3.4 Коли використовувати який підхід?

**Реактивний агент краще, коли:**
- Середовище **дуже швидкозмінне** (реакція < 10ms)
- Задача **проста** (avoid, seek, wander)
- **Критична** швидкість відповіді
- Обмежені обчислювальні ресурси
- Поведінка добре описується правилами

**BDI агент краще, коли:**
- Потрібне **планування** (послідовність дій)
- Є **кілька конкуруючих цілей**
- Потрібна **пояснюваність** рішень
- Середовище **помірно динамічне**
- Задача **складна** (місії, координація)
- Важлива **адаптивність** до нових ситуацій

---

## 49.4 РЕАЛІЗАЦІЯ BDI НА RUST

### 49.4.1 Структура Beliefs

**Постановка задачі:** Створимо структуру для зберігання переконань агента БПЛА.

```rust
use std::collections::{HashMap, HashSet};
use std::time::Instant;

/// Позиція в 2D просторі
#[derive(Clone, Copy, Debug, PartialEq)]
pub struct Position {
    pub x: f64,
    pub y: f64,
}

/// Виявлена ціль
#[derive(Clone, Debug)]
pub struct Target {
    pub id: u32,
    pub position: Position,
    pub priority: u8,
    pub discovered_at: Instant,
}

/// Виявлена загроза
#[derive(Clone, Debug)]
pub struct Threat {
    pub position: Position,
    pub radius: f64,
    pub threat_level: u8,
}

/// База переконань агента
pub struct Beliefs {
    // Переконання про себе
    pub my_position: Position,
    pub my_battery: u8,
    pub my_heading: f64,
    pub my_speed: f64,
    
    // Переконання про середовище
    pub known_targets: Vec<Target>,
    pub known_threats: Vec<Threat>,
    pub explored_sectors: HashSet<u32>,
    pub weather_conditions: String,
    
    // Переконання про інших агентів
    pub ally_positions: HashMap<u32, Position>,
    pub base_position: Position,
    
    // Метадані
    pub last_perception: Instant,
}

impl Beliefs {
    /// Створює початкові beliefs
    pub fn new(start_position: Position, base: Position) -> Self {
        Self {
            my_position: start_position,
            my_battery: 100,
            my_heading: 0.0,
            my_speed: 0.0,
            known_targets: Vec::new(),
            known_threats: Vec::new(),
            explored_sectors: HashSet::new(),
            weather_conditions: "clear".to_string(),
            ally_positions: HashMap::new(),
            base_position: base,
            last_perception: Instant::now(),
        }
    }
    
    /// Оновлює позицію
    pub fn update_position(&mut self, pos: Position) {
        self.my_position = pos;
        self.last_perception = Instant::now();
    }
    
    /// Додає виявлену ціль
    pub fn add_target(&mut self, target: Target) {
        // Перевіряємо, чи не дублікат
        if !self.known_targets.iter().any(|t| t.id == target.id) {
            self.known_targets.push(target);
        }
    }
    
    /// Перевіряє, чи beliefs застаріли
    pub fn is_stale(&self, max_age_secs: u64) -> bool {
        self.last_perception.elapsed().as_secs() > max_age_secs
    }
}
```

**Розглянемо ключові аспекти цієї структури:**

1. **Групування beliefs** — окремо про себе, середовище, інших агентів
2. **Метадані** — `last_perception` для відстеження свіжості
3. **Методи оновлення** — `update_position`, `add_target`
4. **Перевірка застарілості** — `is_stale`

### 49.4.2 Структура Desires

**Постановка задачі:** Реалізуємо цілі агента з пріоритетами та типами.

```rust
/// Тип бажання
#[derive(Clone, Debug, PartialEq)]
pub enum DesireType {
    /// Досягти точки
    ReachPosition(Position),
    /// Розвідати сектор
    ExploreSector(u32),
    /// Повернутися на базу
    ReturnToBase,
    /// Уникати загрози
    AvoidThreat(Position, f64), // позиція, радіус
    /// Підтримувати заряд
    MaintainBattery(u8), // мінімальний рівень
}

/// Бажання агента
#[derive(Clone, Debug)]
pub struct Desire {
    pub id: u32,
    pub desire_type: DesireType,
    pub priority: u8,           // 0-255, вище = важливіше
    pub is_maintenance: bool,   // true = постійна ціль
    pub deadline: Option<Instant>,
}

/// Множина бажань агента
pub struct Desires {
    desires: Vec<Desire>,
    next_id: u32,
}

impl Desires {
    pub fn new() -> Self {
        Self {
            desires: Vec::new(),
            next_id: 1,
        }
    }
    
    /// Додає нове бажання
    pub fn add(&mut self, desire_type: DesireType, priority: u8, is_maintenance: bool) -> u32 {
        let id = self.next_id;
        self.next_id += 1;
        
        self.desires.push(Desire {
            id,
            desire_type,
            priority,
            is_maintenance,
            deadline: None,
        });
        
        id
    }
    
    /// Повертає бажання за пріоритетом (спочатку найважливіші)
    pub fn by_priority(&self) -> Vec<&Desire> {
        let mut sorted: Vec<_> = self.desires.iter().collect();
        sorted.sort_by(|a, b| b.priority.cmp(&a.priority));
        sorted
    }
    
    /// Видаляє бажання за id
    pub fn remove(&mut self, id: u32) {
        self.desires.retain(|d| d.id != id);
    }
    
    /// Перевіряє, чи є активні бажання
    pub fn has_active(&self) -> bool {
        !self.desires.is_empty()
    }
}
```

### 49.4.3 Структура Intentions

**Постановка задачі:** Реалізуємо наміри з планами виконання.

```rust
/// Крок плану
#[derive(Clone, Debug)]
pub enum PlanStep {
    MoveTo(Position),
    Hover(f64),           // тривалість в секундах
    Scan,
    SendReport,
    Wait(f64),
}

/// Статус виконання кроку
#[derive(Clone, Debug, PartialEq)]
pub enum StepStatus {
    Pending,
    InProgress,
    Completed,
    Failed(String),
}

/// План досягнення цілі
#[derive(Clone, Debug)]
pub struct Plan {
    pub steps: Vec<PlanStep>,
    pub current_step: usize,
    pub step_statuses: Vec<StepStatus>,
}

impl Plan {
    pub fn new(steps: Vec<PlanStep>) -> Self {
        let len = steps.len();
        Self {
            steps,
            current_step: 0,
            step_statuses: vec![StepStatus::Pending; len],
        }
    }
    
    /// Повертає поточний крок
    pub fn current(&self) -> Option<&PlanStep> {
        self.steps.get(self.current_step)
    }
    
    /// Позначає поточний крок завершеним і переходить до наступного
    pub fn advance(&mut self) -> bool {
        if self.current_step < self.steps.len() {
            self.step_statuses[self.current_step] = StepStatus::Completed;
            self.current_step += 1;
            true
        } else {
            false
        }
    }
    
    /// Перевіряє, чи план завершено
    pub fn is_complete(&self) -> bool {
        self.current_step >= self.steps.len()
    }
}

/// Намір агента
#[derive(Clone, Debug)]
pub struct Intention {
    pub desire_id: u32,         // Яке бажання реалізує
    pub plan: Plan,
    pub started_at: Instant,
    pub commitment: u8,         // Рівень прихильності (0-100)
}

impl Intention {
    pub fn new(desire_id: u32, plan: Plan) -> Self {
        Self {
            desire_id,
            plan,
            started_at: Instant::now(),
            commitment: 100,
        }
    }
    
    /// Зменшує commitment (наприклад, при труднощах)
    pub fn weaken_commitment(&mut self, amount: u8) {
        self.commitment = self.commitment.saturating_sub(amount);
    }
    
    /// Перевіряє, чи варто переглянути намір
    pub fn should_reconsider(&self) -> bool {
        self.commitment < 30
    }
}
```

### 49.4.4 BDI-агент: об'єднання компонентів

**Постановка задачі:** Створимо повного BDI-агента з циклом deliberation.

```rust
/// BDI-агент
pub struct BDIAgent {
    pub id: u32,
    pub beliefs: Beliefs,
    pub desires: Desires,
    pub current_intention: Option<Intention>,
}

impl BDIAgent {
    pub fn new(id: u32, start_pos: Position, base_pos: Position) -> Self {
        let mut agent = Self {
            id,
            beliefs: Beliefs::new(start_pos, base_pos),
            desires: Desires::new(),
            current_intention: None,
        };
        
        // Додаємо базові maintenance desires
        agent.desires.add(
            DesireType::MaintainBattery(20),
            250,  // Високий пріоритет
            true,
        );
        
        agent
    }
    
    /// Оновлює beliefs на основі perception
    pub fn perceive(&mut self, new_position: Position, battery: u8) {
        self.beliefs.update_position(new_position);
        self.beliefs.my_battery = battery;
    }
    
    /// Генерує нові desires на основі beliefs
    pub fn generate_options(&mut self) {
        // Якщо батарея низька — додати desire повернення
        if self.beliefs.my_battery < 25 {
            // Перевіряємо, чи вже є такий desire
            let has_return = self.desires.by_priority().iter()
                .any(|d| matches!(d.desire_type, DesireType::ReturnToBase));
            
            if !has_return {
                self.desires.add(DesireType::ReturnToBase, 200, false);
            }
        }
        
        // Якщо є невідомі цілі — додати desires для них
        for target in &self.beliefs.known_targets {
            // Спрощено: додаємо desire досягти цілі
            self.desires.add(
                DesireType::ReachPosition(target.position),
                target.priority,
                false,
            );
        }
    }
    
    /// Процес deliberation — вибір intention
    pub fn deliberate(&mut self) {
        // Перевіряємо, чи потрібно переглянути поточний intention
        if let Some(ref intention) = self.current_intention {
            if !intention.should_reconsider() && !intention.plan.is_complete() {
                return; // Продовжуємо поточний план
            }
        }
        
        // Вибираємо найпріоритетніший desire
        let desires = self.desires.by_priority();
        if let Some(desire) = desires.first() {
            // Генеруємо план для цього desire
            if let Some(plan) = self.create_plan(&desire.desire_type) {
                self.current_intention = Some(Intention::new(desire.id, plan));
            }
        }
    }
    
    /// Створює план для desire (means-ends reasoning)
    fn create_plan(&self, desire_type: &DesireType) -> Option<Plan> {
        match desire_type {
            DesireType::ReachPosition(target) => {
                Some(Plan::new(vec![
                    PlanStep::MoveTo(*target),
                    PlanStep::Hover(2.0),
                    PlanStep::SendReport,
                ]))
            }
            DesireType::ReturnToBase => {
                Some(Plan::new(vec![
                    PlanStep::MoveTo(self.beliefs.base_position),
                    PlanStep::Hover(1.0),
                ]))
            }
            DesireType::ExploreSector(sector_id) => {
                // Спрощений план розвідки
                Some(Plan::new(vec![
                    PlanStep::MoveTo(Position { x: *sector_id as f64 * 100.0, y: 0.0 }),
                    PlanStep::Scan,
                    PlanStep::SendReport,
                ]))
            }
            _ => None,
        }
    }
    
    /// Виконує поточний крок плану
    pub fn execute(&mut self) -> Option<PlanStep> {
        if let Some(ref mut intention) = self.current_intention {
            if let Some(step) = intention.plan.current() {
                return Some(step.clone());
            }
        }
        None
    }
    
    /// Позначає поточний крок завершеним
    pub fn step_completed(&mut self) {
        if let Some(ref mut intention) = self.current_intention {
            intention.plan.advance();
            
            // Якщо план завершено — видаляємо intention
            if intention.plan.is_complete() {
                self.desires.remove(intention.desire_id);
                self.current_intention = None;
            }
        }
    }
    
    /// Повний цикл BDI
    pub fn tick(&mut self) -> Option<PlanStep> {
        self.generate_options();
        self.deliberate();
        self.execute()
    }
}
```

**Розглянемо цикл BDI-агента:**

1. **`perceive()`** — оновлює beliefs з нових даних
2. **`generate_options()`** — створює desires на основі beliefs
3. **`deliberate()`** — вибирає intention з desires
4. **`execute()`** — повертає поточну дію для виконання
5. **`step_completed()`** — просуває план вперед

---

## 49.5 ПЛАН-БІБЛІОТЕКА ДЛЯ АГЕНТІВ БПЛА

### 49.5.1 Концепція Plan Library

Замість генерації планів "з нуля", практичні BDI-системи використовують **бібліотеку готових планів**:

```text
┌─────────────────────────────────────────────────────────────────────┐
│                    PLAN LIBRARY СТРУКТУРА                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ПЛАН: Розвідка сектору                                            │
│  ────────────────────────                                          │
│  ТРИГЕР: desire = ExploreSector(S)                                 │
│  ПЕРЕДУМОВА: battery > 40% AND NOT threat_in(S)                    │
│  КРОКИ:                                                            │
│    1. fly_to(S.entry_point)                                        │
│    2. execute_pattern("snake", S.bounds)                           │
│    3. fly_to(start_position)                                       │
│    4. send_report()                                                │
│  ПОСТУМОВА: sector_explored(S)                                     │
│                                                                     │
│  ─────────────────────────────────────────────────────────────────│
│                                                                     │
│  ПЛАН: Екстрене повернення                                         │
│  ─────────────────────────                                         │
│  ТРИГЕР: battery < 20% OR critical_damage                          │
│  ПЕРЕДУМОВА: TRUE (завжди застосовний)                             │
│  ПРІОРИТЕТ: МАКСИМАЛЬНИЙ                                           │
│  КРОКИ:                                                            │
│    1. abort_current_mission()                                      │
│    2. fly_direct_to(base)                                          │
│    3. request_landing()                                            │
│  ПОСТУМОВА: at_base AND safe                                       │
│                                                                     │
│  ─────────────────────────────────────────────────────────────────│
│                                                                     │
│  ПЛАН: Уникнення загрози                                           │
│  ────────────────────────                                          │
│  ТРИГЕР: threat_detected(T) AND distance(T) < safe_margin          │
│  ПЕРЕДУМОВА: NOT at_base                                           │
│  КРОКИ:                                                            │
│    1. calculate_escape_vector(T)                                   │
│    2. fly_to(escape_point)                                         │
│    3. reassess_situation()                                         │
│  ПОСТУМОВА: distance(T) > safe_margin                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 49.5.2 Реалізація Plan Library

**Постановка задачі:** Створимо бібліотеку планів з перевіркою передумов.

```rust
/// Передумова плану
pub type Precondition = Box<dyn Fn(&Beliefs) -> bool>;

/// Шаблон плану
pub struct PlanTemplate {
    pub name: String,
    pub trigger: DesireType,
    pub precondition: Precondition,
    pub priority: u8,
    pub create_steps: Box<dyn Fn(&Beliefs, &DesireType) -> Vec<PlanStep>>,
}

/// Бібліотека планів
pub struct PlanLibrary {
    templates: Vec<PlanTemplate>,
}

impl PlanLibrary {
    pub fn new() -> Self {
        let mut lib = Self { templates: Vec::new() };
        lib.register_default_plans();
        lib
    }
    
    /// Реєструє стандартні плани
    fn register_default_plans(&mut self) {
        // План: повернення на базу
        self.templates.push(PlanTemplate {
            name: "ReturnToBase".to_string(),
            trigger: DesireType::ReturnToBase,
            precondition: Box::new(|_| true),  // Завжди застосовний
            priority: 200,
            create_steps: Box::new(|beliefs, _| {
                vec![
                    PlanStep::MoveTo(beliefs.base_position),
                    PlanStep::Hover(1.0),
                ]
            }),
        });
        
        // План: досягти позиції
        self.templates.push(PlanTemplate {
            name: "ReachPosition".to_string(),
            trigger: DesireType::ReachPosition(Position { x: 0.0, y: 0.0 }),
            precondition: Box::new(|beliefs| beliefs.my_battery > 30),
            priority: 100,
            create_steps: Box::new(|_, desire| {
                if let DesireType::ReachPosition(pos) = desire {
                    vec![
                        PlanStep::MoveTo(*pos),
                        PlanStep::Hover(2.0),
                        PlanStep::SendReport,
                    ]
                } else {
                    vec![]
                }
            }),
        });
    }
    
    /// Знаходить застосовний план для desire
    pub fn find_plan(&self, desire: &DesireType, beliefs: &Beliefs) -> Option<Plan> {
        for template in &self.templates {
            // Перевіряємо, чи тригер відповідає
            if self.trigger_matches(&template.trigger, desire) {
                // Перевіряємо передумову
                if (template.precondition)(beliefs) {
                    let steps = (template.create_steps)(beliefs, desire);
                    return Some(Plan::new(steps));
                }
            }
        }
        None
    }
    
    /// Перевіряє відповідність тригера
    fn trigger_matches(&self, trigger: &DesireType, desire: &DesireType) -> bool {
        std::mem::discriminant(trigger) == std::mem::discriminant(desire)
    }
}
```

---

## 49.6 ЛАБОРАТОРНА РОБОТА

### Мета роботи

Реалізувати BDI-агента для виконання розвідувальної місії.

### Завдання 1: Розширення Beliefs (3 бали)

Додайте до структури `Beliefs`:
- Історію відвіданих позицій (останні 10)
- Оцінку "небезпечності" різних секторів
- Час з останнього зв'язку з базою

### Завдання 2: Нові Desires (3 бали)

Реалізуйте нові типи desires:
- `FollowAlly(agent_id)` — слідувати за союзником
- `InvestigateAnomaly(position)` — дослідити аномалію

### Завдання 3: Адаптивний Deliberation (4 бали)

Реалізуйте механізм, що:
- Знижує commitment при повторних невдачах
- Автоматично переглядає intention при критичних змінах
- Логує причини перегляду для пояснюваності

---

## 49.7 РЕЗЮМЕ

### Ключові концепції

**BDI** — когнітивна архітектура: Beliefs (знання) + Desires (цілі) + Intentions (плани).

**Deliberation** — процес вибору намірів з бажань.

**Means-ends reasoning** — пошук плану для досягнення цілі.

**Commitment** — прихильність до наміру; баланс між наполегливістю та гнучкістю.

**Plan Library** — готові шаблони планів для типових ситуацій.

### Коли використовувати BDI

| Сценарій | BDI? |
|----------|------|
| Складні місії з кількома цілями | ✅ Ідеально |
| Потреба в поясненні рішень | ✅ Добре |
| Динамічне, але не хаотичне середовище | ✅ Добре |
| Критичний час реакції (< 10ms) | ❌ Занадто повільно |
| Дуже прості задачі | ❌ Overkill |

---

## 🔗 ЗВ'ЯЗОК З НАСТУПНИМ РОЗДІЛОМ

BDI-агент — "мозок" окремого агента. Але для рою потрібна **комунікація**. Як агенти обмінюються beliefs? Як координують intentions? Як синхронізують плани?

У **Розділі 50: Мережева комунікація агентів** ви:
- Вивчите протоколи комунікації між агентами
- Реалізуєте мережевий обмін повідомленнями
- Побудуєте розподілений рій з TCP/UDP
- Дізнаєтесь про discovery та heartbeat механізми

---

> **Наступний розділ:** [Розділ 50: Мережева комунікація агентів](./50_Network_Communication.md)
