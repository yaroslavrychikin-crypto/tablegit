# Entity Framework Core: полное руководство по асинхронной работе, EF Core и практика

Entity Framework Core (EF Core) — это не просто ORM, а полноценный слой абстракции над ADO.NET, который скрывает от разработчика управление соединениями, транзакциями и маппингом данных. Однако эта абстракция порождает собственный набор «ловушек», которые проявляются именно в асинхронном коде и при работе с `DbContext`. Ниже — исчерпывающая теория, рабочие примеры с подробными комментариями, скрипт БД и тренировочные задания.


## 1. Фундаментальные принципы асинхронности в EF Core

### 1.1. Что именно становится асинхронным

В EF Core асинхронными являются только те операции, которые **фактически отправляют запрос в базу данных** или **выполняют ввод-вывод**. Это:

- `ToListAsync()`, `ToArrayAsync()`, `ToDictionaryAsync()`
- `SingleAsync()`, `SingleOrDefaultAsync()`, `FirstAsync()`, `FirstOrDefaultAsync()`
- `CountAsync()`, `AnyAsync()`, `AllAsync()`, `SumAsync()`, `AverageAsync()`, `MaxAsync()`, `MinAsync()`
- `SaveChangesAsync()`
- `AsAsyncEnumerable()` — для потокового чтения
- `ExecuteUpdateAsync()` / `ExecuteDeleteAsync()` (EF Core 7+) — для массовых операций без загрузки сущностей

**Не являются асинхронными** (и не имеют асинхронных версий) операторы, которые только **строят дерево выражений**, но не выполняют его: `Where()`, `OrderBy()`, `Select()`, `Include()`, `ThenInclude()`, `Skip()`, `Take()`. Эти методы лишь модифицируют `IQueryable`, и их вызов не блокирует поток, потому что SQL ещё не отправлен .

### 1.2. Ключевое ограничение: один контекст — одна операция

**EF Core не поддерживает параллельное выполнение нескольких операций на одном экземпляре `DbContext`.** Это не рекомендация, а жёсткое ограничение, за нарушение которого вы получите `InvalidOperationException` с сообщением: *«A second operation was started on this context instance before a previous operation completed»*.

Правило простое: если вы начали асинхронную операцию на контексте, **дождитесь её завершения** (`await`) прежде чем начинать следующую. Если вам нужно выполнять несколько запросов параллельно — создавайте **отдельный `DbContext` для каждой параллельной ветки** (например, через `IDbContextFactory<T>` в ASP.NET Core) .


## 2. `DbContext` и его жизненный цикл

`DbContext` — это **не потокобезопасный** объект. Он представляет собой единицу работы (Unit of Work) и содержит внутри себя:
- Отслеживание изменений (Change Tracker) — кэш загруженных сущностей
- Состояние соединения (открыто/закрыто)
- Кэш первого уровня (Identity Map)

### 2.1. Время жизни в DI-контейнере

В ASP.NET Core `DbContext` обычно регистрируется как **Scoped** — один экземпляр на HTTP-запрос. Это означает:
- В рамках одного запроса все репозитории и сервисы используют **один и тот же** `DbContext`.
- Между запросами контекст создаётся заново.
- Параллельные запросы **не должны** использовать один `DbContext`.

Для фоновых задач (BackgroundService, Hangfire, Quartz) используйте **`IDbContextFactory<T>`**, чтобы создавать контекст вручную на каждую единицу работы, а не внедрять Scoped-зависимость в Singleton-сервис.

### 2.2. Почему «живой» контекст опасен

Если держать `DbContext` открытым слишком долго (например, на всё время жизни Singleton-сервиса), Change Tracker накапливает все загруженные сущности, что приводит к:
- Росту памяти
- Замедлению `SaveChangesAsync` (EF Core проверяет все отслеживаемые сущности)
- Устареванию данных (кеш первого уровня не видит изменений, сделанных другими контекстами)

**Правило:** контекст должен жить ровно столько, сколько длится одна логическая операция (запрос, команда, транзакция).


## 3. Асинхронные LINQ-операторы: полный разбор

### 3.1. Буферизация против потокового чтения

Это, пожалуй, самое важное архитектурное решение при работе с EF Core.

**Буферизация (`ToListAsync`)** — EF Core загружает **все** строки результата в память, создаёт объекты, заполняет Change Tracker, и только потом возвращает `List<T>`. Память растёт линейно с количеством строк. Для маленьких выборок (десятки/сотни строк) это нормально и часто удобно.

**Потоковое чтение (`AsAsyncEnumerable`)** — EF Core возвращает `IAsyncEnumerable<T>`, и вы получаете по одной сущности за раз. Память остаётся **константной** независимо от размера результата. Change Tracker **не заполняется** (если не включено отслеживание), что критически важно для чтения больших объёмов данных .

**Когда что использовать:**
- `ToListAsync()` — если данные нужны для дальнейшей обработки в памяти (сортировка на клиенте, группировка, несколько проходов).
- `AsAsyncEnumerable()` — если данные обрабатываются потоково (запись в файл, отправка в очередь, агрегация на лету) и не требуют повторного чтения .

### 3.2. Клиентские операторы после `AsAsyncEnumerable`

После `AsAsyncEnumerable()` вы можете продолжать строить LINQ-запрос, но все последующие операторы будут выполняться **на клиенте** (в .NET), а не транслироваться в SQL. Это полезно, когда нужна логика, которую EF Core не умеет транслировать (например, вызов локального метода) .

**Важно:** для LINQ-операторов над `IAsyncEnumerable<T>` (например, `Where`, `Select`, `AnyAsync` после `AsAsyncEnumerable`) в .NET до 10 требуется пакет `System.Linq.Async`. В .NET 10 эти операторы встроены .

### 3.3. Конфликт имён `AsAsyncEnumerable` и `AsQueryable`

`DbSet<T>` реализует одновременно `IQueryable<T>` и `IAsyncEnumerable<T>`. Если вы импортировали и `System.Linq`, и `Microsoft.EntityFrameworkCore`, компилятор может выдать ошибку неоднозначности при вызове `AnyAsync()` на `DbSet`. Решение — явно указать, какой интерфейс использовать :

```csharp
// Вариант 1: выполнить на сервере (транслируется в SQL)
await _db.Users.AsQueryable().AnyAsync(u => u.Name == name);

// Вариант 2: выполнить на клиенте (загрузит ВСЕХ Users в память!)
await _db.Users.AsAsyncEnumerable().AnyAsync(u => u.Name == name);
```

**Критическое предупреждение:** вариант с `AsAsyncEnumerable()` перед `AnyAsync()` загрузит **всю таблицу** в память и выполнит проверку на клиенте. Это почти всегда ошибка .


## 4. Отслеживание изменений (Change Tracking) и его влияние на асинхронность

По умолчанию EF Core **отслеживает** все сущности, возвращённые запросом. Это означает:
- Каждая загруженная сущность сохраняется в Change Tracker (память)
- При `SaveChangesAsync` EF Core сравнивает текущие значения с исходными
- Для больших выборок это дорого

**`AsNoTracking()`** отключает отслеживание для конкретного запроса. Это:
- Уменьшает потребление памяти
- Ускоряет чтение (нет накладных расходов на создание записей в Change Tracker)
- **Не позволяет** обновлять сущности через `SaveChanges` (EF Core не знает об изменениях)

**Правило:** используйте `AsNoTracking()` для **всех операций чтения**, где вы не планируете изменять сущности. Для обновлений — либо загружайте с отслеживанием, либо используйте `ExecuteUpdateAsync()` (EF Core 7+), который не загружает сущности вообще.

**`AsNoTrackingWithIdentityResolution()`** — компромисс: отключает отслеживание изменений, но сохраняет Identity Map (один и тот же объект для одной и той же строки в рамках запроса). Полезно, если в результате есть дубликаты по навигационным свойствам.


## 5. Транзакции в EF Core

### 5.1. Неявные транзакции

`SaveChangesAsync()` **автоматически** оборачивает все изменения в транзакцию. Если вы изменили 10 сущностей и вызвали `SaveChangesAsync` один раз — все 10 изменений будут либо применены, либо откачены вместе. Это поведение по умолчанию и его достаточно для большинства сценариев.

### 5.2. Явные транзакции

Если нужно объединить несколько вызовов `SaveChangesAsync` или смешать EF Core с сырым SQL (ADO.NET), используйте `BeginTransactionAsync()`:

```csharp
await using var transaction = await _context.Database.BeginTransactionAsync(token);
try
{
    // Несколько SaveChangesAsync
    await _context.SaveChangesAsync(token);
    // Сырой SQL в той же транзакции
    await _context.Database.ExecuteSqlRawAsync("UPDATE ...", token);
    await transaction.CommitAsync(token);
}
catch
{
    await transaction.RollbackAsync(CancellationToken.None);
    throw;
}
```

**Нюанс:** `RollbackAsync` должен использовать **чистый токен** (`CancellationToken.None`), потому что исходный токен может быть уже отменён. Если передать отменённый токен, откат упадёт с `OperationCanceledException`, и транзакция останется открытой .

### 5.3. Транзакции и `SaveChangesAsync` внутри

Если вы уже открыли явную транзакцию через `BeginTransactionAsync`, то `SaveChangesAsync` **не создаёт собственную** транзакцию, а присоединяется к текущей. Это позволяет безопасно комбинировать несколько сохранений.


## 6. Известные проблемы асинхронности в EF Core

### 6.1. Проблемы с `Microsoft.Data.SqlClient`

Документация EF Core прямо предупреждает: асинхронная реализация `Microsoft.Data.SqlClient` имеет известные проблемы (issues #593, #601 и другие). Если вы наблюдаете неожиданные проблемы с производительностью при работе с **большими текстовыми или бинарными значениями**, рекомендуется попробовать **синхронное** выполнение команд в этих конкретных сценариях .

### 6.2. `CancellationToken` не всегда соблюдается

EF Core передаёт `CancellationToken` в провайдер базы данных (например, `Microsoft.Data.SqlClient`), но **провайдер может его игнорировать** или соблюдать не полностью. Не полагайтесь на то, что отмена сработает мгновенно — всегда проверяйте документацию конкретного провайдера .


## 7. База данных для примеров

### 7.0
Потребуется скачать через nuget:
```c#
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Infrastructure;
using Microsoft.EntityFrameworkCore.SqlServer;
```

Для NET 8 -> 8 версию
Для NET 9 -> 9 версию
Для NET 10 -> 10 версию

### 7.1. Скрипт создания БД

```sql
IF DB_ID('EfCoreTestDb') IS NOT NULL
BEGIN
    ALTER DATABASE EfCoreTestDb SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
    DROP DATABASE EfCoreTestDb;
END
GO
CREATE DATABASE EfCoreTestDb;
GO
USE EfCoreTestDb;
GO

-- Авторы
CREATE TABLE dbo.Authors
(
    Id        INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    Name      NVARCHAR(100)     NOT NULL,
    Country   NVARCHAR(50)      NULL,
    BirthDate DATE              NULL
);
GO

-- Книги (связь 1:N с авторами)
CREATE TABLE dbo.Books
(
    Id          INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    Title       NVARCHAR(200)     NOT NULL,
    AuthorId    INT               NOT NULL
        CONSTRAINT FK_Books_Authors REFERENCES dbo.Authors(Id),
    Price       DECIMAL(18,2)     NOT NULL,
    PublishedAt DATE              NOT NULL,
    Content     NVARCHAR(MAX)     NULL  -- для демонстрации больших полей
);
GO

CREATE INDEX IX_Books_AuthorId ON dbo.Books(AuthorId);
GO

-- Наполнение
INSERT INTO dbo.Authors (Name, Country, BirthDate) VALUES
(N'Лев Толстой', N'Россия', '1828-09-09'),
(N'Фёдор Достоевский', N'Россия', '1821-11-11'),
(N'Джордж Оруэлл', N'Великобритания', '1903-06-25'),
(N'Эрнест Хемингуэй', N'США', '1899-07-21'),
(N'Габриэль Гарсиа Маркес', N'Колумбия', '1927-03-06');
GO

INSERT INTO dbo.Books (Title, AuthorId, Price, PublishedAt, Content) VALUES
(N'Война и мир', 1, 1500.00, '1869-01-01', REPLICATE('A', 10000)),
(N'Анна Каренина', 1, 1200.00, '1878-01-01', NULL),
(N'Преступление и наказание', 2, 900.00, '1866-01-01', NULL),
(N'Братья Карамазовы', 2, 1100.00, '1880-01-01', NULL),
(N'1984', 3, 800.00, '1949-06-08', REPLICATE('B', 50000)),
(N'Скотный двор', 3, 600.00, '1945-08-17', NULL),
(N'Старик и море', 4, 700.00, '1952-09-01', NULL),
(N'Прощай, оружие!', 4, 750.00, '1929-09-27', NULL),
(N'Сто лет одиночества', 5, 1300.00, '1967-05-30', REPLICATE('C', 100000)),
(N'Любовь во время чумы', 5, 950.00, '1985-01-01', NULL);
GO

SELECT COUNT(*) AS AuthorsCount FROM dbo.Authors;  -- 5
SELECT COUNT(*) AS BooksCount FROM dbo.Books;      -- 10
GO
```

### 7.2. Модели и `DbContext`

```csharp
public class Author
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public string? Country { get; set; }
    public DateTime? BirthDate { get; set; }
    public List<Book> Books { get; set; } = new();
}

public class Book
{
    public int Id { get; set; }
    public string Title { get; set; } = "";
    public int AuthorId { get; set; }
    public Author Author { get; set; } = null!;
    public decimal Price { get; set; }
    public DateTime PublishedAt { get; set; }
    public string? Content { get; set; }
}

public class AppDbContext : DbContext
{
    public DbSet<Author> Authors => Set<Author>();
    public DbSet<Book> Books => Set<Book>();

    protected override void OnConfiguring(DbContextOptionsBuilder options)
    {
        options.UseSqlServer(
            @"Server=(localdb)\MSSQLLocalDB;Database=EfCoreTestDb;Integrated Security=true;TrustServerCertificate=true;");
        // Для диагностики в разработке:
        options.EnableSensitiveDataLogging();
        options.LogTo(Console.WriteLine, LogLevel.Information);
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Book>()
            .HasOne(b => b.Author)
            .WithMany(a => a.Books)
            .HasForeignKey(b => b.AuthorId);
    }
}
//Для примера 4 с фабрикой
public class AppDbContext1 : DbContext
{
    // ✔ Единственный конструктор — принимает опции от фабрики/DI
    public AppDbContext1(DbContextOptions<AppDbContext1> options)
        : base(options)
    {
    }

    public DbSet<Author> Authors => Set<Author>();
    public DbSet<Book> Books => Set<Book>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Book>()
            .HasOne(b => b.Author)
            .WithMany(a => a.Books)
            .HasForeignKey(b => b.AuthorId);

        modelBuilder.Entity<Book>()
            .Property(b => b.Price)
            .HasPrecision(18, 2);
    }

    // ❌ OnConfiguring НЕ переопределяем — конфигурация приходит через options
}
```


## 8. Примеры работы (каждый можно запустить)

### Пример 1. Буферизация vs Потоковое чтение — потребление памяти

```csharp
public static async Task Example1_BufferingVsStreamingAsync()
{
    Console.WriteLine("=== Пример 1: Буферизация vs Потоковое чтение ===");

    using var context = new AppDbContext();

    // --- Буферизация: все 10 книг в памяти ---
    var sw = Stopwatch.StartNew();
    var buffered = await context.Books
        .AsNoTracking()
        .ToListAsync();  // загружает все строки, создаёт List<Book>
    sw.Stop();
    Console.WriteLine($"  ToListAsync: {buffered.Count} книг, {sw.ElapsedMilliseconds} мс, " +
                      $"память под объекты: ~{buffered.Count * 200} байт (оценка)");

    // --- Потоковое чтение: по одной книге, память константна ---
    sw.Restart();
    int streamedCount = 0;
    await foreach (var book in context.Books
        .AsNoTracking()
        .AsAsyncEnumerable())  // НЕ загружает всё сразу
    {
        // Обрабатываем книгу по одной — в памяти только одна сущность
        streamedCount++;
    }
    sw.Stop();
    Console.WriteLine($"  AsAsyncEnumerable: {streamedCount} книг, {sw.ElapsedMilliseconds} мс, " +
                      "память: ~200 байт (одна книга за раз)");

    Console.WriteLine("  ✔ Потоковое чтение экономит память при больших выборках.");
}
```

**Ожидаемый результат:** оба метода читают 10 книг, но `AsAsyncEnumerable` не держит все объекты в памяти одновременно.


### Пример 2. `AsNoTracking` — влияние на память и скорость

```csharp
public static async Task Example2_NoTrackingAsync()
{
    Console.WriteLine("\n=== Пример 2: AsNoTracking ===");

    using var context = new AppDbContext();

    // --- С отслеживанием ---
    var tracked = await context.Books.ToListAsync();
    int trackedCount = context.ChangeTracker.Entries().Count();
    Console.WriteLine($"  С отслеживанием: загружено {tracked.Count} книг, " +
                      $"в ChangeTracker: {trackedCount} записей");

    // Очищаем трекер, чтобы следующий запрос был чистым
    context.ChangeTracker.Clear();

    // --- Без отслеживания ---
    var noTracking = await context.Books.AsNoTracking().ToListAsync();
    int noTrackedCount = context.ChangeTracker.Entries().Count();
    Console.WriteLine($"  Без отслеживания: загружено {noTracking.Count} книг, " +
                      $"в ChangeTracker: {noTrackedCount} записей");

    Console.WriteLine("  ✔ AsNoTracking не заполняет ChangeTracker — экономия памяти.");
}
```

**Ожидаемый результат:** после `ToListAsync()` без `AsNoTracking` в трекере 10 записей; с `AsNoTracking` — 0.


### Пример 3. Параллельные запросы на одном `DbContext` — ошибка

```csharp
public static async Task Example3_ParallelOnSameContextAsync()
{
    Console.WriteLine("\n=== Пример 3: Параллельные запросы на одном DbContext ===");

    using var context = new AppDbContext();

    try
    {
        // Запускаем два запроса параллельно на ОДНОМ контексте
        var task1 = context.Books.CountAsync();
        var task2 = context.Authors.CountAsync();

        await Task.WhenAll(task1, task2);  // здесь должно упасть
        Console.WriteLine("  Неожиданно: ошибки не было.");
    }
    catch (InvalidOperationException ex)
    {
        Console.WriteLine($"  ✔ Ожидаемая ошибка: {ex.Message.Split('\n')[0]}");
        Console.WriteLine("  (EF Core не поддерживает параллельные операции на одном контексте.)");
    }
}
```

**Ожидаемый результат:** `InvalidOperationException` — второй запрос начался до завершения первого.


### Пример 4. Параллельные запросы через `IDbContextFactory`

```csharp
// Требуется пакет Microsoft.EntityFrameworkCore (IDbContextFactory встроен в EF Core 5+)
public static async Task Example4_ParallelWithFactoryAsync()
{
    Console.WriteLine("\n=== Пример 4: Параллельные запросы через IDbContextFactory ===");

    // Создаём фабрику вручную (в ASP.NET Core она регистрируется в DI)
    var factory = new PooledDbContextFactory<AppDbContext1>(
        new DbContextOptionsBuilder<AppDbContext1>()
            .UseSqlServer(@"Server=(localdb)\MSSQLLocalDB;Database=EfCoreTestDb;Integrated Security=true;TrustServerCertificate=true;")
            .Options);

    // Каждая задача создаёт СВОЙ контекст
    var task1 = Task.Run(async () =>
    {
        await using var ctx = await factory.CreateDbContextAsync();
        return await ctx.Books.CountAsync();
    });

    var task2 = Task.Run(async () =>
    {
        await using var ctx = await factory.CreateDbContextAsync();
        return await ctx.Authors.CountAsync();
    });

    var results = await Task.WhenAll(task1, task2);
    Console.WriteLine($"  ✔ Параллельно: Books={results[0]}, Authors={results[1]}");
    Console.WriteLine("  (Каждый контекст изолирован — ошибки нет.)");
}
```

**Ожидаемый результат:** оба запроса выполняются параллельно без ошибок, потому что каждый использует свой `DbContext`.


### Пример 5. `SaveChangesAsync` и неявная транзакция

```csharp
public static async Task Example5_ImplicitTransactionAsync()
{
    Console.WriteLine("\n=== Пример 5: Неявная транзакция SaveChangesAsync ===");

    using var context = new AppDbContext();
    int before = await context.Authors.CountAsync();
    Console.WriteLine($"  Авторов до: {before}");

    // Добавляем двух авторов
    context.Authors.Add(new Author { Name = "Тест 1", Country = "X" });
    context.Authors.Add(new Author { Name = "Тест 2", Country = "Y" });

    // SaveChangesAsync оборачивает ОБА INSERT в одну транзакцию
    await context.SaveChangesAsync();
    int after = await context.Authors.CountAsync();
    Console.WriteLine($"  Авторов после: {after} (добавлено {after - before})");

    // Откатываем вручную, чтобы не засорять БД
    var added = context.Authors.Where(a => a.Name.StartsWith("Тест")).ToList();
    context.Authors.RemoveRange(added);
    await context.SaveChangesAsync();
    Console.WriteLine("  Тестовые данные удалены.");
}
```

**Ожидаемый результат:** оба автора добавлены за один `SaveChangesAsync`. Если второй INSERT упадёт (например, из-за нарушения уникальности), первый тоже не сохранится.


### Пример 6. Явная транзакция с откатом

```csharp
public static async Task Example6_ExplicitTransactionAsync()
{
    Console.WriteLine("\n=== Пример 6: Явная транзакция с откатом ===");

    using var context = new AppDbContext();
    int before = await context.Books.CountAsync();
    Console.WriteLine($"  Книг до: {before}");

    await using var tx = await context.Database.BeginTransactionAsync();
    try
    {
        context.Books.Add(new Book
        {
            Title = "Временная книга",
            AuthorId = 1,
            Price = 100,
            PublishedAt = DateTime.UtcNow
        });
        await context.SaveChangesAsync();

        // Намеренно вызываем ошибку — откат
        throw new InvalidOperationException("Симуляция ошибки");
    }
    catch (InvalidOperationException)
    {
        await tx.RollbackAsync(CancellationToken.None);
        Console.WriteLine("  Транзакция откачена.");
    }

    int after = await context.Books.CountAsync();
    Console.WriteLine($"  Книг после: {after} (изменение не сохранилось: {before == after})");
}
```

**Ожидаемый результат:** `before == after`, потому что транзакция откачена.


### Пример 7. `ExecuteUpdateAsync` — массовое обновление без загрузки

```csharp
public static async Task Example7_ExecuteUpdateAsync()
{
    Console.WriteLine("\n=== Пример 7: ExecuteUpdateAsync (EF Core 7+) ===");

    using var context = new AppDbContext();

    // Увеличиваем цену всех книг Оруэлла на 10% БЕЗ загрузки сущностей
    int affected = await context.Books
        .Where(b => b.Author.Name == "Джордж Оруэлл")
        .ExecuteUpdateAsync(setters => setters
            .SetProperty(b => b.Price, b => b.Price * 1.10m));

    Console.WriteLine($"  Обновлено книг: {affected} (одним SQL UPDATE, без загрузки в память)");

    // Проверяем
    var prices = await context.Books
        .Where(b => b.Author.Name == "Джордж Оруэлл")
        .Select(b => new { b.Title, b.Price })
        .AsNoTracking()
        .ToListAsync();
    foreach (var p in prices)
        Console.WriteLine($"    {p.Title}: {p.Price}");

    // Откатываем
    await context.Books
        .Where(b => b.Author.Name == "Джордж Оруэлл")
        .ExecuteUpdateAsync(setters => setters
            .SetProperty(b => b.Price, b => b.Price / 1.10m));
}
```

**Ожидаемый результат:** цены обновлены одним SQL-запросом, без загрузки 2 сущностей в Change Tracker.


### Пример 8. Потоковое чтение с клиентской обработкой

```csharp
public static async Task Example8_StreamingWithClientLogicAsync()
{
    Console.WriteLine("\n=== Пример 8: AsAsyncEnumerable + клиентская логика ===");

    using var context = new AppDbContext();

    // Считаем суммарную стоимость книг, но без загрузки всех в память
    decimal total = 0;
    int count = 0;

    await foreach (var book in context.Books
        .AsNoTracking()
        .AsAsyncEnumerable())
    {
        total += book.Price;
        count++;
        // Здесь можно делать что-то с каждой книгой, не держа предыдущие
    }

    Console.WriteLine($"  Обработано {count} книг, суммарная стоимость: {total:C}");
    Console.WriteLine("  ✔ Память использовалась только для одной книги за раз.");
}
```

**Ожидаемый результат:** сумма считается без загрузки всего списка в память.


### Пример 9. `AsAsyncEnumerable` + `AnyAsync` — ловушка

```csharp
public static async Task Example9_AsAsyncEnumerableTrapAsync()
{
    Console.WriteLine("\n=== Пример 9: Ловушка AsAsyncEnumerable перед AnyAsync ===");

    using var context = new AppDbContext();

    // ❌ НЕПРАВИЛЬНО: AsAsyncEnumerable() заставляет загрузить ВСЁ, потом проверить
    // (для маленькой таблицы это не видно, но для 1M строк — катастрофа)
    var sw = Stopwatch.StartNew();
    bool existsWrong = await context.Books
        .AsAsyncEnumerable()
        .AnyAsync(b => b.Price > 1000);
    sw.Stop();
    Console.WriteLine($"  AsAsyncEnumerable().AnyAsync(): {sw.ElapsedMilliseconds} мс (загружено ВСЁ)");

    // ✔ ПРАВИЛЬНО: серверный AnyAsync транслируется в SQL EXISTS
    sw.Restart();
    bool existsRight = await context.Books
        .AnyAsync(b => b.Price > 1000);
    sw.Stop();
    Console.WriteLine($"  AnyAsync() напрямую: {sw.ElapsedMilliseconds} мс (SQL EXISTS)");
    Console.WriteLine($"  Результаты совпадают: {existsWrong == existsRight}");
}
```

**Ожидаемый результат:** прямой `AnyAsync()` работает быстрее, потому что SQL Server проверяет наличие строки и останавливается, а `AsAsyncEnumerable().AnyAsync()` загружает все строки в память .


### Пример 10. `CancellationToken` и отмена запроса

```csharp
public static async Task Example10_CancellationAsync()
{
    Console.WriteLine("\n=== Пример 10: CancellationToken ===");

    using var context = new AppDbContext();
    using var cts = new CancellationTokenSource(TimeSpan.FromMilliseconds(1));

    try
    {
        // Отменяем запрос через 1 мс — он не успеет выполниться
        var books = await context.Books
            .AsNoTracking()
            .ToListAsync(cts.Token);
        Console.WriteLine($"  Запрос выполнен (неожиданно): {books.Count} книг");
    }
    catch (OperationCanceledException)
    {
        Console.WriteLine("  ✔ Запрос отменён через CancellationToken.");
    }
    catch (SqlException ex) when (ex.Number == -2)
    {
        Console.WriteLine($"  SqlException (таймаут): {ex.Message.Split('\n')[0]}");
    }
}
```

**Ожидаемый результат:** `OperationCanceledException` или `SqlException` — в зависимости от того, как быстро провайдер среагирует на отмену. EF Core передаёт токен в `Microsoft.Data.SqlClient`, но провайдер может не соблюдать его мгновенно .


## 9. Тренировочные задания (10 штук, с каркасом, без кода)

### Задание 1. Сравнить буферизацию и потоковое чтение

**Каркас:**
```csharp
public static async Task<(int BufferedCount, int StreamedCount, long BufferedMs, long StreamedMs)>
    CompareBufferingAsync(AppDbContext context)
{
    // TODO: замерить время ToListAsync и AsAsyncEnumerable для одной и той же выборки.
    // Вернуть (количество книг в буфере, количество потоково прочитанных, время буфера, время потока).
    throw new NotImplementedException();
}
```

**Критерии:** оба счётчика равны 10. Время может отличаться в любую сторону; главное — понять, что буферизация всегда загружает всё, а потоковая — по одной строке.

---

### Задание 2. Доказать, что `AsNoTracking` не заполняет ChangeTracker

**Каркас:**
```csharp
public static async Task<(int TrackedEntries, int NoTrackingEntries)> ProveNoTrackingAsync(AppDbContext context)
{
    // TODO: загрузить 5 книг С отслеживанием → вернуть количество записей в ChangeTracker.
    // Очистить ChangeTracker.
    // Загрузить 5 книг БЕЗ отслеживания → вернуть количество записей.
    throw new NotImplementedException();
}
```

**Критерии:** `(5, 0)`. Если второе значение не 0 — `AsNoTracking` не применился.

---

### Задание 3. Воспроизвести ошибку параллельного доступа к `DbContext`

**Каркас:**
```csharp
public static async Task<string> ReproduceParallelErrorAsync(AppDbContext context)
{
    // TODO: запустить Task.WhenAll(CountAsync, CountAsync) на одном контексте.
    // Перехватить исключение и вернуть его тип и первые 60 символов сообщения.
    throw new NotImplementedException();
}
```

**Критерии:** тип исключения — `InvalidOperationException`. Сообщение содержит «second operation» или «context instance».

---

### Задание 4. Выполнить параллельные запросы безопасно

**Каркас:**
```csharp
public static async Task<(int Books, int Authors)> ParallelQueriesSafeAsync(IDbContextFactory<AppDbContext> factory)
{
    // TODO: создать два контекста из фабрики, выполнить CountAsync на каждом параллельно.
    // Вернуть результаты.
    throw new NotImplementedException();
}
```

**Критерии:** оба запроса выполняются без исключений; результат `(10, 5)`.

---

### Задание 5. Явная транзакция с откатом

**Каркас:**
```csharp
public static async Task<bool> TransactionRollbackAsync(AppDbContext context)
{
    // TODO: начать транзакцию, вставить книгу, намеренно вызвать исключение,
    // откатить транзакцию, проверить что книги не добавились.
    // Вернуть true если откат сработал (количество книг не изменилось).
    throw new NotImplementedException();
}
```

**Критерии:** возвращает `true`; количество книг до и после совпадает.

---

### Задание 6. Массовое обновление через `ExecuteUpdateAsync`

**Каркас:**
```csharp
public static async Task<int> BulkPriceIncreaseAsync(AppDbContext context, decimal multiplier)
{
    // TODO: увеличить цену всех книг в заданное количество раз через ExecuteUpdateAsync.
    // Вернуть количество затронутых строк.
    // После проверки — вернуть цены к исходным (тоже через ExecuteUpdateAsync).
    throw new NotImplementedException();
}
```

**Критерии:** возвращает количество книг (10); цены изменились в соответствии с множителем. После отката — вернулись к исходным.

---

### Задание 7. Потоковый подсчёт с `AsAsyncEnumerable`

**Каркас:**
```csharp
public static async Task<decimal> StreamSumPriceAsync(AppDbContext context)
{
    // TODO: используя AsAsyncEnumerable, просуммировать Price всех книг.
    // НЕ использовать ToListAsync.
    throw new NotImplementedException();
}
```

**Критерии:** сумма совпадает с `SELECT SUM(Price) FROM Books` (можно проверить через `SumAsync`).

---

### Задание 8. Отмена запроса через `CancellationToken`

**Каркас:**
```csharp
public static async Task<bool> CancelQueryAsync(AppDbContext context)
{
    // TODO: создать CancellationTokenSource с CancelAfter(1ms),
    // попытаться выполнить ToListAsync с этим токеном.
    // Вернуть true если было выброшено OperationCanceledException,
    // false если запрос успел выполниться.
    throw new NotImplementedException();
}
```

**Критерии:** в большинстве случаев возвращает `true` (отмена срабатывает). Если `false` — запрос был слишком быстрым (допустимо для LocalDB с 10 строками).

---

### Задание 9. Воспроизвести ловушку `AsAsyncEnumerable` перед `AnyAsync`

**Каркас:**
```csharp
public static async Task<(long WrongMs, long RightMs)> CompareAnyAsyncApproachesAsync(AppDbContext context)
{
    // TODO: замерить время двух подходов к AnyAsync:
    // 1. context.Books.AsAsyncEnumerable().AnyAsync(b => b.Price > 1000)
    // 2. context.Books.AnyAsync(b => b.Price > 1000)
    // Вернуть оба времени в миллисекундах.
    throw new NotImplementedException();
}
```

**Критерии:** оба подхода возвращают одинаковый логический результат. Время может быть сопоставимым на 10 строках, но на большой таблице разница будет драматической. Задание на понимание, а не на цифры.

---

### Задание 10. Полный сценарий: транзакция + отмена + очистка

**Каркас:**
```csharp
public static async Task<(bool Cancelled, int BooksBefore, int BooksAfter)> TransactionWithCancelAsync(
    AppDbContext context, CancellationToken userToken)
{
    // TODO:
    // 1. Запомнить количество книг ДО.
    // 2. Начать транзакцию.
    // 3. Вставить книгу.
    // 4. Подождать 500 мс с userToken (сработает отмена).
    // 5. В catch (OperationCanceledException) откатить транзакцию с CancellationToken.None.
    // 6. Вернуть (true, BooksBefore, BooksAfter).
    throw new NotImplementedException();
}
```

**Критерии:** `Cancelled=true`, `BooksBefore == BooksAfter`. Демонстрирует правило «cleanup-токен» для отката.


## 10. Итоговый чеклист по EF Core

| Ситуация | Правило |
|---|---|
| **Чтение больших выборок** | `AsAsyncEnumerable()` + `AsNoTracking()` — потоково, без ChangeTracker |
| **Чтение для дальнейшей обработки** | `ToListAsync()` + `AsNoTracking()` — буферизация оправдана |
| **Обновление сущностей** | Либо загрузка с отслеживанием, либо `ExecuteUpdateAsync()` (EF Core 7+) |
| **Параллельные запросы** | Только через разные `DbContext` (из `IDbContextFactory`) |
| **Транзакции** | `SaveChangesAsync` неявно оборачивает всё; для нескольких операций — `BeginTransactionAsync` |
| **Откат транзакции** | `RollbackAsync(CancellationToken.None)` — всегда чистый токен |
| **`AnyAsync` / `CountAsync`** | **Никогда** не вызывайте после `AsAsyncEnumerable()` — это загрузит всё в память |
| **`CancellationToken`** | Передавайте во все асинхронные методы, но помните, что провайдер может его игнорировать |
| **`DbContext` в DI** | Scoped для HTTP-запросов; `IDbContextFactory` для фоновых задач |


## 11. Ключевые выводы

EF Core скрывает ADO.NET, но не устраняет фундаментальные ограничения: `DbContext` не потокобезопасен, асинхронные операторы выполняются только при материализации, а `AsNoTracking` и потоковое чтение — главные инструменты для работы с большими данными. Главные ловушки: параллельные запросы на одном контексте (ошибка), `AsAsyncEnumerable` перед `AnyAsync` (загрузка всей таблицы), и откат транзакции с отменённым токеном (транзакция остаётся открытой). Все примеры из раздела 8 запускаются в консольном приложении .NET 8 с пакетами `Microsoft.EntityFrameworkCore.SqlServer` и `Microsoft.EntityFrameworkCore`. База данных создаётся скриптом из раздела 7.1.
