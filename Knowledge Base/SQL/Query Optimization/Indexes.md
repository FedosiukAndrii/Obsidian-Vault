В **SQL Server** індекс, як правило, репрезентується у вигляді структури типу **B-дерева** (B-Tree). Це означає, що під час створення індексу сервіс SQL Server будує спеціалізоване дерево, яке дає змогу швидше знаходити та впорядковувати дані.

### Основні моменти

1. **B-дерево (Balanced Tree)**
    
    - Кореневий вузол (root) містить покажчики на гілки дерев.
    - Внутрішні вузли (intermediate nodes) керують розподілом діапазонів ключів.
    - Листові вузли (leaf nodes) містять:
        - Для **кластеризованого** індексу – безпосередньо самі дані таблиці (рядки).
        - Для **некластеризованого** індексу – ключі (ключові значення) та покажчики (pointer) на відповідні рядки у кластеризованому індексі або ж у «купі» (heap).
2. **Кластеризований індекс (Clustered Index)**
    
    - В одному B-дереві листові рівні зберігають дані таблиці «впорядкованими» за індексованими стовпцями.
    - Фізичне сортування таблиці відбувається за ключовими стовпцями кластеризованого індексу.
    - Можна створити лише **один** кластеризований індекс на таблицю, оскільки дані можуть бути зберігані лише в одному конкретному порядку.
3. **Некластеризований індекс (Nonclustered Index)**
    
    - В листовому рівні зберігаються копії ключових стовпців індексу разом із покажчиками на фактичні рядки.
    - Дані в самій таблиці не відсортовані за цим індексом — вони зберігаються або в кластеризованому індексі (якщо він є), або в «купі».
    - Для прискорення запитів може бути створено декілька некластеризованих індексів на одну й ту ж таблицю.
4. **Інші типи індексів**
    
    - **Columnstore**: Зберігає дані по стовпцях, а не по рядках; використовується здебільшого для аналітичних навантажень.
    - **Filtered Index**: Некластеризований індекс, який будується лише над підмножиною рядків (фільтр за умовою `WHERE`).
    - **Full-text**: Спеціальний тип індексу для пошуку по тексту.

Таким чином, **фізично** індекс у SQL Server є структурою B-дерева з кореневими, проміжними та листовими вузлами. Кластеризований індекс зберігає **дані** безпосередньо у листовому рівні, а некластеризований – **ключі й покажчики** на відповідні дані.

Якщо таблиця має кластеризований індекс, то її рядки фізично зберігаються впорядкованими за стовпцями цього індексу (листовий рівень кластеризованого індексу). Якщо ж кластеризованого індексу немає, то рядки розташовані «як є» без явного сортування — у так званій «купі» (heap). Некластеризований індекс створює окрему структуру для швидкого пошуку, але **не змінює** саму фізичну організацію (сортування) даних у таблиці.

---
>Основні ознаки B tree
---

**B-дерево (B-Tree)** – це збалансована деревоподібна структура даних, яка широко використовується в системах зберігання та пошуку (зокрема, в індексах СУБД). Нижче наведені основні ознаки B-дерева:

1. **Збалансованість (Balanced):**  
    Висота дерева мінімально можлива для заданої кількості елементів (усі шляхи від кореня до листового вузла мають однакову довжину).
    
2. **Кілька ключів у кожному вузлі (Multi-key structure):**  
    На відміну від бінарного дерева пошуку, один вузол B-дерева може містити декілька ключів і мати декілька нащадків (дітей).
    
3. **Напівзаповненість вузлів (Min/Max keys):**  
    Кожен вузол (окрім кореневого) містить ключі в межах від ⌈m/2⌉\lceil m/2 \rceil до mm, де mm – максимальна кількість ключів, які вузол може зберігати. Це запобігає «розбалансуванню» дерева.
    
4. **Усі ключі зберігаються в листових вузлах (для кластеризованого варіанту):**  
    У деяких реалізаціях усі дані (або покажчики на дані) розташовуються саме в листових вузлах. Внутрішні вузли містять тільки ключі для навігації.
    
5. **Ефективне читання з диска (в контексті СУБД):**  
    Під час роботи з диском (або SSD) B-дерева мінімізують кількість дискових звернень, оскільки кожен вузол оптимізований під розмір сторінки (page) у файловій системі чи в СУБД.

---

**Визначення ефективності порядку стовпців у складеному (композитному) індексі** зазвичай зводиться до аналізу того, **як СУБД реально використовує** цей індекс у різних запитах. Нижче основні підходи та критерії:

---
>Як визначити ефективність порядку стовпців індексу
---

### 1. Перегляд планів виконання запитів (Execution Plans)

1. **Аналіз плану виконання** (Graphical Execution Plan / Showplan):
    - Подивіться, чи використовується _Index Seek_ (що є кращим) або _Index Scan_.
    - Якщо для запиту виконується _Scan_, це може означати, що порядок стовпців в індексі не відповідає умові `WHERE` або індекс загалом не покриває запит.
2. **Actual vs. Estimated Execution Plan**:
    - Порівняйте реальний (Actual) і план оцінки (Estimated), щоб зрозуміти, чи збігаються очікувані витрати з фактичними.

---

### 2. Статистика використання індексів

- Скористайтеся DMV (Dynamic Management Views), наприклад, `sys.dm_db_index_usage_stats`, щоб перевірити, скільки разів індекс виконував **Seek**, **Scan**, **Lookup** і т. д.
- Якщо індекс не використовується або виконує багато _Scan_ замість _Seek_, можливо, варто **змінити** порядок стовпців або додати нові індекси.

---

### 3. Selectivity (Вибірковість) та кардинальність даних

1. **Вибірковість** стовпця – це ступінь унікальності його значень (кількість унікальних значень відносно загальної кількості рядків).
    - Чим вище вибірковість, тим вигідніше ставити такий стовпець _ліворуч_ у визначенні індексу, оскільки це дасть змогу краще фільтрувати дані.
2. **Кардинальність** – кількість унікальних значень, а також їх розподіл (чи є «гарячі» значення, які повторюються дуже часто).
    - Якщо перший стовпець має низьку вибірковість (багато однакових значень), індекс часто виконуватиме _Scan_ замість _Seek_.

---

### 4. Аналіз найпоширеніших запитів

- Перевірте, які стовпці найчастіше фігурують у `WHERE`, `ORDER BY`, `GROUP BY`.
- Стовпці, які **насамперед** використовуються для фільтрації, доцільно поставити на перші позиції в індексному ключі.
- Якщо ви часто сортуєте за кількома стовпцями підряд (наприклад, `ORDER BY Col1, Col2`), тоді мати ці ж стовпці у такому самому порядку в індексі також допоможе уникнути додаткового сортування.

---

### 5. Покриття запитів (Covering)

1. **Включені стовпці (INCLUDE)**:
    - Якщо потрібні для вибірки або угруповання додаткові стовпці (не лише в `WHERE`), їх можна **додати як INCLUDE**. Це допоможе СУБД задовольнити запит лише індексом, без додаткових пошуків у таблиці.
2. **Перевірка “key lookup”**:
    - Якщо в плані видно операцію _Key Lookup_ (для некластеризованого індексу), це означає, що, крім ключових стовпців індексу, запит потребує й інші стовпці. У такому разі варто або змінити порядок, або додати колонки до індексу (як _INCLUDE_), щоб зменшити overhead від «доначитування» даних із таблиці.

---

## Коротко про головне

1. **Перевірте запити** (особливо найчастіші) та їхні плани виконання, щоб зрозуміти, як саме СУБД використовує індекс.
2. **Враховуйте порядок стовпців**: перші стовпці ключа – найважливіші для пошуку й фільтрації.
3. **Аналізуйте статистику** та вибірковість: стовпець із високою унікальністю доцільно ставити «ліворуч».
4. **Стежте за “Key Lookup”** операціями і, за потреби, доповнюйте індекс _INCLUDE-стовпцями_.

Таким чином, ефективність порядку стовпців у складеному індексі залежить від реальних сценаріїв використання (запитів), характеристик даних (вибірковості) і фактичного плану виконання в СУБД.

---
> Характеристики індексу
---

Нижче наведені основні **характеристики індексу**, які впливають на його ефективність у СУБД:

1. **Селективність (Selectivity)**
    
    - Показує, наскільки «унікальним» є стовпець (або набір стовпців) індексного ключа.
    - Чим вище селективність, тим менший відсоток рядків повертається при пошуку, і тим ефективніше індекс допомагає у відборі даних.
2. **Кардинальність (Cardinality)**
    
    - Кількість унікальних значень індексованого стовпця відносно загальної кількості рядків.
    - Висока кардинальність зазвичай корелює з високою селективністю, але кардинальність і її розподіл (частота різних значень) безпосередньо впливають на плани виконання запитів.
3. **Щільність (Density)**
    
    - Протилежний показник до селективності: відображає, наскільки часто певні значення повторюються.
    - Чим вища щільність, тим «гірше» індекс фільтрує дані, оскільки значення зустрічаються занадто часто.
4. **Покриття (Coverage)**
    
    - Відображає, чи може індекс повністю забезпечити дані для виконання запиту, не звертаючись до таблиці (або до кластеризованого індексу).
    - «Покриття» досягається, коли всі стовпці, потрібні для `SELECT` та `WHERE`, входять до індексу (у ключових стовпцях або `INCLUDE`).
5. **Розмір (Size)**
    
    - Має значення як обсяг на диску, так і в пам’яті (cache).
    - Чим більше колонок в індексі (особливо у ключі), тим більший розмір і, відповідно, вища вартість оновлення та зберігання.
6. **Накладні витрати на оновлення (Maintenance Overhead)**
    
    - Відображає, скільки додаткових операцій запису виконується під час `INSERT`, `UPDATE`, `DELETE`.
    - Чим більше індексів на таблиці, тим «дорожчі» в оновленні дані.
7. **Порядок стовпців (Key Order)**
    
    - Важливо для складених (композитних) індексів. Перший стовпець ключа має найбільший вплив на ефективність пошуку (`Index Seek`).
    - Порядок також важливий для `ORDER BY`, оскільки може дозволити SQL Server уникнути додаткового сортування.
8. **Тип (Type)**
    
    - Кластеризований (Clustered) чи некластеризований (Nonclustered).
    - Різні специфічні типи індексів: Columnstore, Filtered, Unique, Full-text тощо.

Кожна з цих характеристик впливає на те, як часто й наскільки ефективно індекс може використовуватись у запитах, а також якою буде вартість його підтримки.

---
