---
title: Работа с данными
weight: 500
---

## Примеры работы с данными

### Объединение строк

`STRING_AGG` объединяет набор строк с разделителем
`CONCAT` или `CONCAT_WS` объединяет переменное количество аргументов в одну строку. Null значения игнорируются.

```sql
select CONCAT_WS(' ', first_name, last_name, middle_name) as full_name,  -- Объединение 
    STRING_AGG(DISTINCT Class.name, ', ') as classes -- агрегация строк
FROM Schedule
join Class on Class.id = Schedule.class
join Teacher on Teacher.id = Schedule.teacher
group by 1;

-- Результат:
-- Aleksandr Sobolev Vladimirovich	| 11 A, 11 B
-- Petr Romashkin Alekseevich       | 10 A, 11 A, 11 B
```

### Вывод с условием или деление на категории (CASE)

```sql
SELECT name,
CASE
  WHEN SUBSTRING(name, 1, POSITION(' ' IN name) - 1) IN ('10', '11') THEN 'Старшая школа'
  WHEN SUBSTRING(name, 1, POSITION(' ' IN name) - 1) IN ('5', '6', '7', '8', '9') THEN 'Средняя школа'
  ELSE 'Начальная школа'
END AS stage
FROM Class

-- Результат:
-- 8 А | Средняя школа
-- 11 Б | Старшая школа
```

### Примеры задач

Пример 1. Топ-3 пилота по числу перевезённых пассажиров

```sql
select DISTINCT name,
    SUM(capacity) OVER (PARTITION BY pilot_id) as total_passengers
from Pilots
join Flights on pilot_id = first_pilot_id
join Planes on Flights.plane_id = Planes.plane_id
ORDER by total_passengers desc
limit 3;
```

Пример 2. Топ-5 исполнителей, чьи песни чаще всего входят в топ 10 песен
```sql
WITH artists_counts AS (
    select DISTINCT artists.artist_name, 
        DENSE_RANK() over (order by count(artists.artist_name) desc) as artist_rank
    from artists
    join songs on artists.artist_id = songs.artist_id
    join global_song_rank on global_song_rank.song_id = songs.song_id
    where song_rank <= 10
    group by artists.artist_name
)
select * 
from artists_counts
where artist_rank <= 5
order by artist_rank;
```

Пример 3. Третья транзакция пользователя
```sql
WITH ts AS(
    select user_id, 
        spend, 
        transaction_date,
        ROW_NUMBER() OVER(PARTITION BY user_id order by transaction_date) 
    from transactions
)

select user_id, spend, transaction_date 
from ts
where row_number = 3
```

Пример 4. Модель последнего устройства клиента
```sql
-- решение через оконную функцию LAST_VALUE (или FIRST_VALUE)
select DISTINCT ON (user_id)
    user_id,
    start_date,
    end_date,
    -- FIRST_VALUE(vendor_name) over (PARTITION BY user_id order by start_date desc ) as vendor_name,
    LAST_VALUE(vendor_name) over (
        PARTITION BY user_id 
        order by start_date ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) as vendor_name
from HistoryLog
join Devices on Devices.device_id=HistoryLog.device_id
order by user_id, start_date desc;

-- решение через CTE и ранги
with RankedHistory AS (
    select user_id,
        vendor_name,
        start_date,
        end_date,
        DENSE_RANK() OVER (
            PARTITION BY user_id
            order by start_date desc
        ) as rank
    from HistoryLog
    join Devices on Devices.device_id=HistoryLog.device_id
)
select user_id, vendor_name, start_date, end_date
from RankedHistory where rank = 1
```

Пример 5. Нарастающий итог транзакций (на каждую дату вывести сумму транзакций по данному счёту до данной даты включительно)

```sql
select account_id,
    dt,
    SUM(transaction_amt) OVER(
        PARTITION BY account_id
        ORDER BY dt
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW 
    ) as running_total
from TransactionLog
```

Пример 6. Последние 3 взятые произведения пользователем. Для каждой книги указать сколько раз книгу брали за все время
```sql
-- все книги которые пользователь брал
WITH UserBooks AS (
    select user_id, title as book_title,
        ROW_NUMBER() OVER(
            PARTITION BY user_id
            order by date_taken desc
        ) as row_number,
        COUNT(*) OVER (
            PARTITION BY Books.id
        ) as total_times_taken
    from OperationLogs 
    join BookCopies on BookCopies.id = OperationLogs.copy_id
    join BookEditions on BookEditions.id = BookCopies.edition_id
    join Books on Books.id = BookEditions.book_id
)

SELECT user_id, book_title, total_times_taken
from UserBooks
where row_number <= 3
```