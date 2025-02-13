

---
**Interceptors** в Entity Framework Core — це механізм, що дозволяє перехоплювати та змінювати поведінку запитів до бази даних, перш ніж вони будуть виконані. Це потужний інструмент для внесення додаткової логіки або моніторингу роботи вашої програми на рівні запитів.

---

### Що таке Interceptors?

Interceptors дозволяють виконувати певний код під час виконання запитів, команд чи транзакцій. Вони працюють як middleware між вашим кодом і базою даних. Їх можна використовувати для:

1. Логування запитів.
2. Зміни SQL-команд перед виконанням.
3. Додавання метаданих до запитів.
4. Реалізації політики безпеки (наприклад, автоматичного фільтрування даних).
5. Моніторингу продуктивності.

---

### Типи Interceptors в EF Core

EF Core підтримує кілька типів інтерцепторів:

6. **Command Interceptors**:
    
    - Використовуються для перехоплення SQL-команд.
    - Дозволяють змінювати SQL-команду або виконувати додаткові дії перед чи після її виконання.
7. **Connection Interceptors**:
    
    - Використовуються для перехоплення відкриття або закриття з'єднань.
8. **Transaction Interceptors**:
    
    - Використовуються для перехоплення створення, коміту або відкату транзакцій.
9. **SaveChanges Interceptors**:
    
    - Використовуються для перехоплення операцій збереження змін у контексті.

---

### Як реалізувати Interceptor?

#### 1. **Command Interceptor**

Він дозволяє перехоплювати SQL-запити, що виконуються.

**Приклад: Логування SQL-команд**

```csharp
using Microsoft.EntityFrameworkCore.Diagnostics;
using System.Data.Common;

public class LoggingCommandInterceptor : DbCommandInterceptor
{
    public override InterceptionResult<DbDataReader> ReaderExecuting(
        DbCommand command, 
        CommandEventData eventData, 
        InterceptionResult<DbDataReader> result)
    {
        Console.WriteLine($"SQL Command Executed: {command.CommandText}");
        return base.ReaderExecuting(command, eventData, result);
    }
}
```

**Реєстрація в контексті:**

```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.AddInterceptors(new LoggingCommandInterceptor());
    base.OnConfiguring(optionsBuilder);
}
```

---

#### 2. **Connection Interceptor**

Дозволяє перехоплювати події, пов'язані із з'єднаннями.

**Приклад: Логування відкриття з'єднання**

```csharp
using Microsoft.EntityFrameworkCore.Diagnostics;

public class ConnectionLoggingInterceptor : DbConnectionInterceptor
{
    public override void ConnectionOpened(DbConnection connection, ConnectionEndEventData eventData)
    {
        Console.WriteLine($"Connection opened to database: {connection.Database}");
        base.ConnectionOpened(connection, eventData);
    }
}
```

**Реєстрація:**

```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.AddInterceptors(new ConnectionLoggingInterceptor());
    base.OnConfiguring(optionsBuilder);
}
```

---

#### 3. **Transaction Interceptor**

Перехоплює транзакції.

**Приклад: Логування транзакцій**

```csharp
using Microsoft.EntityFrameworkCore.Diagnostics;

public class TransactionLoggingInterceptor : DbTransactionInterceptor
{
    public override void TransactionStarted(
        DbTransaction transaction, 
        TransactionEndEventData eventData)
    {
        Console.WriteLine("Transaction started.");
        base.TransactionStarted(transaction, eventData);
    }
}
```

**Реєстрація:**

```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.AddInterceptors(new TransactionLoggingInterceptor());
    base.OnConfiguring(optionsBuilder);
}
```

---

#### 4. **SaveChanges Interceptor**

Перехоплює операції збереження змін (`SaveChanges` або `SaveChangesAsync`).

**Приклад: Логування перед збереженням**

```csharp
using Microsoft.EntityFrameworkCore.Diagnostics;

public class SaveChangesLoggingInterceptor : SaveChangesInterceptor
{
    public override int SavingChanges(
        DbContextEventData eventData, 
        InterceptionResult<int> result)
    {
        Console.WriteLine($"Saving changes for DbContext: {eventData.Context.GetType().Name}");
        return base.SavingChanges(eventData, result);
    }
}
```

**Реєстрація:**

```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.AddInterceptors(new SaveChangesLoggingInterceptor());
    base.OnConfiguring(optionsBuilder);
}
```

---

### Переваги Interceptors

10. **Централізований контроль:**
    - Ви можете реалізувати загальні політики на рівні запитів, з'єднань або транзакцій.
11. **Легкість моніторингу:**
    - Зручне місце для логування запитів, підрахунку часу виконання тощо.
12. **Безпека:**
    - Можливість додавати автоматичні фільтри для реалізації політик безпеки (наприклад, soft-delete).
13. **Гнучкість:**
    - Можна змінювати SQL-команди перед їх виконанням.

---

### Недоліки

14. **Ускладнення коду:**
    - Інтерцептори можуть зробити код важчим для читання та підтримки.
15. **Перевантаження:**
    - Зайві інтерцептори можуть уповільнювати виконання запитів.
16. **Необхідність глибокого тестування:**
    - Неправильна логіка в інтерцепторі може вплинути на всю роботу програми.

---

### Використання в реальних проектах

17. **Логування:**
    - Логування всіх SQL-запитів у базу даних для аудиту або відлагодження.
18. **Політики безпеки:**
    - Автоматичне застосування фільтрів для забезпечення доступу до даних лише певним користувачам.
19. **Моніторинг продуктивності:**
    - Вимірювання часу виконання запитів.
20. **Обробка специфічних помилок:**
    - Наприклад, автоматичне повторення транзакцій при збої.

---

### Рекомендації

21. Використовуйте інтерцептори тільки там, де вони дійсно потрібні.
22. Уникайте складних обчислень у межах інтерцепторів.
23. Завжди тестуйте продуктивність після додавання інтерцепторів.
24. Для великих проєктів створюйте окремі класи для різних типів інтерцепторів.

Якщо потрібна допомога з реалізацією специфічного інтерцептора або хочеш обговорити реальний сценарій використання, дай знати!