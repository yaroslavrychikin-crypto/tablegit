# 25 задач по асинхронному ADO.NET на базе AsyncAdoTestDb

Строка подключения для всех задач:
```csharp
private const string ConnectionString =
    @"Server=(localdb)\MSSQLLocalDB;Database=AsyncAdoTestDb;Integrated Security=true;TrustServerCertificate=true;";
```

---

## ТЕМА 1. Базовые асинхронные операции (задачи 1–4)

**Цель:** освоить `OpenAsync`, `ExecuteReaderAsync`, `ExecuteScalarAsync`, `ExecuteNonQueryAsync` и `await using`.

---

### Задача 1. Открыть соединение асинхронно и получить версию сервера

**Задание:** Напишите метод `Task<string> GetServerVersionAsync()`, который открывает соединение через `OpenAsync`, выполняет `SELECT @@VERSION` через `ExecuteScalarAsync` и возвращает строку.

**Проверяется:**
- `await using` для `SqlConnection` и `SqlCommand`
- `OpenAsync` без `CancellationToken` (пока без отмены)

**Ожидаемый результат:** строка вида `Microsoft SQL Server 2022 ...`.

**Ловушка:** если используете `using` (синхронный), соединение освободится синхронно — в async-коде это антипаттерн, хоть и скомпилируется.

---

### Задача 2. Посчитать количество строк в таблице

**Задание:** Метод `Task<int> GetPersonCountAsync()` возвращает `COUNT(*)` из `Person.Person`.

**Проверяется:**
- `ExecuteScalarAsync` возвращает `object?` — нужно привести к `int` через `Convert.ToInt32` или `(int)`.
- `await using` для `SqlCommand`.

**Ожидаемый результат:** `50` (или больше, если вы запускали тесты с INSERT).

**Ловушка:** `ExecuteScalarAsync` возвращает `null`, если запрос вернул пустой результат. Для `COUNT(*)` — никогда, но привычка проверять `is int` полезна.

---

### Задача 3. Прочитать 5 персон в список объектов

**Задание:** Метод `Task<List<Person>> GetTop5PeopleAsync()` возвращает первые 5 записей из `Person.Person` (по `BusinessEntityID`).

**Проверяется:**
- `ExecuteReaderAsync` + `ReadAsync` в цикле `while`
- Синхронные геттеры (`GetInt32`, `GetString`) после `ReadAsync` — это **правильно** для маленьких полей.

**Ожидаемый результат:** список из 5 `Person`.

**Ловушка:** не забывайте `await using` для `SqlDataReader` — это `IAsyncDisposable`.

---

### Задача 4. Обновить LastName по Id и вернуть число затронутых строк

**Задание:** Метод `Task<int> RenamePersonAsync(int id, string newLastName)` обновляет `LastName` и возвращает результат `ExecuteNonQueryAsync`.

**Проверяется:**
- Параметризация (`@Id`, `@LastName`) — **обязательна**, никакой конкатенации.
- `SqlDbType` и `Size` для `NVarChar`.

**Ожидаемый результат:** `1` для существующего `Id`, `0` для несуществующего.

**Ловушка:** если не указать `Size` для `NVarChar`, ADO.NET возьмёт размер по значению, что может привести к фрагментации плана запроса.

---

## ТЕМА 2. CancellationToken и таймауты (задачи 5–8)

**Цель:** научиться связывать пользовательскую отмену с внутренним таймаутом и различать `OperationCanceledException` от `SqlException`.

---

### Задача 5. Добавить CancellationToken во все методы темы 1

**Задание:** Перепишите методы из задач 1–4, добавив параметр `CancellationToken token` во все вызовы (`OpenAsync`, `ExecuteReaderAsync`, `ReadAsync`, `ExecuteScalarAsync`, `ExecuteNonQueryAsync`).

**Проверяется:**
- Токен передаётся во **все** асинхронные вызовы, а не только в `OpenAsync`.

**Ожидаемый результат:** код компилируется, поведение не меняется, но теперь можно отменить операцию.

**Ловушка:** если забыть передать токен в `ReadAsync`, отмена во время чтения строк не сработает.

---

### Задача 6. Отменить долгий запрос через 2 секунды

**Задание:** Используйте `dbo.usp_SlowQuery` (из предыдущего скрипта) с `@DelaySeconds = 30`. Создайте `CancellationTokenSource`, вызовите `CancelAfter(TimeSpan.FromSeconds(2))`, перехватите `OperationCanceledException` и выведите `"Отменено за 2 сек"`.

**Проверяется:**
- Понимание, что `CancellationToken` **прерывает** ожидание, а не просто проверяется между операциями.

**Ожидаемый результат:** сообщение через ~2 секунды, **не** 30.

**Ловушка:** `TaskCanceledException` наследуется от `OperationCanceledException` — ловите базовый тип.

---

### Задача 7. CommandTimeout = 2 секунды без токена

**Задание:** Тот же `usp_SlowQuery`, но **без** `CancellationToken`, с `CommandTimeout = 2`. Перехватите `SqlException` с `ex.Number == -2`.

**Проверяется:**
- Разница между `CommandTimeout` (→ `SqlException`, Number = -2) и `CancellationToken` (→ `OperationCanceledException`).

**Ожидаемый результат:** `SqlException` через ~2 секунды.

**Ловушка:** `Number == -2` — это **единственный** надёжный способ отличить таймаут от других `SqlException`. Не ориентируйтесь на текст сообщения (он локализуется).

---

### Задача 8. Связывание токенов: пользовательская отмена + внутренний таймаут

**Задание:** Напишите метод `Task<List<Person>> GetPeopleWithTimeoutAsync(CancellationToken userToken, int timeoutSeconds)`. Внутри создайте `CancellationTokenSource.CreateLinkedTokenSource(userToken)`, вызовите `CancelAfter(timeoutSeconds)`, передайте `cts.Token` во все вызовы. Верните людей.

**Проверяется:**
- `CreateLinkedTokenSource` — правильный способ комбинировать отмены.

**Ожидаемый результат:** при `userToken` = отмена или таймаут сработает — `OperationCanceledException`.

**Ловушка:** не забывайте `using` для `cts` — иначе утечка таймера.

---

## ТЕМА 3. Транзакции и обработка ошибок (задачи 9–12)

**Цель:** освоить `BeginTransactionAsync`, `CommitAsync`, `RollbackAsync` и корректную обработку ошибок.

---

### Задача 9. Простая транзакция с commit

**Задание:** Вставьте две персоны в одной транзакции, закоммитьте, верните количество вставленных строк (должно быть 2).

**Проверяется:**
- `BeginTransactionAsync` + `CommitAsync`
- `SqlCommand(..., transaction)` — передача транзакции в команду.

**Ожидаемый результат:** `2`.

**Ловушка:** если не передать `transaction` в `SqlCommand`, команда выполнится вне транзакции (или выбросит `InvalidOperationException` при `Enlist`).

---

### Задача 10. Транзакция с откатом при ошибке

**Задание:** Вставьте одну персону, затем — намеренно `NULL` в `FirstName` (нарушение `NOT NULL`). Перехватите `SqlException`, выполните `RollbackAsync`, верните `false`.

**Проверяется:**
- `RollbackAsync` в `catch`
- Проверка, что первая вставка не сохранилась (через `COUNT(*)` до/после).

**Ожидаемый результат:** `false`, количество строк в таблице не изменилось.

**Ловушка:** `RollbackAsync` должен использовать **чистый** токен (`CancellationToken.None`), а не отменённый.

---

### Задача 11. Откат при отмене транзакции

**Задание:** Начните транзакцию, вставьте строку, подождите `Task.Delay(2000, cancelToken)`. Через 1 секунду отмените `cancelToken`. В `catch (OperationCanceledException)` выполните `RollbackAsync(CancellationToken.None)`.

**Проверяется:**
- Правило «cleanup-токен»: откат не должен использовать отменённый токен.

**Ожидаемый результат:** `OperationCanceledException` через ~1 секунду, откат успешен, строка не вставлена.

**Ловушка:** если передать `cancelToken` в `RollbackAsync`, получите `OperationCanceledException` **внутри** `catch`, и исходная транзакция останется открытой.

---

### Задача 12. AggregateException при двойной ошибке

**Задание:** В `catch` после ошибки операции выполните `RollbackAsync` с **заведомо отменённым** токеном (создайте `CancellationTokenSource`, вызовите `Cancel()`). Перехватите `OperationCanceledException` от отката и оберните оба исключения в `AggregateException`.

**Проверяется:**
- Понимание, что откат тоже может упасть, и это нужно обрабатывать отдельно.

**Ожидаемый результат:** `AggregateException` с двумя внутренними.

**Ловушка:** `AggregateException` должен содержать **оба** исключения: исходное и отката.

---

## ТЕМА 4. MARS и множественные reader'ы (задачи 13–15)

**Цель:** понять, когда MARS нужен, а когда вреден.

---

### Задача 13. Воспроизвести ошибку без MARS

**Задание:** Откройте соединение **без** `MultipleActiveResultSets=True`. Выполните `ExecuteReaderAsync` для `Person.Person`, **не закрывая** reader, попробуйте выполнить `ExecuteNonQueryAsync` для `UPDATE Person.Person SET LastName = LastName`. Перехватите `InvalidOperationException`.

**Проверяется:**
- Понимание ограничения «один открытый reader на соединение».

**Ожидаемый результат:** `InvalidOperationException` с сообщением про «already an open DataReader».

**Ловушка:** `await using` для первого reader'а не поможет — reader **открыт** до конца блока, а второй запрос выполняется внутри этого блока.

---

### Задача 14. Тот же сценарий с MARS

**Задание:** Повторите задачу 13, но добавьте `MultipleActiveResultSets=True` в строку подключения. Убедитесь, что второй запрос выполняется.

**Проверяется:**
- Включение MARS через строку подключения.

**Ожидаемый результат:** оба запроса выполняются, ошибки нет.

**Ловушка:** не забывайте, что соединение с MARS и без — **разные пулы**. Не смешивайте их в одном приложении.

---

### Задача 15. MARS: два reader'а одновременно

**Задание:** С MARS откройте два reader'а: один на `Person.Person`, второй на `HumanResources.Employee`. Прочитайте оба **до конца**, чередуя `ReadAsync` (например, читайте по одной строке из каждого в цикле, пока оба не закончатся).

**Проверяется:**
- Понимание, что MARS позволяет **чередовать** чтение.

**Ожидаемый результат:** вывод обеих таблиц вперемешку.

**Ловушка:** MARS **не выполняет запросы параллельно на сервере**. Если один `SELECT` содержит `WAITFOR`, второй reader заблокируется.

---

## ТЕМА 5. Потоковое чтение больших данных (задачи 16–18)

**Цель:** освоить `SequentialAccess`, `GetStream`, `GetFieldValueAsync` для BLOB.

---

### Задача 16. Прочитать 1 МБ BLOB в `byte[]`

**Задание:** Метод `Task<byte[]> GetDocumentDataAsync(int id)` возвращает `LargeData` для документа с данным `Id`.

**Проверяется:**
- `CommandBehavior.SequentialAccess`
- `GetFieldValueAsync<byte[]>` для полей `VARBINARY(MAX)`

**Ожидаемый результат:** массив длиной `1_048_576` для `id=2`.

**Ловушка:** для `VARBINARY(MAX)` без `SequentialAccess` драйвер загрузит **всю строку** в буфер. Для 10 МБ это заметно по памяти.

---

### Задача 17. Прочитать 10 МБ BLOB **без** загрузки в память

**Задание:** Метод `Task<long> GetDocumentSizeAsync(int id)` возвращает размер BLOB, но **не** создаёт массив `byte[]`. Используйте `GetStream` и `stream.CopyToAsync(Stream.Null)`.

**Проверяется:**
- `GetStream` + `CopyToAsync` для потоковой обработки.

**Ожидаемый результат:** `10_485_760` без выделения 10 МБ памяти.

**Ловушка:** `stream.Length` не всегда работает для потокового BLOB (может выбросить `NotSupportedException`). Считайте через `ReadAsync` в буфер или через `CopyToAsync`.

---

### Задача 18. Потоковая запись BLOB из файла

**Задание:** Метод `Task InsertDocumentFromFileAsync(string filePath)` вставляет файл как `VARBINARY(MAX)` в `dbo.Documents`. Используйте `File.OpenRead` + параметр с `SqlDbType.VarBinary` и `Size = -1` (MAX).

**Проверяется:**
- `SqlDbType.VarBinary` с `Size = -1` — правильный способ передать `VARBINARY(MAX)`.

**Ожидаемый результат:** файл сохранён в БД, размер в БД = размеру файла.

**Ловушка:** если указать `Size = 8000` (или не указать вовсе), драйвер обрежет данные или выбросит `SqlException` о truncation.

---

## ТЕМА 6. SqlBulkCopy и пул соединений (задачи 19–22)

**Цель:** освоить массовую загрузку и понять поведение пула.

---

### Задача 19. Загрузить 1000 строк через SqlBulkCopy

**Задание:** Создайте `DataTable` с колонками `Name`, `Amount`, `CreatedAt` (1000 строк), загрузите в `dbo.MyTargetTable` через `WriteToServerAsync`.

**Проверяется:**
- `SqlBulkCopy` + `ColumnMappings`
- `WriteToServerAsync` с `CancellationToken`

**Ожидаемый результат:** 1000 строк в `MyTargetTable`.

**Ловушка:** если не задать `ColumnMappings`, `SqlBulkCopy` маппит колонки **по позиции**, что может привести к неверным данным.

---

### Задача 20. Отменить SqlBulkCopy на середине

**Задание:** Загрузите 50 000 строк, но отмените через `CancelAfter(50ms)`. Перехватите `OperationCanceledException`. Проверьте `COUNT(*)` в `MyTargetTable` — должно быть **меньше 50 000** (частичная загрузка).

**Проверяется:**
- `WriteToServerAsync` уважает `CancellationToken`.

**Ожидаемый результат:** `OperationCanceledException`, частичные данные в таблице.

**Ловушка:** `SqlBulkCopy` **не атомарен** по умолчанию. Если нужна атомарность — оберните в транзакцию (`bulkCopy.ExternalTransaction` или `SqlTransaction`).

---

### Задача 21. Проверить переиспользование пула

**Задание:** Откройте и закройте 5 соединений подряд. В каждом выполните `SELECT @@SPID`. Соберите `SPID` в список. Убедитесь, что все 5 `SPID` одинаковы.

**Проверяется:**
- Понимание, что пул возвращает то же физическое соединение.

**Ожидаемый результат:** `spids.Distinct().Count() == 1`.

**Ловушка:** при **первом** запуске приложения пул пуст, и первое соединение создаётся физически. Это нормально — оно и будет переиспользовано.

---

### Задача 22. Воспроизвести исчерпание пула

**Задание:** Используйте строку подключения с `Max Pool Size=1;Connect Timeout=2`. Откройте первое соединение и **не закрывайте**. Попробуйте открыть второе — перехватите `SqlException` с сообщением о таймауте пула.

**Проверяется:**
- Понимание, что `Connect Timeout` — это таймаут **пула**, а не TCP.

**Ожидаемый результат:** `SqlException` через ~2 секунды.

**Ловушка:** `Connect Timeout` — это **не** `CommandTimeout` и **не** `CancellationToken`. Это отдельный механизм.

---

## ТЕМА 7. Комплексные сценарии (задачи 23–25)

**Цель:** собрать всё вместе.

---

### Задача 23. Репозиторий с полным async/await и ConfigureAwait

**Задание:** Напишите класс `AsyncPersonRepository` с методами:
- `Task<List<Person>> GetAllAsync(CancellationToken)`
- `Task<Person?> GetByIdAsync(int id, CancellationToken)`
- `Task<int> InsertAsync(Person p, CancellationToken)`
- `Task<bool> UpdateAsync(Person p, CancellationToken)`
- `Task<bool> DeleteAsync(int id, CancellationToken)`

Во всех методах используйте `await using`, `ConfigureAwait(false)`, параметризацию, `CancellationToken`.

**Проверяется:**
- Всё сразу: `await using`, `ConfigureAwait(false)`, параметры, токены.

**Ожидаемый результат:** все CRUD-операции работают.

**Ловушка:** `ConfigureAwait(false)` — **только** в библиотечном коде. В UI-коде (WPF/WinForms) — не используйте, иначе обновление UI упадёт.

---

### Задача 24. Пакетная обработка с ограничением параллелизма

**Задание:** Прочитайте **все** `BusinessEntityID` из `Person.Person`. Для каждого выполните `GetByIdAsync` (из задачи 23) **параллельно**, но не более **5 одновременно** (используйте `SemaphoreSlim(5)`). Соберите результаты в список.

**Проверяется:**
- `SemaphoreSlim` + `Task.WhenAll` для контроля параллелизма.
- Понимание, что `Task.WhenAll` без ограничения может **исчерпать пул** соединений.

**Ожидаемый результат:** все персоны получены, одновременно открыто не более 5 соединений.

**Ловушка:** не используйте `Parallel.ForEachAsync` с `MaxDegreeOfParallelism` в старых версиях .NET — только `SemaphoreSlim` переносим.

---

### Задача 25. Сквозной сценарий: импорт CSV в БД с транзакцией и отменой

**Задание:** Метод `Task<int> ImportPersonsFromCsvAsync(string csvPath, CancellationToken userToken)`:
1. Читает CSV-файл (формат: `FirstName,LastName` на каждой строке).
2. Открывает соединение и транзакцию.
3. Вставляет каждую строку параметризованным `INSERT`.
4. Каждые 100 строк — `CommitAsync` (частями, чтобы не держать одну огромную транзакцию).
5. При `OperationCanceledException` — `RollbackAsync(CancellationToken.None)` текущей части.
6. Возвращает общее количество вставленных строк.

**Проверяется:**
- Всё сразу: файлы, транзакции, отмена, cleanup-токен, параметризация.

**Ожидаемый результат:** для файла на 1000 строк — `1000`.

**Ловушка:** при отмене между `CommitAsync` и началом следующей транзакции нужно **не забыть** закрыть текущую транзакцию откатом, иначе следующая часть начнётся с «грязной» транзакции.

---

## Сводная таблица тем и задач

| Тема | Задачи | Ключевые концепции |
|---|---|---|
| 1. Базовые операции | 1–4 | `OpenAsync`, `ExecuteReaderAsync`, `ExecuteScalarAsync`, `ExecuteNonQueryAsync`, `await using` |
| 2. Отмена и таймауты | 5–8 | `CancellationToken`, `CommandTimeout`, `CreateLinkedTokenSource` |
| 3. Транзакции | 9–12 | `BeginTransactionAsync`, `CommitAsync`, `RollbackAsync`, cleanup-токен, `AggregateException` |
| 4. MARS | 13–15 | `MultipleActiveResultSets`, ограничения, чередование reader'ов |
| 5. Потоковые BLOB | 16–18 | `SequentialAccess`, `GetStream`, `SqlDbType.VarBinary` + `Size=-1` |
| 6. BulkCopy и пул | 19–22 | `SqlBulkCopy`, `ColumnMappings`, `Max Pool Size`, `Connect Timeout` |
| 7. Комплексные | 23–25 | Репозиторий, `SemaphoreSlim`, пакетная обработка, сквозной CSV-импорт |

