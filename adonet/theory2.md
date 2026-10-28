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
