### **Песимістична та оптимістична конкурентність у базах даних**

У системах з багатокористувацьким доступом до даних важливо правильно керувати конкурентним доступом до ресурсів, щоб уникнути конфліктів та забезпечити узгодженість даних. У цьому контексті використовуються дві основні стратегії керування конкурентністю:

- **Песимістична конкурентність (Pessimistic Concurrency)**
- **Оптимістична конкурентність (Optimistic Concurrency)**

Кожен із підходів має свої переваги та недоліки залежно від характеру системи та очікуваного рівня конкуренції між транзакціями.

---

## **1. Песимістична конкурентність (Pessimistic Concurrency)**

### **Принцип роботи**

Песимістичний підхід передбачає, що конфлікти між транзакціями є ймовірними, тому система запобігає можливим конфліктам заздалегідь, використовуючи блокування (locks). Основна ідея – якщо один користувач працює з даними, інші користувачі не можуть їх змінювати або читати, доки операція не завершиться.

### **Типи блокувань**

- **Exclusive Lock (X-lock, монопольне блокування)** – повний доступ до ресурсу надається одній транзакції, інші користувачі не можуть його читати чи змінювати.
- **Shared Lock (S-lock, розділене блокування)** – дозволяє читання даних, але блокує їх зміну іншими транзакціями.
- **Update Lock (U-lock, оновлююче блокування)** – унікальний тип блокування, який дозволяє читання, але зарезервовує право на майбутнє оновлення.

### **Приклад у SQL Server**

Припустимо, що нам потрібно оновити запис у таблиці `Products`. Використаємо явне блокування:

```sql
BEGIN TRANSACTION;

SELECT * FROM Products WITH (UPDLOCK, HOLDLOCK) WHERE ProductID = 1;

UPDATE Products SET Price = Price + 10 WHERE ProductID = 1;

COMMIT TRANSACTION;
```

У цьому прикладі:

- `UPDLOCK` – оновлююче блокування для запобігання "переваги оновлення" (update anomaly).
- `HOLDLOCK` – еквівалентне `SERIALIZABLE`, що запобігає іншим транзакціям читати або змінювати запис до завершення транзакції.

### **Переваги**

- Гарантує цілісність даних, оскільки блокує інші транзакції до завершення операції.
- Підходить для середовищ із високою конкуренцією, де конфлікти виникають часто.
- Запобігає проблемам "dirty read", "lost update" та "non-repeatable read".

### **Недоліки**

- Може призвести до **deadlocks** (взаємне блокування), якщо транзакції блокують один одного.
- Використання блокувань зменшує рівень паралельності, що може знижувати продуктивність.
- Довгі транзакції можуть блокувати інші процеси, викликаючи затримки.

---

## **2. Оптимістична конкурентність (Optimistic Concurrency)**

### **Принцип роботи**

Оптимістичний підхід виходить з припущення, що конфлікти між транзакціями трапляються рідко. Тому система дозволяє транзакціям працювати паралельно, а перевірка конфліктів виконується лише на етапі запису.

Цей підхід часто використовує механізм **Versioning (версійність даних)** або **Timestamping (мітки часу)** для перевірки, чи не було змін у даних під час їх обробки.

### **Приклад реалізації**

Одним із найпоширеніших способів реалізації оптимістичної конкурентності є використання `RowVersion` або `Timestamp`.

#### **Приклад у SQL Server**

1. **Додамо поле `RowVersion` у таблицю**

```sql
ALTER TABLE Products ADD RowVersion ROWVERSION;
```

2. **Зчитування даних користувачем**

```sql
SELECT ProductID, Price, RowVersion FROM Products WHERE ProductID = 1;
```

1. **Оновлення з перевіркою версії**

```sql
UPDATE Products 
SET Price = Price + 10 
WHERE ProductID = 1 AND RowVersion = @OldRowVersion;
```

Якщо `RowVersion` змінився, то жоден рядок не оновиться, і користувач отримає повідомлення про конфлікт.

### **Переваги**

- Висока продуктивність у сценаріях з низькою конкуренцією, оскільки транзакції не блокують одна одну.
- Дозволяє максимальну паралельність операцій.
- Менше ризику **deadlocks**.

### **Недоліки**

- Якщо конфлікти все ж таки виникають, деякі транзакції доведеться повторювати.
- Вимагає додаткової логіки в додатку для обробки конфліктів.
- Не підходить для систем із високою частотою записів, оскільки конфлікти можуть бути частими.

---

## **3. Порівняння підходів**

|Характеристика|Песимістична конкурентність|Оптимістична конкурентність|
|---|---|---|
|**Механізм**|Блокування|Перевірка версій або міток часу|
|**Придатність для**|Високої конкуренції|Низької конкуренції|
|**Ризик блокування**|Високий|Відсутній|
|**Ризик повторних спроб**|Низький|Високий|
|**Паралельність**|Низька|Висока|
|**Проблеми продуктивності**|Deadlocks, блокування|Повторні спроби запису|

---

## **4. Висновок**

- Якщо система має **високу конкуренцію між транзакціями** (наприклад, фінансові операції, замовлення в e-commerce), **песимістична конкурентність** буде ефективнішою.
- Якщо конфлікти **малоймовірні** (наприклад, система з переважно читальними операціями або великий обсяг транзакцій на запис у розподілених системах), **оптимістична конкурентність** є кращим вибором.
- Часто реальні системи комбінують обидва підходи: наприклад, використовують **оптимістичну конкурентність** для читання та **песимістичну конкурентність** для критичних записів.

Обираючи між цими підходами, слід враховувати характер навантаження, частоту оновлення даних та необхідний рівень паралельності.

---
### **Реалізація оптимістичної конкурентності в .NET та SQL Server**

Оптимістична конкурентність ґрунтується на перевірці того, чи змінилися дані з моменту їх отримання. Це дозволяє мінімізувати блокування та покращити продуктивність у сценаріях з низькою ймовірністю конфліктів. У SQL Server для цього зазвичай використовують `RowVersion` або `Timestamp`, а в .NET – механізм `ConcurrencyCheck`.

---

## **1. Налаштування бази даних**

Додамо колонку `RowVersion` (яка автоматично оновлюється при зміні рядка):

```sql
ALTER TABLE Products ADD RowVersion ROWVERSION;
```

Приклад структури таблиці:

```sql
CREATE TABLE Products (
    ProductID INT PRIMARY KEY,
    Name NVARCHAR(100),
    Price DECIMAL(18,2),
    RowVersion ROWVERSION
);
```

---

## **2. Використання ADO.NET**

Приклад роботи з ADO.NET для реалізації оптимістичної конкурентності.

### **2.1. Отримання даних разом із `RowVersion`**

```csharp
using (var connection = new SqlConnection(connectionString))
{
    string selectQuery = "SELECT ProductID, Price, RowVersion FROM Products WHERE ProductID = @ProductID";

    using (var command = new SqlCommand(selectQuery, connection))
    {
        command.Parameters.AddWithValue("@ProductID", 1);
        connection.Open();

        using (var reader = command.ExecuteReader())
        {
            if (reader.Read())
            {
                int productId = reader.GetInt32(0);
                decimal price = reader.GetDecimal(1);
                byte[] rowVersion = (byte[])reader["RowVersion"];

                Console.WriteLine($"ProductID: {productId}, Price: {price}, RowVersion: {BitConverter.ToString(rowVersion)}");
            }
        }
    }
}
```

---

### **2.2. Оновлення запису з перевіркою `RowVersion`**

```csharp
using (var connection = new SqlConnection(connectionString))
{
    string updateQuery = @"
        UPDATE Products 
        SET Price = @NewPrice
        WHERE ProductID = @ProductID AND RowVersion = @RowVersion";

    using (var command = new SqlCommand(updateQuery, connection))
    {
        command.Parameters.AddWithValue("@ProductID", 1);
        command.Parameters.AddWithValue("@NewPrice", 150.00m);
        command.Parameters.AddWithValue("@RowVersion", rowVersion);  // Передаємо отриману версію

        connection.Open();
        int rowsAffected = command.ExecuteNonQuery();

        if (rowsAffected == 0)
        {
            Console.WriteLine("Конфлікт! Дані були змінені іншим користувачем.");
        }
        else
        {
            Console.WriteLine("Оновлення успішне!");
        }
    }
}
```

Якщо `RowVersion` змінився з моменту читання, оновлення не виконається, що дозволяє уникнути перезапису змін іншими користувачами.

---

## **3. Використання Entity Framework Core**

В EF Core є вбудована підтримка оптимістичної конкурентності через `RowVersion`.

### **3.1. Налаштування моделі**

```csharp
public class Product
{
    public int ProductID { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }

    [Timestamp]  // Автоматична підтримка оптимістичної конкурентності
    public byte[] RowVersion { get; set; }
}
```

### **3.2. Контекст бази даних**

```csharp
public class AppDbContext : DbContext
{
    public DbSet<Product> Products { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Product>()
            .Property(p => p.RowVersion)
            .IsRowVersion();
    }
}
```

---

### **3.3. Читання даних**

```csharp
using (var context = new AppDbContext())
{
    var product = context.Products.Find(1);
    Console.WriteLine($"ProductID: {product.ProductID}, Price: {product.Price}");
}
```

---

### **3.4. Оновлення даних з обробкою конфлікту**

```csharp
using (var context = new AppDbContext())
{
    var product = context.Products.Find(1);
    if (product != null)
    {
        product.Price += 10;

        try
        {
            context.SaveChanges();
            Console.WriteLine("Оновлення успішне!");
        }
        catch (DbUpdateConcurrencyException)
        {
            Console.WriteLine("Конфлікт! Дані були змінені іншим користувачем.");
        }
    }
}
```

Коли `SaveChanges()` виконується, EF Core автоматично перевіряє `RowVersion`. Якщо дані змінилися з моменту їх отримання, кидається виняток `DbUpdateConcurrencyException`.

---

## **4. Обробка конфліктів**

Якщо при оновленні виникає конфлікт, можна:

2. Повторно завантажити нові дані та запропонувати користувачеві повторне оновлення.
3. Автоматично перезаписати або об’єднати дані (merge).
4. Повідомити користувача про конфлікт.

Приклад ручної обробки конфліктів у EF Core:

```csharp
catch (DbUpdateConcurrencyException ex)
{
    foreach (var entry in ex.Entries)
    {
        if (entry.Entity is Product)
        {
            var currentValues = entry.CurrentValues;
            var databaseValues = entry.GetDatabaseValues();

            if (databaseValues != null)
            {
                Console.WriteLine("Конфлікт! Нові значення в БД:");
                foreach (var property in databaseValues.Properties)
                {
                    Console.WriteLine($"{property.Name}: {databaseValues[property]}");
                }

                // Вирішуємо конфлікт: можемо або зберегти поточні дані, або оновити їх з БД.
                entry.OriginalValues.SetValues(databaseValues);
            }
        }
    }
}
```

---

## **5. Висновок**

✅ **Оптимістична конкурентність** – ефективний механізм, який зменшує блокування та підвищує паралельність транзакцій.  
✅ Використання `RowVersion` дозволяє запобігати втраті оновлень без блокування записів.  
✅ В .NET її легко реалізувати як через **ADO.NET**, так і через **Entity Framework Core**.  
✅ У випадку конфліктів слід передбачити логіку обробки: повторне оновлення, merge або сповіщення користувача.

Цей підхід особливо добре підходить для **веб-додатків, розподілених систем і сервісів з великою кількістю читання** та **нечастими оновленнями**.