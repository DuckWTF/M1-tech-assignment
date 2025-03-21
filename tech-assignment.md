## Задание 1

Ниже приведён пример данных в таблицах и запросы, иллюстрирующие, как решать поставленные задачи.

### 1.1. Актуальное состояние товаров на 2020-06-01

**Таблица** `items` (история изменения цен и названий товаров):

| item_id | name               | price | update_date |
|---------|--------------------|-------|------------|
| 1       | Ручка гелевая      | 10    | 2020-02-01 |
| 2       | Карандаш 1HH       | 2     | 2020-01-01 |
| 1       | Ручка шариковая    | 10    | 2020-03-01 |
| 3       | Ластик             | 5     | 2020-07-01 |
| 2       | Карандаш 1HH       | 3     | 2020-05-01 |
| 1       | Ручка шариковая    | 5     | 2020-05-01 |
| 2       | Карандаш 1H        | 7     | 2020-06-01 |

#### Решение через оконные функции

```sql
WITH items_validity AS (
    SELECT
        item_id,
        name,
        price,
        update_date AS valid_from,
        LEAD(update_date, 1, DATE '9999-12-31')
            OVER (PARTITION BY item_id ORDER BY update_date) AS valid_to
    FROM items
)
SELECT
    iv.item_id,
    iv.name,
    iv.price,
    iv.valid_from AS update_date
FROM items_validity iv
WHERE iv.valid_from <= DATE '2020-06-01'
  AND iv.valid_to   >  DATE '2020-06-01';
```

1. Через оконную функцию `LEAD` формируем интервалы (valid_from, valid_to) действия цены.  
2. Выбираем из полученных интервалов те, которые захватывают дату `2020-06-01`.

#### Решение через подзапрос с группировкой

```sql
SELECT i.item_id,
       i.name,
       i.price,
       i.update_date
FROM items i
JOIN (
    SELECT item_id,
           MAX(update_date) AS last_update
    FROM items
    WHERE update_date <= DATE '2020-06-01'
    GROUP BY item_id
) t ON i.item_id  = t.item_id
    AND i.update_date = t.last_update;
```

1. В подзапросе выбираем для каждого `item_id` максимальную дату `update_date`, которая не превышает 2020-06-01.  
2. Соединяем это с основной таблицей, чтобы получить соответствующую цену, название и т. д.

---

### 1.2. Товары, купленные по цене больше или равно 3

**Таблица** `orders` (заказы):

| order_id | user_id | item_id | order_date |
|----------|--------|---------|------------|
| 1        | 1      | 1       | 2020-02-01 |
| 2        | 2      | 2       | 2020-02-01 |
| 3        | 1      | 3       | 2020-07-01 |
| 4        | 3      | 2       | 2020-07-01 |
| 5        | 2      | 1       | 2020-04-01 |
| 6        | 1      | 1       | 2020-06-01 |

Чтобы во всех запросах учитывать актуальную цену товара на момент заказа, воспользуемся CTE `actual_prices`. Для каждого заказа в `orders` выбираем последнюю запись из `items`, у которой `update_date` ≤ `order_date`.

```sql
WITH actual_prices AS (
    SELECT
        o.order_id,
        o.user_id,
        o.item_id,
        o.order_date,
        (
            SELECT i.price
            FROM items i
            WHERE i.item_id = o.item_id
              AND i.update_date <= o.order_date
            ORDER BY i.update_date DESC
            LIMIT 1
        ) AS actual_price
    FROM orders o
)
SELECT
    order_id,
    user_id,
    item_id,
    order_date,
    actual_price
FROM actual_prices
WHERE actual_price >= 3;
```

---

### 1.3. Сумма покупок клиента 1

```sql
WITH actual_prices AS (
    SELECT
        o.order_id,
        o.user_id,
        o.item_id,
        o.order_date,
        (
            SELECT i.price
            FROM items i
            WHERE i.item_id = o.item_id
              AND i.update_date <= o.order_date
            ORDER BY i.update_date DESC
            LIMIT 1
        ) AS actual_price
    FROM orders o
)
SELECT
    SUM(actual_price) AS total_for_user_1
FROM actual_prices
WHERE user_id = 1;
```

---

### 1.4. Сумма всех покупок до 2020-05-01 включительно

```sql
WITH actual_prices AS (
    SELECT
        o.order_id,
        o.user_id,
        o.item_id,
        o.order_date,
        (
            SELECT i.price
            FROM items i
            WHERE i.item_id = o.item_id
              AND i.update_date <= o.order_date
            ORDER BY i.update_date DESC
            LIMIT 1
        ) AS actual_price
    FROM orders o
)
SELECT
    SUM(actual_price) AS total_before_may
FROM actual_prices
WHERE order_date <= DATE '2020-05-01';
```

---

### 1.5. Сумма всех заказов и средняя цена заказа поквартально

```sql
WITH actual_prices AS (
    SELECT
        o.order_id,
        o.user_id,
        o.item_id,
        o.order_date,
        (
            SELECT i.price
            FROM items i
            WHERE i.item_id = o.item_id
              AND i.update_date <= o.order_date
            ORDER BY i.update_date DESC
            LIMIT 1
        ) AS actual_price
    FROM orders o
)
SELECT
    DATE_TRUNC('quarter', order_date) AS quarter_start,
    SUM(actual_price)                 AS total_amount,
    AVG(actual_price)                 AS average_price
FROM actual_prices
GROUP BY DATE_TRUNC('quarter', order_date)
ORDER BY quarter_start;
```

---

### 1.6. Как оптимизировать запросы для больших объемов данных

1. **Индексы**  
   - Создать индекс по `(item_id, update_date)` в таблице `items`. Тогда при поиске нужной цены `WHERE item_id = ? AND update_date <= ? ORDER BY update_date DESC` база данных не будет сканировать всю таблицу.  
   - При частых запросах по дате заказов или пользователям также полезно иметь индексы по `(order_date)`, `(user_id, order_date)` и т. д.

2. **Материализация актуальных цен**  
   - Предварительно рассчитать периоды действия цены (`valid_from`, `valid_to`) и хранить в отдельной таблице или материализованном представлении. Тогда при сопоставлении заказа с ценой не придётся выполнять сортировку и искать последнее предшествующее обновление «на лету».

3. **Партиционирование**  
   - Партиционировать таблицу `orders` (и при необходимости `items`) по дате. Это ускорит запросы по недавним периодам.

4. **LATERAL / JOIN + правильный план**  
   - Можно использовать специальные техники (например, `LATERAL` join) для эффективной выборки последней цены.  
   - Если какие-то итоги часто требуются в реальном времени, можно кэшировать результаты во внешнем хранилище (Redis, Memcached) или вести агрегированные таблицы/materialized views и выбирать оттуда.

---

## Задание 2

У нас есть поток заказов (примерно 100 000 заказов в день), которые записываются в таблицу `orders`.  
Данные переносятся в DWH, но заказы в первые 6 месяцев активно меняются (обновляется статус, стоимость и т. д.), а затем ещё 2–5 лет возможны редкие изменения. При этом мы **не** можем использовать явное поле `updated_at`.

### 2.1. Как можно построить архитектуру DWH на основе Data Vault 2.0?

**Общие принципы Data Vault**  
- **Hub** хранит бизнес-ключи (например, `order_id`) и минимальный набор тех. полей (hash key, load date и т. д.).  
- **Link** отражает связи между хабами (например, «заказ–товар», «заказ–клиент»).  
- **Satellite** хранит атрибуты, которые изменяются во времени (статус, сумма и т. д.). В Satellite для каждой версии записи есть `load_date` (когда версия попала в DWH), `hash_diff` (для сравнения с предыдущей версией).  

**Хранение актуальной версии и историчности**  
- В Raw Vault накапливаются **все** версии изменений (каждое изменение — новая строка в Satellite).  
- Для получения актуальной версии (например, при аналитических запросах) применяют либо **PIT (Point-In-Time) таблицу**, либо **Latest-satellite view**.  
  - **PIT** заранее хранит ссылки на последнюю версию атрибутов.  
  - **Latest-satellite view** динамически выбирает самую свежую запись из Satellite.  

Таким образом:  
- **История** не теряется (все изменения сохраняются).  
- **Актуальные данные** получаем быстро, не просматривая всю историю.  
- **«Путешествие во времени»** становится возможным при помощи PIT или механизма дат в Satellite.  

**Учет разных паттернов обновлений**  
- Можно разделить часто изменяющиеся поля в отдельный Satellite (частые изменения в первые 6 месяцев).  
- Редко изменяющиеся поля хранить в другом Satellite.  

После 6 месяцев (когда изменений почти нет) мы не пишем новые версии в Satellite, пока что-то реально не изменится. Если нужно, «закрываем» запись (end_date) или помечаем её current_ind. Для ускорения аналитики можно формировать PIT-таблицу, обновляемую по расписанию.

### 2.2. Какие сущности вы выделите и какую роль они будут играть?

1. **Hub_Orders**  
   - Хранит бизнес-ключ `order_id` + тех. поля (`order_hk`, `load_date`, `record_source`).  

2. **Hub_Customers**  
   - Хранит бизнес-ключ `customer_id` + тех. поля (`customer_hk`, `load_date`, `record_source`).  

3. **Hub_Products**  
   - Хранит бизнес-ключ `product_id` + тех. поля (`product_hk`, `load_date`, `record_source`).  

4. **Link_OrderCustomer**  
   - Связь между `Hub_Orders` и `Hub_Customers`. Содержит хэш-связку, `load_date`, `record_source` и ссылки на оба Hub.  

5. **Link_OrderItems**  
   - Связь многие-ко-многим «Заказ–Товар» (учитывает количество и проч.). Содержит хэш-связку, `load_date`, `record_source`, а также ссылки на `Hub_Orders` и `Hub_Products`.  

6. **Satellite(s) для Hub_Orders**  
   - Хранит атрибуты заказа (статус, итоговая сумма, дата оплаты и т. д.). При каждом изменении — новая версия (новая строка).  
   - Содержит `order_hk`, `load_date`, `record_source`, `hash_diff` и сами поля (статус, сумма...).  

7. **Satellite(s) для Link_OrderItems**  
   - Хранит детали по каждой позиции заказа: количество, скидка и т. д. Каждый апдейт (например, смена количества) создаёт новую строку в Satellite.  

### 2.3. Как можно обновлять актуальное состояние заказов без использования `updated_at`?

- В Data Vault 2.0 используется **`load_date`** — дата/время загрузки в хранилище, и **`hash_diff`** — хэш от набора атрибутов.  
- При загрузке мы ищем в Hub (по бизнес-ключу `order_id`), есть ли уже запись. Если нет — добавляем. Если есть — смотрим Satellite.  
- В Satellite считаем новый `hash_diff`, сравниваем его с предыдущим. Если хэш не поменялся — запись не вставляем. Если изменился — добавляем новую строку с новой `load_date`.  
- Таким образом, история фиксируется (каждая новая версия — новая строка), а актуальная версия — та, у которой самая поздняя `load_date`, не помеченная `end_date` (или не перекрытая другой версией).

### 2.4. Как можно оптимизировать работу с данными такого рода?

1. **Разделение Satellite на «FastChanges» и «SlowChanges»**  
   - Для часто меняющихся полей (статус, сумма, скидка и т. д.) — отдельный Satellite.  
   - Для редко меняющихся (адрес, другие статические данные) — другой.  

2. **End Dating**  
   - После периода активных изменений можно закрыть старые записи в Satellite (заполнить `end_date`).  
   - При редких изменениях создаётся новая версия.  

3. **PIT (Point-In-Time) и Bridge таблицы**  
   - **PIT** хранит ссылки на актуальные версии Satellite для быстрого получения среза на конкретный момент.  
   - **Bridge** упрощает денормализацию связей «многие ко многим».  

4. **Партиционирование**  
   - Делить большие таблицы Satellite по периоду (к примеру, активная часть в одной партиции, архив в другой).  
   - Ускоряет запросы к «свежим» данным и упрощает работу с «архивом».  

5. **Индексация по хэш-ключам и `load_date`**  
   - Быстрый поиск предыдущей версии (сравнение `hash_diff` и т. д.).  

6. **Материализованные представления**  
   - Если часто нужны актуальные данные, удобно создавать материализованные вьюхи или отдельные агрегаты, чтобы не сканировать весь Raw Vault при каждом запросе.  

Это позволит нам сохранить всю историю изменений, предоставлять быстрый доступ к актуальному состоянию и эффективно управлять данными при больших объёмах и неоднородном характере обновлений.
