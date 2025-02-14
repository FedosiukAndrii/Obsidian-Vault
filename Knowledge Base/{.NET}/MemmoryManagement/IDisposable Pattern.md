
---

## 1. Загальна ідея патерну `IDisposable`

Інтерфейс `IDisposable` слугує для передбачуваного та детермінованого звільнення ресурсів. Під “ресурсами” здебільшого маються на увазі:

1. **Unmanaged ресурси**: дескриптори файлів, сокетів, з’єднань, незакритих хендлів тощо. Зазвичай це обгортки над системними ресурсами (Win32 API, GDI+ тощо).
2. **Керовані об’єкти, що реалізують `IDisposable`**: наприклад, `Stream`, `SqlConnection`, `HttpClient` тощо, які також потребують явного закриття або звільнення.

Ключова перевага патерну `IDisposable` над GC (Garbage Collector) у .NET полягає в тому, що GC не гарантує “негайного” прибирання об’єктів, а виконує збирання сміття лише тоді, коли система вирішить, що це доцільно. Використання `IDisposable` дозволяє чітко управляти життєвим циклом важливих ресурсів.

---

## 2. Типові підходи до реалізації

### 2.1. Базова реалізація для керованих ресурсів

Якщо ваш клас володіє виключно **керованими** ресурсами (наприклад, поля типів, що також реалізують `IDisposable`), можна обмежитись простою реалізацією:

```csharp
public class SimpleResourceHolder : IDisposable
{
    private bool _disposed = false;
    private SomeDisposableResource _someResource = new SomeDisposableResource();

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }

    protected virtual void Dispose(bool disposing)
    {
        if (_disposed)
            return;

        if (disposing)
        {
            // Звільняємо керовані ресурси
            _someResource.Dispose();
        }

        // Тут поки що немає unmanaged-логіки

        _disposed = true;
    }
}
```

Зверніть увагу:

- `Dispose(bool disposing)` – **шаблонний метод** від Microsoft, який дозволяє коректно обробляти сценарії, коли викликається фіналізатор (тобто `disposing` буде `false`).
- `GC.SuppressFinalize(this)` викликається лише у випадках, коли є фіналізатор, але часто його викликають навіть і без фіналізатора на майбутнє. Втім, якщо фіналізатора немає, викликати `GC.SuppressFinalize()` не обов’язково.
- `_disposed` запобігає **подвійному** виклику звільнення ресурсів.

### 2.2. Розширена реалізація з власним фіналізатором

Якщо ваш клас використовує **unmanaged ресурси** безпосередньо (тобто не через готові “SafeHandle” чи обгортки), зазвичай вам потрібен ще фіналізатор (`~ClassName`), щоб “підчищати” ресурси навіть у випадку, коли виклик `Dispose()` пропущено. Тоді повна реалізація виглядає так:

```csharp
public class ComplexResourceHolder : IDisposable
{
    // Приклад: поле IntPtr, яке зберігає посилання на unmanaged ресурс
    private IntPtr _unmanagedHandle;
    // Приклад: керований ресурс
    private SomeDisposableResource _managedResource;
    private bool _disposed = false;

    public ComplexResourceHolder()
    {
        // Ініціалізація _unmanagedHandle, наприклад, зверненням до системного API
        // _unmanagedHandle = SomeNativeApi.OpenHandle(...);
        _managedResource = new SomeDisposableResource();
    }

    ~ComplexResourceHolder()
    {
        // Виклик Dispose із disposing = false
        Dispose(false);
    }

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this); 
        // Відключаємо виклик фіналізатора, оскільки ресурси вже звільнено
    }

    protected virtual void Dispose(bool disposing)
    {
        if (_disposed)
            return;

        if (disposing)
        {
            // Звільняємо виключно керовані ресурси
            _managedResource?.Dispose();
        }

        // Звільняємо unmanaged-ресурс
        if (_unmanagedHandle != IntPtr.Zero)
        {
            // SomeNativeApi.CloseHandle(_unmanagedHandle);
            _unmanagedHandle = IntPtr.Zero;
        }

        _disposed = true;
    }
}
```

Зверніть увагу:

1. Фіналізатор (деструктор) викликає `Dispose(false)`, що дозволяє **не** чіпати керовані ресурси, оскільки вони можуть бути уже зібрані або в некоректному стані.
2. Якщо ви **не володієте безпосередньо** unmanaged ресурсом, то фіналізатор зазвичай **не потрібен** (і його потрібно уникати заради кращої продуктивності та меншого тиску на GC).
3. Завжди викликайте `GC.SuppressFinalize(this)` із `Dispose()`, щоб уникнути непотрібного виклику фіналізатора після коректного звільнення.

---

## 3. Використання `SafeHandle` та сучасні підходи

### 3.1. `SafeHandle` та його нащадки

Microsoft рекомендує, коли можливо, використовувати `SafeHandle` (або його нащадки на кшталт `SafeFileHandle`) замість роботи з “сирим” `IntPtr`. `SafeHandle` сам реалізує критичний фіналізатор, що надійно звільняє нативний ресурс. Тоді ваша власна логіка Dispose суттєво спрощується:

```csharp
public class SafeHandleExample : IDisposable
{
    private bool _disposed = false;
    private SafeHandle _safeHandle; // Наприклад, SafeFileHandle, SafeWaitHandle тощо

    // Також керовані ресурси
    private SomeDisposableResource _resource;

    public SafeHandleExample(string filePath)
    {
        // _safeHandle = CreateFileSafeHandle(filePath); // Уявна функція
        _resource = new SomeDisposableResource();
    }

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }

    protected virtual void Dispose(bool disposing)
    {
        if (_disposed)
            return;

        if (disposing)
        {
            // Звільняємо керовані ресурси
            _resource.Dispose();

            // Звільняємо SafeHandle
            _safeHandle?.Dispose();
        }

        // Додатковий unmanaged clean-up зазвичай не потрібен, 
        // оскільки SafeHandle вже робить все сама

        _disposed = true;
    }
}
```

Такий підхід:

- Зводить до мінімуму кількість ручної логіки, позбавляє необхідності писати власний фіналізатор.
- Зменшує ризики memory leak у випадках некоректного коду.
- Покращує безпеку, бо `SafeHandle` використовує критичні фіналізатори та додаткові перевірки.

### 3.2. `IAsyncDisposable` (C# 8+)

У більш нових версіях .NET і C#, поруч із `IDisposable`, з’явився `IAsyncDisposable`, що дозволяє виконувати асинхронну логіку звільнення ресурсів (`DisposeAsync()`). Це зручно при роботі з мережевими стрімами, асинхронними з’єднаннями або іншими асинхронними API, де коректне закриття/флешинг ресурсів вимагає await-операцій:

```csharp
public class AsyncResourceHolder : IAsyncDisposable
{
    private bool _disposed = false;
    private readonly SomeAsyncDisposableResource _resource;

    public AsyncResourceHolder()
    {
        _resource = new SomeAsyncDisposableResource();
    }

    public async ValueTask DisposeAsync()
    {
        await DisposeAsyncCore();

        // Те саме, що й в Dispose-варіанті: SuppressFinalize, якщо є фіналізатор
        GC.SuppressFinalize(this);
    }

    protected virtual async ValueTask DisposeAsyncCore()
    {
        if (!_disposed)
        {
            // Асинхронне звільнення
            if (_resource != null)
                await _resource.DisposeAsync();

            _disposed = true;
        }
    }
}
```

Для використання в коді передбачено ключове слово `await using`:

```csharp
await using var resourceHolder = new AsyncResourceHolder();
// Далі працюємо з ресурсом
```

---

## 4. Патерн “Dispose/Finalize” у спадкових класах (інших розробників)

Якщо клас **успадковується**, дуже важливо надати можливість нащадкам переглядати метод `Dispose(bool disposing)`:

4. `Dispose(bool disposing)` має бути оголошено `protected virtual` (або `protected override` у нащадках).
5. Нащадок викликає `base.Dispose(disposing)` у своєму перевантаженні, щоб забезпечити коректне звільнення базових ресурсів.
6. В жодному разі не можна **змінювати** сигнатуру `public void Dispose()` — це частина контракту `IDisposable`.

При неправильному виклику базового `Dispose(bool)` легко отримати витік ресурсів або дубльовані спроби закривати той самий ресурс.

---

## 5. Поширені помилки та рекомендації

7. **Подвійний виклик `Dispose()`.**  
    Повинен бути безпечним. Об’єкт не повинен “впасти” при другому виклику. Перевірка `_disposed` оберігає від повторних спроб звільнити те саме.
    
8. **Відсутність `Dispose()` для залежних об’єктів, що реалізують `IDisposable`.**  
    Якщо ваш клас має поля типів, які також реалізують `IDisposable`, ви відповідаєте за їх Dispose. Не припускайте, що це зроблять самі поля “якось магічно”.
    
9. **Неправильний виклик у фіналізаторі.**  
    У фіналізаторі **не можна** звертатися до керованих об’єктів напряму, адже вони можуть бути вже зібрані GC. Використовується патерн “`Dispose(false)`”.
    
10. **Надлишковий фіналізатор.**  
    Якщо немає unmanaged ресурсів, немає потреби у власному фіналізаторі. Він створює додаткові накладні витрати при роботі GC.
    
11. **Використання `IDisposable` без “using” (або “await using”).**  
    Хоча формально клас-споживач може викликати `Dispose()` вручну, ідіоматично та безпечніше використовувати `using` / `await using`, що автоматично забезпечить виклик `Dispose()` у кінці блоку.
    

---

## 6. Приклади розширеного використання (Use Cases)

12. **Робота з файловими потоками**:  
    Коли відкриваєте файли (`FileStream` чи `StreamReader`), обов’язково загортайте у `using`. Якщо створюєте власний клас, який управляє кількома потоками, _всі_ внутрішні потоки потрібно коректно Dispose.
    
13. **Робота з SQL-з’єднаннями**:  
    `SqlConnection`, `SqlCommand`, `SqlDataReader` – усі підтримують `IDisposable`. Ваш клас, що володіє цими об’єктами, так само має імплементувати `IDisposable`. Це дозволить уникнути блокувань, вичерпання пулу з’єднань тощо.
    
14. **Захоплення нативних ресурсів (WinAPI, хендли тощо)**:  
    Коли маєте справу з `IntPtr` (через P/Invoke), краще розглянути `SafeHandle`. Якщо неможливо, ви зобов’язані самі грамотно прописати логіку звільнення та (за потреби) фіналізатор.
    
15. **Асинхронні сценарії**:  
    Якщо ваш ресурс повинен виконати асинхронні виклики при звільненні (наприклад, дописати дані у мережу, закрити асинхронний потік), використовуйте `IAsyncDisposable` й `DisposeAsync()`.
    

---

## 7. Підсумок

Реалізація `IDisposable` – важливий патерн для керованого звільнення ресурсів, котрий безпосередньо впливає на продуктивність і надійність додатків .NET. На старшому рівні (advanced) слід звертати увагу на такі моменти:

- **Чітка архітектура**: розмежовуйте керовані та некеровані ресурси і пам’ятайте про фази звільнення (`Dispose(bool disposing)`).
- **Уникнення фіналізаторів**, якщо дійсно немає unmanaged-ресурсів. Це збільшує пропускну здатність GC і зменшує затримки.
- **Безпечність до повторних викликів** `Dispose()`.
- **Наслідування**: дозволяйте нащадкам правильно перевантажувати методи, викликаючи базовий `Dispose(bool)`.
- **Рекомендації від Microsoft**: уникайте прямих `IntPtr`, частіше застосовуйте `SafeHandle`.
- **Сучасні можливості**: у випадках асинхронного звільнення використовуйте `IAsyncDisposable`.

Дотримання цих практик зробить ваш код надійнішим і більш прогнозованим, а також позитивно вплине на продуктивність та масштабованість .NET-рішень.