
# Общая база данных `ShopDb`

Все три темы работают с одной БД (SQL Server / LocalDB). Схема и данные одинаковые — это позволяет сравнивать подходы.

## SQL-скрипт создания и наполнения

```sql
CREATE DATABASE ShopDb;
GO
USE ShopDb;
GO

CREATE TABLE Categories (
    Id   INT IDENTITY PRIMARY KEY,
    Name NVARCHAR(100) NOT NULL
);

CREATE TABLE Products (
    Id          INT IDENTITY PRIMARY KEY,
    Name        NVARCHAR(100) NOT NULL,
    Price       DECIMAL(18,2) NOT NULL,
    Stock       INT NOT NULL DEFAULT 0,
    CategoryId  INT NULL,
    Description NVARCHAR(500) NULL,
    CONSTRAINT FK_Products_Categories
        FOREIGN KEY (CategoryId) REFERENCES Categories(Id)
);

CREATE TABLE Orders (
    Id           INT IDENTITY PRIMARY KEY,
    CustomerName NVARCHAR(100) NOT NULL,
    OrderDate    DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    Total        DECIMAL(18,2) NOT NULL DEFAULT 0
);

CREATE TABLE OrderItems (
    Id        INT IDENTITY PRIMARY KEY,
    OrderId   INT NOT NULL,
    ProductId INT NOT NULL,
    Quantity  INT NOT NULL,
    UnitPrice DECIMAL(18,2) NOT NULL,
    CONSTRAINT FK_OrderItems_Orders
        FOREIGN KEY (OrderId) REFERENCES Orders(Id),
    CONSTRAINT FK_OrderItems_Products
        FOREIGN KEY (ProductId) REFERENCES Products(Id)
);

CREATE TABLE AuditLog (
    Id        INT IDENTITY PRIMARY KEY,
    Message   NVARCHAR(500) NOT NULL,
    CreatedAt DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);
GO

-- Наполнение
INSERT INTO Categories (Name) VALUES
    (N'Электроника'), (N'Книги'), (N'Одежда');

INSERT INTO Products (Name, Price, Stock, CategoryId) VALUES
    (N'Ноутбук',           1200.00, 10, 1),
    (N'Смартфон',           800.00, 25, 1),
    (N'Наушники',           150.00, 50, 1),
    (N'Роман "Война и мир"', 25.00,100, 2),
    (N'Учебник по C#',       45.00, 60, 2),
    (N'Футболка',            20.00,200, 3),
    (N'Джинсы',              60.00, 80, 3),
    (N'Клавиатура',          90.00, 40, 1),
    (N'Мышь',                35.00, 70, 1),
    (N'Свитер',              55.00, 30, 3);

INSERT INTO Orders (CustomerName, Total) VALUES
    (N'Иван',  1235.00),
    (N'Мария',  150.00);

INSERT INTO OrderItems (OrderId, ProductId, Quantity, UnitPrice) VALUES
    (1, 1, 1, 1200.00),
    (1, 9, 1,   35.00),
    (2, 5, 2,   45.00),
    (2, 6, 3,   20.00);
GO
```

**Строка подключения** (одинаковая для всех тем):

```json
{
  "ConnectionStrings": {
    "ShopDb": "Server=(localdb)\\MSSQLLocalDB;Database=ShopDb;Trusted_Connection=True;TrustServerCertificate=True;"
  }
}
```

## Общие C#-модели

```csharp
public class Category
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public List<Product> Products { get; set; } = new();
}

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public decimal Price { get; set; }
    public int Stock { get; set; }
    public int? CategoryId { get; set; }
    public string? Description { get; set; }
    public Category? Category { get; set; }
}

public class Order
{
    public int Id { get; set; }
    public string CustomerName { get; set; } = "";
    public DateTime OrderDate { get; set; }
    public decimal Total { get; set; }
    public List<OrderItem> Items { get; set; } = new();
}

public class OrderItem
{
    public int Id { get; set; }
    public int OrderId { get; set; }
    public int ProductId { get; set; }
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
}

public class AuditLog
{
    public int Id { get; set; }
    public string Message { get; set; } = "";
    public DateTime CreatedAt { get; set; }
}
```

---

# Тема 1. ADO.NET

**Что понадобится**
- Таблицы: `Products`, `Categories`, `Orders`, `OrderItems`, `Products.Stock`.
- Модели: `Product`, `Category`, `Order`, `OrderItem` (обычные POCO, без навигаций — заполняются вручную в `Read()`).
- NuGet: `Microsoft.Data.SqlClient`, `Microsoft.Extensions.Configuration.Json`.
- Строка подключения — из `appsettings.json`.

**Задача 1. Базовое подключение и чтение (Connected-модель)**  
Подключитесь к `ShopDb` и выведите все товары (`Id`, `Name`, `Price`). Используйте `SqlConnection`, `SqlCommand`, `SqlDataReader`.  
**Требования:** строка подключения из конфигурации; `using` для соединения и ридера.  
**Проверка:** в консоли отображаются все строки `Products`.

**Задача 2. Параметризованный запрос**  
Добавьте фильтр по `@MinPrice` и `@CategoryId`, передаваемый через `SqlParameter`.  
**Требования:** никакой конкатенации — только параметры.  
**Проверка:** при разных значениях возвращаются корректные подмножества; инъекция `' OR 1=1 --` не срабатывает.

**Задача 3. CRUD через `SqlCommand`**  
Реализуйте `AddProduct`, `UpdateProductPrice`, `DeleteProduct` через `ExecuteNonQuery()`.  
**Требования:** обработка `SqlException` (например, нарушение FK при удалении категории с товарами).  
**Проверка:** после add — строка появилась; update — цена изменилась; delete связанной записи обрабатывается без падения.

**Задача 4. `DataSet` + `SqlDataAdapter` (Disconnected)**  
Загрузите `Products` в `DataSet` через `SqlDataAdapter`, покажите в `DataGridView`, реализуйте «Обновить»/«Сохранить».  
**Требования:** `SqlCommandBuilder` для генерации команд обновления.  
**Проверка:** изменения в гриде сохраняются в БД.

**Задача 5. Транзакция**  
Оформите заказ: вставка в `Orders`, несколько `OrderItems` и `UPDATE Products SET Stock = Stock - @qty`. Всё — в `SqlTransaction`.  
**Требования:** при ошибке на любом шаге — `Rollback()`.  
**Проверка:** при искусственном исключении после вставки в `Orders` ни одна таблица не изменилась.

**Задача 6. Асинхронность**  
Перепишите задачу 2 на `OpenAsync`, `ExecuteReaderAsync`, `ReadAsync`.  
**Требования:** `CancellationToken`, `await using`.  
**Проверка:** отмена токена → `OperationCanceledException`, соединение освобождается.

**Задача 7. Пакетная вставка**  
Вставьте 10 000 товаров двумя способами: (а) 10 000 `ExecuteNonQuery`, (б) `SqlBulkCopy`. Замерьте время.  
**Требования:** настройте `BatchSize` и `ColumnMappings`.  
**Проверка:** `SqlBulkCopy` значительно быстрее; данные идентичны.

---

# Тема 2. Entity Framework Core

**Что понадобится**
- Та же БД `ShopDb`.  
- Модели из общего блока (с навигациями `Product.Category`, `Category.Products`, `Order.Items`).
- NuGet: `Microsoft.EntityFrameworkCore.SqlServer`, `Microsoft.EntityFrameworkCore.Design`.
- Два варианта начала работы:  
  1. **Database-first:** `dotnet ef dbcontext scaffold "…" Microsoft.EntityFrameworkCore.SqlServer -o Models` — сгенерирует контекст из готовой БД.  
  2. **Code-first:** использовать приведённый ниже `ShopDbContext`. Так как БД уже создана, **не вызывайте `EnsureCreated()`/`Migrate()`** — просто работайте с существующей схемой.

**`ShopDbContext`**

```csharp
public class ShopDbContext : DbContext
{
    public DbSet<Category> Categories => Set<Category>();
    public DbSet<Product>  Products   => Set<Product>();
    public DbSet<Order>    Orders     => Set<Order>();
    public DbSet<OrderItem> OrderItems => Set<OrderItem>();
    public DbSet<AuditLog> AuditLogs  => Set<AuditLog>();

    protected override void OnConfiguring(DbContextOptionsBuilder o)
    {
        if (!o.IsConfigured)
            o.UseSqlServer("Server=(localdb)\\MSSQLLocalDB;Database=ShopDb;Trusted_Connection=True;TrustServerCertificate=True;");
    }

    protected override void OnModelCreating(ModelBuilder mb)
    {
        mb.Entity<Product>()
          .HasOne(p => p.Category)
          .WithMany(c => c.Products)
          .HasForeignKey(p => p.CategoryId);
    }
}
```

**Задача 1. Настройка `DbContext` и первая выборка**  
Выведите все товары через `context.Products.ToList()`.  
**Требования:** строка подключения из конфигурации.  
**Проверка:** в консоли все строки `Products`.

**Задача 2. Базовый CRUD**  
`Add`, `Update`, `Delete`, `GetById` через `Add()/Update()/Remove()/SaveChanges()`.  
**Требования:** `GetById` возвращает `null` для несуществующего id; удаление несуществующего не падает.  
**Проверка:** изменения отражаются в БД.

**Задача 3. Простой LINQ**  
Товары с `Price > 100`, сортировка по `Name`, проекция `{ Name, Price }`.  
**Требования:** `Where` + `OrderBy` + `Select` в одном запросе.  
**Проверка:** только товары дороже 100, отсортированы, только два поля.

**Задача 4. «Один ко многим» + `Include`**  
Выведите товары вместе с названием категории.  
**Требования:** `Include(p => p.Category)`; связь настраивается в `OnModelCreating` (колонка `CategoryId` в БД уже есть — миграция не нужна).  
**Проверка:** `Category` не `null` у каждого товара.

**Задача 5. Агрегация**  
(1) всего товаров, (2) средняя цена, (3) количество по категориям.  
**Требования:** `Count()`, `Average()`, `GroupBy()` — без `ToList()` до агрегации.  
**Проверка:** значения соответствуют БД, по одной строке на категорию.

**Задача 6. Асинхронное взаимодействие**  
Перепишите задачи 2–3 через `ToListAsync`, `FirstOrDefaultAsync`, `FindAsync`, `SaveChangesAsync`.  
**Требования:** `Task`/`Task<T>`, `CancellationToken` во всех вызовах, `await` в `Main`.  
**Проверка:** данные сохраняются так же, как в синхронной версии; отмена → `OperationCanceledException`.

**Задача 7. Транзакция**  
Оформите заказ: `Orders` + `OrderItems` + `Products.Stock` в одной транзакции.  
**Требования:** `BeginTransactionAsync` → `CommitAsync`/`RollbackAsync`; `try/catch`.  
**Проверка:** при искусственной ошибке (например, `Stock < Quantity`) — ни одна таблица не изменилась.

---

# Тема 3. Dapper

**Что понадобится**
- Та же БД `ShopDb`.  
- Модели из общего блока (POCO достаточно; навигации для multi-mapping заполняются Dapper).  
- NuGet: `Dapper`, `Microsoft.Data.SqlClient`.  
- Хелпер соединения:

```csharp
public static class Db
{
    public const string ConnectionString =
        "Server=(localdb)\\MSSQLLocalDB;Database=ShopDb;Trusted_Connection=True;TrustServerCertificate=True;";

    public static SqlConnection Open()
    {
        var c = new SqlConnection(ConnectionString);
        c.Open();
        return c;
    }
}
```

**Задача 1. Базовый CRUD**  
`GetAll()`, `GetById(id)`, `Insert(Product)`, `Update(Product)`, `Delete(id)` на `Query<T>`, `QueryFirstOrDefault<T>`, `Execute`.  
**Требования:** параметризация; строка подключения из конфигурации.  
**Проверка:** все методы корректно меняют БД; `GetById` возвращает `null` для несуществующего id.

**Задача 2. Параметры и защита от инъекций**  
Фильтрация `GetAll` по `CategoryId` и диапазону цен через анонимный объект (`new { CategoryId = 1, MinPrice = 10, MaxPrice = 100 }`).  
**Требования:** без интерполяции SQL.  
**Проверка:** `' OR 1=1 --` возвращает пусто, а не все строки.

**Задача 3. Multi-mapping «один ко многим»**  
Один запрос с `JOIN` и `Query<Category, Product, Category>(..., splitOn: "ProductId")` — категории со списком товаров.  
**Требования:** коллекция `Products` заполняется.  
**Проверка:** количество соответствует БД; у категории без товаров список пустой (не `null`).

**Задача 4. `QueryMultiple`**  
Один вызов возвращает: (1) всего товаров, (2) товары дороже средней цены, (3) категории.  
**Требования:** `QueryMultiple` + `Read<T>()`; вернуть DTO/кортеж с тремя полями.  
**Проверка:** три набора корректны и не перемешаны.

**Задача 5. Транзакция**  
Переместите остаток между двумя товарами: `UPDATE` двух записей `Products.Stock` + запись в `AuditLog`. Всё — в `IDbTransaction`, `Execute(..., transaction: tx)`.  
**Требования:** `Rollback()` при исключении.  
**Проверка:** при искусственной ошибке после первого `UPDATE` изменения не сохраняются ни в одной таблице.

**Задача 6. Асинхронность**  
`GetAllAsync`, `InsertAsync`, `DeleteAsync` на `QueryAsync`/`ExecuteAsync` с `CancellationToken`.  
**Требования:** возвращают `Task`/`Task<T>`; токен во всех вызовах.  
**Проверка:** отмена токена → `OperationCanceledException`; соединение закрывается.

**Задача 7. `DynamicParameters` для сложного поиска**  
Метод поиска с необязательными фильтрами: `name` (LIKE), `minPrice`, `maxPrice`, `categoryIds` (список). SQL строится динамически.  
**Требования:** `IN @CategoryIds` через `DynamicParameters`; при пустых фильтрах — все товары.  
**Проверка:** SQL без условий для `null`-параметров; каждое сочетание фильтров даёт корректный результат; инъекции невозможны.
