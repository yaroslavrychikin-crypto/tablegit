# Dapper

## 1. Что такое Dapper и зачем он нужен

Dapper — это микро-ORM для .NET от Stack Overflow. Его философия: **вы пишете SQL сами, Dapper избавляет вас от рутины маппинга результатов в C#-объекты и безопасной передачи параметров**. Он не генерирует SQL из LINQ, не отслеживает изменения объектов, не управляет связями и не кэширует результаты. Именно поэтому его производительность близка к «голому» ADO.NET, а поведение предсказуемо.

Dapper уместен там, где нужен полный контроль над SQL и максимальная скорость: сложные JOIN’ы, оконные функции, CTE, отчёты, пакетные операции. Для простого CRUD с богатой моделью домена EF Core может быть удобнее, но когда SQL нужно «довести до предела» — Dapper вне конкуренции.

**Что Dapper делает:**
- Выполняет SQL и маппит строки в объекты (`Query<T>`).
- Выполняет команды без результата (`Execute`).
- Возвращает скалярные значения (`ExecuteScalar`).
- Поддерживает multi-mapping для JOIN’ов (`Query<T1, T2, TReturn>`).
- Работает с параметрами (анонимные объекты, `DynamicParameters`).
- Поддерживает транзакции и хранимые процедуры.

**Чего Dapper не делает:**
- Не отслеживает изменения объектов.
- Не генерирует SQL из LINQ.
- Не управляет связями (lazy loading, каскады).
- Не кэширует результаты запросов.

---

## 2. База данных для всех примеров

Все примеры используют SQLite in-memory. Схема — простая библиотека: авторы, книги, читатели, выдачи.

```csharp
using Microsoft.Data.Sqlite;
using Dapper;

var connectionString = "Data Source=:memory:";
using var connection = new SqliteConnection(connectionString);
connection.Open();

// Создание схемы
connection.Execute("""
    CREATE TABLE Authors (
        Id INTEGER PRIMARY KEY AUTOINCREMENT,
        Name TEXT NOT NULL,
        Country TEXT
    );

    CREATE TABLE Books (
        Id INTEGER PRIMARY KEY AUTOINCREMENT,
        Title TEXT NOT NULL,
        AuthorId INTEGER NOT NULL,
        Year INTEGER,
        FOREIGN KEY (AuthorId) REFERENCES Authors(Id)
    );

    CREATE TABLE Readers (
        Id INTEGER PRIMARY KEY AUTOINCREMENT,
        FullName TEXT NOT NULL,
        Email TEXT UNIQUE
    );

    CREATE TABLE Loans (
        Id INTEGER PRIMARY KEY AUTOINCREMENT,
        BookId INTEGER NOT NULL,
        ReaderId INTEGER NOT NULL,
        LoanDate TEXT NOT NULL,
        ReturnDate TEXT,
        FOREIGN KEY (BookId) REFERENCES Books(Id),
        FOREIGN KEY (ReaderId) REFERENCES Readers(Id)
    );
    """);

// Начальные данные
connection.Execute("""
    INSERT INTO Authors (Name, Country) VALUES 
        ('Лев Толстой', 'Россия'),
        ('Фёдор Достоевский', 'Россия'),
        ('Джордж Оруэлл', 'Великобритания');

    INSERT INTO Books (Title, AuthorId, Year) VALUES
        ('Война и мир', 1, 1869),
        ('Анна Каренина', 1, 1877),
        ('Преступление и наказание', 2, 1866),
        ('1984', 3, 1949);

    INSERT INTO Readers (FullName, Email) VALUES
        ('Иван Петров', 'ivan@example.com'),
        ('Мария Сидорова', 'maria@example.com');

    INSERT INTO Loans (BookId, ReaderId, LoanDate) VALUES
        (1, 1, '2025-01-10'),
        (3, 2, '2025-01-15');
    """);
```

**Пакеты NuGet:**
```bash
dotnet add package Dapper
dotnet add package Microsoft.Data.Sqlite   # для SQLite
# или Microsoft.Data.SqlClient для SQL Server, MySqlConnector для MySQL, Npgsql для PostgreSQL
```

---

## 3. Как Dapper выполняет команды: внутренний контур

Dapper — это набор extension-методов над `IDbConnection`. При вызове `connection.Execute(...)` или `connection.Query<T>(...)` происходит следующее:

1. Создаётся `CommandDefinition`, инкапсулирующий SQL, параметры, транзакцию, таймаут и `CommandType` .
2. Dapper проверяет, является ли параметр коллекцией. Если да — включается режим пакетного выполнения: команда выполняется для каждого элемента, результаты суммируются .
3. Настраивается `IDbCommand`, параметры маппятся из анонимного объекта, `DynamicParameters` или словаря.
4. Вызывается соответствующий метод ADO.NET: `ExecuteNonQuery()` для `Execute`, `ExecuteScalar()` для `ExecuteScalar`, `ExecuteReader()` для `Query`.
5. При чтении результатов Dapper кэширует информацию о маппинге колонок в свойства типа, чтобы последующие вызовы с тем же SQL и типом работали быстрее .

**Практическое следствие:** первый вызов `Query<T>` с новым SQL и новым типом может быть чуть медленнее. Если вы генерируете уникальные SQL-строки на каждый запрос (динамический WHERE), кэш растёт; для одноразовых команд можно передать `CommandFlags.NoCache`.

---

## 4. Основные методы: Execute, Query, QueryFirst, QuerySingle, ExecuteScalar

### 4.1 Execute — команды без результата

`Execute` возвращает количество затронутых строк. Подходит для INSERT, UPDATE, DELETE, DDL.

```csharp
// Вставка с параметрами: параметры передаются отдельно от SQL — защита от инъекций
var affected = connection.Execute(
    "INSERT INTO Authors (Name, Country) VALUES (@Name, @Country)",
    new { Name = "Михаил Булгаков", Country = "Россия" });

Console.WriteLine($"Добавлено авторов: {affected}"); // 1
```

**Пакетное выполнение:** если передать массив объектов, Dapper выполнит команду для каждого элемента. Это не настоящий bulk-insert, а цикл на стороне Dapper.

```csharp
var newReaders = new[]
{
    new { FullName = "Алексей Смирнов", Email = "alex@example.com" },
    new { FullName = "Ольга Кузнецова", Email = "olga@example.com" }
};
var count = connection.Execute(
    "INSERT INTO Readers (FullName, Email) VALUES (@FullName, @Email)", newReaders);
// count == 2
```

**Нюанс:** для тысяч строк это N отдельных round-trip’ов. Для больших объёмов используйте транзакцию вокруг вызовов, `SqlBulkCopy` (SQL Server) или `Dapper.Plus`.

### 4.2 Query<T> — список объектов

```csharp
public class Author
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public string? Country { get; set; }
}

var authors = connection.Query<Author>("SELECT * FROM Authors").ToList();
foreach (var a in authors)
    Console.WriteLine($"{a.Id}: {a.Name} ({a.Country})");
```

**Нюанс маппинга:** Dapper сопоставляет имена колонок со свойствами **без учёта регистра**. Если в БД `full_name`, а в C# `FullName`, стандартный маппинг не сработает. Решения:
- `DefaultTypeMap.MatchNamesWithUnderscores = true;` — глобально, один раз при старте.
- Псевдонимы в SQL: `SELECT full_name AS FullName`.
- `Dapper.FluentMap` для точечных правил .

### 4.3 Буферизация, отложенное выполнение и стриминг

`Query<T>` по умолчанию **буферизует** результат: Dapper читает весь `IDataReader` в список и только потом возвращает `IEnumerable<T>`. Это означает:
- Соединение освобождается сразу после вызова (если оно было открыто внутри `using`).
- Память выделяется под весь результат сразу.

Для больших результатов используйте `buffered: false` — тогда `IEnumerable<T>` будет ленивым, а соединение должно оставаться открытым на момент перечисления.

```csharp
// Буферизованный запрос: безопасно использовать после закрытия соединения
List<Author> authors;
using (var conn = new SqliteConnection("Data Source=:memory:"))
{
    conn.Open();
    // ... схема и данные ...
    authors = conn.Query<Author>("SELECT * FROM Authors").ToList();
}
// authors доступен, соединение закрыто

// Небуферизованный (потоковый) запрос
using (var conn = new SqliteConnection("Data Source=:memory:"))
{
    conn.Open();
    // ... схема и данные ...
    foreach (var author in conn.Query<Author>("SELECT * FROM Authors", buffered: false))
    {
        // обработка по одной строке, память не растёт пропорционально размеру результата
    }
}
```

**Нюанс:** при `buffered: false` нельзя закрывать соединение до завершения перечисления. Если вернуть `IEnumerable<T>` из метода, который закрывает соединение, перечисление упадёт.

### 4.4 QueryFirst, QuerySingle и их «OrDefault»-варианты

| Метод | Нет строк | Одна строка | Больше одной |
|---|---|---|---|
| `QueryFirst<T>` | исключение | возвращает | возвращает первую |
| `QueryFirstOrDefault<T>` | `default(T)` / `null` | возвращает | возвращает первую |
| `QuerySingle<T>` | исключение | возвращает | исключение |
| `QuerySingleOrDefault<T>` | `default(T)` / `null` | возвращает | исключение |

**Практическое правило:** `QuerySingle` — там, где бизнес-логика гарантирует уникальность (выборка по первичному ключу). Если не уверены, что строка есть, — `QuerySingleOrDefault`. `QueryFirst` уместен, когда нужна «любая подходящая» строка и отсутствие данных — ошибка.

```csharp
// Поиск по уникальному email: если email нет — null, если дубликат — исключение
var reader = connection.QuerySingleOrDefault<Reader>(
    "SELECT * FROM Readers WHERE Email = @Email",
    new { Email = "ivan@example.com" });

// Первая книга автора: если книг нет — null
var firstBook = connection.QueryFirstOrDefault<Book>(
    "SELECT * FROM Books WHERE AuthorId = @AuthorId ORDER BY Year",
    new { AuthorId = 1 });
```

**Ловушка:** `QuerySingle` читает все строки, чтобы убедиться, что она ровно одна. На больших результатах это может быть неожиданно дорого.

### 4.5 ExecuteScalar — одно значение

```csharp
var bookCount = connection.ExecuteScalar<int>("SELECT COUNT(*) FROM Books");
Console.WriteLine($"Книг в базе: {bookCount}"); // 4
```

`ExecuteScalar<T>` берёт первую колонку первой строки и приводит к `T`. Если строк нет — вернёт `default(T)`.

**Возврат Id после вставки** — диалект-зависим:

```csharp
// SQLite: last_insert_rowid()
var newId = connection.ExecuteScalar<long>(
    "INSERT INTO Authors (Name, Country) VALUES (@Name, @Country); SELECT last_insert_rowid();",
    new { Name = "Тестовый Автор", Country = "Россия" });

// SQL Server: SCOPE_IDENTITY(); PostgreSQL: RETURNING id
```

Dapper не абстрагирует эту разницу, потому что это часть SQL-диалекта.

---

## 5. Multi-mapping и splitOn: детали, которые ломают код

Multi-mapping — самая мощная и самая коварная функция Dapper. Механизм: Dapper читает одну строку результата и «режет» её на несколько объектов по колонкам, указанным в `splitOn`.

### 5.1 Как работает splitOn

Предположим, SQL возвращает колонки в порядке: `BookId, Title, AuthorId, AuthorName, Country`.

```csharp
connection.Query<Book, Author, Book>(
    sql,
    (book, author) => { book.Author = author; return book; },
    splitOn: "AuthorId");
```

Dapper маппит колонки **до** `AuthorId` (не включая её) в первый тип `Book`. Колонки **начиная с** `AuthorId` — во второй тип `Author` .

**Критический нюанс:** `splitOn` указывает на колонку, которая становится **первой колонкой второго объекта**. Порядок колонок в SELECT имеет значение.

### 5.2 Дефолтное значение splitOn и почему оно опасно

Если не указать `splitOn`, Dapper использует `"Id"`. Это работает, когда обе таблицы имеют `Id`, но создаёт неоднозначность: Dapper возьмёт **первую** встреченную `Id` как границу. Если в SELECT обе `Id` присутствуют, вторая может быть проигнорирована или перезаписана.

**Решение:** псевдонимы (`SELECT b.Id, b.Title, a.Id AS AuthorId, a.Name`) или явный `splitOn: "AuthorId"`.

```csharp
// Явный псевдоним для колонки автора — предсказуемый splitOn
var sql = """
    SELECT 
        b.Id, b.Title, b.Year,
        a.Id AS AuthorId, a.Name, a.Country
    FROM Books b
    JOIN Authors a ON a.Id = b.AuthorId
    """;

var books = connection.Query<Book, Author, Book>(
    sql,
    (book, author) => { book.Author = author; return book; },
    splitOn: "AuthorId"  // граница: AuthorId — первая колонка Author
).ToList();
```

### 5.3 Три и более типа в одном запросе

`splitOn` может принимать несколько колонок через запятую:

```csharp
var sql = """
    SELECT 
        b.Id, b.Title,
        a.Id AS AuthorId, a.Name,
        r.Id AS ReaderId, r.FullName
    FROM Books b
    JOIN Authors a ON a.Id = b.AuthorId
    JOIN Loans l ON l.BookId = b.Id
    JOIN Readers r ON r.Id = l.ReaderId
    """;

var result = connection.Query<Book, Author, Reader, Book>(
    sql,
    (book, author, reader) => { book.Author = author; /* ... */ return book; },
    splitOn: "AuthorId,ReaderId");
```

Порядок generic-параметров должен соответствовать порядку `splitOn`.

### 5.4 Почему multi-mapping не решает один-ко-многим

Если у автора 10 книг, JOIN вернёт 10 строк. Dapper создаст **10 отдельных объектов `Author`** — по одному на строку. Свойство-коллекция `Books` внутри `Author` не заполнится автоматически. Dapper не отслеживает идентичность объектов и не группирует строки.

Три подхода к один-ко-многим:

**Подход A: `QueryMultiple` — два запроса, один round-trip (рекомендуемый).**

```csharp
using var multi = connection.QueryMultiple("""
    SELECT * FROM Authors WHERE Id = @Id;
    SELECT * FROM Books WHERE AuthorId = @Id;
    """, new { Id = 1 });

var author = multi.ReadSingle<Author>();
author.Books = multi.Read<Book>().ToList();
```

**Подход B: дедупликация в памяти после multi-mapping.**

```csharp
var lookup = new Dictionary<int, Author>();
var authors = connection.Query<Author, Book, Author>(
    sql,
    (author, book) =>
    {
        if (!lookup.TryGetValue(author.Id, out var existing))
        {
            existing = author;
            existing.Books = new List<Book>();
            lookup[author.Id] = existing;
        }
        existing.Books.Add(book);
        return existing;
    },
    splitOn: "BookId").Distinct().ToList();
```

**Подход C: JSON-агрегация на стороне БД** (SQL Server, PostgreSQL) — один запрос, один объект на родителя, но требует поддержки JSON в СУБД.

---

## 6. QueryMultiple: несколько result set за один вызов

`QueryMultiple` возвращает `GridReader`, из которого результаты читаются последовательно, в порядке, в котором их вернул SQL.

```csharp
using var multi = connection.QueryMultiple("""
    SELECT COUNT(*) FROM Books;
    SELECT * FROM Authors ORDER BY Name;
    SELECT * FROM Readers;
    """);

var bookCount = multi.ReadSingle<int>();
var authors = multi.Read<Author>().ToList();
var readers = multi.Read<Reader>().ToList();
```

**Нюансы:**
1. **Порядок чтения обязателен.** `GridReader` продвигается вперёд только после полного вычитывания текущего result set.
2. **`ReadSingle<T>` для скаляров** работает, но `ReadFirstOrDefault<T>` может быть безопаснее.
3. **`QueryMultiple` комбинируется с multi-mapping внутри одного `Read`:** `multi.Read<Book, Author, Book>(...)` допустимо.

---

## 7. DynamicParameters: когда анонимного объекта недостаточно

Анонимные объекты покрывают 90% случаев. `DynamicParameters` нужен, когда:
- Параметр должен быть передан как output или input-output.
- SQL генерируется динамически, и набор параметров заранее неизвестен.
- Нужно явно указать `DbType`, `Size` или `Direction`.

```csharp
var p = new DynamicParameters();
p.Add("@Name", "Иван", DbType.String, size: 100);
p.Add("@Email", "ivan@example.com");
// Output-параметр (для хранимых процедур SQL Server)
p.Add("@NewId", dbType: DbType.Int32, direction: ParameterDirection.Output);

connection.Execute("CreateReader", p, commandType: CommandType.StoredProcedure);

var newId = p.Get<int>("@NewId");
```

**Динамический WHERE с DynamicParameters:**

```csharp
public IEnumerable<Book> SearchBooks(string? titlePart, int? yearFrom, int? authorId)
{
    var sql = new StringBuilder("SELECT * FROM Books WHERE 1=1");
    var p = new DynamicParameters();

    if (!string.IsNullOrWhiteSpace(titlePart))
    {
        sql.Append(" AND Title LIKE @TitlePart");
        p.Add("@TitlePart", $"%{titlePart}%");
    }
    if (yearFrom.HasValue)
    {
        sql.Append(" AND Year >= @YearFrom");
        p.Add("@YearFrom", yearFrom.Value);
    }
    if (authorId.HasValue)
    {
        sql.Append(" AND AuthorId = @AuthorId");
        p.Add("@AuthorId", authorId.Value);
    }

    return connection.Query<Book>(sql.ToString(), p);
}
```

**Важное предупреждение:** `DynamicParameters` передаёт значения как параметры ADO.NET — это безопасно. Но если вы вставляете в SQL имена колонок или таблиц из пользовательского ввода, параметризация не спасёт — нужен белый список допустимых значений.

---

## 8. Транзакции: явное управление и типичные ошибки

Dapper не открывает и не закрывает транзакции автоматически. Каждый вызов внутри транзакции должен получать объект `IDbTransaction`.

```csharp
using var connection = new SqliteConnection("Data Source=:memory:");
connection.Open();
// ... схема ...

using var tx = connection.BeginTransaction();
try
{
    var authorId = connection.ExecuteScalar<long>(
        "INSERT INTO Authors (Name, Country) VALUES (@Name, @Country); SELECT last_insert_rowid();",
        new { Name = "Новый Автор", Country = "Италия" },
        tx);

    connection.Execute(
        "INSERT INTO Books (Title, AuthorId, Year) VALUES (@Title, @AuthorId, @Year)",
        new { Title = "Новая Книга", AuthorId = authorId, Year = 2025 },
        tx);

    tx.Commit();
}
catch
{
    tx.Rollback();
    throw;
}
```

**Что происходит, если забыть передать `tx`:** команда выполнится в контексте соединения, но **вне транзакции**. `Rollback()` её не откатит. Это одна из самых частых и самых незаметных ошибок при работе с Dapper.

**Вложенные транзакции:** ADO.NET не поддерживает вложенные транзакции на уровне `BeginTransaction` в привычном смысле. Для композиции используйте `TransactionScope` (с осторожностью в асинхронном коде) или паттерн Unit of Work поверх Dapper, который централизует управление соединением и транзакцией .

---

## 9. Хранимые процедуры: входные, выходные и CommandType

Для SQL Server вызов процедуры:

```csharp
var authors = connection.Query<Author>(
    "GetAuthorsByCountry",
    new { Country = "Россия" },
    commandType: CommandType.StoredProcedure);
```

Ключевой момент — `commandType: CommandType.StoredProcedure`. Без него Dapper попытается выполнить текст `"GetAuthorsByCountry"` как обычный SQL и получит синтаксическую ошибку.

**Выходные параметры** требуют `DynamicParameters`:

```csharp
var p = new DynamicParameters();
p.Add("@PlaceNumber", 1);
p.Add("@PlaceName", "Первое место");
p.Add("@PrizeAmount", 1000m);
p.Add("@ID", dbType: DbType.Int32, direction: ParameterDirection.Output);

connection.Execute("SP_Prizes_Insert", p, commandType: CommandType.StoredProcedure);
var newId = p.Get<int>("@ID");
```

Этот паттерн подробно разобран в материалах по Dapper + SQL Server .

**Нюанс для SQLite:** хранимых процедур в привычном смысле нет, поэтому примеры на SQLite используют обычный SQL. Принцип передачи `commandType` важен понимать для перехода на SQL Server или PostgreSQL.

---

## 10. Типовые ловушки и как их обойти

| Ситуация | Что происходит | Решение |
|---|---|---|
| `splitOn` колонка не найдена | `Multi-map error: splitOn column 'X' was not found` | Убедиться, что колонка есть в SELECT, часто забывают `a.Id AS AuthorId`  |
| JOIN один-ко-многим через multi-mapping | Дублирование родительских объектов | `QueryMultiple` или дедупликация через `GroupBy`  |
| `Query<T>` с колонкой `full_name` и свойством `FullName` | Свойство остаётся `null` | `MatchNamesWithUnderscores = true` или псевдоним в SQL  |
| `Execute` с массивом на 10 000 объектов | 10 000 отдельных round-trip’ов | Транзакция + пакетная вставка, либо `SqlBulkCopy` / `Dapper.Plus` |
| Забыли `transaction:` в вызове внутри `BeginTransaction` | Команда вне транзакции, откат не работает | Передавать `tx` в каждый вызов |
| `QuerySingle`, а строк 0 или 2 | Исключение | Осознанно выбирать `QuerySingle` только при гарантии уникальности |
| Порядок колонок в SELECT не совпадает с ожиданием `splitOn` | Данные попадают не в те свойства | Явные псевдонимы и явный `splitOn` |

---

## 11. Шпаргалка по выбору инструмента

| Задача | Инструмент |
|---|---|
| Простой список объектов | `Query<T>` |
| Одна строка, может отсутствовать | `QueryFirstOrDefault<T>` |
| Гарантированно одна строка | `QuerySingle<T>` |
| Одно скалярное значение | `ExecuteScalar<T>` |
| INSERT/UPDATE/DELETE | `Execute` |
| JOIN двух таблиц | `Query<T1, T2, TReturn>` + `splitOn` |
| JOIN трёх таблиц | `Query<T1, T2, T3, TReturn>` + `splitOn` с двумя колонками |
| Один-ко-многим | `QueryMultiple` (два SELECT) |
| Несколько result set | `QueryMultiple` + `Read<T>` |
| Выходные параметры | `DynamicParameters` + `ParameterDirection.Output` |
| Динамический WHERE | `StringBuilder` + `DynamicParameters` |
| Атомарность | `BeginTransaction` + передача `tx` в каждый вызов |
| Bulk-вставка | `SqlBulkCopy` / `Dapper.Plus` (не `Execute` с массивом) |

---

## 12. Тренировочные задания (10 штук, с каркасом без кода)

Задания используют ту же схему БД (Authors, Books, Readers, Loans). Для каждого указан ожидаемый результат. Код не приводится — только каркас.

**Задание 1. Базовый CRUD + ExecuteScalar**
Добавьте нового читателя «Сергей Волков» с email `sergey@example.com`. Верните его `Id`. Затем выведите всех читателей, отсортированных по имени.
*Проверка:* в списке появился Сергей, всего читателей — 4 (с учётом добавленных ранее Алексея и Ольги).

**Задание 2. Параметризованный поиск с QueryFirstOrDefault**
Напишите метод `GetReaderByEmail(string email)`, возвращающий `Reader?`. Если email `null` или пустой — вернуть `null`, не выполняя запрос.
*Проверка:* существующий email → объект; несуществующий → `null`; `null` → `null` без исключения.

**Задание 3. JOIN двух таблиц с multi-mapping**
Выведите все книги с информацией об авторе (имя и страна). Используйте `Query<Book, Author, Book>` и `splitOn: "AuthorId"`.
*Проверка:* 4 книги, у каждой `Author.Name` не пустое.

**Задание 4. Один-ко-многим через QueryMultiple**
Для автора с `Id = 1` получите его данные и список всех его книг одним вызовом `QueryMultiple`.
*Ожидаемый результат:* Лев Толстой, книги «Война и мир» и «Анна Каренина».

**Задание 5. Агрегация и ExecuteScalar**
Найдите количество книг, написанных авторами из России. Используйте `ExecuteScalar<int>` и JOIN в SQL.
*Ожидаемый результат:* 3.

**Задание 6. Транзакция с откатом**
Напишите метод, который в одной транзакции: 1) вставляет новую книгу; 2) вставляет запись о выдаче; 3) намеренно выбрасывает исключение после второй вставки. Убедитесь, что после исключения ни книга, ни выдача не сохранились.
*Проверка:* `SELECT COUNT(*) FROM Books` не изменился.

**Задание 7. Обновление с проверкой количества**
Обновите email читателя с `Id = 1` на `newemail@example.com`. Метод возвращает `true`, если обновлена ровно одна строка, и `false` иначе. Используйте `Execute`.
*Проверка:* первый вызов — `true`, повторный с тем же email (если строка всё равно обновляется) — тоже `true`, но проверьте семантику.

**Задание 8. Поиск с LIKE и DynamicParameters**
Реализуйте поиск книг по части названия. Параметр `titlePart` может быть `null` — тогда поиск не применяется. Используйте `DynamicParameters` для условного добавления условия.
*Проверка:* при `titlePart = "Война"` — одна книга; при `null` — все.

**Задание 9. Удаление с каскадной проверкой**
Удалите читателя с `Id = 2`. Сначала проверьте, есть ли у него невозвращённые выдачи (`ReturnDate IS NULL`). Если есть — не удаляйте и верните сообщение об ошибке. Если нет — удалите и верните `true`.
*Ожидаемый результат:* у читателя Id=2 есть выдача книги 3 без `ReturnDate`, поэтому удаление отклонено.

**Задание 10. Сложный отчёт (LEFT JOIN + GROUP BY)**
Выведите список авторов с количеством их книг, включая авторов без книг (`LEFT JOIN`), отсортированный по убыванию количества.
*Ожидаемый результат (для исходных данных):* Толстой — 2, Достоевский — 1, Оруэлл — 1, Булгаков — 0 (если добавлен).

**Задание 11 (дополнительное). Пакетная вставка и производительность**
Сгенерируйте 1000 объектов `Author` и вставьте их: 1) через `connection.Execute(sql, authors)` без транзакции; 2) через `connection.Execute(sql, authors)` внутри транзакции. Замерьте время через `Stopwatch`. Объясните разницу.
*Проверка:* второй вариант значительно быстрее, потому что все INSERT выполняются в рамках одной транзакции.

**Задание 12 (дополнительное). Обработка дубликатов при multi-mapping**
Напишите запрос, возвращающий книги с авторами через JOIN, с `splitOn: "AuthorId"`. Сгруппируйте результат по `Author.Id` и выведите для каждого автора список его книг. Убедитесь, что количество уникальных авторов с книгами соответствует ожидаемому.
*Проверка:* три группы, книги не дублируются внутри группы.

---

## 13. Итог

Dapper — это инструмент, который даёт вам ровно столько, сколько нужно: выполнение SQL, маппинг, параметры, транзакции. Всё остальное — на вашей стороне. Понимание внутреннего контура (буферизация, кэш маппинга, splitOn, поведение `Execute` с коллекциями) отличает код, который «работает», от кода, который «работает предсказуемо и быстро». Практика на заданиях выше закрепит именно эти нюансы.
