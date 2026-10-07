```sql
explain (analyze , costs off,VERBOSE ) select * from t where a = 1 and b = 't';

```

Создали составной индекс по двум полям a and b
Составной индекс по двум полям весит меньше чем два индекса по отдельным полям.

```text
Index Scan using t_a_b_idx on public.t (actual time=0.013..0.013 rows=0 loops=1)
"  Output: a, b, c"
  Index Cond: ((t.a = 1) AND (t.b = 't'::text))
Planning Time: 0.463 ms
Execution Time: 0.044 ms
```

Получаем скан по индексу.

Допустим есть индекс по полям а и b.
Если напишем запрос который будет искать только по полю а, то сканирование будет через созданный составной индекс.
Потому что при поиске по индексу в дереве данные отсортированы по а. Пожтому такой индекс будет работать.

Но если в запросе будет поиск по b а индекс будет построен сначала по а и потом по b то вероятее всего получим seq scan.

```SQL
explain (analyze , costs off,VERBOSE ) select * from t where a = 1;
```

```TEXT
Index Scan using t_a_b_idx on public.t (actual time=0.014..0.015 rows=1 loops=1)
"  Output: a, b, c"
  Index Cond: (t.a = 1)
Planning Time: 0.053 ms
Execution Time: 0.026 ms

```

И поиск по b:

```sql
explain (analyze , costs off,VERBOSE ) select * from t where b = 't';
```

```text
Seq Scan on public.t (actual time=4.119..4.630 rows=1049 loops=1)
"  Output: a, b, c"
  Filter: (t.b = 't'::text)
  Rows Removed by Filter: 98951
Planning Time: 0.059 ms
Execution Time: 4.671 ms

```


