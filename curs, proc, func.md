# Практические примеры: Курсоры, Процедуры и Функции в MS SQL Server (T-SQL)



---

## 1. Подготовка тестовых данных

```sql
-- Создаём базу (если нужно)
IF DB_ID('company_db') IS NULL
    CREATE DATABASE company_db;
GO

USE company_db;
GO

-- Удаляем таблицы, если уже существуют (порядок важен из-за FK)
IF OBJECT_ID('dbo.employees', 'U') IS NOT NULL DROP TABLE dbo.employees;
IF OBJECT_ID('dbo.departments', 'U') IS NOT NULL DROP TABLE dbo.departments;
GO

-- Таблица отделов
CREATE TABLE dbo.departments (
    id    INT IDENTITY(1,1) PRIMARY KEY,
    name  NVARCHAR(50) NOT NULL UNIQUE
);

-- Таблица сотрудников
CREATE TABLE dbo.employees (
    id          INT IDENTITY(1,1) PRIMARY KEY,
    name        NVARCHAR(100) NOT NULL,
    department  NVARCHAR(50),
    salary      DECIMAL(10,2),
    hire_date   DATE
);
GO

-- Наполняем данными
INSERT INTO dbo.departments (name) VALUES (N'IT'), (N'HR'), (N'Sales');

INSERT INTO dbo.employees (name, department, salary, hire_date) VALUES
(N'Иван Петров',    N'IT',    120000, '2019-03-15'),
(N'Мария Сидорова', N'IT',    150000, '2018-07-01'),
(N'Олег Кузнецов',  N'HR',     80000, '2020-01-20'),
(N'Анна Иванова',   N'Sales',  95000, '2021-05-10'),
(N'Пётр Смирнов',   N'Sales', 110000, '2017-11-03');
GO
```

> **Важно:** `GO` — это не команда T-SQL, а разделитель пакетов для клиентов (SSMS, sqlcmd). `CREATE PROCEDURE` и `CREATE FUNCTION` **должны** быть первыми инструкциями в пакете, поэтому перед ними всегда ставим `GO`.

---

## 2. Простая хранимая процедура без параметров

```sql
CREATE OR ALTER PROCEDURE dbo.ListAllEmployees
AS
BEGIN
    -- Отключаем вывод "N rows affected" для чистоты результата
    SET NOCOUNT ON;

    SELECT id, name, department, salary
    FROM dbo.employees
    ORDER BY department, salary DESC;
END;
GO

-- Вызов
EXEC dbo.ListAllEmployees;
-- либо сокращённо:
-- EXEC ListAllEmployees;
```

> `CREATE OR ALTER` (SQL Server 2016 SP1+) удобнее, чем `DROP + CREATE` — не теряет права доступа.

---

## 3. Процедура с входными параметрами

**Задача:** повысить зарплату в отделе на заданный процент.

```sql
CREATE OR ALTER PROCEDURE dbo.RaiseSalary
    @Department NVARCHAR(50),   -- входной параметр
    @Percent    DECIMAL(5,2)    -- входной параметр
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @Count INT = 0;     -- локальная переменная

    -- Проверка входных данных
    IF @Percent <= 0 OR @Percent > 100
    BEGIN
        -- THROW выкидывает исключение (SQL Server 2012+)
        THROW 50001, N'Процент должен быть в диапазоне (0, 100]', 1;
    END;

    -- Обновляем зарплаты
    UPDATE dbo.employees
    SET salary = salary * (1 + @Percent / 100.0)
    WHERE department = @Department;

    -- @@ROWCOUNT — системная переменная с числом затронутых строк
    SET @Count = @@ROWCOUNT;

    IF @Count = 0
        PRINT N'В отделе ' + @Department + N' нет сотрудников';
    ELSE
        PRINT N'Обновлено сотрудников: ' + CAST(@Count AS NVARCHAR(10));
END;
GO

-- Вызов с именованными параметрами (рекомендуется)
EXEC dbo.RaiseSalary @Department = N'IT',    @Percent = 10;
EXEC dbo.RaiseSalary @Department = N'Sales', @Percent = 5;

-- Вызов с позиционными параметрами
EXEC dbo.RaiseSalary N'HR', 7;
```

---

## 4. Процедура с выходным параметром (OUTPUT)

**Задача:** вернуть среднюю зарплату отдела через OUTPUT-параметр.

```sql
CREATE OR ALTER PROCEDURE dbo.GetAvgSalary
    @Department NVARCHAR(50),
    @AvgSalary  DECIMAL(10,2) OUTPUT   -- выходной параметр
AS
BEGIN
    SET NOCOUNT ON;

    SELECT @AvgSalary = AVG(salary)
    FROM dbo.employees
    WHERE department = @Department;

    -- Защита от NULL
    IF @AvgSalary IS NULL
        SET @AvgSalary = 0;
END;
GO

-- Вызов: объявляем переменную, передаём её с ключевым словом OUTPUT
DECLARE @result DECIMAL(10,2);
EXEC dbo.GetAvgSalary @Department = N'IT', @AvgSalary = @result OUTPUT;
SELECT @result AS avg_it_salary;
```

---

## 5. Процедура, возвращающая значение через RETURN

`RETURN` в T-SQL возвращает **целое число** (код статуса), а не произвольное значение. Для произвольных типов используйте OUTPUT-параметры.

```sql
CREATE OR ALTER PROCEDURE dbo.CountEmployees
    @Department NVARCHAR(50)
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @cnt INT;
    SELECT @cnt = COUNT(*) FROM dbo.employees WHERE department = @Department;

    RETURN @cnt;   -- вернётся как код выхода
END;
GO

-- Читаем возвращаемое значение
DECLARE @rc INT;
EXEC @rc = dbo.CountEmployees @Department = N'IT';
SELECT @rc AS employee_count;
```

---

## 6. Явный курсор — построчная обработка

**Задача:** пройти по всем IT-сотрудникам, повысить зарплату на 10% и вывести лог.

```sql
CREATE OR ALTER PROCEDURE dbo.RaiseItSalaries
AS
BEGIN
    SET NOCOUNT ON;

    -- 1. Переменные для значений из курсора
    DECLARE @id       INT;
    DECLARE @name     NVARCHAR(100);
    DECLARE @salary   DECIMAL(10,2);
    DECLARE @newSal   DECIMAL(10,2);

    -- 2. Объявляем курсор
    DECLARE emp_cursor CURSOR LOCAL FAST_FORWARD FOR
        SELECT id, name, salary
        FROM dbo.employees
        WHERE department = N'IT';

    -- LOCAL        — курсор виден только внутри процедуры/пакета (рекомендуется)
    -- FAST_FORWARD — оптимизированный read-only курсор только вперёд (быстрее)

    -- 3. Открываем курсор
    OPEN emp_cursor;

    -- 4. Извлекаем первую строку
    FETCH NEXT FROM emp_cursor INTO @id, @name, @salary;

    -- 5. Цикл, пока есть строки
    WHILE @@FETCH_STATUS = 0
    BEGIN
        SET @newSal = @salary * 1.10;

        UPDATE dbo.employees
        SET salary = @newSal
        WHERE id = @id;

        PRINT N'Сотрудник ' + @name + N': '
            + CAST(@salary AS NVARCHAR(20)) + N' → '
            + CAST(@newSal AS NVARCHAR(20));

        -- Извлекаем следующую строку
        FETCH NEXT FROM emp_cursor INTO @id, @name, @salary;
    END;

    -- 6. Закрываем и освобождаем курсор
    CLOSE emp_cursor;
    DEALLOCATE emp_cursor;
END;
GO

-- Запуск
EXEC dbo.RaiseItSalaries;
```

**Ключевые моменты:**
- `@@FETCH_STATUS = 0` — строка успешно получена; `-1` — строк больше нет; `-2` — строка отсутствует.
- `CLOSE` освобождает результирующий набор, `DEALLOCATE` удаляет определение курсора. **Оба обязательны.**
- `LOCAL FAST_FORWARD` — лучший выбор по умолчанию для большинства задач.

---

## 7. Курсор с параметрами (переменные в SELECT)

В T-SQL нельзя объявить курсор с параметрами как в PostgreSQL, но можно использовать переменные в `SELECT`.

```sql
CREATE OR ALTER PROCEDURE dbo.ShowEmployeesByDept
    @Dept NVARCHAR(50)
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @name   NVARCHAR(100);
    DECLARE @salary DECIMAL(10,2);

    -- Курсор использует переменную @Dept
    DECLARE cur CURSOR LOCAL FAST_FORWARD FOR
        SELECT name, salary
        FROM dbo.employees
        WHERE department = @Dept;

    OPEN cur;

    FETCH NEXT FROM cur INTO @name, @salary;
    WHILE @@FETCH_STATUS = 0
    BEGIN
        PRINT @Dept + N': ' + @name + N' — ' + CAST(@salary AS NVARCHAR(20));
        FETCH NEXT FROM cur INTO @name, @salary;
    END;

    CLOSE cur;
    DEALLOCATE cur;
END;
GO

EXEC dbo.ShowEmployeesByDept @Dept = N'Sales';
```

---

## 8. Скалярная функция (RETURNS scalar)

**Задача:** функция считает среднюю зарплату по отделу.

```sql
CREATE OR ALTER FUNCTION dbo.fn_AvgSalary(@Department NVARCHAR(50))
RETURNS DECIMAL(10,2)
AS
BEGIN
    DECLARE @avg DECIMAL(10,2);

    SELECT @avg = AVG(salary)
    FROM dbo.employees
    WHERE department = @Department;

    RETURN ISNULL(@avg, 0);
END;
GO

-- Использование в SELECT
SELECT d.name AS department,
       dbo.fn_AvgSalary(d.name) AS avg_salary
FROM dbo.departments d;
```

> ⚠️ **Производительность:** скалярные функции в T-SQL (до SQL Server 2019) выполняются построчно и сильно тормозят запросы. По возможности заменяйте их на `APPLY` / инлайн-выражения.

---

## 9. Табличная функция (Inline Table-Valued Function, iTVF)

**Задача:** функция-отчёт — сотрудники с флагом "высокая зарплата".

```sql
CREATE OR ALTER FUNCTION dbo.fn_EmployeeReport(@MinSalary DECIMAL(10,2))
RETURNS TABLE
AS
RETURN
(
    SELECT
        e.name       AS emp_name,
        e.department AS department,
        e.salary     AS salary,
        CAST(CASE WHEN e.salary >= @MinSalary THEN 1 ELSE 0 END AS BIT) AS is_top
    FROM dbo.employees e
);
GO

-- Используем как обычную таблицу
SELECT * FROM dbo.fn_EmployeeReport(100000) ORDER BY salary DESC;
```

**Преимущества inline TVF:**
- Оптимизатор SQL Server «разворачивает» её в основной запрос (как CTE).
- Работает быстро, в отличие от скалярных функций.

---

## 10. Многострочная табличная функция (Multi-Statement TVF)

Нужна, когда логика сложнее одного `SELECT` (несколько шагов, переменные).

```sql
CREATE OR ALTER FUNCTION dbo.fn_DeptStats(@MinSalary DECIMAL(10,2))
RETURNS @result TABLE (
    department  NVARCHAR(50),
    emp_count   INT,
    avg_salary  DECIMAL(10,2),
    top_count   INT
)
AS
BEGIN
    INSERT INTO @result (department, emp_count, avg_salary, top_count)
    SELECT
        department,
        COUNT(*),
        AVG(salary),
        SUM(CASE WHEN salary >= @MinSalary THEN 1 ELSE 0 END)
    FROM dbo.employees
    GROUP BY department;

    RETURN;
END;
GO

SELECT * FROM dbo.fn_DeptStats(100000);
```

> ⚠️ MS TVF медленнее inline-версии: оптимизатор не может переписать запрос и оценивает кардинальность «на глазок» (всегда 1 строка). Применяйте только когда действительно нужна многошаговая логика.

---

## 11. Комбинированный пример: функция + курсор + процедура

**Задача:**
1. Функция `fn_EmployeesByDept` возвращает сотрудников отдела.
2. Процедура `BonusPayout` использует её как источник для курсора.

```sql
-- Табличная функция-источник
CREATE OR ALTER FUNCTION dbo.fn_EmployeesByDept(@Dept NVARCHAR(50))
RETURNS TABLE
AS
RETURN
(
    SELECT id, name, salary
    FROM dbo.employees
    WHERE department = @Dept
);
GO

-- Процедура с курсором по результату функции
CREATE OR ALTER PROCEDURE dbo.BonusPayout
    @Dept    NVARCHAR(50),
    @Percent DECIMAL(5,2)
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @id     INT;
    DECLARE @name   NVARCHAR(100);
    DECLARE @salary DECIMAL(10,2);
    DECLARE @bonus  DECIMAL(10,2);
    DECLARE @total  DECIMAL(10,2) = 0;

    DECLARE cur CURSOR LOCAL FAST_FORWARD FOR
        SELECT id, name, salary FROM dbo.fn_EmployeesByDept(@Dept);

    OPEN cur;

    FETCH NEXT FROM cur INTO @id, @name, @salary;
    WHILE @@FETCH_STATUS = 0
    BEGIN
        SET @bonus = @salary * @Percent / 100.0;
        SET @total = @total + @bonus;

        UPDATE dbo.employees
        SET salary = salary + @bonus
        WHERE id = @id;

        PRINT N'Бонус ' + @name + N': +'
            + CAST(@bonus AS NVARCHAR(20))
            + N' (итого по отделу: ' + CAST(@total AS NVARCHAR(20)) + N')';

        FETCH NEXT FROM cur INTO @id, @name, @salary;
    END;

    CLOSE cur;
    DEALLOCATE cur;

    PRINT N'ИТОГО выплачено бонусов: ' + CAST(@total AS NVARCHAR(20))
        + N' по отделу ' + @Dept;
END;
GO

-- Запуск
EXEC dbo.BonusPayout @Dept = N'IT', @Percent = 15;
```

---

## 12. Обработка исключений (TRY / CATCH)

```sql
CREATE OR ALTER PROCEDURE dbo.SafeInsertEmployee
    @Name       NVARCHAR(100),
    @Department NVARCHAR(50),
    @Salary     DECIMAL(10,2)
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        -- Проверки входных данных
        IF @Name IS NULL OR @Department IS NULL
            THROW 50002, N'Имя и отдел обязательны', 1;

        IF @Salary <= 0
            THROW 50003, N'Зарплата должна быть положительной', 1;

        INSERT INTO dbo.employees (name, department, salary, hire_date)
        VALUES (@Name, @Department, @Salary, CAST(GETDATE() AS DATE));

        PRINT N'Сотрудник ' + @Name + N' добавлен';
    END TRY
    BEGIN CATCH
        PRINT N'Ошибка: ' + ERROR_MESSAGE()
            + N' (код: ' + CAST(ERROR_NUMBER() AS NVARCHAR(10)) + N')';
        -- При необходимости пробрасываем ошибку дальше:
        -- THROW;
    END CATCH;
END;
GO

EXEC dbo.SafeInsertEmployee N'Тестовый', N'IT', 90000;
EXEC dbo.SafeInsertEmployee N'Плохой', N'IT', -5;   -- сработает CATCH
```

**Полезные функции в CATCH:**
- `ERROR_MESSAGE()` — текст ошибки
- `ERROR_NUMBER()` — код ошибки
- `ERROR_SEVERITY()` / `ERROR_STATE()` — уровень и состояние
- `ERROR_LINE()` / `ERROR_PROCEDURE()` — где именно произошла ошибка

---

## 13. Шпаргалка: Процедура vs Функция в T-SQL

| Характеристика          | PROCEDURE                       | FUNCTION                                |
|-------------------------|---------------------------------|-----------------------------------------|
| Возврат значения        | Через `RETURN` (INT) или OUTPUT | Обязательно `RETURNS`                   |
| Вызов                   | `EXEC proc ...`                 | `SELECT dbo.func(...)`                  |
| Использование в SELECT  | ❌                               | ✅                                       |
| Изменение данных (DML)  | ✅                               | ❌ (в T-SQL нельзя делать INSERT/UPDATE) |
| Управление транзакциями | ✅                               | ❌                                       |
| Может быть табличной    | Нет                             | Да (inline и multi-statement)            |

---

## 14. Шпаргалка: типы курсоров в T-SQL

| Тип                | Описание                                                      |
|--------------------|---------------------------------------------------------------|
| `FORWARD_ONLY`     | Только вперёд (по умолчанию для `FAST_FORWARD`)               |
| `SCROLL`           | Прокрутка в любую сторону (`FETCH FIRST/LAST/PRIOR/ABSOLUTE`) |
| `STATIC`           | Снимок данных — изменения в базе не видны                     |
| `DYNAMIC`          | Видны все изменения (дорого по ресурсам)                      |
| `KEYSET`           | Видны обновления, но не вставки                               |
| `READ_ONLY`        | Только чтение                                                 |
| `FAST_FORWARD`     | `FORWARD_ONLY` + `READ_ONLY` — самый быстрый                  |
| `LOCAL`            | Виден только внутри процедуры/пакета (рекомендуется)          |
| `GLOBAL`           | Виден на уровне всей сессии                                   |

**Правило:** в 90% случаев используйте `LOCAL FAST_FORWARD`.

---

## 15. Когда курсор — плохая идея

В T-SQL курсоры — **одна из самых медленных конструкций**. Прежде чем использовать, проверьте, можно ли решить задачу множественным запросом:

```sql
-- ❌ Плохо: курсор для обновления всех строк отдела
-- ✅ Хорошо: один UPDATE
UPDATE dbo.employees
SET salary = salary * 1.10
WHERE department = N'IT';
```

Курсор оправдан, когда:
- нужна пошаговая логика с разными действиями для каждой строки;
- вызываются другие процедуры для каждой строки;
- обработка зависит от состояния предыдущих строк (накопительные итоги, парсинг строк).

---

## Типичные ошибки новичков в T-SQL

1. **Забыть `DEALLOCATE`** после `CLOSE` — утечка памяти и блокировок.
2. **Использовать `GLOBAL` курсоры** там, где достаточно `LOCAL` — мешают другим сессиям.
3. **Не проверять `@@FETCH_STATUS`** — бесконечный цикл или ошибка.
4. **Путать `RETURN` (INT) с возвратом значения** — для данных используйте OUTPUT-параметры или SELECT.
5. **Использовать скалярные функции в WHERE/SELECT на большой таблице** — резкое падение производительности (до SQL Server 2019).
6. **Забыть `SET NOCOUNT ON`** — клиент получает лишние сообщения, ломающие логику приложений.
7. **Не использовать схему `dbo.`** при обращении к объектам — снижает производительность (лишний поиск).
