
## 0. Скрипт создания базы и всех объектов

Сохраните как `CreateAdoDemoDB.sql` и выполните в SSMS / Azure Data Studio / `sqlcmd`.

```sql
-- ============================================================
-- 1. Создание базы данных (если её нет)
-- ============================================================
IF DB_ID('AdoDemoDB') IS NULL
BEGIN
    CREATE DATABASE AdoDemoDB;
END
GO

USE AdoDemoDB;
GO

-- ============================================================
-- 2. Таблица Users — используется в примерах Connection/Command
--    (совпадает с примерами из разделов 3.3, 4.1, 7.1, 8.x, 9.x)
-- ============================================================
IF OBJECT_ID('dbo.Users', 'U') IS NULL
BEGIN
    CREATE TABLE dbo.Users
    (
        Id       INT IDENTITY(1,1) PRIMARY KEY,
        Name     NVARCHAR(100) NOT NULL,
        Age      INT           NOT NULL,
        Email    NVARCHAR(200) NULL,
        Avatar   VARBINARY(MAX) NULL,   -- для примера с byte[] (9.6)
        CreatedAt DATETIME2(0) NOT NULL DEFAULT SYSUTCDATETIME()
    );
END
GO

-- ============================================================
-- 3. Таблица DemoUsers — используется в полном демо-примере (раздел 11)
-- ============================================================
IF OBJECT_ID('dbo.DemoUsers', 'U') IS NULL
BEGIN
    CREATE TABLE dbo.DemoUsers
    (
        Id    INT IDENTITY(1,1) PRIMARY KEY,
        Name  NVARCHAR(100) NOT NULL,
        Age   INT           NOT NULL,
        Email NVARCHAR(200) NULL
    );
END
GO

-- ============================================================
-- 4. Таблица Orders — используется в примере с датами (9.5)
-- ============================================================
IF OBJECT_ID('dbo.Orders', 'U') IS NULL
BEGIN
    CREATE TABLE dbo.Orders
    (
        Id        INT IDENTITY(1,1) PRIMARY KEY,
        UserId    INT NOT NULL REFERENCES dbo.Users(Id),
        Amount    DECIMAL(18,2) NOT NULL,
        CreatedAt DATETIME2(0) NOT NULL DEFAULT SYSUTCDATETIME()
    );
END
GO

-- ============================================================
-- 5. Таблица Accounts — фигурировала в примере транзакций
--    (транзакции в тексте убраны, но таблица пусть будет —
--     пригодится, если добавите их позже)
-- ============================================================
IF OBJECT_ID('dbo.Accounts', 'U') IS NULL
BEGIN
    CREATE TABLE dbo.Accounts
    (
        Id      INT IDENTITY(1,1) PRIMARY KEY,
        Owner   NVARCHAR(100) NOT NULL,
        Balance DECIMAL(18,2) NOT NULL DEFAULT 0
    );
END
GO

-- ============================================================
-- 6. Тестовые данные
-- ============================================================
IF NOT EXISTS (SELECT 1 FROM dbo.Users)
BEGIN
    INSERT INTO dbo.Users (Name, Age, Email) VALUES
        (N'Анна',  22, N'anna@example.com'),
        (N'Борис', 35, NULL),
        (N'Вера',  28, N'vera@example.com'),
        (N'Глеб',  41, N'gleb@example.com'),
        (N'Дарья', 19, N'daria@example.com'),
        (N'Егор',  30, NULL);
END
GO

IF NOT EXISTS (SELECT 1 FROM dbo.DemoUsers)
BEGIN
    INSERT INTO dbo.DemoUsers (Name, Age, Email) VALUES
        (N'Анна',  22, N'anna@example.com'),
        (N'Борис', 35, NULL),
        (N'Вера',  28, N'vera@example.com'),
        (N'Глеб',  41, N'gleb@example.com');
END
GO

IF NOT EXISTS (SELECT 1 FROM dbo.Orders)
BEGIN
    INSERT INTO dbo.Orders (UserId, Amount, CreatedAt) VALUES
        (1, 1500.00, '2024-02-10'),
        (1, 3200.50, '2024-06-15'),
        (2,  990.00, '2024-03-01'),
        (3, 5000.00, '2025-01-20'),
        (4,  120.75, '2025-05-05');
END
GO

IF NOT EXISTS (SELECT 1 FROM dbo.Accounts)
BEGIN
    INSERT INTO dbo.Accounts (Owner, Balance) VALUES
        (N'Иванов', 10000.00),
        (N'Петров',  2500.00);
END
GO

-- ============================================================
-- 7. Пользовательский тип для TVP-примера (раздел 9.7)
-- ============================================================
IF TYPE_ID('dbo.IdList') IS NULL
BEGIN
    CREATE TYPE dbo.IdList AS TABLE (Id INT PRIMARY KEY);
END
GO

-- ============================================================
-- 8. Хранимые процедуры
-- ============================================================

-- 8.1 usp_GetUsersByAge — пример из раздела 10.1
CREATE OR ALTER PROCEDURE dbo.usp_GetUsersByAge
    @minAge INT
AS
BEGIN
    SET NOCOUNT ON;
    SELECT Id, Name, Age, Email
    FROM dbo.Users
    WHERE Age >= @minAge
    ORDER BY Age;
END
GO

-- 8.2 usp_CreateUser — пример из раздела 10.2
--     Возвращает новый Id через OUTPUT и код возврата
CREATE OR ALTER PROCEDURE dbo.usp_CreateUser
    @name NVARCHAR(100),
    @age  INT,
    @newId INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    IF @name IS NULL OR LEN(@name) = 0
    BEGIN
        RETURN 1;  -- пустое имя — ошибка
    END

    INSERT INTO dbo.Users (Name, Age) VALUES (@name, @age);
    SET @newId = CAST(SCOPE_IDENTITY() AS INT);

    RETURN 0;  -- успех
END
GO

-- 8.3 usp_DoSomething — пример из раздела 8.6
--     Просто возвращает код 0 и делает что-то полезное
CREATE OR ALTER PROCEDURE dbo.usp_DoSomething
    @in INT
AS
BEGIN
    SET NOCOUNT ON;

    -- Демонстрационная операция: логируем в отдельную таблицу
    IF OBJECT_ID('dbo.CallLog', 'U') IS NULL
    BEGIN
        CREATE TABLE dbo.CallLog
        (
            Id        INT IDENTITY(1,1) PRIMARY KEY,
            Value     INT NOT NULL,
            CalledAt  DATETIME2(0) NOT NULL DEFAULT SYSUTCDATETIME()
        );
    END

    INSERT INTO dbo.CallLog (Value) VALUES (@in);

    RETURN 0;
END
GO

-- 8.4 usp_DemoCount — пример из раздела 11 (создаётся на лету в коде)
--     Здесь создаём заранее, чтобы был доступен в любой момент
CREATE OR ALTER PROCEDURE dbo.usp_DemoCount
    @minAge INT,
    @total  INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SELECT @total = COUNT(*) FROM dbo.DemoUsers WHERE Age >= @minAge;
END
GO

-- ============================================================
-- 9. Проверка: показать все объекты
-- ============================================================
PRINT '=== Объекты в AdoDemoDB ===';
SELECT 'TABLE' AS Kind, name
FROM sys.tables
UNION ALL
SELECT 'PROC', name
FROM sys.procedures
UNION ALL
SELECT 'TYPE', name
FROM sys.table_types
ORDER BY Kind, name;
GO
```

---

## 0.1. Строка подключения для примеров

Примеры из руководства использовали `Database=TestDB`. Замените на `AdoDemoDB`:

```csharp
string connectionString =
    "Server=localhost;Database=AdoDemoDB;Integrated Security=true;TrustServerCertificate=true;";
```

Если у вас SQL Express:

```csharp
string connectionString =
    "Server=.\\SQLEXPRESS;Database=AdoDemoDB;Integrated Security=true;TrustServerCertificate=true;";
```
---

## 1. Подготовка среды

### 1.1 Установка пакета

```bash
dotnet add package Microsoft.Data.SqlClient
```

### 1.2 Необходимые пространства имён

```csharp
using Microsoft.Data.SqlClient;
using System.Data;
```

---

# Часть I. SqlConnection — соединение с базой данных

`SqlConnection` представляет логическое соединение с SQL Server. Правильное управление соединением — фундамент всей работы с ADO.NET.

## 2. Строка подключения: полный разбор

Строка подключения состоит из пар `ключ=значение`, разделённых точкой с запятой. Рассмотрим все важные ключи.

### 2.1 Базовые ключи

```csharp
// Проверка подлинности Windows (Integrated Security)
string cs1 = "Server=localhost;Database=TestDB;Integrated Security=true;TrustServerCertificate=true;";

// Проверка подлинности SQL Server (логин/пароль)
string cs2 = "Server=localhost;Database=TestDB;User Id=sa;Password=YourPassword;TrustServerCertificate=true;";

// Именованный экземпляр (SQL Express)
string cs3 = "Server=.\\SQLEXPRESS;Database=TestDB;Integrated Security=true;TrustServerCertificate=true;";

// Удалённый сервер с портом
string cs4 = "Server=192.168.1.10,1433;Database=TestDB;User Id=app_user;Password=P@ssw0rd;TrustServerCertificate=true;";
```

**Разбор ключей:**

| Ключ | Алиас | Назначение |
|------|-------|-----------|
| `Server` / `Data Source` | — | Адрес сервера, экземпляр, порт |
| `Database` / `Initial Catalog` | — | Имя начальной базы данных |
| `Integrated Security` / `Trusted_Connection` | — | `true` = Windows-аутентификация |
| `User Id` / `UID` | — | Логин SQL-аутентификации |
| `Password` / `PWD` | — | Пароль SQL-аутентификации |
| `TrustServerCertificate` | — | Принять самоподписанный сертификат (нужно локально) |
| `Encrypt` | — | Шифрование канала (`true`/`false`/`Mandatory`/`Optional`) |

### 2.2 Ключи производительности и поведения

```csharp
string cs = "Server=localhost;Database=TestDB;Integrated Security=true;" +
            "TrustServerCertificate=true;" +
            "Connect Timeout=15;" +          // таймаут установки соединения (сек)
            "Application Name=MyApp;" +       // имя приложения в sys.dm_exec_sessions
            "Pooling=true;" +                 // включить пул (по умолчанию true)
            "Min Pool Size=2;" +              // минимум соединений в пуле
            "Max Pool Size=100;" +            // максимум соединений в пуле
            "Connection Lifetime=300;" +      // макс. время жизни соединения (сек)
            "MultipleActiveResultSets=false;" +// MARS: несколько активных ридеров
            "Packet Size=4096;";              // размер сетевого пакета
```

**Пояснения:**

- **Connect Timeout** — сколько ждать установки физического соединения, прежде чем выбросить `SqlException`.
- **Application Name** — полезно для диагностики: в SQL Server через `sys.dm_exec_sessions` видно, какое приложение держит сессию.
- **Pooling** — пул соединений включён по умолчанию. Физические соединения переиспользуются, поэтому `Open()`/`Close()` в цикле — не проблема.
- **MultipleActiveResultSets (MARS)** — если `true`, можно открыть несколько `SqlDataReader` на одном соединении одновременно. По умолчанию `false`.
- **Packet Size** — размер пакета TDS; увеличение может помочь при передаче больших BLOB.

### 2.3 Программное построение через SqlConnectionStringBuilder

Безопаснее и читаемее строить строку через `SqlConnectionStringBuilder` — он корректно экранирует пароли со спецсимволами:

```csharp
var builder = new SqlConnectionStringBuilder
{
    DataSource = "localhost",
    InitialCatalog = "TestDB",
    IntegratedSecurity = true,
    TrustServerCertificate = true,
    ConnectTimeout = 15,
    ApplicationName = "MyApp"
};
string connectionString = builder.ConnectionString;
Console.WriteLine(connectionString);
```

## 3. Свойства SqlConnection

`SqlConnection` предоставляет множество свойств для чтения состояния и настроек. Ниже — самый полный практический перечень.

### 3.1 Свойства состояния и идентификации

```csharp
using (var conn = new SqlConnection(connectionString))
{
    Console.WriteLine("ConnectionString: " + conn.ConnectionString);
    Console.WriteLine("State (до Open): " + conn.State);          // Closed
    Console.WriteLine("DataSource: " + conn.DataSource);         // localhost
    Console.WriteLine("Database (до Open): " + conn.Database);   // "" (пусто до открытия)

    conn.Open();

    Console.WriteLine("State (после Open): " + conn.State);          // Open
    Console.WriteLine("Database (после Open): " + conn.Database);    // TestDB
    Console.WriteLine("ServerVersion: " + conn.ServerVersion);       // напр. 16.00.1000
    Console.WriteLine("WorkstationId: " + conn.WorkstationId);       // имя машины клиента
    Console.WriteLine("PacketSize: " + conn.PacketSize);
    Console.WriteLine("ClientConnectionId: " + conn.ClientConnectionId); // GUID физического соединения
    Console.WriteLine("ConnectionTimeout: " + conn.ConnectionTimeout);
}
```

**Разбор важных свойств:**

| Свойство | Тип | Что даёт |
|----------|-----|----------|
| `State` | `ConnectionState` | `Closed`, `Open`, `Connecting`, `Executing`, `Fetching`, `Broken` |
| `Database` | `string` | Текущая БД; можно менять на лету — см. `ChangeDatabase` ниже |
| `DataSource` | `string` | Сервер из строки подключения |
| `ServerVersion` | `string` | Версия SQL Server (строка) |
| `WorkstationId` | `string` | Имя клиентской машины |
| `ClientConnectionId` | `Guid` | ID соединения для диагностики (совпадает в Extended Events) |
| `ConnectionTimeout` | `int` | Таймаут установки соединения в секундах |
| `PacketSize` | `int` | Текущий размер сетевого пакета |
| `FireInfoMessageEventOnUserErrors` | `bool` | Превращать ошибки уровня 11–16 в `InfoMessage` |

### 3.2 Методы SqlConnection

```csharp
using (var conn = new SqlConnection(connectionString))
{
    conn.Open();

    // 1. ChangeDatabase — переключить текущую БД без пересоздания соединения
    conn.ChangeDatabase("master");
    Console.WriteLine("БД после переключения: " + conn.Database); // master

    // 2. CreateCommand — фабричный метод (удобно, не нужно передавать conn)
    using (var cmd = conn.CreateCommand())
    {
        cmd.CommandText = "SELECT DB_NAME()";
        Console.WriteLine("DB_NAME(): " + cmd.ExecuteScalar());
    }

    // 3. BeginTransaction — см. отдельный раздел (здесь не рассматриваем)
    // 4. OpenAsync / CloseAsync — асинхронные версии
    // 5. Dispose — освобождение (вызывается автоматически в using)
}
```

**Полный список ключевых методов:**

| Метод | Назначение |
|-------|-----------|
| `Open()` | Открыть соединение синхронно |
| `OpenAsync(ct)` | Открыть соединение асинхронно |
| `Close()` | Закрыть (вернуть в пул) |
| `ChangeDatabase(name)` | Переключить текущую БД |
| `CreateCommand()` | Создать `SqlCommand`, связанный с этим соединением |
| `GetSchema()` / `GetSchema(collection)` | Метаданные схемы — см. ниже |
| `ResetStatistics()` | Сбросить статистику выполнения |
| `Dispose()` | Освободить ресурсы |

### 3.3 GetSchema — метаданные базы данных

Один из самых полезных, но редко упоминаемых методов. Возвращает `DataTable` с описанием объектов БД.

```csharp
using (var conn = new SqlConnection(connectionString))
{
    conn.Open();

    // Список всех таблиц
    DataTable tables = conn.GetSchema("Tables");
    Console.WriteLine("Таблицы:");
    foreach (DataRow row in tables.Rows)
    {
        // TABLE_CATALOG, TABLE_SCHEMA, TABLE_NAME, TABLE_TYPE
        Console.WriteLine($"  {row["TABLE_SCHEMA"]}.{row["TABLE_NAME"]} ({row["TABLE_TYPE"]})");
    }

    // Столбцы конкретной таблицы
    DataTable columns = conn.GetSchema("Columns", new[] { "TestDB", null, "Users", null });
    Console.WriteLine("Столбцы Users:");
    foreach (DataRow row in columns.Rows)
    {
        Console.WriteLine($"  {row["COLUMN_NAME"]} — {row["DATA_TYPE"]} (max {row["CHARACTER_MAXIMUM_LENGTH"]})");
    }

    // Все базы данных
    DataTable dbs = conn.GetSchema("Databases");
    Console.WriteLine("Базы данных:");
    foreach (DataRow row in dbs.Rows)
        Console.WriteLine("  " + row["database_name"]);
}
```

**Распространённые коллекции для `GetSchema`:**

| Коллекция | Что содержит |
|-----------|--------------|
| `Tables` | Таблицы и представления |
| `Columns` | Столбцы (с типами, длинами) |
| `Databases` | Все базы данных на сервере |
| `Indexes` | Индексы |
| `ForeignKeys` | Внешние ключи |
| `Procedures` | Хранимые процедуры |
| `Views` | Представления |
| `Users` | Пользователи БД |

### 3.4 Событие InfoMessage

SQL Server может посылать клиенту информационные сообщения (например, `PRINT` в T-SQL) — они приходят не через исключение, а через событие `InfoMessage`:

```csharp
using (var conn = new SqlConnection(connectionString))
{
    // Подписываемся ДО открытия — событие доступно, даже если соединение закроется
    conn.InfoMessage += (sender, e) =>
    {
        Console.WriteLine($"[SQL PRINT] {e.Message}");
        foreach (SqlError err in e.Errors)
            Console.WriteLine($"  Класс {err.Class}, номер {err.Number}");
    };

    conn.Open();

    using (var cmd = new SqlCommand("PRINT 'Hello from SQL Server!'", conn))
        cmd.ExecuteNonQuery();
}
```

### 3.5 Событие StateChange

Позволяет отслеживать переходы состояния соединения (`Open` ↔ `Closed` ↔ `Broken`):

```csharp
using (var conn = new SqlConnection(connectionString))
{
    conn.StateChange += (s, e) =>
        Console.WriteLine($"Состояние: {e.OriginalState} → {e.CurrentState}");

    conn.Open();
    conn.Close();
}
```

## 4. Правильное управление жизненным циклом соединения

### 4.1 Идиома using — открыть как можно позже

```csharp
// Правильно: открываем прямо перед работой, закрываем сразу после
public int GetUserCount()
{
    using (var conn = new SqlConnection(connectionString))
    using (var cmd = new SqlCommand("SELECT COUNT(*) FROM Users", conn))
    {
        conn.Open();
        return (int)cmd.ExecuteScalar();
    }
}
```

### 4.2 Пул соединений — ключевое поведение

Пул включён по умолчанию. Когда вы вызываете `Close()` (или выходите из `using`), физическое соединение **не закрывается**, а возвращается в пул. Следующий `Open()` с **идентичной строкой подключения** получит его обратно без сетевого хендшейка.

```csharp
// 1000 итераций открытия/закрытия — это НЕ 1000 физических соединений
for (int i = 0; i < 1000; i++)
{
    using (var conn = new SqlConnection(connectionString))
    {
        conn.Open();
        // работа
    }
}
```

**Важно:** если изменить строку подключения (например, `Application Name`), для новой строки будет создан отдельный пул.

Очистить пул можно методом `ClearPool` / `ClearAllPools`:

```csharp
SqlConnection.ClearPool(new SqlConnection(connectionString));
// или для всех пулов процесса:
SqlConnection.ClearAllPools();
```

### 4.3 Try/Catch и типичные ошибки

`Open()` может выбросить `SqlException`. Ловите его для диагностики:

```csharp
try
{
    using (var conn = new SqlConnection(connectionString))
    {
        conn.Open();
        Console.WriteLine("Соединение установлено");
    }
}
catch (SqlException ex)
{
    Console.WriteLine($"SQL-ошибка #{ex.Number}: {ex.Message}");
    foreach (SqlError err in ex.Errors)
        Console.WriteLine($"  Сервер: {err.Server}, процедура: {err.Procedure}, строка: {err.LineNumber}");
}
```

Типичные `SqlException` при подключении:

| Номер | Причина |
|-------|---------|
| 18456 | Login failed (неверный логин/пароль) |
| 4060 | Cannot open database (нет доступа) |
| -1 / 2 | A network-related error (сервер не найден) |
| 10060 | Timeout expired |
| 233 | Ошибка на уровне канала (обычно конфигурация) |

---

# Часть II. SqlCommand — выполнение команд

`SqlCommand` инкапсулирует SQL-текст (или хранимую процедуру) и параметры и умеет его исполнять.

## 5. Создание SqlCommand

Три способа, все эквивалентны:

```csharp
// Способ 1: конструктор с текстом и соединением
var cmd1 = new SqlCommand("SELECT 1", conn);

// Способ 2: через CreateCommand у соединения
var cmd2 = conn.CreateCommand();
cmd2.CommandText = "SELECT 1";

// Способ 3: конструктор + последующее присвоение
var cmd3 = new SqlCommand { Connection = conn, CommandText = "SELECT 1" };
```

## 6. Свойства SqlCommand

### 6.1 CommandTimeout

Максимальное время выполнения команды в секундах. По умолчанию — 30. `0` = без таймаута (не рекомендуется).

```csharp
using (var cmd = new SqlCommand("WAITFOR DELAY '00:00:05'; SELECT 1", conn))
{
    conn.Open();
    cmd.CommandTimeout = 10; // ждать до 10 секунд
    cmd.ExecuteScalar();
}
```

### 6.2 CommandType

Определяет, как интерпретировать `CommandText`:

```csharp
// Text (по умолчанию) — обычный T-SQL
cmd.CommandType = CommandType.Text;
cmd.CommandText = "SELECT * FROM Users";

// StoredProcedure — имя процедуры без EXEC
cmd.CommandType = CommandType.StoredProcedure;
cmd.CommandText = "usp_GetUsers";
```

**`CommandType.TableDirect`** — возвращает всё содержимое таблицы напрямую (используется редко, требует `DataAdapter`).

### 6.3 Transaction и UpdatedRowSource

`Transaction` — привязка к транзакции (здесь не рассматриваем). `UpdatedRowSource` — используется с `DataAdapter`. В контексте этой статьи не нужны.

### 6.4 Параметры — важные свойства коллекции

```csharp
using (var cmd = new SqlCommand("SELECT @p", conn))
{
    cmd.Parameters.Add("@p", SqlDbType.Int).Value = 42;
    Console.WriteLine("Кол-во параметров: " + cmd.Parameters.Count);
    foreach (SqlParameter p in cmd.Parameters)
        Console.WriteLine($"{p.ParameterName}: {p.SqlDbType}, значение={p.Value}");
}
```

## 7. Методы выполнения SqlCommand

### 7.1 ExecuteNonQuery — INSERT / UPDATE / DELETE

Возвращает количество затронутых строк. Для DDL (CREATE TABLE и т.п.) возвращает `-1`.

```csharp
using (var conn = new SqlConnection(connectionString))
{
    conn.Open();

    // INSERT
    using (var cmd = new SqlCommand(
        "INSERT INTO Users (Name, Age) VALUES (@n, @a)", conn))
    {
        cmd.Parameters.Add("@n", SqlDbType.NVarChar, 100).Value = "Анна";
        cmd.Parameters.Add("@a", SqlDbType.Int).Value = 22;
        int inserted = cmd.ExecuteNonQuery();
        Console.WriteLine($"Вставлено: {inserted}");
    }

    // UPDATE
    using (var cmd = new SqlCommand(
        "UPDATE Users SET Age = Age + 1 WHERE Age < @max", conn))
    {
        cmd.Parameters.Add("@max", SqlDbType.Int).Value = 30;
        int updated = cmd.ExecuteNonQuery();
        Console.WriteLine($"Обновлено: {updated}");
    }

    // DELETE
    using (var cmd = new SqlCommand("DELETE FROM Users WHERE Age > @max", conn))
    {
        cmd.Parameters.Add("@max", SqlDbType.Int).Value = 100;
        int deleted = cmd.ExecuteNonQuery();
        Console.WriteLine($"Удалено: {deleted}");
    }
}
```

**Несколько операторов в одном CommandText** — через `;`:

```csharp
using (var cmd = new SqlCommand(
    "INSERT INTO Users (Name) VALUES (@a); INSERT INTO Users (Name) VALUES (@b);",
    conn))
{
    cmd.Parameters.Add("@a", SqlDbType.NVarChar, 50).Value = "Первый";
    cmd.Parameters.Add("@b", SqlDbType.NVarChar, 50).Value = "Второй";
    int total = cmd.ExecuteNonQuery(); // вернёт суммарно (обычно 2)
    Console.WriteLine("Всего строк: " + total);
}
```

### 7.2 ExecuteScalar — одно значение

Возвращает первую ячейку первой строки результата (`object`). Если строк нет — `null`.

```csharp
using (var cmd = new SqlCommand("SELECT COUNT(*) FROM Users", conn))
{
    object raw = cmd.ExecuteScalar();
    // Проверяем на null и приводим к int
    int count = raw == DBNull.Value || raw == null ? 0 : Convert.ToInt32(raw);
    Console.WriteLine("Пользователей: " + count);
}

// Пример со string
using (var cmd = new SqlCommand("SELECT Name FROM Users WHERE Id = 1", conn))
{
    object raw = cmd.ExecuteScalar();
    string name = raw as string ?? "не найдено";
    Console.WriteLine("Имя: " + name);
}
```

**Важно:** даже если значение в БД NULL, `ExecuteScalar` вернёт `DBNull.Value`, а не `null` в C#. Всегда проверяйте `raw == DBNull.Value`.

### 7.3 ExecuteReader — потоковое чтение

Возвращает `SqlDataReader`. Самые важные методы:

| Метод / свойство | Что делает |
|------------------|-----------|
| `Read()` | Переход к следующей строке; `false`, если строк больше нет |
| `HasRows` | Есть ли хоть одна строка в результате (до первого `Read`) |
| `FieldCount` | Число столбцов в текущей строке |
| `GetName(i)` / `GetOrdinal(name)` | Имя столбца по индексу / индекс по имени |
| `GetValue(i)` / `this[i]` | Значение как `object` по индексу |
| `this["col"]` | Значение как `object` по имени |
| `GetInt32(i)`, `GetString(i)`, `GetDateTime(i)`, `GetDecimal(i)`, `GetBoolean(i)` … | Типизированное чтение |
| `IsDBNull(i)` | Проверка на NULL по индексу |
| `GetFieldType(i)` | Тип поля |
| `GetDataTypeName(i)` | Имя SQL-типа (`nvarchar` и т.п.) |
| `RecordsAffected` | Строк затронуто (после чтения) |
| `NextResult()` | Переход к следующему результату (если несколько `SELECT`) |
| `Close()` / `Dispose()` | Закрыть ридер (закрывает и соединение, если не было открыто до) |

**Базовое чтение с именованными столбцами:**

```csharp
using (var cmd = new SqlCommand("SELECT Id, Name, Age, Email FROM Users", conn))
using (SqlDataReader r = cmd.ExecuteReader())
{
    if (!r.HasRows)
    {
        Console.WriteLine("Нет данных");
        return;
    }

    while (r.Read())
    {
        int id = r.GetInt32(r.GetOrdinal("Id"));
        string name = r.GetString(r.GetOrdinal("Name"));
        int age = r.GetInt32(r.GetOrdinal("Age"));

        // Email может быть NULL — проверяем
        string email = r.IsDBNull(r.GetOrdinal("Email"))
            ? null
            : r.GetString(r.GetOrdinal("Email"));

        Console.WriteLine($"{id} | {name} | {age} | {email ?? "—"}");
    }
}
```

**Метаданные ридера:**

```csharp
using (var cmd = new SqlCommand("SELECT Id, Name, Age FROM Users", conn))
using (SqlDataReader r = cmd.ExecuteReader())
{
    Console.WriteLine($"Столбцов: {r.FieldCount}");
    for (int i = 0; i < r.FieldCount; i++)
        Console.WriteLine($"  [{i}] {r.GetName(i)} : {r.GetDataTypeName(i)} (CLR: {r.GetFieldType(i).Name})");

    while (r.Read())
    {
        // Прочитать всю строку как object[]
        object[] values = new object[r.FieldCount];
        int read = r.GetValues(values);
        Console.WriteLine($"Прочитано {read} значений: {string.Join(", ", values)}");
    }
}
```

**Несколько результирующих наборов (NextResult):**

```csharp
string sql = "SELECT COUNT(*) FROM Users; SELECT TOP 3 Name FROM Users;";
using (var cmd = new SqlCommand(sql, conn))
using (SqlDataReader r = cmd.ExecuteReader())
{
    // Результат 1: COUNT
    if (r.Read())
        Console.WriteLine("Пользователей: " + r.GetInt32(0));

    // Результат 2: имена
    r.NextResult();
    while (r.Read())
        Console.WriteLine("Имя: " + r.GetString(0));
}
```

**Чтение как DataTable через DataAdapter** — удобно, когда нужен весь результат в памяти:

```csharp
using (var cmd = new SqlCommand("SELECT * FROM Users WHERE Age > @a", conn))
{
    cmd.Parameters.Add("@a", SqlDbType.Int).Value = 18;

    using (var adapter = new SqlDataAdapter(cmd))
    {
        DataTable dt = new DataTable();
        adapter.Fill(dt);
        Console.WriteLine($"Загружено строк: {dt.Rows.Count}");
        foreach (DataRow row in dt.Rows)
            Console.WriteLine($"{row["Id"]} | {row["Name"]} | {row["Age"]}");
    }
}
```

### 7.4 ExecuteXmlReader — для `FOR XML`

Если запрос возвращает XML (например, `SELECT ... FOR XML PATH`), можно получить `XmlReader`:

```csharp
using (var cmd = new SqlCommand(
    "SELECT Id, Name FROM Users FOR XML PATH('User'), ROOT('Users')", conn))
using (XmlReader xr = cmd.ExecuteXmlReader())
{
    XDocument doc = XDocument.Load(xr);
    Console.WriteLine(doc.ToString());
}
```

### 7.5 Асинхронные варианты

Все методы выполнения имеют асинхронные аналоги: `ExecuteNonQueryAsync`, `ExecuteScalarAsync`, `ExecuteReaderAsync` — принимают `CancellationToken`:

```csharp
using (var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5)))
{
    using (var cmd = new SqlCommand("WAITFOR DELAY '00:00:02'; SELECT 42", conn))
    {
        await conn.OpenAsync(cts.Token);
        object result = await cmd.ExecuteScalarAsync(cts.Token);
        Console.WriteLine("Результат: " + result);
    }
}
```

---

# Часть III. SqlParameter — параметризация

Параметры — обязательный инструмент. Они защищают от SQL-инъекций и корректно передают типы.

## 8. Способы добавления параметров

### 8.1 AddWithValue — просто, но с оговорками

```csharp
cmd.Parameters.AddWithValue("@name", "Иван");
cmd.Parameters.AddWithValue("@age", 30);
```

**Проблема:** `AddWithValue` выводит тип из CLR-значения. Строка `"Иван"` → `nvarchar(размер по значению)`. Это может привести к неверным планам запроса (разные планы для разных длин) и к неявным преобразованиям. Для продакшена предпочитайте явный тип.

### 8.2 Add с явным типом — рекомендуется

```csharp
cmd.Parameters.Add("@name", SqlDbType.NVarChar, 100).Value = "Иван";
cmd.Parameters.Add("@age", SqlDbType.Int).Value = 30;
cmd.Parameters.Add("@price", SqlDbType.Decimal).Value = 199.99m;
// точность и масштаб для decimal:
var p = cmd.Parameters.Add("@amount", SqlDbType.Decimal);
p.Precision = 18;
p.Scale = 2;
p.Value = 1234.56m;
```

**Основные SqlDbType:**

| SqlDbType | C# тип | Примечание |
|-----------|--------|-----------|
| `Int` | `int` | |
| `BigInt` | `long` | |
| `SmallInt` | `short` | |
| `TinyInt` | `byte` | |
| `Bit` | `bool` | |
| `Decimal` | `decimal` | Требует `Precision`/`Scale` |
| `Money` | `decimal` | |
| `Float` | `double` | |
| `Real` | `float` | |
| `NVarChar` | `string` | Unicode, длина обязательна (или `-1` для MAX) |
| `VarChar` | `string` | ANSI |
| `NChar` | `string` | Фиксированная длина |
| `DateTime` | `DateTime` | `datetime` (точность ~3 мс) |
| `DateTime2` | `DateTime` | `datetime2` (точнее) |
| `Date` | `DateTime` | Только дата |
| `Time` | `TimeSpan` | Только время |
| `UniqueIdentifier` | `Guid` | |
| `VarBinary` | `byte[]` | Бинарные данные |

### 8.3 Параметры для `NVarChar(MAX)` и `VarBinary(MAX)`

```csharp
// Строка произвольной длины
cmd.Parameters.Add("@body", SqlDbType.NVarChar, -1).Value = longText;

// Бинарные данные произвольной длины
cmd.Parameters.Add("@data", SqlDbType.VarBinary, -1).Value = bytes;
```

### 8.4 NULL

В C# `null` недопустим как значение параметра — используйте `DBNull.Value`:

```csharp
cmd.Parameters.Add("@comment", SqlDbType.NVarChar, 500).Value =
    string.IsNullOrEmpty(comment) ? (object)DBNull.Value : comment;
```

### 8.5 Выходной параметр (Output)

Хотя `OUTPUT` чаще используется с процедурами, его можно применять и в чистом T-SQL:

```csharp
using (var cmd = new SqlCommand("SET @result = @a + @b;", conn))
{
    cmd.Parameters.Add("@a", SqlDbType.Int).Value = 3;
    cmd.Parameters.Add("@b", SqlDbType.Int).Value = 4;

    var result = new SqlParameter("@result", SqlDbType.Int)
    {
        Direction = ParameterDirection.Output
    };
    cmd.Parameters.Add(result);

    cmd.ExecuteNonQuery();
    Console.WriteLine("Сумма: " + result.Value);
}
```

**Возможные направления:**

| Direction | Смысл |
|-----------|-------|
| `Input` | По умолчанию, входной |
| `Output` | Выходной |
| `InputOutput` | И входной, и выходной |
| `ReturnValue` | Код возврата процедуры |

### 8.6 ReturnValue — код возврата

```csharp
using (var cmd = new SqlCommand("usp_DoSomething", conn))
{
    cmd.CommandType = CommandType.StoredProcedure;

    var ret = new SqlParameter("@ret", SqlDbType.Int)
    {
        Direction = ParameterDirection.ReturnValue
    };
    cmd.Parameters.Add(ret);

    cmd.Parameters.Add("@in", SqlDbType.Int).Value = 5;

    cmd.ExecuteNonQuery();
    Console.WriteLine("Код возврата: " + ret.Value);
}
```

## 9. Практические примеры параметризации

### 9.1 Поиск по подстроке (LIKE) — безопасно

```csharp
string search = "Ан";

using (var cmd = new SqlCommand(
    "SELECT Id, Name FROM Users WHERE Name LIKE @pattern", conn))
{
    // Плейсхолдеры % включаем в значение, а не в SQL
    cmd.Parameters.Add("@pattern", SqlDbType.NVarChar, 100).Value = $"%{search}%";

    using (var r = cmd.ExecuteReader())
        while (r.Read())
            Console.WriteLine($"{r.GetInt32(0)}: {r.GetString(1)}");
}
```

**Обратите внимание:** `%` внутри значения параметра — это буквальный символ, а не спецсимвол T-SQL. SQL Server его не интерпретирует, поэтому шаблон выглядит как `'%Ан%'` и работает корректно.

### 9.2 Запрос со списком `IN` — параметризация

Параметризовать `IN (@a, @b, @c)` динамически требует генерации SQL-текста, но имена параметров — фиксированные:

```csharp
int[] ids = { 1, 2, 3, 5, 8 };

// Генерируем список @p0, @p1, ...
var names = ids.Select((_, i) => "@p" + i).ToArray();
string sql = $"SELECT Id, Name FROM Users WHERE Id IN ({string.Join(",", names)})";

using (var cmd = new SqlCommand(sql, conn))
{
    for (int i = 0; i < ids.Length; i++)
        cmd.Parameters.Add("@p" + i, SqlDbType.Int).Value = ids[i];

    using (var r = cmd.ExecuteReader())
        while (r.Read())
            Console.WriteLine($"{r.GetInt32(0)}: {r.GetString(1)}");
}
```

**Нельзя** сделать `IN (@list)` с одним параметром-строкой `"1,2,3"` — сервер воспримет её как одно значение.

### 9.3 Вставка с `SCOPE_IDENTITY()` — получить Id новой строки

```csharp
using (var cmd = new SqlCommand(
    "INSERT INTO Users (Name, Age) OUTPUT INSERTED.Id VALUES (@n, @a)", conn))
{
    cmd.Parameters.Add("@n", SqlDbType.NVarChar, 100).Value = "Мария";
    cmd.Parameters.Add("@a", SqlDbType.Int).Value = 27;

    int newId = (int)cmd.ExecuteScalar();
    Console.WriteLine("Новый Id: " + newId);
}
```

### 9.4 Пакетная вставка через один Command + несколько наборов параметров

`SqlCommand` не поддерживает наборы параметров как `DbCommand` в некоторых провайдерах — придётся либо строить многострочный INSERT, либо повторять команду. Пример многострочной параметризованной вставки:

```csharp
var users = new[] { ("Ольга", 33), ("Пётр", 41), ("Елена", 25) };

var valuesSql = string.Join(",", users.Select((_, i) => $"(@n{i}, @a{i})"));
string sql = $"INSERT INTO Users (Name, Age) VALUES {valuesSql}";

using (var cmd = new SqlCommand(sql, conn))
{
    for (int i = 0; i < users.Length; i++)
    {
        cmd.Parameters.Add($"@n{i}", SqlDbType.NVarChar, 100).Value = users[i].Item1;
        cmd.Parameters.Add($"@a{i}", SqlDbType.Int).Value = users[i].Item2;
    }
    Console.WriteLine("Вставлено: " + cmd.ExecuteNonQuery());
}
```

### 9.5 Работа с датами и временем

```csharp
using (var cmd = new SqlCommand(
    "SELECT Id FROM Orders WHERE CreatedAt >= @from AND CreatedAt < @to", conn))
{
    cmd.Parameters.Add("@from", SqlDbType.DateTime2).Value = new DateTime(2024, 1, 1);
    cmd.Parameters.Add("@to",   SqlDbType.DateTime2).Value = new DateTime(2025, 1, 1);

    using (var r = cmd.ExecuteReader())
        while (r.Read())
            Console.WriteLine("Order Id: " + r.GetInt32(0));
}
```

### 9.6 Бинарные данные (byte[])

```csharp
byte[] avatar = File.ReadAllBytes(@"C:\avatar.png");

using (var cmd = new SqlCommand(
    "UPDATE Users SET Avatar = @img WHERE Id = @id", conn))
{
    cmd.Parameters.Add("@img", SqlDbType.VarBinary, -1).Value = avatar;
    cmd.Parameters.Add("@id", SqlDbType.Int).Value = 1;
    cmd.ExecuteNonQuery();
}
```

### 9.7 Параметр Table-Valued (TVP) — кратко

Для передачи таблицы значений (обычно в процедуру) используется `SqlDbType.Structured`. Требуется заранее создать пользовательский тип в БД. Пример:

```csharp
// В БД: CREATE TYPE IdList AS TABLE (Id INT);
var dt = new DataTable();
dt.Columns.Add("Id", typeof(int));
dt.Rows.Add(1);
dt.Rows.Add(2);

using (var cmd = new SqlCommand(
    "SELECT Name FROM Users WHERE Id IN (SELECT Id FROM @ids)", conn))
{
    var p = cmd.Parameters.AddWithValue("@ids", dt);
    p.SqlDbType = SqlDbType.Structured;
    p.TypeName = "IdList"; // имя TVP-типа в БД

    using (var r = cmd.ExecuteReader())
        while (r.Read())
            Console.WriteLine(r.GetString(0));
}
```

## 10. Хранимые процедуры

`CommandType.StoredProcedure` + параметры — стандартный путь.

### 10.1 Простая процедура

```csharp
using (var cmd = new SqlCommand("usp_GetUsersByAge", conn))
{
    cmd.CommandType = CommandType.StoredProcedure;
    cmd.Parameters.Add("@minAge", SqlDbType.Int).Value = 18;

    using (var r = cmd.ExecuteReader())
        while (r.Read())
            Console.WriteLine($"{r.GetInt32(0)}: {r.GetString(1)}");
}
```

### 10.2 Процедура с Output и ReturnValue

```csharp
using (var cmd = new SqlCommand("usp_CreateUser", conn))
{
    cmd.CommandType = CommandType.StoredProcedure;

    cmd.Parameters.Add("@name", SqlDbType.NVarChar, 100).Value = "Новый";
    cmd.Parameters.Add("@age", SqlDbType.Int).Value = 20;

    var newId = new SqlParameter("@newId", SqlDbType.Int) { Direction = ParameterDirection.Output };
    cmd.Parameters.Add(newId);

    var ret = new SqlParameter("@ret", SqlDbType.Int) { Direction = ParameterDirection.ReturnValue };
    cmd.Parameters.Add(ret);

    cmd.ExecuteNonQuery();

    Console.WriteLine($"Новый Id: {newId.Value}, код возврата: {ret.Value}");
}
```

---

# Часть IV. Полные готовые к запуску примеры

## 11. Полный демонстрационный пример

```csharp
using System;
using System.Data;
using Microsoft.Data.SqlClient;

class Program
{
    static string cs =
        "Server=localhost;Database=TestDB;Integrated Security=true;TrustServerCertificate=true;" +
        "Application Name=AdoDemo;Connect Timeout=15;";

    static void Main()
    {
        CreateTable();
        InsertUsers();
        QueryUsers();
        UpdateUsers();
        UseStoredProcedure();
        DemonstrateSchema();
    }

    static void CreateTable()
    {
        using var conn = new SqlConnection(cs);
        conn.Open();
        const string sql = @"
            IF OBJECT_ID('dbo.DemoUsers','U') IS NULL
            CREATE TABLE dbo.DemoUsers (
                Id INT IDENTITY(1,1) PRIMARY KEY,
                Name NVARCHAR(100) NOT NULL,
                Age INT NOT NULL,
                Email NVARCHAR(200) NULL
            );";
        using var cmd = new SqlCommand(sql, conn);
        cmd.ExecuteNonQuery();
        Console.WriteLine("[+] Таблица DemoUsers готова");
    }

    static void InsertUsers()
    {
        using var conn = new SqlConnection(cs);
        conn.Open();

        var users = new (string Name, int Age, string Email?)[]
        {
            ("Анна", 22, "anna@example.com"),
            ("Борис", 35, null),
            ("Вера", 28, "vera@example.com"),
            ("Глеб", 41, "gleb@example.com"),
        };

        // Одна команда, многострочный INSERT, OUTPUT Id
        var placeholders = string.Join(",",
            users.Select((_, i) => $"(@n{i}, @a{i}, @e{i})"));
        string sql = $"INSERT INTO DemoUsers (Name, Age, Email) VALUES {placeholders}; " +
                     $"SELECT COUNT(*) FROM DemoUsers;";

        using var cmd = new SqlCommand(sql, conn);
        for (int i = 0; i < users.Length; i++)
        {
            cmd.Parameters.Add($"@n{i}", SqlDbType.NVarChar, 100).Value = users[i].Name;
            cmd.Parameters.Add($"@a{i}", SqlDbType.Int).Value = users[i].Age;
            cmd.Parameters.Add($"@e{i}", SqlDbType.NVarChar, 200).Value =
                (object?)users[i].Email ?? DBNull.Value;
        }
        int total = (int)cmd.ExecuteScalar()!;
        Console.WriteLine($"[+] Всего пользователей: {total}");
    }

    static void QueryUsers()
    {
        using var conn = new SqlConnection(cs);
        conn.Open();
        using var cmd = new SqlCommand(
            "SELECT Id, Name, Age, Email FROM DemoUsers WHERE Age >= @min ORDER BY Age", conn);
        cmd.Parameters.Add("@min", SqlDbType.Int).Value = 18;

        using var r = cmd.ExecuteReader();
        Console.WriteLine("[+] Пользователи (возраст >= 18):");
        while (r.Read())
        {
            int id = r.GetInt32(0);
            string name = r.GetString(1);
            int age = r.GetInt32(2);
            string email = r.IsDBNull(3) ? "—" : r.GetString(3);
            Console.WriteLine($"    {id} | {name,-10} | {age,3} | {email}");
        }
    }

    static void UpdateUsers()
    {
        using var conn = new SqlConnection(cs);
        conn.Open();
        using var cmd = new SqlCommand(
            "UPDATE DemoUsers SET Age = Age + 1 WHERE Age < @max", conn);
        cmd.Parameters.Add("@max", SqlDbType.Int).Value = 30;
        int updated = cmd.ExecuteNonQuery();
        Console.WriteLine($"[+] Обновлено строк: {updated}");
    }

    static void UseStoredProcedure()
    {
        using var conn = new SqlConnection(cs);
        conn.Open();

        // Создаём процедуру на лету
        const string create = @"
            CREATE OR ALTER PROCEDURE dbo.usp_DemoCount
                @minAge INT,
                @total INT OUTPUT
            AS
            BEGIN
                SET NOCOUNT ON;
                SELECT @total = COUNT(*) FROM DemoUsers WHERE Age >= @minAge;
            END;";
        using (var c = new SqlCommand(create, conn)) c.ExecuteNonQuery();

        // Вызываем
        using var cmd = new SqlCommand("dbo.usp_DemoCount", conn);
        cmd.CommandType = CommandType.StoredProcedure;
        cmd.Parameters.Add("@minAge", SqlDbType.Int).Value = 30;

        var outParam = new SqlParameter("@total", SqlDbType.Int) { Direction = ParameterDirection.Output };
        cmd.Parameters.Add(outParam);

        cmd.ExecuteNonQuery();
        Console.WriteLine($"[+] usp_DemoCount (>=30): {outParam.Value}");
    }

    static void DemonstrateSchema()
    {
        using var conn = new SqlConnection(cs);
        conn.Open();

        var tables = conn.GetSchema("Tables", new[] { "TestDB", null, null, null });
        Console.WriteLine("[+] Таблицы в TestDB:");
        foreach (DataRow row in tables.Rows)
            Console.WriteLine($"    {row["TABLE_SCHEMA"]}.{row["TABLE_NAME"]}");

        var cols = conn.GetSchema("Columns", new[] { "TestDB", "dbo", "DemoUsers", null });
        Console.WriteLine("[+] Столбцы DemoUsers:");
        foreach (DataRow row in cols.Rows)
            Console.WriteLine($"    {row["COLUMN_NAME"]} : {row["DATA_TYPE"]}");
    }
}
```

**Что демонстрирует пример:**

1. Установка соединения с явными параметрами строки.
2. `ExecuteNonQuery` для DDL (`CREATE TABLE`).
3. Многострочный параметризованный `INSERT` + `ExecuteScalar`.
4. `ExecuteReader` с проверкой `IsDBNull` и типизированным чтением.
5. `ExecuteNonQuery` для `UPDATE`.
6. Вызов хранимой процедуры с `OUTPUT`-параметром.
7. `GetSchema` для метаданных.

## 12. Пример: обработка исключений и таймаутов

```csharp
using System;
using System.Data;
using Microsoft.Data.SqlClient;

class ErrorHandling
{
    static string cs = "Server=localhost;Database=TestDB;Integrated Security=true;TrustServerCertificate=true;";

    public static void Run()
    {
        try
        {
            using var conn = new SqlConnection(cs);
            conn.Open();

            using var cmd = new SqlCommand("SELECT 1/0", conn); // деление на ноль
            cmd.ExecuteScalar();
        }
        catch (SqlException ex) when (ex.Number == 8134)
        {
            // 8134 — divide by zero
            Console.WriteLine("Ошибка деления на ноль в SQL: " + ex.Message);
        }
        catch (SqlException ex) when (ex.Number == -2)
        {
            // -2 — таймаут
            Console.WriteLine("Таймаут: " + ex.Message);
        }
        catch (SqlException ex)
        {
            Console.WriteLine($"SQL-ошибка #{ex.Number}: {ex.Message}");
            foreach (SqlError e in ex.Errors)
                Console.WriteLine($"  [{e.Class}] {e.Message} (proc: {e.Procedure}, line: {e.LineNumber})");
        }
        catch (InvalidOperationException ex)
        {
            // например, соединение закрыто
            Console.WriteLine("Ошибка состояния соединения: " + ex.Message);
        }
    }
}
```

---

# Часть V. Сводная таблица ключевых правил

| Правило | Пояснение |
|---------|-----------|
| **using на всё** | `SqlConnection`, `SqlCommand`, `SqlDataReader`, `SqlDataAdapter` реализуют `IDisposable` |
| **Открывайте соединение поздно, закрывайте рано** | Пул сам оптимизирует физические соединения |
| **Всегда параметры, никогда конкатенация** | Защита от инъекций |
| **Указывайте `SqlDbType` явно** | `Add("@p", SqlDbType.X)` вместо `AddWithValue` |
| **`DBNull.Value` вместо `null`** | Для SQL NULL |
| **Проверяйте `IsDBNull`** | Перед `GetString`/`GetInt32` |
| **`ExecuteScalar` может вернуть `null`** | Проверяйте результат |
| **`CommandTimeout` по умолчанию 30 с** | Увеличивайте для тяжёлых запросов |
| **`CommandType.StoredProcedure` для процедур** | Имя без `EXEC` |
| **`OUTPUT INSERTED.Id` вместо `SELECT SCOPE_IDENTITY()`** | Более безопасно и атомарно |
| **`GetSchema` для метаданных** | Быстрый способ узнать структуру БД |
| **`InfoMessage` для PRINT/RAISERROR(10)** | Информационные сообщения сервера |
| **Ловите `SqlException.Number`** | Различайте типы ошибок |
| **Асинхронные методы в веб-приложениях** | Не блокируйте потоки пула |
