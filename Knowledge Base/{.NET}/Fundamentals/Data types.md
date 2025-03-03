У .NET всі типи даних діляться на **значимі** (_value types_) та **посилальні** (_reference types_). Це розділення є основоположним і визначає спосіб зберігання та передавання даних у пам’яті.

---

## 1. Значимі типи (_Value Types_)

Значимі типи **зберігають значення безпосередньо у тій частині пам’яті, яка їм виділена**. При присвоєнні значення іншій змінній **копіюється саме значення**, а не посилання.

### Приклади:

- **Примітивні типи**: `int`, `float`, `double`, `bool`, `char`
- **Структури (`struct`)**: `DateTime`, `TimeSpan`, `Guid`
- **Перерахування (`enum`)**: `enum DayOfWeek { Monday, Tuesday, ... }`
- **Nullable Value Types (`int?`, `double?`)**

### Де зберігаються?

- Локальні змінні значимих типів **зберігаються в стеку**.
- Якщо значимий тип є частиною об’єкта (`class`), то він **зберігається в купі разом із класом**.

### Приклад коду:

```csharp
struct Point
{
    public int X;
    public int Y;
}

Point p1 = new Point { X = 10, Y = 20 };
Point p2 = p1; // Копіюється саме значення, а не посилання.
p2.X = 50;
Console.WriteLine(p1.X); // 10, оскільки p1 та p2 - незалежні екземпляри.
```

---

## 2. Посилальні типи (_Reference Types_)

Посилальні типи **зберігають у своїй змінній посилання на об’єкт у пам’яті купи**, а не саме значення. При передаванні змінної передається **посилання**, а не копія.

### Приклади:

- **Класи (`class`)**: `string`, `List<T>`, `object`
- **Масиви (`array`)**: `int[]`, `string[]`
- **Інтерфейси (`interface`)**
- **Динамічні типи (`dynamic`)**
- **Делегати (`delegate`)**

### Де зберігаються?

- **Сам об’єкт (instance) зберігається в купі (heap)**.
- **Посилання на об’єкт знаходиться в стеку**.

### Приклад:

```csharp
class Person
{
    public string Name;
}

Person p1 = new Person { Name = "Alice" };
Person p2 = p1; // Копіюється посилання на той самий об'єкт.
p2.Name = "Bob";
Console.WriteLine(p1.Name); // "Bob", бо p1 і p2 посилаються на той самий об’єкт.
```

---

## 3. Nullable типи (`Nullable<T>`, `T?`)

Значимі типи у .NET **не можуть бути null** за замовчуванням. Але **nullable-типи** (`Nullable<T>`) дозволяють зберігати `null` у значимих типах.

### Декларація:

```csharp
int? a = null;
double? b = 3.14;
```

### Використання `HasValue` та `Value`:

```csharp
if (a.HasValue)
    Console.WriteLine(a.Value);
else
    Console.WriteLine("a is null");
```

### Оператор об'єднання з `null` (`??`):

```csharp
int result = a ?? 100; // Якщо a == null, буде присвоєно 100.
```

---

## 4. Де зберігаються змінні?

### Загальні правила:

1. **Локальні змінні значимих типів** → стек.
2. **Параметри методів значимих типів** → стек.
3. **Члени класів значимих типів** → знаходяться в об'єкті класу (тобто в купі).
4. **Посилальні типи**:
    - Посилання на об’єкт → стек.
    - Сам об’єкт → купа.
5. **Статичні поля (`static`)** → купа (у спеціальній області _High Frequency Heap_).
6. **Змінні всередині `async`-методів** → можуть зберігатися в купі через автоматичну машину станів (_state machine_).

---

## 5. Додаткові нюанси

### **Boxing & Unboxing**

**Boxing** — процес перетворення значимого типу у посилальний (упаковка в `object` або `interface`).

```csharp
int num = 123;
object obj = num; // Boxing: значення переноситься в купу.
```

**Unboxing** — зворотний процес.

```csharp
int num2 = (int)obj; // Unboxing: об’єкт приводиться назад до значимого типу.
```

Ці операції **дорогі** за продуктивністю, тому їх слід мінімізувати.

### **String — особливий посилальний тип**

`string` у .NET є **посилальним типом**, але поводиться як незмінний (_immutable_).

```csharp
string s1 = "Hello";
string s2 = s1;
s1 = "World";
Console.WriteLine(s2); // "Hello" - оскільки строки незмінні.
```

Кожна зміна `string` створює **новий об'єкт у пам’яті**, що може викликати проблеми з продуктивністю (тому для частих змін рядків слід використовувати `StringBuilder`).

---

## Висновки:

- **Значимі типи (struct, enum, primitive types) зберігаються в стеку або всередині об’єкта в купі**.
- **Посилальні типи (class, array, interface, delegate) містяться в купі, а посилання на них — у стеку**.
- **Nullable типи (`T?`) дозволяють значимим типам містити `null`**.
- **Boxing/Unboxing — дорога операція, яку слід уникати**.
- **Строки (`string`) — це посилальний тип, але поводяться як значимий через незмінність**.

Якщо є додаткові питання або хочеш глибше розібрати якийсь нюанс, питай! 🚀