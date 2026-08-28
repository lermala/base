---
title: Индексы и оптимизация
weight: 50
---

## План
### Индексы и оптимизация
- [Как я уронил прод на полтора часа (и при чем тут soft delete и partial index) / Habr](https://habr.com/ru/companies/skyeng/articles/802191)

Что добавить: 
- Индексы
- Поисковые пути (search_path)
- B-tree
- Hash index
- Clustered / Non-clustered
- Composite indexes
- Search paths (поисковые пути)
- Query Planner
- Explain Plan
- Оптимизация запросов

### Транзакции и конкурентность

- Транзакции
- ACID
- COMMIT / ROLLBACK
- Isolation levels
- Dirty Read
- Phantom Read
- MVCC
- Блокировки
- Deadlocks
- Сериализация

### Архитектура и масштабирование

- Репликация
- Шардирование
- Партиционирование
- Failover
- Кластеры
- Оркестрация
- Kubernetes + DB
- Stateful services

## Индексы в SQL

`Индекс` — это специальная структура данных, которая используется для ускорения поиска и обработки информации в таблицах базы данных. Он позволяет системе быстро находить нужные записи, минимизируя время выполнения запросов. Принцип работы можно сравнить с оглавлением в книге или предметным указателем — вместо последовательного перебора всех страниц система сразу обращается к указателю и находит нужные страницы.

Индексы ускоряют операции чтения, но могут замедлить операции записи, так как каждый индекс нужно обновить при изменении данных.

PostgreSQL поддерживает различные типы индексов:

- B-tree (по умолчанию) — для операций сравнения и сортировки
- Hash — для операций равенства
- GIN — для составных значений (массивы, JSON)
- GiST — для геометрических данных и полнотекстового поиска
- BRIN — для очень больших таблиц с естественной сортировкой

### Создание индекса

```sql
CREATE INDEX idx_email ON Users (email);
CREATE UNIQUE INDEX idx_email ON Users (email);
CREATE INDEX idx_full_name ON Student (last_name, first_name);
```

### Просмотреть все индексы
```sql
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'users';
```

### Удаление индекса:
```sql
DROP INDEX idx_email;
```

### Статистика запроса
Для получения статистики запроса используется:
- `EXPLAIN` - покажет план выполнения запроса без запуска.
- `EXPLAIN ANALYZE` - запустит запрос и покажет реальную статистику

```sql
EXPLAIN SELECT * FROM users WHERE email = 'maria.campbell@aol.com'

EXPLAIN ANALYZE
  SELECT id, first_name, last_name
  FROM Student
  WHERE first_name LIKE 'A%'
  AND last_name LIKE 'L%';
```

## Почему индекс не работает?
### Селективность

Селективность (Selectivity) — это метрика, которая говорит, насколько "уникальны" данные в колонке.

- Низкая селективность (много дублей, как у US, Gender) — индекс бесполезен.
- Высокая селективность (все значения уникальны, как Email) — индекс работает идеально.


Оптимизатор PostgreSQL (как и другие БД) обычно переключается на индекс, только если запрос выбирает менее 5-10% строк.

- Если выбираем мало (редкие данные) — Index Scan.
- Если выбираем много (популярные данные) — Seq Scan.

Вывод: Не создавайте индексы на колонки с низкой селективностью (пол, булевы флаги, статусы), если не планируете искать только редкие значения (например, только 'Deleted' пользователей или только ошибки в логах).

### Как функции убивают

SARGable = Search ARGument ABLE — способность аргумента поиска использовать индекс.

Чтобы индекс работал, колонка в условии WHERE должна оставаться неизменной — без функций и вычислений.

Правило: Оставьте колонку «чистой» с левой стороны оператора сравнения.
```sql
-- ❌ Плохо: колонка обернута в функцию
WHERE UPPER(email) = 'ADMIN@EXAMPLE.COM'

-- ✅ Хорошо: колонка чистая
WHERE email = 'admin@example.com'
```

#### Дата и время

```sql
CREATE INDEX idx_orders_date ON orders(order_date);

-- ❌ Плохо: вычисляется каждая строка
EXPLAIN ANALYZE SELECT * FROM orders
WHERE EXTRACT(YEAR FROM order_date) = 2023;

-- ✅ Хорошо: используется диапазон дат
EXPLAIN ANALYZE SELECT * FROM orders
WHERE order_date >= '2023-01-01' AND order_date < '2024-01-01';
```

#### Арифметика в условии

Пример. Найти товары, которые после 10% наценки будут стоить больше 1000:
```sql
CREATE INDEX idx_products_price ON products(price);

-- ❌ Плохо: вычисляется каждая строка
EXPLAIN ANALYZE SELECT * FROM products
WHERE price * 1.1 > 1000;

-- ✅ Хорошо: перенесли всю математику в правую часть
EXPLAIN ANALYZE SELECT * FROM products
WHERE price > 909.09;
```

#### LIKE с %

```sql
CREATE INDEX idx_name ON users(name varchar_pattern_ops);

-- ❌ Ищем '%Anna%', потому что разработчик решил перестраховаться
EXPLAIN ANALYZE SELECT * FROM users WHERE name LIKE '%Anna%';

-- ✅ Ищем только тех, кто начинается на 'Anna'
EXPLAIN ANALYZE SELECT * FROM users WHERE name LIKE 'Anna%';
```

{{< callout type="info" >}}
  Зачем varchar_pattern_ops? По умолчанию PostgreSQL сортирует строки с учётом локали (collation), что ломает LIKE 'prefix%'. Класс varchar_pattern_ops использует побайтовое сравнение — и индекс начинает работать.
{{< /callout >}}

Если бизнес-требование жёсткое — «Нужно обязательно находить 'Anna' по запросу 'nna'» — стандартный B-Tree индекс бессилен. Для таких задач используются другие инструменты (Trigram indexes или Full Text Search), о которых мы поговорим позже.

#### CAST

```sql
CREATE INDEX idx_users_phone ON users(phone_number);

-- ❌ Мы применили кастинг (функцию) к колонке
EXPLAIN ANALYZE SELECT * FROM users WHERE phone_number::bigint = 79001234567;

-- ✅ Приводим значение к строке
EXPLAIN ANALYZE SELECT * FROM users WHERE phone_number = '79001234567';
```

## Индексы
### Составные индексы

**Составной индекс** — это не просто "два индекса в одном". Это единая структура, отсортированная сначала по первой колонке, а затем (для одинаковых значений первой) — по второй. **Порядок колонок тут критически важен.**

Правило: Составной индекс работает, только если вы используете колонки слева направо, не пропуская начало. Нет первой колонки — индекс не работает.
Золотое правило: Сначала колонки, где мы ищем точное совпадение (=), затем колонки с диапазонами (>, <).

Пример:
```sql
CREATE INDEX idx_cat_price ON products(category_id, price);
```

#### Правила выбора порядка колонок
Как понять, создавать индекс (A, B) или (B, A)?

1. Самые частые фильтры — вперед. Ставьте на первое место ту колонку, которая чаще всего используется в фильтрах WHERE сама по себе. Это сделает индекс универсальным.
2. Равенство перед диапазоном. Это критически важно для производительности. Если в запросе есть равенство (category_id = 1) и диапазон (price > 100), то колонка с равенством должна идти первой.

### Покрывающие индексы

**Покрывающий индекс** — это индекс, который содержит все данные, нужные для запроса. Базе не нужно ходить в таблицу.

Пример
```sql
CREATE INDEX idx_products_cat_cover ON products(category_id) INCLUDE (name, price);

EXPLAIN ANALYZE SELECT name, price FROM products WHERE category_id = 5;
```

В примере:
- category_id — ключ индекса. По нему база ищет нужные строки.
- name, price — дополнительные данные. Хранятся рядом с ключом, чтобы не ходить в таблицу.
Когда запрос просит SELECT name, price WHERE category_id = 5:

База находит строки по ключу category_id
Тут же забирает name и price из индекса
В таблицу идти не нужно → Index Only Scan

### Частичный индекс

Частичный индекс — это индекс, который строится только по тем строкам, которые удовлетворяют условию WHERE.

Синтаксис максимально прост. Мы просто добавляем WHERE в конец команды создания индекса:

```sql
CREATE INDEX idx_orders_active_status
ON orders(status)
WHERE status = 'new';
```

### INCLUDE или составной индекс?
Вопрос: А чем (category_id) INCLUDE (price) отличается от (category_id, price)?

Оба позволяют сделать Index Only Scan. Но есть важное отличие:

Составной индекс (category_id, price) — данные отсортированы по обоим колонкам. Запрос с ORDER BY price получит сортировку бесплатно.

Индекс с INCLUDE — отсортирован только по ключу. Для ORDER BY price нужна дополнительная сортировка.

Когда что использовать:
- Нужна сортировка по второй колонке → составной индекс
- Вторая колонка только для SELECT → INCLUDE (индекс меньше)

Только выборка данных:
```sql
SELECT name, price FROM products WHERE category_id = 5;

-- Покрывающий индекс
CREATE INDEX idx ON products(category_id) INCLUDE (name, price);
```

Выборка + сортировка:

```sql
SELECT name, price FROM products WHERE category_id = 5 ORDER BY price;

-- Составной индекс (сортировка бесплатно)
CREATE INDEX idx ON products(category_id, price);
```