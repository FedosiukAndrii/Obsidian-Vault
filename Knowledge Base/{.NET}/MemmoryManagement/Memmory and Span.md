### **Memory та Span у .NET**

У сучасному .NET існують **`Span<T>`** та **`Memory<T>`**, які значно покращують роботу з пам’яттю, дозволяючи ефективно оперувати буферами без зайвих алокацій. Це критично важливо для високопродуктивних застосунків, що працюють з великими обсягами даних (наприклад, обробка файлів, мережеві операції, криптографія).

---

## **1. `Span<T>` – швидка робота з пам’яттю без алокації**

### **Ключові особливості `Span<T>`:**

- **Структура (value type)** – не створює додаткових об’єктів у heap, працює в stack.
- **Доступ до масивів, `stackalloc`, `UnsafeMemory`** – ефективно працює з будь-якими блоками пам'яті.
- **Оперує даними без копіювання** – дозволяє маніпулювати даними в масиві без зайвих алокацій.

### **Основні сценарії використання `Span<T>`:**

1. **Обробка частини масиву без копіювання:**
    
    ```csharp
    int[] array = { 1, 2, 3, 4, 5 };
    Span<int> span = new Span<int>(array, 1, 3); // Доступ до {2, 3, 4}
    
    span[0] = 10; // array[1] зміниться на 10
    ```
    
2. **Робота з `stackalloc` для виділення пам’яті у стеку:**
    
    ```csharp
    Span<int> stackSpan = stackalloc int[5];
    stackSpan[0] = 42;
    ```
    
3. **Швидка операція над `string` без алокації:**
    
    ```csharp
    ReadOnlySpan<char> span = "Hello, World!".AsSpan(7, 5);
    Console.WriteLine(span.ToString()); // "World"
    ```
    

### **Обмеження `Span<T>`:**

- **Не можна зберігати у полях класу** – через те, що `Span<T>` використовує стекову пам'ять.
- **Не можна передавати в асинхронні методи** – `Span<T>` живе тільки у поточному контексті.

---

## **2. `Memory<T>` – гнучкіший варіант `Span<T>`**

### **Ключові особливості `Memory<T>`:**

- **Reference type** (на відміну від `Span<T>`) – можна зберігати у полях класу.
- **Підтримує heap-пам’ять** – може бути використаний у `async`/`await`.
- **Метод `.Span` для отримання `Span<T>`**.

### **Приклади використання `Memory<T>`:**

4. **Використання з масивами:**
    
    ```csharp
    Memory<int> memory = new int[] { 1, 2, 3, 4, 5 };
    Memory<int> slice = memory.Slice(1, 3); // {2, 3, 4}
    ```
    
5. **Робота з `async`-методами (недоступне для `Span<T>`):**
    
    ```csharp
    async Task ProcessDataAsync(Memory<byte> memory)
    {
        byte[] buffer = memory.ToArray(); // Копіюємо у heap (необхідно в async)
        await File.WriteAllBytesAsync("output.bin", buffer);
    }
    ```
    

### **Головна відмінність `Memory<T>` від `Span<T>`:**

|Параметр|`Span<T>`|`Memory<T>`|
|---|---|---|
|**Тип**|`struct`|`class`|
|**Heap чи stack?**|Stack (або stackalloc)|Heap (або масив)|
|**Можна зберігати у полях класу?**|❌ Ні|✅ Так|
|**Можна передавати в async?**|❌ Ні|✅ Так|
|**Призначення**|Високошвидкісні операції у стеку|Гнучке управління пам'яттю|

---

## **3. Оптимізація роботи з пам’яттю**

Один із основних сценаріїв використання `Memory<T>` та `Span<T>` – це **зменшення алокацій та копіювань**. Наприклад, замість традиційної операції **`Substring()`**, яка створює новий рядок, можна використовувати `Span<T>`:

```csharp
ReadOnlySpan<char> original = "Hello, World!".AsSpan();
ReadOnlySpan<char> world = original.Slice(7, 5);
Console.WriteLine(world.ToString()); // "World"
```

При роботі з великими файлами можна уникнути зайвого копіювання пам’яті, що критично важливо для продуктивності:

```csharp
Memory<byte> buffer = new byte[1024]; // Виділяємо один раз
await stream.ReadAsync(buffer); // Читаємо у вже виділений буфер
```

---

## **4. Висновки**

- `Span<T>` – **ефективний для короткотривалих операцій у стеку**, швидкий, але **не може бути використаний в async**.
- `Memory<T>` – **працює з heap, зберігається в полях класу**, підтримує **async**, але трохи менш ефективний у використанні.
- Використання `Span<T>` та `Memory<T>` дозволяє значно **оптимізувати роботу з пам’яттю**, зменшити кількість алокацій та GC-пауз.

---

Якщо ти готуєшся до **Middle/Senior .NET** співбесіди, то знання `Span<T>` та `Memory<T>` є **критично важливим** для питань на тему **пам’яті, performance tuning та zero-allocation програмування**.

Чи є якісь конкретні кейси, які ти хочеш розглянути детальніше? 🚀