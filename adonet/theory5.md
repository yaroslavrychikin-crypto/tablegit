# Пул соединений в ADO.NET: полное руководство с примерами и задачами

Пул соединений (connection pool) — это, пожалуй, самая недооценённая и одновременно самая частая причина production-инцидентов в приложениях, работающих с SQL Server. Разработчики часто воспринимают `SqlConnection.Open()` как лёгкую операцию, не задумываясь о том, что за кулисами работает сложный механизм, который может как спасти приложение, так и стать источником загадочных таймаутов и «утечек» соединений. Ниже — исчерпывающая теория, рабочие примеры и тренировочные задания.


## 1. Что такое пул соединений и где он находится

**Пул соединений — это клиентский механизм, реализованный внутри процесса вашего приложения, а не на стороне SQL Server.** Это фундаментальное понимание: SQL Server не знает о существовании пула. Он видит только физические соединения, которые открывает и закрывает клиент. Пул — это оптимизация на стороне драйвера (`Microsoft.Data.SqlClient`), которая позволяет избежать повторного выполнения дорогостоящих операций: установки TCP-соединения, TLS-рукопожатия, аутентификации, проверки членства в транзакции и т.д. 

Когда вы вызываете `SqlConnection.Open()`, драйвер **не создаёт новое физическое соединение**, если в пуле есть свободное соединение с точно такой же конфигурацией. Вместо этого он «одалживает» соединение из пула. Когда вы вызываете `Close()` или `Dispose()`, соединение **не закрывается физически**, а возвращается обратно в пул для повторного использования. 

**Ключевое следствие:** если вы забыли закрыть соединение (или читатель), оно не вернётся в пул. Пул будет считать его занятым. Когда все соединения в пуле заняты, новые `Open()` будут ждать освобождения, пока не истечёт `Connect Timeout`, после чего выбросят исключение `InvalidOperationException: Timeout expired... pool... max pool size was reached`. 


## 2. Ключ пула: почему «почти одинаковые» строки подключения создают разные пулы

Пул создаётся для каждой **уникальной конфигурации соединения**. Ключ пула включает:

- **Точную строку подключения** (порядок ключевых слов имеет значение!)
- **Идентичность Windows** (при Integrated Security) — разные пользователи создают разные пулы
- **Экземпляр `SqlCredential`** (если используется) — даже с одинаковыми логином/паролем разные экземпляры объекта создают разные пулы
- **Значение `AccessToken`** (при прямом указании) — каждая замена токена создаёт новый пул
- **Экземпляр `AccessTokenCallback`** (при использовании callback) — разные экземпляры делегата = разные пулы
- **Провайдер SSPI** (при кастомном Kerberos/NTLM) — экземпляр провайдера участвует в ключе
- **Наличие ambient-транзакции** — соединения, вlisted в транзакцию, используют отдельное «подразделение» внутри пула 

**Практическое правило:** создайте **одну каноническую строку подключения** и используйте её везде. Не добавляйте в `Application Name` или `Workstation ID` значения, уникальные для каждого запроса (например, `Application Name=MyApp-{Guid}`). Это приведёт к фрагментации пулов и потенциальному исчерпанию лимита соединений на сервере. 


## 3. Параметры пула: полный разбор

Все параметры задаются в строке подключения или через `SqlConnectionStringBuilder`.

### `Pooling` (по умолчанию `true`)

Включает или отключает пул. Отключайте **только** для диагностики или для workload'ов, которые по каким-то причинам не могут безопасно пулироваться. В production всегда `true`. 

### `Min Pool Size` (по умолчанию `0`)

Минимальное количество физических соединений, которые пул **держит открытыми** после создания. Если `> 0`, пул не будет закрывать соединения при простое, и приложение всегда будет иметь «тёплые» соединения. **Осторожно:** положительное значение конфликтует с serverless-архитектурами (Azure SQL Database serverless, scale-to-zero), где простой сервер должен автоматически приостанавливаться. 

### `Max Pool Size` (по умолчанию `100`)

Максимальное количество физических соединений в **одном пуле**. Когда все заняты, `Open()` ждёт до `Connect Timeout`, затем выбрасывает исключение. **Не увеличивайте** это значение, не проверив предварительно:
- Все ли соединения и reader'ы освобождаются на всех путях кода
- Завершаются ли команды и транзакции вовремя
- Не заблокирован ли workload
- Может ли сервер выдержать `Max Pool Size` × количество пулов × количество экземпляров приложения 

### `Connect Timeout` (по умолчанию `15` секунд)

**Это не таймаут на установку TCP-соединения.** Это таймаут на **получение соединения из пула** (включая время ожидания в очереди, если пул исчерпан). Если `Open()` не может получить соединение за это время, выбрасывается `InvalidOperationException`. 

### `Load Balance Timeout` / `Connection Lifetime` (по умолчанию `0` = отключено)

Если возраст соединения при возврате в пул превышает это значение (в секундах), соединение **закрывается физически**, а не возвращается в пул. Полезно для балансировки нагрузки между серверами в кластере или для принудительного обновления соединений после изменения конфигурации сервера. 

### `Connection Idle Timeout` (по умолчанию `300` секунд, начиная с версии 7.1.0)

Соединение, простаивающее в пуле дольше этого времени, может быть удалено. Значение `0` отключает idle-экспирацию. **Требует** установки AppContext switch `Switch.Microsoft.Data.SqlClient.UseLegacyIdleTimeoutBehavior` в `false` (по умолчанию `true`, что означает legacy-поведение). 

### `Pool Blocking Period` (по умолчанию `Auto`)

Управляет поведением после ошибки аутентификации. При неудачной аутентификации пул входит в **период блокировки**: все последующие попытки открыть соединение с теми же параметрами **немедленно** выбрасывают закэшированное исключение, не пытаясь аутентифицироваться снова. Первый период — 5 секунд, затем удваивается до максимума в 1 минуту. 

- `Auto`: блокировка включена для обычных SQL Server, отключена для распознанных Azure SQL endpoints
- `AlwaysBlock`: блокировка для всех
- `NeverBlock`: отключена (может привести к «шторму аутентификации» при проблемах с кредами или сетью)


## 4. Пул и асинхронность: что происходит с `OpenAsync`

`OpenAsync` ведёт себя с пулом точно так же, как `Open`, но **не блокирует поток** во время ожидания. Если в пуле есть свободное соединение, `OpenAsync` завершается **синхронно** (Task уже completed). Если соединений нет, `Task` будет ждать освобождения асинхронно, освобождая поток для другой работы. 

**Важный нюанс из GitHub issue:** в некоторых версиях `Microsoft.Data.SqlClient` (7.0.0-preview2) при работе с LocalDB, находящимся в состоянии `Stopped`, `OpenAsync` может «зависнуть» на полный `Connect Timeout` и выбросить **вводящее в заблуждение** сообщение о таймауте пула, хотя на самом деле проблема в том, что LocalDB не запустился. Это подчёркивает важность проверки реальной причины: если пул пуст, `Max Pool Size` не достигнут, а `OpenAsync` висит — проблема может быть не в пуле. 


## 5. Примеры для запуска: база данных

### 5.1. Скрипт создания БД

```sql
-- ============================================================
-- БД для примеров с пулом соединений
-- ============================================================
IF DB_ID('PoolTestDb') IS NOT NULL
BEGIN
    ALTER DATABASE PoolTestDb SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
    DROP DATABASE PoolTestDb;
END
GO
CREATE DATABASE PoolTestDb;
GO
USE PoolTestDb;
GO

-- Простая таблица для имитации работы
CREATE TABLE dbo.Items
(
    Id    INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    Name  NVARCHAR(100) NOT NULL,
    Value INT NOT NULL
);
GO

-- Заполняем данными
;WITH N AS (SELECT TOP 100 ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS n FROM sys.all_objects)
INSERT INTO dbo.Items (Name, Value)
SELECT CONCAT('Item-', n), n * 10 FROM N;
GO

-- Хранимая процедура, имитирующая долгую операцию
CREATE OR ALTER PROCEDURE dbo.usp_Wait
    @Seconds INT = 5
AS
BEGIN
    SET NOCOUNT ON;
    WAITFOR DELAY @Seconds;
    SELECT 'Done' AS Result;
END
GO

-- Проверка
SELECT COUNT(*) AS ItemsCount FROM dbo.Items; -- 100
GO
```

### 5.2. Строки подключения для примеров

```csharp
public static class PoolDb
{
    // Обычная строка (пул по умолчанию: Min=0, Max=100, Connect Timeout=15)
    public const string Normal =
        @"Server=(localdb)\MSSQLLocalDB;Database=PoolTestDb;Integrated Security=true;TrustServerCertificate=true;";

    // Ограниченный пул: максимум 2 соединения, таймаут 2 секунды
    public const string Limited =
        @"Server=(localdb)\MSSQLLocalDB;Database=PoolTestDb;Integrated Security=true;TrustServerCertificate=true;" +
        "Max Pool Size=2;Connect Timeout=2;";

    // Без пула (для сравнения производительности)
    public const string NoPool =
        @"Server=(localdb)\MSSQLLocalDB;Database=PoolTestDb;Integrated Security=true;TrustServerCertificate=true;Pooling=false;";

    // С «тёплым» пулом: Min Pool Size=5
    public const string Warm =
        @"Server=(localdb)\MSSQLLocalDB;Database=PoolTestDb;Integrated Security=true;TrustServerCertificate=true;Min Pool Size=5;";

    // Для проверки фрагментации: Application Name уникален
    public static string WithAppName(string appName) =>
        $@"Server=(localdb)\MSSQLLocalDB;Database=PoolTestDb;Integrated Security=true;TrustServerCertificate=true;Application Name={appName};";
}
```


## 6. Примеры работы пула (каждый можно запустить)

### Пример 1. Базовое переиспользование: один и тот же SPID

```csharp
using System.Data;
using Microsoft.Data.SqlClient;

public static async Task Example1_PoolReuseAsync()
{
    Console.WriteLine("=== Пример 1: Переиспользование пула (SPID) ===");

    var spids = new List<int>();

    for (int i = 0; i < 5; i++)
    {
        // await using гарантирует возврат соединения в пул даже при исключении
        await using var connection = new SqlConnection(PoolDb.Normal);
        await connection.OpenAsync();

        await using var cmd = new SqlCommand("SELECT @@SPID", connection);
        int spid = Convert.ToInt32(await cmd.ExecuteScalarAsync());
        spids.Add(spid);

        Console.WriteLine($"  Итерация {i + 1}: SPID = {spid}");
        // На выходе из using соединение возвращается в пул
    }

    Console.WriteLine($"  Уникальных SPID: {spids.Distinct().Count()}");
    Console.WriteLine(spids.Distinct().Count() == 1
        ? "  ✔ Пул переиспользовал одно физическое соединение."
        : "  ⚠ Пул создал несколько соединений (возможно, из-за параллелизма или первого запуска).");
}
```

**Ожидаемый результат:** все 5 итераций возвращают **один и тот же SPID**. Это доказывает, что пул вернул то же физическое соединение. 


### Пример 2. Что происходит при исчерпании пула

```csharp
public static async Task Example2_PoolExhaustionAsync()
{
    Console.WriteLine("\n=== Пример 2: Исчерпание пула (Max Pool Size=2) ===");

    // Открываем два соединения и НЕ закрываем их — пул исчерпан
    await using var conn1 = new SqlConnection(PoolDb.Limited);
    await using var conn2 = new SqlConnection(PoolDb.Limited);
    await conn1.OpenAsync();
    await conn2.OpenAsync();
    Console.WriteLine("  Открыто 2 соединения (Max Pool Size=2). Пул исчерпан.");

    // Пытаемся открыть третье — должно упасть через Connect Timeout=2
    var sw = System.Diagnostics.Stopwatch.StartNew();
    await using var conn3 = new SqlConnection(PoolDb.Limited);
    try
    {
        await conn3.OpenAsync();
        Console.WriteLine("  Неожиданно: третье соединение открылось.");
    }
    catch (InvalidOperationException ex)
    {
        sw.Stop();
        Console.WriteLine($"  ✔ Ожидаемое исключение через ~{sw.ElapsedMilliseconds} мс:");
        Console.WriteLine($"    {ex.Message.Split('\n')[0]}");
    }
}
```

**Ожидаемый результат:** третье `OpenAsync` ждёт ~2 секунды (Connect Timeout=2) и выбрасывает `InvalidOperationException` с текстом про «pool... max pool size was reached». **Важно:** это `InvalidOperationException`, а не `SqlException`. 


### Пример 3. Утечка соединения: почему `Close()` недостаточно

```csharp
public static async Task Example3_ConnectionLeakAsync()
{
    Console.WriteLine("\n=== Пример 3: Утечка соединения ===");

    // Правильно: await using возвращает соединение в пул
    Console.WriteLine("  Правильный паттерн:");
    for (int i = 0; i < 3; i++)
    {
        await using var conn = new SqlConnection(PoolDb.Limited);
        await conn.OpenAsync();
        await using var cmd = new SqlCommand("SELECT @@SPID", conn);
        Console.WriteLine($"    SPID = {await cmd.ExecuteScalarAsync()}");
    }

    // Неправильно: соединение не освобождается
    Console.WriteLine("  Неправильный паттерн (утечка):");
    var leaked = new List<SqlConnection>();
    try
    {
        for (int i = 0; i < 3; i++)
        {
            var conn = new SqlConnection(PoolDb.Limited);
            await conn.OpenAsync();
            leaked.Add(conn); // не закрываем!
            Console.WriteLine($"    SPID = {await new SqlCommand("SELECT @@SPID", conn).ExecuteScalarAsync()}");
        }
    }
    catch (InvalidOperationException ex)
    {
        Console.WriteLine($"    ✔ Пул исчерпан через {leaked.Count} соединения: {ex.Message.Split('\n')[0]}");
    }
    finally
    {
        // Освобождаем «утекшие» соединения
        foreach (var c in leaked) await c.DisposeAsync();
        Console.WriteLine("    Утекшие соединения освобождены.");
    }
}
```

**Ключевой вывод:** `Close()` и `Dispose()` **оба** возвращают соединение в пул. Но если вы не вызываете ни то, ни другое (или полагаетесь на GC), соединение остаётся занятым до финализации. В .NET `SqlConnection` реализует `IDisposable`, и `await using` — правильный паттерн. 


### Пример 4. Фрагментация пула из-за разных строк подключения

```csharp
public static async Task Example4_PoolFragmentationAsync()
{
    Console.WriteLine("\n=== Пример 4: Фрагментация пула ===");

    // Одна и та же БД, но разные строки → разные пулы
    var strings = new[]
    {
        PoolDb.WithAppName("App-A"),
        PoolDb.WithAppName("App-B"),
        PoolDb.WithAppName("App-A"), // повторно использует пул App-A
    };

    var spids = new List<int>();
    foreach (var cs in strings)
    {
        await using var conn = new SqlConnection(cs);
        await conn.OpenAsync();
        await using var cmd = new SqlCommand("SELECT @@SPID", conn);
        spids.Add(Convert.ToInt32(await cmd.ExecuteScalarAsync()));
    }

    Console.WriteLine($"  SPID'ы: [{string.Join(", ", spids)}]");
    // App-A (первый) и App-A (третий) должны иметь одинаковый SPID
    // App-B — другой SPID, потому что это другой пул
    if (spids[0] == spids[2] && spids[0] != spids[1])
        Console.WriteLine("  ✔ Подтверждено: разные Application Name = разные пулы.");
    else
        Console.WriteLine("  ⚠ Неожиданный результат (возможно, пул ещё не прогрелся).");
}
```


### Пример 5. `Min Pool Size` и «тёплые» соединения

```csharp
public static async Task Example5_MinPoolSizeAsync()
{
    Console.WriteLine("\n=== Пример 5: Min Pool Size=5 ===");

    // Открываем 5 соединений параллельно, чтобы прогреть пул
    var tasks = Enumerable.Range(0, 5).Select(async _ =>
    {
        await using var conn = new SqlConnection(PoolDb.Warm);
        await conn.OpenAsync();
        await using var cmd = new SqlCommand("SELECT @@SPID", conn);
        return Convert.ToInt32(await cmd.ExecuteScalarAsync());
    });
    var spids = await Task.WhenAll(tasks);
    Console.WriteLine($"  Прогрев: {spids.Distinct().Count()} уникальных SPID (Min=5).");

    // После закрытия всех соединений пул должен сохранить минимум 5
    // Проверяем: открываем 5 новых — они должны быть быстрыми и переиспользовать SPID'ы
    var sw = System.Diagnostics.Stopwatch.StartNew();
    var tasks2 = Enumerable.Range(0, 5).Select(async _ =>
    {
        await using var conn = new SqlConnection(PoolDb.Warm);
        await conn.OpenAsync();
        await using var cmd = new SqlCommand("SELECT @@SPID", conn);
        return Convert.ToInt32(await cmd.ExecuteScalarAsync());
    });
    var spids2 = await Task.WhenAll(tasks2);
    sw.Stop();
    Console.WriteLine($"  Повторное открытие 5: {sw.ElapsedMilliseconds} мс, SPID'ы: [{string.Join(", ", spids2.Distinct())}]");
    Console.WriteLine("  (С Min Pool Size соединения должны переиспользоваться без задержек.)");
}
```


### Пример 6. `Connect Timeout` — это таймаут пула, не TCP

```csharp
public static async Task Example6_ConnectTimeoutAsync()
{
    Console.WriteLine("\n=== Пример 6: Connect Timeout — таймаут пула ===");

    // Занимаем весь пул
    await using var conn1 = new SqlConnection(PoolDb.Limited);
    await conn1.OpenAsync();

    await using var conn2 = new SqlConnection(PoolDb.Limited);
    await conn2.OpenAsync();

    // Третье соединение ждёт 2 секунды (Connect Timeout=2 в PoolDb.Limited)
    var sw = System.Diagnostics.Stopwatch.StartNew();
    try
    {
        await using var conn3 = new SqlConnection(PoolDb.Limited);
        await conn3.OpenAsync();
    }
    catch (InvalidOperationException)
    {
        sw.Stop();
        Console.WriteLine($"  ✔ Таймаут пула сработал через {sw.ElapsedMilliseconds} мс (Connect Timeout=2).");
        Console.WriteLine("    Это НЕ таймаут TCP — это ожидание свободного соединения в пуле.");
    }
}
```


### Пример 7. `Pooling=false` vs `Pooling=true`: разница в производительности

```csharp
public static async Task Example7_PoolingPerformanceAsync()
{
    Console.WriteLine("\n=== Пример 7: Pooling=false vs Pooling=true ===");

    async Task<long> MeasureAsync(string cs, int iterations)
    {
        var sw = System.Diagnostics.Stopwatch.StartNew();
        for (int i = 0; i < iterations; i++)
        {
            await using var conn = new SqlConnection(cs);
            await conn.OpenAsync();
            await using var cmd = new SqlCommand("SELECT 1", conn);
            await cmd.ExecuteScalarAsync();
        }
        sw.Stop();
        return sw.ElapsedMilliseconds;
    }

    const int N = 20;
    long noPool = await MeasureAsync(PoolDb.NoPool, N);
    long withPool = await MeasureAsync(PoolDb.Normal, N);

    Console.WriteLine($"  Pooling=false: {noPool} мс для {N} открытий");
    Console.WriteLine($"  Pooling=true:  {withPool} мс для {N} открытий");
    Console.WriteLine($"  ✔ Пул быстрее в {noPool / Math.Max(withPool, 1)} раз(а).");
}
```

**Ожидаемый результат:** `Pooling=true` будет в 10–100 раз быстрее, потому что каждое `Open` с `Pooling=false` включает полное TCP- и TLS-рукопожатие, аутентификацию и т.д. 


### Пример 8. Асинхронное ожидание пула

```csharp
public static async Task Example8_AsyncPoolWaitAsync()
{
    Console.WriteLine("\n=== Пример 8: OpenAsync и асинхронное ожидание пула ===");

    // Занимаем пул
    await using var conn1 = new SqlConnection(PoolDb.Limited);
    await conn1.OpenAsync();
    await using var conn2 = new SqlConnection(PoolDb.Limited);
    await conn2.OpenAsync();
    Console.WriteLine("  Пул исчерпан (2/2). Запускаем OpenAsync для третьего...");

    // Запускаем третье открытие асинхронно
    var openTask = Task.Run(async () =>
    {
        await using var conn3 = new SqlConnection(PoolDb.Limited);
        try
        {
            await conn3.OpenAsync();
            return "Успех";
        }
        catch (InvalidOperationException)
        {
            return "Таймаут пула";
        }
    });

    // Пока третье ждёт, освобождаем одно соединение через 500 мс
    await Task.Delay(500);
    Console.WriteLine("  Освобождаем conn2...");
    await conn2.DisposeAsync();

    // Третье должно успеть получить освободившееся соединение
    var result = await openTask;
    Console.WriteLine($"  Результат третьего: {result}");
    Console.WriteLine("  (Если бы освободили позже 2 сек — был бы таймаут.)");
}
```


## 7. Тренировочные задания (10 штук, с каркасом, без кода)

Ниже — задания для самостоятельной отработки. Для каждого дан **каркас** (сигнатура метода и пустая реализация) и **критерии проверки**. Код нужно написать самостоятельно.

---

### Задание 1. Проверить, что пул переиспользует соединение

**Каркас:**
```csharp
public static async Task<bool> IsSamePhysicalConnectionAsync(string cs, int attempts)
{
    // TODO: открыть attempts соединений последовательно,
    // собрать @@SPID, вернуть true, если все SPID одинаковы.
    throw new NotImplementedException();
}
```

**Критерии:** метод возвращает `true` для `PoolDb.Normal` и `attempts=5`. Если вернуть `false` — пул не работает или строка подключения меняется.

---

### Задание 2. Найти количество уникальных физических соединений при параллельной нагрузке

**Каркас:**
```csharp
public static async Task<int> CountUniqueSpidsAsync(string cs, int parallelCount)
{
    // TODO: запустить parallelCount задач Task.WhenAll,
    // каждая открывает соединение, получает @@SPID, закрывает.
    // Вернуть количество уникальных SPID.
    throw new NotImplementedException();
}
```

**Критерии:** для `PoolDb.Normal` и `parallelCount=20` вернуть значение **от 1 до 20** (зависит от того, насколько быстро задачи освобождают соединения). Для `PoolDb.Limited` (Max=2) — **не более 2**.

---

### Задание 3. Воспроизвести исчерпание пула и вернуть текст исключения

**Каркас:**
```csharp
public static async Task<string> ExhaustPoolAndGetMessageAsync(string limitedCs)
{
    // TODO: открыть Max Pool Size соединений и НЕ закрывать их.
    // Попытаться открыть ещё одно. Перехватить исключение.
    // Вернуть первые 80 символов сообщения исключения.
    throw new NotImplementedException();
}
```

**Критерии:** для `PoolDb.Limited` вернуть строку, содержащую `"Timeout expired"` и `"pool"`. Время выполнения — около 2 секунд (Connect Timeout=2).

---

### Задание 4. Доказать, что `Close()` возвращает соединение в пул

**Каркас:**
```csharp
public static async Task<bool> CloseReturnsToPoolAsync(string cs)
{
    // TODO: открыть соединение, получить SPID, вызвать Close() (не Dispose).
    // Открыть новое соединение, получить SPID. Вернуть true, если SPID совпадают.
    throw new NotImplementedException();
}
```

**Критерии:** возвращает `true`. Демонстрирует, что `Close()` достаточно для возврата в пул (но в production используйте `using`/`await using`).

---

### Задание 5. Показать фрагментацию пула

**Каркас:**
```csharp
public static async Task<(bool Fragmented, string Details)> DemonstrateFragmentationAsync()
{
    // TODO: открыть соединение со строкой A (Application Name=App1),
    // затем со строкой B (Application Name=App2),
    // затем снова со строкой A.
    // Сравнить SPID первого и третьего.
    // Вернуть (true, ...) если SPID[0]==SPID[2] и SPID[0]!=SPID[1].
    throw new NotImplementedException();
}
```

**Критерии:** возвращает `Fragmented=true`, потому что `App1` и `App2` — разные пулы, но повторное использование `App1` даёт тот же SPID.

---

### Задание 6. Измерить ускорение от пула

**Каркас:**
```csharp
public static async Task<(long NoPoolMs, long PoolMs, double Speedup)> MeasurePoolSpeedupAsync(
    string noPoolCs, string poolCs, int iterations)
{
    // TODO: замерить время iterations открытий/закрытий с Pooling=false и Pooling=true.
    // Вернуть кортеж с временами и коэффициентом ускорения.
    throw new NotImplementedException();
}
```

**Критерии:** `Speedup` > 5 для `iterations=20` на LocalDB. Если < 5 — что-то не так (возможно, кэш ОС сглаживает разницу).

---

### Задание 7. Показать, что `Min Pool Size` сохраняет соединения

**Каркас:**
```csharp
public static async Task<bool> MinPoolKeepsConnectionsAsync(string warmCs)
{
    // TODO: открыть 5 соединений параллельно, закрыть все.
    // Подождать 3 секунды.
    // Открыть 5 новых соединений и проверить, что все SPID из первого набора
    // (или хотя бы большинство) переиспользованы.
    throw new NotImplementedException();
}
```

**Критерии:** возвращает `true` для `PoolDb.Warm` (Min=5). С `Min=0` (PoolDb.Normal) соединения могли бы быть закрыты при простое, и SPID могли бы измениться.

---

### Задание 8. `Connect Timeout` и его отличие от `CommandTimeout`

**Каркас:**
```csharp
public static async Task<(string ExceptionType, long ElapsedMs)> DistinguishTimeoutsAsync(
    string limitedCs, int commandTimeoutSeconds)
{
    // TODO:
    // 1. Заполнить пул (открыть Max Pool Size соединений).
    // 2. Попытаться открыть ещё одно → замерить время и тип исключения (Connect Timeout).
    // 3. Освободить пул.
    // 4. Выполнить dbo.usp_Wait с CommandTimeout=commandTimeoutSeconds → замерить тип и время.
    // Вернуть тип исключения для шага 2 и время шага 2.
    throw new NotImplementedException();
}
```

**Критерии:** на шаге 2 — `InvalidOperationException` через ~2 сек. На шаге 4 — `SqlException` с `Number == -2` через `commandTimeoutSeconds`.

---

### Задание 9. Асинхронное ожидание освобождения пула

**Каркас:**
```csharp
public static async Task<bool> AsyncWaitForPoolSlotAsync(string limitedCs)
{
    // TODO:
    // 1. Заполнить пул (Max Pool Size соединений).
    // 2. Запустить Task, который вызывает OpenAsync (будет ждать).
    // 3. Через 300 мс освободить одно соединение.
    // 4. Дождаться Task и проверить, что OpenAsync завершился успешно.
    throw new NotImplementedException();
}
```

**Критерии:** возвращает `true` (третье соединение дождалось освобождения и открылось). Время выполнения — около 300–500 мс, а не 2 секунды (Connect Timeout).

---

### Задание 10. Полный сценарий: пул + отмена + очистка

**Каркас:**
```csharp
public static async Task<(bool Cancelled, int OpenConnections)> PoolWithCancellationAsync(
    string limitedCs, CancellationToken userToken)
{
    // TODO:
    // 1. Заполнить пул до Max Pool Size.
    // 2. Запустить OpenAsync с userToken для ещё одного соединения.
    // 3. Через 200 мс отменить userToken.
    // 4. Перехватить OperationCanceledException.
    // 5. Освободить все соединения.
    // 6. Вернуть (true, количество соединений, которые были открыты до отмены).
    throw new NotImplementedException();
}
```

**Критерии:** возвращает `Cancelled=true` и `OpenConnections=2` (для `PoolDb.Limited` с Max=2). Демонстрирует, что `CancellationToken` прерывает ожидание пула **раньше** `Connect Timeout`.


## 8. Чеклист по пулу для production

| Ситуация | Правило |
|---|---|
| **Освобождение соединений** | Всегда `await using` для `SqlConnection`, `SqlDataReader`, `SqlCommand`, `SqlTransaction` |
| **Строка подключения** | Одна каноническая строка. Никаких уникальных `Application Name` на запрос |
| **`Max Pool Size`** | Не увеличивать, пока не проверены все пути освобождения соединений |
| **`Connect Timeout`** | Это таймаут пула, не TCP. Увеличивать только если измерено, что пул перегружен |
| **`Min Pool Size`** | Только если измерения подтверждают необходимость «тёплых» соединений. Не использовать в serverless |
| **Аутентификация** | `Pool Blocking Period=Auto`. Не отключать без необходимости |
| **Диагностика** | Если `OpenAsync` висит, а пул пуст — проверьте, не LocalDB ли в `Stopped` (известный баг)  |
| **Утечки** | `Close()` и `Dispose()` возвращают в пул. GC — нет. Не полагайтесь на финализацию |


## 9. Итог

Пул соединений — это клиентский кэш физических соединений. Он создаётся для каждой уникальной конфигурации. `Open()` берёт соединение из пула, `Close()`/`Dispose()` возвращает. Если пул исчерпан, `Open()` ждёт до `Connect Timeout`, затем выбрасывает `InvalidOperationException`. Асинхронность (`OpenAsync`) позволяет ждать, не блокируя поток. Главные источники проблем: утечки соединений (забыли `using`), фрагментация пулов (разные строки подключения), и непонимание того, что `Connect Timeout` — это таймаут пула, а не сети.

Все примеры из раздела 6 можно запустить в одном консольном проекте .NET 8 с пакетом `Microsoft.Data.SqlClient`. База данных создаётся скриптом из раздела 5.1. Тренировочные задания из раздела 7 позволяют проверить понимание каждой концепции на практике.
