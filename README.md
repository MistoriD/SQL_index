# Домашнее задание к занятию "`Индексы`" - `Шмагин Максим`


### Задание 1.

```
Напишите запрос к учебной базе данных, который вернёт процентное отношение общего размера всех индексов к общему размеру всех таблиц.

```

Ответ:

`Запрос:`

```sql
USE sakila;

SELECT 
    ROUND(
        (SUM(t.index_length) / SUM(t.data_length + t.index_length)) * 100, 2
    ) AS index_to_table_ratio_percent
FROM information_schema.tables t
WHERE t.table_schema = 'sakila';
```
`Вывод запроса:`

```
+------------------------------+
| index_to_table_ratio_percent |
+------------------------------+
|                        35.35 |
+------------------------------+
```
### Задание 2. 


Выполните explain analyze следующего запроса:

```sql
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id
```

- перечислите узкие места;
- оптимизируйте запрос: внесите корректировки по использованию операторов, при необходимости добавьте индексы.

Ответ:

`Узкие места`

1. "FROM p, r, c, i, f" — БД соединяет все таблицы друг с другом (16тыс × 1тыс × 599 × 4тыс × 1тыс = триллионы комбинаций), а не только нужные пары.

2. "p.payment_date = r.rental_date" — Сравнивает даты вместо ID аренды. Правильно: p.rental_id = r.rental_id (один-к-одному).

3. "DATE(p.payment_date) = '2005-07-30'" — Функция DATE() ломает индекс. БД не может быстро найти нужные строки, сканирует всё.

4. "DISTINCT + SUM() OVER()" — Два тяжёлых действия сразу: БД создаёт временную таблицу в памяти + сортирует её.

`Оптимизированный запрос:`

```sql
SELECT CONCAT(c.last_name, ' ', c.first_name) customer_name,
       SUM(p.amount) total_amount
FROM payment p
JOIN rental r ON p.rental_id = r.rental_id AND DATE(p.payment_date) = '2005-07-30'
JOIN customer c ON r.customer_id = c.customer_id
JOIN inventory i ON r.inventory_id = i.inventory_id
JOIN film f ON i.film_id = f.film_id
GROUP BY c.customer_id, c.last_name, c.first_name, f.title;
```

`Рекомендуемые индексы:`

```
CREATE INDEX idx_payment_date_rental ON payment(payment_date, rental_id);
CREATE INDEX idx_rental_customer_inventory ON rental(customer_id, inventory_id);
```


