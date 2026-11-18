Сравнение Dapper и ADO.NET: инструменты, использование, примеры и измерение производительности

В начале работы — инструменты и методика измерения: Stopwatch для времени и способы замера памяти. Затем — примеры на одном сценарии для ADO.NET и Dapper, чтобы разница была видна напрямую.

---

0. Измерение производительности: Stopwatch и память

Прежде чем сравнивать подходы, нужно уметь честно измерять. Ниже — базовые инструменты и правила, без которых любые «Dapper быстрее» или «ADO.NET экономнее» остаются мнением.

0.1 Stopwatch: замер времени

System.Diagnostics.Stopwatch — высокоточный таймер на основе QueryPerformanceCounter. Базовый шаблон:

```csharp
using System.Diagnostics;

var sw = Stopwatch.StartNew();
// ... измеряемый код ...
sw.Stop();
Console.WriteLine($"Elapsed: {sw.ElapsedMilliseconds} ms ({sw.ElapsedTicks} ticks)");
```

Правила корректного замера:

1. Прогрев. Первый вызов включает JIT-компиляцию, инициализацию пула соединений, кэш маппинга Dapper. Первый замер всегда завышен. Сделайте 1–2 прогона «вхолостую», затем измеряйте.
2. Много итераций. Один прогон ничего не говорит. Замеряйте N итераций (например, 100–1000) и берите среднее или медиану. Для наносекундных операций — миллионы итераций.
3. Одинаковые условия. Одна и та же БД, одна и та же схема, одинаковый объём данных, одинаковый IsolationLevel, одинаковый пул соединений.
4. Отделяйте открытие соединения от выполнения. Если в одном подходе соединение открывается внутри метода, а в другом — снаружи, сравнение нечестное.
5. Учитывайте кэш. Dapper кэширует маппинг после первого вызова; SQL Server кэширует план после первого выполнения. Замеряйте «прогретый» сценарий отдельно от «холодного».

Шаблон замера с прогревом и итерациями:

```csharp
public static (double avgMs, double medianMs) Measure(Action action, int warmup = 3, int iterations = 50)
{
    // Прогрев
    for (int i = 0; i < warmup; i++) action();

    var times = new List<double>(iterations);
    for (int i = 0; i < iterations; i++)
    {
        var sw = Stopwatch.StartNew();
        action();
        sw.Stop();
        times.Add(sw.Elapsed.TotalMilliseconds);
    }

    times.Sort();
    var avg = times.Average();
    var median = times[times.Count / 2];
    return (avg, median);
}
```

Нюанс: Stopwatch.Elapsed имеет разрешение порядка 100 нс на Windows. Для операций короче этого используйте ElapsedTicks и делите на Stopwatch.Frequency, либо увеличивайте число итераций внутри одного замера.

0.2 Замер памяти: GC и рабочий набор

Память измеряется сложнее, чем время, потому что GC работает асинхронно и недетерминированно. Есть несколько уровней.

Уровень 1: выделенная managed-память (GC.GetAllocatedBytesForCurrentThread).

Самый точный способ для сравнения аллокаций в одном потоке. Возвращает суммарное число байт, выделенных потоком с момента старта (или с последнего сброса). Разница до и после — аллокации измеряемого кода.

```csharp
public static long MeasureAllocations(Action action)
{
    // Сброс счётчика для текущего потока (доступно с .NET Core 3.0)
    GC.Collect();
    GC.WaitForPendingFinalizers();
    GC.Collect();

    var before = GC.GetAllocatedBytesForCurrentThread();
    action();
    var after = GC.GetAllocatedBytesForCurrentThread();
    return after - before;
}
```

Важно: GC.GetAllocatedBytesForCurrentThread считает выделения, а не «живую» память. Если объект создан и сразу собран — он всё равно учтён. Это именно то, что нужно для сравнения «сколько мусора создаёт подход».

Уровень 2: общая managed-память (GC.GetTotalMemory).

Показывает, сколько памяти занято managed-кучей в данный момент. Зависит от момента вызова GC, поэтому менее стабилен.

```csharp
GC.Collect();
GC.WaitForPendingFinalizers();
GC.Collect();
var before = GC.GetTotalMemory(forceFullCollection: true);

action();

GC.Collect();
GC.WaitForPendingFinalizers();
GC.Collect();
var after = GC.GetTotalMemory(forceFullCollection: true);
var delta = after - before;
```

Нюанс: GC.GetTotalMemory(true) сам по себе запускает сборку и искажает измерения. Для сравнения подходов он годится, но абсолютные значения малоинформативны.

Уровень 3: счётчики GC (GC.CollectionCount).

Показывает, сколько раз срабатывал GC поколений 0, 1, 2. Косвенный, но полезный показатель: больше аллокаций → больше сборок → больше пауз.

```csharp
var gen0Before = GC.CollectionCount(0);
var gen1Before = GC.CollectionCount(1);
var gen2Before = GC.CollectionCount(2);

action();

var gen0 = GC.CollectionCount(0) - gen0Before;
var gen1 = GC.CollectionCount(1) - gen1Before;
var gen2 = GC.CollectionCount(2) - gen2Before;
Console.WriteLine($"GC: gen0={gen0}, gen1={gen1}, gen2={gen2}");
```

Уровень 4: пиковая память процесса (Process.PeakWorkingSet64).

Показывает пиковый рабочий набор процесса. Полезно для потоковых сценариев, где важно, сколько памяти занимает обработка большого result set’а.

```csharp
using System.Diagnostics;

var process = Process.GetCurrentProcess();
var before = process.PeakWorkingSet64;
action();
process.Refresh();
var after = process.PeakWorkingSet64;
Console.WriteLine($"Peak working set delta: {(after - before) / 1024 / 1024} MB");
```

Нюанс: рабочий набор включает native-память драйвера, буферы сети, JIT-код. Он не изолирует managed-аллокации, но хорошо показывает «сколько всего съел сценарий».

0.3 Что и как сравнивать

Метрика Инструмент Что показывает
Время выполнения Stopwatch Скорость
Аллокации managed GC.GetAllocatedBytesForCurrentThread «Мусор», создаваемый подходом
Живая managed-память GC.GetTotalMemory(true) Занятая куча после сборки
Частота GC GC.CollectionCount(n) Нагрузка на GC
Пиковая память процесса Process.PeakWorkingSet64 Общий пик, включая native
Потоковость buffered: false vs SqlDataReader Память при больших результатах

Правила честного сравнения памяти:

1. Прогрев обязателен. JIT, пул соединений, кэш маппинга — всё это аллоцирует память при первом вызове.
2. GC.Collect() перед замером. Иначе в дельту попадут объекты предыдущих итераций.
3. Замеряйте на одинаковых данных. Разный объём result set’а — разные аллокации.
4. Разделяйте «холодный» и «прогретый» замер. Dapper кэширует маппинг, и второй вызов аллоцирует меньше.
5. Не сравнивайте GC.GetTotalMemory без forceFullCollection — получите шум.

0.4 Полный шаблон сравнения

```csharp
public static void Compare(string name, Action action, int warmup = 3, int iterations = 50)
{
    // Прогрев
    for (int i = 0; i < warmup; i++) action();

    // Время
    var times = new List<double>(iterations);
    for (int i = 0; i < iterations; i++)
    {
        var sw = Stopwatch.StartNew();
        action();
        sw.Stop();
        times.Add(sw.Elapsed.TotalMilliseconds);
    }
    times.Sort();
    var median = times[times.Count / 2];
    var p95 = times[(int)(times.Count * 0.95)];

    // Память (одна итерация после сборки)
    GC.Collect();
    GC.WaitForPendingFinalizers();
    GC.Collect();

    var allocBefore = GC.GetAllocatedBytesForCurrentThread();
    var gen0Before = GC.CollectionCount(0);
    var gen2Before = GC.CollectionCount(2);

    action();

    var allocAfter = GC.GetAllocatedBytesForCurrentThread();
    var gen0 = GC.CollectionCount(0) - gen0Before;
    var gen2 = GC.CollectionCount(2) - gen2Before;

    Console.WriteLine($"{name}: median={median:F2} ms, p95={p95:F2} ms, " +
                      $"alloc={(allocAfter - allocBefore) / 1024.0:F1} KB, " +
                      $"gen0={gen0}, gen2={gen2}");
}
```

Нюанс: замер аллокаций на одной итерации шумный, если операция маленькая. Для стабильности оберните action() в цикл на 100–1000 итераций и делите результат.

---

1. Какие инструменты используются

1.1 ADO.NET

ADO.NET — встроенный в .NET набор API для работы с реляционными БД. Ключевые типы:

Тип Назначение
SqlConnection Соединение с SQL Server
SqlCommand SQL-команда или вызов процедуры
SqlParameter Параметр команды (защита от инъекций)
SqlDataReader Потоковое чтение результата построчно
SqlDataAdapter / DataTable Отсоединённая модель, загрузка результата в таблицу
SqlTransaction Явная транзакция
SqlBulkCopy Массовая вставка

ADO.NET даёт полный контроль над каждым шагом: открытие соединения, создание команды, добавление параметров, чтение результата, маппинг вручную, закрытие. Максимально гибко и максимально многословно.

Пакет NuGet:

```bash
dotnet add package Microsoft.Data.SqlClient
```

1.2 Dapper

Dapper — микро-ORM, надстройка над ADO.NET. Он не заменяет ADO.NET, а оборачивает его: вы по-прежнему пишете SQL, но Dapper берёт на себя создание команды, добавление параметров, чтение SqlDataReader и маппинг строк в объекты.

Пакеты NuGet:

```bash
dotnet add package Dapper
dotnet add package Microsoft.Data.SqlClient
```

Dapper не работает без ADO.NET-драйвера: он вызывает те же IDbConnection, IDbCommand, IDataReader, что и вы вручную, но прячет рутину.

1.3 Общая база данных для примеров

```sql
IF DB_ID('DapperVsAdoDemo') IS NULL
    CREATE DATABASE DapperVsAdoDemo;
GO

USE DapperVsAdoDemo;
GO

IF OBJECT_ID('dbo.Loans', 'U') IS NOT NULL DROP TABLE dbo.Loans;
IF OBJECT_ID('dbo.Books', 'U') IS NOT NULL DROP TABLE dbo.Books;
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

CREATE TABLE dbo.Loans (
    Id         INT IDENTITY(1,1) PRIMARY KEY,
    BookId     INT NOT NULL,
    LoanDate   DATE NOT NULL,
    ReturnDate DATE NULL,
    CONSTRAINT FK_Loans_Books FOREIGN KEY (BookId) REFERENCES dbo.Books(Id)
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

INSERT INTO dbo.Loans (BookId, LoanDate) VALUES
    (1, '2025-01-10'),
    (3, '2025-01-15');
GO
```

Строка подключения:

```csharp
const string connectionString =
    "Server=(localdb)\\MSSQLLocalDB;Database=DapperVsAdoDemo;Trusted_Connection=True;TrustServerCertificate=True;";
```

1.4 Модели для маппинга

```csharp
public class Author
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public string? Country { get; set; }
}

public class Book
{
    public int Id { get; set; }
    public string Title { get; set; } = "";
    public int AuthorId { get; set; }
    public int? Year { get; set; }
    public Author? Author { get; set; }
}
```

---

2. Как использовать ADO.NET

2.1 SELECT с маппингом вручную

```csharp
using Microsoft.Data.SqlClient;

public static List<Author> GetAuthorsByCountry(string country)
{
    var result = new List<Author>();

    using var connection = new SqlConnection(connectionString);
    using var command = new SqlCommand(
        "SELECT Id, Name, Country FROM dbo.Authors WHERE Country = @Country",
        connection);

    command.Parameters.Add(new SqlParameter("@Country", System.Data.SqlDbType.NVarChar, 100)
    {
        Value = country
    });

    connection.Open();
    using var reader = command.ExecuteReader();

    while (reader.Read())
    {
        result.Add(new Author
        {
            Id = reader.GetInt32(reader.GetOrdinal("Id")),
            Name = reader.GetString(reader.GetOrdinal("Name")),
            Country = reader.IsDBNull(reader.GetOrdinal("Country"))
                ? null
                : reader.GetString(reader.GetOrdinal("Country"))
        });
    }

    return result;
}
```

Что здесь важно:

· SqlParameter с явным SqlDbType.NVarChar и размером — корректная работа с кириллицей.
· reader.GetOrdinal("Id") — поиск индекса колонки по имени. В цикле лучше кэшировать индексы вне цикла.
· reader.IsDBNull(...) — обязательная проверка для nullable-колонок.
· using на соединении, команде и reader’е — обязательное освобождение ресурсов.

2.2 INSERT с возвратом Id

```csharp
public static int CreateAuthor(string name, string? country)
{
    using var connection = new SqlConnection(connectionString);
    using var command = new SqlCommand(
        "INSERT INTO dbo.Authors (Name, Country) OUTPUT INSERTED.Id VALUES (@Name, @Country)",
        connection);

    command.Parameters.Add(new SqlParameter("@Name", System.Data.SqlDbType.NVarChar, 200) { Value = name });
    command.Parameters.Add(new SqlParameter("@Country", System.Data.SqlDbType.NVarChar, 100)
    {
        Value = (object?)country ?? DBNull.Value
    });

    connection.Open();
    return (int)command.ExecuteScalar();
}
```

Нюанс: для null нужно передавать DBNull.Value, иначе ADO.NET бросит исключение.

2.3 UPDATE с проверкой количества строк

```csharp
public static bool UpdateAuthorCountry(int authorId, string country)
{
    using var connection = new SqlConnection(connectionString);
    using var command = new SqlCommand(
        "UPDATE dbo.Authors SET Country = @Country WHERE Id = @Id",
        connection);

    command.Parameters.Add(new SqlParameter("@Country", System.Data.SqlDbType.NVarChar, 100) { Value = country });
    command.Parameters.Add(new SqlParameter("@Id", System.Data.SqlDbType.Int) { Value = authorId });

    connection.Open();
    var affected = command.ExecuteNonQuery();
    return affected == 1;
}
```

2.4 JOIN с ручным маппингом двух объектов

```csharp
public static List<Book> GetBooksWithAuthors()
{
    var result = new List<Book>();

    using var connection = new SqlConnection(connectionString);
    using var command = new SqlCommand("""
        SELECT 
            b.Id, b.Title, b.AuthorId, b.Year,
            a.Id AS Author_Id, a.Name AS Author_Name, a.Country AS Author_Country
        FROM dbo.Books b
        JOIN dbo.Authors a ON a.Id = b.AuthorId
        """, connection);

    connection.Open();
    using var reader = command.ExecuteReader();

    while (reader.Read())
    {
        result.Add(new Book
        {
            Id = reader.GetInt32(reader.GetOrdinal("Id")),
            Title = reader.GetString(reader.GetOrdinal("Title")),
            AuthorId = reader.GetInt32(reader.GetOrdinal("AuthorId")),
            Year = reader.IsDBNull(reader.GetOrdinal("Year")) ? null : reader.GetInt32(reader.GetOrdinal("Year")),
            Author = new Author
            {
                Id = reader.GetInt32(reader.GetOrdinal("Author_Id")),
                Name = reader.GetString(reader.GetOrdinal("Author_Name")),
                Country = reader.IsDBNull(reader.GetOrdinal("Author_Country"))
                    ? null
                    : reader.GetString(reader.GetOrdinal("Author_Country"))
            }
        });
    }

    return result;
}
```

Что здесь важно:

· Колонки двух таблиц явно псевдонимированы (Author_Id), иначе Id перезапишется.
· Каждое поле читается вручную — при изменении схемы нужно править и SQL, и маппинг.

2.5 Транзакция

```csharp
public static void CreateBookWithLoan(string title, int authorId, int year)
{
    using var connection = new SqlConnection(connectionString);
    connection.Open();
    using var transaction = connection.BeginTransaction();

    try
    {
        using var cmdBook = new SqlCommand(
            "INSERT INTO dbo.Books (Title, AuthorId, Year) OUTPUT INSERTED.Id VALUES (@Title, @AuthorId, @Year)",
            connection, transaction);
        cmdBook.Parameters.Add(new SqlParameter("@Title", System.Data.SqlDbType.NVarChar, 300) { Value = title });
        cmdBook.Parameters.Add(new SqlParameter("@AuthorId", System.Data.SqlDbType.Int) { Value = authorId });
        cmdBook.Parameters.Add(new SqlParameter("@Year", System.Data.SqlDbType.Int) { Value = (object?)year ?? DBNull.Value });
        var bookId = (int)cmdBook.ExecuteScalar();

        using var cmdLoan = new SqlCommand(
            "INSERT INTO dbo.Loans (BookId, LoanDate) VALUES (@BookId, @LoanDate)",
            connection, transaction);
        cmdLoan.Parameters.Add(new SqlParameter("@BookId", System.Data.SqlDbType.Int) { Value = bookId });
        cmdLoan.Parameters.Add(new SqlParameter("@LoanDate", System.Data.SqlDbType.Date) { Value = DateTime.UtcNow.Date });
        cmdLoan.ExecuteNonQuery();

        transaction.Commit();
    }
    catch
    {
        transaction.Rollback();
        throw;
    }
}
```

Нюанс: каждый SqlCommand внутри транзакции должен получать её в конструкторе (connection, transaction).

2.6 Пакетная вставка

```csharp
public static void BulkInsertAuthors(IEnumerable<Author> authors)
{
    using var connection = new SqlConnection(connectionString);
    connection.Open();

    var table = new System.Data.DataTable();
    table.Columns.Add("Name", typeof(string));
    table.Columns.Add("Country", typeof(string));
    foreach (var a in authors)
        table.Rows.Add(a.Name, a.Country);

    using var bulk = new SqlBulkCopy(connection)
    {
        DestinationTableName = "dbo.Authors",
        BatchSize = 5000
    };
    bulk.ColumnMappings.Add("Name", "Name");
    bulk.ColumnMappings.Add("Country", "Country");
    bulk.WriteToServer(table);
}
```

---

3. Как использовать Dapper

3.1 SELECT с автоматическим маппингом

```csharp
using Dapper;

public static List<Author> GetAuthorsByCountry(string country)
{
    using var connection = new SqlConnection(connectionString);
    return connection.Query<Author>(
        "SELECT Id, Name, Country FROM dbo.Authors WHERE Country = @Country",
        new { Country = country }).ToList();
}
```

Что изменилось:

· Нет SqlCommand, нет SqlParameter, нет SqlDataReader.
· Параметр передаётся анонимным объектом — Dapper сам создаёт SqlParameter с корректным типом.
· Маппинг автоматический, IsDBNull обрабатывается автоматически.

3.2 INSERT с возвратом Id

```csharp
public static int CreateAuthor(string name, string? country)
{
    using var connection = new SqlConnection(connectionString);
    return connection.ExecuteScalar<int>(
        "INSERT INTO dbo.Authors (Name, Country) OUTPUT INSERTED.Id VALUES (@Name, @Country)",
        new { Name = name, Country = country });
}
```

Нюанс: null в анонимном объекте Dapper сам превращает в DBNull.Value.

3.3 UPDATE с проверкой количества строк

```csharp
public static bool UpdateAuthorCountry(int authorId, string country)
{
    using var connection = new SqlConnection(connectionString);
    var affected = connection.Execute(
        "UPDATE dbo.Authors SET Country = @Country WHERE Id = @Id",
        new { Country = country, Id = authorId });
    return affected == 1;
}
```

3.4 JOIN с multi-mapping

```csharp
public static List<Book> GetBooksWithAuthors()
{
    using var connection = new SqlConnection(connectionString);
    var sql = """
        SELECT 
            b.Id, b.Title, b.AuthorId, b.Year,
            a.Id AS AuthorId2, a.Name, a.Country
        FROM dbo.Books b
        JOIN dbo.Authors a ON a.Id = b.AuthorId
        """;

    return connection.Query<Book, Author, Book>(
        sql,
        (book, author) => { book.Author = author; return book; },
        splitOn: "AuthorId2").ToList();
}
```

Что изменилось:

· Маппинг двух объектов — один вызов с лямбдой.
· splitOn указывает, с какой колонки начинается второй объект. Колонка-граница должна быть явно псевдонимирована (AuthorId2), иначе Dapper возьмёт первую Id и сломает маппинг.

3.5 Транзакция

```csharp
public static void CreateBookWithLoan(string title, int authorId, int year)
{
    using var connection = new SqlConnection(connectionString);
    connection.Open();
    using var transaction = connection.BeginTransaction();

    try
    {
        var bookId = connection.ExecuteScalar<int>(
            "INSERT INTO dbo.Books (Title, AuthorId, Year) OUTPUT INSERTED.Id VALUES (@Title, @AuthorId, @Year)",
            new { Title = title, AuthorId = authorId, Year = year },
            transaction);

        connection.Execute(
            "INSERT INTO dbo.Loans (BookId, LoanDate) VALUES (@BookId, @LoanDate)",
            new { BookId = bookId, LoanDate = DateTime.UtcNow.Date },
            transaction);

        transaction.Commit();
    }
    catch
    {
        transaction.Rollback();
        throw;
    }
}
```

Нюанс: объект transaction передаётся третьим аргументом в каждый вызов.

3.6 Пакетная вставка

```csharp
public static void BulkInsertAuthors(IEnumerable<Author> authors)
{
    using var connection = new SqlConnection(connectionString);
    connection.Open();

    // Dapper выполнит INSERT для каждого элемента — это НЕ SqlBulkCopy
    connection.Execute(
        "INSERT INTO dbo.Authors (Name, Country) VALUES (@Name, @Country)",
        authors);
}
```

Важное ограничение: Execute с коллекцией — это N отдельных round-trip’ов. Для тысяч строк это медленно. Быстрый bulk в Dapper-мире — SqlBulkCopy, TVP или Dapper.Plus.

---

4. Прямое сравнение на одном сценарии

4.1 SELECT списка

Шаг ADO.NET Dapper
Создать соединение new SqlConnection(...) new SqlConnection(...)
Создать команду new SqlCommand(sql, connection) —
Добавить параметры command.Parameters.Add(new SqlParameter(...)) для каждого Анонимный объект new { Country = country }
Открыть соединение connection.Open() Dapper открывает сам, если закрыто
Выполнить command.ExecuteReader() connection.Query<Author>(...)
Прочитать while (reader.Read()) { ... } —
Маппинг Вручную по колонкам, с IsDBNull Автоматически
Закрыть reader using —
Строк кода (примерно) 20–25 3–5

4.2 INSERT с возвратом Id

Шаг ADO.NET Dapper
Команда + параметры SqlCommand + SqlParameter с DBNull.Value для null Анонимный объект
Выполнение command.ExecuteScalar() connection.ExecuteScalar<int>(...)
Обработка null Вручную Автоматически
Строк кода (примерно) 8–10 2–3

4.3 JOIN двух таблиц

Шаг ADO.NET Dapper
SQL Тот же Тот же
Псевдонимы Обязательны для избежания коллизий Обязательны, особенно для splitOn
Маппинг Вручную для каждого объекта Лямбда + splitOn
Обработка IsDBNull Вручную для каждого nullable-поля Автоматически
Строк кода (примерно) 30–40 8–12

4.4 Транзакция

Шаг ADO.NET Dapper
Создание транзакции connection.BeginTransaction() То же
Передача в команду В конструктор SqlCommand(..., transaction) Третьим аргументом в каждый вызов
Commit/Rollback Вручную Вручную
Строк кода (примерно) 25–30 15–20

---

5. Замер производительности на практике

5.1 Время: SELECT списка

```csharp
Compare("ADO.NET SELECT", () => GetAuthorsByCountryAdo("Россия"));
Compare("Dapper SELECT", () => GetAuthorsByCountryDapper("Россия"));
```

Ожидаемый результат: на маленьких результатах разница в пределах шума. На больших (тысячи строк) Dapper может быть немного быстрее за счёт кэша маппинга после прогрева, но узкое место — сеть и SQL Server.

5.2 Время: INSERT с возвратом Id

```csharp
Compare("ADO.NET INSERT", () => CreateAuthorAdo("Тест", "Россия"));
Compare("Dapper INSERT", () => CreateAuthorDapper("Тест", "Россия"));
```

Ожидаемый результат: сопоставимо. Основное время — round-trip к SQL Server.

5.3 Память: аллокации на SELECT

```csharp
var adoAlloc = MeasureAllocations(() => GetAuthorsByCountryAdo("Россия"));
var dapperAlloc = MeasureAllocations(() => GetAuthorsByCountryDapper("Россия"));
Console.WriteLine($"ADO.NET alloc: {adoAlloc} bytes");
Console.WriteLine($"Dapper alloc: {dapperAlloc} bytes");
```

Ожидаемый результат: ADO.NET аллоцирует SqlCommand, SqlParameter[], List<Author>, объекты Author. Dapper дополнительно аллоцирует лямбды, CommandDefinition, но после прогрева кэш маппинга снижает аллокации. На маленьких результатах разница невелика; на больших Dapper обычно аллоцирует меньше за счёт оптимизированного чтения.

5.4 Память: потоковое чтение большого результата

```csharp
// ADO.NET: SqlDataReader, без буферизации
var adoPeak = MeasurePeakMemory(() =>
{
    using var connection = new SqlConnection(connectionString);
    using var command = new SqlCommand("SELECT * FROM dbo.Books", connection);
    connection.Open();
    using var reader = command.ExecuteReader();
    while (reader.Read()) { /* обработка */ }
});

// Dapper: buffered: false
var dapperPeak = MeasurePeakMemory(() =>
{
    using var connection = new SqlConnection(connectionString);
    foreach (var book in connection.Query<Book>("SELECT * FROM dbo.Books", buffered: false))
    {
        // обработка
    }
});
```

Ожидаемый результат: оба подхода потоковые, пиковая память низкая. Если использовать Query<Book> без buffered: false, Dapper буферизует весь результат — пиковая память вырастет пропорционально числу строк.

5.5 Память: пакетная вставка

```csharp
var authors = Enumerable.Range(1, 1000)
    .Select(i => new Author { Name = $"Автор {i}", Country = "Россия" })
    .ToList();

Compare("ADO.NET SqlBulkCopy", () => BulkInsertAuthorsAdo(authors));
Compare("Dapper Execute collection", () => BulkInsertAuthorsDapper(authors));
```

Ожидаемый результат: SqlBulkCopy на порядки быстрее и аллоцирует меньше (один DataTable). Execute с коллекцией делает 1000 round-trip’ов и аллоцирует CommandDefinition на каждую итерацию.

5.6 Интерпретация результатов

· Время. Dapper и ADO.NET сопоставимы на простых запросах. Разница проявляется на сложных JOIN’ах и больших результатах — там Dapper экономит код, но не «магически ускоряет» выполнение.
· Память. ADO.NET с SqlDataReader — минимальные аллокации при потоковой обработке. Dapper с buffered: false — близко. Dapper с буферизацией — аллоцирует под весь результат. SqlBulkCopy — минимум аллокаций на bulk-вставке.
· GC. Больше аллокаций → больше сборок поколения 0. На маленьких запросах разница незначительна; на циклах в тысячи вызовов Dapper с кэшем маппинга обычно даёт меньше сборок, чем ручной ADO.NET с GetOrdinal в цикле.

---

6. Когда что выбирать

ADO.NET уместен, когда:

· Нужен полный контроль над каждым шагом выполнения.
· Результат очень большой и должен обрабатываться потоково без буферизации.
· Нужны специализированные механизмы: SqlBulkCopy, TVP, SqlDataAdapter, DataSet.
· Проект не хочет добавлять зависимости.
· Нужна тонкая настройка таймаутов, isolation level, CommandBehavior.

Dapper уместен, когда:

· Нужен SQL-контроль, но не нужна рутина маппинга.
· Много JOIN’ов и сложных выборок, где ручной маппинг занимает десятки строк.
· Важна читаемость кода и скорость разработки.
· Команда уже знает SQL и не хочет ORM, генерирующий запросы.

На практике Dapper и ADO.NET часто сосуществуют: Dapper — для 90% запросов, ADO.NET — для SqlBulkCopy, TVP и особых случаев.

---

7. Итоговая таблица

Критерий ADO.NET Dapper
Зависимости Только драйвер Драйвер + Dapper
Контроль над выполнением Полный Полный (Dapper не скрывает SQL)
Ручной маппинг Да, всегда Нет, автоматически
Обработка DBNull Вручную Автоматически
Multi-mapping (JOIN) Вручную Query<T1, T2, TReturn> + splitOn
Параметры SqlParameter явно Анонимный объект / DynamicParameters
Транзакции Явно Явно, но короче
Bulk-вставка SqlBulkCopy Нужен SqlBulkCopy или Dapper.Plus
Потоковое чтение SqlDataReader Query<T>(buffered: false)
Кэш маппинга Нет Есть
Строк кода на запрос Много Мало
Порог входа Средний Низкий (если знаете SQL)
Производительность Базовая Сопоставимая
Измерение времени Stopwatch Stopwatch
Измерение памяти GC.GetAllocatedBytesForCurrentThread, Process.PeakWorkingSet64 То же

---

8. Тренировочные задания

Задания используют схему из раздела 1.3. Для каждого нужно реализовать решение двумя способами — на ADO.NET и на Dapper — и сравнить время (Stopwatch) и память (GC.GetAllocatedBytesForCurrentThread, GC.CollectionCount).

Задание 1. SELECT с фильтром
Получить всех авторов заданной страны. Реализовать на ADO.NET (с SqlDataReader и ручным маппингом) и на Dapper (Query<Author>). Замерить время и аллокации. Сравнить количество строк кода.

Задание 2. INSERT с возвратом Id
Добавить автора и вернуть его Id. В ADO.NET использовать OUTPUT INSERTED.Id и ExecuteScalar, в Dapper — ExecuteScalar<int>. Замерить время и аллокации. Обратить внимание на обработку null для Country.

Задание 3. UPDATE с проверкой строк
Обновить страну автора по Id. Вернуть true, если обновлена ровно одна строка. Замерить время и аллокации. Сравнить обработку null.

Задание 4. JOIN двух таблиц
Получить книги с авторами. В ADO.NET — ручной маппинг с псевдонимами колонок. В Dapper — Query<Book, Author, Book> с splitOn. Замерить время и аллокации. Сравнить, сколько мест нужно править при добавлении новой колонки.

Задание 5. Транзакция
Вставить книгу и выдачу в одной транзакции. В ADO.NET передавать transaction в конструктор SqlCommand, в Dapper — третьим аргументом. Проверить, что при исключении обе вставки откатываются. Замерить время.

Задание 6. Пакетная вставка
Вставить 1000 авторов. Сравнить три варианта: цикл ExecuteNonQuery в ADO.NET, Execute с коллекцией в Dapper и SqlBulkCopy. Замерить время и аллокации через Stopwatch и GC.GetAllocatedBytesForCurrentThread. Объяснить разницу.

Задание 7. Обработка DBNull
Получить всех авторов, у которых Country может быть NULL. В ADO.NET использовать IsDBNull, в Dapper — nullable-свойство. Сравнить, что произойдёт, если в ADO.NET забыть проверку. Замерить аллокации.

Задание 8. Потоковое чтение
Получить 100 000 книг. В ADO.NET — SqlDataReader с обработкой по одной строке. В Dapper — Query<Book>(buffered: false). Сравнить пиковую память (Process.PeakWorkingSet64) и время.

Задание 9. GC-нагрузка
Выполнить 1000 SELECT-запросов подряд каждым подходом. Замерить GC.CollectionCount(0), GC.CollectionCount(2) до и после. Объяснить, какой подход создаёт больше нагрузки на GC и почему.

Задание 10. Выбор инструмента
Для каждого из пяти сценариев (простой SELECT, JOIN, bulk-вставка, TVP, отчёт с агрегацией) выбрать ADO.NET или Dapper и обосновать выбор в одном абзаце. Опираться на критерии из раздела 6 и результаты замеров.

---

9. Вывод

ADO.NET — фундамент, на котором стоит Dapper. Dapper не заменяет ADO.NET, а сокращает рутину: параметры, маппинг, DBNull, multi-mapping. ADO.NET остаётся незаменимым для SqlBulkCopy, TVP, потокового чтения и тонкой настройки выполнения. Чтобы сравнивать их честно, нужны Stopwatch с прогревом и итерациями, GC.GetAllocatedBytesForCurrentThread для аллокаций, GC.CollectionCount для нагрузки на GC и Process.PeakWorkingSet64 для пиковой памяти. Понимание обоих уровней и корректная методика измерений — условие предсказуемой и быстрой работы с SQL Server.
