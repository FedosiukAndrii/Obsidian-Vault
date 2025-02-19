### Transaction Management в EF Core на рівні **Advanced**

Управління транзакціями в **Entity Framework Core (EF Core)** є критичним аспектом для забезпечення узгодженості даних у складних бізнес-операціях. На **просунутому рівні** варто розглянути різні підходи та механізми, які дають максимальний контроль над транзакціями.

---

## **1. Вбудоване управління транзакціями в EF Core**

EF Core підтримує **автоматичні транзакції** при виконанні `SaveChanges()`. Це означає, що якщо **SaveChanges()** викликається без явного використання транзакції, всі зміни застосовуються в рамках однієї транзакції.

```csharp
using (var context = new ApplicationDbContext())
{
    context.Add(new Product { Name = "Laptop", Price = 1200 });
    context.SaveChanges(); // Виконується в одній транзакції
}
```

Однак це не дає гнучкості для складних сценаріїв, коли потрібно працювати з **кількома SaveChanges()** або **роботою з кількома контекстами**.

---

## **2. Явне управління транзакціями**

Для розширеного контролю можна використовувати **DbContext.Database.BeginTransaction()**.

### **2.1. Використання явної транзакції**

```csharp
using (var context = new ApplicationDbContext())
{
    using (var transaction = context.Database.BeginTransaction())
    {
        try
        {
            context.Add(new Product { Name = "Phone", Price = 800 });
            context.SaveChanges(); // Перше збереження в рамках транзакції

            context.Add(new Order { ProductId = 1, Quantity = 2 });
            context.SaveChanges(); // Друге збереження

            transaction.Commit(); // Фіксуємо транзакцію
        }
        catch
        {
            transaction.Rollback(); // Відкат у разі помилки
        }
    }
}
```

**Плюси цього підходу**:

- Можна контролювати **момент коміту**.
- Забезпечується **узгодженість кількох операцій**.
- Підтримує **відкат у разі помилки**.

---

## **3. Транзакції з декількома DbContext**

Якщо у вас **декілька контекстів**, необхідно використовувати **одну транзакцію для всіх контекстів**.

```csharp
using (var context1 = new ApplicationDbContext())
using (var context2 = new AnotherDbContext())
{
    using (var transaction = context1.Database.BeginTransaction())
    {
        try
        {
            context1.Products.Add(new Product { Name = "Tablet", Price = 500 });
            context1.SaveChanges();

            context2.Orders.Add(new Order { ProductId = 1, Quantity = 1 });
            context2.SaveChanges();

            transaction.Commit();
        }
        catch
        {
            transaction.Rollback();
        }
    }
}
```

**Важливо**: Два різні **DbContext не можуть розділяти одну транзакцію** за замовчуванням. Це можливо тільки через **sp_getapplock** або **TransactionScope** (див. далі).

---

## **4. Використання TransactionScope**

Якщо необхідно об'єднати операції в різних контекстах або навіть у різних базах даних, можна використовувати **TransactionScope**.

```csharp
using (var scope = new TransactionScope())
{
    using (var context1 = new ApplicationDbContext())
    {
        context1.Products.Add(new Product { Name = "Monitor", Price = 300 });
        context1.SaveChanges();
    }

    using (var context2 = new AnotherDbContext())
    {
        context2.Orders.Add(new Order { ProductId = 2, Quantity = 3 });
        context2.SaveChanges();
    }

    scope.Complete(); // Коміт всіх операцій
}
```

**Особливості**:

- `TransactionScope` автоматично об’єднує всі транзакції.
- Підтримується **розподілена транзакція** (якщо це можливо в SQL Server).
- `TransactionScope` може **не працювати у .NET Core** без **System.Transactions.Configuration** (потрібен повноцінний DTC).

> **ВАЖЛИВО**: Починаючи з **EF Core 3.0**, `TransactionScope` працює лише в **синхронних** сценаріях.

---

## **5. Управління транзакціями на рівні SQL**

Якщо потрібно виконати транзакцію на рівні **запитів SQL**, можна використовувати `DbContext.Database.ExecuteSqlRaw()`.

```csharp
using (var context = new ApplicationDbContext())
{
    using (var transaction = context.Database.BeginTransaction())
    {
        try
        {
            context.Database.ExecuteSqlRaw("INSERT INTO Products (Name, Price) VALUES ('SSD', 250)");
            context.Database.ExecuteSqlRaw("INSERT INTO Orders (ProductId, Quantity) VALUES (3, 5)");

            transaction.Commit();
        }
        catch
        {
            transaction.Rollback();
        }
    }
}
```

Цей метод дозволяє **безпосередньо працювати із SQL** у транзакції.

---

## **6. Політика повторних спроб (Retry Policy)**

Якщо застосунок працює з **Azure SQL** або іншими **хмарними базами**, потрібно використовувати механізми **повторних спроб у разі збоїв**.

EF Core підтримує **Resilient Transaction Pattern**:

```csharp
var strategy = context.Database.CreateExecutionStrategy();

await strategy.ExecuteAsync(async () =>
{
    using (var transaction = await context.Database.BeginTransactionAsync())
    {
        try
        {
            context.Products.Add(new Product { Name = "Camera", Price = 600 });
            await context.SaveChangesAsync();

            transaction.Commit();
        }
        catch
        {
            transaction.Rollback();
        }
    }
});
```

Цей підхід використовує **ExecutionStrategy**, який повторно виконує транзакцію у разі **тимчасових збоїв**.

---

## **7. Використання Outbox Pattern (RabbitMQ, Kafka)**

Якщо система використовує **асинхронну обробку подій** (наприклад, **RabbitMQ, Kafka**), важливо уникати проблеми подвійного відправлення повідомлень у разі відкату транзакції.

**Outbox Pattern** вирішує це:

1. Подія спочатку **записується в БД** у рамках транзакції.
2. Окремий **бекграунд-воркер** читає повідомлення з БД та публікує в **чергу**.

```csharp
using (var transaction = await context.Database.BeginTransactionAsync())
{
    try
    {
        var product = new Product { Name = "Drone", Price = 1500 };
        context.Products.Add(product);
        await context.SaveChangesAsync();

        context.OutboxMessages.Add(new OutboxMessage
        {
            EventType = "ProductCreated",
            Payload = JsonConvert.SerializeObject(product)
        });

        await context.SaveChangesAsync();
        await transaction.CommitAsync();
    }
    catch
    {
        await transaction.RollbackAsync();
    }
}
```

Потім окремий процес читає `OutboxMessages` та публікує в **RabbitMQ / Kafka**.

---

## **Висновки**

EF Core надає кілька потужних інструментів для роботи з транзакціями: ✅ **Автоматичні транзакції** (`SaveChanges()`).  
✅ **Явні транзакції** (`BeginTransaction()`, `TransactionScope`).  
✅ **Транзакції між кількома DbContext**.  
✅ **Використання SQL-запитів у транзакціях**.  
✅ **Політики повторних спроб (Retry Policy)** для хмарних БД.  
✅ **Outbox Pattern** для інтеграції з чергами повідомлень.

Якщо проект працює у **високонавантаженому середовищі**, варто комбінувати **ExecutionStrategy + Outbox Pattern + TransactionScope** для досягнення максимальної надійності. 🚀