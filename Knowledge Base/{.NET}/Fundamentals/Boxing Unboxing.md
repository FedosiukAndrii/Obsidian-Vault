### **Boxing & Unboxing у .NET (Advanced)**

`Boxing` і `Unboxing` – це механізми перетворення між **значимими типами (`struct`)** та **посилальними типами (`object`, `interface`)** у .NET. Вони є важливими для розуміння ефективного керування пам'яттю та продуктивності додатків.

---

## **1. Що таке Boxing?**

`Boxing` – це **перетворення значимого типу (`value type`) у посилальний (`reference type`)**. Це означає, що значення `struct` загортається в `object`, що **переміщує його з стеку (`stack`) у купу (`heap`)**.

### **Приклад Boxing**

```csharp
int num = 42;       // num знаходиться в стеку
object obj = num;   // Boxing: num упаковується в об'єкт у купі
```

**Що відбувається під капотом?**

1. Виділяється пам'ять у купі (`heap`) для нового об'єкта.
2. Значення `num` копіюється з **стеку** у **новий об’єкт у купі**.
3. Посилання на цей об'єкт зберігається в змінній `obj`.

⏳ **Цей процес є дорогим**, оскільки включає:

- Виділення пам’яті у купі (heap allocation).
- Копіювання значення.
- Можливу додаткову роботу **Garbage Collector (GC)**.

---

## **2. Що таке Unboxing?**

`Unboxing` – це **зворотний процес**: перетворення `object` назад у `value type`, тобто вилучення значення з купи назад у стек.

### **Приклад Unboxing**

```csharp
object obj = 42;  // Boxing
int num = (int)obj; // Unboxing
```

**Що відбувається під капотом?**

4. Перевіряється, чи об'єкт справді містить значення очікуваного типу (`int` у цьому випадку).
5. Значення **копіюється** з купи в **стек**.
6. Якщо тип не збігається, буде `InvalidCastException`:

```csharp
object obj = 42;
double d = (double)obj; // InvalidCastException, бо obj містить int, а не double!
```

⏳ **Unboxing теж дорогий**, тому що включає:

- Перевірку типу (`type check`).
- Копіювання значення назад у стек.

---

## **3. Чому Boxing/Unboxing є проблемою?**

Boxing та Unboxing можуть суттєво впливати на **продуктивність** через:

7. **Додаткові виділення пам'яті у купі (heap allocations)** – може спричиняти частіші запуски Garbage Collector.
8. **Низьку швидкість виконання** – копіювання та перевірки типів займають більше часу, ніж робота з `struct` напряму.

### **Приклад з низькою продуктивністю**

```csharp
void Process()
{
    object[] values = new object[10000]; // Масив посилальних типів
    for (int i = 0; i < 10000; i++)
    {
        values[i] = i; // 10000 операцій boxing!
    }
}
```

🔴 **Кожен `i` упаковується в `object` (boxing), що призводить до тисяч виділень пам’яті!**

---

## **4. Як уникати Boxing/Unboxing?**

### ✅ **1. Використовувати `generic` замість `object`**

Методи, що приймають `object`, призводять до boxing/unboxing. Generics (`T`) дозволяють уникнути цього.

**Поганий варіант (Boxing)**:

```csharp
void Print(object obj)
{
    Console.WriteLine(obj);
}

int number = 42;
Print(number); // Boxing
```

**Оптимізований варіант (Generics, без Boxing)**:

```csharp
void Print<T>(T value)
{
    Console.WriteLine(value);
}

int number = 42;
Print(number); // Без Boxing!
```

---

### ✅ **2. Використовувати `struct` з Generics**

При роботі зі списками та колекціями використання `List<T>` замість `ArrayList` дозволяє уникнути boxing.

**Поганий варіант (Boxing)**:

```csharp
ArrayList list = new ArrayList();
list.Add(10); // Boxing
int value = (int)list[0]; // Unboxing
```

**Оптимізований варіант (`List<T>`, без Boxing)**:

```csharp
List<int> list = new List<int>();
list.Add(10); // Без Boxing
int value = list[0]; // Без Unboxing
```

---

### ✅ **3. Використовувати `readonly struct`**

Звичайні `struct` можуть випадково викликати boxing, якщо вони змінювані. `readonly struct` забороняє змінюваність і зменшує ризик boxing.

```csharp
readonly struct Point
{
    public int X { get; }
    public int Y { get; }

    public Point(int x, int y) => (X, Y) = (x, y);
}
```

---

## **5. Де Boxing/Unboxing неминучий?**

Є випадки, коли boxing **важко уникнути**:

9. **Робота з не generics API (`ArrayList`, `Hashtable`)**.
10. **Використання `object` або інтерфейсів (`IComparable`, `ICloneable`)**.
11. **Збереження `struct` у динамічних структурах (`dynamic`, `object`)**.

Приклад:

```csharp
int x = 5;
IComparable comp = x; // Boxing (інтерфейси працюють з посилальними типами)
```

---

## **Висновки**

### ❌ Boxing/Unboxing є шкідливим через:

- Додаткові **виділення пам’яті в купі**.
- **Збільшення навантаження на Garbage Collector**.
- **Зниження продуктивності** через перевірку типів та копіювання значень.

### ✅ Як уникнути:

- Використовувати **generics (`T`)**.
- Замість `ArrayList`, `Hashtable` використовувати **`List<T>`, `Dictionary<TKey, TValue>`**.
- Використовувати **`readonly struct`**, якщо структура не змінюється.
- Уникати передачі значимих типів у **`object`** або **інтерфейси**.

🚀 **Загальне правило:** якщо код активно працює з `struct` і відбувається **часта упаковка (`Boxing`)**, слід **переробити його** для використання Generics (`T`) або працювати напряму з значимими типами.

Якщо хочеш глибше розібрати конкретні аспекти (наприклад, аналізувати IL-код boxing/unboxing), кажи! 🔥

---
>- В яких ситуаціях буде або не буде boxing для nullable типу
---
### **Boxing та `Nullable<T>`: Як це працює?**

У випадку з `Nullable<T>` (`int?`, `double?`, тощо) **boxing може або не може відбутися** залежно від контексту. Давайте розглянемо детально.

---

## **1. Коли відбувається `boxing` для `Nullable<T>`?**

Boxing для `Nullable<T>` відбувається **тільки якщо `HasValue == true`**.  
При цьому `Nullable<T>` **автоматично розпаковується (`unwrap`) перед boxing'ом**, і в купі зберігається не сам `Nullable<T>`, а його базове значення.

### **Приклад Boxing**

```csharp
int? nullableValue = 42;
object boxed = nullableValue; // Boxing
```

**Що відбувається під капотом?**

1. `Nullable<int>` містить значення `42` (`HasValue == true`).
2. Перед boxing-ом `Nullable<T>` автоматично **розпаковується в `int`**.
3. `int` упаковується (boxing) в `object`.

📌 **В результаті в купі зберігається `int`, а не `Nullable<int>`!**

---

## **2. Коли boxing НЕ відбувається?**

### ❌ **1. Якщо `Nullable<T>` містить `null`**

Якщо `Nullable<T>` не має значення (`HasValue == false`), boxing **не відбувається**.  
Замість цього `object` отримує `null`!

```csharp
int? nullableValue = null;
object boxed = nullableValue; // НЕ відбувається boxing, boxed = null
```

**Що відбувається під капотом?**

- Оскільки `nullableValue` не містить значення (`HasValue == false`), результат boxing’у буде просто `null`, а не boxed-об’єкт у купі.

---

### ❌ **2. Якщо `Nullable<T>` передається в generics без обмеження `class`**

Якщо `Nullable<T>` використовується в generics, то boxing **не відбувається**, оскільки `Nullable<T>` – це структура (`struct`).

```csharp
void Print<T>(T value)
{
    Console.WriteLine(value);
}

int? nullableInt = 42;
Print(nullableInt); // Немає boxing, бо T = int? (struct)
```

✅ **Тут generics зберігає значимий тип напряму, без перетворення в `object`!**

Але якщо generics має обмеження `where T : class`, відбувається boxing (див. наступний приклад).

---

## **3. Коли boxing може бути неочікуваним?**

### ❗ **Якщо `Nullable<T>` передається в метод, що приймає `object` або інтерфейс**

```csharp
void PrintObject(object obj)
{
    Console.WriteLine(obj);
}

int? nullableInt = 42;
PrintObject(nullableInt); // Boxing відбувається
```

📌 **Оскільки метод очікує `object`, `Nullable<int>` розпаковується в `int`, і тільки тоді відбувається boxing.**

---

### ❗ **Якщо `Nullable<T>` передається в інтерфейс**

```csharp
interface IExample { }
struct MyStruct : IExample { }

IExample example = new MyStruct(); // Boxing
```

📌 **Boxing відбувається, бо `struct` передається через інтерфейс.**

Але з `Nullable<T>` це працює так:

```csharp
IExample example = (IExample)(new MyStruct()?); // Boxing!
```

📌 **Тут boxing відбувається, бо `Nullable<T>` розпаковується у `T`, а `T` вже проходить через інтерфейс.**

---

## **Висновки:**

4. **Boxing відбувається, якщо `Nullable<T>` має значення (`HasValue == true`) та приводиться до `object` чи інтерфейсу.**
    
    ```csharp
    int? x = 42;
    object obj = x; // Boxing (в купі буде `int`, а не `Nullable<int>`)
    ```
    
5. **Boxing НЕ відбувається, якщо `Nullable<T>` містить `null` – в `object` буде просто `null`.**
    
    ```csharp
    int? x = null;
    object obj = x; // obj == null (немає boxing)
    ```
    
6. **Якщо `Nullable<T>` передається у Generics (`T` без `class`-обмеження), boxing не відбувається.**
    
    ```csharp
    void Print<T>(T value) { Console.WriteLine(value); }
    int? x = 42;
    Print(x); // Немає boxing
    ```
    
7. **Якщо `Nullable<T>` передається у метод, що приймає `object`, boxing відбувається.**
    
    ```csharp
    void Print(object obj) { Console.WriteLine(obj); }
    int? x = 42;
    Print(x); // Boxing
    ```
    
8. **Якщо `Nullable<T>` приводиться до інтерфейсу (`IComparable` тощо), boxing відбувається.**
    
    ```csharp
    IComparable x = (IComparable)(42 as int?); // Boxing
    ```
    

### **Практичне правило:**

🔹 Використовуй `Nullable<T>` **без перетворення у `object` або інтерфейси**, щоб уникати непотрібного `boxing`.  
🔹 **Generics (`T`) допомагають уникати `boxing/unboxing`.**  
🔹 Якщо `Nullable<T>` **`null`**, то `boxing` не відбувається – `object` просто отримує `null`.

Якщо хочеш розглянути ще глибші аспекти, наприклад, як це виглядає на рівні **IL-коду**, дай знати! 🚀
