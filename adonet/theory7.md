# Dapper для SQL Server: полная теория, примеры и тренировочные задания (с хранимыми процедурами)

Ниже — объединённый материал из предыдущих ответов, переписанный под **Microsoft SQL Server**, с полным скриптом базы данных, **включая все хранимые процедуры, которые используются в примерах**. Примеры можно запустить на локальном SQL Server (LocalDB, Express или полноценный инстанс).

---

## 1. Что такое Dapper и зачем он нужен

Dapper — микро-ORM для .NET от Stack Overflow. Философия: **вы пишете SQL сами, Dapper избавляет от рутины маппинга результатов в C#-объекты и безопасной передачи параметров**. Он не генерирует SQL из LINQ, не отслеживает изменения объектов, не управляет связями и не кэширует результаты. Поэтому его производительность близка к «голому» ADO.NET, а поведение предсказуемо.

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

## 2. Подключение и пакеты NuGet

```bash
dotnet add package Dapper
dotnet add package Microsoft.Data.SqlClient
```

`Microsoft.Data.SqlClient` — современный драйвер для SQL Server (замена устаревшему `System.Data.SqlClient`).

**Строки подключения:**

```csharp
// LocalDB (устанавливается с Visual Studio)
// "Server=(localdb)\\MSSQLLocalDB;Database=DapperDemo;Trusted_Connection=True;TrustServerCertificate=True;"

// SQL Server Express
// "Server=.\\SQLEXPRESS;Database=DapperDemo;Trusted_Connection=True;TrustServerCertificate=True;"

// Полноценный инстанс с логином/паролем
// "Server=localhost;Database=DapperDemo;User Id=sa;Password=YourPassword;TrustServerCertificate=True;"

const string connectionString =
    "Server=(localdb)\\MSSQLLocalDB;Database=DapperDemo;Trusted_Connection=True;TrustServerCertificate=True;";
```

`TrustServerCertificate=True` нужен для локальных инстансов с самоподписанным сертификатом — иначе `Microsoft.Data.SqlClient` 4+ откажется подключаться.

---

## 3. База данных для всех примеров (полный скрипт)

Скрипт создаёт БД, схему, начальные данные, **хранимые процедуры и табличный тип** для всех примеров. Его можно выполнить один раз через `sqlcmd`, SSMS или прямо из кода (например, `connection.Execute(script)` — но помните, что `GO` не является частью T-SQL, его нужно либо убрать, либо выполнять скрипт через `sqlcmd`/SSMS).

```sql
IF DB_ID('DapperDemo') IS NULL
    CREATE DATABASE DapperDemo;
GO

USE DapperDemo;
GO

-- Удаление в порядке зависимостей
IF OBJECT_ID('dbo.GetAuthorsByCountry', 'P') IS NOT NULL DROP PROCEDURE dbo.GetAuthorsByCountry;
IF OBJECT_ID('dbo.CreateAuthor', 'P') IS NOT NULL DROP PROCEDURE dbo.CreateAuthor;
IF OBJECT_ID('dbo.GetAuthorWithBooks', 'P') IS NOT NULL DROP PROCEDURE dbo.GetAuthorWithBooks;
IF OBJECT_ID('dbo.GetBooksByAuthor', 'P') IS NOT NULL DROP PROCEDURE dbo.GetBooksByAuthor;
IF OBJECT_ID('dbo.GetReaderByEmail', 'P') IS NOT NULL DROP PROCEDURE dbo.GetReaderByEmail;
IF OBJECT_ID('dbo.UpdateReaderEmail', 'P') IS NOT NULL DROP PROCEDURE dbo.UpdateReaderEmail;
IF OBJECT_ID('dbo.SearchBooks', 'P') IS NOT NULL DROP PROCEDURE dbo.SearchBooks;
IF OBJECT_ID('dbo.CreateLoan', 'P') IS NOT NULL DROP PROCEDURE dbo.CreateLoan;
IF OBJECT_ID('dbo.ReturnLoan', 'P') IS NOT NULL DROP PROCEDURE dbo.ReturnLoan;
IF OBJECT_ID('dbo.AuthorReport', 'P') IS NOT NULL DROP PROCEDURE dbo.AuthorReport;
IF OBJECT_ID('dbo.ReaderLoanStats', 'P') IS NOT NULL DROP PROCEDURE dbo.ReaderLoanStats;
IF OBJECT_ID('dbo.AuthorType', 'T') IS NOT NULL DROP TYPE dbo.AuthorType;

IF OBJECT_ID('dbo.Loans', 'U') IS NOT NULL DROP TABLE dbo.Loans;
IF OBJECT_ID('dbo.Books', 'U') IS NOT NULL DROP TABLE dbo.Books;
IF OBJECT_ID('dbo.Readers', 'U') IS NOT NULL DROP TABLE dbo.Readers;
IF OBJECT_ID('dbo.Authors', 'U') IS NOT NULL DROP TABLE dbo.Authors;
GO

CREATE TABLE dbo.Authors (
    Id       INT IDENTITY(1,1) PRIMARY KEY,
    Name     NVARCHAR(200) NOT NULL,
    Country  NVARCHAR(100) NULL
);

CREATE TABLE dbo.Books (
    Id        INT IDENTITY(1,1) PRIMARY KEY,
    Title     NVARCHAR(300) NOT NULL,
    AuthorId  INT NOT NULL,
    Year      INT NULL,
    CONSTRAINT FK_Books_Authors FOREIGN KEY (AuthorId) REFERENCES dbo.Authors(Id)
);

CREATE TABLE dbo.Readers (
    Id        INT IDENTITY(1,1) PRIMARY KEY,
    FullName  NVARCHAR(200) NOT NULL,
    Email     NVARCHAR(200) NOT NULL UNIQUE
);

CREATE TABLE dbo.Loans (
    Id         INT IDENTITY(1,1) PRIMARY KEY,
    BookId     INT NOT NULL,
    ReaderId   INT NOT NULL,
    LoanDate   DATE NOT NULL,
    ReturnDate DATE NULL,
    CONSTRAINT FK_Loans_Books   FOREIGN KEY (BookId)   REFERENCES dbo.Books(Id),
    CONSTRAINT FK_Loans_Readers FOREIGN KEY (ReaderId) REFERENCES dbo.Readers(Id)
);
GO

INSERT INTO dbo.Authors (Name, Country) VALUES
    (N'Лев Толстой',        N'Россия'),
    (N'Фёдор Достоевский',  N'Россия'),
    (N'Джордж Оруэлл',      N'Великобритания');

INSERT INTO dbo.Books (Title, AuthorId, Year) VALUES
    (N'Война и мир',               1, 1869),
    (N'Анна Каренина',             1, 1877),
    (N'Преступление и наказание',  2, 1866),
    (N'1984',                      3, 1949);

INSERT INTO dbo.Readers (FullName, Email) VALUES
    (N'Иван Петров',    N'ivan@example.com'),
    (N'Мария Сидорова', N'maria@example.com');

INSERT INTO dbo.Loans (BookId, ReaderId, LoanDate) VALUES
    (1, 1, '2025-01-10'),
    (3, 2, '2025-01-15');
GO

-- ============================================================
-- ТАБЛИЧНЫЙ ТИП для TVP-примера
-- ============================================================
CREATE TYPE dbo.AuthorType AS TABLE (
    Name    NVARCHAR(200),
    Country NVARCHAR(100)
);
GO

-- ============================================================
-- ХРАНИМЫЕ ПРОЦЕДУРЫ, ИСПОЛЬЗУЕМЫЕ В ПРИМЕРАХ
-- ============================================================

-- 1) Авторы по стране (раздел 10.1)
CREATE OR ALTER PROCEDURE dbo.GetAuthorsByCountry
    @Country NVARCHAR(100)
AS
BEGIN
    SET NOCOUNT ON;
    SELECT Id, Name, Country
    FROM dbo.Authors
    WHERE Country = @Country;
END
GO

-- 2) Создание автора с OUTPUT-параметром (раздел 10.1)
CREATE OR ALTER PROCEDURE dbo.CreateAuthor
    @Name    NVARCHAR(200),
    @Country NVARCHAR(100),
    @NewId   INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    INSERT INTO dbo.Authors (Name, Country) VALUES (@Name, @Country);
    SET @NewId = CAST(SCOPE_IDENTITY() AS INT);
END
GO

-- 3) Автор + его книги (демонстрация QueryMultiple с процедурой)
CREATE OR ALTER PROCEDURE dbo.GetAuthorWithBooks
    @AuthorId INT
AS
BEGIN
    SET NOCOUNT ON;
    SELECT Id, Name, Country FROM dbo.Authors WHERE Id = @AuthorId;
    SELECT Id, Title, AuthorId, Year FROM dbo.Books WHERE AuthorId = @AuthorId;
END
GO

-- 4) Книги автора (отдельная процедура для QueryMultiple)
CREATE OR ALTER PROCEDURE dbo.GetBooksByAuthor
    @AuthorId INT
AS
BEGIN
    SET NOCOUNT ON;
    SELECT Id, Title, AuthorId, Year FROM dbo.Books WHERE AuthorId = @AuthorId;
END
GO

-- 5) Читатель по email (демонстрация QueryFirstOrDefault через процедуру)
CREATE OR ALTER PROCEDURE dbo.GetReaderByEmail
    @Email NVARCHAR(200)
AS
BEGIN
    SET NOCOUNT ON;
    SELECT Id, FullName, Email FROM dbo.Readers WHERE Email = @Email;
END
GO

-- 6) Обновление email читателя (демонстрация Execute с OUTPUT-параметром)
CREATE OR ALTER PROCEDURE dbo.UpdateReaderEmail
    @ReaderId INT,
    @NewEmail NVARCHAR(200),
    @RowsAffected INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    UPDATE dbo.Readers SET Email = @NewEmail WHERE Id = @ReaderId;
    SET @RowsAffected = @@ROWCOUNT;
END
GO

-- 7) Динамический поиск книг (демонстрация DynamicParameters + процедуры)
CREATE OR ALTER PROCEDURE dbo.SearchBooks
    @TitlePart NVARCHAR(300) = NULL,
    @YearFrom  INT = NULL,
    @AuthorId  INT = NULL
AS
BEGIN
    SET NOCOUNT ON;
    SELECT Id, Title, AuthorId, Year
    FROM dbo.Books
    WHERE (@TitlePart IS NULL OR Title LIKE '%' + @TitlePart + '%')
      AND (@YearFrom  IS NULL OR Year >= @YearFrom)
      AND (@AuthorId  IS NULL OR AuthorId = @AuthorId);
END
GO

-- 8) Создание выдачи (демонстрация транзакции внутри процедуры)
CREATE OR ALTER PROCEDURE dbo.CreateLoan
    @BookId   INT,
    @ReaderId INT,
    @LoanDate DATE,
    @NewId    INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;

    IF NOT EXISTS (SELECT 1 FROM dbo.Books WHERE Id = @BookId)
        THROW 50001, 'Книга не найдена', 1;

    IF NOT EXISTS (SELECT 1 FROM dbo.Readers WHERE Id = @ReaderId)
        THROW 50002, 'Читатель не найден', 1;

    INSERT INTO dbo.Loans (BookId, ReaderId, LoanDate)
    VALUES (@BookId, @ReaderId, @LoanDate);

    SET @NewId = CAST(SCOPE_IDENTITY() AS INT);
END
GO

-- 9) Возврат книги (демонстрация Execute с проверкой количества строк)
CREATE OR ALTER PROCEDURE dbo.ReturnLoan
    @LoanId     INT,
    @ReturnDate DATE,
    @RowsAffected INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    UPDATE dbo.Loans
    SET ReturnDate = @ReturnDate
    WHERE Id = @LoanId AND ReturnDate IS NULL;
    SET @RowsAffected = @@ROWCOUNT;
END
GO

-- 10) Отчёт по авторам (LEFT JOIN + GROUP BY, раздел 13, задание 10)
CREATE OR ALTER PROCEDURE dbo.AuthorReport
AS
BEGIN
    SET NOCOUNT ON;
    SELECT a.Id, a.Name, a.Country, COUNT(b.Id) AS BookCount
    FROM dbo.Authors a
    LEFT JOIN dbo.Books b ON b.AuthorId = a.Id
    GROUP BY a.Id, a.Name, a.Country
    ORDER BY BookCount DESC, a.Name;
END
GO

-- 11) Статистика по читателям (раздел 13, задание 10)
CREATE OR ALTER PROCEDURE dbo.ReaderLoanStats
AS
BEGIN
    SET NOCOUNT ON;
    SELECT 
        r.Id,
        r.FullName,
        COUNT(l.Id) AS TotalLoans,
        SUM(CASE WHEN l.ReturnDate IS NULL THEN 1 ELSE 0 END) AS ActiveLoans
    FROM dbo.Readers r
    LEFT JOIN dbo.Loans l ON l.ReaderId = r.Id
    GROUP BY r.Id, r.FullName
    ORDER BY r.FullName;
END
GO
```

**Отличия от SQLite-версии:**
- `NVARCHAR` вместо `TEXT` — корректная работа с кириллицей.
- `IDENTITY(1,1)` — автоинкремент.
- `DATE` вместо строки — маппится в `DateTime` без парсинга.
- Явные `CONSTRAINT` с именами — удобнее для отладки.
- `SET NOCOUNT ON` в каждой процедуре — иначе Dapper прочитает лишний result set.
- `CREATE OR ALTER` — идемпотентное создание процедур (SQL Server 2016 SP1+).
- `THROW` — современный способ генерации ошибок в T-SQL.

---

## 4. Как Dapper выполняет команды: внутренний контур

Dapper — набор extension-методов над `IDbConnection`. При вызове `connection.Execute(...)` или `connection.Query<T>(...)` происходит следующее:

1. Создаётся `CommandDefinition`, инкапсулирующий SQL, параметры, транзакцию, таймаут и `CommandType` .
2. Dapper проверяет, является ли параметр коллекцией. Если да — включается режим пакетного выполнения: команда выполняется для каждого элемента, результаты суммируются .
3. Настраивается `IDbCommand`, параметры маппятся из анонимного объекта, `DynamicParameters` или словаря.
4. Вызывается соответствующий метод ADO.NET: `ExecuteNonQuery()` для `Execute`, `ExecuteScalar()` для `ExecuteScalar`, `ExecuteReader()` для `Query`.
5. При чтении результатов Dapper кэширует информацию о маппинге колонок в свойства типа, чтобы последующие вызовы с тем же SQL и типом работали быстрее .

**Практическое следствие:** первый вызов `Query<T>` с новым SQL и новым типом может быть чуть медленнее. Для одноразовых команд можно передать `CommandFlags.NoCache`.

**Важно для SQL Server:** Dapper использует `sp_executesql` с параметрами, поэтому планы запросов переиспользуются SQL Server между вызовами с разными значениями параметров.

---

## 5. Основные методы: Execute, Query, QueryFirst, QuerySingle, ExecuteScalar

### 5.1 Execute — команды без результата

`Execute` возвращает количество затронутых строк. Подходит для INSERT, UPDATE, DELETE, DDL.

```csharp
using Microsoft.Data.SqlClient;
using Dapper;

using var connection = new SqlConnection(connectionString);
connection.Open();

// Вставка с параметрами
var affected = connection.Execute(
    "INSERT INTO dbo.Authors (Name, Country) VALUES (@Name, @Country)",
    new { Name = "Михаил Булгаков", Country = "Россия" });

Console.WriteLine($"Добавлено авторов: {affected}"); // 1
```

**Пакетное выполнение:** если передать массив объектов, Dapper выполнит команду для каждого элемента.

```csharp
var newReaders = new[]
{
    new { FullName = "Алексей Смирнов", Email = "alex@example.com" },
    new { FullName = "Ольга Кузнецова", Email = "olga@example.com" }
};
var count = connection.Execute(
    "INSERT INTO dbo.Readers (FullName, Email) VALUES (@FullName, @Email)", newReaders);
// count == 2
```

**Нюанс для SQL Server:** для тысяч строк используйте `SqlBulkCopy` или TVP (см. раздел 10).

### 5.2 Query<T> — список объектов

```csharp
public class Author
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public string? Country { get; set; }
}

var authors = connection.Query<Author>("SELECT * FROM dbo.Authors").ToList();
foreach (var a in authors)
    Console.WriteLine($"{a.Id}: {a.Name} ({a.Country})");
```

**Нюанс маппинга:** Dapper сопоставляет имена колонок со свойствами **без учёта регистра**. Если в БД `full_name`, а в C# `FullName`, стандартный маппинг не сработает. Решения: `MatchNamesWithUnderscores = true`, псевдонимы в SQL, `Dapper.FluentMap`.

### 5.3 Буферизация, отложенное выполнение и стриминг

`Query<T>` по умолчанию **буферизует** результат: Dapper читает весь `IDataReader` в список. Для больших результатов используйте `buffered: false`.

```csharp
// Буферизованный запрос
List<Author> authors;
using (var conn = new SqlConnection(connectionString))
{
    conn.Open();
    authors = conn.Query<Author>("SELECT * FROM dbo.Authors").ToList();
}

// Небуферизованный (потоковый) запрос
using (var conn = new SqlConnection(connectionString))
{
    conn.Open();
    foreach (var author in conn.Query<Author>("SELECT * FROM dbo.Authors", buffered: false))
    {
        // обработка по одной строке
    }
}
```

**Нюанс:** при `buffered: false` нельзя закрывать соединение до завершения перечисления.

### 5.4 QueryFirst, QuerySingle и их «OrDefault»-варианты

| Метод | Нет строк | Одна строка | Больше одной |
|---|---|---|---|
| `QueryFirst<T>` | исключение | возвращает | возвращает первую |
| `QueryFirstOrDefault<T>` | `default(T)` / `null` | возвращает | возвращает первую |
| `QuerySingle<T>` | исключение | возвращает | исключение |
| `QuerySingleOrDefault<T>` | `default(T)` / `null` | возвращает | исключение |

```csharp
var reader = connection.QuerySingleOrDefault<Reader>(
    "SELECT * FROM dbo.Readers WHERE Email = @Email",
    new { Email = "ivan@example.com" });

var firstBook = connection.QueryFirstOrDefault<Book>(
    "SELECT * FROM dbo.Books WHERE AuthorId = @AuthorId ORDER BY Year",
    new { AuthorId = 1 });
```

**Ловушка:** `QuerySingle` читает все строки, чтобы убедиться, что она ровно одна.

### 5.5 ExecuteScalar и возврат Id после вставки

```csharp
var bookCount = connection.ExecuteScalar<int>("SELECT COUNT(*) FROM dbo.Books");
```

**Возврат Id после INSERT** — в SQL Server через `SCOPE_IDENTITY()` или `OUTPUT INSERTED.Id`:

```csharp
// Вариант 1: SCOPE_IDENTITY()
var newId = connection.ExecuteScalar<int>(
    "INSERT INTO dbo.Authors (Name, Country) VALUES (@Name, @Country); SELECT CAST(SCOPE_IDENTITY() AS INT);",
    new { Name = "Тестовый Автор", Country = "Россия" });

// Вариант 2: OUTPUT INSERTED.Id — рекомендованный Microsoft, работает и с триггерами
var newId2 = connection.ExecuteScalar<int>(
    "INSERT INTO dbo.Authors (Name, Country) OUTPUT INSERTED.Id VALUES (@Name, @Country)",
    new { Name = "Второй Автор", Country = "Россия" });
```

**Нюанс:** `@@IDENTITY` использовать нельзя — он может вернуть Id из триггера.

---

## 6. Multi-mapping и splitOn: детали, которые ломают код

Multi-mapping — самая мощная и самая коварная функция Dapper. Dapper читает одну строку результата и «режет» её на несколько объектов по колонкам, указанным в `splitOn`.

### 6.1 Как работает splitOn

```csharp
connection.Query<Book, Author, Book>(
    sql,
    (book, author) => { book.Author = author; return book; },
    splitOn: "AuthorId");
```

Dapper маппит колонки **до** `AuthorId` (не включая её) в первый тип `Book`. Колонки **начиная с** `AuthorId` — во второй тип `Author` .

**Критический нюанс:** `splitOn` указывает на колонку, которая становится **первой колонкой второго объекта**. Порядок колонок в SELECT имеет значение.

### 6.2 Дефолтное значение splitOn и почему оно опасно

Если не указать `splitOn`, Dapper использует `"Id"`. В SQL Server-схеме выше **обе таблицы имеют `Id`**, поэтому `SELECT * FROM Books b JOIN Authors a ...` вернёт две колонки `Id`, и Dapper возьмёт **первую** как границу — данные автора попадут не туда.

**Решение:** псевдонимы или явный `splitOn: "AuthorId"`.

```csharp
var sql = """
    SELECT 
        b.Id, b.Title, b.Year,
        a.Id AS AuthorId, a.Name, a.Country
    FROM dbo.Books b
    JOIN dbo.Authors a ON a.Id = b.AuthorId
    """;

var books = connection.Query<Book, Author, Book>(
    sql,
    (book, author) => { book.Author = author; return book; },
    splitOn: "AuthorId"
).ToList();
```

### 6.3 Три и более типа в одном запросе

```csharp
var sql = """
    SELECT 
        b.Id, b.Title,
        a.Id AS AuthorId, a.Name,
        r.Id AS ReaderId, r.FullName
    FROM dbo.Books b
    JOIN dbo.Authors a ON a.Id = b.AuthorId
    JOIN dbo.Loans l ON l.BookId = b.Id
    JOIN dbo.Readers r ON r.Id = l.ReaderId
    """;

var result = connection.Query<Book, Author, Reader, Book>(
    sql,
    (book, author, reader) => { book.Author = author; /* ... */ return book; },
    splitOn: "AuthorId,ReaderId");
```

Порядок generic-параметров должен соответствовать порядку `splitOn`.

### 6.4 Почему multi-mapping не решает один-ко-многим

Если у автора 10 книг, JOIN вернёт 10 строк. Dapper создаст **10 отдельных объектов `Author`** — по одному на строку. Свойство-коллекция `Books` не заполнится автоматически.

Три подхода:

**Подход A: `QueryMultiple` — два запроса, один round-trip (рекомендуемый).** Можно использовать процедуру `dbo.GetAuthorWithBooks`:

```csharp
using var multi = connection.QueryMultiple(
    "dbo.GetAuthorWithBooks",
    new { AuthorId = 1 },
    commandType: CommandType.StoredProcedure);

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

**Подход C: JSON-агрегация на стороне SQL Server.**

```csharp
var sql = """
    SELECT 
        a.Id, a.Name, a.Country,
        (SELECT b.Id, b.Title, b.Year
         FROM dbo.Books b
         WHERE b.AuthorId = a.Id
         FOR JSON PATH) AS BooksJson
    FROM dbo.Authors a
    """;

var authors = connection.Query<AuthorWithJson>(sql)
    .Select(a => new Author
    {
        Id = a.Id,
        Name = a.Name,
        Country = a.Country,
        Books = string.IsNullOrEmpty(a.BooksJson)
            ? new List<Book>()
            : JsonSerializer.Deserialize<List<Book>>(a.BooksJson)!
    }).ToList();
```

`FOR JSON PATH` — нативный SQL Server-способ сериализации.

---

## 7. QueryMultiple: несколько result set за один вызов

`QueryMultiple` возвращает `GridReader`, из которого результаты читаются последовательно.

```csharp
using var multi = connection.QueryMultiple("""
    SELECT COUNT(*) FROM dbo.Books;
    SELECT * FROM dbo.Authors ORDER BY Name;
    SELECT * FROM dbo.Readers;
    """);

var bookCount = multi.ReadSingle<int>();
var authors = multi.Read<Author>().ToList();
var readers = multi.Read<Reader>().ToList();
```

**Тот же подход через хранимую процедуру `dbo.GetAuthorWithBooks`:**

```csharp
using var multi = connection.QueryMultiple(
    "dbo.GetAuthorWithBooks",
    new { AuthorId = 1 },
    commandType: CommandType.StoredProcedure);

var author = multi.ReadSingle<Author>();
author.Books = multi.Read<Book>().ToList();
```

**Нюансы:**
1. **Порядок чтения обязателен.** `GridReader` продвигается вперёд только после полного вычитывания текущего result set.
2. **`ReadSingle<T>` для скаляров** работает, но `ReadFirstOrDefault<T>` может быть безопаснее.
3. **`QueryMultiple` комбинируется с multi-mapping внутри одного `Read`:** `multi.Read<Book, Author, Book>(...)` допустимо.
4. **В процедурах обязателен `SET NOCOUNT ON`** — иначе лишний result set с `@@ROWCOUNT` сломает порядок чтения.

---

## 8. DynamicParameters: когда анонимного объекта недостаточно

`DynamicParameters` нужен, когда:
- Параметр должен быть передан как output или input-output.
- SQL генерируется динамически, и набор параметров заранее неизвестен.
- Нужно явно указать `DbType`, `Size` или `Direction`.

### 8.1 Вызов процедуры с OUTPUT-параметром (`dbo.CreateAuthor`)

```csharp
var p = new DynamicParameters();
p.Add("@Name", "Процедурный Автор");
p.Add("@Country", "Россия");
p.Add("@NewId", dbType: DbType.Int32, direction: ParameterDirection.Output);

connection.Execute("dbo.CreateAuthor", p, commandType: CommandType.StoredProcedure);
var newId = p.Get<int>("@NewId");
```

### 8.2 Вызов процедуры `dbo.UpdateReaderEmail` с проверкой количества строк

```csharp
var p = new DynamicParameters();
p.Add("@ReaderId", 1);
p.Add("@NewEmail", "newemail@example.com");
p.Add("@RowsAffected", dbType: DbType.Int32, direction: ParameterDirection.Output);

connection.Execute("dbo.UpdateReaderEmail", p, commandType: CommandType.StoredProcedure);
var rows = p.Get<int>("@RowsAffected");
bool updated = rows == 1;
```

### 8.3 Вызов процедуры `dbo.SearchBooks` с опциональными параметрами

Процедура уже содержит логику `(@Param IS NULL OR ...)`, поэтому можно передавать `null`:

```csharp
var books = connection.Query<Book>(
    "dbo.SearchBooks",
    new { TitlePart = (string?)null, YearFrom = 1900, AuthorId = (int?)null },
    commandType: CommandType.StoredProcedure);
```

**Динамический WHERE на стороне C# (альтернатива процедуре):**

```csharp
public IEnumerable<Book> SearchBooks(string? titlePart, int? yearFrom, int? authorId)
{
    var sql = new StringBuilder("SELECT * FROM dbo.Books WHERE 1=1");
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

**Предупреждение:** `DynamicParameters` передаёт значения как параметры ADO.NET — это безопасно. Но имена колонок/таблиц из пользовательского ввода параметризовать нельзя — нужен белый список.

**Нюанс SQL Server:** для кириллицы не указывайте `DbType.AnsiString` — используйте `NVARCHAR` (Dapper ставит `DbType.String` по умолчанию для `string`).

---

## 9. Транзакции: явное управление и типичные ошибки

Dapper не открывает и не закрывает транзакции автоматически. Каждый вызов внутри транзакции должен получать объект `IDbTransaction`.

```csharp
using var connection = new SqlConnection(connectionString);
connection.Open();

using var tx = connection.BeginTransaction();
try
{
    var authorId = connection.ExecuteScalar<int>(
        "INSERT INTO dbo.Authors (Name, Country) OUTPUT INSERTED.Id VALUES (@Name, @Country)",
        new { Name = "Новый Автор", Country = "Италия" },
        tx);

    connection.Execute(
        "INSERT INTO dbo.Books (Title, AuthorId, Year) VALUES (@Title, @AuthorId, @Year)",
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

**Что происходит, если забыть передать `tx`:** команда выполнится вне транзакции, `Rollback()` её не откатит. Это одна из самых частых и самых незаметных ошибок.

**Вложенные транзакции:** ADO.NET не поддерживает вложенные транзакции на уровне `BeginTransaction`. Для композиции используйте `TransactionScope` (с осторожностью в асинхронном коде) или паттерн Unit of Work.

**Нюанс SQL Server:** по умолчанию используется `READ COMMITTED` с блокировками (или `READ_COMMITTED_SNAPSHOT`, если включён на БД). Для отчётных запросов может понадобиться `IsolationLevel.Snapshot`:

```csharp
using var tx = connection.BeginTransaction(IsolationLevel.Snapshot);
```

**Транзакция внутри процедуры:** `dbo.CreateLoan` использует `SET XACT_ABORT ON` и `THROW` — при ошибке вся процедура откатывается автоматически, даже если вызывающий код не открывал явную транзакцию.

---

## 10. Хранимые процедуры, возврат Id и bulk-операции

### 10.1 Хранимые процедуры: входные, выходные и CommandType

Все процедуры уже созданы в разделе 3. Примеры вызова:

```csharp
// Процедура, возвращающая result set (dbo.GetAuthorsByCountry)
var authors = connection.Query<Author>(
    "dbo.GetAuthorsByCountry",
    new { Country = "Россия" },
    commandType: CommandType.StoredProcedure);

// Процедура с OUTPUT-параметром (dbo.CreateAuthor)
var p = new DynamicParameters();
p.Add("@Name", "Процедурный Автор");
p.Add("@Country", "Россия");
p.Add("@NewId", dbType: DbType.Int32, direction: ParameterDirection.Output);

connection.Execute("dbo.CreateAuthor", p, commandType: CommandType.StoredProcedure);
var newId = p.Get<int>("@NewId");

// Процедура с несколькими result set (dbo.GetAuthorWithBooks)
using var multi = connection.QueryMultiple(
    "dbo.GetAuthorWithBooks",
    new { AuthorId = 1 },
    commandType: CommandType.StoredProcedure);
var author = multi.ReadSingle<Author>();
author.Books = multi.Read<Book>().ToList();

// Процедура с OUTPUT-параметром и проверкой строк (dbo.ReturnLoan)
var p2 = new DynamicParameters();
p2.Add("@LoanId", 1);
p2.Add("@ReturnDate", DateTime.UtcNow.Date);
p2.Add("@RowsAffected", dbType: DbType.Int32, direction: ParameterDirection.Output);
connection.Execute("dbo.ReturnLoan", p2, commandType: CommandType.StoredProcedure);
bool returned = p2.Get<int>("@RowsAffected") == 1;

// Процедура-отчёт (dbo.AuthorReport)
var report = connection.Query<AuthorReportRow>(
    "dbo.AuthorReport",
    commandType: CommandType.StoredProcedure).ToList();

// Процедура-отчёт по читателям (dbo.ReaderLoanStats)
var stats = connection.Query<ReaderLoanStatsRow>(
    "dbo.ReaderLoanStats",
    commandType: CommandType.StoredProcedure).ToList();
```

**Ключевой момент** — `commandType: CommandType.StoredProcedure`. Без него Dapper попытается выполнить текст `"dbo.GetAuthorsByCountry"` как обычный SQL и получит синтаксическую ошибку.

**Совет:** в процедурах используйте `SET NOCOUNT ON;` — иначе SQL Server пришлёт лишний result set с количеством затронутых строк, и Dapper может его прочитать как первый результат, сломав `ReadSingle`/`ReadFirst`.

### 10.2 Bulk-вставка: SqlBulkCopy

`Execute` с массивом выполняет N отдельных INSERT. Для тысяч строк это медленно. `SqlBulkCopy` — нативный механизм SQL Server.

```csharp
public static void BulkInsertAuthors(SqlConnection connection, IEnumerable<Author> authors)
{
    var table = new DataTable();
    table.Columns.Add("Name", typeof(string));
    table.Columns.Add("Country", typeof(string));

    foreach (var a in authors)
        table.Rows.Add(a.Name, a.Country);

    using var bulk = new SqlBulkCopy(connection)
    {
        DestinationTableName = "dbo.Authors",
        BatchSize = 5000,
        BulkCopyTimeout = 120
    };
    bulk.ColumnMappings.Add("Name", "Name");
    bulk.ColumnMappings.Add("Country", "Country");
    bulk.WriteToServer(table);
}
```

**Нюанс:** `SqlBulkCopy` не возвращает Id вставленных строк. Если они нужны — `OUTPUT INSERTED.Id` с табличной переменной, `MERGE` с `OUTPUT` или коммерческий `Dapper.Plus`.

### 10.3 Табличные параметры (TVP)

Альтернатива `SqlBulkCopy` для пакетных операций с возможностью JOIN к таблице параметров. Тип `dbo.AuthorType` уже создан в разделе 3.

```csharp
var table = new DataTable();
table.Columns.Add("Name", typeof(string));
table.Columns.Add("Country", typeof(string));
foreach (var a in authors) table.Rows.Add(a.Name, a.Country);

var p = new DynamicParameters();
p.Add("@Authors", table.AsTableValuedParameter("dbo.AuthorType"));

connection.Execute("""
    INSERT INTO dbo.Authors (Name, Country)
    SELECT Name, Country FROM @Authors;
    """, p);
```

TVP поддерживаются `Microsoft.Data.SqlClient` и Dapper «из коробки» через `AsTableValuedParameter`.

---

## 11. Типовые ловушки и как их обойти

| Ситуация | Что происходит | Решение |
|---|---|---|
| `splitOn` колонка не найдена | `Multi-map error: splitOn column 'X' was not found` | Убедиться, что колонка есть в SELECT, часто забывают `a.Id AS AuthorId` |
| JOIN один-ко-многим через multi-mapping | Дублирование родительских объектов | `QueryMultiple` или дедупликация через `GroupBy` |
| `Query<T>` с колонкой `full_name` и свойством `FullName` | Свойство остаётся `null` | `MatchNamesWithUnderscores = true` или псевдоним в SQL |
| `Execute` с массивом на 10 000 объектов | 10 000 отдельных round-trip’ов | Транзакция + `SqlBulkCopy` или TVP |
| Забыли `transaction:` в вызове внутри `BeginTransaction` | Команда вне транзакции, откат не работает | Передавать `tx` в каждый вызов |
| `QuerySingle`, а строк 0 или 2 | Исключение | Осознанно выбирать `QuerySingle` только при гарантии уникальности |
| Процедура без `SET NOCOUNT ON` | Dapper читает лишний result set | Добавить `SET NOCOUNT ON;` в процедуру |
| Кириллица превращается в `?????` | Параметр передан как `AnsiString` | Не указывать `DbType.AnsiString`, использовать `NVARCHAR` |
| `@@IDENTITY` после INSERT с триггером | Возвращает Id из триггера, не из вставки | `SCOPE_IDENTITY()` или `OUTPUT INSERTED.Id` |
| Вызов процедуры без `commandType` | Синтаксическая ошибка | `commandType: CommandType.StoredProcedure` |

---

## 12. Шпаргалка по выбору инструмента

| Задача | Инструмент |
|---|---|
| Простой список объектов | `Query<T>` |
| Одна строка, может отсутствовать | `QueryFirstOrDefault<T>` |
| Гарантированно одна строка | `QuerySingle<T>` |
| Одно скалярное значение | `ExecuteScalar<T>` |
| INSERT/UPDATE/DELETE | `Execute` |
| Возврат Id после INSERT | `OUTPUT INSERTED.Id` (или `SCOPE_IDENTITY()`) |
| JOIN двух таблиц | `Query<T1, T2, TReturn>` + `splitOn` |
| JOIN трёх таблиц | `Query<T1, T2, T3, TReturn>` + `splitOn` с двумя колонками |
| Один-ко-многим | `QueryMultiple` или `FOR JSON PATH` |
| Несколько result set | `QueryMultiple` + `Read<T>` |
| Выходные параметры | `DynamicParameters` + `ParameterDirection.Output` |
| Динамический WHERE | `StringBuilder` + `DynamicParameters` или процедура с `(@Param IS NULL OR ...)` |
| Атомарность | `BeginTransaction` + передача `tx` в каждый вызов |
| Bulk-вставка | `SqlBulkCopy` или TVP |
| Хранимые процедуры | `commandType: CommandType.StoredProcedure` + `SET NOCOUNT ON` |

---

## 13. Тренировочные задания (10 штук, с каркасом без кода)

Задания используют схему SQL Server и процедуры из раздела 3. Для каждого указан ожидаемый результат. Код не приводится — только каркас.

**Задание 1. Execute и ExecuteScalar: возврат Id**
Напишите метод, который вставляет нового автора и возвращает его `Id` через `OUTPUT INSERTED.Id`. Затем вставляет две книги этого автора в одной транзакции и возвращает количество вставленных книг.
*Проверка:* после вызова в БД появляется один автор и две книги, связанные корректным `AuthorId`.

**Задание 2. QueryFirstOrDefault и обработка отсутствия данных**
Напишите метод `GetReaderByEmail(string email)`, возвращающий `Reader?`. Если email `null` или пустой — вернуть `null`, не выполняя запрос. Можно использовать процедуру `dbo.GetReaderByEmail` или прямой SQL.
*Проверка:* существующий email → объект; несуществующий → `null`; `null` → `null` без исключения.

**Задание 3. JOIN двух таблиц с multi-mapping**
Выведите все книги с информацией об авторе (имя и страна). Используйте `Query<Book, Author, Book>` и `splitOn: "AuthorId"`.
*Проверка:* 4 книги, у каждой `Author.Name` не пустое.

**Задание 4. Один-ко-многим через QueryMultiple и процедуру**
Для автора с `Id = 1` получите его данные и список всех его книг одним вызовом `QueryMultiple` с процедурой `dbo.GetAuthorWithBooks`.
*Ожидаемый результат:* Лев Толстой, книги «Война и мир» и «Анна Каренина».

**Задание 5. Агрегация и ExecuteScalar**
Найдите количество книг, написанных авторами из России. Используйте `ExecuteScalar<int>` и JOIN в SQL.
*Ожидаемый результат:* 3.

**Задание 6. Транзакция с откатом**
Напишите метод, который в одной транзакции: 1) вставляет новую книгу; 2) вставляет запись о выдаче; 3) намеренно выбрасывает исключение после второй вставки. Убедитесь, что после исключения ни книга, ни выдача не сохранились.
*Проверка:* `SELECT COUNT(*) FROM dbo.Books` не изменился.

**Задание 7. Вызов процедуры с OUTPUT-параметром**
Вызовите процедуру `dbo.CreateAuthor` через `DynamicParameters`, получите `@NewId` и проверьте, что автор с этим Id существует в БД.
*Проверка:* `QuerySingle<Author>` по полученному Id возвращает объект с ожидаемым именем.

**Задание 8. Динамический поиск**
Реализуйте поиск книг по части названия с помощью процедуры `dbo.SearchBooks`. Параметр `titlePart` может быть `null` — тогда поиск не применяется. Убедитесь, что процедура корректно обрабатывает `null`.
*Проверка:* при `titlePart = "Война"` — одна книга; при `null` — все.

**Задание 9. Вызов процедуры с OUTPUT и проверкой строк**
Вызовите процедуру `dbo.ReturnLoan` для выдачи с `Id = 1` и текущей датой. Получите `@RowsAffected`. Повторный вызов для той же выдачи должен вернуть `0` (потому что `ReturnDate IS NULL` уже не выполняется).
*Проверка:* первый вызов — `rows == 1`; второй — `rows == 0`.

**Задание 10. Отчёт через процедуру**
Вызовите процедуру `dbo.AuthorReport` и выведите список авторов с количеством книг. Затем вызовите `dbo.ReaderLoanStats` и выведите статистику по читателям.
*Ожидаемый результат:* Толстой — 2, Достоевский — 1, Оруэлл — 1, Булгаков — 0 (если добавлен); у Ивана Петрова — 1 выдача (активная), у Марии Сидоровой — 1 выдача (активная).

---

## 14. Итог

Dapper — инструмент, который даёт ровно столько, сколько нужно: выполнение SQL, маппинг, параметры, транзакции, хранимые процедуры. Всё остальное — на вашей стороне. Понимание внутреннего контура (буферизация, кэш маппинга, `splitOn`, поведение `Execute` с коллекциями), диалекта SQL Server (`OUTPUT INSERTED.Id`, `SET NOCOUNT ON`, `NVARCHAR`, `SqlBulkCopy`, TVP) и типовых ловушек отличает код, который «работает», от кода, который «работает предсказуемо и быстро». Практика на заданиях выше закрепит именно эти нюансы.
