### Thread-Safe Колекції в .NET

У багатопотоковому середовищі, коли декілька потоків працюють з одними і тими ж структурами даних, потрібно забезпечити їхню **потокобезпечність (Thread-Safety)**. У .NET для цього передбачені спеціальні колекції, які входять до простору імен `System.Collections.Concurrent`.

Розглянемо детально кожну з них:

---

## 1. `ConcurrentDictionary<TKey, TValue>`

### Опис

`ConcurrentDictionary<TKey, TValue>` – це потокобезпечний аналог `Dictionary<TKey, TValue>`, який дозволяє ефективно працювати з даними без явного використання блокувань (`lock`).

### Основні особливості:

- Використовує **fine-grained locking** (тобто блокує тільки необхідні частини словника, а не весь словник загалом).
- Підтримує **атомарні операції** (`AddOrUpdate`, `TryAdd`, `TryRemove`).
- Ефективніше працює в середовищі, де часто змінюються дані.

### Приклад використання:

```csharp
var dict = new ConcurrentDictionary<string, int>();

// Додавання нового елемента
dict.TryAdd("item1", 1);

// Оновлення або додавання (якщо ключа не існує)
dict.AddOrUpdate("item1", 1, (key, oldValue) => oldValue + 1);

// Отримання значення
if (dict.TryGetValue("item1", out int value))
{
    Console.WriteLine($"item1: {value}");
}

// Видалення
dict.TryRemove("item1", out _);
```

---

## 2. `ConcurrentQueue<T>`

### Опис

`ConcurrentQueue<T>` – це потокобезпечна черга, яка працює за принципом **FIFO (First-In, First-Out)**.

### Основні особливості:

- Використовує **lock-free алгоритм**, що забезпечує швидку роботу.
- Чудово підходить для **черг обробки задач** у багатопотокових середовищах.
- Відсутня підтримка індексації або довільного доступу до елементів.

### Приклад використання:

```csharp
var queue = new ConcurrentQueue<int>();

// Додавання елементів
queue.Enqueue(1);
queue.Enqueue(2);

// Вилучення елемента (без блокування)
if (queue.TryDequeue(out int result))
{
    Console.WriteLine($"Dequeued: {result}");
}

// Перегляд першого елемента без видалення
if (queue.TryPeek(out int peekResult))
{
    Console.WriteLine($"First element: {peekResult}");
}
```

---

## 3. `ConcurrentStack<T>`

### Опис

`ConcurrentStack<T>` – потокобезпечний **стек**, який працює за принципом **LIFO (Last-In, First-Out)**.

### Основні особливості:

- Використовує **lock-free алгоритм** для швидкої роботи.
- Хороший вибір для **глибокої рекурсії** та **відкату операцій**.
- Не дозволяє доступ до довільних елементів.

### Приклад використання:

```csharp
var stack = new ConcurrentStack<int>();

// Додавання елементів
stack.Push(1);
stack.Push(2);

// Вилучення елемента (останнього доданого)
if (stack.TryPop(out int poppedValue))
{
    Console.WriteLine($"Popped: {poppedValue}");
}

// Перегляд останнього елемента без видалення
if (stack.TryPeek(out int peekedValue))
{
    Console.WriteLine($"Top element: {peekedValue}");
}
```

---

## 4. `ConcurrentBag<T>`

### Опис

`ConcurrentBag<T>` – це потокобезпечна **неупорядкована колекція**, оптимізована для сценаріїв, коли один і той же потік як додає, так і зчитує дані.

### Основні особливості:

- Використовує **thread-local storage**, що робить її ефективною, коли кожен потік взаємодіє в основному зі своїми елементами.
- Не гарантує порядок отримання елементів.
- Підходить для **пулів ресурсів** (наприклад, пулів підключень).

### Приклад використання:

```csharp
var bag = new ConcurrentBag<int>();

// Додавання елементів
bag.Add(1);
bag.Add(2);

// Вилучення елемента
if (bag.TryTake(out int result))
{
    Console.WriteLine($"Taken: {result}");
}

// Перевірка наявності елемента (без видалення)
if (bag.TryPeek(out int peekResult))
{
    Console.WriteLine($"Peeked: {peekResult}");
}
```

---

## 5. `BlockingCollection<T>`

### Опис

`BlockingCollection<T>` – це **обгортка** для `IProducerConsumerCollection<T>`, яка дозволяє **контролювати швидкість потоків виробників і споживачів**.

### Основні особливості:

- Може використовувати `ConcurrentQueue<T>`, `ConcurrentStack<T>` або `ConcurrentBag<T>` як основу.
- Дозволяє встановлювати **максимальний розмір колекції** (щоб обмежити швидкість додавання).
- Використовується для **парадигми Producer-Consumer**.

### Приклад використання:

```csharp
var blockingCollection = new BlockingCollection<int>(new ConcurrentQueue<int>(), 5);

// Додавання елементів (блокує, якщо колекція заповнена)
blockingCollection.Add(1);
blockingCollection.Add(2);

// Вилучення елементів (блокує, якщо колекція порожня)
int item = blockingCollection.Take();
Console.WriteLine($"Taken: {item}");
```

---

## Висновок: яку колекцію обрати?

|Колекція|Коли використовувати|
|---|---|
|**ConcurrentDictionary<TKey, TValue>**|Коли потрібно потокобезпечне зберігання пар "ключ-значення"|
|**ConcurrentQueue**|Для обробки задач у черзі (FIFO)|
|**ConcurrentStack**|Для сценаріїв зворотного відкату (LIFO)|
|**ConcurrentBag**|Коли порядок неважливий, і потоки часто взаємодіють зі своїми даними|
|**BlockingCollection**|Для реалізації патерну "виробник-споживач"|

---

### Додаткові Поради:

- Використовуйте `ConcurrentBag<T>` тільки якщо один і той же потік частіше додає і забирає свої дані.
- `BlockingCollection<T>` підходить для **регулювання потоків** (обмеження швидкості виконання завдань).
- `ConcurrentQueue<T>` і `ConcurrentStack<T>` кращі за `BlockingCollection<T>`, якщо немає потреби в блокуванні потоків.

Якщо у тебе є конкретний кейс використання, можу допомогти вибрати оптимальну структуру або оптимізувати код. 😊