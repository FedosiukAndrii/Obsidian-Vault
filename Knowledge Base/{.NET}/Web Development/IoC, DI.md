### **IoC Container та Життєвий Цикл Об'єктів на Рівні Advanced**

**Inversion of Control (IoC) Container** – це механізм для управління залежностями в додатках, що дозволяє відділити створення об'єктів від їх використання, забезпечуючи Dependency Injection (DI).

#### **1. Основні концепції IoC Container**

1. **Реєстрація залежностей**  
    Контейнер має можливість зберігати інформацію про те, як створювати та надавати об'єкти.
    
2. **Резолюція залежностей**  
    При створенні об'єкта контейнер автоматично впроваджує всі його залежності.
    
3. **Життєвий цикл об'єктів**  
    Контейнер управляє життєвим циклом об'єктів, визначаючи, коли вони створюються і знищуються.
    

---

## **Життєвий цикл об'єктів у IoC контейнері**

Життєвий цикл об'єктів (Lifecycle) визначає, як довго екземпляр об'єкта існує в пам'яті.

### **1. Scoped Lifecycle Models**

#### **1.1. Transient**

🔹 **Опис**: Новий екземпляр класу створюється щоразу, коли він запитується.  
🔹 **Переваги**:

- Відсутність shared state.
- Мінімізує ризик race condition в багатопотокових середовищах.
- Оптимальний варіант для lightweight об'єктів.  
    🔹 **Недоліки**:
- Високі витрати на створення об'єктів при частих запитах.  
    🔹 **Приклад**:

```csharp
services.AddTransient<IMyService, MyService>();
```

🔹 **Коли використовувати**:

- Сервіси без стану (stateless).
- Легкі, швидкостворювані об'єкти.

#### **1.2. Scoped**

🔹 **Опис**: Один екземпляр створюється на кожен HTTP-запит (у веб-застосунках).  
🔹 **Переваги**:

- Використання спільного екземпляра всередині одного запиту.
- Мінімізація витрат у порівнянні з Transient.  
    🔹 **Недоліки**:
- Не підходить для фонових сервісів або коду, що не виконується у межах HTTP-запиту.  
    🔹 **Приклад**:

```csharp
services.AddScoped<IMyService, MyService>();
```

🔹 **Коли використовувати**:

- Сервіси, які мають стан, що залежить від запиту (наприклад, робота з `DbContext` у EF Core).
- Взаємодія з кешами, що оновлюються між запитами.

#### **1.3. Singleton**

🔹 **Опис**: Один екземпляр класу створюється на весь життєвий цикл додатку.  
🔹 **Переваги**:

- Зменшення навантаження на пам’ять (повторне використання об'єкта).
- Корисно для конфігураційних сервісів, кешу, логування.  
    🔹 **Недоліки**:
- Можливі проблеми з потокобезпечністю.
- Дані можуть змінюватися несподівано, що може призвести до стану гонки (race condition).  
    🔹 **Приклад**:

```csharp
services.AddSingleton<IMyService, MyService>();
```

🔹 **Коли використовувати**:

- Кешовані сервіси (MemoryCache, Configuration).
- Логери, клієнти до бази даних (наприклад, `HttpClientFactory`).

---

### **2. Контроль Життєвого Циклу через Factory Methods**

Іноді необхідно точніше контролювати життєвий цикл залежностей, використовуючи фабричні методи.

#### **2.1. Використання фабричного методу для налаштування об'єкта**

```csharp
services.AddScoped<IMyService>(serviceProvider =>
{
    var config = serviceProvider.GetRequiredService<IConfiguration>();
    return new MyService(config["SomeKey"]);
});
```

🔹 **Коли використовувати?**

- Коли потрібна додаткова логіка для створення екземпляра.

---

### **3. Контроль життєвого циклу через `IDisposable`**

Контейнер відповідає за виклик `Dispose` у об'єктів, які реалізують `IDisposable`.

#### **3.1. Правильне використання `IDisposable` у DI**

```csharp
public class MyService : IDisposable
{
    public void Dispose()
    {
        // Очистка ресурсів
    }
}
```

Якщо сервіс зареєстрований як `Scoped` або `Singleton`, то контейнер правильно викличе `Dispose` після закінчення його життєвого циклу.

> **⚠️ Важливо:** Для `Transient` об'єктів контейнер **не відповідає** за виклик `Dispose`, оскільки вони можуть бути створені поза контекстом IoC.

---

## **4. Глибокий аналіз залежностей та циклів**

Залежності можуть утворювати цикли, які ускладнюють роботу контейнера.

### **4.1. Циклічні залежності**

Якщо два сервіси залежать один від одного, контейнер не зможе правильно їх створити.

#### **4.1.1. Неправильний приклад циклічної залежності**

```csharp
public class A
{
    public A(B b) { }
}

public class B
{
    public B(A a) { }
}
```

🔹 **Рішення**:

4. Використовувати `Lazy<T>`:

```csharp
public class A
{
    private readonly Lazy<B> _b;
    public A(Lazy<B> b) { _b = b; }
}
```

5. Використати фабрику (`Func<T>`) для створення залежності на вимогу:

```csharp
public class A
{
    private readonly Func<B> _bFactory;
    public A(Func<B> bFactory) { _bFactory = bFactory; }
}
```

6. Винести залежність в інший рівень абстракції.

---

## **5. Використання `IServiceScopeFactory` для створення Scoped залежностей в Singleton**

У випадку, коли Singleton повинен отримувати Scoped залежності, можна використовувати `IServiceScopeFactory`.

```csharp
public class MySingletonService
{
    private readonly IServiceScopeFactory _serviceScopeFactory;

    public MySingletonService(IServiceScopeFactory serviceScopeFactory)
    {
        _serviceScopeFactory = serviceScopeFactory;
    }

    public void DoWork()
    {
        using (var scope = _serviceScopeFactory.CreateScope())
        {
            var scopedService = scope.ServiceProvider.GetRequiredService<IMyScopedService>();
            scopedService.Process();
        }
    }
}
```

🔹 **Це дозволяє Singleton-сервісу безпечно створювати Scoped-залежності.**

---

## **Висновки**

- IoC Container значно спрощує управління залежностями та життєвим циклом об'єктів.
- Важливо вибирати правильний життєвий цикл (`Transient`, `Scoped`, `Singleton`) для ефективного використання ресурсів.
- Потрібно уникати циклічних залежностей або вирішувати їх через `Lazy<T>`, `Func<T>` або фабричні методи.
- `IServiceScopeFactory` дозволяє використовувати Scoped сервіси всередині Singleton-об'єктів.

Знання цих механізмів дозволить вам розробляти високопродуктивні, масштабовані та підтримувані .NET-додатки. 🚀

---

## **Dependency Injection (DI) на рівні Advanced у .NET**

### **1. Вступ до Dependency Injection**

**Dependency Injection (DI)** є ключовим патерном проектування у **.NET**, що дозволяє передавати залежності через конструктор, метод або властивість, зменшуючи зв'язність між компонентами та покращуючи тестованість і масштабованість коду.

### **2. Типи Ін'єкції Залежностей**

#### **2.1. Constructor Injection (Основний підхід)**

Цей метод використовується найчастіше, оскільки гарантує отримання всіх необхідних залежностей під час створення об'єкта.

🔹 **Приклад:**

```csharp
public class OrderService : IOrderService
{
    private readonly IPaymentService _paymentService;

    public OrderService(IPaymentService paymentService)
    {
        _paymentService = paymentService ?? throw new ArgumentNullException(nameof(paymentService));
    }
}
```

🔹 **Коли використовувати?**

- Коли залежність є **обов’язковою** для коректної роботи сервісу.
- Коли кількість залежностей **обмежена** (рекомендується не більше 5).

---

#### **2.2. Property Injection (Setter Injection)**

Використовується у випадках, коли залежність є **необов’язковою** або коли її неможливо передати через конструктор.

🔹 **Приклад:**

```csharp
public class ReportGenerator
{
    public ILogger Logger { get; set; }

    public void Generate()
    {
        Logger?.Log("Генеруємо звіт...");
    }
}
```

🔹 **Коли використовувати?**

- Якщо залежність є **опціональною**.
- Коли залежність може змінюватися після створення об’єкта.

---

#### **2.3. Method Injection**

Передача залежностей як параметрів методу.

🔹 **Приклад:**

```csharp
public class EmailNotifier
{
    public void SendNotification(string message, IEmailService emailService)
    {
        emailService.Send(message);
    }
}
```

🔹 **Коли використовувати?**

- Коли залежність використовується **лише в одному методі**.
- Коли потрібно **мінімізувати залежності в класі**.

---

### **3. Життєві Цикли та DI**

IoC-контейнер керує життєвим циклом залежностей через **Scoped**, **Singleton** та **Transient**, що дозволяє контролювати створення та використання екземплярів.

🔹 **Типові сценарії:**

|Життєвий цикл|Коли використовувати?|
|---|---|
|`Transient`|Легковагі сервіси, без збереження стану (stateless).|
|`Scoped`|Сервіси, що залежать від поточного HTTP-запиту.|
|`Singleton`|Кешовані або глобальні сервіси, що існують протягом усього часу життя додатка.|

---

### **4. Управління Динамічними Залежностями**

Іноді залежності не можуть бути визначені під час компіляції. У таких випадках використовується **Factory Pattern**, `Func<T>` або `Lazy<T>`.

#### **4.1. Використання Factory Pattern**

🔹 **Приклад:**

```csharp
public interface IServiceFactory
{
    IReportService CreateService(string reportType);
}

public class ServiceFactory : IServiceFactory
{
    private readonly IServiceProvider _serviceProvider;

    public ServiceFactory(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public IReportService CreateService(string reportType)
    {
        return reportType switch
        {
            "PDF" => _serviceProvider.GetRequiredService<PdfReportService>(),
            "Excel" => _serviceProvider.GetRequiredService<ExcelReportService>(),
            _ => throw new ArgumentException("Unknown report type")
        };
    }
}
```

🔹 **Коли використовувати?**

- Якщо створювані сервіси залежать від **динамічних параметрів**.
- Якщо потрібно **інкапсулювати логіку створення залежностей**.

---

#### **4.2. Використання `Func<T>` для створення об’єктів на вимогу**

```csharp
public class OrderProcessor
{
    private readonly Func<IDiscountService> _discountServiceFactory;

    public OrderProcessor(Func<IDiscountService> discountServiceFactory)
    {
        _discountServiceFactory = discountServiceFactory;
    }

    public void ProcessOrder()
    {
        var discountService = _discountServiceFactory();
        discountService.ApplyDiscount();
    }
}
```

🔹 **Коли використовувати?**

- Коли створення залежності є **витратним** і потрібно створювати її **лише при потребі**.

---

#### **4.3. Використання `Lazy<T>` для відкладеної ініціалізації**

```csharp
public class HeavyService
{
    public HeavyService() { Console.WriteLine("HeavyService створено!"); }
}

public class Consumer
{
    private readonly Lazy<HeavyService> _heavyService;

    public Consumer(Lazy<HeavyService> heavyService)
    {
        _heavyService = heavyService;
    }

    public void UseService()
    {
        var service = _heavyService.Value; // Створюється лише тут
        Console.WriteLine("Використання HeavyService...");
    }
}
```

🔹 **Коли використовувати?**

- Коли залежність **дорого створювати**, і вона **може не знадобитися**.

---

### **5. Впровадження DI у Middleware та Background Services**

#### **5.1. Використання DI у Middleware**

🔹 **Приклад власного Middleware з DI:**

```csharp
public class CustomMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger _logger;

    public CustomMiddleware(RequestDelegate next, ILogger<CustomMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task Invoke(HttpContext context)
    {
        _logger.LogInformation("Запит обробляється Middleware...");
        await _next(context);
    }
}
```

🔹 **Реєстрація в `Startup.cs`:**

```csharp
app.UseMiddleware<CustomMiddleware>();
```

---

#### **5.2. Використання DI у Hosted Services**

🔹 **Приклад Background Service з DI:**

```csharp
public class WorkerService : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public WorkerService(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var scope = _scopeFactory.CreateScope();
        var myService = scope.ServiceProvider.GetRequiredService<IMyScopedService>();

        while (!stoppingToken.IsCancellationRequested)
        {
            myService.DoWork();
            await Task.Delay(1000, stoppingToken);
        }
    }
}
```

🔹 **Реєстрація в `Program.cs`:**

```csharp
services.AddHostedService<WorkerService>();
```

---

### **6. Вирішення Потенційних Проблем**

#### **6.1. Циклічні залежності**

🔹 **Якщо клас A залежить від класу B, а B – від A:**

```csharp
public class A { public A(B b) { } }
public class B { public B(A a) { } }
```

🔹 **Рішення:**

1. **Рефакторинг через третій клас (наприклад, Service Locator).**
2. **Lazy або Func для відкладеного створення.**
3. **Factory Pattern для явного створення об'єктів.**

---

### **7. Висновки**

- **Constructor Injection** – основний і рекомендований підхід.
- **Factory Pattern, Func і Lazy** дозволяють керувати створенням залежностей **на вимогу**.
- **Scoped сервіси не можна напряму використовувати у Singleton** – слід застосовувати `IServiceScopeFactory`.
- **DI в Middleware та Background Services** потребує окремих підходів.
- **Уникнення циклічних залежностей** покращує гнучкість та стабільність системи.

Це знання дає змогу будувати **гнучкі, масштабовані та тестовані** .NET-додатки! 🚀