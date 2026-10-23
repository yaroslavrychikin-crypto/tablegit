
## Часть 0. Подготовка в Visual Studio

### 0.1. Подключение к серверу через обозреватель объектов SQL Server

1. Меню **Вид → Обозреватель объектов SQL Server** (View → SQL Server Object Explorer).
2. Правой кнопкой по **SQL Server** → **Добавить SQL Server**.
3. Введите имя сервера: `localhost` или `.\SQLEXPRESS` или `(localdb)\MSSQLLocalDB`.
4. Аутентификация: **Проверка подлинности Windows**.
5. **Подключить**.

### 0.2. Создание базы

1. Разверните подключённый сервер → правой кнопкой по **Базы данных** → **Добавить новую базу данных**.
2. Имя: `AdoNetPractice` → **ОК**.

### 0.3. Выполнение SQL-скрипта

1. Правой кнопкой по базе `AdoNetPractice` → **Создать запрос** (New Query).
2. Вставьте скрипт ниже → нажмите **Выполнить** (зелёная кнопка ▶ или Ctrl+Shift+E).

```sql
USE AdoNetPractice;
GO

-- Удаляем таблицы, если уже есть (чтобы можно было запускать повторно)
IF OBJECT_ID('Orders')     IS NOT NULL DROP TABLE Orders;
IF OBJECT_ID('Accounts')   IS NOT NULL DROP TABLE Accounts;
IF OBJECT_ID('Products')   IS NOT NULL DROP TABLE Products;
IF OBJECT_ID('Customers')  IS NOT NULL DROP TABLE Customers;
IF OBJECT_ID('Categories') IS NOT NULL DROP TABLE Categories;
GO

CREATE TABLE Categories (
    Id   INT IDENTITY(1,1) PRIMARY KEY,
    Name NVARCHAR(100) NOT NULL UNIQUE
);

CREATE TABLE Products (
    Id          INT IDENTITY(1,1) PRIMARY KEY,
    CategoryId  INT NOT NULL REFERENCES Categories(Id),
    Name        NVARCHAR(200) NOT NULL,
    Price       DECIMAL(18,2) NOT NULL CHECK (Price >= 0),
    Stock       INT NOT NULL DEFAULT 0,
    Description NVARCHAR(MAX) NULL,
    CreatedAt   DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);

CREATE TABLE Customers (
    Id       INT IDENTITY(1,1) PRIMARY KEY,
    FullName NVARCHAR(200) NOT NULL,
    Email    NVARCHAR(200) NULL,
    Bonus    DECIMAL(18,2) NOT NULL DEFAULT 0
);

CREATE TABLE Orders (
    Id         INT IDENTITY(1,1) PRIMARY KEY,
    CustomerId INT NOT NULL REFERENCES Customers(Id),
    ProductId  INT NOT NULL REFERENCES Products(Id),
    Quantity   INT NOT NULL CHECK (Quantity > 0),
    Total      DECIMAL(18,2) NOT NULL,
    Status     NVARCHAR(20) NOT NULL DEFAULT 'Pending',
    CreatedAt  DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);

CREATE TABLE Accounts (
    Id      INT IDENTITY(1,1) PRIMARY KEY,
    Owner   NVARCHAR(100) NOT NULL,
    Balance DECIMAL(18,2) NOT NULL CHECK (Balance >= 0)
);
GO

INSERT INTO Categories (Name) VALUES
    (N'Электроника'), (N'Книги'), (N'Одежда');

INSERT INTO Products (CategoryId, Name, Price, Stock, Description) VALUES
    (1, N'Смартфон',   29999.00, 50,  N'Флагман, 256 ГБ'),
    (1, N'Наушники',    4999.00, 120, NULL),
    (1, N'Планшет',    19999.00, 30,  N'10 дюймов'),
    (2, N'Роман',        599.00, 300, N'Классика'),
    (2, N'Учебник SQL', 1299.00, 80,  NULL),
    (3, N'Футболка',     999.00, 200, N'Хлопок 100%'),
    (3, N'Куртка',      5999.00, 40,  N'Зимняя');

INSERT INTO Customers (FullName, Email, Bonus) VALUES
    (N'Иван Петров',    N'ivan@example.com', 100.00),
    (N'Мария Сидорова', NULL,                 50.00),
    (N'Пётр Иванов',    N'petr@example.com',   0.00);

INSERT INTO Orders (CustomerId, ProductId, Quantity, Total, Status) VALUES
    (1, 1, 2, 59998.00, N'Completed'),
    (2, 4, 3,  1797.00, N'Pending'),
    (3, 5, 1,  1299.00, N'Completed'),
    (1, 6, 5,  4995.00, N'Pending');

INSERT INTO Accounts (Owner, Balance) VALUES
    (N'Иван', 10000.00), (N'Мария', 5000.00);
GO

-- Хранимые процедуры
CREATE OR ALTER PROCEDURE usp_GetProductsByCategory
    @CategoryId INT
AS
BEGIN
    SET NOCOUNT ON;
    SELECT Id, Name, Price, Stock
    FROM Products WHERE CategoryId = @CategoryId ORDER BY Name;
END;
GO

CREATE OR ALTER PROCEDURE usp_AddProduct
    @CategoryId INT, @Name NVARCHAR(200), @Price DECIMAL(18,2),
    @Stock INT, @NewId INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    INSERT INTO Products (CategoryId, Name, Price, Stock)
    VALUES (@CategoryId, @Name, @Price, @Stock);
    SET @NewId = SCOPE_IDENTITY();
END;
GO

CREATE OR ALTER PROCEDURE usp_GetCustomerStats
    @CustomerId INT, @OrdersCount INT OUTPUT, @TotalSpent DECIMAL(18,2) OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SELECT @OrdersCount = COUNT(*), @TotalSpent = ISNULL(SUM(Total), 0)
    FROM Orders WHERE CustomerId = @CustomerId;
END;
GO

CREATE OR ALTER PROCEDURE usp_DeleteProduct
    @ProductId INT
AS
BEGIN
    SET NOCOUNT ON;
    IF NOT EXISTS (SELECT 1 FROM Products WHERE Id = @ProductId) RETURN -1;
    IF EXISTS (SELECT 1 FROM Orders WHERE ProductId = @ProductId) RETURN -2;
    DELETE FROM Products WHERE Id = @ProductId;
    RETURN 0;
END;
GO
```
# 25 задач 

**Исходные таблицы:** `Categories`, `Products`, `Customers`, `Orders`, `Accounts`.
**Исходные процедуры:** `usp_GetProductsByCategory`, `usp_AddProduct`, `usp_GetCustomerStats`, `usp_DeleteProduct`.

---

## Connection

1. Подключитесь к базе и выведите `conn.State`, `conn.Database`, `conn.DataSource`, версию сервера, имя текущей базы и имя пользователя.
2. Вынесите строку подключения и таймаут команды в константы, примените `CommandTimeout` к команде и выполните `SELECT 1`.
3. Выполните `SELECT 1` внутри `using`-блока для соединения. Затем после выхода из блока выведите `conn.State` и объясните результат.
4. Откройте соединение, закройте его через `Close()`, затем откройте повторно и измерьте время обоих `Open()` с помощью `Stopwatch`. Объясните разницу через пул соединений.
5. Сформируйте две строки подключения с одинаковым набором ключевых слов, но разным порядком. Откройте по одному соединению с каждой и объясните, почему создаются два разных пула.

---

## ExecuteNonQuery

6. Добавьте клиента «Алексей Смирнов» с email `alex@example.com` и бонусом 0. Верните сгенерированный `Id` через `OUTPUT INSERTED.Id`.
7. Повысьте цену всех товаров категории «Электроника» на 10% и выведите число изменённых строк.
8. Удалите все заказы со статусом `Cancelled`. Если удалено 0 строк — выведите предупреждение, иначе — число удалённых.
9. Создайте временную таблицу `#Temp (Id INT)`, вставьте в неё одну строку. Выведите результат `ExecuteNonQuery()` для `CREATE` и для `INSERT`. Объясните разницу.
10. Добавьте нового клиента и сразу обновите его бонус до 100 в двух отдельных командах. Выведите результаты `ExecuteNonQuery()` обеих команд.

---

## ExecuteScalar

11. Посчитайте количество клиентов с `Email IS NULL`. Объясните, почему результат не равен `DBNull`.
12. Получите максимальную цену товаров категории «Одежда». Затем повторите запрос для несуществующей категории и корректно обработайте `DBNull`.
13. Вставьте товар «Тестовый» в категорию 1 с ценой 100 и остатком 0. Получите `Id` через `SCOPE_IDENTITY()`. Выведите тип полученного значения и объясните, почему нужно `Convert.ToInt32`, а не `(int)`.
14. Повторите задачу 13, но через `OUTPUT INSERTED.Id`. Сравните тип возвращаемого значения с предыдущей задачей.
15. Получите суммарную стоимость всех заказов клиента с `Id = 1`. Обработайте случай, когда у клиента нет заказов.

---

## ExecuteReader

16. Прочитайте все товары (`Id, Name, Price, Description`) и выведите их. Для NULL в `Description` выводите «—».
17. Одним вызовом `ExecuteReader` выполните два `SELECT`: список клиентов и список заказов. Прочитайте оба набора через `NextResult()`.
18. Выполните запрос товаров с `Price > 100000` (результат пустой). Проверьте `HasRows` и убедитесь, что цикл `Read()` не выполнялся.
19. Прочитайте `Id, Name, Price` из `Products`, но доступ к столбцам осуществляйте по именам через `GetOrdinal`. Ординалы получите один раз до цикла.
20. Прочитайте `Id` и `Description` из `Products` с `CommandBehavior.SequentialAccess`. Используйте `GetChars`, чтобы получить первые 10 символов `Description` без чтения всего поля.

---

## Параметры

21. Напишите метод поиска клиента по email. Сначала продемонстрируйте уязвимость конкатенации на входе `' OR '1'='1`, затем перепишите через параметр и покажите, что возвращается 0 строк.
22. Вставьте товар с `Description = NULL`. Используйте `DBNull.Value`, а не C# `null`. Затем прочитайте его и проверьте `IsDBNull`.
23. Вставьте товар с описанием длиной 5000 символов. Сначала попробуйте `AddWithValue` — получите ошибку. Затем исправьте через `SqlDbType.NVarChar, -1`.
24. Вставьте денежную сумму `123.45m` в `Orders.Total`, явно задав `Precision = 18` и `Scale = 2`. Прочитайте обратно и убедитесь, что значение не исказилось.

---

## Хранимые процедуры

25. Вызовите `usp_GetProductsByCategory` для категории 1 через `CommandType.StoredProcedure`, прочитайте результат через `DataReader`. Затем вызовите `usp_AddProduct` с `OUTPUT`-параметром и получите новый `Id`. Затем вызовите `usp_GetCustomerStats` для клиента 1 с двумя `OUTPUT`-параметрами. Наконец, вызовите `usp_DeleteProduct` для трёх разных `ProductId`: несуществующего, существующего с заказами и существующего без заказов — и разберите `ReturnValue` в каждом случае.

