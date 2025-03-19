# Домашнее задание к занятию "Индексы" - Морозов Александр

### Задание 1

Напишите запрос к учебной базе данных, который вернёт процентное отношение общего размера всех индексов к общему размеру всех таблиц.

```
SELECT SUM(index_length) / SUM(data_length) * 100
FROM INFORMATION_SCHEMA.TABLES
WHERE table_schema = 'sakila';
```

![alt text](https://github.com/Mars12121/hw-12-05/blob/main/img/1.png)

### Задание 2

Выполните explain analyze следующего запроса:
```sql
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id
```
- перечислите узкие места;
- оптимизируйте запрос: внесите корректировки по использованию операторов, при необходимости добавьте индексы.

Ответ:
Анализ запроса без индексов

```sql
explain analyze
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id
```

```
Table scan on <temporary>  (cost=2.5..2.5 rows=0) (actual time=6028..6028 rows=391 loops=1)
    -> Temporary table with deduplication  (cost=0..0 rows=0) (actual time=6028..6028 rows=391 loops=1)
        -> Window aggregate with buffering: sum(p.amount) OVER (PARTITION BY c.customer_id,f.title )   (actual time=2654..5828 rows=642000 loops=1)
            -> Sort: c.customer_id, f.title  (actual time=2654..2710 rows=642000 loops=1)
                -> Stream results  (cost=22.6e+6 rows=16.5e+6) (actual time=0.683..2105 rows=642000 loops=1)
                    -> Nested loop inner join  (cost=22.6e+6 rows=16.5e+6) (actual time=0.677..1855 rows=642000 loops=1)
                        -> Nested loop inner join  (cost=20.9e+6 rows=16.5e+6) (actual time=0.671..1652 rows=642000 loops=1)
                            -> Nested loop inner join  (cost=19.3e+6 rows=16.5e+6) (actual time=0.664..1425 rows=642000 loops=1)
                                -> Inner hash join (no condition)  (cost=1.65e+6 rows=16.5e+6) (actual time=0.651..37.6 rows=634000 loops=1)
                                    -> Filter: (cast(p.payment_date as date) = '2005-07-30')  (cost=1.72 rows=16500) (actual time=0.283..5.54 rows=634 loops=1)
                                        -> Table scan on p  (cost=1.72 rows=16500) (actual time=0.259..4.21 rows=16044 loops=1)
                                    -> Hash
                                        -> Covering index scan on f using idx_title  (cost=112 rows=1000) (actual time=0.0424..0.239 rows=1000 loops=1)
                                -> Covering index lookup on r using rental_date (rental_date = p.payment_date)  (cost=0.969 rows=1) (actual time=0.00152..0.00204 rows=1.01 loops=634000)
                            -> Single-row index lookup on c using PRIMARY (customer_id = r.customer_id)  (cost=250e-6 rows=1) (actual time=181e-6..206e-6 rows=1 loops=642000)
                        -> Single-row covering index lookup on i using PRIMARY (inventory_id = r.inventory_id)  (cost=250e-6 rows=1) (actual time=150e-6..175e-6 rows=1 loops=642000)
```
Больше всего времени уходит на выполнение Оконной функции и сортировки Sort: c.customer_id, f.title

Запрос выводит всех Клиентов у которых есть платежи за 2005-07-30, если дата Платежа совпадает с датой Аренды и Сумму платежей по таким арендам. Таблица film f здесь лишняя, т.к. поля из нее не выводятся в select. Поэтому f.title мошно исключить и из оконной функции. Получается запрос:

```sql
explain analyze
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id)
from payment p, rental r, customer c, inventory i
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id
```

```
-> Table scan on <temporary>  (cost=2.5..2.5 rows=0) (actual time=9.85..9.9 rows=391 loops=1)
    -> Temporary table with deduplication  (cost=0..0 rows=0) (actual time=9.85..9.85 rows=391 loops=1)
        -> Window aggregate with buffering: sum(p.amount) OVER (PARTITION BY c.customer_id )   (actual time=8.81..9.69 rows=642 loops=1)
            -> Sort: c.customer_id  (actual time=8.79..8.82 rows=642 loops=1)
                -> Stream results  (cost=30875 rows=16520) (actual time=0.297..8.64 rows=642 loops=1)
                    -> Nested loop inner join  (cost=30875 rows=16520) (actual time=0.291..8.42 rows=642 loops=1)
                        -> Nested loop inner join  (cost=25093 rows=16520) (actual time=0.287..7.34 rows=642 loops=1)
                            -> Nested loop inner join  (cost=19311 rows=16520) (actual time=0.28..6.58 rows=642 loops=1)
                                -> Filter: (cast(p.payment_date as date) = '2005-07-30')  (cost=1674 rows=16500) (actual time=0.266..5.15 rows=634 loops=1)
                                    -> Table scan on p  (cost=1674 rows=16500) (actual time=0.256..3.93 rows=16044 loops=1)
                                -> Covering index lookup on r using rental_date (rental_date = p.payment_date)  (cost=0.969 rows=1) (actual time=0.00158..0.00211 rows=1.01 loops=634)
                            -> Single-row index lookup on c using PRIMARY (customer_id = r.customer_id)  (cost=0.25 rows=1) (actual time=0.00101..0.00103 rows=1 loops=642)
                        -> Single-row covering index lookup on i using PRIMARY (inventory_id = r.inventory_id)  (cost=0.25 rows=1) (actual time=0.00149..0.00153 rows=1 loops=642)
```

Добавлен индекс на payment_date и изменено условие:

```sql
explain analyze
SELECT CONCAT(c.last_name, ' ', c.first_name) 'Клиент', SUM(p.amount) AS 'Сумма'
FROM payment p
  JOIN customer c ON c.customer_id = p.customer_id
  JOIN rental r ON r.rental_id = p.rental_id 
  JOIN inventory i ON i.inventory_id = r.inventory_id
where p.payment_date >= '2005-07-30' AND p.payment_date < DATE_ADD('2005-07-30', INTERVAL 1 DAY)
	and p.payment_date = r.rental_date
GROUP by c.customer_id
```

```
-> Table scan on <temporary>  (actual time=22.3..22.3 rows=391 loops=1)
    -> Aggregate using temporary table  (actual time=22.3..22.3 rows=391 loops=1)
        -> Nested loop inner join  (cost=590 rows=31.7) (actual time=15.7..21.9 rows=634 loops=1)
            -> Nested loop inner join  (cost=579 rows=31.7) (actual time=15.7..20.9 rows=634 loops=1)
                -> Nested loop inner join  (cost=351 rows=634) (actual time=0.0694..1.26 rows=634 loops=1)
                    -> Filter: ((r.rental_date >= TIMESTAMP'2005-07-30 00:00:00') and (r.rental_date < <cache>(('2005-07-30' + interval 1 day))))  (cost=129 rows=634) (actual time=0.0617..0.295 rows=634 loops=1)
                        -> Covering index range scan on r using rental_date over ('2005-07-30 00:00:00' <= rental_date < '2005-07-31 00:00:00')  (cost=129 rows=634) (actual time=0.0599..0.184 rows=634 loops=1)
                    -> Single-row covering index lookup on i using PRIMARY (inventory_id = r.inventory_id)  (cost=0.25 rows=1) (actual time=0.00138..0.0014 rows=1 loops=634)
                -> Filter: (p.payment_date = r.rental_date)  (cost=0.257 rows=0.05) (actual time=0.0304..0.0309 rows=1 loops=634)
                    -> Index lookup on p using fk_payment_rental (rental_id = r.rental_id)  (cost=0.257 rows=1.03) (actual time=0.0302..0.0307 rows=1 loops=634)
            -> Single-row index lookup on c using PRIMARY (customer_id = p.customer_id)  (cost=0.253 rows=1) (actual time=0.00135..0.00138 rows=1 loops=634)
```


## Дополнительные задания (со звёздочкой*)
Эти задания дополнительные, то есть не обязательные к выполнению, и никак не повлияют на получение вами зачёта по этому домашнему заданию. Вы можете их выполнить, если хотите глубже шире разобраться в материале.

### Задание 3*

Самостоятельно изучите, какие типы индексов используются в PostgreSQL. Перечислите те индексы, которые используются в PostgreSQL, а в MySQL — нет.

*Приведите ответ в свободной форме.*