## Задание 3

1. Создайте таблицу с большим количеством данных:
    ```sql
    CREATE TABLE test_cluster AS 
    SELECT 
        generate_series(1,1000000) as id,
        CASE WHEN random() < 0.5 THEN 'A' ELSE 'B' END as category,
        md5(random()::text) as data;
    ```

2. Создайте индекс:
    ```sql
    CREATE INDEX test_cluster_cat_idx ON test_cluster(category);
    ```

3. Измерьте производительность до кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    "QUERY PLAN"
"Bitmap Heap Scan on test_cluster  (cost=59.17..7696.73 rows=5000 width=68) (actual time=24.009..90.941 rows=500222 loops=1)"
"  Recheck Cond: (category = 'A'::text)"
"  Heap Blocks: exact=8334"
"  ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..57.92 rows=5000 width=0) (actual time=22.780..22.781 rows=500222 loops=1)"
"        Index Cond: (category = 'A'::text)"
"Planning Time: 0.721 ms"
"Execution Time: 106.279 ms"
    
    *Объясните результат:*
    Мы создали таблицу с большим количеством данных, и проходим по ней ища совпадения, т.к. данных много, то это долго.

4. Выполните кластеризацию:
    ```sql
    CLUSTER test_cluster USING test_cluster_cat_idx;
    ```
    
    *Результат:*
    CLUSTER

Запрос завершён успешно, время выполнения: 1 secs 254 msec.

5. Измерьте производительность после кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    "QUERY PLAN"
"Index Scan using test_cluster_cat_idx on test_cluster  (cost=0.42..14607.67 rows=499500 width=39) (actual time=0.484..70.449 rows=500222 loops=1)"
"  Index Cond: (category = 'A'::text)"
"Planning Time: 0.684 ms"
"Execution Time: 87.165 ms"
    
    *Объясните результат:*
    Данные в таблице упорядочились по столбу индекса, что позволило ускорить запрос.

6. Сравните производительность до и после кластеризации:
    
    *Сравнение:*
    После кластеризации скорость увеличилась, т.к. на сортированных данных запрос выполняется быстрее. 
