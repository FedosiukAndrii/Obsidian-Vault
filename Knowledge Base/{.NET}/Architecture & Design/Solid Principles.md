
---

## 1. Single Responsibility Principle (SRP)

**Принцип єдиної відповідальності** вимагає, щоб кожен клас чи модуль мав одну чітку зону відповідальності. Якщо в класі з’являється більше ніж одна причина для зміни, це є сигналом, що порушується SRP.

### Приклад (C# .NET)

**Поганий приклад** (порушення SRP):

```csharp
public class OrderService
{
    public void CreateOrder(Order order)
    {
        // 1. Зберігаємо замовлення у БД
        SaveToDatabase(order);

        // 2. Відправляємо email-підтвердження
        SendConfirmationEmail(order);

        // 3. Генеруємо PDF-рахунок
        GeneratePdfInvoice(order);
    }

    private void SaveToDatabase(Order order) { /* ... */ }
    private void SendConfirmationEmail(Order order) { /* ... */ }
    private void GeneratePdfInvoice(Order order) { /* ... */ }
}
```

У цьому класі `OrderService` виконується щонайменше три різні обов’язки:

1. Збереження даних у базі.
2. Надсилання email-повідомлень.
3. Генерація PDF.

Таким чином, якщо зміниться логіка відправлення електронних листів або формат PDF, нам доведеться змінювати той самий клас `OrderService`. Це порушує SRP, бо цей клас має надто багато причин для зміни.

**Покращений приклад** (дотримання SRP):

```csharp
public class OrderRepository
{
    public void Save(Order order)
    {
        // Збереження замовлення у БД
    }
}

public class EmailService
{
    public void SendOrderConfirmation(Order order)
    {
        // Відправляємо підтвердження
    }
}

public class PdfInvoiceGenerator
{
    public byte[] Generate(Order order)
    {
        // Генерація PDF
    }
}

public class OrderService
{
    private readonly OrderRepository _orderRepository;
    private readonly EmailService _emailService;
    private readonly PdfInvoiceGenerator _pdfGenerator;

    public OrderService(
        OrderRepository orderRepository,
        EmailService emailService,
        PdfInvoiceGenerator pdfGenerator)
    {
        _orderRepository = orderRepository;
        _emailService = emailService;
        _pdfGenerator = pdfGenerator;
    }

    public void CreateOrder(Order order)
    {
        _orderRepository.Save(order);
        _emailService.SendOrderConfirmation(order);
        var pdfInvoice = _pdfGenerator.Generate(order);

        // Можливо, зробити щось із PDF (зберегти тощо)
    }
}
```

Тепер кожен клас відповідає за свою єдину зону відповідальності: репозиторій зберігає дані, сервіс відправляє email, генератор створює PDF. Кожен із цих класів можна змінювати незалежно, не зачіпаючи інші.

---

## 2. Open/Closed Principle (OCP)

**Принцип відкритості/закритості** каже, що програмні сутності мають бути відкритими для розширення, але закритими для модифікації. Іншими словами, ми повинні мати можливість додавати нову функціональність, не змінюючи вже наявний код.

### Приклад (C# .NET)

Припустімо, що у нас є клас, який обраховує комісію за платежами:

**Порушення OCP**:

```csharp
public class PaymentCalculator
{
    public decimal CalculateFee(string paymentType, decimal amount)
    {
        switch (paymentType)
        {
            case "CreditCard":
                return amount * 0.03m;
            case "PayPal":
                return amount * 0.05m;
            default:
                throw new NotSupportedException("Unknown payment type");
        }
    }
}
```

Кожного разу, коли нам потрібно додати новий тип платежу, доведеться змінювати метод `CalculateFee`, що порушує принцип відкритості/закритості.

**Покращена версія** може використовувати **поліморфізм** і **фабричний метод** (або залежності через інтерфейси/DI-контейнер):

```csharp
public interface IPaymentFeeCalculator
{
    decimal CalculateFee(decimal amount);
}

public class CreditCardPaymentCalculator : IPaymentFeeCalculator
{
    public decimal CalculateFee(decimal amount) => amount * 0.03m;
}

public class PayPalPaymentCalculator : IPaymentFeeCalculator
{
    public decimal CalculateFee(decimal amount) => amount * 0.05m;
}

public class PaymentCalculatorFactory
{
    public static IPaymentFeeCalculator CreateCalculator(string paymentType)
    {
        return paymentType switch
        {
            "CreditCard" => new CreditCardPaymentCalculator(),
            "PayPal" => new PayPalPaymentCalculator(),
            _ => throw new NotSupportedException("Unknown payment type")
        };
    }
}

public class PaymentService
{
    public decimal CalculateFee(string paymentType, decimal amount)
    {
        var calculator = PaymentCalculatorFactory.CreateCalculator(paymentType);
        return calculator.CalculateFee(amount);
    }
}
```

Тепер для додавання нового платіжного методу достатньо:

4. Створити новий клас, що реалізує `IPaymentFeeCalculator`.
5. Додати цей клас до фабрики (або налаштувати це через IoC-контейнер).

Якщо скористатися контейнером залежностей, можна буде навіть уникнути зміни фабричного методу, **повністю винісши логіку створення** у конфігурацію DI.

---

## 3. Liskov Substitution Principle (LSP)

**Принцип підстановки Барбари Лісков** стверджує, що об’єкти підкласів мають бути взаємозамінними з об’єктами своїх батьків. Тобто, якщо клас B наслідується від класу A, то всюди, де очікується A, можна підставити B без порушення коректності програми.

### Приклад (C# .NET)

Уявімо, що у нас є базовий клас для відправлення повідомлень та підклас для SMS:

**Порушення LSP**:

```csharp
public class MessageSender
{
    public virtual void Send(string message)
    {
        // Відправлення будь-яким універсальним способом
    }
}

public class SmsSender : MessageSender
{
    public override void Send(string message)
    {
        if (message.Length > 160)
        {
            throw new ArgumentException("SMS не може бути довшим за 160 символів!");
        }
        // Відправка SMS
    }
}
```

Тут порушення полягає в тому, що базовий клас `MessageSender` не накладав обмежень на довжину повідомлення. При заміні базового класу на підклас `SmsSender` ми отримуємо виняток, якщо повідомлення занадто довге. Тобто підклас вносить додаткову поведінкову умову, яка не прописана в базовому класі.

**Можливе виправлення**:

6. Запровадити в базовий клас або в окремий інтерфейс поняття максимальної довжини повідомлення (або умови, за якими воно відправляється).
7. Або для SMS передбачити механізми поділу повідомлення, якщо це прийнятно для бізнес-логіки.

Наприклад:

```csharp
public abstract class MessageSender
{
    public abstract int MaxLength { get; }
    public virtual void Send(string message)
    {
        if (message.Length > MaxLength)
            throw new ArgumentException($"Перевищено максимально дозволену довжину {MaxLength} символів");

        // Логіка спільна для всіх типів повідомлень
    }
}

public class SmsSender : MessageSender
{
    public override int MaxLength => 160;

    public override void Send(string message)
    {
        base.Send(message);
        // Логіка відправки SMS
    }
}

public class EmailSender : MessageSender
{
    public override int MaxLength => 10000;

    public override void Send(string message)
    {
        base.Send(message);
        // Логіка відправки Email
    }
}
```

Таким чином, кожен підклас відповідає очікуванням, визначеним у батьківському класі. Тепер використання `SmsSender` там, де очікується `MessageSender`, не призводить до несподіваних винятків (окрім тих, що заявлені контрактом).

---

## 4. Interface Segregation Principle (ISP)

**Принцип розділення інтерфейсів** вимагає, щоб клієнти не були змушені реалізовувати інтерфейси, які вони не використовують. Краще мати декілька вузько спрямованих інтерфейсів, ніж один “важкий” інтерфейс.

### Приклад (C# .NET)

Проблемна ситуація виникає, коли у нас занадто великий інтерфейс:

```csharp
public interface IRepository<T>
{
    void Add(T entity);
    T GetById(int id);
    IEnumerable<T> GetAll();
    void Update(T entity);
    void Delete(int id);

    // Наприклад, ще методи для пагінації, фільтрації тощо
    IEnumerable<T> GetByFilter(Func<T, bool> predicate);
}
```

Якщо якийсь тип репозиторію повинен лише читати дані, то йому все одно доводиться реалізовувати методи `Add`, `Update`, `Delete`, які не потрібні.

**Правильна сегрегація**:

```csharp
public interface IReadableRepository<T>
{
    T GetById(int id);
    IEnumerable<T> GetAll();
    IEnumerable<T> GetByFilter(Func<T, bool> predicate);
}

public interface IWriteableRepository<T>
{
    void Add(T entity);
    void Update(T entity);
    void Delete(int id);
}

// Якщо потрібно, можна поєднати в один
public interface IRepository<T> : IReadableRepository<T>, IWriteableRepository<T>
{
}
```

Тепер, якщо у вас є, наприклад, репозиторій лише для читання:

```csharp
public class ReadOnlyUserRepository : IReadableRepository<User>
{
    public User GetById(int id) { /* ... */ }
    public IEnumerable<User> GetAll() { /* ... */ }
    public IEnumerable<User> GetByFilter(Func<User, bool> predicate) { /* ... */ }
}
```

Він не зобов’язаний реалізовувати методи додавання/видалення/оновлення.

---

## 5. Dependency Inversion Principle (DIP)

**Принцип інверсії залежностей** стверджує, що:

8. Модулі високого рівня не повинні залежати від модулів низького рівня. І ті, й інші повинні залежати від абстракцій.
9. Абстракції не повинні залежати від деталей. Деталі мають залежати від абстракцій.

Залежності у вашому додатку мають будуватися “від деталей до абстракцій”, а не навпаки. Це означає, що “вища” логіка не повинна напряму покладатися на конкретну реалізацію “нижчого” рівня, а мала б використовувати інтерфейси чи абстрактні класи.

### Приклад (C# .NET)

Розглянемо `OrderService`, що напряму залежить від конкретного класу `SqlOrderRepository`:

**Порушення DIP**:

```csharp
public class SqlOrderRepository
{
    public void SaveOrder(Order order)
    {
        // Збереження в SQL
    }
}

public class OrderService
{
    private SqlOrderRepository _orderRepository = new SqlOrderRepository();

    public void CreateOrder(Order order)
    {
        _orderRepository.SaveOrder(order);
    }
}
```

Тепер `OrderService` напряму залежить від `SqlOrderRepository`. Якщо нам потрібно змінити сховище даних (наприклад, замінити на NoSQL або використати REST-API), доведеться модифікувати `OrderService`.

**Дотримання DIP**:

```csharp
public interface IOrderRepository
{
    void SaveOrder(Order order);
}

public class SqlOrderRepository : IOrderRepository
{
    public void SaveOrder(Order order)
    {
        // Збереження в SQL
    }
}

public class NoSqlOrderRepository : IOrderRepository
{
    public void SaveOrder(Order order)
    {
        // Збереження в NoSQL
    }
}

public class OrderService
{
    private readonly IOrderRepository _orderRepository;

    // Інжектуємо IOrderRepository
    public OrderService(IOrderRepository orderRepository)
    {
        _orderRepository = orderRepository;
    }

    public void CreateOrder(Order order)
    {
        _orderRepository.SaveOrder(order);
    }
}
```

Тепер `OrderService` залежить від **інтерфейсу** `IOrderRepository`, а не від конкретної реалізації. Таким чином “вищий” за ієрархією клас відділений від деталей збереження (SQL, NoSQL, тощо). Ця залежність можна легко підмінити (mock) у юніт-тестах, або замінити її іншою реалізацією репозиторію.

У сучасному .NET зазвичай використовують вбудовані механізми Dependency Injection (наприклад, `IServiceCollection`) для реєстрації та впровадження залежностей:

```csharp
// Припустимо, в Program.cs або Startup.cs (в .NET 5 і нижче)
services.AddScoped<IOrderRepository, SqlOrderRepository>();
services.AddScoped<OrderService>();
```

---

## Підсумки та поради

10. **Правильне розділення відповідальностей (SRP)** допомагає уникнути монолітного коду, коли зміна будь-якого аспекту вимагає переписування великої кількості логіки.
11. **Принцип відкритості/закритості (OCP)** особливо актуальний, коли ви очікуєте, що функціональність додаватиметься або варіюватиметься з часом. Використовуйте патерни на кшталт фабричних методів, абстрактних фабрик, стратегії тощо.
12. **Підстановка Лісков (LSP)** не лише про наслідування, а й про коректне визначення контрактів і обмежень у базових класах. Особливо важливо в бібліотеках та фреймворках, які використовують підкласування.
13. **Розділення інтерфейсів (ISP)** допомагає боротися з “багатофункціональними” інтерфейсами, що перевантажують імплементації. Краща модульність та зрозумілість коду.
14. **Інверсія залежностей (DIP)** — одна з найважливіших основ для побудови масштабованих, тестованих застосунків у .NET. Спираючись на контейнери DI, можна легко керувати залежностями.

Застосовуючи SOLID-принципи на практиці, завжди пам’ятайте про **гнучкість**, **тестованість** та **розширюваність** вашого коду. У реальних проектах часто є компроміси, але знання та вміння вибірково застосовувати ці принципи ускладненнями в коді (refactoring overhead) робить вас розробником рівня Senior.