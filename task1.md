# Задание 1: BRIN индексы и bitmap-сканирование

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

4. Создайте BRIN индекс по колонке category:
   ```sql
   CREATE INDEX t_books_brin_cat_idx ON t_books USING brin(category);
   ```

5. Найдите книги с NULL значением category:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE category IS NULL;
   ```
   
   *План выполнения:*
   "QUERY PLAN"
"Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.010..0.011 rows=0 loops=1)"
"  Recheck Cond: (category IS NULL)"
"  ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.009..0.009 rows=0 loops=1)"
"        Index Cond: (category IS NULL)"
"Planning Time: 0.314 ms"
"Execution Time: 0.039 ms"
   
   *Объясните результат:*
   [Ваше объяснение]

6. Создайте BRIN индекс по автору:
   ```sql
   CREATE INDEX t_books_brin_author_idx ON t_books USING brin(author);
   ```

7. Выполните поиск по категории и автору:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books 
   WHERE category = 'INDEX' AND author = 'SYSTEM';
   ```
   
   *План выполнения:*
   "QUERY PLAN"
"Bitmap Heap Scan on t_books  (cost=12.16..2379.31 rows=1 width=33) (actual time=22.327..22.327 rows=0 loops=1)"
"  Recheck Cond: ((category)::text = 'INDEX'::text)"
"  Rows Removed by Index Recheck: 150000"
"  Filter: ((author)::text = 'SYSTEM'::text)"
"  Heap Blocks: lossy=1225"
"  ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.16 rows=76143 width=0) (actual time=0.258..0.258 rows=12250 loops=1)"
"        Index Cond: ((category)::text = 'INDEX'::text)"
"Planning Time: 3.561 ms"
"Execution Time: 23.198 ms"
   
   *Объясните результат (обратите внимание на bitmap scan):*
   [Ваше объяснение]

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   "QUERY PLAN"
"Sort  (cost=3100.14..3100.15 rows=6 width=7) (actual time=39.661..39.662 rows=6 loops=1)"
"  Sort Key: category"
"  Sort Method: quicksort  Memory: 25kB"
"  ->  HashAggregate  (cost=3100.00..3100.06 rows=6 width=7) (actual time=39.368..39.370 rows=6 loops=1)"
"        Group Key: category"
"        Batches: 1  Memory Usage: 24kB"
"        ->  Seq Scan on t_books  (cost=0.00..2725.00 rows=150000 width=7) (actual time=0.017..12.830 rows=150000 loops=1)"
"Planning Time: 0.726 ms"
"Execution Time: 40.081 ms"
   
   *Объясните результат:*
   [Ваше объяснение]

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   "QUERY PLAN"
"Aggregate  (cost=3100.04..3100.05 rows=1 width=8) (actual time=24.164..24.166 rows=1 loops=1)"
"  ->  Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=0) (actual time=24.153..24.154 rows=0 loops=1)"
"        Filter: ((author)::text ~~ 'S%'::text)"
"        Rows Removed by Filter: 150000"
"Planning Time: 2.091 ms"
"Execution Time: 24.221 ms"
   
   *Объясните результат:*
   [Ваше объяснение]

10. Создайте индекс для регистронезависимого поиска:
    ```sql
    CREATE INDEX t_books_lower_title_idx ON t_books(LOWER(title));
    ```

11. Подсчитайте книги, начинающиеся на 'O':
    ```sql
    EXPLAIN ANALYZE
    SELECT COUNT(*) 
    FROM t_books 
    WHERE LOWER(title) LIKE 'o%';
    ```
   
   *План выполнения:*
   "QUERY PLAN"
"Aggregate  (cost=3476.88..3476.89 rows=1 width=8) (actual time=44.085..44.086 rows=1 loops=1)"
"  ->  Seq Scan on t_books  (cost=0.00..3475.00 rows=750 width=0) (actual time=44.079..44.081 rows=1 loops=1)"
"        Filter: (lower((title)::text) ~~ 'o%'::text)"
"        Rows Removed by Filter: 149999"
"Planning Time: 0.684 ms"
"Execution Time: 44.145 ms"
   
   *Объясните результат:*
   [Ваше объяснение]

12. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_brin_cat_idx;
    DROP INDEX t_books_brin_author_idx;
    DROP INDEX t_books_lower_title_idx;
    ```

13. Создайте составной BRIN индекс:
    ```sql
    CREATE INDEX t_books_brin_cat_auth_idx ON t_books 
    USING brin(category, author);
    ```

14. Повторите запрос из шага 7:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE category = 'INDEX' AND author = 'SYSTEM';
    ```
   
   *План выполнения:*
   "QUERY PLAN"
"Bitmap Heap Scan on t_books  (cost=12.16..2379.31 rows=1 width=33) (actual time=2.383..2.384 rows=0 loops=1)"
"  Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))"
"  Rows Removed by Index Recheck: 8819"
"  Heap Blocks: lossy=73"
"  ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.16 rows=76143 width=0) (actual time=0.059..0.059 rows=730 loops=1)"
"        Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))"
"Planning Time: 0.559 ms"
"Execution Time: 2.409 ms"
   
   *Объясните результат:*
   [Ваше объяснение]
