# Полное руководство по DataSet, DataAdapter и DataTable в ADO.NET (C#)

## Часть 0. Создание учебной базы данных

Для всех примеров будем использовать SQL Server (LocalDB или Express). Создадим базу `ShopDB` с двумя связанными таблицами.

```sql
-- Создание базы данных (если ещё не существует)
IF DB_ID('ShopDB') IS NULL
    CREATE DATABASE ShopDB;
GO

USE ShopDB;
GO

-- Таблица категорий (родительская)
IF OBJECT_ID('Categories', 'U') IS NOT NULL DROP TABLE Categories;
CREATE TABLE Categories (
    CategoryID   INT PRIMARY KEY IDENTITY(1,1),
    CategoryName NVARCHAR(50) NOT NULL
);

-- Таблица товаров (дочерняя, связана с Categories)
IF OBJECT_ID('Products', 'U') IS NOT NULL DROP TABLE Products;
CREATE TABLE Products (
    ProductID   INT PRIMARY KEY IDENTITY(1,1),
    ProductName NVARCHAR(100) NOT NULL,
    CategoryID  INT NOT NULL FOREIGN KEY REFERENCES Categories(CategoryID),
    Price       DECIMAL(10,2) NOT NULL,
    Stock       INT NOT NULL DEFAULT 0
);

-- Начальные данные
INSERT INTO Categories (CategoryName) VALUES
(N'Электроника'), (N'Книги'), (N'Одежда');

INSERT INTO Products (ProductName, CategoryID, Price, Stock) VALUES
(N'Смартфон',    1, 29999.00, 15),
(N'Ноутбук',     1, 54999.00, 8),
(N'Наушники',    1,  4999.00, 30),
(N'Роман "1984"',2,   599.00, 50),
(N'Учебник C#',  2,  1200.00, 25),
(N'Футболка',    3,   899.00, 100);
GO
```

Теперь у нас есть база с двумя связанными таблицами и тестовыми данными.


## Часть 1. DataTable — таблица данных в памяти

### Что это такое

`DataTable` — это объект, представляющий **одну таблицу** с данными в оперативной памяти. Он содержит коллекцию строк (`DataRowCollection`), коллекцию столбцов (`DataColumnCollection`) и ограничения (`ConstraintCollection`). DataTable не знает, откуда пришли данные — из SQL Server, из XML или созданы вручную .

### Пример 1.1: Создание DataTable вручную и заполнение данными

```csharp
using System;
using System.Data;

namespace DataSetGuide
{
    class DataTableManualExample
    {
        static void Main()
        {
            // 1. Создаем пустую таблицу с именем "Employees"
            DataTable employees = new DataTable("Employees");

            // 2. Определяем схему: добавляем столбцы с типами
            //    DataColumn(name, type) — имя и .NET-тип данных
            employees.Columns.Add("Id", typeof(int));
            employees.Columns.Add("Name", typeof(string));
            employees.Columns.Add("Salary", typeof(decimal));

            // 3. Добавляем строки.
            //    NewRow() создает строку с той же схемой, что и таблица
            DataRow row1 = employees.NewRow();
            row1["Id"] = 1;
            row1["Name"] = "Анна";
            row1["Salary"] = 75000m;
            employees.Rows.Add(row1); // добавляем в коллекцию строк

            // Можно добавить строку через массив значений (порядок = порядок столбцов)
            employees.Rows.Add(2, "Борис", 82000m);

            // 4. Чтение данных: перебор всех строк
            Console.WriteLine("=== Ручная DataTable ===");
            foreach (DataRow row in employees.Rows)
            {
                // Доступ по имени столбца (удобнее) или по индексу
                Console.WriteLine($"ID: {row["Id"]}, Имя: {row["Name"]}, Зарплата: {row["Salary"]}");
            }

            // 5. Поиск строки через Select (возвращает массив DataRow)
            DataRow[] found = employees.Select("Name = 'Анна'");
            if (found.Length > 0)
                Console.WriteLine($"\nНайдено: {found[0]["Name"]}");

            // 6. Изменение данных
            found[0]["Salary"] = 80000m;
            Console.WriteLine($"Новая зарплата: {found[0]["Salary"]}");
        }
    }
}
```

**Комментарии к ключевым моментам:**
- `DataTable` можно создать без базы данных — это полностью автономный объект .
- `NewRow()` создает `DataRow`, но не добавляет её в таблицу. Это позволяет заполнить все поля до добавления, чтобы избежать ошибок валидации.
- `Select()` использует синтаксис, похожий на SQL WHERE, но работает **только в памяти**.


## Часть 2. DataAdapter — мост между базой и DataTable/DataSet

### Что это такое

`DataAdapter` — это компонент, который выполняет две задачи:
1. **Заполняет** DataTable или DataSet данными из источника (через `Fill`).
2. **Обновляет** источник данных на основе изменений в DataTable/DataSet (через `Update`) .

`SqlDataAdapter` — реализация для SQL Server. При вызове `Fill()` адаптер сам открывает соединение (если оно закрыто), выполняет `SelectCommand`, заполняет таблицу и закрывает соединение .

### Пример 2.1: Заполнение DataTable через SqlDataAdapter

```csharp
using System;
using System.Data;
using System.Data.SqlClient;

namespace DataSetGuide
{
    class DataAdapterFillExample
    {
        // Строка подключения — замените на свою, если используете другой сервер
        private const string ConnectionString =
            @"Data Source=(localdb)\MSSQLLocalDB;Initial Catalog=ShopDB;Integrated Security=True;";

        static void Main()
        {
            // 1. Создаем DataTable с именем "Products"
            DataTable productsTable = new DataTable("Products");

            // 2. Создаем адаптер.
            //    Конструктор принимает SQL-запрос и строку подключения.
            //    Адаптер сам создаст SqlConnection и SqlCommand внутри.
            using (SqlDataAdapter adapter = new SqlDataAdapter(
                "SELECT ProductID, ProductName, Price, Stock FROM Products", 
                ConnectionString))
            {
                // 3. Fill заполняет DataTable данными.
                //    ВАЖНО: Fill автоматически создает столбцы в DataTable,
                //    если их ещё нет, на основе метаданных результата запроса.
                adapter.Fill(productsTable);
            }
            // using гарантирует Dispose адаптера (и его соединения)

            // 4. Вывод данных
            Console.WriteLine("=== Товары из БД ===");
            Console.WriteLine($"Столбцов: {productsTable.Columns.Count}, Строк: {productsTable.Rows.Count}\n");

            foreach (DataRow row in productsTable.Rows)
            {
                Console.WriteLine($"{row["ProductID"]}: {row["ProductName"]} — {row["Price"]:C}");
            }

            // 5. DataTable можно менять в памяти — изменения НЕ попадут в БД
            //    пока мы не вызовем adapter.Update() (см. следующий пример)
            productsTable.Rows[0]["Price"] = 27999m;
            Console.WriteLine($"\nЦена изменена в памяти: {productsTable.Rows[0]["Price"]:C}");
        }
    }
}
```

**Важно:** `Fill()` создает таблицу, если её нет, но **не создает первичный ключ**, если в запросе не указан `MissingSchemaAction.AddWithKey` . Для простого чтения это не критично.

### Пример 2.2: Обновление базы данных через DataAdapter и CommandBuilder

Чтобы отправить изменения из DataTable обратно в БД, DataAdapter нужны команды `InsertCommand`, `UpdateCommand`, `DeleteCommand`. `SqlCommandBuilder` может сгенерировать их автоматически, если `SelectCommand` возвращает первичный ключ .

```csharp
using System;
using System.Data;
using System.Data.SqlClient;

namespace DataSetGuide
{
    class DataAdapterUpdateExample
    {
        private const string ConnectionString =
            @"Data Source=(localdb)\MSSQLLocalDB;Initial Catalog=ShopDB;Integrated Security=True;";

        static void Main()
        {
            DataTable productsTable = new DataTable("Products");
            SqlDataAdapter adapter;

            // 1. Загружаем данные
            using (SqlConnection conn = new SqlConnection(ConnectionString))
            {
                adapter = new SqlDataAdapter(
                    "SELECT ProductID, ProductName, CategoryID, Price, Stock FROM Products",
                    conn);
                
                // CommandBuilder анализирует SelectCommand и генерирует
                // Insert/Update/Delete команды автоматически.
                // Требование: в запросе должен быть первичный ключ.
                SqlCommandBuilder builder = new SqlCommandBuilder(adapter);

                adapter.Fill(productsTable);
            }

            Console.WriteLine($"Загружено строк: {productsTable.Rows.Count}");

            // 2. Вносим изменения в память
            //    Добавляем новый товар
            DataRow newRow = productsTable.NewRow();
            newRow["ProductName"] = "Планшет";
            newRow["CategoryID"] = 1;
            newRow["Price"] = 18999m;
            newRow["Stock"] = 12;
            productsTable.Rows.Add(newRow);

            //    Изменяем существующий
            productsTable.Rows[0]["Price"] = 26999m;

            //    Удаляем строку (например, последнюю)
            productsTable.Rows[productsTable.Rows.Count - 1].Delete();

            // 3. Отправляем изменения в БД
            //    adapter.Update анализирует RowState каждой строки:
            //    Added -> InsertCommand, Modified -> UpdateCommand, Deleted -> DeleteCommand
            using (SqlConnection conn = new SqlConnection(ConnectionString))
            {
                adapter.Update(productsTable);
            }

            Console.WriteLine("Изменения отправлены в БД.");
        }
    }
}
```

**Что происходит при `Update()`:** DataAdapter просматривает каждую строку DataTable и смотрит на её `RowState`. Для добавленных строк выполняется `InsertCommand`, для изменённых — `UpdateCommand`, для удалённых — `DeleteCommand` .


## Часть 3. DataSet — контейнер для нескольких таблиц и связей

### Что это такое

`DataSet` — это **набор DataTable** плюс связи (`DataRelation`) между ними. Это мини-база данных в памяти. DataSet может содержать несколько несвязанных таблиц или связанные таблицы с ограничениями целостности .

### Пример 3.1: Заполнение DataSet двумя таблицами и создание связи

```csharp
using System;
using System.Data;
using System.Data.SqlClient;

namespace DataSetGuide
{
    class DataSetRelationExample
    {
        private const string ConnectionString =
            @"Data Source=(localdb)\MSSQLLocalDB;Initial Catalog=ShopDB;Integrated Security=True;";

        static void Main()
        {
            DataSet shopDS = new DataSet("ShopDataSet");

            // 1. Заполняем Categories
            using (SqlDataAdapter catAdapter = new SqlDataAdapter(
                "SELECT CategoryID, CategoryName FROM Categories", ConnectionString))
            {
                catAdapter.Fill(shopDS, "Categories"); // имя таблицы в DataSet
            }

            // 2. Заполняем Products
            using (SqlDataAdapter prodAdapter = new SqlDataAdapter(
                "SELECT ProductID, ProductName, CategoryID, Price FROM Products", ConnectionString))
            {
                prodAdapter.Fill(shopDS, "Products");
            }

            // 3. Создаем связь между таблицами (как внешний ключ, но в памяти)
            //    DataRelation(имя, родительский столбец, дочерний столбец)
            DataColumn parentCol = shopDS.Tables["Categories"].Columns["CategoryID"];
            DataColumn childCol  = shopDS.Tables["Products"].Columns["CategoryID"];

            DataRelation relation = new DataRelation(
                "CategoryProducts",  // имя связи
                parentCol,
                childCol);
            shopDS.Relations.Add(relation);

            // 4. Навигация по связи: для каждой категории выводим её товары
            Console.WriteLine("=== Категории и их товары ===");
            foreach (DataRow catRow in shopDS.Tables["Categories"].Rows)
            {
                Console.WriteLine($"\n[{catRow["CategoryName"]}]");

                // GetChildRows возвращает все дочерние строки по связи
                DataRow[] products = catRow.GetChildRows("CategoryProducts");
                foreach (DataRow p in products)
                {
                    Console.WriteLine($"  - {p["ProductName"]} ({p["Price"]:C})");
                }
            }

            // 5. Обратная навигация: от товара к категории
            DataRow firstProduct = shopDS.Tables["Products"].Rows[0];
            DataRow parentCategory = firstProduct.GetParentRow("CategoryProducts");
            Console.WriteLine($"\nТовар '{firstProduct["ProductName"]}' " +
                              $"принадлежит категории '{parentCategory["CategoryName"]}'");
        }
    }
}
```

**Ключевое преимущество DataSet:** связь `DataRelation` позволяет перемещаться между таблицами без повторных запросов к БД. `GetChildRows` и `GetParentRow` работают так же, как навигация по внешним ключам в реляционной БД .


## Часть 4. Задачи на разбор (5 штук с решениями)

### Задача 1. Создание DataTable вручную

**Условие:** Создайте DataTable `Students` с полями `Id` (int), `FullName` (string), `Grade` (double). Добавьте трех студентов и выведите тех, у кого оценка выше 4.5.

**Решение:**
```csharp
DataTable students = new DataTable("Students");
students.Columns.Add("Id", typeof(int));
students.Columns.Add("FullName", typeof(string));
students.Columns.Add("Grade", typeof(double));

students.Rows.Add(1, "Иванов Иван", 4.8);
students.Rows.Add(2, "Петрова Анна", 4.2);
students.Rows.Add(3, "Сидоров Олег", 5.0);

DataRow[] excellent = students.Select("Grade > 4.5");
foreach (DataRow r in excellent)
    Console.WriteLine($"{r["FullName"]}: {r["Grade"]}");
```

---

### Задача 2. Заполнение DataTable через адаптер с фильтрацией

**Условие:** Загрузите только товары с ценой > 5000 в DataTable и выведите их названия.

**Решение:**
```csharp
DataTable expensive = new DataTable();
using (SqlDataAdapter da = new SqlDataAdapter(
    "SELECT ProductName, Price FROM Products WHERE Price > 5000",
    ConnectionString))
{
    da.Fill(expensive);
}
foreach (DataRow row in expensive.Rows)
    Console.WriteLine(row["ProductName"]);
```

---

### Задача 3. DataSet с двумя таблицами и подсчет количества

**Условие:** Загрузите Categories и Products в DataSet, создайте связь. Для каждой категории выведите количество товаров.

**Решение:**
```csharp
DataSet ds = new DataSet();
using (SqlDataAdapter da = new SqlDataAdapter("SELECT * FROM Categories", ConnectionString))
    da.Fill(ds, "Categories");
using (SqlDataAdapter da = new SqlDataAdapter("SELECT * FROM Products", ConnectionString))
    da.Fill(ds, "Products");

ds.Relations.Add("CatProd",
    ds.Tables["Categories"].Columns["CategoryID"],
    ds.Tables["Products"].Columns["CategoryID"]);

foreach (DataRow cat in ds.Tables["Categories"].Rows)
{
    int count = cat.GetChildRows("CatProd").Length;
    Console.WriteLine($"{cat["CategoryName"]}: {count} товаров");
}
```

---

### Задача 4. Изменение данных в DataTable без сохранения в БД

**Условие:** Загрузите товары, поднимите цену всех товаров категории 1 на 10%, выведите новые цены. Убедитесь, что в БД ничего не изменилось.

**Решение:**
```csharp
DataTable dt = new DataTable();
using (SqlDataAdapter da = new SqlDataAdapter(
    "SELECT ProductID, ProductName, CategoryID, Price FROM Products", ConnectionString))
{
    da.Fill(dt);
}

foreach (DataRow row in dt.Select("CategoryID = 1"))
{
    decimal oldPrice = (decimal)row["Price"];
    row["Price"] = oldPrice * 1.10m;
    Console.WriteLine($"{row["ProductName"]}: {oldPrice} -> {row["Price"]}");
}
// БД не тронута, потому что не было вызова adapter.Update()
```

---

### Задача 5. Обновление БД через DataAdapter

**Условие:** Загрузите товары, измените цену одного товара, добавьте новый товар, удалите один существующий. Сохраните изменения в БД через `SqlCommandBuilder`.

**Решение:**
```csharp
DataTable dt = new DataTable();
using (SqlConnection conn = new SqlConnection(ConnectionString))
{
    SqlDataAdapter da = new SqlDataAdapter(
        "SELECT ProductID, ProductName, CategoryID, Price, Stock FROM Products", conn);
    SqlCommandBuilder cb = new SqlCommandBuilder(da);
    da.Fill(dt);

    // Изменение
    dt.Rows[0]["Price"] = 25999m;

    // Добавление
    DataRow newRow = dt.NewRow();
    newRow["ProductName"] = "Клавиатура";
    newRow["CategoryID"] = 1;
    newRow["Price"] = 3500m;
    newRow["Stock"] = 40;
    dt.Rows.Add(newRow);

    // Удаление (последняя строка)
    dt.Rows[dt.Rows.Count - 1].Delete();

    da.Update(dt);
}
Console.WriteLine("БД обновлена.");
```
---
## Итог

- **DataTable** — одна таблица в памяти. Можно создать вручную или заполнить через адаптер.
- **DataAdapter** — мост: `Fill()` тянет данные из БД в DataTable/DataSet, `Update()` отправляет изменения обратно.
- **DataSet** — контейнер для нескольких DataTable + DataRelation. Позволяет работать со связанными данными без постоянного подключения к БД.

Все три класса реализуют **отключенную модель** ADO.NET: вы загружаете данные, работаете с ними в памяти, и только потом (если нужно) отправляете изменения обратно .
