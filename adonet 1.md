

# Тема 1. Архитектура ADO.NET и базовые классы (MSSQL)

## 1.1. Провайдер

Для MSSQL в современном .NET используется пакет `Microsoft.Data.SqlClient`. Старый `System.Data.SqlClient` оставлен для совместимости, но новые возможности (Entra ID, Always Encrypted, новые типы) есть только в `Microsoft.Data.SqlClient`.

```csharp
// Установка пакета:
// dotnet add package Microsoft.Data.SqlClient

using Microsoft.Data.SqlClient;
```

## 1.2. Базовые классы и их назначение

| Класс | Назначение |
|---|---|
| `SqlConnection` | Физическое соединение с сервером |
| `SqlCommand` | SQL-инструкция или хранимая процедура |
| `SqlDataReader` | Потоковое чтение результата только вперёд |
| `SqlDataAdapter` | Мост между базой и `DataTable`/`DataSet` |
| `SqlParameter` | Параметр команды |
| `SqlTransaction` | Транзакция на соединении |
| `SqlException` | Ошибка SQL Server (коллекция `Errors`) |

## 1.3. Пример: общий метод чтения через базовый `DbConnection`

```csharp
using System.Data.Common;
using Microsoft.Data.SqlClient;

// Принимаем DbConnection, чтобы метод не был жёстко привязан к SqlConnection.
// Для MSSQL сюда всегда придёт SqlConnection, но сигнатура остаётся универсальной.
public static List<string> GetNames(DbConnection connection, string sql)
{
    var results = new List<string>();

    // CreateCommand() создаёт команду того же типа, что и соединение
    using (DbCommand cmd = connection.CreateCommand())
    {
        cmd.CommandText = sql;
        connection.Open(); // открываем здесь; вызывающий код не должен открывать повторно
        try
        {
            using (DbDataReader reader = cmd.ExecuteReader())
            {
                while (reader.Read()) // false — строки закончились
                    results.Add(reader.GetString(0)); // по ординалу — быстрее
            }
        }
        finally
        {
            connection.Close(); // закрываем в finally, чтобы не утекало при исключении
        }
    }
    return results;
}

// Вызов:
using (var conn = new SqlConnection(DbConfig.ConnectionString))
{
    var names = GetNames(conn, "SELECT Name FROM Users");
}
```

**Комментарий:** если написать `new SqlCommand(sql, connection)`, метод станет жёстко привязан к SqlClient. `connection.CreateCommand()` возвращает команду, соответствующую типу соединения.

## 1.4. Пример: DataAdapter как мост между MSSQL и DataTable

```csharp
using System.Data;
using Microsoft.Data.SqlClient;

using (var conn = new SqlConnection(DbConfig.ConnectionString))
using (var adapter = new SqlDataAdapter("SELECT Id, Name, Price FROM Products", conn))
{
    // CommandBuilder автоматически строит INSERT/UPDATE/DELETE
    // на основе SelectCommand. Работает только для простых запросов с одним SELECT.
    using (var builder = new SqlCommandBuilder(adapter))
    {
        var table = new DataTable();
        adapter.Fill(table); // открывает и закрывает соединение сам

        table.Rows[0]["Price"] = 123.45m; // меняем в памяти
        adapter.Update(table);            // отправляем обратно одним пакетом
    }
}
```

**Важно:** `CommandBuilder` не работает с JOIN, подзапросами и хранимыми процедурами. Для сложных запросов команды INSERT/UPDATE/DELETE задаются вручную.

---

# Тема 2. Connection и строка подключения (консольный C#, MSSQL)

## 2.1. Базовый пример с using

```csharp
using Microsoft.Data.SqlClient;

string connectionString =
    "Server=localhost;Database=TestDb;Integrated Security=true;TrustServerCertificate=true";

using (var conn = new SqlConnection(connectionString))
{
    try
    {
        conn.Open(); // физическое соединение или взятие из пула
        Console.WriteLine(conn.State); // Open

        using (var cmd = new SqlCommand("SELECT @@VERSION", conn))
        {
            Console.WriteLine(cmd.ExecuteScalar());
        }
    }
    catch (SqlException ex)
    {
        // SqlException может содержать несколько ошибок сразу
        foreach (SqlError err in ex.Errors)
            Console.WriteLine($"#{err.Number}: {err.Message} (Severity {err.Class})");
    }
} // Close() вызывается автоматически, соединение возвращается в пул
```

## 2.2. Ключевые слова строки подключения

```csharp
string cs = string.Join(";",
    "Server=tcp:myserver.database.windows.net,1433", // хост и порт
    "Database=AppDb",                                 // имя базы
    "User Id=appuser",                                // логин SQL-аутентификации
    "Password=StrongP@ss",                            // пароль
    "Encrypt=True",                                   // шифрование канала (по умолчанию True в новых версиях)
    "TrustServerCertificate=False",                   // проверять сертификат сервера
    "Connection Timeout=15",                          // таймаут установки соединения (сек)
    "Command Timeout=30",                             // таймаут выполнения команды (сек)
    "Pooling=True",                                   // использовать пул (по умолчанию True)
    "Min Pool Size=0",                                // минимум соединений в пуле
    "Max Pool Size=100",                              // максимум соединений в пуле
    "MultipleActiveResultSets=False"                  // MARS — по умолчанию выключен
);
```

**Нюанс:** порядок ключевых слов влияет на ключ пула. Две строки с одинаковым набором, но разным порядком, создают **два разных пула**.

## 2.3. Интегрированная аутентификация Windows

```csharp
// Вход под текущей учётной записью Windows
string cs = "Server=localhost;Database=TestDb;Integrated Security=true;TrustServerCertificate=true";

// Эквивалентная запись
string cs2 = "Server=localhost;Database=TestDb;Trusted_Connection=true;TrustServerCertificate=true";
```

## 2.4. Аутентификация через AccessToken (Entra ID)

```csharp
using (var conn = new SqlConnection("Server=myserver.database.windows.net;Database=AppDb"))
{
    conn.AccessToken = await GetAccessTokenAsync(); // например, из Azure.Identity
    conn.Open();
}
```

**Нюанс:** значение токена входит в ключ пула. Замена токена создаёт новый пул, а старые соединения остаются аутентифицированы старым токеном. После замены просроченного токена может потребоваться `SqlConnection.ClearPool(conn)`.

## 2.5. Конфигурационный файл: appsettings.json (консольное приложение)

### Файл appsettings.json (копируется в выходную папку)

```json
{
  "ConnectionStrings": {
    "Default": "Server=localhost;Database=AppDb;Integrated Security=true;TrustServerCertificate=true",
    "Reporting": "Server=report-srv;Database=Reports;User Id=rep;Password=StrongP@ss;Encrypt=True"
  },
  "Database": {
    "CommandTimeout": 30,
    "MaxRetries": 3
  }
}
```

### Чтение в Program.cs через ConfigurationBuilder

```csharp
using Microsoft.Extensions.Configuration;

// Консольное приложение: строим конфигурацию вручную
var config = new ConfigurationBuilder()
    .SetBasePath(Directory.GetCurrentDirectory())
    .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
    .AddEnvironmentVariables() // переменные окружения перекрывают файл
    .Build();

// GetConnectionString ищет ключ в секции ConnectionStrings
string cs = config.GetConnectionString("Default")
    ?? throw new InvalidOperationException("Строка подключения 'Default' не найдена");

// Чтение скалярных настроек
int commandTimeout = config.GetValue<int>("Database:CommandTimeout", 30);

using (var conn = new SqlConnection(cs))
{
    conn.Open();
    using (var cmd = new SqlCommand("SELECT 1", conn))
    {
        cmd.CommandTimeout = commandTimeout;
        Console.WriteLine(cmd.ExecuteScalar());
    }
}
```

**Важно:** в `.csproj` файл `appsettings.json` должен копироваться в выходную папку:

```xml
<ItemGroup>
  <None Update="appsettings.json">
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
  </None>
</ItemGroup>
```

### Класс-обёртка для строки подключения

```csharp
public static class DbConfig
{
    // Загружается один раз при старте приложения
    public static string ConnectionString { get; private set; } = null!;

    public static void Initialize(IConfiguration config)
    {
        ConnectionString = config.GetConnectionString("Default")
            ?? throw new InvalidOperationException("Строка подключения 'Default' не найдена");
    }
}

// Program.cs
var config = new ConfigurationBuilder()
    .SetBasePath(Directory.GetCurrentDirectory())
    .AddJsonFile("appsettings.json", optional: false)
    .Build();

DbConfig.Initialize(config);

// Далее везде используется DbConfig.ConnectionString
using (var conn = new SqlConnection(DbConfig.ConnectionString)) { /* ... */ }
```

### app.config (для .NET Framework-консоли)

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <connectionStrings>
    <add name="Default"
         connectionString="Server=localhost;Database=AppDb;Integrated Security=true"
         providerName="System.Data.SqlClient" />
  </connectionStrings>
</configuration>
```

```csharp
// Чтение через ConfigurationManager (требуется ссылка на System.Configuration)
using System.Configuration;
string cs = ConfigurationManager.ConnectionStrings["Default"].ConnectionString;
```

### Защита строки подключения в консольном приложении

```csharp
// 1. Dev: User Secrets — не храните пароль в репозитории
// dotnet user-secrets init
// dotnet user-secrets set "ConnectionStrings:Default" "Server=...;Password=..."

// 2. Prod: переменные окружения
// set ConnectionStrings__Default=Server=...;Password=...

// В коде AddEnvironmentVariables() уже подключён — переменные окружения
// автоматически перекроют значения из appsettings.json
```

---

# Тема 3. Command: ExecuteNonQuery, ExecuteScalar, ExecuteReader (MSSQL)

## 3.1. ExecuteNonQuery

```csharp
using (var conn = new SqlConnection(DbConfig.ConnectionString))
{
    conn.Open();

    // INSERT: возвращает число вставленных строк
    using (var cmd = new SqlCommand("INSERT INTO Logs (Message, CreatedAt) VALUES (@m, @d)", conn))
    {
        cmd.Parameters.Add("@m", SqlDbType.NVarChar, 500).Value = "Старт приложения";
        cmd.Parameters.Add("@d", SqlDbType.DateTime2).Value = DateTime.UtcNow;
        int inserted = cmd.ExecuteNonQuery(); // 1
    }

    // UPDATE: возвращает число изменённых строк
    using (var cmd = new SqlCommand("UPDATE Users SET IsActive = 0 WHERE LastLogin < @cutoff", conn))
    {
        cmd.Parameters.Add("@cutoff", SqlDbType.DateTime2).Value = DateTime.UtcNow.AddYears(-1);
        int affected = cmd.ExecuteNonQuery();
        Console.WriteLine($"Деактивировано: {affected}");
    }

    // DDL: возвращает -1, а не 0
    using (var cmd = new SqlCommand("CREATE TABLE #Temp (Id int)", conn))
    {
        int ddlResult = cmd.ExecuteNonQuery(); // -1
    }
}
```

## 3.2. ExecuteScalar

```csharp
using (var conn = new SqlConnection(DbConfig.ConnectionString))
{
    conn.Open();

    // COUNT: всегда возвращает число, даже если 0
    using (var cmd = new SqlCommand("SELECT COUNT(*) FROM Orders WHERE Status = @s", conn))
    {
        cmd.Parameters.Add("@s", SqlDbType.NVarChar, 20).Value = "Pending";
        int count = (int)cmd.ExecuteScalar(); // 0, если строк нет
    }

    // MAX: может вернуть DBNull, если в таблице нет строк или все значения NULL
    using (var cmd = new SqlCommand("SELECT MAX(Price) FROM Products WHERE CategoryId = @c", conn))
    {
        cmd.Parameters.Add("@c", SqlDbType.Int).Value = 5;
        object result = cmd.ExecuteScalar();

        // result == null  → запрос не вернул ни одной строки
        // result == DBNull.Value → агрегат вернул NULL
        decimal? maxPrice = (result == null || result == DBNull.Value)
            ? (decimal?)null
            : Convert.ToDecimal(result);
    }

    // INSERT ... OUTPUT INSERTED.Id — вернуть сгенерированный ID
    using (var cmd = new SqlCommand(
        "INSERT INTO Users (Name) OUTPUT INSERTED.Id VALUES (@n)", conn))
    {
        cmd.Parameters.Add("@n", SqlDbType.NVarChar, 100).Value = "Иван";
        int newId = (int)cmd.ExecuteScalar();
    }

    // SCOPE_IDENTITY() возвращает decimal — нужно Convert.ToInt32, а не (int)
    using (var cmd = new SqlCommand(
        "INSERT INTO Users (Name) VALUES (@n); SELECT SCOPE_IDENTITY();", conn))
    {
        cmd.Parameters.Add("@n", SqlDbType.NVarChar, 100).Value = "Пётр";
        int newId2 = Convert.ToInt32(cmd.ExecuteScalar());
    }
}
```

## 3.3. ExecuteReader: базовое чтение

```csharp
using (var conn = new SqlConnection(DbConfig.ConnectionString))
{
    conn.Open();
    using (var cmd = new SqlCommand("SELECT Id, Name, Email FROM Users", conn))
    using (var reader = cmd.ExecuteReader())
    {
        if (reader.HasRows) // можно узнать о наличии данных до цикла
        {
            while (reader.Read())
            {
                int id = reader.GetInt32(0);       // по ординалу — быстрее
                string name = reader.GetString(1);

                // Проверка NULL обязательна — иначе InvalidCastException
                string email = reader.IsDBNull(2) ? null : reader.GetString(2);

                // По имени — медленнее (поиск в метаданных), но читаемее
                // int id2 = reader.GetInt32(reader.GetOrdinal("Id"));
            }
        }
    }
}
```

## 3.4. ExecuteReader: несколько результирующих наборов

```csharp
using (var cmd = new SqlCommand("SELECT Id FROM Users; SELECT Id FROM Orders;", conn))
using (var reader = cmd.ExecuteReader())
{
    // Первый набор
    while (reader.Read())
        Console.WriteLine($"User: {reader.GetInt32(0)}");

    // Переход ко второму набору
    if (reader.NextResult())
    {
        while (reader.Read())
            Console.WriteLine($"Order: {reader.GetInt32(0)}");
    }
}
```

## 3.5. ExecuteReader: CommandBehavior

```csharp
// SequentialAccess — для больших BLOB: читать столбцы по порядку, потоково
using (var cmd = new SqlCommand("SELECT Id, Bio FROM Users", conn))
using (var reader = cmd.ExecuteReader(CommandBehavior.SequentialAccess))
{
    while (reader.Read())
    {
        int id = reader.GetInt32(0);

        // GetChars/GetBytes вместо GetString для больших полей
        if (!reader.IsDBNull(1))
        {
            long length = reader.GetChars(1, 0, null, 0, 0); // узнать длину
            var buffer = new char[length];
            reader.GetChars(1, 0, buffer, 0, (int)length);
        }
    }
}

// CloseConnection — соединение закроется вместе с reader
public SqlDataReader GetReader(SqlConnection conn)
{
    var cmd = new SqlCommand("SELECT * FROM Logs", conn);
    return cmd.ExecuteReader(CommandBehavior.CloseConnection);
}
```

## 3.6. ExecuteReader: GetSqlTypes против Get*

```csharp
// GetSqlInt32 возвращает SqlInt32 — нативный тип SQL Server, без boxing
using (var reader = cmd.ExecuteReader())
{
    while (reader.Read())
    {
        SqlInt32 id = reader.GetSqlInt32(0);     // может быть SqlInt32.Null
        SqlString name = reader.GetSqlString(1); // может быть SqlString.Null

        if (!id.IsNull) Console.WriteLine(id.Value);
        if (!name.IsNull) Console.WriteLine(name.Value);
    }
}
```

## 3.7. Параметризация: все варианты

```csharp
using (var cmd = new SqlCommand(
    "SELECT * FROM Products WHERE CategoryId = @Cat AND Price < @Max AND Name LIKE @Pattern", conn))
{
    // Явный тип — рекомендуется
    cmd.Parameters.Add("@Cat", SqlDbType.Int).Value = 5;
    cmd.Parameters.Add("@Max", SqlDbType.Decimal).Value = 1000m;

    // NVarChar с размером; -1 означает MAX
    cmd.Parameters.Add("@Pattern", SqlDbType.NVarChar, 100).Value = "%телефон%";

    // NULL: DBNull, а не C# null
    cmd.Parameters.Add("@Desc", SqlDbType.NVarChar, 500).Value =
        string.IsNullOrEmpty(desc) ? DBNull.Value : (object)desc;

    // AddWithValue — короче, но тип выводится из значения
    cmd.Parameters.AddWithValue("@Other", someValue);
}
```

**Нюанс:** `AddWithValue` со строкой выводит `NVarChar` с размером до 4000. Если строка длиннее — нужно явно задать `SqlDbType.NVarChar, -1`.

## 3.8. Хранимые процедуры: все направления параметров

```csharp
using (var cmd = new SqlCommand("usp_ProcessOrder", conn))
{
    cmd.CommandType = CommandType.StoredProcedure;

    // Входной
    cmd.Parameters.Add("@OrderId", SqlDbType.Int).Value = 100;

    // InputOutput: передаём значение, получаем изменённое
    var io = cmd.Parameters.Add("@Status", SqlDbType.NVarChar, 50);
    io.Direction = ParameterDirection.InputOutput;
    io.Value = "Pending";

    // Output: только возврат
    var outParam = cmd.Parameters.Add("@ProcessedAt", SqlDbType.DateTime2);
    outParam.Direction = ParameterDirection.Output;

    // ReturnValue: код возврата процедуры (RETURN @code) — добавлять ПЕРВЫМ
    var rc = cmd.Parameters.Add("@RC", SqlDbType.Int);
    rc.Direction = ParameterDirection.ReturnValue;

    cmd.ExecuteNonQuery();

    string newStatus = (string)io.Value;
    DateTime? processedAt = outParam.Value == DBNull.Value ? null : (DateTime?)outParam.Value;
    int returnCode = (int)rc.Value;
}
```

---

# Тема 4. Параметры: типы, NULL, вывод типов, ловушки (MSSQL)

## 4.1. Почему конкатенация опасна

```csharp
// ПЛОХО: SQL-инъекция
string sql = "SELECT * FROM Users WHERE Name = '" + userInput + "'";
// Если userInput = "' OR '1'='1", вернутся все пользователи

// ХОРОШО: параметр
string sql2 = "SELECT * FROM Users WHERE Name = @name";
cmd.Parameters.Add("@name", SqlDbType.NVarChar, 100).Value = userInput;
```

## 4.2. Вывод типов и его ловушки

```csharp
// string → NVarChar(size до 4000). Для длинных строк явно указываем MAX:
cmd.Parameters.Add("@Body", SqlDbType.NVarChar, -1).Value = longText;

// byte[] > 8000 → нужен VarBinary(MAX), иначе ошибка «слишком длинный параметр»
cmd.Parameters.Add("@Data", SqlDbType.VarBinary, -1).Value = bigBytes;

// decimal: точность и масштаб важны для денег
cmd.Parameters.Add("@Amount", SqlDbType.Decimal).Value = 123.45m;
cmd.Parameters["@Amount"].Precision = 18;
cmd.Parameters["@Amount"].Scale = 2;

// DateTime vs DateTime2: DateTime2 точнее и имеет больший диапазон
cmd.Parameters.Add("@When", SqlDbType.DateTime2).Value = DateTime.UtcNow;
```

## 4.3. NULL: DBNull.Value

```csharp
// C# null означает «параметр не задан» → ошибка «parameter not supplied»
// DBNull.Value означает «значение в базе NULL»
cmd.Parameters.Add("@Note", SqlDbType.NVarChar, 200).Value =
    note == null ? DBNull.Value : (object)note;

// При чтении: IsDBNull перед Get*
string note = reader.IsDBNull(ordinal) ? null : reader.GetString(ordinal);
```

## 4.4. Расширение для безопасного чтения

```csharp
public static class ReaderExtensions
{
    public static T? GetNullable<T>(this SqlDataReader r, int ordinal) where T : struct
        => r.IsDBNull(ordinal) ? (T?)null : r.GetFieldValue<T>(ordinal);

    public static string? GetNullableString(this SqlDataReader r, int ordinal)
        => r.IsDBNull(ordinal) ? null : r.GetString(ordinal);
}

// Использование
decimal? price = reader.GetNullable<decimal>(2);
string? bio = reader.GetNullableString(3);
```

## 4.5. Ловушка `as int?`

```csharp
// Работает, но молча проглатывает неверный тип:
object value = reader["Age"]; // в базе smallint
int? age = value as int?;     // null, хотя значение есть — баг скрыт

// Правильно: явное приведение или Convert
int? age2 = reader.IsDBNull(ordinal) ? (int?)null : Convert.ToInt32(reader[ordinal]);
```

---

# Тема 5. Транзакции: Commit, Rollback, Savepoints, вложенность (MSSQL)

## 5.1. Базовая транзакция

```csharp
using (var conn = new SqlConnection(DbConfig.ConnectionString))
{
    conn.Open();
    using (var tran = conn.BeginTransaction())
    {
        try
        {
            // Каждая команда должна быть создана с этой транзакцией (3-й параметр)
            using (var cmd = new SqlCommand(
                "UPDATE Accounts SET Balance = Balance - @amt WHERE Id = @id", conn, tran))
            {
                cmd.Parameters.Add("@amt", SqlDbType.Decimal).Value = 100m;
                cmd.Parameters.Add("@id", SqlDbType.Int).Value = 1;
                if (cmd.ExecuteNonQuery() == 0)
                    throw new InvalidOperationException("Счёт не найден");
            }

            using (var cmd = new SqlCommand(
                "UPDATE Accounts SET Balance = Balance + @amt WHERE Id = @id", conn, tran))
            {
                cmd.Parameters.Add("@amt", SqlDbType.Decimal).Value = 100m;
                cmd.Parameters.Add("@id", SqlDbType.Int).Value = 2;
                cmd.ExecuteNonQuery();
            }

            tran.Commit(); // фиксируем обе операции
        }
        catch
        {
            tran.Rollback(); // откатываем всё
            throw;
        }
    }
}
```

**Нюанс:** если команда создана без транзакции (`new SqlCommand(sql, conn)`), она выполнится вне её, и `Rollback` её не затронет — данные останутся изменёнными.

## 5.2. Уровни изоляции

```csharp
// ReadCommitted — по умолчанию в SQL Server
using (var tran = conn.BeginTransaction(IsolationLevel.ReadCommitted)) { /* ... */ }

// Serializable — самый строгий, блокирует диапазоны
using (var tran = conn.BeginTransaction(IsolationLevel.Serializable)) { /* ... */ }

// Snapshot — требует включения на уровне базы; читатели не блокируют писателей
using (var tran = conn.BeginTransaction(IsolationLevel.Snapshot)) { /* ... */ }
```

## 5.3. Savepoints: частичный откат

```csharp
using (var tran = conn.BeginTransaction())
{
    try
    {
        // Шаг 1: создать клиента
        using (var cmd = new SqlCommand(
            "INSERT INTO Customers (Name) OUTPUT INSERTED.Id VALUES (@n)", conn, tran))
        {
            cmd.Parameters.Add("@n", SqlDbType.NVarChar, 100).Value = "Новый клиент";
            int customerId = (int)cmd.ExecuteScalar();

            // Точка сохранения после создания клиента
            tran.Save("AfterCustomer");

            try
            {
                // Шаг 2: заказ материалов (может упасть)
                using (var cmd2 = new SqlCommand(
                    "INSERT INTO MaterialRequests (CustomerId, PartId) VALUES (@c, @p)", conn, tran))
                {
                    cmd2.Parameters.Add("@c", SqlDbType.Int).Value = customerId;
                    cmd2.Parameters.Add("@p", SqlDbType.Int).Value = 999;
                    cmd2.ExecuteNonQuery();
                }
                tran.Commit(); // всё прошло
            }
            catch (SqlException)
            {
                // Откат только шага 2; клиент остаётся созданным
                tran.Rollback("AfterCustomer");

                // Транзакция ещё активна — нужно явно закоммитить
                tran.Commit();
            }
        }
    }
    catch
    {
        tran.Rollback(); // полный откат при других ошибках
        throw;
    }
}
```

**Нюанс:** после `Rollback("point")` транзакция остаётся активной. Если не вызвать `Commit()`, изменения до savepoint будут потеряны при закрытии соединения.

## 5.4. Вложенность и `TransactionScope`

`SqlTransaction` не поддерживает настоящую вложенность — второй `BeginTransaction()` на том же соединении выбросит исключение. Для распределённых транзакций используется `TransactionScope`, но в .NET Core на Linux он не поддерживается (нужен Windows + MSDTC).

```csharp
using System.Transactions;

// Ambient-транзакция: команды внутри scope автоматически в неё входят
using (var scope = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled))
{
    using (var conn = new SqlConnection(DbConfig.ConnectionString))
    {
        conn.Open();
        using (var cmd = new SqlCommand("INSERT INTO Logs (Message) VALUES (@m)", conn))
        {
            cmd.Parameters.Add("@m", SqlDbType.NVarChar, 200).Value = "test";
            cmd.ExecuteNonQuery();
        }
    }
    scope.Complete(); // без Complete транзакция откатится при Dispose
}
```

**Нюанс:** `TransactionScope` с несколькими соединениями/ресурсами повышает транзакцию до распределённой (MSDTC). В консольном приложении на Linux это не сработает — используйте `SqlTransaction` в рамках одного соединения.

