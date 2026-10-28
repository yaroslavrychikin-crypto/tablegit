## Часть 0. База данных для задач

Сохраните как `CreateProcTxDB.sql` и выполните в SSMS.

```sql
-- ============================================================
-- Создание базы
-- ============================================================
IF DB_ID('ProcTxDemoDB') IS NULL
    CREATE DATABASE ProcTxDemoDB;
GO

USE ProcTxDemoDB;
GO

-- ============================================================
-- Таблица Customers — клиенты
-- ============================================================
IF OBJECT_ID('dbo.Customers','U') IS NULL
BEGIN
    CREATE TABLE dbo.Customers
    (
        Id        INT IDENTITY(1,1) PRIMARY KEY,
        FullName  NVARCHAR(100) NOT NULL,
        Email     NVARCHAR(200) NULL,
        Balance   DECIMAL(18,2) NOT NULL DEFAULT 0,
        IsActive  BIT NOT NULL DEFAULT 1,
        CreatedAt DATETIME2(0) NOT NULL DEFAULT SYSUTCDATETIME()
    );
END
GO

-- ============================================================
-- Таблица Products — товары
-- ============================================================
IF OBJECT_ID('dbo.Products','U') IS NULL
BEGIN
    CREATE TABLE dbo.Products
    (
        Id       INT IDENTITY(1,1) PRIMARY KEY,
        Name     NVARCHAR(100) NOT NULL,
        Price    DECIMAL(18,2) NOT NULL,
        Stock    INT NOT NULL DEFAULT 0,
        IsActive BIT NOT NULL DEFAULT 1
    );
END
GO

-- ============================================================
-- Таблица Orders — заказы
-- ============================================================
IF OBJECT_ID('dbo.Orders','U') IS NULL
BEGIN
    CREATE TABLE dbo.Orders
    (
        Id          INT IDENTITY(1,1) PRIMARY KEY,
        CustomerId  INT NOT NULL REFERENCES dbo.Customers(Id),
        ProductId   INT NOT NULL REFERENCES dbo.Products(Id),
        Quantity    INT NOT NULL,
        TotalAmount DECIMAL(18,2) NOT NULL,
        CreatedAt   DATETIME2(0) NOT NULL DEFAULT SYSUTCDATETIME()
    );
END
GO

-- ============================================================
-- Таблица AuditLog — журнал операций
-- ============================================================
IF OBJECT_ID('dbo.AuditLog','U') IS NULL
BEGIN
    CREATE TABLE dbo.AuditLog
    (
        Id        INT IDENTITY(1,1) PRIMARY KEY,
        EventType NVARCHAR(50) NOT NULL,
        Message   NVARCHAR(500) NOT NULL,
        LoggedAt  DATETIME2(0) NOT NULL DEFAULT SYSUTCDATETIME()
    );
END
GO

-- ============================================================
-- Тестовые данные
-- ============================================================
IF NOT EXISTS (SELECT 1 FROM dbo.Customers)
BEGIN
    INSERT INTO dbo.Customers (FullName, Email, Balance) VALUES
        (N'Иван Иванов',    N'ivan@example.com',   5000.00),
        (N'Мария Петрова',  N'maria@example.com',  1200.50),
        (N'Пётр Сидоров',   NULL,                  0.00),
        (N'Ольга Смирнова', N'olga@example.com',   300.00),
        (N'Сергей Волков',  N'sergey@example.com', 10000.00);
END
GO

IF NOT EXISTS (SELECT 1 FROM dbo.Products)
BEGIN
    INSERT INTO dbo.Products (Name, Price, Stock) VALUES
        (N'Книга',       500.00,  10),
        (N'Ручка',       50.00,   100),
        (N'Ноутбук',     50000.00, 3),
        (N'Монитор',     15000.00, 5),
        (N'Клавиатура',  2500.00,  7);
END
GO

-- ============================================================
-- Готовые процедуры (для тренировочных задач)
-- ============================================================

-- usp_GetCustomersByMinBalance — простой SELECT
CREATE OR ALTER PROCEDURE dbo.usp_GetCustomersByMinBalance
    @MinBalance DECIMAL(18,2)
AS
BEGIN
    SET NOCOUNT ON;
    SELECT Id, FullName, Email, Balance
    FROM dbo.Customers
    WHERE Balance >= @MinBalance
    ORDER BY Balance DESC;
END
GO

-- usp_DepositWithLog — процедура с DML + INSERT в лог
CREATE OR ALTER PROCEDURE dbo.usp_DepositWithLog
    @CustomerId INT,
    @Amount     DECIMAL(18,2),
    @NewBalance DECIMAL(18,2) OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    IF @Amount <= 0
        THROW 50001, N'Сумма должна быть положительной', 1;

    UPDATE dbo.Customers
    SET Balance = Balance + @Amount
    WHERE Id = @CustomerId;

    IF @@ROWCOUNT = 0
        THROW 50002, N'Клиент не найден', 1;

    INSERT INTO dbo.AuditLog (EventType, Message)
    VALUES (N'Deposit', CONCAT(N'CustomerId=', @CustomerId, N', +', @Amount));

    SELECT @NewBalance = Balance FROM dbo.Customers WHERE Id = @CustomerId;
END
GO

PRINT 'База ProcTxDemoDB готова.';
PRINT 'Таблицы: Customers, Products, Orders, AuditLog';
PRINT 'Процедуры: usp_GetCustomersByMinBalance, usp_DepositWithLog';
```

---

## Часть 1. Пять тренировочных задач с полным разбором

### Тренировочная задача 1. Минимальная транзакция

**Цель:** понять, как выглядит базовая транзакция `BeginTransaction` → `Commit`.

**Задание:** перевести 100 с баланса клиента 1 клиенту 2, используя транзакцию.

---

### Тренировочная задача 2. Откат при ошибке

**Цель:** увидеть, что при исключении изменения не применяются.

**Задание:** выполнить ту же операцию, но во втором UPDATE указать несуществующий Id. Убедиться, что баланс первого клиента тоже не изменился.

---

### Тренировочная задача 3. `using` и `try/catch` — правильный порядок

**Цель:** понять, в каком порядке освобождать ресурсы.

**Задание:** переписать задачу 2 так, чтобы `SqlTransaction` освобождался через `using`, а `Commit`/`Rollback` вызывались корректно.


---

### Тренировочная задача 4. `@` — команда без транзакции падает

**Цель:** убедиться, что связь команды с транзакцией обязательна.

**Задание:** попробовать выполнить `SqlCommand` с `Connection`, у которого открыта транзакция, но без указания `Transaction` в команде. Поймать `InvalidOperationException`.

---

### Тренировочная задача 5. `ExecuteNonQuery` внутри транзакции

**Цель:** увидеть, что результат `ExecuteNonQuery` отражает изменения, но они ещё не зафиксированы.

**Задание:** внутри транзакции обновить трёх клиентов, напечатать сумму `affected`. До `Commit` отдельным соединением проверить, видны ли изменения (не видны — изоляция по умолчанию `READ COMMITTED` блокирует чтение).


---

## Часть 2. Задачи по темам (6–30)

### Тема A. Простые хранимые процедуры (задачи 6–10)

#### Задача 6 ⭐. Вызов готовой процедуры `usp_GetCustomersByMinBalance`

**Задание:** вызвать процедуру с `@MinBalance = 1000`, вывести `Id`, `FullName`, `Balance`. Затем с `@MinBalance = 999999` — убедиться, что `HasRows = false`.

**Ключ:** `CommandType.StoredProcedure`, `cmd.CommandText = "dbo.usp_GetCustomersByMinBalance"` (без `EXEC`), `ExecuteReader`.

---

#### Задача 7 ⭐. Процедура со скалярным результатом

**Задание:** создать процедуру `usp_GetTotalStock`, возвращающую одно число через `SELECT SUM(Stock) FROM Products`. Вызвать её через `ExecuteScalar`. Обработать случай `NULL` (если таблица пуста — но она не пуста; для проверки — временно удалите все товары, вызовите, верните).

**Ключ:** процедура с `SELECT SUM(...)` — возвращает одну ячейку; `ExecuteScalar` вернёт `DBNull.Value` при отсутствии строк (после `ISNULL` — `0`).

---

#### Задача 8 ⭐. Процедура с одним входным параметром

**Задание:** создать `usp_GetProductsCheaperThan(@MaxPrice DECIMAL(18,2))`, возвращающую товары дешевле указанной цены, с колонками `Id, Name, Price, Stock`. Вызвать для `@MaxPrice = 3000`.

**Ключ:** `CREATE OR ALTER PROCEDURE` + `SqlParameter` с `SqlDbType.Decimal` (задать `Precision = 18, Scale = 2`).

---

#### Задача 9 ⭐⭐. Два входных параметра

**Задание:** создать `usp_GetOrdersByCustomer(@CustomerId INT, @MinAmount DECIMAL(18,2))`, возвращающую заказы клиента с суммой не меньше указанной. Вызвать для клиента 1 с минимумом 100.

**Ключ:** два `SqlParameter`; после вызова вывести `FieldCount` и прочитать все строки.

---

#### Задача 10 ⭐⭐. Процедура с OUTPUT

**Задание:** создать `usp_GetCustomerBalance(@CustomerId INT, @Balance DECIMAL(18,2) OUTPUT)`. Если клиента нет — через `THROW` бросить ошибку `50010`. Вызвать для существующего и несуществующего `Id`.

**Ключ:** `@Balance ... OUTPUT`, `SqlParameter.Direction = ParameterDirection.Output`, `THROW 50010, N'...', 1`.

---

### Тема B. Процедуры с OUTPUT и ReturnValue (задачи 11–15)

#### Задача 11 ⭐⭐. Output + ReturnValue (успех)

**Задание:** создать `usp_CreateOrder(@CustomerId INT, @ProductId INT, @Quantity INT, @NewOrderId INT OUTPUT)`, которая:
- проверяет достаточность `Stock` у товара,
- вычитает `Quantity` из `Stock`,
- вставляет заказ в `Orders` с `TotalAmount = Price * Quantity`,
- возвращает `@NewOrderId`,
- делает `RETURN 0` при успехе.

Вызвать для клиента 1 и товара 2 (Ручка) с количеством 3. Проверить, что `Stock` уменьшился, а заказ создан.

**Ключ:** `RETURN 0`, `SET @NewOrderId = SCOPE_IDENTITY()`, `SqlParameter` с `Direction = Output` и `Direction = ReturnValue`.

---

#### Задача 12 ⭐⭐. Output + ReturnValue (бизнес-ошибка)

**Задание:** вызвать `usp_CreateOrder` с `Quantity = 999` для товара с `Stock = 10`. Процедура должна вернуть `RETURN 1` (нехватка), `@NewOrderId = NULL`. Проверить, что `Stock` не изменился.

**Ключ:** в процедуре `IF @Stock < @Quantity BEGIN SET @NewOrderId = NULL; RETURN 1; END`. На клиенте проверять `ret.Value` и `@NewOrderId == DBNull`.

---

#### Задача 13 ⭐⭐⭐. THROW внутри процедуры

**Задание:** создать `usp_Withdraw(@CustomerId INT, @Amount DECIMAL(18,2), @NewBalance DECIMAL(18,2) OUTPUT)`, которая:
- если `@Amount <= 0` → `THROW 50020, 'Amount must be positive', 1`,
- если баланс меньше суммы → `THROW 50021, 'Insufficient funds', 1`,
- иначе списывает и возвращает `@NewBalance`.

Вызвать три раза: с корректной суммой, с отрицательной, с суммой больше баланса. Ловить `SqlException` и различать по `ex.Number`.

**Ключ:** `catch (SqlException ex) when (ex.Number == 50020)`.

---

#### Задача 14 ⭐⭐⭐. Использование ReturnValue для классификации

**Задание:** изменить `usp_CreateOrder` так, чтобы коды возврата были:
- `0` — успех,
- `1` — товар не найден,
- `2` — нехватка на складе,
- `3` — клиент не найден.

Вызвать четыре раза (успех + три ошибки) и по `ret.Value` вывести человекочитаемое сообщение.

**Ключ:** `RETURN 1/2/3` вместо `THROW` — клиент не ловит исключение, а читает код.

---

#### Задача 15 ⭐⭐⭐. Процедура с двумя OUTPUT

**Задание:** создать `usp_GetStats(@CustomerId INT, @OrderCount INT OUTPUT, @TotalSpent DECIMAL(18,2) OUTPUT)`, которая считает количество и сумму всех заказов клиента. Вызвать для клиента с заказами и для клиента без заказов. Убедиться, что `@OrderCount = 0`, `@TotalSpent = 0` (не `NULL`).

**Ключ:** `ISNULL` в `SELECT @TotalSpent = ISNULL(SUM(TotalAmount), 0)`.

---

### Тема C. Транзакции — базовые (задачи 16–20)

#### Задача 16 ⭐⭐. Простейшая транзакция (перевод)

**Задание:** реализовать перевод 200 с клиента 1 клиенту 3 в одной транзакции. Вывести балансы до и после. Убедиться, что оба `UPDATE` применились.

**Ключ:** два `SqlCommand` с одинаковым `tx`; `Commit` в конце.

---

#### Задача 17 ⭐⭐. Откат по проверке `affected == 0`

**Задание:** то же, но во втором `UPDATE` использовать несуществующего клиента (`Id = 9999`). Вручную проверить `affected == 0` и выполнить `Rollback`. Вывести балансы до и после — должны совпасть.

**Ключ:** `if (affected == 0) { tx.Rollback(); return; }`.

---

#### Задача 18 ⭐⭐. Откат по `SqlException`

**Задание:** в транзакции вставить в `Customers` клиента с `FullName = NULL` (провоцирует `NOT NULL` violation). Поймать `SqlException`, выполнить `Rollback`. Убедиться, что `COUNT(*)` не изменился.

**Ключ:** `catch (SqlException ex) { tx.Rollback(); Console.WriteLine(ex.Number); }` — номер 515 (Cannot insert the value NULL).

---

#### Задача 19 ⭐⭐⭐. Транзакция с процедурой

**Задание:** вызвать `usp_DepositWithLog` (уже создана) **внутри** транзакции. Затем намеренно выполнить `THROW` на клиенте через отдельную `SqlCommand` с `RAISERROR` и откатить. Проверить, что ни баланс, ни `AuditLog` не изменились.

**Ключ:** процедура, вызванная внутри транзакции, участвует в ней. Если на клиенте откатить транзакцию — откатятся и её изменения.

---

#### Задача 20 ⭐⭐⭐. Условный `Commit`/`Rollback` по бизнес-правилу

**Задание:** реализовать «перевод только если после списания баланс отправителя не станет отрицательным». Внутри транзакции сначала прочитать баланс через `SELECT ... WITH (UPDLOCK)` (не углубляясь в изоляцию — просто чтобы прочитать актуальное значение), проверить, затем выполнить оба `UPDATE` и `Commit`. Если баланс недостаточен — `Rollback` без изменений.

**Ключ:** `if (balance < amount) { tx.Rollback(); return; }` до любых `UPDATE`.

---

### Тема D. Комбинированные задачи (задачи 21–25)

#### Задача 21 ⭐⭐⭐. Создание заказа: процедура + транзакция на клиенте

**Задание:** реализовать на клиенте (без процедуры) сценарий:
1. Открыть транзакцию.
2. Прочитать `Stock` товара и `Balance` клиента.
3. Проверить достаточность.
4. Списать `Stock`.
5. Списать `Balance` клиента.
6. Вставить `Order`.
7. `Commit`.
Любая ошибка — `Rollback`. Протестировать на успех и на «нехватку баланса».

**Ключ:** все команды — с `tx`; проверки — через `ExecuteScalar` и `if (...) { tx.Rollback(); return; }`.

---

#### Задача 22 ⭐⭐⭐. Процедура + OUTPUT + транзакция

**Задание:** создать `usp_RefundOrder(@OrderId INT, @Refunded DECIMAL(18,2) OUTPUT)`, которая:
- находит заказ,
- возвращает `Quantity` на `Stock` товара,
- возвращает `TotalAmount` на `Balance` клиента,
- возвращает сумму в `@Refunded`,
- делает `RETURN 0` при успехе, `RETURN 1` если заказ не найден.

Вызвать её **внутри** клиентской транзакции. Если `RETURN != 0` — `Rollback`. Если успех — дополнительно записать в `AuditLog` и `Commit`. Протестировать на существующем и несуществующем `OrderId`.

**Ключ:** процедура и `INSERT INTO AuditLog` — обе команды с одним `tx`.

---

#### Задача 23 ⭐⭐⭐⭐. Пакетная обработка в транзакции

**Задание:** для массива `CustomerId = {1,2,3}` начислить по 50 бонусов (прибавить к `Balance`). Всё — в одной транзакции. После цикла — `Commit`. Если на каком-то шаге `affected == 0` — `Rollback` всей пачки. Проверить, что при успехе все три клиента получили по 50.

**Ключ:** цикл внутри `try`, счётчик `affected`, `Commit` после цикла.

---

#### Задача 24 ⭐⭐⭐⭐. Процедура-«агрегатор» с транзакцией внутри

**Задание:** создать `usp_TransferFunds(@FromId INT, @ToId INT, @Amount DECIMAL(18,2))`, которая:
- начинает **свою** транзакцию через `BEGIN TRAN` в T-SQL,
- проверяет баланс отправителя (при недостатке — `ROLLBACK; THROW 50030`),
- выполняет два `UPDATE`,
- пишет в `AuditLog`,
- делает `COMMIT`.

Вызвать с клиента 5 (10000) на клиента 3 (0) сумму 500. Проверить балансы. Вызвать с клиента 3 (500) на клиента 1 сумму 99999 — поймать `SqlException` с `Number == 50030`, убедиться, что балансы не изменились.

**Ключ:** транзакция **внутри процедуры** — `BEGIN TRAN ... COMMIT / ROLLBACK`. На клиенте транзакцию не открывать.

---

#### Задача 25 ⭐⭐⭐⭐. Процедура + клиентская транзакция + OUTPUT

**Задание:** создать `usp_ApplyDiscount(@CustomerId INT, @Percent DECIMAL(5,2), @OldBalance DECIMAL(18,2) OUTPUT, @NewBalance DECIMAL(18,2) OUTPUT)`, которая уменьшает баланс на процент и возвращает старое и новое значения. Вызвать **внутри** клиентской транзакции. Если `@Percent` вне диапазона `0..100` — процедура делает `THROW 50040`. Клиент по `catch` делает `Rollback`. Успешный вызов — `Commit`, дополнительно `INSERT INTO AuditLog` в той же транзакции.

**Ключ:** два `OUTPUT`-параметра + `SqlParameter.Direction = Output`; клиентская транзакция охватывает и вызов процедуры, и запись в `AuditLog`.

---

### Тема E. Финальные комплексные задачи (задачи 26–30)

#### Задача 26 ⭐⭐⭐⭐. Оформление заказа: процедура + проверки + ReturnValue

**Задание:** создать `usp_PlaceOrder(@CustomerId INT, @ProductId INT, @Quantity INT, @OrderId INT OUTPUT)`, которая:
- проверяет клиента (`RETURN 1` если нет),
- проверяет товар (`RETURN 2` если нет),
- проверяет `Stock` (`RETURN 3` если мало),
- проверяет `Balance` клиента (`RETURN 4` если мало),
- при успехе: списывает `Stock`, списывает `Balance`, вставляет `Order`, возвращает `@OrderId`, `RETURN 0`.

Обернуть на клиенте в транзакцию, чтобы при `RETURN != 0` ничего не применилось (даже если процедура частично что-то сделала — на практике она ничего не сделает при возврате до `INSERT`, но пусть клиент тоже подстрахуется `Rollback`).

Протестировать все пять исходов.

**Ключ:** проверки через `IF NOT EXISTS (...) RETURN 1;` и т.д. На клиенте — `tx.Commit()` только если `ret.Value == 0`.

---

#### Задача 27 ⭐⭐⭐⭐. Процедура «перевод с комиссией»

**Задание:** создать `usp_TransferWithFee(@FromId INT, @ToId INT, @Amount DECIMAL(18,2), @Fee DECIMAL(18,2) OUTPUT)`, где комиссия = 1% от суммы (минимум 10). Процедура внутри своей транзакции (`BEGIN TRAN`):
- проверяет, что у отправителя хватает `Amount + Fee`,
- списывает `Amount + Fee`,
- начисляет получателю `Amount`,
- пишет в `AuditLog` две записи (списание + зачисление),
- `COMMIT`.

При нехватке — `ROLLBACK; THROW 50050`. Вызвать на успех и на провал, проверить балансы и `AuditLog`.

**Ключ:** `SET @Fee = CASE WHEN @Amount * 0.01 < 10 THEN 10 ELSE @Amount * 0.01 END;`.

---

#### Задача 28 ⭐⭐⭐⭐⭐. Массовое списание с транзакцией

**Задание:** списать с каждого клиента с `Balance > 1000` по 100 в одной транзакции. Если клиентов больше 3 — `Commit`, иначе `Rollback` (искусственное правило). Протестировать на текущих данных (клиентов с балансом > 1000 — четверо? проверьте, подстройте правило).

**Ключ:** `SELECT Id FROM Customers WHERE Balance > 1000`; цикл `UPDATE`; `if (count > 3) Commit else Rollback`.

---

#### Задача 29 ⭐⭐⭐⭐⭐. Процедура с OUTPUT + Rollback внутри

**Задание:** создать `usp_WithdrawWithAudit(@CustomerId INT, @Amount DECIMAL(18,2), @NewBalance DECIMAL(18,2) OUTPUT)`, которая:
- `BEGIN TRAN`,
- списывает `Amount`,
- пишет в `AuditLog`,
- если новый баланс стал `< 0` — `ROLLBACK; THROW 50060`,
- иначе `COMMIT` и возвращает `@NewBalance`.

Вызвать дважды: с суммой меньше баланса и с суммой больше. Во втором случае — `SqlException` 50060. Убедиться, что ни `Balance`, ни `AuditLog` не изменились после второго вызова (для отката — `SELECT` до и после).

**Ключ:** откат внутри процедуры откатывает и `UPDATE`, и `INSERT` в лог.

---

#### Задача 30 ⭐⭐⭐⭐⭐. Полный сценарий: заказ → оплата → отмена

**Задание:** реализовать три процедуры и прогнать полный цикл на клиенте в трёх транзакциях:

1. `usp_CreateOrderDraft(@CustomerId, @ProductId, @Quantity, @OrderId OUTPUT) RETURN 0/1..4` — как в задаче 26, но `Stock` **не** списывает (только резервирует через отдельную таблицу `Reserved INT` в `Products`; добавьте столбец).
2. `usp_PayOrder(@OrderId, @NewBalance OUTPUT) RETURN 0/1` — проверяет `Balance` клиента, списывает `TotalAmount`, помечает заказ как оплаченный (добавьте `IsPaid BIT` в `Orders`).
3. `usp_CancelOrder(@OrderId) RETURN 0/1` — снимает резерв со `Stock`, возвращает `Balance` клиенту.

Сценарий A (успех):
- Открыть транзакцию 1 → `usp_CreateOrderDraft` → `Commit`.
- Открыть транзакцию 2 → `usp_PayOrder` → `Commit`.
- Проверить `Stock`, `Reserved`, `Balance`, `IsPaid`.

Сценарий B (отмена после создания):
- `usp_CreateOrderDraft` → `Commit`.
- `usp_CancelOrder` → `Commit`.
- Проверить, что `Reserved` вернулся, `Balance` не менялся.

Сценарий C (провал оплаты):
- `usp_CreateOrderDraft` с количеством, при котором `TotalAmount > Balance` → `RETURN 4`, `Rollback`.
- Убедиться, что `Stock`, `Reserved` не изменились.

**Ключ:** три отдельные процедуры; каждая — со своей логикой `RETURN`; на клиенте каждая — своя транзакция с `Commit`/`Rollback` по коду возврата. Внутри процедур — обычный DML без собственных транзакций (транзакцию держит клиент).

**Подготовка БД для задачи 30:**

```sql
-- Выполнить один раз перед задачей 30
ALTER TABLE dbo.Products ADD Reserved INT NOT NULL DEFAULT 0;
ALTER TABLE dbo.Orders ADD IsPaid BIT NOT NULL DEFAULT 0;
GO
```
