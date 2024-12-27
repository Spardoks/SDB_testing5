# Домашнее задание к занятию "`Индексы`" - `Виталий Коряко`

https://github.com/netology-code/sdb-homeworks/blob/main/12-05.md


### Задание 1

Напишите запрос к учебной базе данных, который вернёт процентное отношение общего размера всех индексов к общему размеру всех таблиц.

### Решение 1

```
SELECT SUM(index_length) / SUM(data_length) * 100 as index_mem_perc
FROM information_schema.tables
WHERE data_length IS NOT NULL  AND index_length IS NOT NULL;
```
```
+----------------+
| index_mem_perc |
+----------------+
|        22.3702 |
+----------------+
```

### Задание 2

Выполните explain analyze следующего запроса:
```
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id
```
- перечислите узкие места;
- оптимизируйте запрос: внесите корректировки по использованию операторов, при необходимости добавьте индексы.

### Решение 2

```
explain analyze
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id

-> Table scan on <temporary>  (cost=2.5..2.5 rows=0) (actual time=4292..4292 rows=391 loops=1)
    -> Temporary table with deduplication  (cost=0..0 rows=0) (actual time=4292..4292 rows=391 loops=1)
        -> Window aggregate with buffering: sum(p.amount) OVER (PARTITION BY c.customer_id,f.title )   (actual time=1895..4136 rows=642000 loops=1)
            -> Sort: c.customer_id, f.title  (actual time=1895..1938 rows=642000 loops=1)
                -> Stream results  (cost=10.5e+6 rows=16.1e+6) (actual time=0.447..1455 rows=642000 loops=1)
                    -> Nested loop inner join  (cost=10.5e+6 rows=16.1e+6) (actual time=0.442..1262 rows=642000 loops=1)
                        -> Nested loop inner join  (cost=8.85e+6 rows=16.1e+6) (actual time=0.437..1099 rows=642000 loops=1)
                            -> Nested loop inner join  (cost=7.24e+6 rows=16.1e+6) (actual time=0.431..926 rows=642000 loops=1)
                                -> Inner hash join (no condition)  (cost=1.61e+6 rows=16.1e+6) (actual time=0.42..29.3 rows=634000 loops=1)
                                    -> Filter: (cast(p.payment_date as date) = '2005-07-30')  (cost=1.68 rows=16086) (actual time=0.181..4.18 rows=634 loops=1)
                                        -> Table scan on p  (cost=1.68 rows=16086) (actual time=0.173..2.92 rows=16044 loops=1)
                                    -> Hash
                                        -> Covering index scan on f using idx_title  (cost=103 rows=1000) (actual time=0.0377..0.178 rows=1000 loops=1)
                                -> Covering index lookup on r using rental_date (rental_date = p.payment_date)  (cost=0.25 rows=1) (actual time=974e-6..0.00128 rows=1.01 loops=634000)
                            -> Single-row index lookup on c using PRIMARY (customer_id = r.customer_id)  (cost=250e-6 rows=1) (actual time=113e-6..138e-6 rows=1 loops=642000)
                        -> Single-row covering index lookup on i using PRIMARY (inventory_id = r.inventory_id)  (cost=250e-6 rows=1) (actual time=100e-6..124e-6 rows=1 loops=642000)
```
Попробуем выявить узкие места запроса:
- выполняется 4292 мс - довольно долго
- видно, что основную часть времени занимают `Sort (actual time=1895..1938)` и `Window aggregate (actual time=1895..4136)`

Для упрощения сначала попробуем понять, что примерно делает запрос
- запрос пытается получить стоимость аренды всех фильмов, оплаченных и арендуемых в одну дату, для каждого пользователя
  - учитывая это, я бы сделал следующее
    - убрал бы работу с таблицей inventory, film, так как нам важно знать аренду и пользователя, для которого аренда создавалась, а фильм определится автоматически
      - в связи с чем убрал бы ещё выбор таблицы rental, связав бы её через join с пользователями
    - убрал бы оконную функцию и сделал бы суммирование по группе платежей и пользователей, для которых этот платёж был сделан (- window)
      - убрал бы distinct - он в таком случае не понадобится (- sort)
    - выбор таблицы payment надо оставить, так как за аренду для конкретного пользователя мог заплатить кто-то другой и его надо учесть в суммировании (join мог бы отсечь важные суммы)

Упрощаем на основе размышлений о желаемом выводе
```
explain analyze
select concat(c.last_name, ' ', c.first_name) as customer_fi, SUM(p.amount)
from payment p, customer c
join rental r on r.customer_id = c.customer_id  
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date
group by customer_fi;

-> Table scan on <temporary>  (actual time=6.03..6.07 rows=391 loops=1)
    -> Aggregate using temporary table  (actual time=6.03..6.03 rows=391 loops=1)
        -> Nested loop inner join  (cost=13169 rows=16698) (actual time=0.275..5.57 rows=642 loops=1)
            -> Nested loop inner join  (cost=7324 rows=16698) (actual time=0.267..5 rows=642 loops=1)
                -> Filter: (cast(p.payment_date as date) = '2005-07-30')  (cost=1633 rows=16086) (actual time=0.253..3.97 rows=634 loops=1)
                    -> Table scan on p  (cost=1633 rows=16086) (actual time=0.243..3.08 rows=16044 loops=1)
                -> Covering index lookup on r using rental_date (rental_date = p.payment_date)  (cost=0.25 rows=1.04) (actual time=0.00112..0.00149 rows=1.01 loops=634)
            -> Single-row index lookup on c using PRIMARY (customer_id = r.customer_id)  (cost=0.25 rows=1) (actual time=723e-6..749e-6 rows=1 loops=642)
```

Всё также получили 391 строку, но уже за 6 мс вместо 4292.
Убедимся, остортировав результаты первого и второго запросов, что оптимизированный запрос идентичен

[new_query.txt](./new_query.txt)
```
select concat(c.last_name, ' ', c.first_name) as customer_fi, SUM(p.amount)
from payment p, customer c
join rental r on r.customer_id = c.customer_id  
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date
group by customer_fi
order by customer_fi;
```
[old_query.txt](./old_query.txt)
```
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id
order by concat(c.last_name, ' ', c.first_name);
```

Команда diff в помощь
```
$ diff old_query.txt new_query.txt 
$ echo $?
0
```

## Дополнительные задания (со звёздочкой*)
Эти задания дополнительные, то есть не обязательные к выполнению, и никак не повлияют на получение вами зачёта по этому домашнему заданию. Вы можете их выполнить, если хотите глубже шире разобраться в материале.

### Задание 3*

Самостоятельно изучите, какие типы индексов используются в PostgreSQL. Перечислите те индексы, которые используются в PostgreSQL, а в MySQL — нет.

Приведите ответ в свободной форме.