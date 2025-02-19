### **WeakReference в .NET**

`WeakReference<T>` – це клас у .NET, що дозволяє зберігати слабке посилання на об'єкт, яке не заважає **Garbage Collector (GC)** його видаляти, якщо на цей об'єкт більше немає сильних посилань.

#### **Чому це важливо?**

У звичайному випадку, коли ми зберігаємо об'єкт у змінній або колекції, на нього створюється **сильне посилання**, яке не дозволяє GC його видалити. Це може призвести до витоків пам’яті, якщо об'єкт більше не потрібен.

`WeakReference<T>` дозволяє зберігати об'єкт, але GC може його видалити, якщо недостатньо пам’яті.

---

## **🔍 Основи використання WeakReference**

```csharp
using System;

class Program
{
    static void Main()
    {
        var strongReference = new object();
        WeakReference<object> weakReference = new WeakReference<object>(strongReference);

        Console.WriteLine(weakReference.TryGetTarget(out var obj)); // True (об'єкт існує)

        strongReference = null; // Відпускаємо сильне посилання
        GC.Collect(); // Викликаємо GC

        Console.WriteLine(weakReference.TryGetTarget(out obj)); // False (об'єкт міг бути видалений)
    }
}
```

### **Основні методи**

|Метод|Опис|
|---|---|
|`new WeakReference<T>(T target)`|Створює слабке посилання на об'єкт.|
|`TryGetTarget(out T target)`|Перевіряє, чи об'єкт ще існує.|
|`SetTarget(T target)`|Встановлює нове слабке посилання.|

---

## **📌 Де застосовувати WeakReference**

### ✅ **Коли використовувати:**

1. **Кешування великих об'єктів**, щоб GC міг звільнити їх, якщо недостатньо пам’яті.
2. **Події (event handlers)**, щоб уникнути витоків пам’яті.
3. **Тимчасові об’єкти, які можуть не знадобитися в майбутньому.**
4. **Об’єкти у довготривалих структурах (наприклад, `Dictionary<K, V>`)**, де вони можуть ставати застарілими.

### ❌ **Коли не використовувати:**

1. **Якщо об'єкт завжди потрібен** – GC може видалити його в будь-який момент.
2. **Для критичних даних** – немає гарантії, що об’єкт буде доступним.

---

## **🔄 Приклад кешування з WeakReference**

```csharp
using System;
using System.Collections.Generic;

class Cache<T>
{
    private Dictionary<string, WeakReference<T>> _cache = new();

    public void Add(string key, T item)
    {
        _cache[key] = new WeakReference<T>(item);
    }

    public T Get(string key)
    {
        if (_cache.TryGetValue(key, out var weakRef) && weakRef.TryGetTarget(out var target))
        {
            return target;
        }
        return default; // Об'єкт був видалений GC
    }
}

class Program
{
    static void Main()
    {
        Cache<string> cache = new();
        cache.Add("key1", "Hello, WeakReference!");

        GC.Collect(); // Примусовий збір сміття

        string value = cache.Get("key1");
        Console.WriteLine(value ?? "Object was collected"); // Може бути null
    }
}
```

---

### **📝 Висновки**

- **WeakReference** дозволяє зберігати посилання на об'єкт, не блокуючи його збір GC.
- Корисно для **кешування**, **обробки подій** та **оптимізації пам’яті**.
- Не підходить для критичних даних, оскільки GC може видалити об’єкт у будь-який момент.

Якщо у вас є конкретний кейс, можу допомогти з оптимізацією! 🚀