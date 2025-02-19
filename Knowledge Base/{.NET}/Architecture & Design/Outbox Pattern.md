### **Що таке `OutboxMessages`?**

`OutboxMessages` — це **таблиця в базі даних**, яка використовується в **Outbox Pattern** для збереження подій перед їх передачею в **системи обміну повідомленнями** (Kafka, RabbitMQ, Azure Service Bus тощо).

Ця таблиця виконує роль **буферу (transient storage)** для повідомлень, що дозволяє:

1. **Записати бізнес-дані і подію в одній транзакції**.
2. **Гарантувати, що подія не буде втрачена** у разі збою черги повідомлень.
3. **Відправляти події асинхронно через фоновий процес**.
4. **Забезпечити повторне відправлення**, якщо перша спроба не вдалася.

---

## **Структура таблиці `OutboxMessages`**

Вона містить такі основні поля:

```csharp
public class OutboxMessage
{
    public Guid Id { get; set; } = Guid.NewGuid(); // Унікальний ідентифікатор події
    public string EventType { get; set; } = null!; // Тип події (наприклад, OrderCreated, UserRegistered)
    public string Payload { get; set; } = null!;   // JSON-дані події
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow; // Час створення події
    public bool Processed { get; set; } = false;   // Чи була подія оброблена
    public DateTime? ProcessedAt { get; set; }     // Час обробки події (коли вона була відправлена)
}
```

📌 **Основні аспекти:**

- **`Id`** – Унікальний ідентифікатор події.
- **`EventType`** – Тип події, який визначає, в яку чергу її слід відправити (`OrderCreated`, `UserUpdated` тощо).
- **`Payload`** – JSON-об'єкт із фактичними даними події.
- **`CreatedAt`** – Час створення події.
- **`Processed`** – Флаг, що вказує, чи була подія вже відправлена у чергу повідомлень.
- **`ProcessedAt`** – Час, коли подія була відправлена.

---

## **Як `OutboxMessages` працює у транзакціях?**

Приклад **створення замовлення (`Order`)** та збереження події в `OutboxMessages`:

```csharp
using (var context = new ApplicationDbContext())
{
    using (var transaction = context.Database.BeginTransaction())
    {
        try
        {
            // 1. Створюємо нове замовлення
            var order = new Order { CustomerId = 1, TotalAmount = 300 };
            context.Orders.Add(order);
            context.SaveChanges();

            // 2. Створюємо подію і додаємо в OutboxMessages
            var outboxMessage = new OutboxMessage
            {
                EventType = "OrderCreated",
                Payload = JsonConvert.SerializeObject(order) // Зберігаємо об'єкт у форматі JSON
            };
            context.OutboxMessages.Add(outboxMessage);
            context.SaveChanges();

            // 3. Комітимо транзакцію (Order і подія записані разом)
            transaction.Commit();
        }
        catch
        {
            transaction.Rollback(); // Відкат усіх змін, якщо сталася помилка
        }
    }
}
```

✅ **Гарантія узгодженості:** замовлення (`Order`) та подія (`OutboxMessages`) зберігаються разом.  
❌ **Якщо запис у RabbitMQ зробити тут і він зафейлиться** – подія може загубитися.

---

## **Обробка `OutboxMessages` у фоні (Background Worker)**

Щоб подія була відправлена, створимо **Background Service**, який періодично перевіряє `OutboxMessages` і публікує повідомлення.

```csharp
public class OutboxProcessor : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly IMessageBus _messageBus;
    private readonly ILogger<OutboxProcessor> _logger;

    public OutboxProcessor(IServiceScopeFactory scopeFactory, IMessageBus messageBus, ILogger<OutboxProcessor> logger)
    {
        _scopeFactory = scopeFactory;
        _messageBus = messageBus;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                using (var scope = _scopeFactory.CreateScope())
                {
                    var context = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();

                    var messages = await context.OutboxMessages
                        .Where(x => !x.Processed)
                        .ToListAsync(stoppingToken);

                    foreach (var message in messages)
                    {
                        try
                        {
                            // Відправляємо подію у RabbitMQ / Kafka
                            await _messageBus.PublishAsync(message.EventType, message.Payload);

                            // Позначаємо як оброблену
                            message.Processed = true;
                            message.ProcessedAt = DateTime.UtcNow;

                            await context.SaveChangesAsync(stoppingToken);
                        }
                        catch (Exception ex)
                        {
                            _logger.LogError(ex, "Помилка відправки події {MessageId}", message.Id);
                        }
                    }
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Помилка у Outbox Processor");
            }

            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }
}
```

---

## **Чому `OutboxMessages` корисний?**

### **🔹 1. Захист від втрати повідомлень**

Якщо RabbitMQ / Kafka **недоступний**, подія **не загубиться**, а залишиться в таблиці `OutboxMessages` до наступної успішної спроби.

### **🔹 2. Гарантія узгодженості даних**

Якщо записати бізнес-дані (`Order`) і подію окремо, можливий стан, коли:

- `Order` записано, а `OrderCreated` – **ні** (система впала).
- `OrderCreated` відправлено, а `Order` – **ні** (база впала).

`OutboxMessages` усуває цю проблему за рахунок **однієї транзакції**.

### **🔹 3. Повторна обробка**

Якщо повідомлення не було успішно відправлено в RabbitMQ, воно **не буде видалене** з `OutboxMessages`, і фоновий процес спробує його повторно.

### **🔹 4. Масштабованість**

Можна використовувати **кілька інстансів сервісів**, які споживають `OutboxMessages`, дозволяючи розподілити навантаження.

---

## **Висновок**

`OutboxMessages` — це **таблиця-буфер**, яка дозволяє гарантувати **консистентність** між записами в БД і подіями в асинхронних чергах.  
Використання `Outbox Pattern` в **.NET + EF Core** дозволяє: ✅ Виконувати **бізнес-операції та публікацію подій** **в одній транзакції**.  
✅ Запобігати **втраті подій** у разі падіння черги повідомлень.  
✅ Гарантувати **повторне відправлення** подій при збоях.

Цей патерн є **обов’язковим у мікросервісах** 🚀, де потрібна **узгодженість між базою даних та подієвою системою (Event-Driven Architecture)**.