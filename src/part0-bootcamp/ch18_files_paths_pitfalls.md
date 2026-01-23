# Робота з файлами та шляхами

---

## 📋 Анотація

Цей розділ охоплює повний спектр роботи з файлами в Rust — від базових операцій читання та запису до підступних кросплатформних відмінностей. Ви дізнаєтесь, як правильно відкривати, читати та записувати файли, чому шлях, що працює на Linux, може не працювати на Windows, та як уникнути втрати даних через некоректну обробку кодувань. У контексті рою БПЛА файлова система використовується для логування телеметрії, збереження конфігурацій місій, кешування карт — і помилка може призвести до втрати критичних даних польоту.

---

## 🎯 Цілі навчання

Після завершення цього розділу ви зможете:

1. **Читати та записувати** файли різними способами
2. **Використовувати** буферизований I/O для ефективності
3. **Працювати** з шляхами через Path та PathBuf
4. **Розуміти** кросплатформні відмінності файлових систем
5. **Обробляти** імена файлів з нестандартним кодуванням
6. **Уникати** типових помилок при роботі з файлами
7. **Забезпечувати** атомарність критичних операцій

---

## 📚 Ключові терміни

| Термін | Визначення |
|--------|------------|
| **File** | Структура для роботи з відкритим файлом |
| **Path** | Незмінний слайс шляху (аналог &str) |
| **PathBuf** | Володіючий буфер шляху (аналог String) |
| **BufReader/BufWriter** | Обгортки для буферизованого I/O |
| **OsString/OsStr** | Рядки у форматі операційної системи |
| **file descriptor** | Числовий ідентифікатор відкритого файлу в ОС |

---

## 💡 Мотиваційний кейс: Зниклі логи польоту

Команда розробляла систему логування для дронів. Код працював ідеально на машинах розробників (macOS та Linux). Але коли систему розгорнули на наземній станції з Windows, почалися проблеми.

Спочатку система не могла створити директорію для логів:
```rust
let log_path = format!("logs/{}/flight.log", mission_id);
std::fs::create_dir_all(&log_path)?;  // Помилка на Windows!
```

Причина: `mission_id` містив символ `:` (формат часу `14:30:00`), який заборонений у шляхах Windows.

Після виправлення виникла нова проблема: логи існували, але були порожні. Аналіз показав:
```rust
fn log_telemetry(file: &mut File, data: &str) {
    write!(file, "{}\n", data).unwrap();
    // Дані в буфері, але не на диску!
}
```

При аварійному завершенні (crash, втрата живлення) буферизовані дані втрачались. На Linux це траплялось рідше через агресивніший flush, на Windows — постійно.

Фінальна проблема: деякі логи містили "кракозябри" замість кирилиці у назвах точок маршруту. Причина — змішування UTF-8 та локального кодування Windows (CP1251).

Три різні проблеми, одна причина: недостатнє розуміння відмінностей файлових систем.

---

# ЧАСТИНА 1: ОСНОВИ РОБОТИ З ФАЙЛАМИ

---

## ТЕОРІЯ: ВІДКРИТТЯ ТА СТВОРЕННЯ ФАЙЛІВ

### Структура File

`std::fs::File` представляє відкритий файл. Це обгортка над file descriptor операційної системи:

```rust
use std::fs::File;

// Відкрити існуючий файл для читання
let file = File::open("data.txt")?;

// Створити новий файл для запису (перезапише існуючий!)
let file = File::create("output.txt")?;
```

Важливо розуміти: `File::open` відкриває лише для читання, `File::create` лише для запису (і створює файл, якщо не існує).

### Що повертають ці функції

Обидві функції повертають `Result<File, std::io::Error>`:

```rust
use std::fs::File;
use std::io;

fn open_config() -> io::Result<File> {
    let file = File::open("config.toml")?;
    Ok(file)
}

// Можливі помилки:
// - NotFound: файл не існує
// - PermissionDenied: немає прав доступу
// - AlreadyExists: (для create з певними опціями)
// - та інші...
```

### OpenOptions — повний контроль

Для складніших сценаріїв використовуйте `OpenOptions`:

```rust
use std::fs::OpenOptions;

// Відкрити для читання і запису
let file = OpenOptions::new()
    .read(true)
    .write(true)
    .open("data.txt")?;

// Відкрити для дописування (append)
let file = OpenOptions::new()
    .append(true)
    .open("log.txt")?;

// Створити, лише якщо не існує
let file = OpenOptions::new()
    .write(true)
    .create_new(true)  // Помилка, якщо файл існує
    .open("unique.txt")?;

// Створити або відкрити існуючий, не обрізаючи
let file = OpenOptions::new()
    .write(true)
    .create(true)      // Створити, якщо не існує
    .truncate(false)   // Не обрізати існуючий
    .open("data.txt")?;
```

### Таблиця режимів OpenOptions

| Метод | Опис |
|-------|------|
| `.read(true)` | Дозволити читання |
| `.write(true)` | Дозволити запис |
| `.append(true)` | Запис у кінець файлу |
| `.truncate(true)` | Обрізати файл до нуля при відкритті |
| `.create(true)` | Створити, якщо не існує |
| `.create_new(true)` | Створити лише якщо не існує (помилка інакше) |

### RAII: автоматичне закриття

File реалізує Drop — файл автоматично закривається при виході зі scope:

```rust
fn process_file() -> io::Result<()> {
    let file = File::open("data.txt")?;
    // ... робота з файлом ...
    Ok(())
}  // file виходить зі scope → Drop → файл закривається
```

Не потрібно викликати `close()` вручну. Але якщо потрібно обробити помилку закриття:

```rust
use std::fs::File;
use std::io::Write;

fn write_important_data() -> io::Result<()> {
    let mut file = File::create("important.txt")?;
    write!(file, "critical data")?;
    
    // Явний sync для гарантії запису на диск
    file.sync_all()?;
    
    Ok(())
}
```

---

## ТЕОРІЯ: ЧИТАННЯ ФАЙЛІВ

### Найпростіший спосіб: read_to_string

Для невеликих текстових файлів:

```rust
use std::fs;

let content = fs::read_to_string("config.toml")?;
println!("Config:\n{}", content);
```

Це читає весь файл у пам'ять як String. Простий, але має обмеження:
- Весь файл має поміститись у пам'ять
- Файл має бути валідним UTF-8

### Читання бінарних даних: read

```rust
use std::fs;

let bytes: Vec<u8> = fs::read("image.png")?;
println!("File size: {} bytes", bytes.len());
```

### Читання з File напряму

```rust
use std::fs::File;
use std::io::Read;

let mut file = File::open("data.txt")?;
let mut content = String::new();
file.read_to_string(&mut content)?;
```

Або читання у буфер:

```rust
use std::fs::File;
use std::io::Read;

let mut file = File::open("data.bin")?;
let mut buffer = [0u8; 1024];
let bytes_read = file.read(&mut buffer)?;
println!("Read {} bytes", bytes_read);
```

### Читання по рядках: BufReader

Для великих файлів читайте по рядках:

```rust
use std::fs::File;
use std::io::{BufRead, BufReader};

let file = File::open("large_log.txt")?;
let reader = BufReader::new(file);

for line in reader.lines() {
    let line = line?;  // lines() повертає Result для кожного рядка
    println!("{}", line);
}
```

Це ефективно використовує пам'ять — у кожен момент у пам'яті лише поточний рядок.

### BufReader: чому він важливий

Без буферизації кожен виклик `read` — це системний виклик, що повільно:

```rust
// Повільно: багато системних викликів
let mut file = File::open("data.txt")?;
let mut byte = [0u8; 1];
while file.read(&mut byte)? > 0 {
    process_byte(byte[0]);
}

// Швидко: читання блоками
let file = File::open("data.txt")?;
let mut reader = BufReader::new(file);  // Буфер 8KB за замовчуванням
let mut byte = [0u8; 1];
while reader.read(&mut byte)? > 0 {
    process_byte(byte[0]);
}
```

BufReader читає великі блоки з диску і віддає їх по частинах. Різниця в продуктивності може бути 100x.

### Власний розмір буфера

```rust
let file = File::open("huge_file.dat")?;
let reader = BufReader::with_capacity(64 * 1024, file);  // 64KB буфер
```

---

## ТЕОРІЯ: ЗАПИС У ФАЙЛИ

### Найпростіший спосіб: fs::write

```rust
use std::fs;

fs::write("output.txt", "Hello, World!")?;

// Або бінарні дані
let data = vec![0u8, 1, 2, 3, 4];
fs::write("output.bin", &data)?;
```

`fs::write` створює файл, якщо не існує, і перезаписує існуючий.

### Запис через File

```rust
use std::fs::File;
use std::io::Write;

let mut file = File::create("output.txt")?;
write!(file, "Hello, {}!\n", "World")?;
writeln!(file, "Line 2")?;
file.write_all(b"Raw bytes")?;
```

Різниця між методами:
- `write!` / `writeln!` — форматований вивід (як println!)
- `write_all` — записує всі байти або повертає помилку
- `write` — може записати частину (повертає кількість записаних байт)

### BufWriter для ефективності

```rust
use std::fs::File;
use std::io::{BufWriter, Write};

let file = File::create("output.txt")?;
let mut writer = BufWriter::new(file);

for i in 0..10000 {
    writeln!(writer, "Line {}", i)?;
}

// Важливо: flush для гарантії запису
writer.flush()?;
```

Без BufWriter кожен writeln! — це системний виклик. З BufWriter дані накопичуються в пам'яті та записуються блоками.

### Гарантія запису на диск: sync

Навіть після `flush()` дані можуть бути в кеші операційної системи. Для критичних даних:

```rust
let mut file = File::create("critical.dat")?;
file.write_all(b"important data")?;

// flush() — з буфера Rust у кеш ОС
// sync_all() — з кешу ОС на фізичний диск
file.sync_all()?;
```

`sync_all()` повільний, використовуйте лише для критичних даних.

---

## ТЕОРІЯ: ДОПИСУВАННЯ ДО ФАЙЛУ (APPEND)

### Відкриття для append

```rust
use std::fs::OpenOptions;
use std::io::Write;

let mut file = OpenOptions::new()
    .append(true)
    .create(true)  // Створити, якщо не існує
    .open("log.txt")?;

writeln!(file, "New log entry")?;
```

Режим append гарантує, що кожен запис додається в кінець файлу, навіть якщо кілька процесів пишуть одночасно (на рівні ОС).

### Різниця між append та write + seek

```rust
// Append: атомарний запис у кінець
let mut file = OpenOptions::new().append(true).open("log.txt")?;

// Write + seek: НЕ атомарний!
let mut file = OpenOptions::new().write(true).open("log.txt")?;
file.seek(SeekFrom::End(0))?;  // Інший процес може записати тут!
file.write_all(b"data")?;      // Можемо перезаписати чужі дані
```

---

## ТЕОРІЯ: ПОЗИЦІОНУВАННЯ У ФАЙЛІ (SEEK)

### Трейт Seek

```rust
use std::fs::File;
use std::io::{Read, Seek, SeekFrom};

let mut file = File::open("data.bin")?;

// Перейти на позицію 100 від початку
file.seek(SeekFrom::Start(100))?;

// Перейти на 50 байт вперед від поточної позиції
file.seek(SeekFrom::Current(50))?;

// Перейти на 10 байт до кінця
file.seek(SeekFrom::End(-10))?;

// Отримати поточну позицію
let position = file.stream_position()?;
```

### Визначення розміру файлу

```rust
use std::fs::File;
use std::io::{Seek, SeekFrom};

let mut file = File::open("data.bin")?;
let size = file.seek(SeekFrom::End(0))?;
file.seek(SeekFrom::Start(0))?;  // Повернутись на початок

println!("File size: {} bytes", size);
```

Або простіше через метадані:

```rust
use std::fs;

let metadata = fs::metadata("data.bin")?;
println!("File size: {} bytes", metadata.len());
```

---

## ТЕОРІЯ: РОБОТА З ДИРЕКТОРІЯМИ

### Створення директорій

```rust
use std::fs;

// Створити одну директорію
fs::create_dir("new_folder")?;

// Створити всю ієрархію
fs::create_dir_all("path/to/nested/folder")?;
```

`create_dir_all` не повертає помилку, якщо директорія вже існує.

### Читання вмісту директорії

```rust
use std::fs;

for entry in fs::read_dir(".")? {
    let entry = entry?;
    let path = entry.path();
    
    if path.is_dir() {
        println!("DIR:  {}", path.display());
    } else {
        println!("FILE: {}", path.display());
    }
}
```

### Рекурсивний обхід (walkdir crate)

Стандартна бібліотека не має рекурсивного обходу. Використовуйте `walkdir`:

```rust
use walkdir::WalkDir;

for entry in WalkDir::new("src") {
    let entry = entry?;
    println!("{}", entry.path().display());
}
```

### Видалення

```rust
use std::fs;

// Видалити файл
fs::remove_file("file.txt")?;

// Видалити порожню директорію
fs::remove_dir("empty_folder")?;

// Видалити директорію з вмістом (ОБЕРЕЖНО!)
fs::remove_dir_all("folder_with_contents")?;
```

### Копіювання та переміщення

```rust
use std::fs;

// Копіювати файл
fs::copy("source.txt", "dest.txt")?;

// Перемістити/перейменувати
fs::rename("old_name.txt", "new_name.txt")?;
```

---

## ТЕОРІЯ: МЕТАДАНІ ФАЙЛІВ

### Отримання метаданих

```rust
use std::fs;

let metadata = fs::metadata("file.txt")?;

println!("Size: {} bytes", metadata.len());
println!("Is file: {}", metadata.is_file());
println!("Is dir: {}", metadata.is_dir());
println!("Is symlink: {}", metadata.is_symlink());
println!("Readonly: {}", metadata.permissions().readonly());
```

### Час модифікації

```rust
use std::fs;
use std::time::SystemTime;

let metadata = fs::metadata("file.txt")?;

if let Ok(modified) = metadata.modified() {
    let age = SystemTime::now().duration_since(modified)?;
    println!("Modified {} seconds ago", age.as_secs());
}
```

### Зміна прав доступу (Unix)

```rust
use std::fs;
use std::os::unix::fs::PermissionsExt;

let mut perms = fs::metadata("script.sh")?.permissions();
perms.set_mode(0o755);  // rwxr-xr-x
fs::set_permissions("script.sh", perms)?;
```

---

## ТЕОРІЯ: ТИМЧАСОВІ ФАЙЛИ

### Крейт tempfile

```rust
use tempfile::{tempfile, NamedTempFile, tempdir};
use std::io::Write;

// Анонімний тимчасовий файл (автоматично видаляється)
let mut file = tempfile()?;
write!(file, "temporary data")?;
// Файл видаляється при drop

// Іменований тимчасовий файл
let mut file = NamedTempFile::new()?;
println!("Temp path: {}", file.path().display());
write!(file, "data")?;
// Файл видаляється при drop

// Зберегти тимчасовий файл
let file = NamedTempFile::new()?;
let path = file.into_temp_path();
path.persist("permanent.txt")?;  // Перетворити на постійний

// Тимчасова директорія
let dir = tempdir()?;
let file_path = dir.path().join("data.txt");
fs::write(&file_path, "content")?;
// Директорія і вміст видаляються при drop
```

---

# ЧАСТИНА 2: ШЛЯХИ В RUST

---

## ТЕОРІЯ: PATH ТА PATHBUF

### Аналогія з рядками

| Володіючий | Слайс | Призначення |
|------------|-------|-------------|
| `String` | `&str` | UTF-8 текст |
| `PathBuf` | `&Path` | Шлях файлової системи |
| `OsString` | `&OsStr` | Рядок ОС |

### Створення шляхів

```rust
use std::path::{Path, PathBuf};

// Path — слайс, не володіє даними
let path: &Path = Path::new("/home/user/file.txt");

// PathBuf — володіє даними
let path_buf: PathBuf = PathBuf::from("/home/user/file.txt");

// Конвертація
let path: &Path = path_buf.as_path();
let path_buf: PathBuf = path.to_path_buf();
```

### Побудова шляхів

```rust
use std::path::PathBuf;

let mut path = PathBuf::from("/home/user");
path.push("documents");        // /home/user/documents
path.push("report.txt");       // /home/user/documents/report.txt

// Або через join
let path = PathBuf::from("/home/user")
    .join("documents")
    .join("report.txt");

// join повертає новий PathBuf, не змінює оригінал
let base = Path::new("/home/user");
let full = base.join("file.txt");  // base не змінюється
```

### Компоненти шляху

```rust
use std::path::Path;

let path = Path::new("/home/user/documents/report.txt");

// Батьківська директорія
println!("Parent: {:?}", path.parent());  // Some("/home/user/documents")

// Ім'я файлу
println!("File name: {:?}", path.file_name());  // Some("report.txt")

// Основа імені (без розширення)
println!("Stem: {:?}", path.file_stem());  // Some("report")

// Розширення
println!("Extension: {:?}", path.extension());  // Some("txt")

// Ітерація по компонентах
for component in path.components() {
    println!("{:?}", component);
}
// RootDir, Normal("home"), Normal("user"), Normal("documents"), Normal("report.txt")
```

### Модифікація шляху

```rust
use std::path::PathBuf;

let mut path = PathBuf::from("/home/user/report.txt");

// Змінити розширення
path.set_extension("pdf");  // /home/user/report.pdf

// Змінити ім'я файлу
path.set_file_name("document.docx");  // /home/user/document.docx

// Видалити останній компонент
path.pop();  // /home/user
```

### Перевірки шляху

```rust
use std::path::Path;

let path = Path::new("/home/user/file.txt");

// Чи існує
if path.exists() {
    println!("File exists");
}

// Тип
if path.is_file() {
    println!("It's a file");
}
if path.is_dir() {
    println!("It's a directory");
}
if path.is_symlink() {
    println!("It's a symbolic link");
}

// Абсолютний чи відносний
if path.is_absolute() {
    println!("Absolute path");
}
if path.is_relative() {
    println!("Relative path");
}
```

### Канонізація шляху

```rust
use std::path::Path;

let path = Path::new("./documents/../file.txt");

// Канонічний шлях (розв'язує .., symlinks)
let canonical = path.canonicalize()?;
// Наприклад: /home/user/file.txt

// УВАГА: canonicalize вимагає існування файлу!
```

---

# ЧАСТИНА 3: ПІДСТУПНІ ЗАДАЧІ

---

## ТЕОРІЯ: КРОСПЛАТФОРМНІ ВІДМІННОСТІ

### Розділювачі шляхів

```rust
// НЕ РОБІТЬ ТАК!
let path = "logs/2024/january/data.txt";  // Не працює на Windows

// Правильно: використовуйте Path
use std::path::PathBuf;
let path = PathBuf::from("logs")
    .join("2024")
    .join("january")
    .join("data.txt");
```

| Система | Розділювач | Приклад |
|---------|------------|---------|
| Unix/Linux/macOS | `/` | `/home/user/file.txt` |
| Windows | `\` | `C:\Users\user\file.txt` |

Rust автоматично використовує правильний розділювач через `Path::join()`.

### Заборонені символи в іменах файлів

**Windows забороняє:**
- `< > : " / \ | ? *`
- Імена: `CON, PRN, AUX, NUL, COM1-COM9, LPT1-LPT9`
- Пробіл або крапка в кінці імені

**Unix забороняє:**
- `/` (розділювач)
- `\0` (null terminator)

```rust
fn sanitize_filename(name: &str) -> String {
    let forbidden = ['<', '>', ':', '"', '/', '\\', '|', '?', '*'];
    let mut result: String = name
        .chars()
        .map(|c| if forbidden.contains(&c) { '_' } else { c })
        .collect();
    
    // Видалити пробіли та крапки в кінці (Windows)
    while result.ends_with(' ') || result.ends_with('.') {
        result.pop();
    }
    
    // Перевірити зарезервовані імена Windows
    let reserved = ["CON", "PRN", "AUX", "NUL", 
                    "COM1", "COM2", "COM3", "COM4", "COM5", 
                    "COM6", "COM7", "COM8", "COM9",
                    "LPT1", "LPT2", "LPT3", "LPT4", "LPT5", 
                    "LPT6", "LPT7", "LPT8", "LPT9"];
    
    let upper = result.to_uppercase();
    let stem = upper.split('.').next().unwrap_or("");
    if reserved.contains(&stem) {
        result = format!("_{}", result);
    }
    
    result
}
```

### Чутливість до регістру

**Unix**: регістр важливий. `File.txt` і `file.txt` — різні файли.

**Windows/macOS**: регістр зберігається, але ігнорується при порівнянні.

```rust
use std::path::Path;

// На Unix
Path::new("File.txt").exists();  // true
Path::new("file.txt").exists();  // false (інший файл!)

// На Windows/macOS
Path::new("File.txt").exists();  // true
Path::new("file.txt").exists();  // true (той самий файл!)
```

Пастка: тести проходять на macOS розробника, падають на Linux CI.

### Максимальна довжина шляху

| Система | Обмеження |
|---------|-----------|
| Windows | 260 символів (можна збільшити через `\\?\`) |
| Linux | 4096 символів (PATH_MAX) |
| macOS | 1024 символи |

```rust
// Windows: довгі шляхи через prefix
#[cfg(windows)]
fn long_path(path: &Path) -> PathBuf {
    let mut long = PathBuf::from(r"\\?\");
    long.push(path.canonicalize().unwrap_or_else(|_| path.to_path_buf()));
    long
}
```

---

## ТЕОРІЯ: КОДУВАННЯ ІМЕН ФАЙЛІВ

### Проблема: імена файлів — не UTF-8

Rust рядки (`String`, `&str`) — завжди валідний UTF-8. Але імена файлів у різних системах:

**Linux**: будь-яка послідовність байтів, крім `/` і `\0`. UTF-8 не гарантується.

**macOS**: UTF-8, нормалізований у формі NFD.

**Windows**: UTF-16, деякі символи неможливо представити в UTF-8.

### OsString та OsStr

Rust має спеціальні типи для "рядків ОС":

```rust
use std::ffi::{OsString, OsStr};
use std::path::Path;

let path = Path::new("файл.txt");

// file_name() повертає Option<&OsStr>, не Option<&str>!
let name: Option<&OsStr> = path.file_name();

// Конвертація в &str може провалитись
if let Some(name) = name {
    match name.to_str() {
        Some(s) => println!("UTF-8 name: {}", s),
        None => println!("Non-UTF-8 name: {:?}", name),
    }
}
```

### Lossy конвертація

```rust
use std::path::Path;

let path = Path::new("файл.txt");

// to_string_lossy() завжди успішна, але може замінити символи
let name = path.file_name().unwrap().to_string_lossy();
println!("{}", name);  // Невалідні UTF-8 байти стануть �
```

### Створення файлів з не-UTF-8 іменами

```rust
use std::ffi::OsStr;
use std::os::unix::ffi::OsStrExt;
use std::fs::File;
use std::path::Path;

#[cfg(unix)]
fn create_weird_file() -> std::io::Result<()> {
    // Ім'я з невалідним UTF-8
    let invalid_utf8 = [0x66, 0x69, 0x6c, 0x65, 0xff, 0x2e, 0x74, 0x78, 0x74];
    let name = OsStr::from_bytes(&invalid_utf8);
    let path = Path::new(name);
    
    File::create(path)?;
    Ok(())
}
```

### Пастка: display() vs to_string_lossy()

```rust
let path = Path::new("/some/path");

// display() — для показу користувачу (може мати placeholder для невалідних)
println!("Path: {}", path.display());

// to_string_lossy() — отримати Cow<str>
let s: std::borrow::Cow<str> = path.to_string_lossy();
```

---

## ТЕОРІЯ: СИМВОЛІЧНІ ПОСИЛАННЯ

### Читання symlink

```rust
use std::fs;
use std::path::Path;

let link = Path::new("link_to_file");

// Куди вказує посилання
let target = fs::read_link(link)?;
println!("Link points to: {}", target.display());

// metadata() слідує за symlink
let meta = fs::metadata(link)?;  // Метадані цільового файлу

// symlink_metadata() не слідує
let link_meta = fs::symlink_metadata(link)?;  // Метадані самого посилання
```

### Створення symlink

```rust
use std::os::unix::fs::symlink;

#[cfg(unix)]
fn create_link() -> std::io::Result<()> {
    symlink("/path/to/target", "/path/to/link")?;
    Ok(())
}

#[cfg(windows)]
fn create_link() -> std::io::Result<()> {
    use std::os::windows::fs::{symlink_file, symlink_dir};
    symlink_file("target.txt", "link.txt")?;
    // Windows розрізняє symlinks на файли та директорії
    Ok(())
}
```

### Пастка: нескінченні цикли

```rust
// Якщо a → b → a (цикл)
let canonical = Path::new("a").canonicalize();  // Помилка!
```

При рекурсивному обході враховуйте можливість циклів через symlinks.

---

## ТЕОРІЯ: АТОМАРНІСТЬ ОПЕРАЦІЙ

### Проблема: операції не атомарні

```rust
// Небезпечно: перевірка і створення — окремі операції
if !path.exists() {
    // Інший процес може створити файл тут!
    File::create(path)?;
}
```

### Рішення: create_new

```rust
use std::fs::OpenOptions;

let file = OpenOptions::new()
    .write(true)
    .create_new(true)  // Атомарно: створити лише якщо не існує
    .open(path)?;
```

### Атомарний запис через rename

```rust
use std::fs::{self, File};
use std::io::Write;

fn atomic_write(path: &Path, content: &[u8]) -> std::io::Result<()> {
    // Записати у тимчасовий файл
    let tmp_path = path.with_extension("tmp");
    let mut file = File::create(&tmp_path)?;
    file.write_all(content)?;
    file.sync_all()?;  // Гарантувати запис на диск
    
    // Атомарно перейменувати
    fs::rename(&tmp_path, path)?;
    
    Ok(())
}
```

На більшості файлових систем `rename` атомарний, якщо source і dest на одному розділі.

---

## ТЕОРІЯ: БУФЕРИЗАЦІЯ ТА ВТРАТА ДАНИХ

### Проблема: дані в буфері

```rust
use std::fs::File;
use std::io::{BufWriter, Write};

let file = File::create("data.txt")?;
let mut writer = BufWriter::new(file);

write!(writer, "important data")?;
// Дані в буфері BufWriter!

// Якщо програма crash тут — дані втрачені!
```

### Рішення: явний flush

```rust
let file = File::create("data.txt")?;
let mut writer = BufWriter::new(file);

write!(writer, "important data")?;
writer.flush()?;  // Примусово записати буфер
```

### Drop викликає flush, але ігнорує помилки

```rust
impl<W: Write> Drop for BufWriter<W> {
    fn drop(&mut self) {
        // flush викликається, але помилка ігнорується!
        let _ = self.flush();
    }
}
```

Тому для критичних даних завжди викликайте flush() явно.

### sync_all vs sync_data

```rust
let mut file = File::create("data.txt")?;
file.write_all(b"data")?;

// sync_data: записати дані на диск
file.sync_data()?;

// sync_all: записати дані І метадані (розмір, час модифікації)
file.sync_all()?;
```

`sync_data` швидший, але `sync_all` безпечніший для критичних даних.

---

## ТЕОРІЯ: ПРАВА ДОСТУПУ

### Перевірка прав

```rust
use std::fs;

let metadata = fs::metadata("file.txt")?;
let permissions = metadata.permissions();

if permissions.readonly() {
    println!("File is read-only");
}
```

### Зміна прав (Unix)

```rust
use std::fs;
use std::os::unix::fs::PermissionsExt;

// Зробити виконуваним
let mut perms = fs::metadata("script.sh")?.permissions();
perms.set_mode(0o755);
fs::set_permissions("script.sh", perms)?;

// Зробити read-only
let mut perms = fs::metadata("data.txt")?.permissions();
perms.set_readonly(true);
fs::set_permissions("data.txt", perms)?;
```

### Пастка: перевірка прав ≠ гарантія успіху

```rust
// Погано: TOCTOU (Time Of Check To Time Of Use)
if metadata.permissions().readonly() == false {
    // Права можуть змінитись тут!
    file.write_all(b"data")?;
}

// Правильно: просто спробуйте і обробіть помилку
match file.write_all(b"data") {
    Ok(_) => println!("Written"),
    Err(e) if e.kind() == io::ErrorKind::PermissionDenied => {
        println!("No permission");
    }
    Err(e) => return Err(e),
}
```

---

## ТЕОРІЯ: ВЕЛИКІ ФАЙЛИ

### Проблема: читання всього файлу

```rust
// Небезпечно для великих файлів!
let content = fs::read_to_string("huge_file.txt")?;  // 10GB у пам'яті!
```

### Рішення: потокова обробка

```rust
use std::fs::File;
use std::io::{BufRead, BufReader};

let file = File::open("huge_file.txt")?;
let reader = BufReader::new(file);

for line in reader.lines() {
    let line = line?;
    process_line(&line);  // По одному рядку в пам'яті
}
```

### Memory-mapped files

Для великих файлів з випадковим доступом:

```rust
use memmap2::Mmap;
use std::fs::File;

let file = File::open("huge_file.bin")?;
let mmap = unsafe { Mmap::map(&file)? };

// Доступ до даних як до слайсу
let byte = mmap[1000000];  // ОС завантажить потрібну сторінку
```

---

## ПРАКТИЧНІ РЕКОМЕНДАЦІЇ

### Правило 1: Завжди використовуйте Path для шляхів

```rust
// Погано
let path = "logs/" + &date + "/data.txt";

// Добре
let path = PathBuf::from("logs").join(&date).join("data.txt");
```

### Правило 2: Обробляйте OsStr правильно

```rust
// Погано: може панікувати
let name = path.file_name().unwrap().to_str().unwrap();

// Добре: обробляємо не-UTF-8
let name = path.file_name()
    .map(|n| n.to_string_lossy())
    .unwrap_or_default();
```

### Правило 3: flush() для критичних даних

```rust
let mut writer = BufWriter::new(file);
// ... запис ...
writer.flush()?;  // Обов'язково!
```

### Правило 4: Атомарний запис через rename

```rust
fn safe_write(path: &Path, content: &[u8]) -> io::Result<()> {
    let tmp = path.with_extension("tmp");
    fs::write(&tmp, content)?;
    fs::rename(&tmp, path)?;
    Ok(())
}
```

### Правило 5: Санітизуйте імена файлів від користувача

```rust
let user_filename = get_user_input();
let safe_filename = sanitize_filename(&user_filename);
let path = base_dir.join(safe_filename);
```

### Правило 6: Потокова обробка великих файлів

```rust
// Погано
let data = fs::read("huge.bin")?;

// Добре
let file = File::open("huge.bin")?;
let reader = BufReader::new(file);
```

---

## Застосування до рою БПЛА

### Безпечне логування телеметрії

```rust
use std::fs::{File, OpenOptions};
use std::io::{BufWriter, Write};
use std::path::{Path, PathBuf};

pub struct FlightLogger {
    writer: BufWriter<File>,
    path: PathBuf,
    entries_since_flush: usize,
}

impl FlightLogger {
    pub fn new(mission_id: &str, drone_id: u32) -> std::io::Result<Self> {
        // Санітизуємо mission_id (може містити timestamp з ":")
        let safe_mission = sanitize_filename(mission_id);
        
        let dir = PathBuf::from("logs")
            .join(&safe_mission)
            .join(format!("drone_{}", drone_id));
        
        std::fs::create_dir_all(&dir)?;
        
        let path = dir.join("telemetry.csv");
        
        let file = OpenOptions::new()
            .create(true)
            .append(true)
            .open(&path)?;
        
        let mut writer = BufWriter::new(file);
        
        // Заголовок CSV, якщо файл порожній
        if path.metadata()?.len() == 0 {
            writeln!(writer, "timestamp,lat,lon,alt,battery,status")?;
        }
        
        Ok(Self {
            writer,
            path,
            entries_since_flush: 0,
        })
    }
    
    pub fn log(&mut self, telemetry: &Telemetry) -> std::io::Result<()> {
        writeln!(
            self.writer,
            "{},{},{},{},{},{}",
            telemetry.timestamp,
            telemetry.position.latitude,
            telemetry.position.longitude,
            telemetry.position.altitude,
            telemetry.battery_percent,
            telemetry.status,
        )?;
        
        self.entries_since_flush += 1;
        
        // Flush кожні 10 записів для балансу надійності та продуктивності
        if self.entries_since_flush >= 10 {
            self.writer.flush()?;
            self.entries_since_flush = 0;
        }
        
        Ok(())
    }
    
    pub fn close(mut self) -> std::io::Result<()> {
        self.writer.flush()?;
        // sync для гарантії запису
        self.writer.get_ref().sync_all()?;
        Ok(())
    }
}

fn sanitize_filename(name: &str) -> String {
    name.chars()
        .map(|c| match c {
            '<' | '>' | ':' | '"' | '/' | '\\' | '|' | '?' | '*' => '_',
            c if c.is_control() => '_',
            c => c,
        })
        .collect()
}
```

### Атомарне збереження конфігурації місії

```rust
use std::fs;
use std::path::Path;

pub fn save_mission_config(path: &Path, config: &MissionConfig) -> std::io::Result<()> {
    let json = serde_json::to_string_pretty(config)
        .map_err(|e| std::io::Error::new(std::io::ErrorKind::InvalidData, e))?;
    
    // Атомарний запис
    let tmp_path = path.with_extension("tmp");
    fs::write(&tmp_path, &json)?;
    
    // Перевірка: чи можемо прочитати назад
    let verification: MissionConfig = serde_json::from_str(
        &fs::read_to_string(&tmp_path)?
    ).map_err(|e| std::io::Error::new(std::io::ErrorKind::InvalidData, e))?;
    
    if verification != *config {
        fs::remove_file(&tmp_path)?;
        return Err(std::io::Error::new(
            std::io::ErrorKind::InvalidData,
            "Verification failed"
        ));
    }
    
    fs::rename(&tmp_path, path)?;
    
    // sync директорії для гарантії rename на деяких FS
    #[cfg(unix)]
    if let Some(parent) = path.parent() {
        if let Ok(dir) = fs::File::open(parent) {
            let _ = dir.sync_all();
        }
    }
    
    Ok(())
}
```

### Кросплатформне завантаження карти

```rust
use std::path::{Path, PathBuf};

pub struct MapLoader {
    cache_dir: PathBuf,
}

impl MapLoader {
    pub fn new() -> std::io::Result<Self> {
        // Кросплатформний шлях до кешу
        let cache_dir = if cfg!(windows) {
            PathBuf::from(std::env::var("LOCALAPPDATA").unwrap_or_else(|_| ".".into()))
                .join("DroneSwarm")
                .join("MapCache")
        } else if cfg!(target_os = "macos") {
            PathBuf::from(std::env::var("HOME").unwrap_or_else(|_| ".".into()))
                .join("Library")
                .join("Caches")
                .join("DroneSwarm")
        } else {
            // Linux та інші
            PathBuf::from(std::env::var("XDG_CACHE_HOME")
                .unwrap_or_else(|_| {
                    std::env::var("HOME")
                        .map(|h| format!("{}/.cache", h))
                        .unwrap_or_else(|_| ".cache".into())
                }))
                .join("droneswarm")
        };
        
        std::fs::create_dir_all(&cache_dir)?;
        
        Ok(Self { cache_dir })
    }
    
    pub fn get_tile(&self, x: u32, y: u32, zoom: u8) -> std::io::Result<Vec<u8>> {
        let tile_path = self.cache_dir
            .join(format!("{}", zoom))
            .join(format!("{}", x))
            .join(format!("{}.png", y));
        
        if tile_path.exists() {
            std::fs::read(&tile_path)
        } else {
            // Завантажити з мережі та зберегти
            let data = download_tile(x, y, zoom)?;
            
            if let Some(parent) = tile_path.parent() {
                std::fs::create_dir_all(parent)?;
            }
            
            std::fs::write(&tile_path, &data)?;
            Ok(data)
        }
    }
}
```

---

## Резюме

У цьому розділі ми розглянули роботу з файлами та шляхами.

**Основи**:
- `File::open` для читання, `File::create` для запису
- `OpenOptions` для повного контролю режимів
- `BufReader`/`BufWriter` для ефективного I/O
- `Path` (слайс) та `PathBuf` (володіючий) для шляхів
- RAII автоматично закриває файли

**Кросплатформні відмінності**:
- Розділювачі: `/` на Unix, `\` на Windows
- Заборонені символи: `< > : " / \ | ? *` на Windows
- Регістр: важливий на Linux, ігнорується на Windows/macOS
- Максимальна довжина шляху різна

**Кодування**:
- Імена файлів — не завжди UTF-8
- `OsString`/`OsStr` для безпечної роботи
- `to_string_lossy()` для конвертації з можливою втратою

**Надійність**:
- `flush()` для критичних даних
- `sync_all()` для гарантії запису на диск
- Атомарний запис через тимчасовий файл + rename
- `create_new` для атомарного створення

**Продуктивність**:
- `BufReader`/`BufWriter` для зменшення системних викликів
- Потокова обробка для великих файлів
- Memory-mapped files для випадкового доступу

---

## 🔗 Зв'язок з наступним матеріалом

Ви завершили серію розділів про підступні задачі в Rust! Ці знання формують фундамент для створення надійного, кросплатформного програмного забезпечення, що коректно працює з різноманітними даними, ресурсами та середовищами виконання.
