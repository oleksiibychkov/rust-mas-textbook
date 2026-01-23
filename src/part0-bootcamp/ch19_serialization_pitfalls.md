# Підступні задачі з серіалізацією

---

## 📋 Анотація

Цей розділ розкриває неочевидні аспекти серіалізації та десеріалізації даних, які можуть призвести до тихої втрати інформації, несумісності між системами та загадкових помилок "на рівному місці". Ви дізнаєтесь, чому 64-бітне ціле число може "зіпсуватись" при передачі через JSON, чому NaN та Infinity неможливо серіалізувати в стандартний JSON, та як поле, що існує в структурі, може зникнути після round-trip серіалізації. У контексті рою БПЛА коректна серіалізація критична: телеметрія, команди, координати — все передається між системами, і помилка може означати втрату точності позиціонування або неправильну інтерпретацію команд.

---

## 🎯 Цілі навчання

Після завершення цього розділу ви зможете:

1. **Пояснити** обмеження JSON для числових типів
2. **Уникати** втрати точності при серіалізації великих чисел
3. **Обробляти** спеціальні значення float (NaN, Infinity)
4. **Правильно серіалізувати** enum-и та Option
5. **Розуміти** різницю між форматами (JSON, TOML, bincode)
6. **Використовувати** serde атрибути для кастомізації
7. **Проектувати** схеми даних з урахуванням версіонування

---

## 📚 Ключові терміни

| Термін | Визначення |
|--------|------------|
| **серіалізація** | Перетворення структури даних у послідовність байтів або текст |
| **десеріалізація** | Зворотне перетворення — з байтів/тексту у структуру |
| **serde** | Стандартна бібліотека Rust для серіалізації/десеріалізації |
| **JSON** | JavaScript Object Notation — текстовий формат обміну даними |
| **round-trip** | Цикл серіалізація → десеріалізація, ідеально без втрат |
| **schema evolution** | Зміна формату даних зі збереженням сумісності |

---

## 💡 Мотиваційний кейс: Зниклі мільйони

Фінтех-стартап розробляв систему обробки транзакцій. Backend на Rust обмінювався даними з frontend на JavaScript через JSON API. Все працювало ідеально — доки сума транзакції не перевищила певний поріг.

Клієнт відправив переказ на 9,007,199,254,740,993 центів (приблизно 90 трильйонів доларів — тестова транзакція у sandbox-середовищі). Backend отримав і обробив запит. Але коли клієнт перевірив статус, сума була 9,007,199,254,740,992 — на один цент менше!

Причина: JavaScript Number — це 64-бітний float (IEEE 754 double). Він може точно представити цілі числа лише до 2^53 - 1 = 9,007,199,254,740,991. Число 9,007,199,254,740,993 округляється до найближчого представимого значення.

```javascript
// JavaScript
console.log(9007199254740993);  // 9007199254740992
console.log(9007199254740993 === 9007199254740992);  // true!
```

JSON успадкував це обмеження, бо специфікація не визначає точність чисел. Rust-сервер відправляв правильне число, але JavaScript-клієнт (і багато JSON-парсерів) інтерпретував його неправильно.

Рішення: серіалізувати великі числа як рядки:

```json
{
  "amount": "9007199254740993",
  "currency": "cents"
}
```

Цей інцидент не призвів до фінансових втрат (тестове середовище), але виявив критичну проблему: дані, що здаються коректними на одній стороні, можуть бути пошкодженими на іншій.

---

## ТЕОРІЯ: JSON І ЧИСЛА — НЕБЕЗПЕЧНИЙ СОЮЗ

### Проблема: JSON не визначає точність

Специфікація JSON (RFC 8259) говорить про числа:

> "This specification allows implementations to set limits on the range and precision of numbers accepted."

Тобто кожна реалізація може мати власні обмеження. На практиці більшість парсерів використовують IEEE 754 double (64-бітний float), що дає:

- **Безпечний діапазон цілих**: -(2^53 - 1) до (2^53 - 1), тобто ±9,007,199,254,740,991
- **Максимальне значення**: приблизно ±1.8 × 10^308
- **Точність**: приблизно 15-17 значущих десяткових цифр

### Що відбувається з i64 та u64

Rust типи `i64` та `u64` мають значно більший діапазон:

- **i64**: -9,223,372,036,854,775,808 до 9,223,372,036,854,775,807
- **u64**: 0 до 18,446,744,073,709,551,615

Коли ви серіалізуєте `u64::MAX` у JSON і JavaScript його читає:

```rust
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize)]
struct Data {
    id: u64,
}

let data = Data { id: u64::MAX };  // 18446744073709551615
let json = serde_json::to_string(&data).unwrap();
// json = {"id":18446744073709551615}
```

```javascript
// JavaScript
const data = JSON.parse('{"id":18446744073709551615}');
console.log(data.id);  // 18446744073709552000 — НЕПРАВИЛЬНО!
```

Останні цифри втрачено. JavaScript "бачить" інше число.

### Коли це небезпечно

**Ідентифікатори**: UUID, snowflake IDs, database primary keys часто є великими числами.

```rust
struct User {
    id: u64,  // Snowflake ID може бути > 2^53
}
```

**Фінансові суми**: при роботі з найменшими одиницями (центи, копійки).

**Timestamp у мікросекундах**: Unix timestamp у мікросекундах вже перевищує безпечний діапазон.

```rust
let micros = std::time::SystemTime::now()
    .duration_since(std::time::UNIX_EPOCH)
    .unwrap()
    .as_micros() as u64;
// micros ≈ 1,705,000,000,000,000 — поки безпечно, але росте
```

### Рішення: серіалізація як рядок

```rust
use serde::{Serialize, Deserialize};
use serde_with::{serde_as, DisplayFromStr};

#[serde_as]
#[derive(Serialize, Deserialize)]
struct Data {
    #[serde_as(as = "DisplayFromStr")]
    id: u64,
}

// Або вручну
#[derive(Serialize, Deserialize)]
struct DataManual {
    #[serde(with = "string_u64")]
    id: u64,
}

mod string_u64 {
    use serde::{self, Deserialize, Deserializer, Serializer};
    
    pub fn serialize<S>(value: &u64, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        serializer.serialize_str(&value.to_string())
    }
    
    pub fn deserialize<'de, D>(deserializer: D) -> Result<u64, D::Error>
    where
        D: Deserializer<'de>,
    {
        let s = String::deserialize(deserializer)?;
        s.parse().map_err(serde::de::Error::custom)
    }
}
```

Результат: `{"id":"18446744073709551615"}` — рядок, не число.

### Рішення: використання i128/u128

Якщо вам потрібна арифметика і JSON не покидає Rust-екосистему:

```rust
#[derive(Serialize, Deserialize)]
struct HighPrecision {
    #[serde(with = "string_u128")]
    value: u128,
}
```

Але пам'ятайте: serde_json за замовчуванням не підтримує u128/i128 як числа JSON.

---

## ТЕОРІЯ: NAN ТА INFINITY — ЗАБОРОНЕНІ ЗНАЧЕННЯ

### JSON не підтримує спеціальні float значення

IEEE 754 float має три спеціальні значення:
- **NaN** (Not a Number): результат 0.0/0.0, sqrt(-1)
- **Infinity**: результат 1.0/0.0
- **-Infinity**: результат -1.0/0.0

JSON не має синтаксису для цих значень. Спроба серіалізувати їх призводить до помилки:

```rust
use serde_json;

let data = f64::NAN;
let result = serde_json::to_string(&data);
// result = Err(Error("NaN is not a valid JSON value"))

let data = f64::INFINITY;
let result = serde_json::to_string(&data);
// result = Err(Error("Infinity is not a valid JSON value"))
```

### Коли це стає проблемою

**Обчислення з невалідними даними**:

```rust
#[derive(Serialize)]
struct SensorReading {
    temperature: f64,
    pressure: f64,
}

fn calculate_derived(temp: f64, pressure: f64) -> f64 {
    if pressure == 0.0 {
        return f64::INFINITY;  // Або NAN
    }
    temp / pressure
}

let reading = SensorReading {
    temperature: 25.0,
    pressure: calculate_derived(100.0, 0.0),  // Infinity!
};

serde_json::to_string(&reading).unwrap();  // ПАНІКА!
```

**Статистичні обчислення**:

```rust
fn average(values: &[f64]) -> f64 {
    if values.is_empty() {
        return f64::NAN;  // Середнє пустої множини — невизначене
    }
    values.iter().sum::<f64>() / values.len() as f64
}
```

### Рішення 1: Перевірка перед серіалізацією

```rust
fn safe_serialize<T: Serialize>(value: &T) -> Result<String, Error> {
    // Спочатку серіалізуємо в Value для перевірки
    let json_value = serde_json::to_value(value)?;
    check_for_special_floats(&json_value)?;
    serde_json::to_string(&json_value)
}

fn check_for_special_floats(value: &serde_json::Value) -> Result<(), Error> {
    match value {
        serde_json::Value::Number(n) => {
            if let Some(f) = n.as_f64() {
                if f.is_nan() || f.is_infinite() {
                    return Err(Error::SpecialFloat);
                }
            }
            Ok(())
        }
        serde_json::Value::Array(arr) => {
            for item in arr {
                check_for_special_floats(item)?;
            }
            Ok(())
        }
        serde_json::Value::Object(obj) => {
            for (_, v) in obj {
                check_for_special_floats(v)?;
            }
            Ok(())
        }
        _ => Ok(()),
    }
}
```

### Рішення 2: Кастомна серіалізація з заміною

```rust
use serde::{Serialize, Serializer};

fn serialize_safe_f64<S>(value: &f64, serializer: S) -> Result<S::Ok, S::Error>
where
    S: Serializer,
{
    if value.is_nan() {
        serializer.serialize_none()  // null замість NaN
    } else if value.is_infinite() {
        if *value > 0.0 {
            serializer.serialize_f64(f64::MAX)
        } else {
            serializer.serialize_f64(f64::MIN)
        }
    } else {
        serializer.serialize_f64(*value)
    }
}

#[derive(Serialize)]
struct Reading {
    #[serde(serialize_with = "serialize_safe_f64")]
    value: f64,
}
```

### Рішення 3: Обгортка Option

```rust
#[derive(Serialize, Deserialize)]
struct Reading {
    value: Option<f64>,  // None якщо NaN або невалідне
}

impl Reading {
    fn new(v: f64) -> Self {
        Self {
            value: if v.is_finite() { Some(v) } else { None }
        }
    }
}
```

### Рішення 4: Бінарні формати

Формати як bincode, MessagePack, CBOR підтримують спеціальні float значення:

```rust
use bincode;

let data = f64::NAN;
let bytes = bincode::serialize(&data).unwrap();  // OK!
let restored: f64 = bincode::deserialize(&bytes).unwrap();
assert!(restored.is_nan());
```

---

## ТЕОРІЯ: ENUM СЕРІАЛІЗАЦІЯ — НЕСПОДІВАНІ ФОРМАТИ

### За замовчуванням: externally tagged

```rust
#[derive(Serialize, Deserialize)]
enum Status {
    Active,
    Inactive,
    Pending { since: String },
}

let status = Status::Pending { since: "2024-01-01".into() };
let json = serde_json::to_string(&status).unwrap();
// {"Pending":{"since":"2024-01-01"}}
```

Формат: об'єкт з одним полем, де ключ — назва варіанту.

### Проблема: несумісність з іншими системами

Багато API очікують інший формат:

```json
// Що очікує API
{ "status": "pending", "since": "2024-01-01" }

// Що генерує serde за замовчуванням
{ "Pending": { "since": "2024-01-01" } }
```

### Варіанти представлення enum

**Internally tagged**:
```rust
#[derive(Serialize, Deserialize)]
#[serde(tag = "type")]
enum Event {
    Click { x: i32, y: i32 },
    KeyPress { key: String },
}

// {"type":"Click","x":10,"y":20}
```

**Adjacently tagged**:
```rust
#[derive(Serialize, Deserialize)]
#[serde(tag = "type", content = "data")]
enum Event {
    Click { x: i32, y: i32 },
    KeyPress { key: String },
}

// {"type":"Click","data":{"x":10,"y":20}}
```

**Untagged**:
```rust
#[derive(Serialize, Deserialize)]
#[serde(untagged)]
enum StringOrInt {
    String(String),
    Int(i32),
}

// "hello" або 42
```

### Пастка: untagged enum і порядок варіантів

```rust
#[derive(Serialize, Deserialize)]
#[serde(untagged)]
enum Value {
    Int(i32),
    Float(f64),  // Float завжди матчиться!
}

let json = "42";
let v: Value = serde_json::from_str(json).unwrap();
// v = Float(42.0), не Int(42)!
```

При десеріалізації untagged enum serde пробує варіанти по порядку. `f64` може десеріалізувати будь-яке число JSON, тому `Int` ніколи не спрацює.

Рішення: змініть порядок або використовуйте tagged enum.

---

## ТЕОРІЯ: OPTION І NULL — ТОНКОЩІ

### None серіалізується як відсутність поля

```rust
#[derive(Serialize)]
struct User {
    name: String,
    email: Option<String>,
}

let user = User {
    name: "Alice".into(),
    email: None,
};
let json = serde_json::to_string(&user).unwrap();
// {"name":"Alice"}  — поле email ВІДСУТНЄ
```

### Щоб отримати null, потрібен атрибут

```rust
#[derive(Serialize)]
struct User {
    name: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    email: Option<String>,  // Відсутнє якщо None
}

// Або для явного null:
#[derive(Serialize)]
struct UserWithNull {
    name: String,
    email: Option<String>,  // Без skip — буде null
}
// {"name":"Alice","email":null}
```

### Пастка: десеріалізація відсутнього поля

```rust
#[derive(Deserialize)]
struct Config {
    timeout: Option<u32>,
}

// Це працює:
let c: Config = serde_json::from_str(r#"{"timeout": null}"#).unwrap();
// c.timeout = None

// І це працює:
let c: Config = serde_json::from_str(r#"{}"#).unwrap();
// c.timeout = None
```

Але якщо поле НЕ Option:

```rust
#[derive(Deserialize)]
struct Config {
    timeout: u32,
}

let c: Config = serde_json::from_str(r#"{}"#).unwrap();
// ПОМИЛКА: missing field `timeout`
```

### Значення за замовчуванням

```rust
#[derive(Deserialize)]
struct Config {
    #[serde(default)]
    timeout: u32,  // 0 якщо відсутнє
    
    #[serde(default = "default_retries")]
    retries: u32,  // 3 якщо відсутнє
}

fn default_retries() -> u32 {
    3
}
```

---

## ТЕОРІЯ: ПОРЯДОК ПОЛІВ І ДЕТЕРМІНІЗМ

### HashMap не гарантує порядок

```rust
use std::collections::HashMap;

let mut map = HashMap::new();
map.insert("z", 1);
map.insert("a", 2);
map.insert("m", 3);

let json = serde_json::to_string(&map).unwrap();
// Порядок ключів НЕВИЗНАЧЕНИЙ і може змінюватись!
```

### Чому це проблема

**Тестування**: тести, що порівнюють JSON-рядки, будуть нестабільними.

**Кешування**: хеш JSON-рядка буде різним для однакових даних.

**Diff та version control**: зміни порядку створюють "фантомні" diff-и.

**Підписи та хеші**: криптографічний підпис JSON буде різним.

### Рішення: BTreeMap або indexmap

```rust
use std::collections::BTreeMap;

let mut map = BTreeMap::new();
map.insert("z", 1);
map.insert("a", 2);
map.insert("m", 3);

let json = serde_json::to_string(&map).unwrap();
// {"a":2,"m":3,"z":1}  — завжди відсортовано за ключем
```

```rust
use indexmap::IndexMap;

let mut map = IndexMap::new();
map.insert("z", 1);
map.insert("a", 2);

let json = serde_json::to_string(&map).unwrap();
// {"z":1,"a":2}  — порядок вставки збережено
```

### Порядок полів структури

Поля структури серіалізуються в порядку оголошення:

```rust
#[derive(Serialize)]
struct Data {
    zebra: i32,  // Перше
    alpha: i32,  // Друге
}

let d = Data { zebra: 1, alpha: 2 };
let json = serde_json::to_string(&d).unwrap();
// {"zebra":1,"alpha":2}  — порядок стабільний
```

---

## ТЕОРІЯ: БІНАРНІ ДАНІ ТА BASE64

### JSON не підтримує бінарні дані

JSON — текстовий формат. Бінарні дані (байти, зображення, файли) потрібно кодувати.

```rust
#[derive(Serialize, Deserialize)]
struct Document {
    name: String,
    #[serde(with = "base64_serde")]
    content: Vec<u8>,
}

mod base64_serde {
    use base64::{Engine, engine::general_purpose::STANDARD};
    use serde::{Deserialize, Deserializer, Serialize, Serializer};
    
    pub fn serialize<S>(bytes: &[u8], serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        let encoded = STANDARD.encode(bytes);
        encoded.serialize(serializer)
    }
    
    pub fn deserialize<'de, D>(deserializer: D) -> Result<Vec<u8>, D::Error>
    where
        D: Deserializer<'de>,
    {
        let s = String::deserialize(deserializer)?;
        STANDARD.decode(&s).map_err(serde::de::Error::custom)
    }
}
```

### Overhead base64

Base64 збільшує розмір на ~33%. 3 байти → 4 символи.

Для великих обсягів даних розгляньте:
- Окрему передачу бінарних даних (multipart, presigned URLs)
- Бінарні формати (bincode, MessagePack, Protocol Buffers)

---

## ТЕОРІЯ: ВЕРСІОНУВАННЯ СХЕМИ

### Проблема: схема змінюється

Ваша система еволюціонує. Структури даних змінюються. Але старі дані (в базі, в кеші, в логах) залишаються у старому форматі.

```rust
// Версія 1
struct UserV1 {
    name: String,
}

// Версія 2: додали email
struct UserV2 {
    name: String,
    email: String,  // Нове поле!
}
```

Якщо спробувати десеріалізувати старі дані у нову структуру:

```rust
let old_json = r#"{"name":"Alice"}"#;
let user: UserV2 = serde_json::from_str(old_json).unwrap();
// ПОМИЛКА: missing field `email`
```

### Рішення: зворотня сумісність через Option та default

```rust
#[derive(Deserialize)]
struct User {
    name: String,
    
    #[serde(default)]
    email: Option<String>,  // None для старих даних
    
    #[serde(default = "default_version")]
    schema_version: u32,
}

fn default_version() -> u32 {
    1  // Старі дані без версії вважаються v1
}
```

### Рішення: явне версіонування

```rust
#[derive(Deserialize)]
#[serde(tag = "version")]
enum VersionedUser {
    #[serde(rename = "1")]
    V1 { name: String },
    
    #[serde(rename = "2")]
    V2 { name: String, email: String },
}

impl VersionedUser {
    fn to_latest(self) -> UserV2 {
        match self {
            VersionedUser::V1 { name } => UserV2 {
                name,
                email: String::new(),
            },
            VersionedUser::V2 { name, email } => UserV2 { name, email },
        }
    }
}
```

### Рішення: deny_unknown_fields для строгості

```rust
#[derive(Deserialize)]
#[serde(deny_unknown_fields)]
struct StrictConfig {
    timeout: u32,
    retries: u32,
}

let json = r#"{"timeout":10,"retries":3,"unknown":"field"}"#;
let config: StrictConfig = serde_json::from_str(json).unwrap();
// ПОМИЛКА: unknown field `unknown`
```

Це допомагає виявити помилки в назвах полів та несумісні версії.

---

## ТЕОРІЯ: ФОРМАТИ ТА ЇХ ОСОБЛИВОСТІ

### JSON

**Плюси**: людиночитабельний, універсальний, широка підтримка.
**Мінуси**: verbose, не підтримує NaN/Infinity, проблеми з великими числами, немає бінарних даних.

```rust
use serde_json;
let json = serde_json::to_string(&data)?;
```

### TOML

**Плюси**: людиночитабельний, чудовий для конфігурацій, підтримує дати.
**Мінуси**: не підходить для складних вкладених структур, менш універсальний.

```rust
use toml;
let toml_str = toml::to_string(&config)?;
```

### bincode

**Плюси**: компактний, швидкий, підтримує всі Rust-типи.
**Мінуси**: не людиночитабельний, Rust-специфічний, не стабільний формат.

```rust
use bincode;
let bytes = bincode::serialize(&data)?;
```

### MessagePack

**Плюси**: компактний, швидкий, крос-платформний, підтримує бінарні дані.
**Мінуси**: не людиночитабельний.

```rust
use rmp_serde;
let bytes = rmp_serde::to_vec(&data)?;
```

### Порівняння

| Формат | Розмір | Швидкість | Читабельність | Крос-платформ |
|--------|--------|-----------|---------------|---------------|
| JSON | Великий | Середня | ✓ | ✓ |
| TOML | Середній | Середня | ✓ | ✓ |
| bincode | Малий | Швидка | ✗ | Rust-only |
| MessagePack | Малий | Швидка | ✗ | ✓ |

---

## ПРАКТИЧНІ РЕКОМЕНДАЦІЇ

### Правило 1: Великі числа як рядки в JSON

```rust
#[derive(Serialize, Deserialize)]
struct Transaction {
    #[serde(with = "string_u64")]
    id: u64,
    
    #[serde(with = "string_u64")]
    amount_cents: u64,
}
```

### Правило 2: Обробляйте спеціальні float значення

```rust
fn validate_for_json(value: f64) -> Result<f64, Error> {
    if value.is_nan() {
        Err(Error::NaN)
    } else if value.is_infinite() {
        Err(Error::Infinite)
    } else {
        Ok(value)
    }
}
```

### Правило 3: Використовуйте default для зворотньої сумісності

```rust
#[derive(Deserialize)]
struct Config {
    #[serde(default)]
    new_field: Option<String>,
    
    #[serde(default = "default_timeout")]
    timeout: u32,
}
```

### Правило 4: BTreeMap для детермінованого порядку

```rust
use std::collections::BTreeMap;

#[derive(Serialize)]
struct Data {
    metadata: BTreeMap<String, String>,  // Завжди відсортовано
}
```

### Правило 5: Явно вказуйте формат enum

```rust
#[derive(Serialize, Deserialize)]
#[serde(tag = "type")]  // Явний формат
enum Command {
    Start,
    Stop,
    Move { x: f64, y: f64 },
}
```

### Правило 6: Тестуйте round-trip

```rust
#[test]
fn test_roundtrip() {
    let original = Data {
        id: u64::MAX,
        value: 3.14159,
    };
    
    let json = serde_json::to_string(&original).unwrap();
    let restored: Data = serde_json::from_str(&json).unwrap();
    
    assert_eq!(original.id, restored.id);
    assert!((original.value - restored.value).abs() < 1e-10);
}
```

---

## Застосування до рою БПЛА

### Телеметрія з безпечними числами

```rust
use serde::{Serialize, Deserialize};
use std::collections::BTreeMap;

#[derive(Serialize, Deserialize)]
pub struct DroneTelemetry {
    /// ID дрона як рядок для сумісності з JavaScript
    #[serde(with = "string_u64")]
    pub drone_id: u64,
    
    /// Timestamp як рядок (мікросекунди можуть перевищити safe integer)
    #[serde(with = "string_u64")]
    pub timestamp_us: u64,
    
    /// Позиція — звичайні float, бо координати в безпечному діапазоні
    pub position: Position,
    
    /// Батарея — може бути NaN якщо сенсор відмовив
    #[serde(serialize_with = "serialize_optional_f64")]
    pub battery_percent: Option<f64>,
    
    /// Метадані в детермінованому порядку
    pub metadata: BTreeMap<String, String>,
}

#[derive(Serialize, Deserialize)]
pub struct Position {
    pub latitude: f64,
    pub longitude: f64,
    pub altitude: f64,
}

fn serialize_optional_f64<S>(value: &Option<f64>, serializer: S) -> Result<S::Ok, S::Error>
where
    S: serde::Serializer,
{
    match value {
        Some(v) if v.is_finite() => serializer.serialize_some(v),
        _ => serializer.serialize_none(),
    }
}

mod string_u64 {
    use serde::{self, Deserialize, Deserializer, Serializer};
    
    pub fn serialize<S>(value: &u64, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        serializer.serialize_str(&value.to_string())
    }
    
    pub fn deserialize<'de, D>(deserializer: D) -> Result<u64, D::Error>
    where
        D: Deserializer<'de>,
    {
        let s = String::deserialize(deserializer)?;
        s.parse().map_err(serde::de::Error::custom)
    }
}
```

### Команди з версіонуванням

```rust
#[derive(Serialize, Deserialize)]
#[serde(tag = "command", rename_all = "snake_case")]
pub enum DroneCommand {
    /// Зліт на задану висоту
    Takeoff { 
        altitude: f64,
        #[serde(default)]
        speed: Option<f64>,
    },
    
    /// Посадка
    Land {
        #[serde(default)]
        emergency: bool,
    },
    
    /// Переміщення до точки
    MoveTo {
        target: Position,
        #[serde(default = "default_speed")]
        speed: f64,
    },
    
    /// Виконати місію (ID як рядок!)
    ExecuteMission {
        #[serde(with = "string_u64")]
        mission_id: u64,
    },
}

fn default_speed() -> f64 {
    5.0  // м/с
}
```

### Бінарний протокол для критичних даних

```rust
use bincode;

/// Компактний формат для high-frequency телеметрії
#[derive(Serialize, Deserialize)]
pub struct CompactTelemetry {
    pub drone_id: u32,
    pub timestamp_ms: u64,
    pub lat: f32,  // Достатня точність для координат
    pub lon: f32,
    pub alt: f32,
    pub battery: u8,  // 0-100
    pub status: DroneStatus,
}

impl CompactTelemetry {
    pub fn to_bytes(&self) -> Vec<u8> {
        bincode::serialize(self).expect("Serialization cannot fail")
    }
    
    pub fn from_bytes(bytes: &[u8]) -> Result<Self, bincode::Error> {
        bincode::deserialize(bytes)
    }
}

// Розмір: ~25 байт замість ~200+ байт JSON
```

### Валідація вхідних даних

```rust
impl DroneTelemetry {
    pub fn validate(&self) -> Result<(), ValidationError> {
        // Перевірка координат
        if !(-90.0..=90.0).contains(&self.position.latitude) {
            return Err(ValidationError::InvalidLatitude(self.position.latitude));
        }
        if !(-180.0..=180.0).contains(&self.position.longitude) {
            return Err(ValidationError::InvalidLongitude(self.position.longitude));
        }
        
        // Перевірка висоти (NaN та Infinity недопустимі)
        if !self.position.altitude.is_finite() {
            return Err(ValidationError::InvalidAltitude);
        }
        if self.position.altitude < -100.0 || self.position.altitude > 50000.0 {
            return Err(ValidationError::AltitudeOutOfRange(self.position.altitude));
        }
        
        // Перевірка батареї
        if let Some(battery) = self.battery_percent {
            if !(0.0..=100.0).contains(&battery) {
                return Err(ValidationError::InvalidBattery(battery));
            }
        }
        
        Ok(())
    }
}

pub fn deserialize_and_validate(json: &str) -> Result<DroneTelemetry, Error> {
    let telemetry: DroneTelemetry = serde_json::from_str(json)?;
    telemetry.validate()?;
    Ok(telemetry)
}
```

---

## Резюме

У цьому розділі ми розглянули підступні аспекти серіалізації.

**JSON і великі числа**: JavaScript Number точний лише до 2^53. u64 та i64 можуть втрачати точність. Серіалізуйте великі числа як рядки.

**NaN та Infinity**: JSON не підтримує спеціальні float значення. Перевіряйте перед серіалізацією або використовуйте Option/null.

**Enum серіалізація**: serde має кілька форматів (externally tagged, internally tagged, untagged). Вибір впливає на сумісність з іншими системами.

**Option і null**: None за замовчуванням = відсутнє поле, не null. Використовуйте атрибути для потрібної поведінки.

**Порядок полів**: HashMap не гарантує порядок. Використовуйте BTreeMap або IndexMap для детермінізму.

**Версіонування**: нові поля мають бути Option або мати default. Розгляньте явне версіонування для складних схем.

**Вибір формату**: JSON для API та читабельності, bincode/MessagePack для продуктивності та компактності.

---

## 🔗 Зв'язок з наступним матеріалом

Опанувавши підступності серіалізації, ви готові до вивчення проблем роботи з файлами та шляхами — cross-platform сумісність, кодування імен файлів, та несподівані відмінності між операційними системами.
