# Зміст

[Розум Рою](index.md)

[Передмова](./preface.md)
[Вступ](./introduction.md)

---

# Частина 0: Імперативний Rust (Bootcamp)

- [Розділ 1: Налаштування середовища та перший проєкт](./part0-bootcamp/ch01-setup.md)
- [Розділ 2: Змінні та базові типи даних](./part0-bootcamp/ch02-variables.md)
- [Розділ 3: Функції та параметри](./part0-bootcamp/ch03-functions.md)
- [Розділ 4: Керування потоком виконання](./part0-bootcamp/ch04-control-flow.md)
- [Розділ 5: Прості структури даних (Stack-only)](./part0-bootcamp/ch05-data-structures.md)
- [Розділ 6: Практикум — Агент-патрульний на сітці](./part0-bootcamp/ch06-practicum.md)

---

# Частина I: Агент як автономний об'єкт

- [Розділ 7: Ownership — Концепція володіння](./part1-agent/ch07-ownership-concept.md)
- [Розділ 8: Ownership — Практика помилок компілятора](./part1-agent/ch08-ownership-errors.md)
- [Розділ 9: Borrowing — Іммутабельні посилання](./part1-agent/ch09-borrowing-immutable.md)
- [Розділ 10: Borrowing — Мутабельні посилання та конфлікти](./part1-agent/ch10-borrowing-mutable.md)
- [Розділ 11: Struct — Моделювання стану агента](./part1-agent/ch11-structs.md)
- [Розділ 12: Enum та Pattern Matching — Машина станів агента](./part1-agent/ch12-enums.md)
- [Розділ 13: Модулі та видимість — Організація коду агента](./part1-agent/ch13-modules.md)
- [Розділ 14: Модульне тестування](./part1-agent/ch14-testing.md)
- [Розділ 15: Практикум — Автономний агент v1.0](./part1-agent/ch15-practicum-v1.md)

---

# Частина II: Робастність, дані та інструментарій

- [Розділ 16: Колекції — Vec та динамічні дані](./part2-robustness/ch16-vec.md)
- [Розділ 17: Колекції — HashMap та просторові структури](./part2-robustness/ch17-hashmap.md)
- [Розділ 18: Обробка помилок — Option та Result](./part2-robustness/ch18-option-result.md)
- [Розділ 19: Оператор ? та ланцюжки помилок](./part2-robustness/ch19-question-mark.md)
- [Розділ 20: Власні типи помилок та thiserror](./part2-robustness/ch20-custom-errors.md)
- [Розділ 21: Логування та tracing](./part2-robustness/ch21-tracing.md)
- [Розділ 22: Traits — Базові концепції](./part2-robustness/ch22-traits-basic.md)
- [Розділ 23: Traits — Власні абстракції](./part2-robustness/ch23-traits-custom.md)
- [Розділ 24: Generics — Параметричні типи](./part2-robustness/ch24-generics.md)
- [Розділ 25: Trait Bounds та Where-клаузи](./part2-robustness/ch25-trait-bounds.md)
- [Розділ 26: Серіалізація та serde](./part2-robustness/ch26-serde.md)
- [Розділ 27: Практикум — Робастний агент v2.0](./part2-robustness/ch27-practicum-v2.md)

---

# Частина III: Smart Pointers та спільна пам'ять

- [Розділ 28: Box<T> — Heap Allocation](./part3-smart-pointers/ch28-box.md)
- [Розділ 29: Rc<T> — Shared Ownership](./part3-smart-pointers/ch29-rc.md)
- [Розділ 30: RefCell<T> — Interior Mutability](./part3-smart-pointers/ch30-refcell.md)
- [Розділ 31: Cell<T> та альтернативи RefCell](./part3-smart-pointers/ch31-cell.md)
- [Розділ 32: Lifetimes — Коли компілятор не може вивести](./part3-smart-pointers/ch32-lifetimes.md)
- [Розділ 33: Threads — Базова багатопотоковість](./part3-smart-pointers/ch33-threads.md)
- [Розділ 34: Arc<T> та Mutex<T> — Thread-safe спільний стан](./part3-smart-pointers/ch34-arc-mutex.md)
- [Розділ 35: Channels — Message Passing](./part3-smart-pointers/ch35-channels.md)
- [Розділ 36: Практикум — Локальний рій агентів v3.0](./part3-smart-pointers/ch36-practicum-v3.md)

---

# Частина IV: Асинхронність та масштабування

- [Розділ 37: Async/Await — Концепції](./part4-async/ch37-async-await.md)
- [Розділ 38: Tokio — Async Runtime](./part4-async/ch38-tokio.md)
- [Розділ 39: Async Channels та синхронізація](./part4-async/ch39-async-channels.md)
- [Розділ 40: Streams — Async ітератори](./part4-async/ch40-streams.md)
- [Розділ 41: Actor Model — Концептуальні основи](./part4-async/ch41-actor-concepts.md)
- [Розділ 42: Actor Model — Реалізація на Tokio](./part4-async/ch42-actor-tokio.md)
- [Розділ 43: Rayon — Data Parallelism](./part4-async/ch43-rayon.md)
- [Розділ 44: Практикум — Асинхронний рій v4.0](./part4-async/ch44-practicum-v4.md)

---

# Частина V: Архітектури МАС та екосистема

- [Розділ 45: Макроси — Декларативні (macro_rules!)](./part5-architecture/ch45-macros-declarative.md)
- [Розділ 46: Макроси — Процедурні](./part5-architecture/ch46-macros-procedural.md)
- [Розділ 47: Unsafe Rust — Межі безпеки](./part5-architecture/ch47-unsafe.md)
- [Розділ 48: Entity-Component-System (ECS)](./part5-architecture/ch48-ecs.md)
- [Розділ 49: BDI-агенти — Beliefs-Desires-Intentions](./part5-architecture/ch49-bdi.md)
- [Розділ 50: Мережева комунікація агентів](./part5-architecture/ch50-network.md)
- [Розділ 51: WebAssembly — Агенти у браузері](./part5-architecture/ch51-wasm.md)
- [Розділ 52: Фінальний проєкт — Інтелектуальний автономний рій БПЛА](./part5-architecture/ch52-final-project.md)

---

[Післямова](./afterword.md)

---

# Додатки

- [Додаток C: Міграція з C на Rust](./appendices/appendix-c-migration.md)
