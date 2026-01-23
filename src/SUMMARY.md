# Зміст

[Структурно-логічна схема вивчення дисципліни «Програмування мультиагентних систем»](./Structure.md)
[Розум Рою](./index.md)
[Передмова](./preface.md)
[Вступ](./introduction.md)

---

# Частина 0: Імперативний Rust (Bootcamp)

- [Розділ 1: Налаштування середовища та перший проєкт](./part0-bootcamp/ch01-setup.md)
- [Розділ 2: Змінні та базові типи даних](./part0-bootcamp/ch02-variables.md)
- [Розділ 3: Функції та параметри](./part0-bootcamp/ch03-functions.md)
- [Розділ 4: Керування потоком виконання](./part0-bootcamp/ch04-control-flow.md)
- [Розділ 5: Прості структури даних (Stack-only)](./part0-bootcamp/ch05-data-structures.md)
- [Розділ 6: Область видимості змінних у Rust](./part0-bootcamp/ch06-scope.md)
- [Розділ 7: Практикум — Агент-патрульний на сітці](./part0-bootcamp/ch07-practicum.md)
- [Розділ 8: Підступні задачі з дійсними числами](./part0-bootcamp/ch08-real-error.md)
- [Розділ 9: Підступне виконання умовних операторів](./part0-bootcamp/ch09-conditional-operators.md)
- [Розділ 10: Підступні задачі з операторами циклу](./part0-bootcamp/ch10_loops.md)
- [Розділ 11: Підступні задачі з функціями](./part0-bootcamp/ch11_functions_pitfalls.md)
- [Розділ 12: Підступні задачі з цілими числами](./part0-bootcamp/ch12_integer_pitfalls.md)
- [Розділ 13: Підступні задачі з рядками та текстом](./part0-bootcamp/ch13_strings_pitfalls.md)
- [Розділ 14: Підступні задачі з колекціями](./part0-bootcamp/ch14_collections_pitfalls.md)
- [Розділ 15: Підступні задачі з часом та датами](./part0-bootcamp/ch15_datetime_pitfalls.md)
- [Розділ 16: Підступні задачі з обробкою помилок](./part0-bootcamp/ch16_error_handling_pitfalls.md)
- [Розділ 17: Підступні задачі з паралелізмом](./part0-bootcamp/ch17_concurrency_pitfalls.md)
- [Розділ 18: Робота з файлами та шляхами](./part0-bootcamp/ch18_files_paths_pitfalls.md)
- [Розділ 19: Підступні задачі з серіалізацією](./part0-bootcamp/ch19_serialization_pitfalls.md)
- [Розділ 20: Підступні задачі з пам'яттю та ресурсами](./part0-bootcamp/ch20_memory_pitfalls.md)



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

# Частина VI: Практичний курс

- [Розділ 53: Практичне заняття 1: Перші кроки в Rust](./practical/practical_01_first_steps.md)
- [Розділ 54: Практичне заняття 2: Змінні та типи даних](./practical/practical_02_variables.md)
- [Розділ 55: Практичне заняття 3: Функції](./practical/practical_03_functions.md)
- [Розділ 56: Практичне заняття 4: Умовні оператори](./practical/practical_04_conditionals.md)
- [Розділ 57: Практичне заняття 5: Цикли](./practical/practical_05_loops.md)
- [Розділ 58: Практичне заняття 6: Кортежі та масиви](./practical/practical_06_tuples_arrays.md)
- [Розділ 59: Практичне заняття 7: Ownership та Borrowing](./practical/practical_07_ownership.md)
- [Розділ 60: Практичне заняття 8: Структури та Enum](./practical/practical_08_structs_enums.md)
- [Розділ 61: Практичне заняття 9: Колекції (Vec, HashMap, String)](./practical/practical_09_collections.md)
- [Розділ 62: Практичне заняття 10: Обробка помилок (Result, оператор ?)](./practical/practical_10_error_handling.md)
- [Розділ 63: Практичне заняття 11: Traits та Generics](./practical/practical_11_traits_generics.md)
- [Розділ 64: Практичне заняття 12: Lifetimes (час життя посилань)](./practical/practical_12_lifetimes.md)
- [Розділ 65: Практичне заняття 13: Багатопотоковість (Threads, Arc, Mutex)](./practical/practical_13_threads.md)
- [Розділ 66: Практичне заняття 14: Async/Await та Tokio](./practical/practical_14_async.md)



# Додатки

- [Післямова](./afterword.md)
- [Додаток C: Міграція з C на Rust](./appendices/appendix-c-migration.md)
