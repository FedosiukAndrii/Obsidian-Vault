### **Entity Framework Lazy Loading – детальний розбір**

Lazy Loading (ліниве завантаження) – це одна з стратегій завантаження зв’язаних даних в Entity Framework (EF), яка дозволяє завантажувати дані лише тоді, коли вони фактично потрібні. Це корисно для оптимізації продуктивності застосунку, оскільки дозволяє уникнути завчасного завантаження великих обсягів даних, які можуть бути непотрібні в конкретному контексті.

---

## **1. Основні стратегії завантаження даних**

В **Entity Framework Core** (EF Core) та **Entity Framework 6 (EF6)** є кілька підходів до завантаження зв'язаних даних:

1. **Eager Loading** – Завантаження зв’язаних даних відразу за допомогою `Include()`.
2. **Explicit Loading** – Завантаження даних вручну через методи `Load()` або `Entry().Collection()`.
3. **Lazy Loading** – Автоматичне завантаження даних тільки при першому доступі до навігаційної властивості.

---

## **2. Як працює Lazy Loading?**

Lazy Loading дозволяє відтермінувати завантаження навігаційних властивостей (зв'язаних даних) до моменту першого звернення до них. Це означає, що при отриманні об’єкта його навігаційні властивості спочатку залишаються `null` або не ініціалізованими. Але коли програма звертається до них, Entity Framework робить окремий запит до бази даних і отримує необхідні дані.

---

## **3. Реалізація Lazy Loading в EF Core**

Lazy Loading в **EF Core** можна реалізувати двома способами:

### **3.1 Використання проксі-об’єктів (Proxy-based Lazy Loading)**

Для цього потрібно:

- Встановити навігаційні властивості як `virtual`.
- Використовувати спеціальні проксі-об’єкти.
- Додати залежність `Microsoft.EntityFrameworkCore.Proxies` і викликати `UseLazyLoadingProxies()`.

#### **Приклад налаштування Lazy Loading через проксі**

```csharp
public class Student
{
    public int Id { get; set; }
    public string Name { get; set; }

    // Віртуальна навігаційна властивість для Lazy Loading
    public virtual ICollection<Course> Courses { get; set; }
}

public class Course
{
    public int Id { get; set; }
    public string Title { get; set; }
    
    public virtual ICollection<Student> Students { get; set; }
}

public class AppDbContext : DbContext
{
    public DbSet<Student> Students { get; set; }
    public DbSet<Course> Courses { get; set; }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.UseSqlServer("YourConnectionString")
                      .UseLazyLoadingProxies();
    }
}
```

> **Як це працює?**
> 
> - EF Core створює підкласи об'єктів і перевизначає `virtual` властивості, щоб при їх зверненні робити запит до бази.

#### **Приклад доступу до даних**

```csharp
using (var context = new AppDbContext())
{
    var student = context.Students.FirstOrDefault(s => s.Id == 1);
    var courses = student.Courses; // Окремий SQL-запит виконається тут
}
```

> Тут `student.Courses` викликає додатковий SQL-запит тільки в момент звернення до цієї властивості.

---

### **3.2 Lazy Loading через ін’єкцію `ILazyLoader`**

Починаючи з **EF Core 2.1**, з’явилась можливість реалізувати Lazy Loading без використання проксі-об’єктів. Для цього потрібно:

4. Додати в модель ін'єкцію `ILazyLoader`.
5. Використовувати `_lazyLoader.Load(this, ref navigationProperty)`.

#### **Приклад реалізації без проксі**

```csharp
using Microsoft.EntityFrameworkCore.Infrastructure;

public class Student
{
    private ICollection<Course> _courses;
    private readonly ILazyLoader _lazyLoader;

    public Student() {}

    public Student(ILazyLoader lazyLoader)
    {
        _lazyLoader = lazyLoader;
    }

    public int Id { get; set; }
    public string Name { get; set; }

    public ICollection<Course> Courses
    {
        get => _lazyLoader.Load(this, ref _courses);
        set => _courses = value;
    }
}
```

> **Як це працює?**
> 
> - `ILazyLoader` викликається при зверненні до `Courses` і завантажує дані вручну.

#### **Плюси цього підходу**

- **Гнучкість**: Lazy Loading працює без необхідності `virtual` та проксі-об'єктів.
- **Кращий контроль**: Ви точно знаєте, коли відбувається завантаження даних.

---

## **4. Реалізація Lazy Loading в EF 6**

Для **Entity Framework 6** Lazy Loading працює майже аналогічно, але тут використовується лише підхід через проксі-об’єкти. Потрібно:

- Встановити `virtual` для навігаційних властивостей.
- Включити Lazy Loading у `DbContext`.

```csharp
public class AppDbContext : DbContext
{
    public DbSet<Student> Students { get; set; }
    public DbSet<Course> Courses { get; set; }

    public AppDbContext() : base("YourConnectionString")
    {
        this.Configuration.LazyLoadingEnabled = true;
    }
}
```

> **Lazy Loading працює автоматично**, якщо навігаційні властивості є `virtual`.

---

## **5. Потенційні проблеми та оптимізація**

### **5.1 Проблема N+1 запитів**

Однією з головних проблем Lazy Loading є велика кількість запитів до бази. Наприклад:

```csharp
var students = context.Students.ToList();

foreach (var student in students)
{
    var courses = student.Courses; // Тут буде окремий SQL-запит для кожного студента
}
```

> Якщо у вас 100 студентів, це створить **100 додаткових SQL-запитів**.

### **5.2 Як вирішити проблему N+1?**

6. **Використовувати Eager Loading (`Include()`)**, коли необхідно отримати багато пов'язаних даних:
    
    ```csharp
    var studentsWithCourses = context.Students.Include(s => s.Courses).ToList();
    ```
    
7. **Explicit Loading** – Завантаження зв’язаних даних вручну:
    
    ```csharp
    var student = context.Students.First();
    context.Entry(student).Collection(s => s.Courses).Load();
    ```
    
8. **Фільтрація пов'язаних даних (`Filtered Include`)** – дозволяє включати лише частину зв’язаних даних:
    
    ```csharp
    var students = context.Students.Include(s => s.Courses.Where(c => c.Title.Contains("Math"))).ToList();
    ```
    

---

## **6. Коли використовувати Lazy Loading?**

### ✅ **Lazy Loading варто використовувати:**

- Коли зв'язані дані **не завжди потрібні**.
- Коли працюєте з об'єктами **невеликого розміру**.
- Коли важлива **простота коду** і мінімальна логіка завантаження.

### ❌ **Lazy Loading НЕ варто використовувати:**

- Коли важлива **продуктивність** (через проблему N+1).
- Коли потрібно **завантажити багато зв'язаних даних** одразу.
- У **високонавантажених додатках**, де важливо мінімізувати кількість SQL-запитів.

---

## **7. Висновок**

Lazy Loading – потужний інструмент для ефективного керування завантаженням даних у EF, але він має свої обмеження. Використовуйте його обережно, уникаючи проблеми N+1 запитів, і комбінуйте з Eager та Explicit Loading для оптимізації продуктивності.

Хочеш розібрати кейси з реального застосування або приклади SQL-запитів, які генерує Lazy Loading? 😊