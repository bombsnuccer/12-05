# Домашнее задание к занятию "`Индексы`" - `Сидоренко Алексей`

---

### Задание 1

Напишите запрос к учебной базе данных, который вернёт процентное отношение общего размера всех индексов к общему размеру всех таблиц.

#### Решение

В PostgreSQL для работы с размерами объектов используются системные функции `pg_relation_size` и `pg_table_size`. Запрос к системному представлению `pg_stat_user_tables` позволяет получить сумму размеров всех пользовательских таблиц и их индексов:

```sql
SELECT 
    ROUND(
        100.0 * SUM(pg_relation_size(indexrelid)) / NULLIF(SUM(pg_relation_size(relid)), 0), 
        2
    ) AS index_to_table_ratio_percent
FROM pg_stat_user_tables;
```
Если требуется вывести детализацию для наглядности:

```sql

SELECT 
    pg_size_pretty(SUM(pg_relation_size(relid))) AS total_tables_size,
    pg_size_pretty(SUM(pg_relation_size(indexrelid))) AS total_indexes_size,
    ROUND(
        100.0 * SUM(pg_relation_size(indexrelid)) / NULLIF(SUM(pg_relation_size(relid)), 0), 
        2
    ) AS ratio_percent
FROM pg_stat_user_tables;
```
---

### Задание 2

Выполните explain analyze следующего запроса:
```sql
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id
```
- перечислите узкие места;
- оптимизируйте запрос: внесите корректировки по использованию операторов, при необходимости добавьте индексы.

#### Решение

### Узкие места исходного запроса:
Декартово произведение (Cartesian Product): В запросе отсутствует условие связи между таблицами inventory и film (i.film_id = f.film_id). Из-за этого каждая строка inventory перемножается с каждой строкой film, что приводит к экспоненциальному росту обрабатываемых строк (в базе Sakila это сотни тысяч лишних строк).
Нарушение SARGability (несистемный поиск): Использование функции date(p.payment_date) = '2005-07-30' оборачивает имя колонки в функцию. Это запрещает оптимизатору использовать индекс по колонке payment_date, вынуждая выполнять полное сканирование таблицы (Seq Scan).
Избыточные таблицы: Таблицы inventory и film не участвуют в формировании результирующей выборки (SELECT), но создают огромную нагрузку на этапе JOIN.
Неоптимальная агрегация: Использование оконной функции SUM() OVER() вместе с DISTINCT менее эффективно, чем классическая агрегация с GROUP BY, когда требуется просто получить сумму платежей по каждому клиенту.

### Оптимизация запроса:
Убираем таблицы film и inventory, так как они не нужны для результата.
Заменяем хрупкое условие соединения p.payment_date = r.rental_date на корректное соединение по внешнему ключу p.rental_id = r.rental_id.
Заменяем DISTINCT и оконную функцию на стандартный GROUP BY.
Делаем условие по дате SARGable, используя диапазонное сравнение, что позволит задействовать индекс.

### Оптимизированный запрос:
```sql
SELECT 
    CONCAT(c.last_name, ' ', c.first_name) AS customer_name,
    SUM(p.amount) AS total_amount
FROM payment p
JOIN rental r ON p.rental_id = r.rental_id
JOIN customer c ON r.customer_id = c.customer_id
WHERE p.payment_date >= '2005-07-30' AND p.payment_date < '2005-07-31'
GROUP BY c.customer_id, c.last_name, c.first_name;
```

### Необходимые индексы для ускорения:
```sql
-- Индекс для быстрого отбора платежей за конкретный день (B-Tree по умолчанию)
CREATE INDEX idx_payment_payment_date ON payment(payment_date);

-- Индекс для быстрого соединения таблиц payment и rental (обычно уже существует как FK, но стоит проверить)
CREATE INDEX idx_payment_rental_id ON payment(rental_id);
```
