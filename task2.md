## Задание 2

1. Удалите старую базу данных, если есть:
    ```shell
    docker compose down
    ```

2. Поднимите базу данных из src/docker-compose.yml:
    ```shell
    docker compose down && docker compose up -d
    ```

3. Обновите статистику:
    ```sql
    ANALYZE t_books;
    ```

4. Создайте полнотекстовый индекс:
    ```sql
    CREATE INDEX t_books_fts_idx ON t_books 
    USING GIN (to_tsvector('english', title));
    ```

5. Найдите книги, содержащие слово 'expert':
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE to_tsvector('english', title) @@ to_tsquery('english', 'expert');
    ```
    
    *План выполнения:*
    "QUERY PLAN"
"Bitmap Heap Scan on t_books  (cost=21.03..1336.08 rows=750 width=33) (actual time=0.033..0.034 rows=1 loops=1)"
"  Recheck Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)"
"  Heap Blocks: exact=1"
"  ->  Bitmap Index Scan on t_books_fts_idx  (cost=0.00..20.84 rows=750 width=0) (actual time=0.023..0.023 rows=1 loops=1)"
"        Index Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)"
"Planning Time: 2.501 ms"
"Execution Time: 0.073 ms"
    
    *Объясните результат:*
    Полнотекстовый индекс ускоряет поиск, позволяя не сканировать весь текст, время очень мало. 

7. Удалите индекс:
    ```sql
    DROP INDEX t_books_fts_idx;
    ```

8. Создайте таблицу lookup:
    ```sql
    CREATE TABLE t_lookup (
         item_key VARCHAR(10) NOT NULL,
         item_value VARCHAR(100)
    );
    ```

9. Добавьте первичный ключ:
    ```sql
    ALTER TABLE t_lookup 
    ADD CONSTRAINT t_lookup_pk PRIMARY KEY (item_key);
    ```

10. Заполните данными:
    ```sql
    INSERT INTO t_lookup 
    SELECT 
         LPAD(CAST(generate_series(1, 150000) AS TEXT), 10, '0'),
         'Value_' || generate_series(1, 150000);
    ```

11. Создайте кластеризованную таблицу:
     ```sql
     CREATE TABLE t_lookup_clustered (
          item_key VARCHAR(10) PRIMARY KEY,
          item_value VARCHAR(100)
     );
     ```

12. Заполните её теми же данными:
     ```sql
     INSERT INTO t_lookup_clustered 
     SELECT * FROM t_lookup;
     
     CLUSTER t_lookup_clustered USING t_lookup_clustered_pkey;
     ```

13. Обновите статистику:
     ```sql
     ANALYZE t_lookup;
     ANALYZE t_lookup_clustered;
     ```

14. Выполните поиск по ключу в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     "QUERY PLAN"
"Index Scan using t_lookup_pk on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=2.152..2.156 rows=1 loops=1)"
"  Index Cond: ((item_key)::text = '0000000455'::text)"
"Planning Time: 0.403 ms"
"Execution Time: 2.185 ms"
     
     *Объясните результат:*
     item_key это первичный ключ, поэтому для него автоматически создался индекс, который позволил ускорить запрос.

15. Выполните поиск по ключу в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     "QUERY PLAN"
"Index Scan using t_lookup_clustered_pkey on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.535..0.536 rows=1 loops=1)"
"  Index Cond: ((item_key)::text = '0000000455'::text)"
"Planning Time: 0.232 ms"
"Execution Time: 0.556 ms"
     
     *Объясните результат:*
     В дополнение к автоматически созданному индексу, запрос ускорила и кластеризация, которая упорядочила данные.

16. Создайте индекс по значению для обычной таблицы:
     ```sql
     CREATE INDEX t_lookup_value_idx ON t_lookup(item_value);
     ```

17. Создайте индекс по значению для кластеризованной таблицы:
     ```sql
     CREATE INDEX t_lookup_clustered_value_idx 
     ON t_lookup_clustered(item_value);
     ```

17. Выполните поиск по значению в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     "QUERY PLAN"
"Index Scan using t_lookup_value_idx on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.368..0.369 rows=0 loops=1)"
"  Index Cond: ((item_value)::text = 'T_BOOKS'::text)"
"Planning Time: 1.232 ms"
"Execution Time: 0.391 ms"
     
     *Объясните результат:*
     Запрос выполняется быстро, т.к. используется индекс, позволяющий быстро искать нужные строки, не сканируя всю таблицу.

18. Выполните поиск по значению в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     "QUERY PLAN"
"Index Scan using t_lookup_clustered_value_idx on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.412..0.412 rows=0 loops=1)"
"  Index Cond: ((item_value)::text = 'T_BOOKS'::text)"
"Planning Time: 0.955 ms"
"Execution Time: 0.432 ms"
     
     *Объясните результат:*
     Запрос выполняется достаточно быстро, благодаря использованию индекса.

19. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:
     
     *Сравнение:*
     Время выполнения в кластеризованной и некластеризованной таблице примерно равны. Кластеризация не ускорила время выполнения, т.к. данные изначально были упорядочены.
